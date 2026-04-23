# 📤 API 5: Excel 사용자 등록 — addUserMany

**난이도**: ★★★★ | **복잡도**: 대용량 + 트랜잭션 | **영향**: 월 50회

## 📌 개요

Excel 파일에서 1만+ 행의 사용자를 일괄 등록.

## 🔴 문제점

```java
@Transactional  // 전체 트랜잭션
public void addUserMany(MultipartFile excelFile) {
    Workbook workbook = new XSSFWorkbook(excelFile.getInputStream());  // OOM!
    Sheet sheet = workbook.getSheetAt(0);
    
    for (int rowidx = 0; rowidx < sheet.getPhysicalNumberOfRows(); rowidx++) {
        try {
            Row row = sheet.getRow(rowidx);
            vo.setUpDeptId(getDeptIdByDeptName(
                row.getCell(2).getStringCellValue()  // NPE 위험!
            ));
            addUser(null, vo);
        } catch (ServerBizException e) {
            log.error("", e);  // 예외 삼김
        }
    }
}
```

**문제**:
- `XSSFWorkbook`: 1만 행 이상 OOM
- 헤더 스킵 없음
- NPE 위험
- 예외 삼킴 + @Transactional = 비일관적

## 💡 해결방법: SAX 스트리밍

```java
public BulkImportResult addUserMany(MultipartFile excelFile) {
    List<BulkImportError> errors = new ArrayList<>();
    int ok = 0;
    
    try (OPCPackage pkg = OPCPackage.open(excelFile.getInputStream())) {
        XSSFReader reader = new XSSFReader(pkg);  // SAX 스트리밍
        XMLReader xr = XMLReaderFactory.createXMLReader();
        
        SheetContentsHandler handler = new SheetContentsHandler() {
            int rowNum = 0;
            
            @Override
            public void startRow(int rowNum) {
                this.rowNum = rowNum;
            }
            
            @Override
            public void cell(String cellRef, String value, XSSFDataType type) {
                if (this.rowNum == 0) return;  // 헤더 스킵
                
                try {
                    UserCreateReqVO vo = parseRow(cellRef, value);
                    addUser(vo);
                    ok++;
                } catch (Exception e) {
                    errors.add(new BulkImportError(rowNum, cellRef, e.getMessage()));
                }
            }
        };
        
        xr.setContentHandler(new XSSFSheetXMLHandler(..., handler, ...));
        xr.parse(new InputSource(reader.getSheet(0)));
        
    } catch (Exception e) {
        errors.add(new BulkImportError(-1, "FILE", e.getMessage()));
    }
    
    return new BulkImportResult(ok, errors);
}
```

## 📊 효과

- SAX: 메모리 상수 (< 10MB)
- 트랜잭션 전략: 부분 커밋 + 실패 리포트
- 모든 에러 수집

