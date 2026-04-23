# 📦 API 7: 경쟁제품 데이터 업로드 이력 (Raw 문자열 파싱 + 유효성 검증)

**난이도**: ★★★ | **복잡도**: 데이터 정제 + 파싱 + 검증 | **영향**: 엑셀 업로드 이력 관리

---

## 📌 개요

KNDA(한국의약품유통협회) 판매 데이터, 제품별 목표, 경쟁제품 시장 정보 등을 **엑셀로 대량 업로드**했을 때, 각 업로드의 이력(연/월 범위, 소요 시간, 상태)를 조회.

**엔드포인트**: `GET /v1/competitor/query/knda/history`

**응답 예시**:
```json
{
  "data": [
    {
      "fileName": "KNDA_2024_data.xlsx",
      "uploadedBy": "admin@company.com",
      "uploadedAt": "2024-06-30 14:30:00",
      "uploadedYearMonths": [
        {"year": 2024, "months": [1, 2, 3]},
        {"year": 2025, "months": [1, 2]}
      ],
      "totalCount": 15000,
      "elapsedTime": "5분 30초"
    }
  ]
}
```

---

## 🔴 문제점 (Before)

### 1️⃣ Raw 문자열 형식이 불규칙
```
❌ 문제:
DB에 저장된 형식: "2024:1,2,3/2025:1,2,3,4"

→ "2024:" 뒤에는 항상 ","로 구분된 월이 있나?
→ 월이 빠질 수도 있나? (예: "2024:/2025:1,2")
→ 마지막이 "/"로 끝나는 경우? (예: "2024:1,2,3/")
→ 앞뒤 공백?

→ 각 경우마다 예외 처리 필요
```

### 2️⃣ 유효성 검증 부재
```
❌ 문제:
파싱 후 데이터:
- year = 0 (invalid!)
- months = [0, 13] (invalid!)
- months = [] (empty!)

→ UI 렌더링 시 이상한 데이터 표시
→ 사용자 혼란 (뭐가 업로드 된 거지?)
```

### 3️⃣ 시간 문자열 파싱
```
❌ 문제:
elapsedTime = "5분 30초"

→ 초 단위로 변환 필요: 5*60 + 30 = 330초
→ 다양한 형식 대응 필요:
  - "5분 30초"
  - "5분"
  - "30초"
  - "5m 30s" (영문)
```

### 4️⃣ 삭제 가능 여부 판정
```
❌ 문제:
해당 업로드의 데이터가 다른 테이블에서 참조되고 있다면
→ 삭제할 수 없음 (FK 제약)

→ 참조 수를 미리 조회해야 함
```

---

## 💡 해결방법 (After)

### ✅ 1. 연/월 Raw 문자열 파서

```java
@Service
public class CompetitorProductQueryServiceImpl {
    
    /**
     * Raw 문자열: "2024:1,2,3/2025:1,2,3,4"
     * → List<YearMonthGroupDTO>로 변환
     */
    public List<YearMonthGroupDTO> parseYearMonthGroups(String uploadedYearMonths) {
        List<YearMonthGroupDTO> result = new ArrayList<>();
        
        // 1. 입력 검증
        if (uploadedYearMonths == null || uploadedYearMonths.trim().isEmpty()) {
            return result;
        }
        
        // 2. "/" 기준으로 연도 분할
        String[] yearParts = uploadedYearMonths.split("/");
        
        for (String part : yearParts) {
            part = part.trim();
            
            // ✅ 방어: ":" 구분자 확인
            if (!part.contains(":")) {
                log.warn("Invalid format (no colon): {}", part);
                continue;
            }
            
            try {
                // 3. ":" 기준으로 연도와 월 분할
                String[] split = part.split(":");
                
                if (split.length < 2) {
                    log.warn("Invalid format (split size): {}", part);
                    continue;
                }
                
                // 연도 파싱
                int year = Integer.parseInt(split[0].trim());
                
                // 월 파싱 (쉼표 분리)
                List<Integer> months = Arrays.stream(split[1].split(","))
                    .map(String::trim)
                    .filter(s -> !s.isEmpty())  // ✅ 방어: 빈 문자열 필터
                    .map(Integer::parseInt)
                    .collect(Collectors.toList());
                
                // 유효한 데이터만 추가 (검증은 별도)
                result.add(new YearMonthGroupDTO(year, months));
                
            } catch (NumberFormatException e) {
                log.warn("Number parse error: {}", part, e);
                // 해당 부분만 스킵하고 계속
                continue;
            }
        }
        
        return result;
    }
}
```

**파싱 예시**:
```
입력: "2024:1,2,3/2025:1,2"

Step 1: "/" 분할
→ ["2024:1,2,3", "2025:1,2"]

Step 2: 각각 ":" 분할
→ "2024" + "1,2,3"
→ "2025" + "1,2"

Step 3: "," 분할
→ year=2024, months=[1, 2, 3]
→ year=2025, months=[1, 2]

결과: [
  YearMonthGroupDTO(2024, [1, 2, 3]),
  YearMonthGroupDTO(2025, [1, 2])
]
```

---

### ✅ 2. 유효성 검증 (Stream.anyMatch)

```java
public List<ExcelImportHistoryResDTO> selectProductMarketImportHistory() {
    List<ExcelImportHistoryResDTO> resultList = 
        competitorProductQueryRepo.selectProductMarketImportHistory();
    
    if (resultList == null) {
        resultList = Collections.emptyList();
    }
    
    // ✅ 각 업로드 이력마다 파싱 + 검증
    for (ExcelImportHistoryResDTO dto : resultList) {
        String raw = dto.getUploadedYearMonthsRaw();
        
        // 1. 파싱
        List<YearMonthGroupDTO> parsed = parseYearMonthGroups(raw);
        dto.setUploadedYearMonthsParsed(parsed);
        
        // 2. ✅ 유효성 검증 (invalid 값 감지)
        boolean containsInvalidValue = parsed.stream().anyMatch(group ->
            // year가 0? (invalid)
            group.getYear() == 0 ||
            // months가 null? (invalid)
            group.getMonths() == null ||
            // months가 비어있음? (invalid)
            group.getMonths().isEmpty() ||
            // months에 0이 포함? (month는 1~12, 0은 invalid)
            group.getMonths().stream().anyMatch(month -> month == 0)
        );
        
        dto.setContainsInvalidValue(containsInvalidValue);
        
        // 3. 삭제 가능 여부 판정
        boolean deletable = false;
        if (!containsInvalidValue && !parsed.isEmpty()) {
            // ✅ 참조 수 조회 (해당 데이터를 사용하는 다른 테이블 확인)
            long refCount = competitorProductQueryRepo
                .countByMultipleYearMonthGroups(parsed);
            
            // 참조가 없으면 삭제 가능
            deletable = (refCount == 0);
        }
        
        dto.setDeletable(deletable);
    }
    
    return resultList;
}
```

**검증 예시**:
```
데이터 1: "2024:1,2,3"
→ parsed = [YearMonthGroupDTO(2024, [1, 2, 3])]
→ invalid? false (모든 값 valid)
→ deletable? true (참조 없음)

데이터 2: "2024:0,1,2"
→ parsed = [YearMonthGroupDTO(2024, [0, 1, 2])]
→ invalid? true (month=0은 invalid)
→ deletable? false (invalid 데이터는 삭제 불가)

데이터 3: "0:1,2,3"
→ parsed = [YearMonthGroupDTO(0, [1, 2, 3])]
→ invalid? true (year=0은 invalid)
→ deletable? false
```

---

### ✅ 3. 시간 문자열 파싱

```java
public static int parseElapsedTimeToSeconds(String elapsedTimeStr) {
    if (elapsedTimeStr == null || elapsedTimeStr.isBlank()) {
        return 0;
    }
    
    int totalSeconds = 0;
    
    try {
        // 분 추출: "5분 30초" → "5"
        if (elapsedTimeStr.contains("분")) {
            int minuteIndex = elapsedTimeStr.indexOf("분");
            String minuteStr = elapsedTimeStr.substring(0, minuteIndex).trim();
            
            // 앞의 숫자만 추출 (예: "5분" → "5")
            minuteStr = minuteStr.replaceAll("[^0-9]", "");
            
            if (!minuteStr.isEmpty()) {
                int minutes = Integer.parseInt(minuteStr);
                totalSeconds += minutes * 60;
            }
        }
        
        // 초 추출: "5분 30초" → "30"
        if (elapsedTimeStr.contains("초")) {
            int secondIndex = elapsedTimeStr.indexOf("초");
            
            // "분" 이후 부분만 추출
            int startIdx = elapsedTimeStr.contains("분") 
                ? elapsedTimeStr.indexOf("분") + 1
                : 0;
            
            String secondStr = elapsedTimeStr.substring(startIdx, secondIndex).trim();
            secondStr = secondStr.replaceAll("[^0-9]", "");
            
            if (!secondStr.isEmpty()) {
                int seconds = Integer.parseInt(secondStr);
                totalSeconds += seconds;
            }
        }
        
    } catch (NumberFormatException e) {
        log.warn("Failed to parse elapsed time: {}", elapsedTimeStr, e);
        return 0;
    }
    
    return totalSeconds;
}
```

**파싱 예시**:
```
입력: "5분 30초"
→ 분: "5" → 5 * 60 = 300초
→ 초: "30" → 30초
→ 합계: 330초

입력: "5분"
→ 분: "5" → 5 * 60 = 300초
→ 초: (없음) → 0초
→ 합계: 300초

입력: "45초"
→ 분: (없음) → 0초
→ 초: "45" → 45초
→ 합계: 45초
```

---

### ✅ 4. DTO 구조

```java
@Data
@Builder
public class YearMonthGroupDTO {
    private int year;
    private List<Integer> months;
}

@Data
@Builder
public class ExcelImportHistoryResDTO {
    private String fileId;
    private String fileName;
    private String uploadedBy;
    private LocalDateTime uploadedAt;
    
    // Raw 형식 저장
    private String uploadedYearMonthsRaw;  // "2024:1,2,3/2025:1,2"
    
    // 파싱된 형식
    private List<YearMonthGroupDTO> uploadedYearMonthsParsed;
    
    // 유효성 플래그
    private boolean containsInvalidValue;
    
    // 삭제 가능 여부
    private boolean deletable;
    
    // 기타 정보
    private int totalCount;
    private String elapsedTime;  // "5분 30초"
    private int elapsedTimeSeconds;  // 330
}
```

---

## 📝 핵심 코드 (Service 레이어 전체)

```java
@Service
public class CompetitorProductQueryServiceImpl {
    
    @Transactional(readOnly = true)
    public ProductMarketDataUploadResVO getProductMarketImportHistory() {
        // 1. DB에서 업로드 이력 조회
        List<ExcelImportHistoryResDTO> resultList =
            competitorProductQueryRepo.selectProductMarketImportHistory();
        
        if (resultList == null) {
            resultList = Collections.emptyList();
        }
        
        // 2. 각 업로드마다 파싱 + 검증 + 시간 변환
        for (ExcelImportHistoryResDTO dto : resultList) {
            // 연/월 파싱
            List<YearMonthGroupDTO> parsed = 
                parseYearMonthGroups(dto.getUploadedYearMonthsRaw());
            dto.setUploadedYearMonthsParsed(parsed);
            
            // 유효성 검증
            boolean invalid = parsed.stream().anyMatch(group ->
                group.getYear() == 0 ||
                group.getMonths() == null ||
                group.getMonths().isEmpty() ||
                group.getMonths().stream().anyMatch(m -> m == 0)
            );
            dto.setContainsInvalidValue(invalid);
            
            // 삭제 가능 여부
            boolean deletable = !invalid && !parsed.isEmpty() &&
                competitorProductQueryRepo.countByMultipleYearMonthGroups(parsed) == 0;
            dto.setDeletable(deletable);
            
            // 시간 변환
            int secondsElapsed = parseElapsedTimeToSeconds(dto.getElapsedTime());
            dto.setElapsedTimeSeconds(secondsElapsed);
        }
        
        return new ProductMarketDataUploadResVO(resultList);
    }
}
```

---

## 📊 성능 & 특징

| 항목 | 설명 |
|---|---|
| **파싱** | "/" → ":" → "," 순차 분할 |
| **방어** | NumberFormatException 캐치, 빈문자열 필터 |
| **검증** | year==0, month==0 체크, isEmpty() 체크 |
| **유연성** | 다양한 시간 형식 대응 |

---

## 🛡️ 보안

### 1. 입력 검증
```java
if (uploadedYearMonths == null || uploadedYearMonths.trim().isEmpty()) {
    return new ArrayList<>();
}
```

### 2. 예외 격리
```java
try {
    int year = Integer.parseInt(split[0].trim());
} catch (NumberFormatException e) {
    log.warn("Number parse error: {}", part, e);
    continue;  // 해당 부분 스킵, 전체 실패 아님
}
```

---

## 🔗 관련 기술

- **String**: split, substring, trim, indexOf
- **Stream API**: map, filter, collect, anyMatch
- **정규식**: replaceAll("[^0-9]", "")
- **예외 처리**: try-catch, null-safe

---

## 📚 참고 커밋

- `경쟁 제품 데이터 테이블, api 분리 및 조회 쿼리 수정`
- `경쟁 제품 조회 개수 15개로 제한`

