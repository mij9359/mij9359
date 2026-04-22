# 📊 점수 표준화 시스템 - 권역별 보정계수 기반 최종 등수 결정

> **심사위원 편차를 보정하고, 공정한 최종 등수를 산출하는 정규분포 기반 표준화 시스템**

![Status](https://img.shields.io/badge/Status-Completed-brightgreen)
![Java](https://img.shields.io/badge/Java-17-orange)
![Mathematics](https://img.shields.io/badge/Z--Score-Statistics-blue)
![Excel](https://img.shields.io/badge/Export-Apache_POI-brightgreen)

---

## 🎯 프로젝트 개요

### 문제 상황

**U300 창업지원 심사**에서 6개 권역(분과)별로 심사위원이 다르게 배치되었을 때:

- 🔴 **A권역**: 심사위원들이 점수를 짜게 부여 (평균: 55점)
- 🟡 **B권역**: 심사위원들이 후하게 부여 (평균: 70점)

**같은 실력인데도** A권역 팀은 불리하고, B권역 팀은 유리한 상황 발생!

### 솔루션

**권역별 보정계수**를 계산해서:
1. 각 권역의 평가 기준을 통일 (평탄화)
2. 그 다음에 정규분포 기반 표준화(Z-Score) 적용
3. **공정한 최종 등수** 결정

**결과:** 300개 팀의 최종 등수가 자동으로 산출되고 Excel로 다운로드됨

---

## 🏗️ 표준화 알고리즘

### 5단계 보정 프로세스

```
[Step 1] 보정계수 산출
├─ 전체 평균 구하기
├─ 권역별 평균 구하기
└─ 보정계수 = 전체평균 / 권역별평균
   
   예) 전체평균 62.35, A권역평균 55.00
   → A권역 보정계수 = 62.35 / 55.00 = 1.134
   
[Step 2] 보정원점수 계산
├─ 각 팀의 원점수 × 보정계수
└─ 결과: 모든 권역의 평균이 '전체평균'으로 동일하게 정렬됨
   
   예) A권역 팀의 원점수 55점
   → 보정원점수 = 55 × 1.134 = 62.37점
   (이제 B권역과 비교 가능한 점수!)

[Step 3] 보정 전체 통계 도출
├─ 보정된 모든 팀의 점수로부터
├─ 새로운 전체평균 도출
└─ 새로운 표준편차 도출

[Step 4] Z-Score (표준화) 계산
└─ 최종점수 = (보정원점수 - 보정전체평균) / 보정전체표준편차
   
   예) (62.37 - 62.35) / 5.75 = 0.0035
   (표준정규분포에서의 위치)

[Step 5] 최종 등수 결정
├─ 최종점수 기준 내림차순 정렬
├─ 동점자는 동일 등수 부여 (예: 3,3,3,4 방식)
└─ 최종 등수 반영
```

---

## 📐 수학적 배경

### 보정계수 (Correction Factor / Scale Factor)

**의미:** "이 권역의 심사위원들이 전체보다 얼마나 짜/후했는가?"를 배수로 표현

**공식:**
```
보정계수 = 전체 평균 / 권역별 평균
```

**효과:**
- 보정계수 > 1.0 → 점수를 짜게 준 권역 (점수 상향)
- 보정계수 = 1.0 → 평균 수준의 권역 (변화 없음)
- 보정계수 < 1.0 → 점수를 후하게 준 권역 (점수 하향)

### Z-Score (표준화 점수)

**의미:** "평균으로부터 표준편차의 몇 배만큼 떨어져 있는가?"

**공식:**
```
Z = (X - μ) / σ

X: 보정원점수
μ: 보정 전체 평균
σ: 보정 전체 표준편차
```

**해석:**
- Z > 0 → 평균 이상
- Z > 2 → 상위 2.3% (최상위권)
- Z < -2 → 하위 2.3% (최하위권)

---

## 💻 구현 아키텍처

### 데이터 흐름

```
DB (점수 데이터)
   ↓
[Pass 1] 그룹 통계 및 1차 보정
   ├─ 전체 평균 계산 (모든 팀)
   ├─ 권역별 평균 계산 (GROUP BY 권역)
   ├─ 보정계수 산출 (6개)
   └─ 보정원점수 = 원점수 × 보정계수
   ↓
[Pass 2] 최종 통계 및 표준화
   ├─ 보정원점수 리스트 (예: 300팀)
   ├─ 보정전체 평균 재계산
   ├─ 보정전체 표준편차 재계산
   └─ 최종점수(Z-Score) = (보정원 - 보정평균) / 보정표준편차
   ↓
[Pass 3] 등수 결정
   ├─ 최종점수 내림차순 정렬
   ├─ 동점자 동일 등수
   └─ DB 저장 (evaluation_summary)
   ↓
Excel 다운로드 (보정_점수_결과_yyyyMMdd.xlsx)
```

---

## 🗄️ 데이터베이스 설계

### 필요한 테이블

#### 1) point_result (평가 결과 기본)

```sql
CREATE TABLE point_result (
    pr_no BIGINT PRIMARY KEY COMMENT '평가결과 ID',
    team_no BIGINT COMMENT '팀 ID',
    track_no BIGINT COMMENT '트랙 ID',
    stage_no BIGINT COMMENT '스테이지 ID',
    state INT COMMENT '0: 정상, 1: 삭제'
);
```

#### 2) point_result_detail (세부 점수)

```sql
CREATE TABLE point_result_detail (
    prd_no BIGINT PRIMARY KEY,
    pr_no BIGINT COMMENT 'point_result FK',
    prd_score DECIMAL(5,2) COMMENT '개별 심사위원 점수',
    state INT
);
```

#### 3) point_group (권역/분과)

```sql
CREATE TABLE point_group (
    pg_no BIGINT PRIMARY KEY COMMENT '권역 ID',
    pg_name VARCHAR(100) COMMENT '권역명 (예: "수도권1(서울)")',
    pg_pointer_no INT COMMENT '만점 (예: 10, 100)',
    track_no BIGINT,
    stage_no BIGINT,
    state INT
);
```

#### 4) evaluation_summary (최종 결과 저장)

```sql
CREATE TABLE evaluation_summary (
    es_no BIGINT PRIMARY KEY AUTO_INCREMENT,
    team_no BIGINT COMMENT '팀 ID',
    track_no BIGINT,
    stage_no BIGINT,
    
    -- [원본 점수]
    raw_score_avg DECIMAL(10,4) COMMENT '원점수 평균',
    raw_score_sum DECIMAL(10,2) COMMENT '원점수 합계',
    
    -- [보정된 점수]
    regional_avg DECIMAL(10,4) COMMENT '권역별 평균',
    correction_factor DECIMAL(10,4) COMMENT '보정계수',
    adj_raw_score DECIMAL(10,4) COMMENT '보정원점수',
    
    -- [최종 점수]
    adj_overall_avg DECIMAL(10,4) COMMENT '보정전체평균',
    adj_overall_std_dev DECIMAL(10,4) COMMENT '보정전체표준편차',
    final_z_score DECIMAL(10,4) COMMENT 'Z-점수 (최종점수)',
    
    -- [등수]
    final_rank INT COMMENT '최종 등수',
    result_score DECIMAL(10,4) COMMENT '결과점수 (가점/감점 포함)',
    
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY unique_team_track_stage (team_no, track_no, stage_no)
);
```

---

## 📊 실제 결과 예시

### Excel 산출물 구조 (3개 섹션)

#### **섹션 1: 전체 통계 요약**

| 항목 | 값 |
|------|-----|
| 전체 원점수 평균 | 62.3542 |
| 보정 전체 평균 | 62.3542 |
| 보정 전체 표준편차 | 5.7489 |

*(주: 보정 후에는 평균이 동일하게 맞춰짐)*

#### **섹션 2: 권역별 보정 요약**

| 구분 | 평균 | 보정계수 |
|------|------|---------|
| 수도권1(서울) | 61.24 | 1.018 |
| 수도권2(경기,인천) | 63.45 | 0.982 |
| 충청권 | 62.10 | 1.002 |
| 호남제주권 | 60.85 | 1.024 |
| 대경강원권 | 62.89 | 0.991 |
| 동남권 | 62.56 | 0.996 |

*(예시 데이터)*

#### **섹션 3: 상세 팀 리스트**

| 팀명 | 원점수 | 권역 | 보정계수 | 보정원점수 | Z-점수 | 최종등수 |
|------|--------|------|---------|-----------|--------|---------|
| 팀A | 65.00 | 수도권1 | 1.018 | 66.17 | 0.641 | 45 |
| 팀B | 58.00 | 호남제주 | 1.024 | 59.39 | -0.168 | 238 |
| 팀C | 72.00 | 수도권2 | 0.982 | 70.70 | 1.452 | 18 |

*(실제 데이터는 개인정보이므로 마스킹됨)*

---

## 🔥 핵심 구현 로직 (Java)

### 1단계: 그룹 통계 계산

```java
// [Step 1] 순수 원점수 추출 (가점/감점 제외)
Function<AssignedPointListResVO, Double> getPureScore = t -> {
    String s = t.getScoreAvg();
    if (s == null || "평가 면제".equals(s)) return 0.0;
    
    double scoreAvg = Double.parseDouble(s);
    double bonus = (t.getPtAddition() != null) ? t.getPtAddition() : 0.0;
    double penalty = (t.getPtAdditionMinus() != null) ? t.getPtAdditionMinus() : 0.0;
    
    return scoreAvg - bonus + penalty; // 순수 원점수
};

// [Step 2] 통계 대상: 순수 점수 > 0인 팀만
List<AssignedPointListResVO> scoredTeams = teams.stream()
    .filter(t -> getPureScore.apply(t) > 0)
    .collect(Collectors.toList());

// [Step 3] 전체 평균 & 권역별 평균
double overallRawAvg = scoredTeams.stream()
    .mapToDouble(getPureScore::apply)
    .average().orElse(0.0);

Map<Long, Double> regionalAvgMap = scoredTeams.stream()
    .filter(t -> t.getPgNo() != null)
    .collect(Collectors.groupingBy(
        AssignedPointListResVO::getPgNo,
        Collectors.averagingDouble(getPureScore::apply)
    ));
```

### 2단계: 보정원점수 계산

```java
// [Step 4] 보정계수 산출 + 보정원점수
for (AssignedPointListResVO team : teams) {
    double pureScore = getPureScore.apply(team);
    
    if (pureScore > 0) {
        double regAvg = regionalAvgMap.getOrDefault(team.getPgNo(), overallRawAvg);
        double factor = (regAvg <= 0) ? 1.0 : overallRawAvg / regAvg;
        
        team.setRegionalAvg(roundToFour(regAvg));
        team.setCorrectionFactor(roundToFour(factor));
        team.setAdjRawScore(roundToFour(pureScore * factor)); // ← 핵심
    }
}
```

### 3단계: 최종 표준화

```java
// [Step 5] 보정 전체 평균 & 표준편차
double adjOverallAvg = scoredTeams.stream()
    .mapToDouble(AssignedPointListResVO::getAdjRawScore)
    .average().orElse(0.0);

double adjOverallStdDev = Math.sqrt(
    scoredTeams.stream()
        .mapToDouble(t -> Math.pow(t.getAdjRawScore() - adjOverallAvg, 2))
        .sum() / scoredTeams.size()
);

// [Step 6] Z-Score 최종 계산
for (AssignedPointListResVO team : teams) {
    double displayScore = team.getAdjRawScore() + bonus - penalty;
    team.setResultScore(roundToFour(displayScore));
    
    if (getPureScore.apply(team) > 0 && adjOverallStdDev > 0) {
        double zScore = (displayScore - adjOverallAvg) / adjOverallStdDev;
        team.setFinalZScore(roundToFour(zScore)); // ← 최종점수
    }
}
```

### 4단계: 등수 결정 (동점자 처리)

```java
// [Step 7] 마스터 정렬: 면제팀 → 결과점수 → Z-점수
Comparator<AssignedPointListResVO> masterRankSort = Comparator
    .comparing(isExempt::test, Comparator.reverseOrder())
    .thenComparing(AssignedPointListResVO::getResultScore, Comparator.reverseOrder())
    .thenComparing(AssignedPointListResVO::getFinalZScore, Comparator.reverseOrder());

teams.sort(masterRankSort);

// [Step 8] 등수 부여: 3명 동점 → 3,3,3,4 방식
int i = 0, currentRankBase = 1;
while (i < teams.size()) {
    int j = i;
    
    // 동점 그룹 찾기
    while (j < teams.size() && isSameGroup(teams.get(i), teams.get(j))) {
        j++;
    }
    
    // 그룹 크기만큼 등수 할당
    int groupSize = j - i;
    int calculatedRank = currentRankBase + groupSize - 1;
    
    for (int k = i; k < j; k++) {
        teams.get(k).setFinalRank(calculatedRank);
    }
    
    i = j;
    currentRankBase += groupSize;
}
```

---

## 📥 API 명세

### 조회 API

```
GET /api/admin/evaluation/before-assigned-point-list

Request Parameters:
  - trackNo (Long): 트랙 ID
  - stageNo (Long): 스테이지 ID
  - sortType (String): 정렬 필드
  - sortOrder (String): ASC/DESC

Response:
{
  "code": 200,
  "data": [
    {
      "teamNo": 123,
      "teamName": "팀A",
      "rawScoreAvg": 65.00,
      "correctionFactor": 1.018,
      "adjRawScore": 66.17,
      "overallAvg": 62.35,
      "adjOverallStdDev": 5.75,
      "finalZScore": 0.641,
      "finalRank": 45
    },
    ...
  ]
}
```

### Excel 다운로드 API

```
GET /api/admin/evaluation/before-assigned-point-list-excel

Request Parameters:
  (조회 API와 동일)

Response:
  Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
  Content-Disposition: attachment; filename="보정_점수_결과_20260421.xlsx"
  
  [파일 스트림]
```

---

## ✅ 구현 현황

### 완료된 기능

- ✅ 권역별 보정계수 산출
- ✅ 보정원점수 계산
- ✅ 보정 전체 평균 & 표준편차 도출
- ✅ Z-Score 표준화
- ✅ 최종 등수 결정 (동점자 처리 포함)
- ✅ 조회 API (JSON)
- ✅ Excel 다운로드 API
- ✅ 300개 팀 대량 처리

### 마주친 이슈 & 해결

#### 이슈 1: 가점/감점과 보정계수의 혼동

**문제:**
- 가점만 있는 팀(예: guide팀)이 보정계수 계산에 영향을 미쳐서 왜곡

**해결:**
```java
// 순수 원점수 = scoreAvg - 가점 + 감점
// 보정계수는 순수 원점수로만 계산
double pureScore = scoreAvg - bonus + penalty;
```

#### 이슈 2: 표준편차가 0일 때 (모든 팀이 같은 점수)

**문제:**
```
Z = (X - μ) / 0 → 무한대!
```

**해결:**
```java
if (adjOverallStdDev > 0) {
    zScore = (score - avgScore) / adjOverallStdDev;
} else {
    zScore = 0.0; // 모두 동일 점수면 Z=0
}
```

#### 이슈 3: 면제팀 처리

**문제:**
- 면제팀(예: "평가 면제")은 보정계수 계산에서 제외되어야 함
- 하지만 최종 등수에서는 최상단에 배치

**해결:**
```java
// 통계 대상에서 제외
List<AssignedPointListResVO> scoredTeams = teams.stream()
    .filter(t -> getPureScore.apply(t) > 0)
    .collect(Collectors.toList());

// 정렬 시 면제팀 최상단
Comparator.comparing(isExempt::test, Comparator.reverseOrder())
    .thenComparing(resultScore)
```

---

## 📈 성과

### Before (수작업)

| 작업 | 소요 시간 |
|------|---------|
| 권역별 평균 계산 | 2~3시간 |
| 보정계수 수작업 | 1~2시간 |
| 300팀 수작업 보정 | 불가능 ❌ |
| 등수 계산 | 2~3시간 |
| Excel 작성 | 1~2시간 |
| **총 소요 시간** | **6~10시간** |

### After (자동화)

| 작업 | 소요 시간 |
|------|---------|
| API 호출 | < 1초 |
| 자동 계산 | < 1초 |
| Excel 생성 | < 1초 |
| **총 소요 시간** | **< 3초** ✨ |

**단축 효과: 99.99% 시간 절감!**

---

## 🎓 배운 점

### 1. 통계학의 실무 적용

- **Z-Score 표준화**의 실제 의미 이해
- 평균과 표준편차의 중요성
- 정규분포 기반의 공정성 확보

### 2. 데이터 정합성 관리

- 가점/감점 vs 순수 점수의 구분
- 면제팀 vs 평가 대상의 분리 처리
- 단계별 검증의 필수성

### 3. 대용량 데이터 처리

- 300개 팀의 효율적인 메모리 관리
- Stream API를 활용한 함수형 프로그래밍
- 두 번의 Pass(통과)로 정확성 확보

### 4. 엑셀 자동 생성

- Apache POI를 활용한 복합 레이아웃
- 3개 섹션(요약+권역+상세)의 통합
- 숨겨진 버그: 스타일 적용 시 Merge와의 상호작용

---

## 💡 핵심 인사이트

> **"통계학은 데이터의 공정성을 확보하는 백엔드 개발자의 무기다"**

이 프로젝트가 풀어야 할 핵심 문제:

1. **심사위원별 편차 제거** (보정계수)
2. **수평적 비교 가능성 확보** (표준화)
3. **동점자 공정 처리** (등수 결정)

단순한 점수 순서 매기기가 아니라, **수학과 도메인 이해를 결합한 공정한 평가 시스템**을 만드는 것.

---

## 🚀 앞으로의 개선 방향

- [ ] 신뢰도 구간(Confidence Interval) 추가
- [ ] 이상치(Outlier) 자동 감지 및 처리
- [ ] 가중치 기반 보정계수 (복수 평가항목)
- [ ] 실시간 스트리밍 계산 (대규모 데이터)
- [ ] 머신러닝 기반 예측 등수

---

## 📞 Contact

이 프로젝트는 **U300 온라인 창업 플랫폼**의 일부입니다.

- **전체 플랫폼**: 지원자 심사 → 교육 → 평가 → 투자 연계
- **담당 모듈**: 평가 결과의 표준화 및 등수 결정
- **규모**: 연간 300~400명 대규모 데이터 처리

더 많은 프로젝트 정보는 포트폴리오를 참고해주세요!

---

*프로젝트 기간: 2026년 4월 현재 운영 중*  
*담당자: Backend Engineer (Java/Spring Boot)*  
*최종 결과: 300개 팀의 표준화된 최종 등수 산출*
