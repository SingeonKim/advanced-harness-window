**중요: 아래 단계를 순서대로 완료해야 합니다. 코드 작성을 먼저 하지 마세요.**

PDF 양식을 작성해야 할 경우, 먼저 PDF에 입력 가능한 양식 필드가 있는지 확인하세요. 이 파일의 디렉토리에서 스크립트를 실행하세요:
`python scripts/check_fillable_fields <file.pdf>`, 결과에 따라 "입력 가능한 필드" 또는 "입력 불가능한 필드" 섹션으로 이동하여 해당 지침을 따르세요.

# 입력 가능한 필드

PDF에 입력 가능한 양식 필드가 있는 경우:
- 이 파일의 디렉토리에서 스크립트를 실행하세요: `python scripts/extract_form_field_info.py <input.pdf> <field_info.json>`. 이 형식으로 필드 목록이 담긴 JSON 파일을 생성합니다:
```
[
  {
    "field_id": (필드의 고유 ID),
    "page": (페이지 번호, 1부터 시작),
    "rect": ([left, bottom, right, top] PDF 좌표의 경계 상자, y=0은 페이지 하단),
    "type": ("text", "checkbox", "radio_group", 또는 "choice"),
  },
  // 체크박스에는 "checked_value"와 "unchecked_value" 속성이 있습니다:
  {
    "field_id": (필드의 고유 ID),
    "page": (페이지 번호, 1부터 시작),
    "type": "checkbox",
    "checked_value": (체크박스 선택을 위해 이 값으로 설정),
    "unchecked_value": (체크박스 해제를 위해 이 값으로 설정),
  },
  // 라디오 그룹에는 가능한 선택지 목록이 있는 "radio_options"가 있습니다.
  {
    "field_id": (필드의 고유 ID),
    "page": (페이지 번호, 1부터 시작),
    "type": "radio_group",
    "radio_options": [
      {
        "value": (이 라디오 옵션 선택을 위해 이 값으로 설정),
        "rect": (이 옵션의 라디오 버튼 경계 상자)
      },
      // 다른 라디오 옵션
    ]
  },
  // 다중 선택 필드에는 가능한 선택지 목록이 있는 "choice_options"가 있습니다:
  {
    "field_id": (필드의 고유 ID),
    "page": (페이지 번호, 1부터 시작),
    "type": "choice",
    "choice_options": [
      {
        "value": (이 옵션 선택을 위해 이 값으로 설정),
        "text": (옵션의 표시 텍스트)
      },
      // 다른 선택 옵션
    ],
  }
]
```
- 이 스크립트로 PDF를 PNG로 변환하세요 (각 페이지당 하나의 이미지, 이 파일의 디렉토리에서 실행):
`python scripts/convert_pdf_to_images.py <file.pdf> <output_directory>`
그런 다음 이미지를 분석하여 각 양식 필드의 목적을 파악하세요 (경계 상자 PDF 좌표를 이미지 좌표로 변환해야 합니다).
- 각 필드에 입력할 값이 담긴 `field_values.json` 파일을 이 형식으로 생성하세요:
```
[
  {
    "field_id": "last_name", // extract_form_field_info.py의 field_id와 일치해야 함
    "description": "사용자의 성",
    "page": 1, // field_info.json의 "page" 값과 일치해야 함
    "value": "Simpson"
  },
  {
    "field_id": "Checkbox12",
    "description": "18세 이상인 경우 선택하는 체크박스",
    "page": 1,
    "value": "/On" // 체크박스인 경우, 선택을 위해 "checked_value"를 사용. 라디오 버튼 그룹인 경우 "radio_options"의 "value" 중 하나 사용.
  },
  // 더 많은 필드
]
```
- 이 파일의 디렉토리에서 `fill_fillable_fields.py` 스크립트를 실행하여 작성된 PDF를 생성하세요:
`python scripts/fill_fillable_fields.py <input pdf> <field_values.json> <output pdf>`
이 스크립트는 제공한 필드 ID와 값이 유효한지 확인합니다. 에러 메시지가 출력되면 해당 필드를 수정한 후 다시 시도하세요.

# 입력 불가능한 필드

PDF에 입력 가능한 양식 필드가 없는 경우, 텍스트 주석을 추가합니다. 먼저 PDF 구조에서 좌표를 추출해 보고(더 정확), 필요한 경우 시각적 추정으로 넘어가세요.

## 1단계: 구조 추출 먼저 시도

이 스크립트를 실행하여 텍스트 레이블, 선, 체크박스와 정확한 PDF 좌표를 추출하세요:
`python scripts/extract_form_structure.py <input.pdf> form_structure.json`

다음 내용이 담긴 JSON 파일이 생성됩니다:
- **labels**: 정확한 좌표가 있는 모든 텍스트 요소 (x0, top, x1, bottom, PDF 포인트 단위)
- **lines**: 행 경계를 정의하는 수평선
- **checkboxes**: 체크박스인 작은 사각형 (중심 좌표 포함)
- **row_boundaries**: 수평선으로 계산된 행의 상단/하단 위치

**결과 확인**: `form_structure.json`에 의미 있는 레이블(양식 필드에 해당하는 텍스트 요소)이 있으면 **접근 방식 A: 구조 기반 좌표**를 사용하세요. PDF가 스캔/이미지 기반이고 레이블이 거의 없으면 **접근 방식 B: 시각적 추정**을 사용하세요.

---

## 접근 방식 A: 구조 기반 좌표 (권장)

`extract_form_structure.py`에서 텍스트 레이블을 찾은 경우 사용합니다.

### A.1: 구조 분석

form_structure.json을 읽고 다음을 파악하세요:

1. **레이블 그룹**: 단일 레이블을 형성하는 인접 텍스트 요소 (예: "Last" + "Name")
2. **행 구조**: 비슷한 `top` 값의 레이블은 같은 행
3. **필드 열**: 입력 영역은 레이블 끝 이후에 시작 (x0 = label.x1 + gap)
4. **체크박스**: 구조의 체크박스 좌표 직접 사용

**좌표 시스템**: y=0이 페이지 상단이고 y가 아래로 증가하는 PDF 좌표.

### A.2: 누락 요소 확인

구조 추출이 모든 양식 요소를 감지하지 못할 수 있습니다. 일반적인 경우:
- **원형 체크박스**: 사각형만 체크박스로 감지됨
- **복잡한 그래픽**: 장식 요소 또는 비표준 양식 컨트롤
- **희미하거나 밝은 색상 요소**: 추출되지 않을 수 있음

form_structure.json에 없는 양식 필드가 PDF 이미지에 보이면 해당 필드에 대해 **시각적 분석**을 사용해야 합니다 (아래 "혼합 접근 방식" 참조).

### A.3: PDF 좌표로 fields.json 생성

각 필드에 대해 추출된 구조에서 입력 좌표를 계산하세요:

**텍스트 필드:**
- entry x0 = label x1 + 5 (레이블 이후 작은 간격)
- entry x1 = 다음 레이블의 x0, 또는 행 경계
- entry top = 레이블 top과 동일
- entry bottom = 아래 행 경계선, 또는 label bottom + row_height

**체크박스:**
- form_structure.json의 체크박스 사각형 좌표 직접 사용
- entry_bounding_box = [checkbox.x0, checkbox.top, checkbox.x1, checkbox.bottom]

`pdf_width`와 `pdf_height`를 사용하여 fields.json 생성 (PDF 좌표 신호):
```json
{
  "pages": [
    {"page_number": 1, "pdf_width": 612, "pdf_height": 792}
  ],
  "form_fields": [
    {
      "page_number": 1,
      "description": "성 입력 필드",
      "field_label": "Last Name",
      "label_bounding_box": [43, 63, 87, 73],
      "entry_bounding_box": [92, 63, 260, 79],
      "entry_text": {"text": "Smith", "font_size": 10}
    },
    {
      "page_number": 1,
      "description": "미국 시민권자 예 체크박스",
      "field_label": "Yes",
      "label_bounding_box": [260, 200, 280, 210],
      "entry_bounding_box": [285, 197, 292, 205],
      "entry_text": {"text": "X"}
    }
  ]
}
```

**중요**: `pdf_width`/`pdf_height`를 사용하고 form_structure.json의 좌표를 직접 사용하세요.

### A.4: 경계 상자 검증

작성 전에 경계 상자 에러를 확인하세요:
`python scripts/check_bounding_boxes.py fields.json`

교차하는 경계 상자와 폰트 크기에 비해 너무 작은 입력 상자를 확인합니다. 작성 전에 보고된 에러를 수정하세요.

---

## 접근 방식 B: 시각적 추정 (대체 방법)

PDF가 스캔/이미지 기반이고 구조 추출이 사용 가능한 텍스트 레이블을 찾지 못한 경우(예: 모든 텍스트가 "(cid:X)" 패턴으로 표시) 사용합니다.

### B.1: PDF를 이미지로 변환

`python scripts/convert_pdf_to_images.py <input.pdf> <images_dir/>`

### B.2: 초기 필드 식별

각 페이지 이미지를 검토하여 양식 섹션과 필드 위치의 **대략적인 추정값**을 파악하세요:
- 양식 필드 레이블과 대략적인 위치
- 입력 영역 (텍스트 입력을 위한 선, 상자, 또는 빈 공간)
- 체크박스와 대략적인 위치

각 필드에 대해 대략적인 픽셀 좌표를 기록하세요 (아직 정확할 필요 없음).

### B.3: 확대 세밀화 (정확도를 위해 중요)

각 필드에 대해 추정 위치 주변 영역을 잘라내어 좌표를 정확하게 세밀화하세요.

**ImageMagick을 사용한 확대 자르기:**
```bash
magick <page_image> -crop <width>x<height>+<x>+<y> +repage <crop_output.png>
```

여기서:
- `<x>, <y>` = 자르기 영역의 좌상단 모서리 (대략적 추정값에서 여백 빼기)
- `<width>, <height>` = 자르기 영역 크기 (필드 영역 + 각 방향 ~50px 여백)

**예시:** 약 (100, 150)으로 추정되는 "Name" 필드 세밀화:
```bash
magick images_dir/page_1.png -crop 300x80+50+120 +repage crops/name_field.png
```

(`magick` 명령이 없으면 동일한 인수로 `convert`를 시도하세요).

**자른 이미지를 검토**하여 정확한 좌표를 파악하세요:
1. 입력 영역이 정확히 어디서 시작하는지 파악 (레이블 이후)
2. 입력 영역이 어디서 끝나는지 파악 (다음 필드 또는 가장자리 이전)
3. 입력 선/상자의 상단과 하단 파악

**자르기 좌표를 전체 이미지 좌표로 변환:**
- full_x = crop_x + crop_offset_x
- full_y = crop_y + crop_offset_y

예시: 자르기가 (50, 120)에서 시작하고 자르기 내 입력 상자가 (52, 18)에서 시작하는 경우:
- entry_x0 = 52 + 50 = 102
- entry_top = 18 + 120 = 138

인근 필드를 가능한 경우 단일 자르기로 그룹화하여 **각 필드에 대해 반복**하세요.

### B.4: 세밀화된 좌표로 fields.json 생성

`image_width`와 `image_height`를 사용하여 fields.json 생성 (이미지 좌표 신호):
```json
{
  "pages": [
    {"page_number": 1, "image_width": 1700, "image_height": 2200}
  ],
  "form_fields": [
    {
      "page_number": 1,
      "description": "성 입력 필드",
      "field_label": "Last Name",
      "label_bounding_box": [120, 175, 242, 198],
      "entry_bounding_box": [255, 175, 720, 218],
      "entry_text": {"text": "Smith", "font_size": 10}
    }
  ]
}
```

**중요**: `image_width`/`image_height`와 확대 분석의 정확한 픽셀 좌표를 사용하세요.

### B.5: 경계 상자 검증

작성 전에 경계 상자 에러를 확인하세요:
`python scripts/check_bounding_boxes.py fields.json`

교차하는 경계 상자와 폰트 크기에 비해 너무 작은 입력 상자를 확인합니다. 작성 전에 보고된 에러를 수정하세요.

---

## 혼합 접근 방식: 구조 + 시각적

구조 추출이 대부분의 필드에서 작동하지만 일부 요소(예: 원형 체크박스, 비표준 양식 컨트롤)를 놓친 경우 사용합니다.

1. form_structure.json에서 감지된 필드에는 **접근 방식 A** 사용
2. 누락된 필드의 시각적 분석을 위해 **PDF를 이미지로 변환**
3. 누락된 필드에는 (접근 방식 B의) **확대 세밀화** 사용
4. **좌표 결합**: 구조 추출 필드에는 `pdf_width`/`pdf_height` 사용. 시각적으로 추정한 필드는 이미지 좌표를 PDF 좌표로 변환:
   - pdf_x = image_x * (pdf_width / image_width)
   - pdf_y = image_y * (pdf_height / image_height)
5. fields.json에서 **단일 좌표 시스템 사용** - `pdf_width`/`pdf_height`로 PDF 좌표로 변환

---

## 2단계: 작성 전 검증

**항상 작성 전에 경계 상자를 검증하세요:**
`python scripts/check_bounding_boxes.py fields.json`

다음을 확인합니다:
- 교차하는 경계 상자 (텍스트 겹침 발생)
- 지정된 폰트 크기에 비해 너무 작은 입력 상자

진행 전에 fields.json의 에러를 수정하세요.

## 3단계: 양식 작성

작성 스크립트가 좌표 시스템을 자동으로 감지하고 변환을 처리합니다:
`python scripts/fill_pdf_form_with_annotations.py <input.pdf> fields.json <output.pdf>`

## 4단계: 출력 확인

작성된 PDF를 이미지로 변환하여 텍스트 배치를 확인하세요:
`python scripts/convert_pdf_to_images.py <output.pdf> <verify_images/>`

텍스트가 잘못 배치된 경우:
- **접근 방식 A**: `pdf_width`/`pdf_height`로 form_structure.json의 PDF 좌표를 사용하고 있는지 확인
- **접근 방식 B**: 이미지 치수가 일치하고 좌표가 정확한 픽셀인지 확인
- **혼합**: 시각적으로 추정한 필드의 좌표 변환이 정확한지 확인
