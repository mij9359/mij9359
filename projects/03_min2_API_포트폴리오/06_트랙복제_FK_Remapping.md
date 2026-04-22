# 🔀 트랙 복제: FK Remapping 기반 2-Phase Clone

> **스테이지 그룹 + 스테이지를 구조 그대로 복제하면서 FK를 정확히 재매핑하는 원자적 트랜잭션**

![Status](https://img.shields.io/badge/Status-Production-brightgreen)
![Pattern](https://img.shields.io/badge/Pattern-2_Phase_Clone-blue)
![DataIntegrity](https://img.shields.io/badge/DataIntegrity-FK_Remapping-green)

---

## 🎯 문제 정의

### 상황: 매년 새 경진대회 시작

```
작년(2025) 경진대회 설정:
├─ 트랙 "도전기업"
│  ├─ 서류심사 그룹
│  │  ├─ Stage: 서류 접수 (2025-01-01 ~ 02-15)
│  │  ├─ Stage: 서류 평가 (2025-02-16 ~ 03-01)
│  │  └─ Stage: 발표 신청 (2025-03-02 ~ 03-31)
│  │
│  ├─ 발표심사 그룹
│  │  ├─ Stage: 발표 평가 (2025-04-01 ~ 04-30)
│  │  └─ Stage: 최종 심사 (2025-05-01 ~ 05-31)

올해(2026): "이 구조를 그대로 복사하되 데이터는 분리"
```

### 복잡성: FK 재매핑

```
DB 스키마:

stage_group (스테이지 그룹)
├─ sg_no (PK)          ← 이게 바뀐다!
├─ track_no (FK)

stage (스테이지)
├─ st_no (PK)
├─ track_no (FK)
└─ sg_no (FK) ⚠️ 여기서 참조!

문제:
작년 sg_no = 1, 2
올해 복제 시 신규 sg_no = 3, 4

Stage 복제할 때
├─ 원본: sg_no = 1 (FK 참조)
└─ 복제본: sg_no = 3 (신규로 바뀌어야 함)

하지만 SQL INSERT는 sg_no를 자동 증가로 반환받아야 알 수 있음!
```

### 위험한 상황들

```
❌ 위험 1: sg_no를 그대로 복사
   └─ 여러 트랙이 동일 그룹을 공유 → 한 트랙 수정 시 다른 트랙도 영향

❌ 위험 2: 중간에 실패 (Stage 복제 실패)
   └─ 트랙만 생기고 Stage는 반쪽만
   └─ "좀비" 데이터 ❌

❌ 위험 3: Self-Copy (같은 트랙으로 복제)
   └─ 자기 참조 순환 구조
   └─ 데이터 무결성 파괴 ❌

❌ 위험 4: 타겟에 이미 Stage가 있는데 덮어쓰기
   └─ 기존 심사 데이터 손실
```

---

## 💡 솔루션: 2-Phase Clone + FK Remapping

```
┌────────────────────────────────────────┐
│ @Transactional 시작                    │
├────────────────────────────────────────┤
│                                        │
│ Phase 1: StageGroup 복제 + 매핑        │
│ ├─ sg_no 매핑 테이블 생성              │
│ └─ old_sgNo → new_sgNo 저장            │
│                                        │
│ Phase 2: Stage 복제 + 매핑 적용        │
│ └─ sg_no를 매핑 테이블로 치환           │
│                                        │
├────────────────────────────────────────┤
│ 모든 INSERT 성공 → COMMIT              │
│ 하나라도 실패 → ROLLBACK (좀비 방지)    │
└────────────────────────────────────────┘
```

---

## 🔥 핵심 구현 코드

```java
@Transactional
public void insertTrack(InsertTrackReqVO insertTrackReqVO) {
    // 1. 트랙 본체 INSERT
    TrackVO trackVO = TrackVO.builder()
        .trackYear(insertTrackReqVO.getTrackYear())
        .trackName(insertTrackReqVO.getTrackName())
        .trackStatus(0)
        .operatorNo(insertTrackReqVO.getOperatorNo())
        .build();
    
    int trackInsertResult = stageCommandMapper.insertTrack(trackVO);
    if (trackInsertResult < 1) {
        throw new OperationErrorException(ErrorCode.FAILED_TO_INSERT_TRACK);
    }
    
    Long targetTrackNo = trackVO.getTrackNo(); // MyBatis useGeneratedKeys
    
    // 2. 복제 요청이 있으면 Stage 구조 복제
    if (insertTrackReqVO.getCloneTrackNo() != null) {
        cloneTrackStages(insertTrackReqVO.getCloneTrackNo(), 
                        targetTrackNo, 
                        insertTrackReqVO.getOperatorNo());
    }
    
    // 트랜잭션 자동 COMMIT (성공)
}

private void cloneTrackStages(Long cloneTrackNo, Long targetTrackNo, Long operatorNo) {
    
    // ──────────────────────────────────────────
    // [검증 1] Self-Copy 차단
    // ──────────────────────────────────────────
    if (cloneTrackNo.equals(targetTrackNo)) {
        throw new OperationErrorException(ErrorCode.FAILED_TO_INSERT_TRACK,
            "원본 트랙과 대상 트랙이 같을 수 없습니다");
    }
    
    // ──────────────────────────────────────────
    // [검증 2] 원본 트랙 존재 확인
    // ──────────────────────────────────────────
    if (stageQueryMapper.checkTrackExists(cloneTrackNo) < 1) {
        throw new OperationErrorException(ErrorCode.FAILED_TO_INSERT_TRACK,
            "원본 트랙이 존재하지 않습니다");
    }
    
    // ──────────────────────────────────────────
    // [검증 3] 원본에 Stage 그룹 존재
    // ──────────────────────────────────────────
    if (stageQueryMapper.countStageGroupByTrackNo(cloneTrackNo) < 1) {
        throw new OperationErrorException(ErrorCode.FAILED_TO_INSERT_TRACK,
            "원본 트랙에 스테이지 그룹이 없습니다");
    }
    
    // ──────────────────────────────────────────
    // [검증 4] 타겟 트랙이 비어있어야만 복제 가능
    // ──────────────────────────────────────────
    if (stageQueryMapper.countStageByTrackNo(targetTrackNo) > 0) {
        throw new OperationErrorException(ErrorCode.FAILED_TO_INSERT_TRACK,
            "대상 트랙에 이미 스테이지가 존재합니다");
    }
    
    // ──────────────────────────────────────────
    // [Phase 1] StageGroup 복제 + FK 매핑
    // ──────────────────────────────────────────
    
    // 원본 그룹 조회
    List<StageGroupVO> sourceGroups = 
        stageQueryMapper.selectStageGroupListForClone(cloneTrackNo);
    
    // old_sgNo → new_sgNo 매핑 테이블
    Map<Long, Long> sgNoMap = new HashMap<>(sourceGroups.size());
    
    for (StageGroupVO sourceGroup : sourceGroups) {
        Long oldSgNo = sourceGroup.getSgNo();
        
        // 원본 그룹 정보 복사
        StageGroupVO insertVO = StageGroupVO.builder()
            .trackNo(targetTrackNo)         // ← 새 트랙
            .sgMultiActivation(sourceGroup.getSgMultiActivation())
            .sgTitle(sourceGroup.getSgTitle())
            .sgStartDate(sourceGroup.getSgStartDate())
            .sgEndDate(sourceGroup.getSgEndDate())
            .sgState(0)
            .operatorNo(operatorNo)
            .createdAt(LocalDateTime.now())
            .updatedAt(LocalDateTime.now())
            .build();
        
        // INSERT 실행 (신규 sg_no 자동 생성)
        int result = stageCommandMapper.cloneInsertStageGroup(insertVO);
        if (result < 1) {
            throw new OperationErrorException(ErrorCode.FAILED_TO_INSERT_TRACK,
                "Stage Group 복제 중 오류 발생");
        }
        
        // 🔑 MyBatis useGeneratedKeys로 신규 sg_no 회수
        Long newSgNo = insertVO.getSgNo();
        if (newSgNo == null) {
            throw new OperationErrorException(ErrorCode.FAILED_TO_INSERT_TRACK,
                "생성된 Stage Group의 ID를 알 수 없습니다");
        }
        
        // 매핑 저장
        sgNoMap.put(oldSgNo, newSgNo);
        log.info("[Stage Clone] sg_no 매핑: {} → {}", oldSgNo, newSgNo);
    }
    
    // ──────────────────────────────────────────
    // [Phase 2] Stage 복제 + FK 치환
    // ──────────────────────────────────────────
    
    // 원본 Stage 조회
    List<StageVO> sourceStages = 
        stageQueryMapper.selectStageListForClone(cloneTrackNo);
    
    for (StageVO sourceStage : sourceStages) {
        
        // 🔑 원본의 sg_no를 매핑으로 치환
        Long newSgNo = sgNoMap.get(sourceStage.getSgNo());
        if (newSgNo == null) {
            throw new OperationErrorException(ErrorCode.FAILED_TO_INSERT_TRACK,
                "Stage의 Group 매핑을 찾을 수 없습니다: " + sourceStage.getSgNo());
        }
        
        // Stage 정보 복사 (sg_no는 신규로)
        StageVO insertVO = StageVO.builder()
            .trackNo(targetTrackNo)         // ← 새 트랙
            .sgNo(newSgNo)                  // ← 신규 Group ID!
            .stNo(sourceStage.getStNo())
            .stageTitle(sourceStage.getStageTitle())
            .stageContent(sourceStage.getStageContent())
            .stageOrder(sourceStage.getStageOrder())
            .stagePointType(sourceStage.getStagePointType())
            .stageStartDate(sourceStage.getStageStartDate())
            .stageEndDate(sourceStage.getStageEndDate())
            .stageState(0)
            .operatorNo(operatorNo)
            .createdAt(LocalDateTime.now())
            .updatedAt(LocalDateTime.now())
            .build();
        
        // INSERT 실행
        int result = stageCommandMapper.cloneInsertStage(insertVO);
        if (result < 1) {
            throw new OperationErrorException(ErrorCode.FAILED_TO_INSERT_TRACK,
                "Stage 복제 중 오류 발생");
        }
        
        log.info("[Stage Clone] Stage 복제: {} → {}", 
                 sourceStage.getStNo(), insertVO.getStNo());
    }
    
    log.info("[Track Clone] 트랙 {} ← {} 복제 완료", targetTrackNo, cloneTrackNo);
}
```

### MyBatis XML 설정

```xml
<!-- useGeneratedKeys로 신규 PK를 자동 회수 -->
<insert id="cloneInsertStageGroup" 
        parameterType="com.example.StageGroupVO"
        useGeneratedKeys="true"
        keyProperty="sgNo"
        keyColumn="sg_no">
    
    INSERT INTO stage_group 
        (track_no, sg_multi_activation, sg_title, sg_start_date, sg_end_date, 
         sg_state, operator_no, created_at, updated_at)
    VALUES
        (#{trackNo}, #{sgMultiActivation}, #{sgTitle}, #{sgStartDate}, #{sgEndDate},
         #{sgState}, #{operatorNo}, #{createdAt}, #{updatedAt})
</insert>

<insert id="cloneInsertStage"
        parameterType="com.example.StageVO"
        useGeneratedKeys="true"
        keyProperty="stNo">
    
    INSERT INTO stage
        (track_no, sg_no, st_no, stage_title, stage_content, stage_order,
         stage_point_type, stage_start_date, stage_end_date, stage_state,
         operator_no, created_at, updated_at)
    VALUES
        (#{trackNo}, #{sgNo}, #{stNo}, #{stageTitle}, #{stageContent}, #{stageOrder},
         #{stagePointType}, #{stageStartDate}, #{stageEndDate}, #{stageState},
         #{operatorNo}, #{createdAt}, #{updatedAt})
</insert>
```

---

## ⚠️ 마주친 문제점 & 해결

### **Problem 1: sg_no를 모르면 Stage를 만들 수 없음**

**증상:**
```java
// 원본 sg_no = 1
StageGroupVO group = insert(...);  // sg_no 얼마?

// Stage 만들어야 하는데
stageInsert(..., sgNo = 1);  // ← 원본이랑 같다! ❌
```

**해결:**
```xml
<!-- useGeneratedKeys="true" keyProperty="sgNo" -->
INSERT ... VALUES (...)
```

```java
// INSERT 후 자동으로 insertVO.sgNo에 신규 값이 들어감
int result = mapper.cloneInsertStageGroup(insertVO);
Long newSgNo = insertVO.getSgNo();  // ← 이제 신규 값!
```

### **Problem 2: 중간 실패 시 좀비 데이터**

**증상:**
```
StageGroup 5개 생성 성공
→ Stage 생성 중 3번째에서 실패
→ 새 트랙 + 5개 Group은 남음 (orphan data)
```

**해결:**
```java
@Transactional  // ← 모든 INSERT을 한 경계 안에서
```

실패 시:
```
Exception 발생
→ @Transactional 롤백
→ 트랙도, Group도, Stage도 모두 INSERT 전으로
→ DB 일관성 유지 ✅
```

### **Problem 3: 검증 부족 → 실수 연쇄**

**증상:**
```
A 트랙을 B 트랙으로 복제 (실수)
→ A와 B가 중복 구조
→ A 수정하면 B도 영향
```

**해결:**
```java
// 4단계 검증
if (cloneTrackNo.equals(targetTrackNo)) ❌  // Self-copy
if (checkExists(cloneTrackNo) < 1) ❌       // 원본 없음
if (countGroup(cloneTrackNo) < 1) ❌        // 원본에 Group 없음
if (countStage(targetTrackNo) > 0) ❌       // 타겟이 비어있지 않음
```

---

## 📊 성과

### Before (수작업)

```
작년 Stage 구조 재현
├─ 메모장에 적어놓기 (sg_no, stage 내용)
├─ 운영팀이 직접 INSERT
├─ 실수: sg_no 중복, 필드 누락
└─ 소요: 30분 ~ 1시간 ❌
```

### After (자동화)

```
"원본 트랙 선택" + 클릭
├─ 자동 2-Phase Clone
├─ FK Remapping 자동 수행
├─ 검증 4단계 자동 통과
└─ 소요: 1초 ✅
```

### 효과

```
✅ 운영팀 반복 작업 제거
✅ 데이터 오류 가능성 원천 차단
✅ FK 무결성 100% 보장
✅ 재사용 가능한 코드 (매년 반복 사용)
```

---

## 🎓 배운 점

### 1. FK가 있으면 2-Phase는 필수

> **"Parent를 먼저 만들고 PK를 알아야 Child를 만들 수 있다"**

- Phase 1: Parent (StageGroup) 복제 + 매핑 수집
- Phase 2: Child (Stage) 복제 + 매핑 적용

### 2. MyBatis useGeneratedKeys의 중요성

```xml
<insert ... useGeneratedKeys="true" keyProperty="sgNo">
```

→ INSERT 후 자동 증가 PK를 VO에 주입받아 매핑 가능!

### 3. 사전 검증이 오류를 예방

```
4단계 검증으로:
- Self-copy 차단
- 원본 존재 확인
- 원본에 데이터 있는지 확인
- 타겟이 비었는지 확인
→ 뒤의 오류들을 사전에 방지
```

### 4. @Transactional은 데이터 일관성의 보험

→ 좀비 데이터를 원천적으로 방지!

---

## 📌 핵심 코드 포인트

| 기법 | 코드 |
|------|------|
| Self-copy 차단 | `if (cloneTrackNo.equals(targetTrackNo))` |
| PK 회수 | `useGeneratedKeys="true" keyProperty="sgNo"` |
| FK 매핑 | `Map<oldSgNo, newSgNo>` |
| Phase 분리 | StageGroup 먼저, Stage는 나중에 |
| 원자성 보장 | `@Transactional` + 사전 검증 |

---

*개발자: min2 | 파일: StageCommandService.java:82~191 | 매년 반복 운영 자동화*
