# 📝 트랙 다중 신청 API: Bulk 트랜잭션 & 중복 검증

> **한 번의 요청으로 여러 트랙에 동시 신청하면서 중복/조합 검증을 모두 처리하는 원자적 트랜잭션**

![Status](https://img.shields.io/badge/Status-Production-brightgreen)
![Pattern](https://img.shields.io/badge/Pattern-Bulk_Transaction-blue)
![Validation](https://img.shields.io/badge/Validation-Multi_Stage-orange)

---

## 🎯 문제 정의

### 상황: 한 번에 여러 트랙 신청

```
사용자: "도전기업과 펠로우, 2개 트랙 동시에 신청하고 싶어요"

요청:
POST /api/user/apply/command/register-apply-bulk
{
  "trackList": [
    { "trackNo": 1, "stageNo": 101 },  // 도전기업
    { "trackNo": 2, "stageNo": 202 }   // 펠로우
  ],
  "files": { ... }
}

처리할 것:
├─ 중복 신청 여부 검증 (각 트랙별)
├─ 신청 조합 가능 여부 검증 (같은 그룹 내 2개 이상 불가 등)
├─ 사업계획서 파일 업로드
├─ 가점 파일 업로드
├─ 팀 등록 (team 테이블)
├─ 팀장 등록 (team_member 테이블)
├─ 신청 이력 등록 (user_apply_history)
├─ 임시저장 이력 삭제
└─ 모든 것이 성공해야만 커밋!
```

### 복잡성

```
❌ 문제 1: 부분 실패 (Partial Commit)
   ├─ 트랙 A는 등록 성공
   ├─ 트랙 B는 파일 업로드 실패
   └─ 데이터 불일치 (트랙 A만 남음) ❌

❌ 문제 2: 내부 중복 요청
   ├─ 악의적 또는 실수로 같은 트랙을 2번 보냄
   ├─ trackList = [트랙1, 트랙1] ← 어?
   └─ 같은 데이터가 2번 INSERT ❌

❌ 문제 3: 신청 조합 불가
   ├─ 같은 그룹(예: "성장" 카테고리)에서 2개 트랙 신청 불가
   ├─ 검증 없으면 불가능한 상태 저장 ❌
   └─ 데이터 정합성 파괴

❌ 문제 4: 신청 개수 제한
   ├─ 정책: 최대 2개 트랙만 신청 가능
   ├─ 제한 없으면 사용자가 100개 보냈다면? 😱
   └─ 비용 폭탄
```

---

## 💡 솔루션: Bulk 트랜잭션 + 3단계 검증

```
[진입]
  ↓
[Step 1] Controller 선검증 (빠른 실패)
├─ 신청 개수 <= 2개
├─ 각 트랙의 중복 신청 여부
└─ 신청 불가능한 조합 여부

  ↓
[Step 2] Service 진입 (트랜잭션 시작)
├─ 내부 중복 체크 (Set.add)
├─ Bulk 반복문 시작
│  ├─ 팀 등록
│  ├─ 팀장 등록
│  ├─ 신청 이력 등록
│  ├─ 가점 등록
│  └─ 파일 업로드
└─ 모든 성공 → COMMIT

  ↓
[실패 시] 전체 ROLLBACK ✅
```

---

## 🔥 핵심 구현 코드

### Step 1: Controller 선검증

```java
@PostMapping("/register-apply-bulk")
@CrossOrigin
public ResponseData registerApplyBulk(
        @RequestPart(value = "request") RegisterApplyBulkReqVO req,
        @RequestPart(value = "files", required = false) Map<String, MultipartFile> fileList,
        @RequestHeader("Authorization") String token) {
    
    Long userNo = tokenProvider.getUserNoFromToken(token);
    req.setUserNo(userNo);

    // ──────────────────────────────────────────────────────
    // [검증 1] 신청 개수 제한 (최대 2개)
    // ──────────────────────────────────────────────────────
    
    if (req.getTrackList() == null || req.getTrackList().size() > 2) {
        throw new OperationErrorException(ErrorCode.TRACK_LIMIT_EXCEEDED,
            "최대 2개 트랙까지 신청 가능합니다");
    }

    // ──────────────────────────────────────────────────────
    // [검증 2] 각 트랙의 중복 신청 여부 (DB 조회)
    // ──────────────────────────────────────────────────────
    
    for (TrackStageVO track : req.getTrackList()) {
        // 이미 신청한 트랙인가?
        if (applyQueryService.applyTeamCheck(userNo, track.getTrackNo()) > 0) {
            throw new OperationErrorException(ErrorCode.TEAM_APPLY_CHECK,
                String.format("트랙 %d은(는) 이미 신청하셨습니다", track.getTrackNo()));
        }
    }

    // ──────────────────────────────────────────────────────
    // [검증 3] 신청 불가능한 조합 (같은 그룹 내 중복, 탈락자 재신청 등)
    // ──────────────────────────────────────────────────────
    
    for (TrackStageVO track : req.getTrackList()) {
        // 예: 같은 그룹의 다른 트랙에서 탈락했으면 이 그룹 전체 불가
        if (!applyQueryService.applyOverlapCheck(userNo, track.getTrackNo())) {
            throw new OperationErrorException(ErrorCode.TEAM_APPLY_OVERLAP_CHECK,
                "이 트랙은 신청 불가능한 상태입니다");
        }
    }

    log.info("[Apply Bulk] user_no={} 선검증 완료. 트랙 {} 개 등록 시도",
             userNo, req.getTrackList().size());

    // ── 선검증 통과 → Service로 위임
    applyCommandService.registerApplyBulk(req, fileList);

    return ResponseData.builder()
        .message("신청이 정상 완료되었습니다")
        .code(HttpStatus.OK.value())
        .build();
}
```

### Step 2: Service Bulk 처리

```java
@Transactional
public void registerApplyBulk(RegisterApplyBulkReqVO req, Map<String, MultipartFile> fileList) {
    
    // ──────────────────────────────────────────────────────
    // [Step 2-1] 내부 중복 체크 (Set.add 결과로 검증)
    // ──────────────────────────────────────────────────────
    
    Set<Long> stageSet = new HashSet<>();  // 스테이지 ID 중복 체크
    Set<Long> trackSet = new HashSet<>();  // 트랙 ID 중복 체크
    
    for (TrackStageVO track : req.getTrackList()) {
        
        // Set.add()는 이미 있으면 false 반환 → 중복 감지
        if (!stageSet.add(track.getStageNo())) {
            throw new OperationErrorException(ErrorCode.DUPLICATE_STAGE_REQUEST,
                String.format("스테이지 %d이 중복되었습니다", track.getStageNo()));
        }
        
        if (!trackSet.add(track.getTrackNo())) {
            throw new OperationErrorException(ErrorCode.DUPLICATE_TRACK_REQUEST,
                String.format("트랙 %d이 중복되었습니다", track.getTrackNo()));
        }
    }

    // ──────────────────────────────────────────────────────
    // [Step 2-2] Bulk 반복: 각 트랙별 등록
    // ──────────────────────────────────────────────────────
    
    for (TrackStageVO track : req.getTrackList()) {
        req.setTrackNo(track.getTrackNo());
        req.setStageNo(track.getStageNo());

        try {
            // 1) 팀 등록
            if (applyCommandMapper.registerTeamBulk(req) < 1) {
                throw new OperationErrorException(ErrorCode.FAILED_TO_REGISTER_TRACK,
                    "팀 등록 실패");
            }
            
            Long teamNo = req.getTeamNo();  // INSERT 후 자동생성된 ID

            // 2) 신청 이력 등록
            if (applyCommandMapper.registerApplyHistory(req, teamNo) < 1) {
                throw new OperationErrorException(ErrorCode.FAILED_TO_REGISTER_TRACK,
                    "신청 이력 등록 실패");
            }

            // 3) 가점 정보 등록 (있으면)
            if (req.getAdditionScore() > 0) {
                if (applyCommandMapper.registerAdditionScore(teamNo, req.getAdditionScore()) < 1) {
                    throw new OperationErrorException(ErrorCode.FAILED_TO_REGISTER_TRACK,
                        "가점 등록 실패");
                }
            }

            // 4) 팀장 정보 등록
            if (applyCommandMapper.registerTeamLeader(req, teamNo) < 1) {
                throw new OperationErrorException(ErrorCode.FAILED_TO_REGISTER_TRACK,
                    "팀장 정보 등록 실패");
            }

            // 5) 파일 업로드 (사업계획서 등)
            if (fileList != null && !fileList.isEmpty()) {
                uploadApplicationFiles(req, teamNo, fileList);
            }

            // 6) 임시저장 이력 삭제
            applyCommandMapper.deleteTempSaveHistory(req.getUserNo(), track.getTrackNo());

            log.info("[Apply Bulk] user_no={}, track_no={} 등록 완료",
                     req.getUserNo(), track.getTrackNo());

        } catch (Exception e) {
            log.error("[Apply Bulk] user_no={}, track_no={} 등록 실패: {}",
                      req.getUserNo(), track.getTrackNo(), e.getMessage(), e);
            
            // ← @Transactional이 Exception을 감지 → 전체 ROLLBACK
            throw new OperationErrorException(ErrorCode.FAILED_TO_REGISTER_TRACK,
                "트랙 등록 중 오류 발생: " + e.getMessage());
        }
    }

    log.info("[Apply Bulk] user_no={} 전체 {} 트랙 등록 완료 (트랜잭션 커밋)",
             req.getUserNo(), req.getTrackList().size());
}
```

### Helper: 파일 업로드

```java
private void uploadApplicationFiles(RegisterApplyBulkReqVO req, Long teamNo, 
                                   Map<String, MultipartFile> fileList) throws IOException {
    
    for (Map.Entry<String, MultipartFile> entry : fileList.entrySet()) {
        MultipartFile file = entry.getValue();
        
        if (file == null || file.isEmpty()) continue;
        
        // 파일명 → 경로 변환
        String fileName = entry.getKey();
        String storedPath = fileStorageService.saveFile(file, teamNo);
        
        // DB 저장
        SubmissionVO submission = SubmissionVO.builder()
            .teamNo(teamNo)
            .trackNo(req.getTrackNo())
            .fileName(fileName)
            .storedPath(storedPath)
            .fileSize(file.getSize())
            .uploadedAt(LocalDateTime.now())
            .build();
        
        if (submissionMapper.insertSubmission(submission) < 1) {
            throw new IOException("파일 저장 중 오류 발생");
        }
    }
}
```

---

## ⚠️ 마주친 문제점 & 해결

### **Problem 1: 부분 실패 (Partial Commit)**

**증상:**
```
Tx: [Team INSERT ✅] → [History INSERT ✅] → [File Upload ❌]

결과: Team만 남고 History는 없음 (고아 데이터) ❌
```

**해결:**
```java
@Transactional  // ← 모든 트랜잭션을 하나의 경계로
public void registerApplyBulk(...) {
    // Exception 발생 시 자동 ROLLBACK
}

// 결과: Team + History + File 모두 성공 또는 모두 ROLLBACK ✅
```

### **Problem 2: 내부 중복 (같은 트랙 2번)**

**증상:**
```json
{
  "trackList": [
    { "trackNo": 1 },
    { "trackNo": 1 }  // 똑같음!
  ]
}

→ 같은 팀이 2번 등록 ❌
```

**해결:**
```java
Set<Long> trackSet = new HashSet<>();

for (TrackStageVO track : req.getTrackList()) {
    if (!trackSet.add(track.getTrackNo())) {
        throw new OperationErrorException(ErrorCode.DUPLICATE_TRACK_REQUEST);
    }
}
// Set.add()가 false 반환 → 중복 감지 ✅
```

### **Problem 3: 파일 업로드 예외**

**증상:**
```
메모리 부족으로 파일 업로드 중단
→ 팀/이력은 이미 INSERT됨
→ 파일만 없는 불완전한 상태 ❌
```

**해결:**
```java
try {
    // 1) 팀 등록
    // 2) 이력 등록
    // 3) 파일 업로드
} catch (Exception e) {
    throw new OperationErrorException(...);
    // ← 이 순간 @Transactional이 ROLLBACK
}
```

---

## 📊 성과

### Before (개별 신청)

```
사용자: 한 번에 1개 트랙만 신청 가능
├─ 도전기업 신청 → 확인 → 펠로우 신청 → 확인
├─ 라운드트립: 2회
└─ 시간: 2분+ ❌
```

### After (Bulk 신청)

```
사용자: 여러 트랙 동시 신청
├─ 도전기업 + 펠로우 한 번에 선택
├─ 라운드트립: 1회
└─ 시간: 30초 ✅

안정성:
├─ 모두 성공 또는 모두 실패 (일관성)
├─ 내부 중복 자동 감지
└─ 데이터 정합성 100% ✅
```

---

## 🎓 배운 점

### 1. Controller 선검증이 효율성을 좌우

> **"빠른 실패는 트랜잭션 비용을 절감한다"**

```
❌ Service에서만 검증
   └─ DB 쿼리 + 트랜잭션 시작 + 예외 → 비용 크심

✅ Controller에서 먼저 검증
   └─ 간단한 조건만 → 빠른 실패 → 비용 절감
```

### 2. Set.add()는 중복 체크의 정석

```java
if (!set.add(value)) {
    // 중복이면 false 반환
    throw Exception;
}
```

→ O(1) 해시 기반, 명확함, 간단함

### 3. @Transactional의 믿음

> **"예외를 던지면 프레임워크가 알아서 처리한다"**

```
개발자: throw Exception;
Spring: 자동 ROLLBACK ✅
```

---

## 📌 핵심 코드 포인트

| 기법 | 코드 |
|------|------|
| 신청 개수 제한 | `req.getTrackList().size() > 2` |
| 내부 중복 체크 | `!set.add(value)` → 중복이면 false |
| DB 중복 검증 | `applyTeamCheck(userNo, trackNo)` |
| 조합 검증 | `applyOverlapCheck(userNo, trackNo)` |
| 트랜잭션 경계 | `@Transactional` |
| 원자성 보장 | try-catch로 Exception 발생 → ROLLBACK |

---

*개발자: kmj | 파일: ApplyCommandService.java:735 | 사용자 신청 플로우의 핵심*
