# DPS · DPE · Hancom 데이터로더 비교

3종 데이터로더의 동일 입력 6개 샘플 실측 결과 + 케이스별 추천.

- **DPS** : Upstage Document Parse Standard (`mode=standard`)
- **DPE** : Upstage Document Parse Enhanced (`mode=enhanced`, VLM 기반)
- **Hancom** : 한컴 데이터로더 (`api.sdk.hancom.com`, 비동기 polling)

상세 비교 페이지 (요소별 시각화): https://seungeun-hawa.github.io/upstage/dps_dpe_hancom_loader_comparison.html

---

## 1. 실측 성능

| # | 샘플 | 입력 | DPS (ms / elem) | DPE (ms / elem) | Hancom (ms / elem) |
|---|---|---|---|---|---|
| 1 | 차트·통계표 (2단 PDF) | PDF | 6,491 / 20 | 21,624 / 20 | 22,958 / 17 |
| 2 | 줄 없는 표 (연속 2개) | PDF | 8,432 / 6 | 6,433 / 6 | 60,125 / 6 |
| 3 | 회전 + 처방전 양식 | PNG (한컴은 PDF wrap) | 7,508 / 3 | 51,903 / 3 | 35,112 / 10 |
| 4 | 중첩 표 (표 안의 표) | PNG (한컴은 PDF wrap) | 15,500 / 3 | 14,784 / 3 | 60,162 / 3 |
| 5 | 영문 다단 (Reading Order) | PNG (한컴은 PDF wrap) | 3,382 / 20 | 4,497 / 20 | 60,038 / 20 |
| 6 | HWPX 네이티브 지원 | HWPX | 8,415 / 15 | 10,466 / 15 | 16,389 / body 155 (텍스트 138) |

> **속도**: DPS가 평균 가장 빠름(3–15초). DPE는 VLM 추론으로 케이스별 편차 큼(회전 양식 52초). Hancom은 비동기 큐 대기 포함이라 30초~1분이 기본.
>
> **응답 스키마**: PDF/이미지/HWPX 입력에서는 3종 모두 `elements[]` 구조 + bbox. **단 Hancom의 HWPX는 `body[].contents.text` + `posInfo.docPageNum`으로 완전히 다른 스키마이고 bbox가 없음.**

---

## 2. 케이스별 추천

| # | 케이스 | 추천 | 이유 |
|---|---|---|---|
| 1 | 차트·통계표 (2단 PDF) | **DPE** (차트) / **Hancom** (본문) | DPE는 차트 시리즈명·값까지 매트릭스화. Hancom은 줄바꿈된 문장을 한 element에 묶어서 RAG 청킹에 유리. DPS/DPE는 본문이 단편으로 쪼개지고 "증가함"을 `heading1`로 오분류. |
| 2 | 줄 없는 표 (연속 2개) | **DPE** | DPS와 같은 셀 구조 + `<th scope="col/colgroup">` 시맨틱. 연속 표 자동 병합 우선이면 Hancom. |
| 3 | 회전 + 처방전 양식 | **DPE** | enhanced VLM이 표 텍스트 28k자 / HTML 80k자로 DPS의 14배 정밀. 처방전 본문·체크박스까지 마크업. |
| 4 | 중첩 표 (표 안의 표) | **DPS** | 중첩 `<table>` 구조 그대로 보존. Hancom은 평탄화돼서 거대 병합 + 'AL2년→OL2년' 같은 OCR 오류. |
| 5 | 영문 다단 (Reading Order) | **DPS** | reading order + 영문 단어 분리 정확. Hancom은 'lacksalibrary', 'theirpotential' 등 공백 누락 심각. |
| 6 | HWPX 네이티브 지원 | **DPS/DPE** | HWPX 직접 처리 확인. PDF와 동일 스키마 + bbox + 카테고리 세분화. Hancom은 body 155개로 단락 세밀하지만 bbox 없음. |

---

## 3. 요약 결론

- **일반 PDF / 빠른 처리** → **DPS**. 3–8초로 안정적, bbox·카테고리 세분화 양호.
- **차트·복잡 표·복잡 양식이 핵심** → **DPE**. VLM 추론으로 정밀도 최상, 비용·지연(최대 52초) 감수.
- **HWP/HWPX 네이티브 + RAG용 단락 무결성** → **Hancom** (단, HWPX 응답에 bbox 없음 → 좌표 기반 시각화/하이라이트 불가).
- **영문·이미지 단독·중첩 구조** → DPS/DPE가 명확히 우위.

---

## 4. 한컴 데이터로더 특이사항

- **입력**: PDF · HWP · HWPX 위주. 이미지/MS Office 직접 지원 X → **PDF로 래핑 필요**.
- **호출**: 비동기 전용. `/convert` → polling `/status` → `/download` (zip). `webhook_url` 파라미터 필수 (polling만 써도 더미 URL 전달).
- **HWPX 응답**: `body[]` 구조 + `posInfo.docPageNum`. **bbox 없음**, 카테고리는 `bodypara/subtitle/none` 수준.
- **PDF/이미지 응답**: `elements[]` 구조 + `bbox{left,top,width,height}` (포인트 단위) + `pageSizes`. DocTitle/ListText/FigureName/Figure/Table 등 세분화 카테고리.
- **가격**: 페이지당 10 Credit, 실패 시 환불 (공식 문서 기준).

---

자세한 element별 시각 비교는 [comparison 페이지](https://seungeun-hawa.github.io/upstage/dps_dpe_hancom_loader_comparison.html)에서.
