# 💪 API 6: 동기부여 등록 (팀/본부 이중 중복 체크 + AI 어조 변환)

**난이도**: ★★★ | **복잡도**: 계층 구조 + 중복 검증 + AI 연동 | **영향**: 팀 동기부여 관리

---

## 📌 개요

팀장(LEADER) 또는 본부장(DIRECTOR)이 **팀/본부별로 하루 한 번씩** 동기부여 메시지를 등록.

- 팀별 일일 1건만 등록 가능 (중복 방지)
- 등록 후 AI 서버에서 "딱딱한 원문" → "친근한 어조"로 변환 가능
- 조회 시 당일 없으면 전일 메시지 표시 (fallback)

**엔드포인트**: `POST /v1/activity/daily/command/motivation`

---

## 🔴 문제점 (Before)

### 1️⃣ 팀과 본부의 계층 구조 이해 부족
```
❌ 문제:
- LEADER (팀장) 권한: 특정 팀에만 메시지 등록 가능
- DIRECTOR (본부장) 권한: 전체 본부에 메시지 등록 가능
- 동일 날짜에 팀장 메시지 + 본부장 메시지 둘 다 존재 가능

→ "팀장이 이미 등록했으면, 본부장은 또 등록 가능한가?"
→ 비즈니스 규칙 불명확
```

### 2️⃣ 중복 체크 로직 불완전
```
❌ 기존 코드:
SELECT COUNT(*) FROM crmMotivation
WHERE dept_name = #{deptName} AND DATE(motivation_date) = #{date}

→ 이 쿼리 결과가 1이면 중복
→ 하지만 LEADER와 DIRECTOR의 역할별 분리 안 함
```

### 3️⃣ 조회 시 최신 메시지 선택 어려움
```
❌ 문제:
- 당일 LEADER 메시지: 2024-01-15 10:00
- 전일 LEADER 메시지: 2024-01-14 15:00
- 당일 DIRECTOR 메시지: 없음
- 전일 DIRECTOR 메시지: 2024-01-14 09:00

→ LEADER는 당일, DIRECTOR는 전일?
→ 각 역할별 최신 1건씩 선택 필요
```

---

## 💡 해결방법 (After)

### ✅ 1. 팀/본부 이중 중복 체크

```java
@Service
@Transactional
public class DailyActivityCommandServiceImpl {
    
    public void insertMotivation(
        MotivationInsertDto dto,
        String userId
    ) {
        // 1. 사용자 정보 조회
        UserRoleDto userRole = queryRepo.findUserRole(userId);
        String role = getRole(userId);  // LEADER 또는 DIRECTOR 판별
        
        // 2. 중복 체크: 동일 부서 + 동일 날짜 + 동일 역할
        boolean exists = queryRepo.existsMotivationByDeptAndDate(
            userRole.getDeptName(),
            dto.getMotivationDate()
        );
        
        if (exists) {
            // ✅ 역할별 다른 에러 메시지
            if ("LEADER".equals(role) && "Y".equals(userRole.getCrmMrLeaderYn())) {
                throw new BusinessException(
                    "해당 부서에 이미 해당 날짜의 동기부여가 등록되어 있습니다. (팀장)"
                );
            } else if ("DIRECTOR".equals(role) && "Y".equals(userRole.getCrmHeadLeaderYn())) {
                throw new BusinessException(
                    "해당 부서에 이미 해당 날짜의 동기부여가 등록되어 있습니다. (본부장)"
                );
            }
        }
        
        // 3. 동기부여 등록
        MotivationDto motivationDto = MotivationDto.builder()
            .motivationContent(dto.getMotivationContent())
            .motivationDate(dto.getMotivationDate())
            .deptName(userRole.getDeptName())
            .position(role)  // LEADER / DIRECTOR
            .createdUser(userId)
            .createdDate(LocalDateTime.now())
            .build();
        
        commandRepo.insertMotivation(motivationDto);
    }
    
    /**
     * 사용자 역할 판별
     */
    private String getRole(String employeeId) {
        UserRoleDto userRole = queryRepo.findUserRole(employeeId);
        
        // 팀장 권한 확인
        if ("Y".equalsIgnoreCase(userRole.getCrmMrLeaderYn())) {
            return "LEADER";
        }
        
        // 본부장 권한 확인
        if ("Y".equalsIgnoreCase(userRole.getCrmHeadLeaderYn())) {
            return "DIRECTOR";
        }
        
        throw new BusinessException("요청 유저가 팀장 혹은 본부장이 아닙니다");
    }
}
```

**DB 쿼리**:
```sql
<!-- 중복 체크 -->
SELECT EXISTS (
    SELECT 1 FROM crmMotivation
    WHERE dept_name = #{deptName}
        AND DATE(motivation_date) = #{motivationDate}
        AND position = #{position}  -- LEADER 또는 DIRECTOR
)
```

---

### ✅ 2. AI 어조 변환

```java
@Transactional
public MotivationConvertResDto convertMotivationTone(
    MotivationConvertDto dto
) {
    try {
        // 1. 요청 준비
        OrchestratorRequestDto requestDto = OrchestratorRequestDto.builder()
            .userInput(dto.getUser_input())  // 원문
            .build();
        
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        
        String jsonBody = objectMapper.writeValueAsString(requestDto);
        HttpEntity<String> requestEntity = new HttpEntity<>(jsonBody, headers);
        
        // 2. AI 서버 호출
        ResponseEntity<MotivationConvertResDto> response = restTemplate.exchange(
            motivationApiUrl,
            HttpMethod.POST,
            requestEntity,
            new ParameterizedTypeReference<MotivationConvertResDto>() {}
        );
        
        // 3. 응답 검증
        MotivationConvertResDto body = response.getBody();
        
        if (body == null || body.getResponse() == null ||
            body.getResponse().getBeautiful_message() == null) {
            throw new RuntimeException("AI 서버 응답이 비어 있습니다");
        }
        
        // 4. 결과 반환
        return new MotivationConvertResDto(
            new MotivationConvertResDto.Response(
                body.getResponse().getBeautiful_message()
            )
        );
        
    } catch (JsonProcessingException e) {
        throw new RuntimeException("요청 생성 실패: " + e.getMessage(), e);
        
    } catch (HttpClientErrorException | HttpServerErrorException e) {
        log.error("🟥 HTTP 오류 응답: 상태코드={}, 메시지={}",
            e.getStatusCode(), e.getResponseBodyAsString(), e);
        throw new RuntimeException("AI 서버 오류 응답: " + e.getMessage(), e);
        
    } catch (RestClientException e) {
        throw new RuntimeException("AI 서버 통신 실패: " + e.getMessage(), e);
        
    } catch (NullPointerException e) {
        throw new RuntimeException("AI 응답 파싱 실패: 필드가 null입니다", e);
        
    } catch (Exception e) {
        throw new RuntimeException("AI 서버 호출 실패: " + e.getMessage(), e);
    }
}
```

**사용 예시**:
```
입력: "오늘 영업 실적이 좋으니 모두 수고했어"
AI 응답: "오늘도 수고가 많으셨어요! 뛰어난 영업 실적으로 팀의 자랑입니다."

입력: "업무 효율을 높이자"
AI 응답: "함께라면 더욱 효율적으로 일할 수 있을 거에요!"
```

---

### ✅ 3. Fallback 조회: 당일 없으면 전일

```java
@Transactional(readOnly = true)
public List<MotivationContentDto> getMotivationContent(
    String deptName,
    LocalDate targetDate
) {
    // 1. 본부 정보 조회 (deptName → headquarter 매핑)
    String headquarter = queryRepo.findHeadquarter(deptName);
    
    // 2. 당일 + 전일 데이터 한 번에 조회 (최신순 정렬)
    List<MotivationContentDto> data = queryRepo.getMotivationContentWithFallback(
        deptName,
        headquarter,
        targetDate.minusDays(1),  // 전일
        targetDate                 // 당일
    );
    
    // 3. ✅ LEADER와 DIRECTOR 각각 최신 1건씩 추출
    return Stream.of("LEADER", "DIRECTOR")
        .map(position -> {
            // 해당 역할의 메시지 중 최신 1건
            return data.stream()
                .filter(dto -> position.equals(dto.getPosition()))
                .max(Comparator.comparing(MotivationContentDto::getMotivationDate))
                .orElse(null);
        })
        .filter(Objects::nonNull)
        .toList();
}
```

**동작 예시**:
```
targetDate = 2024-01-15

DB 데이터:
- 2024-01-15 10:00 LEADER "잘하고 있어요"
- 2024-01-14 15:00 LEADER "수고했어요"
- 2024-01-14 09:00 DIRECTOR "화이팅"

조회 결과:
- LEADER: 2024-01-15 10:00 "잘하고 있어요" (당일 최신)
- DIRECTOR: 2024-01-14 09:00 "화이팅" (당일 없으므로 전일 최신)
```

**SQL**:
```sql
SELECT
    m.motivation_id,
    m.motivation_content,
    m.motivation_date,
    m.position,
    e.employee_name AS writer
FROM crmMotivation m
LEFT JOIN crmEmployee e ON m.created_user = e.employee_id
WHERE (m.dept_name = #{deptName} OR m.dept_name = #{headquarter})
    AND m.motivation_date BETWEEN #{startDate} AND #{endDate}
ORDER BY m.motivation_date DESC, m.position;
```

---

## 📝 핵심 코드 (DTO)

```java
@Data
public class MotivationInsertDto {
    private String motivationContent;  // 원문
    private LocalDate motivationDate;
}

@Data
public class MotivationConvertDto {
    private String user_input;  // 딱딱한 원문
}

@Data
public class MotivationConvertResDto {
    private Response response;
    
    @Data
    public static class Response {
        private String beautiful_message;  // AI 변환된 친근한 메시지
    }
}

@Data
@Builder
public class MotivationContentDto {
    private Long motivationId;
    private String motivationContent;
    private LocalDate motivationDate;
    private String position;  // LEADER / DIRECTOR
    private String deptName;
    private String writerName;
}

@Data
public class UserRoleDto {
    private String deptName;        // 부서명
    private String crmMrLeaderYn;   // 팀장 여부
    private String crmHeadLeaderYn; // 본부장 여부
}
```

---

## 📊 성능 & 특징

| 항목 | 설명 |
|---|---|
| **중복 검증** | dept_name + date + position (3중) |
| **역할 판별** | crm_mr_leader_yn / crm_head_leader_yn 확인 |
| **fallback** | 당일 없으면 전일 자동 조회 |
| **AI 어조 변환** | RestTemplate + 5단계 예외처리 |

---

## 🛡️ 보안

### 1. 역할 검증
```java
if ("Y".equalsIgnoreCase(userRole.getCrmMrLeaderYn())) {
    return "LEADER";
}
```

### 2. 부서별 데이터 격리
```java
// 요청자의 부서 데이터만 처리
String userDept = employeeQueryRepo.getDeptName(userId);
if (!deptName.equals(userDept) && !isHeadquarter(userDept, deptName)) {
    throw new UnauthorizedException("타 부서 접근 불가");
}
```

---

## 🔗 관련 기술

- **Stream API**: Comparator.comparing, max, filter
- **날짜**: LocalDate, LocalDateTime
- **예외 처리**: 5단계 계층화
- **데이터 구조**: 역할별 버킷팅

---

## 📚 참고 커밋

- `동기부여 api 작업`
- `동기부여 등록 분기 처리 추가(팀별로 하루에 하나만 등록 가능)`
- `동기부여 말투 변경 작업`
- `동기부여 조회 API 추가`

