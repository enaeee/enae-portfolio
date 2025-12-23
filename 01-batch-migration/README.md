# Spring Batch 전환 사례 – 월간 리포트 운영 자동화 및 안정성 개선

## 📌 개요
기존 DB 프로시저 기반 월간 리포트 생성 작업을  
Spring Scheduler + Service 기반 구조로 전환하고,  
ShedLock을 적용하여 중복 실행을 방지했습니다.

이를 통해  
- 수동 개입 없이 정해진 시점에 정확히 실행  
- 재실행 시에도 데이터 누락/중복 없이 처리  
- 장애 발생 시 원인 추적이 가능한 구조로 개선했습니다.

---

## ❌ Before: DB 프로시저 의존 구조

### 문제점
- 장애 발생 시 원인 추적 어려움  
- 로그 부족 → 운영 가시성 낮음  
- 배치 시간 관리가 어려움  
- 개발자 외에는 제어 불가능  

---

## 🔄 After: Scheduler + Service 기반 자동화

### ✔ 개선 포인트
- 배치 실행 시점과 대상 월이 코드로 명확하게 관리됨  
- 로그 기반으로 실행 여부/실패 지점 즉시 확인 가능  
- 서비스 로직 분리로 요구사항 변경 시 영향 범위 최소화  

---

## 🔧 핵심 구현 1) 스케줄 트리거 + 중복 실행 방지

```java
@Scheduled(cron = "0 0 7 1 * *", zone = "Asia/Seoul") // 매월 1일 07:00
@SchedulerLock(
    name = "Report_Task",
    lockAtLeastFor = "PT30M",
    lockAtMostFor = "PT60M"
)
public void generateMonthlyReport() {
    String currentMonth = getCurrentMonth(); // yyyyMM
    log.info("Starting report generation for month: {}", currentMonth);
    reportService.processReport(currentMonth);
    log.info("Report generation completed for month: {}", currentMonth);
}

-다중 서버 환경에서도 ShedLock으로 중복 실행 방지
-월 1회 정기 작업을 자동화하여 운영자 개입 제거
-시작/완료 로그로 장애 시점 및 대상 월 추적 가능

@Transactional
public void processReport(String yyyymm) {
    validateInput(yyyymm);          // 입력값/날짜 검증
    deleteData(yyyymm);             // 기존 월 데이터 정리
    processDeptData(yyyymm);        // 부서 기준 생성
    processIataPortData(yyyymm);    // IATA 기준 생성
}

-재실행 시에도 동일한 결과를 보장하는
삭제 후 재생성(Idempotent) 패턴 적용
-월 단위 작업을 하나의 트랜잭션으로 처리하여
부분 실패 및 데이터 불일치 방지
-처리 기준을 명확히 분리해 유지보수성 향상

private void insertRptRegi(
        String deptCode,
        String reportNo,
        String yyyymm,
        String dueDate,
        String rptType,
        int seq
) {
    jdbcTemplate.update(
        "INSERT INTO xxx.xxxxx (a, b, yyyymm, c, d, date1, date2, sts, f) " +
        "VALUES (?, ?, to_date(?, 'YYYYMM'), ?, ?, to_date(?, 'yyyymmdd'), NOW(), 'RPTINI','')",
        deptCode, reportNo, yyyymm, rptType, seq, dueDate
    );
}
