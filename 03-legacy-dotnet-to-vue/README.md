# 레거시 .NET 화면 Vue 전환
## 운항기술지원 시스템 UI 현대화 및 사용자 편의성 향상

---

## 📌 개요
기존 .NET 기반으로 운영되던 운항기술지원 시스템 화면들을 Vue 기반으로 재구성하여  
기능은 동일하게 유지하면서도 사용자가 더 빠르고 직관적으로 업무를 처리할 수 있도록 UI/UX를 개선했습니다.

그 중 **Airport 화면**은 공항(Apt)·활주로(Rwy)·장애물(Obst)·교차로(Intersect) 정보 관리,  
파일 업로드/다운로드, 외부 NOTAM 연계까지 한 화면에서 처리하는 기능이 집중되어 있어 대표 사례로 정리합니다.

---

## ❌ Before (.NET 기반 화면의 한계)
- 화면 전환/조회 시 리로드가 잦아 작업 흐름이 끊김
- 코드/입력 방식이 직관적이지 않아 사용자 실수 가능성 존재
- 조회/편집/다운로드 기능이 분산되어 업무 처리 동선이 길어짐
- 데이터 편집(추가/수정/삭제) 과정에서 반복 작업 발생

---

## 🔄 After (Vue 기반 화면 재구성)
### ✔ 개선 방향
- SPA 구조로 전환하여 화면 리로드 최소화
- 입력 실수 방지를 위한 UX 설계(직접 입력/선택 전환, 영문 검증, 대문자 통일)
- 공항/활주로/장애물/교차로 정보를 한 화면에서 조회·편집·내보내기 가능하도록 구성
- 모달 기반 편집 플로우로 업무 처리 속도 향상

---

## ✅ 핵심 구현 1) ICAO/IATA 입력 UX 개선 + 양방향 자동 매핑

### 목적
- 사용자가 ICAO 또는 IATA 중 하나만 입력해도 나머지가 자동으로 동기화되도록 구현
- 영문 외 입력 차단 및 대문자 통일로 데이터 정합성 강화
- 선택 입력 + 직접 입력 전환(직접입력 옵션)으로 편의성 확보

```js
directChange(){
  if(this.displayParams.icaoport === 'direct'){
    this.displayParams.icaoport = '';
    this.directIcao = true;
  }
},
directaChange(){
  if(this.displayParams.iataport === 'direct'){
    this.displayParams.iataport = '';
    this.directIata = true;
  }
},
'displayParams.icaoport'(value, oldVal) {
  if(!StringUtils.engCheck(value)){
    this.displayParams.icaoport = oldVal
  }
  if (value.toUpperCase()) {
    const idx = this.icaoList.filter(i => i.icaoport === value.toUpperCase());
    if (idx && idx.length === 1) {
      this.displayParams.iataport = idx[0].iataport;
    }
  } else {
    this.displayParams.iataport = '';
  }
},
'displayParams.iataport'(value, oldVal) {
  if(!StringUtils.engCheck(value)){
    this.displayParams.iataport = oldVal
  }
  if (value.toUpperCase()) {
    const aidx = this.icaoList.filter(i => i.iataport === value.toUpperCase());
    if (aidx && aidx.length === 1) {
      this.displayParams.icaoport = aidx[0].icaoport;
    }
  } else {
    this.displayParams.icaoport = '';
  }
},
```
## ✅ 핵심 구현 2) 다중 데이터 영역 한 화면 관리 (Apt/Rwy/Obst/Intersect)

### 목표
- ICAO 기반으로 공항/활주로 데이터를 한 번에 조회
- RWY 선택(또는 값 변경) 시 장애물/교차로 데이터를 자동으로 조회하여 사용자 동선을 최소화

### 구현 내용
- 검색 시 **AptInfo, RwyInfo를 동시에 조회**
- RWY 값(`rwydirval`) 변경 시 **ObstInfo, IntersectInfo를 자동 조회**
- 조회 전 불필요 데이터 초기화로 화면 정합성 유지

```js
searchBtn() {
  if (StringUtils.isEmpty(this.displayParams.icaoport)) {
    this.$Message.alert('ICAO코드를 선택해주세요.');
    return;
  }

  // 조회 전 상태 초기화(정합성 유지)
  this.displayParams.rwydirval = '';
  this.displayParams.rwydir = '';
  this.obstRowData = [];
  this.intersectRowData = [];

  const params = lodash.cloneDeep(this.displayParams);

  // AptInfo 조회
  this.$http.get(UrlPath.TODC.APTINFO_EDIT, { params }).then(res => {
    this.aptRowData = res.data;
  });

  // RwyInfo 조회(리스트/그리드)
  this.$http.get(UrlPath.TODC.RWYINFO_EDIT, { params }).then(res => {
    this.rwyList = res.data;
    this.rwyRowData = res.data;
  });
},

searchObst() {
  const params = lodash.cloneDeep(this.displayParams);

  // ObstInfo 조회
  this.$http.get(UrlPath.TODC.OBSTINFO_EDIT, { params }).then(res => {
    this.obstRowData = res.data;
  });

  // IntersectInfo 조회
  this.$http.get(UrlPath.TODC.INTERSECTINFO_EDIT, { params }).then(res => {
    this.intersectRowData = res.data;
  });
},
watch: {
  'displayParams.rwydirval'(value) {
    if (value) {
      this.searchObst(); // RWY 선택/변경 시 자동 조회
    }
  },
},

```


