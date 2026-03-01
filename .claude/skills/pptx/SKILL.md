---
name: pptx
description: "Presentation creation, editing, and analysis. When Claude needs to work with presentations (.pptx files) for: (1) Creating new presentations, (2) Modifying or editing content, (3) Working with layouts, (4) Adding comments or speaker notes, or any other presentation tasks"
license: Proprietary. LICENSE.txt has complete terms
---

# PPTX 생성, 편집, 분석

## 개요

사용자가 .pptx 파일을 생성, 편집, 또는 내용 분석을 요청할 수 있습니다. .pptx 파일은 기본적으로 XML 파일과 기타 리소스를 포함하는 ZIP 아카이브로, 읽거나 편집할 수 있습니다. 작업에 따라 다양한 도구와 워크플로우가 있습니다.

## 내용 읽기 및 분석

### 텍스트 추출
프레젠테이션의 텍스트 내용만 읽으려면 문서를 마크다운으로 변환합니다:

```bash
# 문서를 마크다운으로 변환
python -m markitdown path-to-file.pptx
```

### 원시 XML 접근
주석, 발표자 노트, 슬라이드 레이아웃, 애니메이션, 디자인 요소, 복잡한 포맷팅 등에는 원시 XML 접근이 필요합니다. 이러한 기능을 사용하려면 프레젠테이션의 압축을 풀고 원시 XML 내용을 읽어야 합니다.

#### 파일 압축 풀기
`python ooxml/scripts/unpack.py <office_file> <output_dir>`

**참고**: unpack.py 스크립트는 프로젝트 루트 기준 `skills/pptx/ooxml/scripts/unpack.py`에 있습니다. 이 경로에 스크립트가 없으면 `find . -name "unpack.py"`로 위치를 찾으세요.

#### 주요 파일 구조
* `ppt/presentation.xml` - 메인 프레젠테이션 메타데이터 및 슬라이드 참조
* `ppt/slides/slide{N}.xml` - 개별 슬라이드 내용 (slide1.xml, slide2.xml 등)
* `ppt/notesSlides/notesSlide{N}.xml` - 각 슬라이드의 발표자 노트
* `ppt/comments/modernComment_*.xml` - 특정 슬라이드의 주석
* `ppt/slideLayouts/` - 슬라이드 레이아웃 템플릿
* `ppt/slideMasters/` - 마스터 슬라이드 템플릿
* `ppt/theme/` - 테마 및 스타일 정보
* `ppt/media/` - 이미지 및 기타 미디어 파일

#### 타이포그래피 및 색상 추출
**모방할 예시 디자인이 주어진 경우**: 아래 방법을 사용하여 프레젠테이션의 타이포그래피와 색상을 먼저 분석합니다:
1. **테마 파일 읽기**: 색상(`<a:clrScheme>`) 및 폰트(`<a:fontScheme>`)에 대해 `ppt/theme/theme1.xml` 확인
2. **슬라이드 내용 샘플링**: 실제 폰트 사용(`<a:rPr>`) 및 색상을 위해 `ppt/slides/slide1.xml` 검사
3. **패턴 검색**: 모든 XML 파일에서 색상(`<a:solidFill>`, `<a:srgbClr>`) 및 폰트 참조 grep 검색

## 템플릿 없이 새 PowerPoint 프레젠테이션 생성

처음부터 새 PowerPoint 프레젠테이션을 만들 때는 **html2pptx** 워크플로우를 사용하여 HTML 슬라이드를 정확한 위치 지정으로 PowerPoint로 변환합니다.

### 디자인 원칙

**중요**: 프레젠테이션을 만들기 전에 내용을 분석하고 적절한 디자인 요소를 선택합니다:
1. **주제 고려**: 이 프레젠테이션은 무엇에 관한 것인가? 어떤 톤, 산업, 분위기를 암시하는가?
2. **브랜딩 확인**: 사용자가 회사/조직을 언급하면 브랜드 색상 및 아이덴티티 고려
3. **내용에 맞는 팔레트 선택**: 주제를 반영하는 색상 선택
4. **접근 방식 설명**: 코드 작성 전에 디자인 선택에 대해 설명

**요구사항**:
- 코드 작성 전에 내용에 맞는 디자인 접근 방식 설명 필수
- 웹 안전 폰트만 사용: Arial, Helvetica, Times New Roman, Georgia, Courier New, Verdana, Tahoma, Trebuchet MS, Impact
- 크기, 굵기, 색상을 통한 명확한 시각적 계층 구조 생성
- 가독성 확보: 강한 대비, 적절한 크기의 텍스트, 깔끔한 정렬
- 일관성 유지: 슬라이드 전체에 걸쳐 패턴, 간격, 시각적 언어 반복

#### 색상 팔레트 선택

**창의적으로 색상 선택**:
- **기본값에서 벗어나기**: 이 특정 주제에 진정으로 어울리는 색상은 무엇인가? 자동 선택 지양
- **다양한 각도 고려**: 주제, 산업, 분위기, 에너지 수준, 대상 청중, 브랜드 아이덴티티(언급된 경우)
- **모험적으로 접근**: 예상치 못한 조합 시도 - 의료 프레젠테이션이 꼭 녹색일 필요 없고, 금융이 꼭 네이비일 필요 없음
- **팔레트 구성**: 함께 어울리는 3-5가지 색상 선택 (주요 색상 + 보조 톤 + 강조색)
- **대비 확보**: 텍스트가 배경에서 명확하게 읽혀야 함

**예시 색상 팔레트** (창의성 자극을 위해 사용 - 하나 선택, 조정, 또는 자체 생성):

1. **클래식 블루**: 딥 네이비 (#1C2833), 슬레이트 그레이 (#2E4053), 실버 (#AAB7B8), 오프 화이트 (#F4F6F6)
2. **틸 & 코랄**: 틸 (#5EA8A7), 딥 틸 (#277884), 코랄 (#FE4447), 흰색 (#FFFFFF)
3. **볼드 레드**: 레드 (#C0392B), 브라이트 레드 (#E74C3C), 오렌지 (#F39C12), 옐로우 (#F1C40F), 그린 (#2ECC71)
4. **웜 블러시**: 모브 (#A49393), 블러시 (#EED6D3), 로즈 (#E8B4B8), 크림 (#FAF7F2)
5. **버건디 럭셔리**: 버건디 (#5D1D2E), 크림슨 (#951233), 러스트 (#C15937), 골드 (#997929)
6. **딥 퍼플 & 에메랄드**: 퍼플 (#B165FB), 다크 블루 (#181B24), 에메랄드 (#40695B), 흰색 (#FFFFFF)
7. **크림 & 포레스트 그린**: 크림 (#FFE1C7), 포레스트 그린 (#40695B), 흰색 (#FCFCFC)
8. **핑크 & 퍼플**: 핑크 (#F8275B), 코랄 (#FF574A), 로즈 (#FF737D), 퍼플 (#3D2F68)
9. **라임 & 플럼**: 라임 (#C5DE82), 플럼 (#7C3A5F), 코랄 (#FD8C6E), 블루 그레이 (#98ACB5)
10. **블랙 & 골드**: 골드 (#BF9A4A), 블랙 (#000000), 크림 (#F4F6F6)
11. **세이지 & 테라코타**: 세이지 (#87A96B), 테라코타 (#E07A5F), 크림 (#F4F1DE), 차콜 (#2C2C2C)
12. **차콜 & 레드**: 차콜 (#292929), 레드 (#E33737), 라이트 그레이 (#CCCBCB)
13. **바이브런트 오렌지**: 오렌지 (#F96D00), 라이트 그레이 (#F2F2F2), 차콜 (#222831)
14. **포레스트 그린**: 블랙 (#191A19), 그린 (#4E9F3D), 다크 그린 (#1E5128), 흰색 (#FFFFFF)
15. **레트로 레인보우**: 퍼플 (#722880), 핑크 (#D72D51), 오렌지 (#EB5C18), 앰버 (#F08800), 골드 (#DEB600)
16. **빈티지 어시**: 머스타드 (#E3B448), 세이지 (#CBD18F), 포레스트 그린 (#3A6B35), 크림 (#F4F1DE)
17. **코스탈 로즈**: 올드 로즈 (#AD7670), 비버 (#B49886), 에그셸 (#F3ECDC), 애시 그레이 (#BFD5BE)
18. **오렌지 & 터콰이즈**: 라이트 오렌지 (#FC993E), 그레이시 터콰이즈 (#667C6F), 흰색 (#FCFCFC)
19. **미니멀 B&W + 레드** (검증됨): 블랙 (#000000), 흰색 (#FFFFFF), 레드 (#FF3B30), 그레이 (#333, #666, #999, #F5F5F5). 매우 깔끔하고 고대비. 완전한 디자인 시스템은 [`designs/minimal-bw-red.md`](designs/minimal-bw-red.md) 참조

#### 시각적 디테일 옵션

**기하학적 패턴**:
- 수평 대신 대각선 섹션 구분자
- 비대칭 열 너비 (30/70, 40/60, 25/75)
- 90° 또는 270° 회전된 텍스트 헤더
- 이미지를 위한 원형/육각형 프레임
- 모서리의 삼각형 강조 형태
- 깊이감을 위한 겹치는 도형

**테두리 & 프레임 처리**:
- 한쪽에만 두꺼운 단색 테두리 (10-20pt)
- 대비되는 색상의 이중 선 테두리
- 전체 프레임 대신 모서리 브래킷
- L자형 테두리 (위+왼쪽 또는 아래+오른쪽)
- 헤더 아래 밑줄 강조 (3-5pt 두께)

**타이포그래피 처리**:
- 극단적인 크기 대비 (72pt 헤드라인 vs 11pt 본문)
- 넓은 자간의 전체 대문자 헤더
- 초대형 디스플레이 타입의 번호 섹션
- 데이터/통계/기술 콘텐츠에 모노스페이스 (Courier New)
- 밀집된 정보에 좁은 폰트 (Arial Narrow)
- 강조를 위한 아웃라인 텍스트

**차트 & 데이터 스타일링**:
- 핵심 데이터에 단일 강조색이 있는 단색 차트
- 수직 대신 수평 막대 차트
- 막대 차트 대신 점 플롯
- 최소한의 격자선 또는 없음
- 요소에 직접 표시되는 데이터 레이블 (범례 없음)
- 핵심 지표를 위한 초대형 숫자

**레이아웃 혁신**:
- 텍스트 오버레이가 있는 전체 배경 이미지
- 탐색/컨텍스트를 위한 사이드바 열 (너비 20-30%)
- 모듈식 그리드 시스템 (3×3, 4×4 블록)
- Z 패턴 또는 F 패턴 콘텐츠 흐름
- 색상 도형 위에 떠 있는 텍스트 박스
- 잡지 스타일의 다중 열 레이아웃

**배경 처리**:
- 슬라이드의 40-60%를 차지하는 단색 블록
- 그라디언트 채우기 (수직 또는 대각선만)
- 분할 배경 (두 가지 색상, 대각선 또는 수직)
- 가장자리 간 색상 밴드
- 디자인 요소로서의 여백

### 레이아웃 팁
**중요 - Flexbox만 사용:**
- **`position: absolute` 절대 사용 금지** - PPTX 출력에서 겹치는 도형 생성
- **항상 flexbox 사용** (`display: flex`) 모든 위치 지정에
- 요소 간 간격에는 `margin`, 비례 크기 조정에는 `flex: 1` 사용

**차트 또는 테이블이 있는 슬라이드 생성 시:**
- **두 열 레이아웃 (권장)**: 전체 너비에 걸치는 헤더를 사용하고 아래에 두 열 배치 - 한 열에 텍스트/불릿, 다른 열에 주요 콘텐츠. 더 나은 균형을 제공하고 차트/테이블의 가독성 향상. 각 콘텐츠 유형에 맞게 불균형 열 너비 (예: 40%/60%)의 flexbox 사용
- **전체 슬라이드 레이아웃**: 최대 임팩트와 가독성을 위해 주요 콘텐츠가 전체 슬라이드 차지
- **세로 쌓기 절대 금지**: 단일 열에서 텍스트 아래 차트/테이블 배치 - 가독성 저하 및 레이아웃 문제 발생

### 워크플로우
1. **필수 - 전체 파일 읽기**: [`html2pptx.md`](html2pptx.md)를 처음부터 끝까지 완전히 읽습니다. **이 파일을 읽을 때 절대로 범위 제한을 설정하지 마세요.** 프레젠테이션 생성을 진행하기 전에 상세한 구문, 중요한 포맷팅 규칙, 모범 사례를 위해 전체 파일 내용을 읽으세요.
2. 각 슬라이드에 적절한 치수 (예: 16:9의 경우 720pt × 405pt)의 HTML 파일 생성
   - 모든 텍스트 내용에 `<p>`, `<h1>`-`<h6>`, `<ul>`, `<ol>` 사용
   - 차트/테이블이 추가될 영역에 `class="placeholder"` 사용 (가시성을 위해 회색 배경으로 렌더링)
   - **중요**: Sharp를 사용하여 그라디언트와 아이콘을 PNG 이미지로 먼저 래스터화한 후 HTML에서 참조
   - **레이아웃**: 차트/테이블/이미지가 있는 슬라이드에는 전체 슬라이드 레이아웃 또는 두 열 레이아웃 사용
3. [`html2pptx.js`](scripts/html2pptx.js) 라이브러리를 사용하는 JavaScript 파일을 생성하고 실행하여 HTML 슬라이드를 PowerPoint로 변환하고 프레젠테이션 저장
   - 각 HTML 파일을 처리하기 위해 `html2pptx()` 함수 사용
   - PptxGenJS API를 사용하여 플레이스홀더 영역에 차트 및 테이블 추가
   - `pptx.writeFile()`을 사용하여 프레젠테이션 저장
4. **시각적 검증**: 썸네일 생성 및 레이아웃 문제 검사
   - 썸네일 그리드 생성: `python scripts/thumbnail.py output.pptx workspace/thumbnails --cols 4`
   - 썸네일 이미지를 주의 깊게 읽고 검사:
     - **텍스트 잘림**: 헤더 바, 도형, 슬라이드 가장자리에 의해 잘리는 텍스트
     - **텍스트 겹침**: 텍스트와 다른 텍스트 또는 도형의 겹침
     - **위치 문제**: 슬라이드 경계나 다른 요소에 너무 가까운 콘텐츠
     - **대비 문제**: 텍스트와 배경 사이의 불충분한 대비
   - 문제가 발견되면 HTML 마진/간격/색상 조정 후 프레젠테이션 재생성
   - 모든 슬라이드가 시각적으로 올바를 때까지 반복

## 기존 PowerPoint 프레젠테이션 편집

기존 PowerPoint 프레젠테이션의 슬라이드를 편집할 때는 원시 Office Open XML (OOXML) 형식으로 작업해야 합니다. 여기에는 .pptx 파일 압축 해제, XML 내용 편집, 다시 압축이 포함됩니다.

### 워크플로우
1. **필수 - 전체 파일 읽기**: [`ooxml.md`](ooxml.md) (~500줄)를 처음부터 끝까지 완전히 읽습니다. **이 파일을 읽을 때 절대로 범위 제한을 설정하지 마세요.** 프레젠테이션 편집을 시작하기 전에 OOXML 구조 및 편집 워크플로우에 대한 상세한 안내를 위해 전체 파일 내용을 읽으세요.
2. 프레젠테이션 압축 해제: `python ooxml/scripts/unpack.py <office_file> <output_dir>`
3. XML 파일 편집 (주로 `ppt/slides/slide{N}.xml` 및 관련 파일)
4. **중요**: 각 편집 후 즉시 검증하고 계속 진행 전에 검증 오류 수정: `python ooxml/scripts/validate.py <dir> --original <file>`
5. 최종 프레젠테이션 압축: `python ooxml/scripts/pack.py <input_directory> <office_file>`

## 템플릿을 사용한 새 PowerPoint 프레젠테이션 생성

기존 템플릿의 디자인을 따르는 프레젠테이션을 생성해야 할 때는 템플릿 슬라이드를 복제하고 재배열한 후 플레이스홀더 콘텐츠를 교체합니다.

### 워크플로우
1. **템플릿 텍스트 추출 및 시각적 썸네일 그리드 생성**:
   * 텍스트 추출: `python -m markitdown template.pptx > template-content.md`
   * `template-content.md` 읽기: 템플릿 프레젠테이션 내용을 이해하기 위해 전체 파일 읽기. **이 파일을 읽을 때 절대로 범위 제한을 설정하지 마세요.**
   * 썸네일 그리드 생성: `python scripts/thumbnail.py template.pptx`
   * 자세한 내용은 [썸네일 그리드 생성](#썸네일-그리드-생성) 섹션 참조

2. **템플릿 분석 및 인벤토리를 파일로 저장**:
   * **시각적 분석**: 썸네일 그리드 검토로 슬라이드 레이아웃, 디자인 패턴, 시각적 구조 이해
   * `template-inventory.md`에 다음 내용을 포함하는 템플릿 인벤토리 파일 생성 및 저장:
     ```markdown
     # Template Inventory Analysis
     **Total Slides: [count]**
     **IMPORTANT: Slides are 0-indexed (first slide = 0, last slide = count-1)**

     ## [Category Name]
     - Slide 0: [Layout code if available] - Description/purpose
     - Slide 1: [Layout code] - Description/purpose
     - Slide 2: [Layout code] - Description/purpose
     [... EVERY slide must be listed individually with its index ...]
     ```
   * **썸네일 그리드 활용**: 시각적 썸네일을 참조하여 식별:
     - 레이아웃 패턴 (타이틀 슬라이드, 콘텐츠 레이아웃, 섹션 구분자)
     - 이미지 플레이스홀더 위치 및 수량
     - 슬라이드 그룹 간 디자인 일관성
     - 시각적 계층 및 구조
   * 이 인벤토리 파일은 다음 단계에서 적절한 템플릿 선택에 필수

3. **템플릿 인벤토리를 기반으로 프레젠테이션 개요 작성**:
   * 2단계의 사용 가능한 템플릿 검토
   * 첫 번째 슬라이드에 인트로 또는 타이틀 템플릿 선택 (초기 템플릿 중 하나여야 함)
   * 다른 슬라이드에는 안전하고 텍스트 기반 레이아웃 선택
   * **중요: 레이아웃 구조를 실제 콘텐츠에 맞추기**:
     - 단일 열 레이아웃: 통합된 내러티브 또는 단일 주제에 사용
     - 두 열 레이아웃: 정확히 2개의 별개 항목/개념이 있을 때만 사용
     - 세 열 레이아웃: 정확히 3개의 별개 항목/개념이 있을 때만 사용
     - 이미지 + 텍스트 레이아웃: 실제 삽입할 이미지가 있을 때만 사용
     - 인용구 레이아웃: 사람의 실제 인용구(출처 포함)에만 사용, 강조를 위해 절대 사용 금지
     - 보유한 콘텐츠보다 플레이스홀더가 많은 레이아웃 사용 금지
     - 항목이 2개라면 3열 레이아웃에 억지로 맞추지 말 것
     - 항목이 4개 이상이라면 여러 슬라이드로 나누거나 목록 형식 사용 고려
   * 레이아웃 선택 전에 실제 콘텐츠 조각 수 계산
   * 선택한 레이아웃의 각 플레이스홀더가 의미 있는 콘텐츠로 채워질지 확인
   * 각 콘텐츠 섹션에 최적의 레이아웃 하나 선택
   * 사용 가능한 디자인을 활용하는 콘텐츠 및 템플릿 매핑이 있는 `outline.md` 저장
   * 템플릿 매핑 예시:
      ```
      # Template slides to use (0-based indexing)
      # WARNING: Verify indices are within range! Template with 73 slides has indices 0-72
      # Mapping: slide numbers from outline -> template slide indices
      template_mapping = [
          0,   # Use slide 0 (Title/Cover)
          34,  # Use slide 34 (B1: Title and body)
          34,  # Use slide 34 again (duplicate for second B1)
          50,  # Use slide 50 (E1: Quote)
          54,  # Use slide 54 (F2: Closing + Text)
      ]
      ```

4. **`rearrange.py`를 사용하여 슬라이드 복제, 재정렬, 삭제**:
   * `scripts/rearrange.py` 스크립트를 사용하여 원하는 순서로 슬라이드가 있는 새 프레젠테이션 생성:
     ```bash
     python scripts/rearrange.py template.pptx working.pptx 0,34,34,50,52
     ```
   * 스크립트가 반복된 슬라이드 복제, 사용하지 않는 슬라이드 삭제, 재정렬을 자동으로 처리
   * 슬라이드 인덱스는 0부터 시작 (첫 번째 슬라이드는 0, 두 번째는 1 등)
   * 같은 슬라이드 인덱스가 여러 번 나타나 해당 슬라이드 복제 가능

5. **`inventory.py` 스크립트를 사용하여 모든 텍스트 추출**:
   * **인벤토리 추출 실행**:
     ```bash
     python scripts/inventory.py working.pptx text-inventory.json
     ```
   * **text-inventory.json 읽기**: 모든 도형과 속성을 이해하기 위해 전체 text-inventory.json 파일 읽기. **이 파일을 읽을 때 절대로 범위 제한을 설정하지 마세요.**

   * 인벤토리 JSON 구조:
      ```json
        {
          "slide-0": {
            "shape-0": {
              "placeholder_type": "TITLE",  // 비-플레이스홀더의 경우 null
              "left": 1.5,                  // 인치 단위 위치
              "top": 2.0,
              "width": 7.5,
              "height": 1.2,
              "paragraphs": [
                {
                  "text": "Paragraph text",
                  // 선택적 속성 (기본값이 아닌 경우만 포함):
                  "bullet": true,           // 명시적 불릿 감지됨
                  "level": 0,               // bullet이 true일 때만 포함
                  "alignment": "CENTER",    // CENTER, RIGHT (LEFT 제외)
                  "space_before": 10.0,     // 단락 앞 간격 (포인트 단위)
                  "space_after": 6.0,       // 단락 뒤 간격 (포인트 단위)
                  "line_spacing": 22.4,     // 줄 간격 (포인트 단위)
                  "font_name": "Arial",     // 첫 번째 런에서
                  "font_size": 14.0,        // 포인트 단위
                  "bold": true,
                  "italic": false,
                  "underline": false,
                  "color": "FF0000"         // RGB 색상
                }
              ]
            }
          }
        }
      ```

   * 주요 기능:
     - **슬라이드**: "slide-0", "slide-1" 등으로 명명
     - **도형**: 시각적 위치 (위에서 아래로, 왼쪽에서 오른쪽으로) 순서로 "shape-0", "shape-1" 등
     - **플레이스홀더 유형**: TITLE, CENTER_TITLE, SUBTITLE, BODY, OBJECT, 또는 null
     - **기본 폰트 크기**: 레이아웃 플레이스홀더에서 추출한 포인트 단위의 `default_font_size` (사용 가능한 경우)
     - **슬라이드 번호 필터링**: SLIDE_NUMBER 플레이스홀더 유형의 도형은 인벤토리에서 자동 제외
     - **불릿**: `bullet: true`인 경우 `level`이 항상 포함됨 (0인 경우도)
     - **간격**: `space_before`, `space_after`, `line_spacing` (포인트 단위, 설정된 경우만 포함)
     - **색상**: RGB의 경우 `color` (예: "FF0000"), 테마 색상의 경우 `theme_color` (예: "DARK_1")
     - **속성**: 기본값이 아닌 값만 출력에 포함

6. **교체 텍스트 생성 및 JSON 파일로 데이터 저장**
   이전 단계의 텍스트 인벤토리를 기반으로:
   - **중요**: 인벤토리에 존재하는 도형만 참조 - 실제로 있는 도형만 참조
   - **검증**: replace.py 스크립트가 교체 JSON의 모든 도형이 인벤토리에 존재하는지 검증
     - 존재하지 않는 도형을 참조하면 사용 가능한 도형을 보여주는 오류 발생
     - 존재하지 않는 슬라이드를 참조하면 슬라이드가 없다는 오류 발생
     - 모든 검증 오류는 스크립트 종료 전에 한 번에 표시
   - **중요**: replace.py 스크립트는 내부적으로 inventory.py를 사용하여 모든 텍스트 도형 식별
   - **자동 지우기**: "paragraphs"를 제공하지 않으면 인벤토리의 모든 텍스트 도형이 지워짐
   - 콘텐츠가 필요한 도형에 "paragraphs" 필드 추가 ("replacement_paragraphs" 아님)
   - 교체 JSON에 "paragraphs"가 없는 도형은 자동으로 텍스트가 지워짐
   - 불릿이 있는 단락은 자동으로 왼쪽 정렬됨. `"bullet": true`인 경우 `alignment` 속성 설정 금지
   - 플레이스홀더 텍스트에 적절한 교체 콘텐츠 생성
   - 도형 크기를 사용하여 적절한 콘텐츠 길이 결정
   - **중요**: 원본 인벤토리의 단락 속성 포함 - 텍스트만 제공하지 말 것
   - **중요**: `bullet: true`인 경우 텍스트에 불릿 기호 (•, -, *) 포함 금지 - 자동으로 추가됨
   - **필수 포맷팅 규칙**:
     - 헤더/타이틀에는 일반적으로 `"bold": true`
     - 목록 항목에는 `"bullet": true, "level": 0` (bullet이 true이면 level 필수)
     - 정렬 속성 보존 (예: 중앙 정렬 텍스트에 `"alignment": "CENTER"`)
     - 기본값과 다른 경우 폰트 속성 포함 (예: `"font_size": 14.0`, `"font_name": "Lora"`)
     - 색상: RGB에는 `"color": "FF0000"`, 테마 색상에는 `"theme_color": "DARK_1"`
     - 교체 스크립트는 텍스트 문자열이 아닌 **적절히 포맷된 단락** 기대
     - **겹치는 도형**: 더 큰 default_font_size 또는 더 적절한 placeholder_type의 도형 선호
   - 교체 사항이 있는 업데이트된 인벤토리를 `replacement-text.json`으로 저장
   - **경고**: 다양한 템플릿 레이아웃은 도형 수가 다름 - 교체 전에 항상 실제 인벤토리 확인

   적절한 포맷팅을 보여주는 paragraphs 필드 예시:
   ```json
   "paragraphs": [
     {
       "text": "New presentation title text",
       "alignment": "CENTER",
       "bold": true
     },
     {
       "text": "Section Header",
       "bold": true
     },
     {
       "text": "First bullet point without bullet symbol",
       "bullet": true,
       "level": 0
     },
     {
       "text": "Red colored text",
       "color": "FF0000"
     },
     {
       "text": "Theme colored text",
       "theme_color": "DARK_1"
     },
     {
       "text": "Regular paragraph text without special formatting"
     }
   ]
   ```

   **교체 JSON에 없는 도형은 자동으로 지워짐**:
   ```json
   {
     "slide-0": {
       "shape-0": {
         "paragraphs": [...] // 이 도형은 새 텍스트 받음
       }
       // 인벤토리의 shape-1과 shape-2는 자동으로 지워짐
     }
   }
   ```

   **프레젠테이션의 일반적인 포맷팅 패턴**:
   - 타이틀 슬라이드: 굵은 텍스트, 때로는 중앙 정렬
   - 슬라이드 내 섹션 헤더: 굵은 텍스트
   - 불릿 목록: 각 항목에 `"bullet": true, "level": 0` 필요
   - 본문 텍스트: 일반적으로 특별한 속성 불필요
   - 인용구: 특별한 정렬 또는 폰트 속성이 있을 수 있음

7. **`replace.py` 스크립트를 사용하여 교체 적용**
   ```bash
   python scripts/replace.py working.pptx replacement-text.json output.pptx
   ```

   스크립트는:
   - inventory.py의 함수를 사용하여 모든 텍스트 도형의 인벤토리 먼저 추출
   - 교체 JSON의 모든 도형이 인벤토리에 존재하는지 검증
   - 인벤토리에서 식별된 모든 도형의 텍스트 지우기
   - 교체 JSON에 "paragraphs"가 정의된 도형에만 새 텍스트 적용
   - JSON의 단락 속성을 적용하여 포맷팅 보존
   - 불릿, 정렬, 폰트 속성, 색상 자동 처리
   - 업데이트된 프레젠테이션 저장

   검증 오류 예시:
   ```
   ERROR: Invalid shapes in replacement JSON:
     - Shape 'shape-99' not found on 'slide-0'. Available shapes: shape-0, shape-1, shape-4
     - Slide 'slide-999' not found in inventory
   ```

   ```
   ERROR: Replacement text made overflow worse in these shapes:
     - slide-0/shape-2: overflow worsened by 1.25" (was 0.00", now 1.25")
   ```

## 썸네일 그리드 생성

빠른 분석 및 참조를 위해 PowerPoint 슬라이드의 시각적 썸네일 그리드 생성:

```bash
python scripts/thumbnail.py template.pptx [output_prefix]
```

**기능**:
- 생성: `thumbnails.jpg` (또는 대형 덱의 경우 `thumbnails-1.jpg`, `thumbnails-2.jpg` 등)
- 기본값: 5열, 그리드당 최대 30 슬라이드 (5×6)
- 사용자 정의 접두사: `python scripts/thumbnail.py template.pptx my-grid`
  - 참고: 특정 디렉토리에 출력하려면 접두사에 경로 포함 (예: `workspace/my-grid`)
- 열 조정: `--cols 4` (범위: 3-6, 그리드당 슬라이드 수에 영향)
- 그리드 한계: 3열 = 12 슬라이드/그리드, 4열 = 20, 5열 = 30, 6열 = 42
- 슬라이드는 0부터 인덱싱 (슬라이드 0, 슬라이드 1 등)

**사용 사례**:
- 템플릿 분석: 슬라이드 레이아웃 및 디자인 패턴 빠르게 이해
- 콘텐츠 검토: 전체 프레젠테이션 시각적 개요
- 탐색 참조: 시각적 모양으로 특정 슬라이드 찾기
- 품질 확인: 모든 슬라이드가 올바르게 포맷되었는지 확인

**예시**:
```bash
# 기본 사용법
python scripts/thumbnail.py presentation.pptx

# 옵션 조합: 사용자 정의 이름, 열
python scripts/thumbnail.py template.pptx analysis --cols 4
```

## 슬라이드를 이미지로 변환

PowerPoint 슬라이드를 시각적으로 분석하려면 두 단계로 이미지로 변환합니다:

1. **PPTX를 PDF로 변환**:
   ```bash
   soffice --headless --convert-to pdf template.pptx
   ```

2. **PDF 페이지를 JPEG 이미지로 변환**:
   ```bash
   pdftoppm -jpeg -r 150 template.pdf slide
   ```
   `slide-1.jpg`, `slide-2.jpg` 등의 파일이 생성됩니다.

옵션:
- `-r 150`: 해상도를 150 DPI로 설정 (품질/크기 균형에 맞게 조정)
- `-jpeg`: JPEG 형식 출력 (PNG 원하면 `-png` 사용)
- `-f N`: 변환 시작 페이지 (예: `-f 2`는 2페이지부터 시작)
- `-l N`: 변환 종료 페이지 (예: `-l 5`는 5페이지에서 중지)
- `slide`: 출력 파일 접두사

특정 범위 예시:
```bash
pdftoppm -jpeg -r 150 -f 2 -l 5 template.pdf slide  # 2-5 페이지만 변환
```

## 코드 스타일 가이드라인
**중요**: PPTX 작업용 코드 생성 시:
- 간결한 코드 작성
- 장황한 변수명과 불필요한 작업 지양
- 불필요한 print 문 지양

## 의존성

필수 의존성 (이미 설치되어 있어야 함):

- **markitdown**: `pip install "markitdown[pptx]"` (프레젠테이션에서 텍스트 추출)
- **pptxgenjs**: `npm install -g pptxgenjs` (html2pptx로 프레젠테이션 생성)
- **playwright**: `npm install -g playwright` (html2pptx에서 HTML 렌더링)
- **react-icons**: `npm install -g react-icons react react-dom` (아이콘)
- **sharp**: `npm install -g sharp` (SVG 래스터화 및 이미지 처리)
- **LibreOffice**: `sudo apt-get install libreoffice` (PDF 변환)
- **Poppler**: `sudo apt-get install poppler-utils` (PDF를 이미지로 변환하는 pdftoppm)
- **defusedxml**: `pip install defusedxml` (보안 XML 파싱)
