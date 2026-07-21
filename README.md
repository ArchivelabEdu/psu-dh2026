# psu2026-07 · 피란수도 부산 온톨로지 & 디지털 전시

부산대 **DH 이해하기 III — 인문 데이터로 부산(대) 그려보기** (2026.7.23–24) 워크숍용
레퍼런스 결과물. 실습 데이터 *피란수도 부산(1950–1953)* 으로 **온톨로지 4단계 설계 →
지도·타임라인·관계망·워드클라우드 시각화**의 완성 예시를 담았다.

## 🌐 라이브 전시

**https://archivelabedu.github.io/psu-dh2026/**  ← 브라우저에서 바로 열기 (GitHub Pages)

## 폴더 구성

```
psu2026-07/
├── data/                    실습 원본 데이터 (uploads에서 복사)
│   ├── places.csv           장소 — 지도용 (id,name,lat,lng,type,desc)
│   ├── events.csv           사건 — 타임라인용 (id,date,title,place,desc)
│   ├── triples.csv          관계 — 관계망용 (subject,predicate,object)
│   └── texts.txt            서술형 본문 — 워드클라우드용
├── ontology/
│   ├── design.md            온톨로지 4단계 설계 문서 (개체·클래스·속성·관계)
│   └── ontology.jsonld      기계가독 스키마 + 인스턴스 (JSON-LD, 39개 노드)
├── exhibit/
│   ├── index.html           디지털 전시 페이지 (상호작용 4종 + 종합 해설)
│   └── assets/              대표 이미지 33개 (실사진 21 + SVG 도식 12)
├── CREDITS.md               이미지 출처·라이선스 (Wikimedia Commons)
└── ISSUES.md                작업 이슈·결정 로그
```

> ⚠️ `index.html`은 `assets/` 폴더를 참조합니다. 옮길 때 **exhibit 폴더 통째로** 이동하세요.

## 전시 페이지 열기

`exhibit/index.html`을 **브라우저에서 직접 열면** 된다(더블클릭 또는 드래그).
데이터가 파일 안에 인라인 임베드되어 있어 서버·빌드 없이 `file://`로 바로 동작한다.

네 가지 시각화 + 종합 해설이 한 페이지에 담긴다:

| 섹션 | 데이터 | 라이브러리 | 상호작용 |
|------|--------|-----------|---------|
| 🗺️ 지도 | `places.csv` | Leaflet | 전체화면 · 마커 클릭 시 대표 이미지·정보 패널 · 유형별 범례 필터 |
| 🕰️ 연표 | `events.csv` | vis-timeline | 사건 클릭 시 대표 이미지 + 위키백과 링크 |
| 🕸️ 관계망 | `triples.csv` | vis-network | 원 안 사진 노드(기본)↔점 노드 전환 · 클래스 범례 필터 · 살아 움직이는 배치 |
| ☁️ 언어 | `texts.txt` | wordcloud2.js | 호버 시 빈도수 · 워드클라우드/막대그래프/히트맵 3뷰 |
| 📖 해설 | 종합 | — | 네 시선을 잇는 서술 + 요약 카드 |

> 라이브러리는 CDN(unpkg)에서, 지도 타일은 CARTO에서 불러오므로 **첫 실행 시 인터넷 연결**이 필요하다.
> 대표 이미지는 `assets/`의 로컬 파일이라 오프라인에서도 표시된다.

## 온톨로지 설계 요약

- **클래스**: `Agent`(→ `Person`/`Group`) · `Place` · `Event` · `Work`
- **속성**(data property): `name·lat·lng·placeType·description`(장소), `date·title·description`(사건)
- **관계**(object property): `evacuatedFrom·stayedAt·created·workedAt·occurredAt·gatheredAt·settledAt` 등 19종

자세한 내용은 [`ontology/design.md`](ontology/design.md), 형식 스키마는
[`ontology/ontology.jsonld`](ontology/ontology.jsonld) 참고.

## 활용

- **시연 레퍼런스**: 학생들에게 "이 정도 완성도"를 보여주는 목표 예시
- **실습 스캐폴드**: 데이터·설계·시각화 구조를 그대로 두고 주제/데이터만 교체
- **바이브 코딩 출발점**: `index.html`의 각 섹션이 곧 "지도/타임라인/네트워크/워드클라우드"
  프롬프트의 완성형 결과

> 데이터는 교육용 근사값이다. 실제 프로젝트에서는 각 개체에 `source`(출처) 속성을 추가하라.
