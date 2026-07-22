# psu2026-07 · 피란수도 부산 온톨로지 & 디지털 전시

부산대 **DH 이해하기 III — 인문 데이터로 부산(대) 그려보기** (2026.7.23–24) 워크숍용
레퍼런스 결과물. 실습 데이터 *피란수도 부산(1950–1953)* 을 **온톨로지(주–술–목 트리플)로 정리**해,
지도·연표·관계망·언어로 시각화하고 **실제 SPARQL로 직접 질의**까지 하는 완성 예시를 담았다.

> **핵심 메시지** — 인문 데이터를 트리플로 구조화하면, AI가 원문을 그냥 해석·시각화하는 것과 달리
> **빠짐없이(전수)·근거를 밝히며·환각 대신 '없음'** 으로 답할 수 있다. `질의` 섹션이 이를 실습 데이터로 증명한다.

## 🌐 라이브 전시

**https://archivelabedu.github.io/psu-dh2026/**  ← 브라우저에서 바로 열기 (GitHub Pages)

## 폴더 구성

```
psu2026-07/
├── data/                    실습 원본 데이터
│   ├── places.csv           장소 15 (id,name,lat,lng,type,desc)
│   ├── events.csv           사건 8 (id,date,title,place,desc)
│   ├── triples.csv          관계 26 (subject,predicate,object)
│   └── texts.txt            서술형 본문 (워드클라우드용)
├── ontology/
│   ├── design.md            온톨로지 4단계 설계 (개체·클래스·속성·관계)
│   └── ontology.jsonld      기계가독 스키마 + 인스턴스 (JSON-LD, 39 노드)
├── exhibit/
│   ├── index.html           디지털 전시 페이지 (단일 파일, 데이터 인라인 임베드)
│   └── assets/              대표 이미지 34점 (히어로 1 + 실사진 21 + 자체 도식 12)
├── docs/superpowers/        질의 섹션 설계 스펙 + 구현 계획
├── CREDITS.md               이미지 출처·라이선스 (Wikimedia Commons)
└── ISSUES.md                작업 이슈·결정 로그
```

> ⚠️ `index.html`은 `assets/`를 참조한다. 옮길 땐 **exhibit 폴더 통째로** 이동할 것.

## 여섯 개 섹션

| 섹션 | 데이터 | 라이브러리 | 상호작용 |
|------|--------|-----------|---------|
| 🗺️ 지도 | `places.csv` | Leaflet | 전체화면 · 마커 클릭 정보패널 · **장소 목록 / 스토리 투어** · 유형별 범례 필터 |
| 🕰️ 연표 | `events.csv` | vis-timeline | 사건 클릭 시 대표 이미지 + 위키백과 링크 |
| 🕸️ 관계망 | `triples.csv` | vis-network | 원 안 사진 노드(기본)↔점 노드 · 클래스 범례 필터 · 노드 클릭 시 제목 강조 · 살아 움직이는 배치 |
| 🔎 질의 | `triples.csv` | Oxigraph(WASM) | **실제 SPARQL 실행** · 자연어 답 → 근거 트리플·관계망 하이라이트 · '없음' 정직 응답 · AI 자유해석 대조 · **빈칸 채우기 빌더 + 편집·실행 플레이그라운드** |
| ☁️ 언어 | `texts.txt` | wordcloud2.js | 호버 시 빈도수 · 워드클라우드/막대그래프/히트맵 3뷰 |
| 📖 해설 | 종합 | — | 네 시선을 잇는 서술 + 요약 카드 |

> CDN(unpkg·jsDelivr)·CARTO 지도 타일·Google Fonts를 사용하므로 **첫 실행 시 인터넷 연결**이 필요하다
> (특히 SPARQL 엔진·웹폰트). 대표 이미지는 `assets/`의 로컬 파일이라 오프라인에서도 표시된다.

## 🔎 질의(SPARQL) — 이 프로젝트의 핵심

트리플 기반 질의가 자유 해석과 어떻게 다른지 실습 데이터로 *증명*하는 섹션.

- **실제 SPARQL 1.1 실행** — 인라인 트리플 → N-Triples → 브라우저 SPARQL 엔진(Oxigraph-WASM). 한글 IRI 그대로 사용.
- **큐레이션 질문 6종** — 경로(다단계)·전수·이웃·필터조인·집계·**부재(환각 대신 '없음')**. 인물/장소 파라메트릭.
- **근거·검증** — 답의 출처 트리플을 표시하고 관계망에서 해당 서브그래프를 강조. 실행된 SPARQL을 그대로 노출.
- **AI 대조** — 원문만 준 자유 해석 vs 온톨로지 답을 나란히(누락·과잉·외부지식 환각을 정직하게 표기, 백엔드/키 없음).
- **플레이그라운드** — 빈칸 채우기 드롭다운 빌더 · 직접 SPARQL 편집·실행 · 문법 도움말(예시 9종).

## 온톨로지 설계 요약

- **클래스**: `Agent`(→ `Person`/`Group`) · `Place` · `Event` · `Work`
- **속성**(data property): `name·lat·lng·placeType·description`(장소), `date·title·description`(사건)
- **관계**(object property): `evacuatedFrom·stayedAt·created·workedAt·occurredAt·gatheredAt·settledAt` 등 19종

자세한 내용은 [`ontology/design.md`](ontology/design.md), 형식 스키마는 [`ontology/ontology.jsonld`](ontology/ontology.jsonld) 참고.

## 데이터 · 이미지 출처

- **실습 데이터**는 공개 참고자료(위키백과 · 한국민족문화대백과사전 · 부산역사문화대전)를 바탕으로 **교육용으로 정리·단순화**한 것으로, 좌표·일부 정보는 근사값이다.
- **대표 이미지**는 Wikimedia Commons의 PD·CC0·CC BY(-SA)로 확인된 것만 사용하고, 없는 항목은 자체 SVG 도식으로 대체했다. 전체 목록·라이선스는 [`CREDITS.md`](CREDITS.md) 및 사이트 푸터.
- 각 개체(인물·장소·사건)의 **근거 문서 링크**는 사이트 푸터와 각 항목 상세·캡션에서 연결된다.

## 활용

- **시연 레퍼런스**: 학생들에게 "이 정도 완성도"를 보여주는 목표 예시
- **실습 스캐폴드**: 데이터·설계·시각화 구조를 두고 주제/데이터만 교체
- **바이브 코딩 출발점**: `index.html`의 각 섹션이 곧 프롬프트의 완성형 결과

> 데이터는 교육용 근사값이다. 실제 프로젝트에서는 각 트리플에 개별 `source`(출처) 속성을 기록하라.

## 개발 메모

- 브레인스토밍 → 설계 스펙 → 구현 계획 순서로 진행: [`docs/superpowers/specs`](docs/superpowers/specs) · [`docs/superpowers/plans`](docs/superpowers/plans).
- 디자인: 히어로 배경(1952 부산항, CC0)에 **켄 번스** 효과, 헤드라인 **Nanum Myeongjo**(세리프) + 본문 **Pretendard**, 포인트색 미니멀 로고, 흰 테마.
- 배포: GitHub Pages(main 루트, `.nojekyll`), 루트 `index.html`이 `exhibit/`로 리다이렉트.
