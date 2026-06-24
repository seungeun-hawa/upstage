# Document Parse · 한컴 데이터 로더 기능 비교

> 한컴 데이터 로더 컬럼의 ✅ 표기는 공식 서비스 페이지([sdk.hancom.com/services/1](https://sdk.hancom.com/services/1)) 기준.
> 6개 케이스 실측([시각 비교 페이지](https://seungeun-hawa.github.io/upstage/dps_dpe_hancom_loader_comparison.html))에서 잘 안 된 항목은 **△**로 표시.

## 1. 지원 입력 형식

| 항목 | DP (Document Parse) | 데이터 로더 (한컴) |
|---|---|---|
| 지원 파일 포맷 | PDF, 이미지(JPEG/PNG/TIFF), MS Office(DOCX/PPTX/XLSX), HWP/HWPX | HWP, HWPX, PDF + 이미지(PNG, JPG, JPEG, BMP) ※ MS Office 미지원, TIFF 미명시 |
| 지원 언어 | 한국어, 영어, 일본어, 중국어(간체) | 한국어 (한글 문서 변환 특화) |
| 최대 페이지 / 용량 | 동기 API 100p / 비동기 API 1,000p | 100MB · 1,000p 이하 권장 |
| 호출 방식 | 동기 / 비동기 | 비동기 전용 (Webhook 또는 polling) |
| 출력 포맷 | JSON (`content.{html, markdown, text}`) | JSON · CSV · HTML (이미지·차트·표 별도 저장 가능) |

## 2. 지원 기능

| 기능 | DPS | DPE | 데이터 로더 (한컴) |
|---|---|---|---|
| HWP/HWPX 네이티브 처리 | ✅ (HWPX 실측 8.4s / 15 elem) | ✅ (HWPX 실측 10.5s / 15 elem) | ✅ (HWP/HWPX 응답엔 bbox 없음) |
| 차트 내용 설명 | ❌ | ✅ DPE 전용 | △ (Figure 영역만 분리, raw text dump) |
| 이미지 내용 설명 | ❌ | ✅ DPE 전용 | ✅ (실측 미확인) |
| 복잡한 표 (병합·중첩·선 없음) | 기본 (실측 ✅ 중첩 `<table>` 보존) | 고정밀 | △ (중첩 평탄화 + `AL2년→OL2년` 등 OCR 오류) |
| 차트 데이터 추출 | 기본 | 고정밀 (실측 ✅ 시리즈명·값 매트릭스) | △ (값 매핑 불가, 텍스트 덤프) |
| 다중 페이지 걸친 표 자동 병합 | 기본 | 고정밀 | ✅ (공식 명시 X, 실측에서 줄없는 표 2개 → 1개로 병합) |
| 체크박스 인식 | 기본 | 고정밀 (실측 ✅ `<input type="checkbox">` 마크업) | – (공식 명시 X) |
| 기본 OCR (한·영·일·중 간체) | ✅ | ✅ | △ (한국어 OK, 영문 단어 공백 누락 다수: `lacksalibrary`, `theirpotential`, `ofprefix`) |
| 좌표 정보 반환 (bbox) | ✅ | ✅ | △ (PDF ✅ / HWP·HWPX ❌ — 응답 스키마에 좌표 없음) |
| Reading Order 인식 | ✅ (실측 ✅ 다단 영문 정확) | ✅ (실측 ✅) | △ (영문 다단 단어 분리 약함, HWP는 카테고리 `bodypara/subtitle/none`로 단순) |
| 단순 표 인식 | ✅ | ✅ | ✅ |
| 표 내 이미지/차트 인식 | ✅ | ✅ | ✅ (실측 미확인) |
| 머리말·꼬리말·각주·미주·페이지 번호 인식 | ✅ | ✅ | ✅ |
| 글꼴 정보 추출 (폰트/크기/볼드/정렬) | ❌ | ❌ | ✅ |
| 후보정 스튜디오·라벨링 툴 | ❌ | ❌ | ✅ |
