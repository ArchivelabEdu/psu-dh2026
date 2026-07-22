# 설계: "온톨로지에게 묻다 — SPARQL·AI 질의" 섹션

작성일: 2026-07-23 · 대상: `psu2026-07/exhibit/index.html` (피란수도 부산 디지털 전시)

## 1. 목표와 논지

이번 강의의 핵심 논지: **인문 데이터를 주–술–목(SPO) 트리플로 정리하면, AI가 원문을 그냥 해석·시각화하는 것과 달리 착시·환각 없는 결과를 낼 수 있다.**

이 섹션은 그것을 실습 데이터(피란수도 부산, 25 트리플)로 *일부라도 증명*한다. 증명의 네 기둥:

1. **빠짐없음(exhaustiveness)** — 조건에 맞는 결과를 누락 없이.
2. **다단계 추론(multi-hop)** — 관계를 정확히 따라가는 경로 질의.
3. **근거 추적(provenance)** — 모든 답이 어느 트리플에서 나왔는지 제시·검증 가능.
4. **환각 대신 '없음'** — 데이터에 없으면 지어내지 않고 "없음 + 대신 아는 것"으로 응답.

## 2. 범위 / 비범위

- **범위**: 새 섹션 1개(`질의`), 인메모리 RDF 생성, 실제 SPARQL 실행, 질문 카탈로그, 근거 하이라이트(관계망 연동), AI 자유해석 대조, '없음' 처리.
- **비범위**: 런타임 LLM 호출·백엔드·API 키 없음(정적 GitHub Pages 유지). 기존 4개 섹션 로직 변경 없음(관계망에 하이라이트 훅 1개만 추가).

## 3. 배치·정보구조

- nav에 `질의` 링크 추가(관계망과 언어 사이).
- 섹션: `id="query"`, tag `QUERY · SPARQL · AI`, h2 `온톨로지에게 묻다`, 리드 서술 1개.
- 레이아웃: 좌측 **질문 목록/파라메트릭 선택**, 우측 **결과 카드**.
  - 결과 카드 구성(위→아래): 자연어 질문 · 실행된 SPARQL(모노 코드블록) · **답**(형식화) · **근거 트리플 목록** · `[관계망에서 보기]` 버튼 · **AI 자유 해석 대조**(회색) vs **온톨로지 답**(포인트색).

## 4. 데이터 → RDF 모델

로드 시 인라인 데이터(`PLACES`,`EVENTS`,`TRIPLES`)를 N-Triples로 변환해 엔진 스토어에 적재.

- 네임스페이스: `ex: <https://archivelab.co.kr/ns/busan-refuge#>` (기존 `ontology.jsonld`와 동일), `rdf:`, `rdfs:`, `xsd:`.
- 개체 IRI: `ex:<한글이름>` (예 `ex:이중섭`). 한글은 IRIREF에서 허용됨 — 스파이크에서 검증.
- 클래스: `ex:<개체> rdf:type ex:Person|Place|Event|Work|Group` (관계망의 `CLASS_OF` + "나머지는 Place" 규칙 재사용).
- 라벨: `rdfs:label "이름"@ko`.
- 술어: `TRIPLES`의 서술어에서 한글 괄호 제거 → `ex:evacuatedFrom` 등 19종.
- 장소 속성: `ex:lat`,`ex:lng`,`ex:placeType`(문자열).
- 사건: **8개 EVENTS 전부** `ex:date "YYYY-MM-DD"^^xsd:date`, `ex:occurredAt ex:<장소>`, 라벨 부여. (occurredAt이 TRIPLES에 없는 사건도 EVENTS.place로 보강 → 필터 질의 완전성 확보.)

## 5. SPARQL 엔진 (실제 실행 우선 + 폴백)

- **1순위**: 브라우저용 SPARQL 1.1 엔진 — Comunica(`@comunica/query-sparql` web 번들) 또는 Oxigraph-WASM. CDN 로드, 인메모리 스토어에 N-Triples 적재 후 `SELECT`/`ASK` 실행.
- **스파이크(구현 1단계, 게이트)**: 엔진 로드 → 한글 트리플 3개 적재 → `SELECT` 1건 실제 실행 성공 확인. 두 엔진을 시도해 하나 확정.
- **폴백(스파이크 실패 시만)**: 동일 결과를 JS 순회로 계산하고 화면엔 SPARQL을 그대로 표시. 답·근거·대조·'없음'은 완전히 동일하며 결정적. 실행 주체만 달라짐. UI에 실행 방식 배지 표기("엔진 실행" / "동일 결과·JS 계산").

## 6. 질문 카탈로그 (6종 — 각기 다른 질의 유형)

각 항목: `type · NL · SPARQL · 기대답 · 근거 · 파라메트릭 여부 · AI 자유해석 대조`.

| # | 유형 | 자연어 질문 | 기대 답(결정적) | 파라메트릭 |
|---|---|---|---|---|
| Q1 | 경로 | 〔인물〕은 어디서 피란해 무엇을 만들었나 (기본: 이중섭) | 원산→영도 머묾→은지화(영도 제작) | 인물 선택 |
| Q2 | 전수 | 피란민이 정착한 곳을 모두 | 우암동·아미동·감천 (정확히 3) | — |
| Q3 | 이웃 | 〔장소〕와 직접 연결된 개체·관계 (기본: 광복동) | 김환기·밀다원다방·밀다원시대 | 장소 선택 |
| Q4 | 필터+조인 | 1953년에 발생한 사건과 그 장소 | 국제시장대화재@국제시장 등 4건 | 연도 선택 |
| Q5 | 집계 | 가장 많은 관계로 연결된 개체 Top 3 | 부산·피란민·… (연결 수) | — |
| Q6 | 부재 | 〔인물〕이 창작한 작품은 (기본: 윤이상) | 없음 → "데이터에 없음 + 아는 것" | 인물 선택 |

대표 SPARQL 예:
```sparql
# Q1 경로(multi-hop)
PREFIX ex: <https://archivelab.co.kr/ns/busan-refuge#>
SELECT ?from ?stay ?work ?madeAt WHERE {
  ex:이중섭 ex:evacuatedFrom ?from ; ex:stayedAt ?stay ; ex:created ?work .
  OPTIONAL { ?work ex:createdAt ?madeAt . }
}
# Q5 집계(aggregation) — 관계 엣지만 세기(type/label/속성 제외)
SELECT ?n (COUNT(*) AS ?deg) WHERE {
  { ?n ?p ?o . } UNION { ?s ?p ?n . }
  FILTER(?p IN (ex:evacuatedFrom,ex:stayedAt,ex:created,ex:createdAt,ex:evacuatedTo,
    ex:workedAt,ex:frequented,ex:wrote,ex:setIn,ex:locatedIn,ex:relocatedTo,ex:usedAsOffice,
    ex:residedAt,ex:involvedIn,ex:partOf,ex:occurredAt,ex:gatheredAt,ex:settledAt,ex:formedAt))
} GROUP BY ?n ORDER BY DESC(?deg) LIMIT 3
```

## 7. '없음' 정직 응답

결과 0행이면 지어내지 않고: (1) "데이터에 그런 관계가 없습니다" 표시, (2) 대체 질의 `SELECT ?p ?o WHERE { ex:<주어> ?p ?o }`로 **아는 것**을 나열(예: 윤이상 → 부산 활동). 반환각의 핵심 시연.

## 8. 근거(provenance) 하이라이트 — 관계망 연동

- 답 산출에 쓰인 트리플의 (주어·목적어) 노드 id와 엣지를 카탈로그 항목이 함께 반환.
- 관계망 모듈에 `highlightSubgraph(nodeIds, edgeIndexes)` 훅 추가: 해당 노드·엣지만 강조, 나머지 흐리게. `clearHighlight()`로 복원.
- 결과 카드의 `[관계망에서 보기]` → 관계망으로 스크롤 + 하이라이트.

## 9. AI 자유 해석 대조 (사전 생성, 정직 라벨)

- 각 질문마다, **원문(`texts.txt`)만 준 실제 모델 응답**을 사전 생성해 정적 저장. 라벨: "원문만 읽은 자유 해석(예시)".
- 대조는 공정하게(스트로맨 금지): 자유 해석의 실제 한계만 표기 — 예) Q2 전수는 누락/집결·정착 혼동 가능, Q6 부재는 그럴듯한 작품명 추측 가능. 온톨로지 답 쪽엔 "그래프가 보장하는 성질"(빠짐없음·근거)만 명시.
- 구현 시 서브에이전트로 원문 기반 자유 응답을 생성해 카탈로그에 상수로 저장(런타임 LLM 아님).

## 10. 스타일

- 흰색 테마·세리프 제목·Pretendard 본문 유지. SPARQL은 모노스페이스 코드블록(연회색 배경).
- "온톨로지 답" 강조는 포인트색(주색), "AI 자유 해석"은 중립 회색. 근거 트리플은 `주 –술→ 목` 형태 배지.

## 11. 리스크·완화

- **엔진 통합(번들·한글 IRI)**: 스파이크 게이트로 조기 검증, 실패 시 폴백(§5).
- **집계/필터 SPARQL 지원 편차**: 엔진 확정 시 Q4·Q5까지 실행되는지 스파이크에 포함.
- **대조의 공정성**: 자유 해석 예시는 실제 생성물만, 과장 없이.

## 12. 검증

- 헤드리스 Chrome로 섹션 렌더 + 각 질문 실행 결과가 기대답과 일치하는지 DOM 단언.
- 6개 질문 각각: 답 정확성, 근거 트리플 표시, '없음' 처리(Q6), 하이라이트 동작.
- 콘솔 오류 0. 라이브 배포 후 재검증. 커밋·푸시.
