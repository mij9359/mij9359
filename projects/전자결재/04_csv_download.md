# 📥 API 4: CSV 다운로드 — downloadList / downloadCsv

**난이도**: ★★★★ | **복잡도**: 메모리 관리 | **영향**: 월 100회

## 📌 개요

5만 계정 정보를 CSV로 다운로드.

## 🔴 문제점

```java
// 기존: 메모리에 전부 로드
List<UserDownloadResVO> all = userQueryRepo.selectAll();  // 5만 건 = 300MB
StringBuilder sb = new StringBuilder();
for (UserDownloadResVO v : all) {
    sb.append(v.getName()).append(",").append(v.getId()).append("\n");
}
response.write(sb.toString());  // 한 번에 전송
```

→ 5만 계정 = **~300MB 힙**  
→ TTFB = **수 초**  
→ OOM 위험

## 💡 해결방법: Streaming

```java
@GetMapping(value = "/downloadList/{type}", 
    produces = "text/csv; charset=UTF-8")
public ResponseEntity<StreamingResponseBody> downloadList(
    @PathVariable String type) {
    
    StreamingResponseBody body = out -> {
        try (OutputStreamWriter w = new OutputStreamWriter(out, UTF_8);
             CSVPrinter p = new CSVPrinter(w, 
                 CSVFormat.DEFAULT.withHeader("이름", "직원번호", "부서"))) {
            
            // 한 행씩 처리
            userQueryRepo.streamAll(rh -> {
                UserDownloadResVO v = rh.getResultObject();
                p.printRecord(
                    csvSafe(v.getUserName()),
                    v.getUserId(),
                    v.getDeptName()
                );
            });
        }
    };
    
    return ResponseEntity.ok()
        .contentType(MediaType.parseMediaType("text/csv; charset=UTF-8"))
        .header(CONTENT_DISPOSITION, "attachment;filename=users.csv")
        .body(body);
}

// CSV 특수문자 처리
private String csvSafe(String value) {
    if (value != null && (value.startsWith("=") || value.startsWith("+"))) {
        return "'" + value;
    }
    return value;
}
```

## 📊 성능

| 항목 | Before | After |
|---|---|---|
| **힙** | 300MB | < 5MB |
| **TTFB** | 수초 | < 100ms |
| **개선** | - | **99% ↓** |

