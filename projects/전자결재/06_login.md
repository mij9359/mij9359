# 🔑 API 6: 로그인 — AuthCoreService.login

**난이도**: ★★★★ | **복잡도**: Redis race + 동시성 | **영향**: 일일 10,000회

## 📌 개요

사용자 인증 후 accessToken, refreshToken 발급.

## 🔴 문제점

```java
@Transactional
public LoginResVO login(LoginReqVO req) {
    Authentication auth = authenticationManager.authenticate(...);
    AuthDTO.TokenDto token = tokenProvider.createToken(auth);
    
    // Redis SET overwrite
    if (!saveRefreshToken(token.getRefreshToken(), user.getId())) {
        throw new ServerBizException(JWT_REGIST_ERROR);
    }
    
    authRepository.setLastLoginAt(user.getUserId());
}
```

**문제**:
1. **동일 유저 동시 로그인**: Redis SET overwrite (한쪽 토큰 무효화)
2. **Redis 실패**: DB 롤백 불가
3. **brute-force**: 로그인 실패 잠금 없음

## 💡 해결방법

```java
public LoginResVO login(LoginReqVO req, HttpServletRequest http) {
    // 1. 로그인 실패 카운터
    String failKey = "login:fail:" + userId;
    Long fails = redisManager.increment(failKey);
    if (fails == 1L) redisManager.expire(failKey, Duration.ofMinutes(15));
    if (fails >= 5) {
        authRepository.lockUser(userId);
        throw new ServerBizException(LOCKED_USER);
    }
    
    // 2. 인증
    Authentication auth = authenticationManager.authenticate(...);
    AuthDTO.TokenDto token = tokenProvider.createToken(auth);
    
    // 3. deviceId 기반 멀티 세션
    String deviceId = req.getDeviceId();  // 앱 UUID 또는 fingerprint
    String key = redisKeyConfig.getUserRedisKey(REFRESHTOKEN, userId, deviceId);
    redisManager.setValueWithTtl(key, token.getRefreshToken(), Duration.ofDays(14));
    
    // 4. 실패 카운터 초기화
    redisManager.delete(failKey);
    
    // 5. DB 업데이트
    authRepository.setLastLoginAt(userId);
    
    return new LoginResVO(token, user);
}
```

## 📊 효과

- ✅ 다중 디바이스 로그인
- ✅ brute-force 방어
- ✅ Redis + DB 동기화

