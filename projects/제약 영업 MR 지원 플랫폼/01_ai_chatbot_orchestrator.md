# 🤖 API 1: AI 챗봇 오케스트레이터 연동

**난이도**: ★★★★★ | **복잡도**: 외부 연동 + 5단계 예외처리 | **영향**: 챗봇 안정성

---

## 📌 개요

MR이 RAG 기반 AI 챗봇에 질의하고, AI가 답변하지 못할 경우 사내 관리자(PM/MI/PEx)에게 **자동 Handoff**하는 시스템.

**엔드포인트**: `POST /v1/ai_chat_bot/command/chat/ai`

---

## 🔴 문제점 (Before)

### 1️⃣ 외부 AI 서버의 불안정한 응답
- 네트워크 끊김 → HttpStatusCodeException
- S3 토큰 만료 → "ExpiredToken" 에러
- 응답 형식 변경 → JsonProcessingException
→ 각각 다르게 처리되며 원인 파악 어려움

### 2️⃣ 세션 라이프사이클 불명확
- 유저가 여러 개 세션 동시 생성 가능
- Race condition 발생 가능

### 3️⃣ RAG 응답 가공 로직 복잡
- markdown 형식 파싱 (### 답변: / ### References)
- ref_dict JSON 배열 라벨 재조립

### 4️⃣ 로그 저장 실패가 API 전체 실패로 전파
- 로그는 부가 정보일 뿐

---

## 💡 해결방법 (After)

### ✅ 1. 5단계 예외 계층화 처리

```java
try {
    AiChatResponse result = callOrchestrator(aiRequest);
} catch (HttpStatusCodeException e) {
    if (e.getResponseBody().contains("ExpiredToken")) {
        causeMessage = "S3 토큰 만료";
    }
    throw new AiServerException(causeMessage, e);
} catch (RestClientException | JsonProcessingException | 
         NullPointerException | Exception e) {
    logError(...);
}
```

### ✅ 2. 세션 라이프사이클 엄격 관리

```java
public ChatSessionDto getOrCreateSession(String userId) {
    // 기존 진행 중 세션만 사용
    ChatSessionDto session = queryRepo.findOngoingSession(userId);
    if (session != null) return session;
    
    // 기존 처리 대기 → 종료로 전환
    queryRepo.updateAllPendingToClosedByUserId(userId);
    
    // 새 세션 생성
    return commandRepo.insertSession(newSession);
}
```

### ✅ 3. RAG 응답 마크다운 파싱

```java
String answerText = extractBetween(response, "### 답변:", "### References");
List<Map<String, Object>> refList = objectMapper.readValue(refJson);
// 프론트용 출처 라벨 재조립
```

### ✅ 4. 로그 실패 안전망

```java
try {
    saveAiChatLog(logDto);
} catch (DataAccessException e) {
    log.warn("Log save failed but continue");
    // 비즈니스 흐름은 보호
}
```

---

## 📊 성능 개선

| 항목 | Before | After | 개선 |
|---|---|---|---|
| **에러 파악 시간** | 10분 | 1분 | **90% ↓** |
| **응답 시간** | 800ms | 650ms | **19% ↓** |

---

## 🔗 관련 기술

- Spring: RestTemplate, @Transactional
- MyBatis: 동적 SQL, 세션 관리
- Exception Handling: 5단계 계층화

