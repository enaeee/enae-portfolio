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
- 사용자가 a 또는 b 중 하나만 입력해도 나머지가 자동으로 동기화되도록 구현
- 영문 외 입력 차단 및 대문자 통일로 데이터 정합성 강화
- 선택 입력 + 직접 입력 전환(직접입력 옵션)으로 편의성 확보

```js
directChange(){
  if(this.displayParams.a === 'direct'){
    this.displayParams.a = '';
    this.directIcao = true;
  }
},
directaChange(){
  if(this.displayParams.b === 'direct'){
    this.displayParams.b = '';
    this.directIata = true;
  }
},
'displayParams.a'(value, oldVal) {
  if(!StringUtils.engCheck(value)){
    this.displayParams.a = oldVal
  }
  if (value.toUpperCase()) {
    const idx = this.aList.filter(i => i.a === value.toUpperCase());
    if (idx && idx.length === 1) {
      this.displayParams.b = idx[0].b;
    }
  } else {
    this.displayParams.b = '';
  }
},
'displayParams.b'(value, oldVal) {
  if(!StringUtils.engCheck(value)){
    this.displayParams.b = oldVal
  }
  if (value.toUpperCase()) {
    const aidx = this.aList.filter(i => i.b === value.toUpperCase());
    if (aidx && aidx.length === 1) {
      this.displayParams.a = aidx[0].a;
    }
  } else {
    this.displayParams.a = '';
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
  if (StringUtils.isEmpty(this.displayParams.a)) {
    this.$Message.alert('ICAO코드를 선택해주세요.');
    return;
  }

  // 조회 전 상태 초기화(정합성 유지)
  this.displayParams.rwydirval = '';
  this.displayParams.rwydir = '';
  this.obstRowData = [];
  this.intersectRowData = [];

  const params = lodash.cloneDeep(this.displayParams);

  // AInfo 조회
  this.$http.get(UrlPath.edit, { params }).then(res => {
    this.RowData = res.data;
  });

  // RInfo 조회(리스트/그리드)
  this.$http.get(UrlPath.edit, { params }).then(res => {
    this.rList = res.data;
    this.rRowData = res.data;
  });
},

searchObst() {
  const params = lodash.cloneDeep(this.displayParams);

  // OInfo 조회
  this.$http.get(UrlPath.Oedit, { params }).then(res => {
    this.oRowData = res.data;
  });

watch: {
  'displayParams.rval'(value) {
    if (value) {
      this.searchO(); // RWY 선택/변경 시 자동 조회
    }
  },
},

```
---

## ✅ 핵심 구현 3) 파일 업로드 + 다양한 Export
### 목표
- STX 파일 업로드를 화면에서 즉시 처리하여 운영 편의성 향상
- ICAO 단건 / 전체 / 기종별(Airbus, A220) 등 다양한 Export 요구사항을 UI에서 바로 제공

---

### 1) STX 파일 업로드 (확장자 검증 + FormData 업로드) 및 다운로드

```js
FilesearchBtn () {
  this.initFileInfo();
  document.getElementById('fileUpload').click();
},

inputValueChange () {
  this.fileName = this.$refs.file.value;

  // STX 파일만 허용
  if (!this.fileName.includes(this.fileType) && !this.fileName.includes(this.fileType.toLowerCase())) {
    this.$Message.alert('STX 파일이 아닙니다.');
    this.initFileInfo();
  }
},

initFileInfo() {
  this.$refs.file.value = '';
  this.fileName = '';
},

uploadBtn () {
  if (this.$refs.file.value === '') {
    this.$Message.alert('업로드 할 파일을 선택해주세요');
    return;
  }

  const formData = new FormData();
  formData.append('file', this.$refs.file.files[0]);

  this.$http.post(UrlPath.upload, formData, {
    headers: { 'Content-Type': 'multipart/form-data' }
  })
  .then(() => {
    this.$Message.alert('파일 업데이트를 완료하였습니다.');
  })
  .catch(err => {
    this.$Message.alert(err);
  });
},

exporticao() {
  if (StringUtils.isEmpty(this.displayParams.a)) {
    this.$Message.alert('CODE를 선택해주세요.');
    return;
  }

  const params = lodash.cloneDeep(this.displayParams);
  let fileName = 'airport.stx';

  // 파일명 규칙 생성(예: CODE_YYYY-MM-DD.stx)
  this.$http.get(UrlPath.FILENM, { params }).then(res => {
    fileName =
      res.data[0].a.substring(0, 4) + '_' +
      res.data[0].b.substring(0, 3) + '_' +
      res.data[0].chgdate.substring(0, 10) + '.stx';
  });

  this.downFileCnt++;
  this.$http({
    url: UrlPath.EXPORT,
    method: 'GET',
    responseType: 'blob',
    params: Object.assign({}, params, { format: 'stx' }),
  })
  .then(res => {
    this.downFileCnt--;

    if (window.navigator.msSaveBlob) {
      const blob = new Blob([res.data]);
      window.navigator.msSaveBlob(blob, fileName);
    } else {
      const url = window.URL.createObjectURL(new Blob([res.data]));
      const link = document.createElement('a');
      link.href = url;
      link.setAttribute('download', fileName);
      document.body.appendChild(link);
      link.click();
    }

    if (this.downFileCnt === 0) {
      this.$Message.alert('파일이 생성되었습니다.');
    }
  })
  .catch(error => {
    this.downFileCnt = 0;
    this.$Message.alert(error.message);
  });
},
```
-업로드/다운로드 기능을 화면에서 제공하여 운영자의 수작업(서버 접근/별도 처리)을 줄임
-다양한 Export 요구사항(전체/기종별/단건)을 UI에서 즉시 처리 가능



---

## ✅ 핵심 구현 4) 모달 기반 편집 플로우(이벤트 버스)

### 목표
- 그리드에서 선택한 데이터를 모달로 전달해 빠르게 수정/추가
- “선택 없음 = 추가 모드”로 동작하도록 기본값을 세팅하여 UX 단순화
- 저장 후 재조회로 화면 데이터 정합성 유지

---

### 1) 모달 오픈 → 선택 데이터 전달(수정/추가 분기)

#### 장애물(Obst) 입력 예시
```js
inputObs() {
  this.asyncObs();
},
async asyncObs() {
  await this.inputObsOn();
  this.sendObsData();
},
inputObsOn() {
  this.$bus.$emit(bus.path.InputObsInfoModal); // 모달 오픈
},

sendObsData() {
  // 선택이 없으면 "추가 모드"로 기본값 세팅
  if (this.gridApi3.getSelectedNodes().length === 0) {
    const params = lodash.cloneDeep(this.displayParams);
    params.index = -1;
    params.obslatoff = 0;
    params.rwydir = params.rwydirval;
    this.$bus.$emit(bus.path.INPUT_OBS_SELECT_PARAMS, params);
    return;
  }

  // 선택이 있으면 "수정 모드"로 데이터 전달
  this.gridApi3.forEachNode(rowNode => {
    if (rowNode.selected === true) {
      const params = lodash.cloneDeep(rowNode.data);
      params.heightval = rowNode.data.obsheight;
      params.distanceval = rowNode.data.obsdist;
      params.index = rowNode.rowIndex;

      this.$bus.$emit(bus.path.INPUT_OBS_SELECT_PARAMS, params);
      this.gridApi3.deselectAll();
    }
  });
},
```

### 모달 저장 이벤트 수신 → 서버 저장 → 재조회
```js
mounted() {
  // Obst 저장 결과 수신
  this.$bus.$on(bus.path.INPUT_OBS_PARAMS, newParams => {
    if (newParams.index !== -1) {
      // 수정
      this.$http.post(UrlPath.TODC.OBSTINFO_UP, newParams).then(() => {
        this.searchObst();
      });
    } else {
      // 추가
      this.$http.post(UrlPath.TODC.OBSTINFO_INTO, newParams).then(() => {
        this.searchObst();
      });
    }
  });
},

beforeDestroy() {
  // 화면 종료 시 이벤트 해제(메모리 누수 방지 목적)
  this.$bus.$off(bus.path.INPUT_OBS_PARAMS);
},
```
-그리드 선택 → 모달 편집 → 저장 → 재조회까지 한 흐름으로 구성되어 사용자 작업 속도 향상
-추가/수정 분기를 동일한 UI로 처리하여 사용 방법이 단순해짐
-저장 후 재조회로 화면과 서버 데이터 불일치 최소화

