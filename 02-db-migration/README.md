# 운항기술지원 시스템 DB 마이그레이션
## Oracle → PostgreSQL 전환 및 데이터 정확성 개선

---

## 📌 개요
기존 Oracle 기반 운항기술지원 시스템을  
PostgreSQL 환경으로 전환하면서  
데이터 타입, 날짜 처리, 쿼리 문법 차이로 인한  
**데이터 누락·오차 가능성**을 제거하는 데 집중했습니다.

마이그레이션 이후에도  
- 기존 데이터와 결과가 동일하게 조회되도록 검증  
- SQL 표준화로 유지보수성과 가독성 향상  
을 목표로 작업을 수행했습니다.

---

## ❌ Before: Oracle 중심 구조의 한계

### 문제점
- Oracle 전용 함수(`TO_CHAR`, `SYSDATE` 등)에 의존
- 날짜/문자 타입 혼용으로 쿼리 가독성 저하
- DB 변경 시 전체 SQL 수정 필요
- 데이터 비교/검증 로직 부재

---

## 🔄 After: PostgreSQL 기준 SQL 표준화

### ✔ 개선 방향
- DB 의존 함수 최소화
- 날짜/시간 타입 명확화
- SQL 가독성 및 재사용성 향상
- 결과 값 기준 검증으로 정확성 확보

---

## 🔧 핵심 구현 1) 날짜/시간 처리 방식 개선

### Before (Oracle)
```sql
SELECT *
FROM FLIGHT_LOG
WHERE TO_CHAR(FLIGHT_DATE, 'YYYYMMDD') >= TO_CHAR(SYSDATE - 7, 'YYYYMMDD');
```

### After (PostgreSql)
```sql
SELECT *
FROM flight_log
WHERE flight_date >= CURRENT_DATE - INTERVAL '7 days';
```

-문자열 변환 제거 → 성능 및 가독성 향상
-DB 변경 시 영향 최소화
-날짜 비교 로직 명확화

### 운영 안정성 및 유지보수 개선 효과
-DB 변경에도 SQL 수정 범위 최소화
-데이터 누락/오차 없이 결과 동일성 확보
-쿼리 가독성 향상으로 유지보수 용이
-신규 기능 개발 시 DB 의존도 감소

### 배운 점
- 마이그레이션에서 중요한 것은 속도보다 정확성
- 결과 검증 필수
- sql 표준화는 장기유지보수에 유리함

