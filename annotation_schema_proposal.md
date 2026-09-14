# 도면 인식 정답지 스키마 개선 제안

**대상**: `DR_HULL Piping Diagram 인식 정답지_정보.xlsx`
**작성 배경**: DPE v2 baseline 확인 결과와 함께 검토
**목적**: 심볼 검출(detection) + 관계 추출(relation) 모델 학습·평가에 바로 쓸 수 있는 GT 포맷 정의

---

## 1. 현재 정답지 요약

| 시트 | 컬럼 | 예시 |
|---|---|---|
| FC,SW System Line List | `Line Number`, `REMARK` | `FC001-JABB3-250A` |
| FC,SW System Valve List | `VALVE NO.`, `TYPE 명칭`, `선급`, `압력`, `BORE SIZE`, `N/P내용`, `Remark` | `SW001`, `BFLY LUG`, `TR`, `5K`, `300A`, `NO.2 CARGO MACH. F.W COOLER S.W. OUTLET` |

## 2. 필요 어노테이션 포맷과의 gap

필요한 것은 **bbox + class + 관계**. 현재 시트는 다음이 없습니다.

| 항목 | 현재 상태 | 필요한 상태 |
|---|---|---|
| **bbox** | 없음 (텍스트 리스트만) | 도면 좌표계 기준 심볼·태그의 픽셀 좌표 |
| **class** | `TYPE 명칭`(BFLY LUG, GLBE STR…)에 자연어로 있음 | 고정 taxonomy로 정규화된 클래스 ID |
| **관계** | `N/P내용`에 자연어 서술 (`NO.2 CARGO MACH. F.W COOLER S.W. OUTLET`) | (subject_id, predicate, object_id) 트리플 |
| **페이지·도면 식별자** | 시트에 없음 | `dwg_no`, `page` 필드 |

## 3. 제안 스키마 (v1)

세 파일로 분리해서 저장 (JSON 또는 CSV):

### 3.1 `symbols.json` — 개별 심볼/태그의 bbox + class

```json
{
  "dwg_no": "2597 DA800D101",
  "page": 4,
  "image_size": {"w": 3310, "h": 2341},
  "objects": [
    {
      "id": "obj_0001",
      "class": "valve.butterfly_lug",
      "bbox": [x1, y1, x2, y2],
      "tag": "SW001",
      "attributes": {"pressure": "5K", "bore_size": "300A", "class_grade": "TR"}
    },
    {
      "id": "obj_0002",
      "class": "line_label",
      "bbox": [x1, y1, x2, y2],
      "tag": "FC001-JABB3-250A"
    },
    {
      "id": "obj_0100",
      "class": "equipment.cooler",
      "bbox": [x1, y1, x2, y2],
      "name": "NO.2 CARGO MACH. F.W COOLER"
    }
  ]
}
```

포인트:
- `id`는 도면 내 유니크 — 관계에서 참조
- `class`는 계층 taxonomy(예: `valve.butterfly_lug`, `valve.globe_str`, `equipment.cooler`, `equipment.pump`, `line_label`, `instrument.temp_sensor` 등)
- `tag`는 관측된 심볼 라벨 (`SW001`, `FC001-JABB3-250A`), 관계 매칭 키로도 사용
- 밸브 부속 속성(압력·사이즈)은 `attributes`로 이동

### 3.2 `relations.json` — 태그-장비 관계 트리플

```json
{
  "dwg_no": "2597 DA800D101",
  "page": 4,
  "relations": [
    {
      "id": "rel_001",
      "subject": "obj_0001",
      "predicate": "outlet_of",
      "object": "obj_0100",
      "evidence": "NO.2 CARGO MACH. F.W COOLER S.W. OUTLET"
    },
    {
      "id": "rel_002",
      "subject": "obj_0002",
      "predicate": "labels_pipe_of",
      "object": "obj_0001"
    }
  ]
}
```

`predicate` taxonomy 후보 (합의 필요):
- `inlet_of`, `outlet_of` — 밸브·라인의 장비 연결부
- `labels_pipe_of` — 라인 라벨이 참조하는 배관
- `back_flushing_of` — N/P내용의 `(FOR BACK FLUSHING)` 같은 조건
- `connects` — 일반 연결

### 3.3 `class_taxonomy.yaml` — 클래스 사전

`TYPE 명칭` 실측 목록을 정규화한 매핑표. 예:
```yaml
valve:
  butterfly_lug: {korean: "BFLY LUG", aliases: ["BFLY LUG"]}
  n_retn_flap_check: {korean: "N-RETN FLAP CHK"}
  globe_str: {korean: "GLBE STR"}
  globe_sdnr_str: {korean: "GLBE SDNR STR"}
  t_cont_3way: {korean: "T-CONT 3-WAY R/DIAP"}
  t_cont_2way_pneum: {korean: "T-CONT 2-WAY PNEUM."}
  temp_filter_water: {korean: "TEMP. FILTER(WATER)"}
  flame_arrester: {korean: "FLAME ARRESTER"}
  level_gauge_dial: {korean: "LEVEL GAUGE DIAL"}
line_label: {}
equipment:
  cooler: {}
  pump: {}
  compressor: {}
  tank: {}
```

## 4. 현재 정답지 → 제안 스키마 마이그레이션

기존 시트 두 장은 대부분 자동 변환 가능:
- `Valve List` 각 행 → `symbols.json`의 valve 객체 (bbox는 **비어있음** → 어노테이션 필요)
- `Line List` 각 행 → `symbols.json`의 `line_label` 객체 (bbox 필요)
- `N/P내용` → 규칙 기반 파싱으로 `relations.json` 초안 생성 후 사람이 검수

**남은 어노테이션 작업**: **bbox 라벨링**. 이건 새로 해야 함.

## 5. Songki님께 드릴 질문

1. bbox 라벨링 도구·인력 계획이 있는지 (CVAT/Label Studio 등)
2. `predicate` taxonomy를 위 4개로 충분히 커버 가능한지, `N/P내용`에서 놓치는 패턴이 있는지
3. `TYPE 명칭` 정규화 표를 함께 확정할 수 있는지
4. 이번 도면이 pilot이면, 이후 확장 시 `dwg_no` 부여 방식과 페이지 분할 규칙

---

## 부록: DPE v2 baseline이 이 스키마 관점에서 얼마나 나오는지

| 항목 | DPE v2 결과 |
|---|---|
| 심볼 bbox | figure 1개 (페이지 전체). 개별 심볼 bbox 0개 |
| class | figure/header/footer/caption만. 밸브·장비 클래스 0개 |
| 관계 | 자연어 캡션 안에 파트 언급 2개(`FC001`, `SW001`). 트리플 0개 |

→ DPE v2는 P&ID를 **하나의 그림**으로만 인식. 심볼 단위 baseline 산출 불가. 별도 detection 모델 또는 vision LLM 기반 파이프라인 필요.
