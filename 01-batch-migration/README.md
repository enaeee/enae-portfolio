# Spring Batch 전환 사례 - 월간 리포트 운영 자동화 및 안정성 개선

## 📌 개요
기존 DB 프로시저 기반 월간 리포트 생성 작업을  
Spring Scheduler + Service 기반 구조로 전환하고,
ShedLock을 적용하여 중복 실행을 방지했습니다.

이를 통해  
✔ 수동 개입 없이 정해진 시점에 정확히 실행  
✔ 재실행 시에도 데이터 누락/중복 없이 처리  
✔ 장애 발생 시 원인 추적이 가능한 구조로 개선했습니다.

---

## ❌ Before: DB 프로시저 의존 구조

### 문제점
- 장애 발생 시 원인 추적 어려움
- 로그 부족 → 운영 가시성 낮음
- 배치 시간 관리가 어려움
- 개발자 외에는 제어 불가능

---

## 🔄 After: Spring Batch 기반 자동화

✔ 배치 실행 시점과 대상 월이 코드로 명확하게 관리됨  
✔ 로그 기반으로 실행 여부/실패 지점 즉시 확인 가능  
✔ 서비스 로직 분리로 요구사항 변경 시 영향 범위 최소화

### 적용 코드 예시
//java
@Scheduled(cron = "0 0 7 1 * *", zone = "Asia/Seoul") // 매월 1일 07:00
@SchedulerLock(name = "Report_Task", lockAtLeastFor = "PT30M", lockAtMostFor = "PT60M")
public void generateMonthlyReport() {
    String currentMonth = getCurrentMonth(); // yyyyMM
    log.info("Starting report generation for month: {}", currentMonth);
    reportService.processReport(currentMonth);
    log.info("Report generation completed for month: {}", currentMonth);
}

-운영 서버가 여러 대여도 ShedLock으로 중복 실행 방지
-월 1회, 특정일 반복 작업을 자동화
-로그로 장애 시점/대상 월 추적 가능

//서비스 흐름
@Transactional
public void processReport(String yyyymm) {
    validateInput(yyyymm);          // 입력값/날짜 검증
    deleteData(yyyymm);             // 기존 월 데이터 정리
    processDeptData(yyyymm);        // 부서 기준 생성
    processIataPortData(yyyymm);    // IATA 기준 생성
}

- 재실행 시에도 안전하게 동작하도록 삭제 후 재생성 패턴 사용
- @Transactional로 월별 작업을 원자적으로 처리
- 기준을 명확히 세워 유지보수 용이

//db작업 예
private void insertRptRegi(String deptCode, String reportNo, String yyyymm, String dueDate, String rptType, int seq) {
    jdbcTemplate.update(
        "INSERT INTO xxx.xxxxx (a, b, yyyymm, c, d, date1, date2, sts, f) " +
        "VALUES (?, ?, to_date(?, 'YYYYMM'), ?, ?, to_date(?, 'yyyymmdd'), NOW(), 'RPTINI','')",
        deptCode, reportNo, yyyymm, rptType, seq, dueDate
    );
}

## 🛠 운영 안정성 및 유지보수 개선 효과

- 수동 실행/프로시저 의존 제거로 운영자 실수 감소
- 배치 실패 시 로그 기반으로 즉시 원인 파악 가능
- 재실행 시 데이터 누락/중복 없이 동일한 결과 보장
- 신규 요구사항 추가 시 Service 수정으로 대응 가능
