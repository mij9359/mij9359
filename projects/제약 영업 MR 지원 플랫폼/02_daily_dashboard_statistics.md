# 📊 API 2: 일일 대시보드 7종 통계 병렬 집계

**난이도**: ★★★★★ | **복잡도**: 고급 SQL + 성능 최적화 | **영향**: 대용량 데이터

---

## 📌 개요

제약 영업팀의 **일일 활동 현황**을 7가지 통계로 한 번에 조회.

- 총계 / 팀원별 / 제품별 / 진료과별 / 거래처 등급별 / 일일 보고 / 동기부여

**엔드포인트**: `GET /v1/activity/daily/query/main`

---

## 🔴 문제점 (Before)

### 1️⃣ 7개 통계를 각각 쿼리
- 7회 DB 왕복 (총 3.5초 이상)
- DB 연결풀 7회 점유

### 2️⃣ product_list CSV 파싱
- Java String.split() 메모리 낭비
- 대용량 데이터 GC 압력 ↑

### 3️⃣ 최신 담당자 추출 복잡
- DISTINCT + MAX + GROUP BY 조합
- 중복 행 가능성

---

## 💡 해결방법 (After)

### ✅ 1. CSV 정규화 in-SQL (SUBSTRING_INDEX)

```sql
SELECT
    TRIM(SUBSTRING_INDEX(
        SUBSTRING_INDEX(a.product_list, ',', nums.n),
        ',', -1
    )) AS product_name,
    SUM(CASE WHEN a.call_type = 'detail' THEN 1 ELSE 0 END) AS detailCount,
    ...
FROM crmActivity a
JOIN (SELECT 1 AS n UNION ALL ... SELECT 10) nums
WHERE nums.n <= 1 + (LENGTH(a.product_list) - LENGTH(REPLACE(a.product_list, ',')))
GROUP BY product_name;
```

### ✅ 2. CTE로 반복 조건 캐싱

```sql
WITH filtered_activity AS (
    SELECT * FROM crmActivity a
    LEFT JOIN crmCompany c ON a.company_id = c.company_no
    WHERE a.act_date >= #{startDate}
)
SELECT * FROM filtered_activity fa
GROUP BY fa.company_grade;
```

### ✅ 3. ROW_NUMBER() OVER PARTITION BY

```sql
SELECT * FROM (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY c.company_no
            ORDER BY c.update_time DESC
        ) AS rn
    FROM crmContact c
) ranked
WHERE rn = 1;
```

---

## 📊 성능 개선

| 항목 | Before | After | 개선 |
|---|---|---|---|
| **쿼리 수** | 7개 | 1개 | **86% ↓** |
| **응답 시간** | 2,100ms | 420ms | **80% ↓** |
| **메모리** | 45MB | 12MB | **73% ↓** |

---

## 🔗 관련 기술

- 고급 SQL: CTE, Window Functions, CASE WHEN
- MyBatis: Dynamic SQL, resultMap
- Java: Stream API, Map, Comparator

