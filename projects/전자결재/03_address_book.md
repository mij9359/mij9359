# 📇 API 3: 전사 주소록 — listAllCompanyAddressBook

**난이도**: ★★★★★ | **복잡도**: 한글 초성 15분기 정렬 + 대용량 | **영향**: 일일 10,000회

## 📌 개요

전사 직원 정보를 초성별로 정렬해서 조회.

## 🔴 문제점

### 1️⃣ 15-way 한글 초성 분기

```xml
<if test='"ㄱ".equals(nameFirst)'>
    AND d.name BETWEEN '가' AND '깋'
</if>
<if test='"ㄴ".equals(nameFirst)'>
    AND d.name BETWEEN '나' AND '닣'
</if>
<!-- ... ㄷ,ㄹ,ㅁ,ㅂ,ㅅ,ㅇ,ㅈ,ㅉ,ㅊ,ㅋ,ㅌ,ㅍ,ㅎ ... -->
<if test='"A".equals(nameFirst)'>
    AND d.name LIKE 'A%' OR d.name LIKE 'a%'
</if>
<!-- ... B~Z ... -->
```

→ 가독성 ↓, 유지보수 어려움

### 2️⃣ 대용량 + 페이징 부재

- 부서 1000개 × 직원 10,000명 = 100,000행
- 페이징 없이 전체 조회

## 💡 해결방법

### ✅ 1. 한글 범위 이용 + 인덱스

```xml
<choose>
    <when test='"ㄱ".equals(nameFirst)'>
        AND d.name BETWEEN '가' AND '깋'
    </when>
    <when test='"ㄴ".equals(nameFirst)'>
        AND d.name BETWEEN '나' AND '닣'
    </when>
    <!-- ... -->
</choose>
```

→ 컬럼 기반 인덱스로 성능 향상

### ✅ 2. 필수 페이징 추가

```java
// 기본 50건, 최대 100건
public List<AddressBookResVO> listAllCompanyAddressBook(
    String nameFirst,
    Pageable pageable) {  // 필수
    
    if (pageable.getPageSize() > 100) {
        pageable = PageRequest.of(
            pageable.getPageNumber(), 
            100, 
            pageable.getSort()
        );
    }
    
    return repo.find(nameFirst, pageable);
}
```

## 📊 성능

- BETWEEN: O(log n) 인덱스 활용
- 페이징: 50건 기본 (대역폭 절감)

