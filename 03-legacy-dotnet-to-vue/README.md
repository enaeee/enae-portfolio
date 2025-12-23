# 레거시 .NET 화면 Vue 전환
## 운항기술지원 시스템 UI 현대화 및 사용자 편의성 향상

---

## 📌 개요
기존 .NET 기반으로 운영되던 운항기술지원 시스템 화면들을 Vue 기반으로 재구성하여  
기능은 동일하게 유지하면서도 사용자가 더 빠르고 직관적으로 업무를 처리할 수 있도록 UI/UX를 개선했습니다.

그 중 비행기 관련 정보 관리,  
파일 업로드/다운로드, 외부 파일 연계까지 한 화면에서 처리하는 기능이 집중되어 있어 대표 사례로 정리합니다.

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


## 모든 예시코드는 보안 정책에 따라 실제 엔드포인트/도메인 식별자는 추상화했으며, 구현 패턴 중심으로 재구성했습니다.
---

## ✅ 핵심 구현 1) A/IATA 입력 UX 개선 + 양방향 자동 매핑

### 목적
- 사용자가 a 또는 b 중 하나만 입력해도 나머지가 자동으로 동기화되도록 구현
- 영문 외 입력 차단 및 대문자 통일로 데이터 정합성 강화
- 선택 입력 + 직접 입력 전환(직접입력 옵션)으로 편의성 확보

```js
directChange(){
  if(this.displayParams.a === 'direct'){
    this.displayParams.a = '';
    this.directA = true;
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
## ✅ 핵심 구현 2) 다중 데이터 영역 한 화면 관리

### 목표
- A 기반으로 공항/활주로 데이터를 한 번에 조회
- RWY 선택(또는 값 변경) 시 장애물/교차로 데이터를 자동으로 조회하여 사용자 동선을 최소화

### 구현 내용
- 검색 시 **필요 조건을 동시에 조회**
- 특정 값 변경 시 연계 값 자동 수정
- 조회 전 불필요 데이터 초기화로 화면 정합성 유지

```js
openItemEditor() {
  this.openEditorModal();
  this.sendEditorData();
},

openEditorModal() {
  this.$bus.$emit(UI_EVENT.OPEN_MODAL); // 모달 오픈
},

sendEditorData() {
  const selected = this.gridApi3.getSelectedNodes();

  // 선택이 없으면 "추가 모드"
  if (selected.length === 0) {
    const params = lodash.cloneDeep(this.displayParams);
    params.index = -1;

    // 기본값
    params.offset = 0;
    params.subKey = params.selectedSubKey;

    this.$bus.$emit(UI_EVENT.SEND_SELECTED_PARAMS, params);
    return;
  }

  // 선택이 있으면 "수정 모드"
  const rowNode = selected[0];
  const params = lodash.cloneDeep(rowNode.data);

  // 표시용 값 매핑
  params.valueA = rowNode.data.metricA;
  params.valueB = rowNode.data.metricB;
  params.index = rowNode.rowIndex;

  this.$bus.$emit(UI_EVENT.SEND_SELECTED_PARAMS, params);
  this.gridApi3.deselectAll();
},


```
---

## ✅ 핵심 구현 3) 파일 업로드 + 다양한 Export
### 목표
- STX 파일 업로드를 화면에서 즉시 처리하여 운영 편의성 향상
- A 단건 / 전체 / 기종별 등 다양한 Export 요구사항을 UI에서 바로 제공

---

### 1) STX 파일 업로드 (확장자 검증 + FormData 업로드) 및 다운로드

```js
inputValueChange () {
  const file = this.$refs.file.files?.[0];
  if (!file) return;

  this.fileName = file.name;

  // 전용 확장자만 허용
  const allowedExt = this.allowedExt; // 예: ".dat"
  if (!this.fileName.toLowerCase().endsWith(allowedExt)) {
    this.$Message.alert('허용되지 않는 파일 형식입니다.');
    this.initFileInfo();
  }
},

uploadBtn () {
  const file = this.$refs.file.files?.[0];
  if (!file) {
    this.$Message.alert('업로드 할 파일을 선택해주세요');
    return;
  }

  const formData = new FormData();
  formData.append('file', file);

  this.$http.post(API_ENDPOINT.FILE_UPLOAD, formData, {
    headers: { 'Content-Type': 'multipart/form-data' }
  })
  .then(() => this.$Message.alert('파일 업로드를 완료했습니다.'))
  .catch(e => {
    console.error(e);
    this.$Message.alert('업로드 중 오류가 발생했습니다.');
  });
},

exportA() {
  if (StringUtils.isEmpty(this.displayParams.key)) {
    this.$Message.alert('필수 코드를 선택해주세요.');
    return;
  }

  const params = lodash.cloneDeep(this.displayParams);

  // 파일명 규칙은 서버에서 생성(추상화)
  this.$http.get(API_ENDPOINT.EXPORT_FILENAME, { params }).then(nameRes => {
    const fileName = nameRes.data || 'export.dat';

    this.$http({
      url: API_ENDPOINT.EXPORT_FILE,
      method: 'GET',
      responseType: 'blob',
      params: Object.assign({}, params, { format: 'binary' }),
    })
    .then(res => {
      const url = window.URL.createObjectURL(new Blob([res.data]));
      const link = document.createElement('a');
      link.href = url;
      link.setAttribute('download', fileName);
      document.body.appendChild(link);
      link.click();
      this.$Message.alert('파일이 생성되었습니다.');
    })
    .catch(e => {
      console.error(e);
      this.$Message.alert('파일 생성 중 오류가 발생했습니다.');
    });
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

#### 입력 예시
```js
openItemEditor() {
  this.openEditorModal();
  this.sendEditorData();
},

openEditorModal() {
  this.$bus.$emit(UI_EVENT.OPEN_MODAL); // 모달 오픈(추상화)
},

sendEditorData() {
  const selected = this.gridApi3.getSelectedNodes();

  // 선택이 없으면 "추가 모드"
  if (selected.length === 0) {
    const params = lodash.cloneDeep(this.displayParams);
    params.index = -1;

    // 기본값 세팅(추상화)
    params.offset = 0;
    params.subKey = params.selectedSubKey;

    this.$bus.$emit(UI_EVENT.SEND_SELECTED_PARAMS, params);
    return;
  }

  // 선택이 있으면 "수정 모드"
  const rowNode = selected[0];
  const params = lodash.cloneDeep(rowNode.data);

  params.valueA = rowNode.data.metricA;
  params.valueB = rowNode.data.metricB;
  params.index = rowNode.rowIndex;

  this.$bus.$emit(UI_EVENT.SEND_SELECTED_PARAMS, params);
  this.gridApi3.deselectAll();
},
```

### 모달 저장 이벤트 수신 → 서버 저장 → 재조회
```js
mounted() {
  // 저장 결과 수신
  this.$bus.$on(UI_EVENT.SAVE_EVENT, (newParams) => {
    const isUpdate = newParams.index !== -1;
    const endpoint = isUpdate ? API_ENDPOINT.UPDATE_ITEM : API_ENDPOINT.CREATE_ITEM;

    this.$http.post(endpoint, newParams)
      .then(() => this.refreshDetailList())
      .catch((e) => {
        console.error(e);
        this.$Message.alert('저장 중 오류가 발생했습니다.');
      });
  });
},

beforeDestroy() {
  this.$bus.$off(UI_EVENT.SAVE_EVENT);
},
```
-그리드 선택 → 모달 편집 → 저장 → 재조회까지 한 흐름으로 구성되어 사용자 작업 속도 향상
-추가/수정 분기를 동일한 UI로 처리하여 사용 방법이 단순해짐
-저장 후 재조회로 화면과 서버 데이터 불일치 최소화

