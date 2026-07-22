# "온톨로지에게 묻다 — SPARQL·AI 질의" Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 피란수도 부산 전시에 결정적 SPARQL 질의 섹션을 추가해, 트리플 기반 질의가 자유 해석보다 빠짐없고·다단계이며·근거 추적 가능하고·환각 대신 '없음'을 답한다는 것을 실제 SPARQL 실행으로 증명한다.

**Architecture:** 인라인 트리플 → N-Triples 변환 → 브라우저 SPARQL 엔진(스파이크로 확정, 실패 시 JS 폴백)으로 인메모리 질의. 질문 카탈로그(6종)를 UI에 노출: 자연어→실행 SPARQL→답→근거 트리플→관계망 하이라이트, 그리고 사전 생성한 AI 자유해석과 대조.

**Tech Stack:** 단일 `exhibit/index.html`(Vanilla JS), SPARQL 엔진(Comunica web 또는 Oxigraph-WASM, CDN), 기존 vis-network 재사용, 검증은 헤드리스 Chrome + PIL 크롭.

## Global Constraints

- 런타임 LLM 호출·백엔드·API 키 **금지**(정적 GitHub Pages 유지).
- 네임스페이스 `ex: <https://archivelab.co.kr/ns/busan-refuge#>` (ontology.jsonld와 동일).
- 기존 4개 시각화 로직 변경 금지 — 관계망에 하이라이트 훅 1개만 추가.
- 흰색 테마·세리프 제목(`--serif`)·Pretendard 본문·포인트색(`--point:#d9482b`) 유지.
- 이모지 미사용(미니멀 타이포그래피).
- 검증 = 헤드리스 Chrome DOM/결과 단언. 각 단계 커밋. 완료 후 라이브 배포·재검증.
- 작업 파일은 사실상 `exhibit/index.html` 단일(+ 필요 시 `assets/`의 소규모 데이터). 밀결합이므로 인라인 순차 실행(서브에이전트 병렬 편집 금지). 단, Task 6의 AI 자유해석 문구 생성만 병렬 서브에이전트 사용.

---

### Task 1: SPARQL 엔진 스파이크 (게이트)

**Files:**
- Create(임시): `exhibit/_spike.html` (검증 후 삭제)

**Interfaces:**
- Produces: 확정된 엔진 로드 방식과 `sparqlSelect(query:string):Promise<Array<Record<string,string>>>` 어댑터의 구체 구현(엔진별). 후속 Task는 이 어댑터 시그니처만 의존.

- [ ] **Step 1: Oxigraph-WASM 스파이크 작성** — `_spike.html`에 CDN(`https://cdn.jsdelivr.net/npm/oxigraph@0.4/web.js`, ESM) 로드, 한글 IRI 트리플 3개 적재, SELECT + 집계(COUNT/ORDER BY) 실행, 결과를 `document.title`에 기록.

```html
<script type="module">
import init, * as oxi from "https://cdn.jsdelivr.net/npm/oxigraph@0.4/web.js";
await init();
const store = new oxi.Store();
const NT = `<https://archivelab.co.kr/ns/busan-refuge#이중섭> <https://archivelab.co.kr/ns/busan-refuge#stayedAt> <https://archivelab.co.kr/ns/busan-refuge#영도> .
<https://archivelab.co.kr/ns/busan-refuge#피란민> <https://archivelab.co.kr/ns/busan-refuge#settledAt> <https://archivelab.co.kr/ns/busan-refuge#감천문화마을> .
<https://archivelab.co.kr/ns/busan-refuge#피란민> <https://archivelab.co.kr/ns/busan-refuge#settledAt> <https://archivelab.co.kr/ns/busan-refuge#아미동비석마을> .`;
store.load(NT, {format:"application/n-triples"});
const rows = store.query(`PREFIX ex:<https://archivelab.co.kr/ns/busan-refuge#>
  SELECT ?p (COUNT(*) AS ?c) WHERE { ?s ?p ?o } GROUP BY ?p ORDER BY DESC(?c)`);
document.title = "OXI_OK:" + rows.length + ":" + [...rows].map(r=>r.get("p").value.split("#")[1]+"="+r.get("c").value).join(",");
</script>
```

- [ ] **Step 2: 헤드리스로 실행 확인**

Run:
```bash
cd exhibit && (python3 -m http.server 8799 >/dev/null 2>&1 &) ; sleep 1
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --disable-gpu --virtual-time-budget=12000 --dump-dom "http://localhost:8799/_spike.html" 2>/dev/null | grep -o 'OXI_OK:[^<"]*' | head -1
```
Expected: `OXI_OK:2:settledAt=2,stayedAt=1` (또는 순서 무관 동일 집계). 성공 시 Oxigraph 확정.

- [ ] **Step 3: (Oxigraph 실패 시) Comunica 스파이크** — `https://rdf.js.org/comunica-browser/versions/latest/engines/query-sparql/comunica-browser.js` + N3 Store로 동일 검증. 둘 다 실패하면 **폴백 결정**(엔진 없음, JS 순회 채택)을 기록.

- [ ] **Step 4: 결정 기록·정리** — 확정 엔진(또는 폴백)을 이 파일 상단 주석과 커밋 메시지에 명기, `_spike.html` 삭제.

```bash
rm -f exhibit/_spike.html
git add -A && git commit -m "spike: 브라우저 SPARQL 엔진 확정(<엔진명 or 폴백>)"
```

---

### Task 2: RDF 빌더 + 엔진 어댑터

**Files:**
- Modify: `exhibit/index.html` (`<script>` 데이터부 뒤, 관계망 코드 앞에 삽입)

**Interfaces:**
- Consumes: `PLACES`,`EVENTS`,`TRIPLES`,`CLASS_OF`/`classOf`(관계망 정의보다 앞서면 별도 소형 분류맵 정의).
- Produces:
  - `const NS="https://archivelab.co.kr/ns/busan-refuge#"`
  - `buildNTriples():string` — 전체 데이터의 N-Triples.
  - `async function sparqlSelect(query):Promise<Array<Object>>` — 각 원소는 `{var:{value,type}}` 정규화. (Task1 확정 엔진 구현; 폴백이면 동일 시그니처의 JS 구현으로 카탈로그가 제공하는 `run()`을 대신 호출 — §Task3 note.)
  - `async function sparqlReady():Promise<void>` — 엔진 init + load 완료.

- [ ] **Step 1: N-Triples 빌더 작성** — 개체 타입(`rdf:type`), 라벨(`rdfs:label`), 장소 속성(lat/lng/placeType), **8개 사건 전부** date+occurredAt, TRIPLES 19술어(괄호 제거)를 N-Triples로. 술어·개체 IRI는 `<NS+name>`. 예:

```js
const NS="https://archivelab.co.kr/ns/busan-refuge#";
const RDF="http://www.w3.org/1999/02/22-rdf-syntax-ns#", RDFS="http://www.w3.org/2000/01/rdf-schema#", XSD="http://www.w3.org/2001/XMLSchema#";
const CLS={이중섭:"Person",김환기:"Person",윤이상:"Person",김동리:"Person",이승만:"Person",
  대한민국정부:"Group",피란민:"Group",헌책방:"Group",은지화:"Work",밀다원시대:"Work",
  정부임시수도부산이전:"Event",국제시장대화재:"Event",부산역전대화재:"Event",부산정치파동:"Event",발췌개헌:"Event"};
const clsOf=n=>CLS[n]||"Place";
const iri=n=>`<${NS}${encodeURI(n)}>`;         // 한글 IRI (스파이크에서 원형 허용 확인 시 encodeURI 생략 가능)
function buildNTriples(){
  const L=[]; const seen=new Set();
  const ent=n=>{ if(seen.has(n))return; seen.add(n);
    L.push(`${iri(n)} <${RDF}type> ${iri(clsOf(n))} .`);
    L.push(`${iri(n)} <${RDFS}label> ${JSON.stringify(n)}@ko .`.replace('"@ko','"@ko')); };
  TRIPLES.forEach(([s,p,o])=>{ ent(s); ent(o);
    const pred=p.replace(/\(.*\)/,"");
    L.push(`${iri(s)} ${iri(pred)} ${iri(o)} .`); });
  PLACES.forEach(p=>{ ent(p.name);
    L.push(`${iri(p.name)} ${iri("placeType")} ${JSON.stringify(p.type)} .`);
    L.push(`${iri(p.name)} ${iri("lat")} "${p.lat}"^^<${XSD}decimal> .`);
    L.push(`${iri(p.name)} ${iri("lng")} "${p.lng}"^^<${XSD}decimal> .`); });
  EVENTS.forEach(e=>{ const id=e.title.replace(/\s/g,""); ent(id);
    L.push(`${iri(id)} <${RDF}type> ${iri("Event")} .`);
    L.push(`${iri(id)} <${RDFS}label> ${JSON.stringify(e.title)}@ko .`);
    L.push(`${iri(id)} ${iri("date")} "${e.date}"^^<${XSD}date> .`);
    L.push(`${iri(id)} ${iri("occurredAt")} ${iri(e.place)} .`); });
  return L.join("\n");
}
```
(라벨 라인의 `@ko`는 `${iri(n)} <${RDFS}label> ${JSON.stringify(n)}@ko .` 형태로 최종 정리 — Step에서 정확 문자열로 작성.)

- [ ] **Step 2: 엔진 어댑터 작성**(Task1 확정 엔진; Oxigraph 예):

```js
let _store=null, _ready=null;
async function sparqlReady(){ if(_ready) return _ready; _ready=(async()=>{
  const oxi = await import("https://cdn.jsdelivr.net/npm/oxigraph@0.4/web.js");
  await oxi.default(); _store=new oxi.Store();
  _store.load(buildNTriples(), {format:"application/n-triples"});
})(); return _ready; }
async function sparqlSelect(q){ await sparqlReady();
  return [..._store.query(q)].map(sol=>{ const o={}; for(const [k,v] of sol) o[k]={value:v.value,type:v.termType}; return o; }); }
```

- [ ] **Step 3: 헤드리스 검증** — 임시로 `window.__t=async()=>document.title=JSON.stringify((await sparqlSelect('PREFIX ex:<'+NS+'> SELECT ?o WHERE{ex:피란민 ex:settledAt ?o}')).map(r=>r.o.value.split("#").pop()))` 호출 후 title 확인.

Run: (Task1과 동일 헤드리스 패턴) Expected title 포함: `우암동소막마을`,`아미동비석마을`,`감천문화마을` 3개.

- [ ] **Step 4: 커밋** `git commit -m "feat(query): RDF N-Triples 빌더 + SPARQL 어댑터"`

---

### Task 3: 질문 카탈로그 (6종) + '없음' 처리

**Files:**
- Modify: `exhibit/index.html` (Task2 코드 뒤)

**Interfaces:**
- Consumes: `sparqlSelect`, `NS`, `TRIPLES`.
- Produces: `const QUERIES=[{id,type,label,nl,sparql(params),params?,async run(params)->{rows,answerHTML,provenance:[[s,p,o]],empty,knownHTML?}}]` 및 파라메트릭 선택지 목록(`personList`,`placeList`).

- [ ] **Step 1: 6개 카탈로그 정의** — 각 항목에 실제 SPARQL 문자열 + 결과→answerHTML 포맷터 + provenance(답 개체와 주어를 잇는 TRIPLES 행) 도출. 파라메트릭(Q1/Q3/Q6)은 `params.subject`를 IRI로 치환. 예(Q6 부재):

```js
{ id:"q6", type:"부재", label:"창작한 작품", nl:s=>`${s}이(가) 창작한 작품은?`,
  params:{subject:{choices:()=>personList, default:"윤이상"}},
  sparql:s=>`PREFIX ex:<${NS}>\nSELECT ?work WHERE {\n  { ex:${s} ex:created ?work } UNION { ex:${s} ex:wrote ?work }\n}`,
  async run({subject}){
    const rows=await sparqlSelect(this.sparql(subject));
    if(rows.length) return {rows, answerHTML: rows.map(r=>lbl(r.work.value)).join(", "),
      provenance: TRIPLES.filter(([s,p,o])=>s===subject && /created|wrote/.test(p)), empty:false};
    // 없음: 아는 것 조회
    const known=await sparqlSelect(`PREFIX ex:<${NS}>\nSELECT ?p ?o WHERE { ex:${subject} ?p ?o FILTER(?p!=<${RDFS}label>) }`);
    return {rows:[], empty:true,
      answerHTML:`데이터에 <b>${subject}</b>의 창작 작품 관계가 <b>없습니다</b>.`,
      knownHTML:`대신 아는 것: `+known.filter(r=>!r.p.value.endsWith("type")).map(r=>`${short(r.p.value)} → ${lbl(r.o.value)}`).join(", "),
      provenance:[]};
  } }
```
(`lbl`/`short`/`lbl(iri)` 헬퍼: IRI→한글 이름/술어명. Step에서 정의.)

- [ ] **Step 2: 헬퍼·목록 정의** — `lbl(iri)=decodeURI(iri.split("#").pop())`, `short(pred)` 동일, `personList`/`placeList`는 `CLS`/`PLACES`에서.

- [ ] **Step 3: 헤드리스 검증(전 카탈로그)** — `window.__run=async(i,p)=>document.title=JSON.stringify(await QUERIES[i].run(p||{}))` 로 6개 각각 실행, 기대답 문자열 포함 확인:
  - Q1(이중섭): 원산·영도·은지화 / Q2: 3개 마을 / Q3(광복동): 김환기·밀다원다방·밀다원시대 / Q4(1953): 4건 / Q5: 부산 최상위 / Q6(윤이상): `empty:true` + 부산.

- [ ] **Step 4: 커밋** `git commit -m "feat(query): 질문 카탈로그 6종 + '없음' 처리"`

---

### Task 4: 섹션 HTML + CSS

**Files:**
- Modify: `exhibit/index.html` (nav에 링크, 관계망·언어 사이 섹션, `<style>` 규칙)

**Interfaces:**
- Produces: DOM `#query`(섹션), `#q-list`(질문 목록), `#q-result`(결과 카드), CSS 클래스 `.q-*`,`.tri`,`.contrast`.

- [ ] **Step 1: nav 링크 추가** — `<div class="nav-links">` 안 `관계망`과 `언어` 사이에 `<a href="#query">질의</a>`.

- [ ] **Step 2: 섹션 HTML 추가** — 관계망 `</section>` 뒤:

```html
<section id="query">
  <div class="sec-head">
    <div class="tag">QUERY · SPARQL · AI</div>
    <h2>온톨로지에게 묻다</h2>
    <p class="sub">질문을 고르면 <b>실제 SPARQL</b>이 그래프에 실행되어, 근거 트리플과 함께 답합니다. 같은 질문에 대한 ‘원문만 읽은 AI 자유 해석’과 나란히 비교해 보세요.</p>
    <div class="lead">트리플로 정리된 데이터는 <b>빠짐없이</b>(전수), <b>여러 단계를 따라</b>(경로), <b>근거를 밝히며</b> 답하고, 모르면 지어내지 않고 <b>‘없음’</b>이라 답합니다 — AI가 원문을 그냥 해석하는 것과의 결정적 차이입니다.</div>
  </div>
  <div class="viz q-wrap">
    <ul class="q-list" id="q-list"></ul>
    <div class="q-result" id="q-result"><div class="q-hint">왼쪽에서 질문을 선택하세요.</div></div>
  </div>
</section>
```

- [ ] **Step 3: CSS 추가**(흰 테마·세리프·포인트색 준수) — `.q-wrap{display:flex;gap:16px}` / `.q-list`(좌 260px, 클릭 목록, 유형 배지) / `.q-result`(우 flex:1) / SPARQL `pre.q-sparql`(모노, 연회색 배경, 가로 스크롤) / `.tri`(주–술→목 배지) / `.contrast`(2열: `.c-ai` 회색 / `.c-onto` 포인트색 좌측 보더) / `#query .q-hint` 힌트 / 반응형 `@media(max-width:760px)` 세로 스택.

- [ ] **Step 4: 헤드리스 렌더 확인** — 섹션 존재·nav 6링크·레이아웃 크롭 확인. 커밋 `git commit -m "feat(query): 섹션 HTML/CSS"`

---

### Task 5: 결과 카드 렌더 + 파라메트릭 UI 배선

**Files:**
- Modify: `exhibit/index.html` (Task3 뒤 스크립트)

**Interfaces:**
- Consumes: `QUERIES`, `sparqlSelect`, DOM `#q-list`,`#q-result`.
- Produces: `renderQList()`, `async selectQuery(id, params)`, `resultCardHTML(q, out)`.

- [ ] **Step 1: 목록 렌더** — `renderQList()`가 `QUERIES`로 `#q-list` 채우고(유형 배지+질문), 클릭 시 `selectQuery(id)`. 파라메트릭 항목은 카드 안에 `<select>` 노출.

- [ ] **Step 2: 결과 카드 렌더** — `selectQuery`가 `q.run(params)`를 await → 카드에 순서대로: NL 질문 · `<pre class="q-sparql">`(실행된 SPARQL) · **답**(answerHTML; empty면 '없음' 스타일 + knownHTML) · **근거 트리플**(provenance를 `.tri` 배지로) · `[관계망에서 보기]` 버튼(Task7) · 대조 패널(Task6). 파라메트릭 `<select>` change → 재실행.

- [ ] **Step 3: 헤드리스 검증** — 각 질문 클릭(프로그램적으로 `selectQuery`) 후 `#q-result`에 기대 답/SPARQL/근거 배지가 DOM에 있는지 단언(6종). Q6은 '없음' 클래스 표시 확인.

- [ ] **Step 4: 커밋** `git commit -m "feat(query): 결과 카드·파라메트릭 배선"`

---

### Task 6: AI 자유 해석 대조 (사전 생성)

**Files:**
- Modify: `exhibit/index.html` (카탈로그에 `aiFree` 상수 필드 추가 + 대조 렌더)

**Interfaces:**
- Consumes: `texts.txt` 원문(질문별 자유 응답 생성 근거), `QUERIES`.
- Produces: 각 질문의 `aiFree:{answer:string, gap:string}`(정직 라벨용) + `contrastHTML(q)`.

- [ ] **Step 1: 원문 기반 자유 응답 생성(병렬 서브에이전트)** — 6개 질문 각각, "다음 원문(texts.txt 전문)만 근거로, 이 질문에 2~3문장으로 답하라. 원문에 없으면 일반 지식으로 자연스럽게 보완하라(실제 LLM의 자유 해석을 재현)"로 6개 독립 에이전트 실행 → 각 응답 텍스트 수집. (런타임 아님; 결과를 상수로 코드에 저장.)

- [ ] **Step 2: gap(차이) 주석 작성** — 각 질문의 자유 응답이 그래프 답 대비 갖는 실제 한계(누락/혼동/추측)를 한 줄로 정직하게. 온톨로지 쪽은 "그래프가 보장: 빠짐없음/근거" 문구.

- [ ] **Step 3: 대조 렌더** — 결과 카드에 `.contrast` 2열 추가: 좌 `.c-ai`(원문만 읽은 자유 해석 + gap), 우 `.c-onto`(위 답 요약 + 보장 성질). 라벨 명확히("예시").

- [ ] **Step 4: 헤드리스 검증** — 6개 질문에 대조 2열이 모두 렌더되는지 확인. 커밋 `git commit -m "feat(query): AI 자유해석 대조 패널"`

---

### Task 7: 근거 하이라이트 — 관계망 연동

**Files:**
- Modify: `exhibit/index.html` (관계망 코드에 훅, 결과 카드 버튼 배선)

**Interfaces:**
- Consumes: `network`,`netNodes`,`netEdges`,`nodeMap`.
- Produces: `highlightSubgraph(nodeIds:string[])`, `clearHighlight()`; 결과 카드 `[관계망에서 보기]`가 provenance 노드로 호출 + 관계망으로 스크롤.

- [ ] **Step 1: 하이라이트 훅** — `highlightSubgraph(ids)`: 대상 노드는 원색 유지, 나머지 노드 `color:{opacity}`/흐리게 + 관련 없는 엣지 흐리게(vis `netNodes.update`/`netEdges.update`). `clearHighlight()` 복원. (기존 뷰 토글/필터와 충돌 없게 별도 상태.)

- [ ] **Step 2: 버튼 배선** — 결과 카드의 `[관계망에서 보기]` → provenance의 (주어·목적어) 노드 id 집합으로 `highlightSubgraph`, `#network` 스크롤. 다른 질문 선택/`clear` 시 복원.

- [ ] **Step 3: 헤드리스 검증** — Q1 실행→하이라이트 호출 후 대상 노드(이중섭·원산·영도·은지화)만 강조 상태인지 DOM/옵션 단언. 커밋 `git commit -m "feat(query): 근거 서브그래프 하이라이트"`

---

### Task 8: 전수 검증 + 배포

**Files:** (검증만; 코드 수정은 버그 발견 시)

- [ ] **Step 1: 로컬 헤드리스 전수** — HTTP 서버로 6개 질문 각각 실행: 답 정확성, SPARQL 표시, 근거 배지, '없음'(Q6), 대조 2열, 하이라이트. 콘솔 오류 0. 전체 페이지 렌더 크롭 육안 확인.
- [ ] **Step 2: 문서 갱신** — `README.md`(질의 섹션 소개), `ISSUES.md`(엔진 선택·폴백 여부·한글 IRI 처리 기록), `CREDITS.md` 불변.
- [ ] **Step 3: 커밋·푸시** — `git add -A && git commit -m "feat: 온톨로지 SPARQL·AI 질의 섹션" && git push`
- [ ] **Step 4: 라이브 검증** — Pages 빌드 대기 후 `https://archivelabedu.github.io/psu-dh2026/` 에서 질의 섹션·SPARQL 실행이 HTTPS에서 동작하는지 헤드리스로 재확인(엔진 WASM/번들 로드 포함).

---

## Self-Review

- **Spec coverage**: §3 배치→T4, §4 RDF→T2, §5 엔진/스파이크/폴백→T1·T2, §6 카탈로그→T3, §7 '없음'→T3, §8 하이라이트→T7, §9 AI 대조→T6, §10 스타일→T4, §12 검증→T8. 누락 없음.
- **Placeholder scan**: 실제 코드/SPARQL/명령 포함. 폴백 경로는 T1 결과에 따라 T2 어댑터만 JS 구현으로 대체(시그니처 동일) — 명시됨.
- **Type consistency**: `sparqlSelect(q)->[{var:{value,type}}]`가 T2 정의·T3/T5 사용에서 일치. `highlightSubgraph(ids)`/`clearHighlight()` 명칭 T7 일관. `run(params)->{rows,answerHTML,provenance,empty,knownHTML}` 형태 T3 정의·T5 소비 일치.
- **엔진 리스크**: T1 게이트가 Oxigraph→Comunica→JS폴백 순으로 확정, 후속은 어댑터 시그니처만 의존하므로 안전.
