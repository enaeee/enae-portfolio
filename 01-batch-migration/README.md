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
@Scheduled(cron = "0 0 0 1 * *", zone = "Asia/Seoul") 
@SchedulerLock(
    name = "Report",
    lockAtLeastFor = "PT1M",
    lockAtMostFor = "PT30M"
)
public void runMonthlyReport() {
  String targetMonth = resolveTargetMonth(); // yyyyMM
  log.info("Monthly report started. targetMonth={}", targetMonth);
  reportService.generate(targetMonth);
  log.info("Monthly report finished. targetMonth={}", targetMonth);
}

-다중 서버 환경에서도 ShedLock으로 중복 실행 방지
-월 1회 정기 작업을 자동화하여 운영자 개입 제거
-시작/완료 로그로 장애 시점 및 대상 월 추적 가능

@Transactional
public void generate(String targetMonth) {
    validate(targetMonth);          // 입력값/날짜 검증
    cleanup(targetMonth);             // 기존 데이터 정리
    generateByGroup(targetMonth);        //  기준 생성
    generateByType(targetMonth);    // 타입 기준 생성
}

-재실행 시에도 동일한 결과를 보장하는
삭제 후 재생성 패턴 적용
-월 단위 작업을 하나의 트랜잭션으로 처리하여
부분 실패 및 데이터 불일치 방지
-처리 기준을 명확히 분리해 유지보수성 향상

private void insertRpt(
        String code,
        String no,
        String yyyymm,
        String dueDate,
        String type,
        int seq
) {
    jdbcTemplate.update(
        "INSERT INTO xxx.xxxxx (a, b, yyyymm, c, d, date1, date2, sts, f) " +
        "VALUES (?, ?, to_date(?, 'YYYYMM'), ?, ?, to_date(?, 'yyyymmdd'), NOW(), 'STATUS','')",
        code, no, yyyymm, dueDate, type, seq
    );
}

**운영 안정성 및 유지보수 개선 효과**
-수동 실행 및 DB 프로시저 의존 제거로 운영자 실수 감소
-배치 실패 시 로그 기반으로 즉시 원인 파악 가능
-재실행 시 데이터 누락/중복 없이 동일한 결과 보장
-신규 요구사항 추가 시 Service 단위 수정으로 대응 가능
