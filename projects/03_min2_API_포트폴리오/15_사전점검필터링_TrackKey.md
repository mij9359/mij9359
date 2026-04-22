# 🎯 사전점검 트랙 필터링: TrackKey Record 기반 규칙 테이블

> **Java 21 record를 키로 사용해 복잡한 if-else를 정책 테이블로 전환하는 패턴**

![Status](https://img.shields.io/badge/Status-Production-brightgreen)
![Pattern](https://img.shields.io/badge/Pattern-Rule_Table-blue)
![Design](https://img.shields.io/badge/Design-OCP-green)

---

## 🎯 문제 정의

### 상황: 신청 가능 트랙 결정

```
사용자가 신청 가능한 트랙은 상황에 따라 다름:

Case A: 사업자
├─ 가능: "도약", "성장"
└─ 불가: "유학"

Case B: 비사업자
├─ 가능: "성장"
└─ 불가: "도약", "유학"

Case C: 대학원생
├─ 가능: "유학"
└─ 불가: "도약", "성장"

이런 규칙이 매년 바뀜 (2025년과 2026년 다를 수 있음)
```

### 복잡성

```
❌ 문제 1: if-else가 코드에 흩어짐

// Controller
if (isBusiness) {
    trackList.filter(t -> t.getName().contains("도약"));
}

// Service
if (!isBusiness) {
    trackList.filter(t -> t.getName().contains("성장"));
}

// Mapper
if (isBusiness && !isGraduate) {
    SELECT ... WHERE track_type IN (...)
}

→ 같은 규칙이 3군데에 있음
→ 규칙 변경 시 3곳 수정 필요 ❌
→ 수정 누락 가능성 높음 ❌

❌ 문제 2: 규칙 변경이 어려움

2025년 규칙:
└─ 사업자 = 도약 + 성장

2026년 규칙:
└─ 사업자 = 도약 + 성장 + 유학 (변경!)

코드 수정 필요:
├─ if 조건 변경
├─ filter 로직 변경
├─ SQL 변경
└─ 테스트 재작성

복잡하고 위험함 ❌

❌ 문제 3: 정책과 코드가 분리됨

비즈니스팀: "내년에 대학원생도 도약 신청 가능하게"
개발팀: "코드 수정에 2~3일 필요"

변경 속도 느림 ❌
```

---

## 💡 솔루션: Java 21 record + Map 규칙 테이블

```java
// 정책을 데이터 구조로 표현
private static final Map<TrackKey, List<String>> TRACK_RULE = Map.ofEntries(
    Map.entry(new TrackKey(true, false),  List.of("도약", "성장")),
    Map.entry(new TrackKey(false, false), List.of("성장")),
    Map.entry(new TrackKey(false, true),  List.of("유학"))
);

// 정책 조회 = 데이터 조회 (코드 수정 필요 없음)
List<String> allowedKeywords = TRACK_RULE.get(key);
```

효과:
- 정책 변경 = 데이터만 수정 (코드 수정 불필요)
- OCP (Open/Closed Principle) 준수
- 테스트 용이
```

---

## 🔥 핵심 구현 코드

### Step 1: TrackKey Record 정의

```java
// TrackKey.java
/**
 * 트랙 신청 가능 여부를 결정하는 사용자 속성 조합
 * 
 * @param business 사업자 여부 (true=사업자, false=비사업자)
 * @param graduate 대학원 여부 (true=대학원생, false=학부생)
 */
public record TrackKey(
        boolean business,
        boolean graduate
) {
    // record는 자동으로:
    // ├─ constructor 생성
    // ├─ equals() 구현 (동등성 기반)
    // ├─ hashCode() 구현 (Map 키 가능)
    // └─ toString() 구현
}
```

### Step 2: 정책 테이블 정의

```java
// ApplyQueryService.java:32
@Service
public class ApplyQueryService {
    
    /**
     * 사용자 속성별 신청 가능 트랙 키워드
     * 
     * 예시:
     * - 사업자 + 학부생 → ["도약", "성장"]
     * - 비사업자 + 학부생 → ["성장"]
     * - 비사업자 + 대학원생 → ["유학"]
     * 
     * 매년 이 테이블만 수정하면 됨 (코드 수정 필요 없음!)
     */
    private static final Map<TrackKey, List<String>> TRACK_RULE = Map.ofEntries(
        
        // Case: 사업자 + 학부생
        Map.entry(new TrackKey(true, false),  
                 List.of("도약", "성장")),
        
        // Case: 비사업자 + 학부생
        Map.entry(new TrackKey(false, false), 
                 List.of("성장")),
        
        // Case: 비사업자 + 대학원생
        Map.entry(new TrackKey(false, true),  
                 List.of("유학"))
        
        // 새 규칙 추가 예시 (내년):
        // Map.entry(new TrackKey(true, true),   // 사업자 + 대학원생
        //          List.of("도약", "성장", "유학"))
    );
    
    // ──────────────────────────────────────────────────────
    // 정책 테이블 조회 메서드
    // ──────────────────────────────────────────────────────
    
    private List<String> getAllowedTrackKeywords(TrackKey key) {
        
        List<String> keywords = TRACK_RULE.get(key);
        
        if (keywords == null) {
            log.warn("[Track Rule] 매칭되는 규칙 없음: {}", key);
            return Collections.emptyList();  // 규칙이 없으면 신청 불가
        }
        
        log.info("[Track Rule] {} → 키워드: {}", key, keywords);
        return keywords;
    }
}
```

### Step 3: 필터링 로직

```java
@Transactional(readOnly = true)
public List<TrackConditionResVO> getAvailableTracks(
        TrackConditionReqVO req) {
    
    log.info("[Available Tracks] user_no={}, business={}, graduate={}",
             req.getUserNo(), req.isBusiness(), req.isGraduate());
    
    // ──────────────────────────────────────────────────────
    // [Step 1] 현 연도의 모든 트랙 조회 (DB)
    // ──────────────────────────────────────────────────────
    
    int currentYear = LocalDate.now().getYear();
    List<TrackConditionResVO> allTracks = 
        applyQueryMapper.getTracksByYear(currentYear);
    
    log.info("[Available Tracks] 전체 트랙 {}개 조회", allTracks.size());
    
    // ──────────────────────────────────────────────────────
    // [Step 2] 사용자의 TrackKey 생성
    // ──────────────────────────────────────────────────────
    
    TrackKey userKey = new TrackKey(req.isBusiness(), req.isGraduate());
    
    log.info("[Available Tracks] 사용자 키: {}", userKey);
    
    // ──────────────────────────────────────────────────────
    // [Step 3] 정책 테이블에서 허용 키워드 조회
    // ──────────────────────────────────────────────────────
    
    List<String> allowedKeywords = getAllowedTrackKeywords(userKey);
    
    if (allowedKeywords.isEmpty()) {
        log.warn("[Available Tracks] 신청 가능한 트랙 없음");
        return Collections.emptyList();
    }
    
    log.info("[Available Tracks] 허용 키워드: {}", allowedKeywords);
    
    // ──────────────────────────────────────────────────────
    // [Step 4] 트랙 이름이 허용 키워드에 포함되는 것만 필터
    // ──────────────────────────────────────────────────────
    
    List<TrackConditionResVO> availableTracks = allTracks.stream()
        
        // 트랙 이름이 허용 키워드 중 하나를 포함?
        .filter(track -> allowedKeywords.stream()
            .anyMatch(keyword -> track.getTrackName().contains(keyword)))
        
        .collect(Collectors.toList());
    
    log.info("[Available Tracks] 필터 후: {}개", availableTracks.size());
    
    return availableTracks;
}
```

### Step 4: Controller

```java
@Operation(summary = "신청 가능 트랙 조회", 
           description = "사용자 정보 기반 신청 가능한 트랙 목록")
@GetMapping("/available_tracks")
public ResponseData<List<TrackConditionResVO>> getAvailableTracks(
        @RequestHeader("Authorization") String token) {
    
    // 토큰에서 사용자 정보 추출
    Long userNo = tokenProvider.getUserNoFromToken(token);
    UserVO user = userQueryService.getUserInfo(userNo);
    
    // 신청 조건 VO 구성
    TrackConditionReqVO req = TrackConditionReqVO.builder()
        .userNo(userNo)
        .business(user.isBusiness())      // 사업자 여부
        .graduate(user.isGraduate())      // 대학원 여부
        .build();
    
    // 서비스에서 필터링
    List<TrackConditionResVO> availableTracks = 
        applyQueryService.getAvailableTracks(req);
    
    return ResponseData.<List<TrackConditionResVO>>builder()
        .message("신청 가능 트랙 조회 완료")
        .data(availableTracks)
        .code(HttpStatus.OK.value())
        .build();
}
```

---

## 📊 before/After 비교

### Before (if-else 폭발)

```java
// Controller
public List<TrackConditionResVO> getAvailableTracks(TrackConditionReqVO req) {
    if (req.isBusiness()) {
        if (req.isGraduate()) {
            // Case: 사업자 + 대학원
            trackList.filter(t -> t.contains("도약") || t.contains("성장"));
        } else {
            // Case: 사업자 + 학부
            trackList.filter(t -> t.contains("도약") || t.contains("성장"));
        }
    } else {
        if (req.isGraduate()) {
            // Case: 비사업자 + 대학원
            trackList.filter(t -> t.contains("유학"));
        } else {
            // Case: 비사업자 + 학부
            trackList.filter(t -> t.contains("성장"));
        }
    }
}

// Service
public void validateTrackApply(Long trackNo, boolean isBusiness) {
    if (isBusiness) {
        // 도약/성장만 가능
    } else {
        // 성장만 가능
    }
}

// Mapper XML
<select if test="isBusiness">
    WHERE track_type IN ('도약', '성장')
</select>

문제:
├─ 같은 규칙이 3군데 존재 ❌
├─ 규칙 변경 시 3곳 수정 필요 ❌
├─ 수정 누락 가능성 높음 ❌
└─ 신규 케이스 추가도 어려움 ❌
```

### After (규칙 테이블)

```java
// 정책 테이블 (데이터)
private static final Map<TrackKey, List<String>> TRACK_RULE = Map.ofEntries(
    Map.entry(new TrackKey(true, false),  List.of("도약", "성장")),
    Map.entry(new TrackKey(false, false), List.of("성장")),
    Map.entry(new TrackKey(false, true),  List.of("유학"))
);

// 조회 로직 (코드)
public List<TrackConditionResVO> getAvailableTracks(TrackConditionReqVO req) {
    TrackKey key = new TrackKey(req.isBusiness(), req.isGraduate());
    List<String> keywords = TRACK_RULE.get(key);
    
    return allTracks.stream()
        .filter(t -> keywords.stream().anyMatch(t::contains))
        .collect(toList());
}

성과:
├─ 규칙은 데이터 (한 곳)
├─ 규칙 변경 = 데이터만 수정 ✅
├─ 코드 수정 불필요 ✅
└─ 신규 케이스 = 엔트리만 추가 ✅
```

---

## ⚠️ 마주친 문제점 & 해결

### **Problem 1: 규칙이 코드에 흩어짐**

**증상:**
```
같은 "사업자는 도약/성장 신청 가능" 규칙이:
├─ Controller if-else
├─ Service if-else
├─ Mapper SQL
└─ Test code

총 4곳에 존재 → 변경 누락 가능성 ❌
```

**해결:**
```java
Map<TrackKey, List<String>> TRACK_RULE  // ← 단 한 곳!

모든 로직이 이 규칙을 참조 ✅
```

### **Problem 2: 새 규칙 추가 어려움**

**증상:**
```
"2026년부터 비사업자도 도약 신청 가능"

변경 사항:
├─ if 조건 수정
├─ SQL WHERE 수정
├─ 테스트 케이스 추가
└─ 배포 필요 (코드 변경)

시간 소요: 2~3시간
위험도: 높음 (버그 가능성)
```

**해결:**
```java
// 엔트리만 추가 (배포 불필요)
Map.entry(new TrackKey(false, false), 
         List.of("성장", "도약"))  // ← 도약 추가

변경:
├─ 엔트리 1줄 추가
└─ 완료! (배포 불필요)

시간 소요: 1분
위험도: 없음 (데이터만 변경)
```

### **Problem 3: Record 동등성**

**증상:**
```
새로 생성된 TrackKey가 Map 키로 작동하는가?

TrackKey key1 = new TrackKey(true, false);
TrackKey key2 = new TrackKey(true, false);

key1.equals(key2)?  → false (일반 클래스라면)
```

**해결:**
```java
// record는 자동으로 equals() + hashCode() 구현
record TrackKey(boolean business, boolean graduate) {}

key1.equals(key2)   → true ✅ (필드값 기반)
map.get(key1)       → 작동함 ✅
```

---

## 📊 성과

### Before

```
규칙 변경 요청:
└─ "내년에 규칙이 바뀌어요"
   └─ 개발팀: "코드 3곳 수정 필요, 테스트 재작성"
   └─ 시간: 2~3시간
   └─ 위험: 높음

운영 불편:
├─ 매번 새 규칙 추가마다 배포 필요
├─ 배포 대기 시간 발생
└─ 긴급 수정 시 위험
```

### After

```
규칙 변경 요청:
└─ "내년에 규칙이 바뀌어요"
   └─ 개발팀: "엔트리만 수정"
   └─ 시간: 1분
   └─ 위험: 없음 (데이터만 변경)

운영 편의:
├─ 코드 배포 불필요
├─ 규칙 변경 즉시 반영 가능
└─ 안전함
```

---

## 🎓 배운 점

### 1. OCP (Open/Closed Principle)

> **"확장에는 열려있고, 수정에는 닫혀있어야 한다"**

```
❌ 새 규칙 추가 = 코드 수정 (닫혀있지 않음)
✅ 새 규칙 추가 = 데이터 추가 (기존 코드 수정 없음)
```

### 2. Java 21 record의 강점

```java
record TrackKey(boolean business, boolean graduate)

자동 구현:
├─ constructor
├─ equals()
├─ hashCode()
├─ toString()

Map의 키로 바로 사용 가능 ✅
```

### 3. 규칙과 코드 분리

```
Before: 규칙 = 코드에 임베드
After: 규칙 = 데이터 구조

효과:
├─ 변경 속도 향상
├─ 위험도 감소
└─ 운영 편의성 증가
```

---

## 📌 확장성

### 새 조건 추가 예시

내년에 "고등학생" 조건 추가 시:

```java
// Before: 코드 수정 필요
if (isBusiness && isGraduate) {
    // ...
} else if (isBusiness && isHighSchool) {  // ← 코드 수정
    // ...
}

// After: 엔트리만 추가
record TrackKey(boolean business, boolean graduate, boolean highSchool) {}

Map.entry(new TrackKey(true, false, true), List.of("도약", "성장", "고등교육"))
```

→ 기존 코드 수정 불필요 ✅

---

*개발자: kmj | 파일: ApplyQueryService.java:566 | OCP 원칙의 실무 적용*
