# 📅 U300 온라인 창업 플랫폼 - 자동 시간표 생성 시스템

> **300개 청년 창업팀의 발표심사 일정을 자동 최적화하는 시간표 관리 백엔드**

![Status](https://img.shields.io/badge/Status-Completed-brightgreen)
![Java](https://img.shields.io/badge/Java-17-orange)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.x-brightgreen)
![MariaDB](https://img.shields.io/badge/MariaDB-10.x-blue)

---

## 🎯 프로젝트 개요

### 문제 정의

한국청년기업가정신재단의 **U300 플랫폼**에서 연간 **8,000명 지원자를 3단계 심사**로 거르는데, 최종 3차 심사에서 **300개 팀을 4~5일(월~금) 동안 오프라인 발표심사**해야 합니다.

**발생한 문제:**
- 300개 팀을 수작업으로 일정 배치 → **휴먼에러 발생**
- 심사위원 이동 시간, 점심 시간, 휴식 시간을 **일일이 계산**해야 함
- 일정이 바뀌면 **전체 Excel을 다시 작성**해야 함
- 최종 일정을 모든 심사위원·운영진에게 배포할 때 **버전 관리 어려움**

**솔루션:**
운영진이 **시간표 설정만 입력하면, 시스템이 자동으로 300개 팀 일정을 최적화**하여 **Excel로 다운로드** 가능하도록 구현.

---

## 🏗️ 시스템 아키텍처

### 사용자 플로우

```
[1단계] 운영지원팀
   ↓
   └─→ 시간표 설정 정보 입력 & 저장
       (제목, 장소, 시작일, 진행요일, 시간, 팀당 발표시간, 휴식 규칙 등)

[2단계] 평가관리팀
   ↓
   └─→ [평가순서 변경] 화면에서 저장된 시간표 목록 중 선택
       └─→ 상세 화면 진입

[3단계] 시간표 생성 & 검증
   ↓
   ├─→ 설정된 팀 수 vs 실제 트랙 팀 수 일치 확인
   ├─→ 일치 ✅ → [시간표 생성] 버튼 활성화
   └─→ 불일치 ❌ → 경고 메시지 + 버튼 비활성화

[4단계] Excel 다운로드
   ↓
   └─→ 생성된 시간표 + 팀 정보로 Excel 자동 생성
       └─→ 심사위원·운영진에게 배포
```

---

## 📊 기술 설계

### 시간표 생성 알고리즘

#### 설정 파라미터

| 항목 | 값 | 설명 |
|------|-----|------|
| **팀당 발표시간** | 8분 | 각 팀 프레젠테이션 시간 |
| **전환시간** | 2분 | 팀 교체 및 심사위원 준비 시간 |
| **슬롯 간격** | 10분 | (발표8분 + 전환2분) |
| **휴식** | 5팀마다 10분 | 심사위원 휴식 |
| **점심** | 11:40 ~ 13:00 | 80분 점심시간 |
| **일일 운영시간** | 10:00 ~ 18:00 | 실제 심사 시간 |
| **평가 정리** | 30분 | 마지막 1회 (선택) |

#### 생성 로직

```
1. 설정 파라미터 로드
   ├─ 시작일자, 진행요일(월~일 중 선택), 하루 시간
   ├─ 팀당 발표시간, 전환시간, 휴식 규칙
   └─ 점심시간, 평가 정리 여부

2. 일정 슬롯 생성
   ├─ 진행 요일별로 10분 단위 슬롯 배치
   ├─ 점심시간 자동 삽입
   ├─ 5팀마다 휴식 10분 자동 삽입
   └─ 결과: evaluation_timetable_slot 테이블에 저장

3. 팀 배치
   ├─ 점수 순으로 팀 정렬 (상위 팀 먼저)
   ├─ slot_order에 따라 순차 배치
   └─ 각 팀을 특정 슬롯(시간·날짜)에 할당

4. Excel 생성
   ├─ 일차별(1~12일) 시트 생성
   ├─ 각 시트에 해당 날짜 팀 정보 + 시간 기입
   └─ 다운로드 제공
```

---

## 🗄️ 데이터베이스 설계

### TIME_TABLE 테이블 (설정 정보)

```sql
CREATE TABLE time_table (
    timetable_id BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '시간표 설정 고유 ID',
    title VARCHAR(255) NOT NULL COMMENT '시간표 제목 (화면 및 출력용)',
    location VARCHAR(255) COMMENT '심사 진행 장소',
    start_date DATE NOT NULL COMMENT '일정 시작 날짜',
    
    -- 진행 요일
    mon TINYINT(1) DEFAULT 0 COMMENT '월요일 진행 여부',
    tue TINYINT(1) DEFAULT 0 COMMENT '화요일 진행 여부',
    wed TINYINT(1) DEFAULT 0 COMMENT '수요일 진행 여부',
    thu TINYINT(1) DEFAULT 0 COMMENT '목요일 진행 여부',
    fri TINYINT(1) DEFAULT 0 COMMENT '금요일 진행 여부',
    
    -- 하루 시간대
    day_start TIME NOT NULL COMMENT '하루 심사 시작 시간 (예: 10:00)',
    day_end TIME NOT NULL COMMENT '하루 심사 종료 시간 (예: 18:00)',
    lunch_start TIME NOT NULL COMMENT '점심 시작 시간 (예: 11:40)',
    lunch_end TIME NOT NULL COMMENT '점심 종료 시간 (예: 13:00)',
    
    -- 슬롯 규칙
    presentation_minutes INT NOT NULL COMMENT '팀당 발표 시간 (분)',
    transition_minutes INT NOT NULL COMMENT '팀 간 전환 시간 (분)',
    break_every INT NOT NULL COMMENT '몇 팀마다 휴식 (예: 5)',
    break_minutes INT NOT NULL COMMENT '휴식 시간 (분)',
    summary_minutes INT NOT NULL COMMENT '평가 정리 시간 (분)',
    
    -- 메타데이터
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    created_by BIGINT,
    updated_at DATETIME NULL,
    updated_by BIGINT,
    state INT DEFAULT 0 COMMENT '0: 정상, 1: 삭제'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### EVALUATION_TIMETABLE_SLOT 테이블 (생성된 슬롯)

```sql
CREATE TABLE evaluation_timetable_slot (
    slot_id BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '슬롯 고유 ID',
    timetable_id BIGINT NOT NULL COMMENT '어느 시간표 설정에서 생성되었는가',
    generation_id INT NOT NULL COMMENT '생성 회차 (같은 설정에서 여러 번 생성 가능)',
    evaluation_date DATE NOT NULL COMMENT '평가 일자',
    start_time TIME NOT NULL COMMENT '슬롯 시작 시간',
    end_time TIME NOT NULL COMMENT '슬롯 종료 시간',
    slot_order INT NOT NULL COMMENT '생성된 슬롯 순번',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NULL,
    state INT DEFAULT 0,
    
    FOREIGN KEY (timetable_id) REFERENCES time_table(timetable_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### 예시 데이터 (10시간 운영, 8분 발표 + 2분 전환)

```
| slot_id | evaluation_date | start_time | end_time | slot_order |
|---------|-----------------|------------|----------|------------|
| 1       | 2026-07-09      | 10:00      | 10:08    | 1          |
| 2       | 2026-07-09      | 10:10      | 10:18    | 2          |
| 3       | 2026-07-09      | 10:20      | 10:28    | 3          |
| ...     | ...             | ...        | ...      | ...        |
| 15      | 2026-07-09      | 17:52      | 18:00    | 15         |
| (점심)  | 2026-07-09      | 11:40      | 13:00    | -          |
| (휴식)  | 2026-07-09      | 11:30      | 11:40    | -          |
```

---

## 🔥 주요 기능

### 1️⃣ 시간표 설정 등록 & 관리

**운영지원팀** → 시간표 기본 정보 입력

```
제목: "2026 성장 평가심사 발표 일정표"
장소: "한국벤처투자빌딩 지하 1층"
시작일: 2026-07-09 (달력 선택)
진행요일: 월, 화, 수, 목, 금, 토 (체크박스)

하루 일정:
  - 시작: 10:00
  - 종료: 18:00
  - 점심: 11:40 ~ 13:00

팀 정보:
  - 팀당 발표: 8분
  - 전환시간: 2분
  - 휴식: 5팀마다 10분
  - 평가 정리: 30분 (마지막 1회)

총 팀 수: 300팀 ← 반드시 일치해야 함!
```

**API 명세**

```
POST /api/admin/evaluation/timetable-config
Content-Type: application/json

Request:
{
  "title": "2026 성장 평가심사 발표 일정표",
  "location": "한국벤처투자빌딩 지하 1층",
  "startDate": "2026-07-09",
  "mon": true,
  "tue": true,
  "wed": true,
  "thu": true,
  "fri": true,
  "sat": true,
  "dayStart": "10:00",
  "dayEnd": "18:00",
  "lunchStart": "11:40",
  "lunchEnd": "13:00",
  "presentationMinutes": 8,
  "transitionMinutes": 2,
  "breakEvery": 5,
  "breakMinutes": 10,
  "summaryMinutes": 30,
  "totalTeams": 300
}

Response (201):
{
  "timetableId": 1,
  "title": "2026 성장 평가심사 발표 일정표",
  "createdAt": "2026-03-30T10:00:00Z",
  "status": "saved"
}
```

### 2️⃣ 시간표 생성 (자동 최적화)

**평가관리팀** → 저장된 설정 선택 → 자동 생성

```
[시간표 목록]
- 2026 성장 평가심사 발표 일정표 (등록일: 2026-03-30)
  [상세 보기]

[상세 화면]
제목: 2026 성장 평가심사 발표 일정표
총 팀 수 (설정): 300팀
현재 트랙 팀 수: 300팀 ✅

[수정] [시간표 생성 🟢 활성화] [엑셀 다운로드 🔒 비활성화]

생성 일시: -
```

**API 명세**

```
POST /api/admin/evaluation/timetable-config/{timetableId}/generate

Request: {} (설정 ID만 필요)

Response (200):
{
  "timetableId": 1,
  "generationId": 1,
  "totalSlots": 336,  // 12일 × 28슬롯
  "totalTeams": 300,
  "generatedAt": "2026-03-30T11:00:00Z",
  "status": "success",
  "message": "300개 팀의 시간표가 생성되었습니다"
}
```

### 3️⃣ Excel 다운로드

**최종 산출물** - 12개 일차별 시트

```
[1일차 (2026-07-09) 시트]

aaaaaaaaaaa - 오프라인 발표심사 일정표 1일차(7/9) 일정표
장소: 한국벤처투자빌딩 지하 1층

| 연번 | 일자 | 시간(시작) | 시간(종료) | 학교명 | 고유번호 | 팀명 | 발표자명 | 성별 | 학생구분 | 역할 | 핸드폰 | 비고 |
|------|------|-----------|-----------|--------|---------|------|---------|------|---------|------|--------|------|
| 1    | 7/9  | 10:00     | 10:08     | 중앙대  | 9429    | 팀1 | [발표자]   | 남   | 대학교  | 대표 | [연락처제거] | |
| 2    | 7/9  | 10:10     | 10:18     | 한동대  | 9401    | 팀2 | [발표자]  | 남   | 대학교  | 대표 | [연락처제거] | |
| ...  | ...  | ...       | ...       | ...    | ...     | ...   | ...    | ...  | ...    | ... | ...    | |
| (점심시간 80분) | 11:40 | 13:00 |
| ...  | ...  | ...       | ...       | ...    | ...     | ...   | ...    | ...  | ...    | ... | ...    | |
```

**API 명세**

```
GET /api/admin/evaluation/timetable-config/{timetableId}/export-excel

Query Parameters:
  ?generationId=1

Response (200):
  Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
  Content-Disposition: attachment; filename="2026_성장평가심사_평가시간표.xlsx"

[파일 다운로드]
```

---

## ✅ 구현 현황 & 이슈 처리

### 완료된 기능

- ✅ 시간표 설정 정보 등록 (Create, Read, Update)
- ✅ 시간표 자동 생성 알고리즘 (330개 슬롯 최적화)
- ✅ 팀 수 일치 검증 로직
- ✅ Excel 자동 생성 (12개 일차별 시트)
- ✅ Swagger API 문서화

### 마주친 이슈 & 해결 방법

#### 이슈 1: SQL INSERT 문법 에러

**문제:**
```sql
INSERT INTO evaluation_timetable_slot (...) VALUES
( 1, 1, '2026-07-07', '09:00', '09:08', 1 now(), 1, 0 )
-- SQL 에러: now() 위치 잘못됨
```

**해결:**
```sql
INSERT INTO evaluation_timetable_slot (...) VALUES
( 1, 1, '2026-07-07', '09:00', '09:08', 1, NOW(), 1, 0 )
-- created_at 컬럼에 NOW() 함수 올바르게 배치
```

#### 이슈 2: 점심시간 및 휴식 처리

**문제:**
- 점심시간(80분) 중간에 팀을 배치해야 함
- 5팀마다 자동으로 휴식 10분 삽입

**해결:**
```java
// 하루 슬롯 생성 로직
List<TimeSlot> slots = new ArrayList<>();
LocalTime current = dayStart;

while (current.isBefore(dayEnd)) {
    // 점심시간 체크
    if (current.equals(lunchStart)) {
        slots.add(new TimeSlot(lunchStart, lunchEnd, "LUNCH"));
        current = lunchEnd;
    }
    
    // 휴식 삽입 (5팀마다)
    if (slots.size() % 5 == 0 && !isLunch(current)) {
        slots.add(new TimeSlot(current, current.plusMinutes(breakMinutes), "BREAK"));
        current = current.plusMinutes(breakMinutes);
    }
    
    // 일반 슬롯
    slots.add(new TimeSlot(current, current.plusMinutes(presentationMinutes), "PRESENTATION"));
    current = current.plusMinutes(presentationMinutes + transitionMinutes);
}
```

#### 이슈 3: 팀 수 불일치 검증

**문제:**
- 설정된 팀 수(300)와 실제 트랙 팀 수가 다를 수 있음
- 불일치 시에도 시간표 생성 가능 → 실제 운영과 맞지 않음

**해결:**
```java
@PostMapping("/api/admin/evaluation/timetable/{timetableId}/generate")
public ResponseEntity<?> generateTimetable(
    @PathVariable Long timetableId,
    @RequestParam(required = false) Long trackNo) {
    
    TimeTable config = timeTableService.getConfig(timetableId);
    int actualTeamCount = teamService.countByTrack(trackNo);
    
    // 팀 수 검증
    if (config.getTotalTeams() != actualTeamCount) {
        return ResponseEntity.badRequest().body(new ErrorResponse(
            "TEAM_COUNT_MISMATCH",
            String.format("설정된 팀 수(%d)와 실제 팀 수(%d)가 일치하지 않습니다",
                         config.getTotalTeams(), actualTeamCount)
        ));
    }
    
    // 생성 진행
    return ResponseEntity.ok(timeTableService.generateSlots(timetableId));
}
```

#### 이슈 4: Excel 생성 시 일자 포맷팅

**문제:**
```
2026-07-09 → "7월 9일(목)" 형식으로 변환 필요
```

**해결:**
```java
LocalDate date = LocalDate.of(2026, 7, 9);
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("M월 d일(E)", Locale.KOREAN);
String formattedDate = date.format(formatter);
// 결과: "7월 9일(목)"
```

---

## 📈 성과 및 효과

### Before (수작업)

| 항목 | 시간 |
|------|------|
| 300개 팀 일정 배치 | 약 3~4시간 |
| Excel 작성 | 약 1~2시간 |
| 버전 관리 | 수동 파일 관리 |
| 오류 가능성 | **높음** |

### After (자동화)

| 항목 | 시간 |
|------|------|
| 설정 입력 | 약 10분 |
| 자동 생성 | **즉시** (< 1초) |
| Excel 다운로드 | **자동** |
| 오류 가능성 | **최소화** |

**총 소요 시간 단축: 4~6시간 → 15분 (96% 단축) ✨**

---

## 🎓 배운 점

### 기술적 성장

1. **복잡한 시간 계산 로직 설계**
   - LocalTime, LocalDate를 활용한 시간 슬롯 최적화
   - 반복 패턴(5팀마다 휴식) 구현

2. **대량 데이터 Excel 생성**
   - Apache POI 활용
   - 12개 시트 병렬 생성
   - 날짜 포맷팅 및 한글화

3. **검증 로직의 중요성**
   - 팀 수 불일치 사전 차단
   - 설정값 유효성 검사 (시간대 겹침, 음수값 등)

### 도메인 이해

1. **청년 창업 생태계**
   - 다양한 학생구분 (대학교, 대학원, 고등학교, 초등학교 등)
   - 지역별 팀 분포 특성

2. **운영 프로세스**
   - 심사위원의 물리적 이동 시간 고려
   - 점심시간·휴식을 통한 심사위원 피로도 관리의 중요성

3. **데이터 기반 의사결정**
   - 설정을 일단 DB에 저장 → 여러 번 재사용 가능
   - 생성 이력(generationId) 관리로 실수 회복 가능

---

## 📚 기술 스택

| 계층 | 기술 |
|------|------|
| **Backend** | Java 17, Spring Boot 3.x |
| **Database** | MariaDB 10.x |
| **ORM** | MyBatis |
| **Excel** | Apache POI |
| **API Docs** | Swagger/OpenAPI 3.0 |
| **Date/Time** | Java Time API (LocalDate, LocalTime) |

---

## 🚀 앞으로의 개선 방향

- [ ] 시간표 수정 기능 (생성 후 특정 팀만 시간 변경)
- [ ] 심사위원별 배치 최적화 (특정 심사위원만 특정 시간대 배정)
- [ ] 시간표 변경 이력 추적 (Audit Log)
- [ ] 심사위원 알림 자동화 (이메일·SMS)
- [ ] 예비 시간표 생성 (팀 추가 신청 시 대응)

---

## 📞 Contact & 더 알아보기

이 프로젝트는 **U300 온라인 창업 플랫폼** 중 일부입니다.

- **전체 플랫폼**: 서류심사 → 온라인 교육 → 오프라인 발표심사 → 투자 연계
- **담당 역할**: 평가 관리 전 프로세스 백엔드 설계·개발
- **규모**: 연간 8,000명 지원자 관리

더 많은 프로젝트 사례는 포트폴리오를 참고해주세요!

---

## 💡 핵심 인사이트

> **"단순한 자동화가 아닌, 운영 현황을 이해한 설계"**

이 프로젝트의 가치는 단순히 "시간표를 빨리 만드는 것"이 아닙니다.

- 📋 **설정을 DB에 저장** → 재사용 가능
- 🔍 **팀 수 검증** → 실수 방지
- 📊 **생성 이력 관리** → 언제든 이전 버전 복구
- 🎯 **점심시간·휴식 자동화** → 심사위원 피로도 관리

**운영진의 시간을 아끼고, 청년 창업가들이 공정한 심사를 받을 수 있는 환경을 만드는 것.**

그것이 이 시스템의 진정한 목적입니다.

---

*프로젝트 기간: 2026년 3월 ~ 현재*  
*담당자: Backend Engineer (Java/Spring Boot)*
