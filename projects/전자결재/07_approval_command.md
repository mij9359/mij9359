# 📋 API 7: 결재 승인/반려 — ApprovalCommandServiceImpl

**난이도**: ★★★★ | **복잡도**: race condition | **영향**: 일일 5,000회

## 📌 개요

결재 문서를 승인/반려 처리.

## 🔴 문제점

```java
@Transactional
public void approve(ApprovalReqVO req) {
    // SELECT
    DocumentApproval da = repo.findByDocAndOrder(...);
    if (da.getApprovalState() != 0) {
        throw new ServerBizException(ALREADY_PROCESSED);
    }
    
    // UPDATE (SELECT-UPDATE 사이 락 없음!)
    updateApprovalState(docNo, order, 1);
    
    notifyNextApprover(docNo);
}
```

**Race Condition**:
```
Thread A (결재자)        Thread B (대리인, 동시)
SELECT state=0 ✓
                         SELECT state=0 ✓
UPDATE state=1 ✓
                         UPDATE state=1 ✓  (덮어쓰기)
notify() ✓
                         notify() ✓  (중복)
```

## 💡 해결방법: 조건부 UPDATE

```sql
<!-- ApprovalCommand.xml -->
<update id="approveIfWaiting">
    UPDATE document_approval
       SET approval_state = 1,
           approval_at = NOW(),
           approval_user_cd = #{userCd}
     WHERE doc_no = #{docNo}
       AND approval_order = #{approvalOrder}
       AND approval_state = 0  <!-- 조건부 -->
</update>
```

```java
@Transactional
public void approve(ApprovalReqVO req) {
    // 한 번에 조건 확인 + UPDATE
    int affected = repo.approveIfWaiting(req);
    
    if (affected == 0) {
        // 이미 처리됨
        DocumentApproval cur = queryRepo.findByDocAndOrder(req);
        String error = alreadyProcessedError(cur.getApprovalState());
        throw new ServerBizException(error);
    }
    
    // affectedRows=1 보장 → 단 1회만 실행
    notifyNextApprover(req.getDocNo());
}
```

## 📊 효과

| 시나리오 | Before | After |
|---|---|---|
| **동시 결재** | 이중 승인 | 1건만 성공 |
| **알림** | 2회 | 1회 |
| **락** | 필요 | 불필요 |

