---
name: docx
description: "사용자가 Word 문서(.docx 파일)를 생성, 읽기, 편집, 또는 조작하고 싶을 때 이 스킬을 사용하세요. 트리거: \"Word doc\", \"word document\", \".docx\" 언급, 또는 목차, 헤딩, 페이지 번호, 레터헤드가 있는 전문 문서 요청. 또한 .docx 파일에서 콘텐츠 추출 또는 재구성, 문서에 이미지 삽입 또는 교체, Word 파일에서 찾기-바꾸기, 추적 변경 또는 댓글 작업, 콘텐츠를 세련된 Word 문서로 변환 시 사용. 사용자가 Word 또는 .docx 파일로 \"보고서\", \"메모\", \"편지\", \"템플릿\" 등을 요청하면 이 스킬을 사용. PDF, 스프레드시트, Google Docs, 또는 문서 생성과 무관한 일반 코딩 작업에는 사용하지 마세요."
license: 독점 소프트웨어. 전체 약관은 LICENSE.txt
---

# DOCX 생성, 편집, 분석

## 개요

.docx 파일은 XML 파일을 포함하는 ZIP 아카이브입니다.

## 빠른 참조

| 작업 | 방법 |
|------|----------|
| 콘텐츠 읽기/분석 | `pandoc` 또는 원시 XML을 위해 압축 해제 |
| 새 문서 생성 | `docx-js` 사용 - 아래 새 문서 생성 참고 |
| 기존 문서 편집 | 압축 해제 → XML 편집 → 재압축 - 아래 기존 문서 편집 참고 |

### .doc를 .docx로 변환

편집 전에 레거시 `.doc` 파일을 변환해야 합니다:

```bash
python scripts/office/soffice.py --headless --convert-to docx document.doc
```

### 콘텐츠 읽기

```bash
# 추적 변경이 있는 텍스트 추출
pandoc --track-changes=all document.docx -o output.md

# 원시 XML 접근
python scripts/office/unpack.py document.docx unpacked/
```

### 이미지로 변환

```bash
python scripts/office/soffice.py --headless --convert-to pdf document.docx
pdftoppm -jpeg -r 150 document.pdf page
```

### 추적 변경 수락

모든 추적 변경이 수락된 깔끔한 문서를 생성하려면 (LibreOffice 필요):

```bash
python scripts/accept_changes.py input.docx output.docx
```

---

## 새 문서 생성

JavaScript로 .docx 파일을 생성한 다음 검증합니다. 설치: `npm install -g docx`

### 설정
```javascript
const { Document, Packer, Paragraph, TextRun, Table, TableRow, TableCell, ImageRun,
        Header, Footer, AlignmentType, PageOrientation, LevelFormat, ExternalHyperlink,
        TableOfContents, HeadingLevel, BorderStyle, WidthType, ShadingType,
        VerticalAlign, PageNumber, PageBreak } = require('docx');

const doc = new Document({ sections: [{ children: [/* 콘텐츠 */] }] });
Packer.toBuffer(doc).then(buffer => fs.writeFileSync("doc.docx", buffer));
```

### 검증
파일 생성 후 검증합니다. 검증 실패 시 압축 해제하고, XML을 수정하고, 재압축합니다.
```bash
python scripts/office/validate.py doc.docx
```

### 페이지 크기

```javascript
// 중요: docx-js는 기본적으로 US Letter가 아닌 A4
// 일관된 결과를 위해 항상 페이지 크기를 명시적으로 설정
sections: [{
  properties: {
    page: {
      size: {
        width: 12240,   // 8.5인치 (DXA 단위)
        height: 15840   // 11인치 (DXA 단위)
      },
      margin: { top: 1440, right: 1440, bottom: 1440, left: 1440 } // 1인치 여백
    }
  },
  children: [/* 콘텐츠 */]
}]
```

**일반적인 페이지 크기 (DXA 단위, 1440 DXA = 1인치):**

| 용지 | 너비 | 높이 | 콘텐츠 너비 (1인치 여백) |
|-------|-------|--------|---------------------------|
| US Letter | 12,240 | 15,840 | 9,360 |
| A4 (기본) | 11,906 | 16,838 | 9,026 |

**가로 방향:** docx-js는 내부적으로 너비/높이를 교환하므로, 세로 치수를 전달하고 교환은 라이브러리에 맡기세요:
```javascript
size: {
  width: 12240,   // 짧은 변을 너비로 전달
  height: 15840,  // 긴 변을 높이로 전달
  orientation: PageOrientation.LANDSCAPE  // docx-js가 XML에서 교환
},
// 콘텐츠 너비 = 15840 - 왼쪽 여백 - 오른쪽 여백 (긴 변 사용)
```

### 스타일 (내장 헤딩 재정의)

Arial을 기본 폰트로 사용합니다 (보편적 지원). 가독성을 위해 제목은 검정색 유지.

```javascript
const doc = new Document({
  styles: {
    default: { document: { run: { font: "Arial", size: 24 } } }, // 12pt 기본값
    paragraphStyles: [
      // 중요: 내장 스타일을 재정의하려면 정확한 ID 사용
      { id: "Heading1", name: "Heading 1", basedOn: "Normal", next: "Normal", quickFormat: true,
        run: { size: 32, bold: true, font: "Arial" },
        paragraph: { spacing: { before: 240, after: 240 }, outlineLevel: 0 } }, // TOC를 위해 outlineLevel 필수
      { id: "Heading2", name: "Heading 2", basedOn: "Normal", next: "Normal", quickFormat: true,
        run: { size: 28, bold: true, font: "Arial" },
        paragraph: { spacing: { before: 180, after: 180 }, outlineLevel: 1 } },
    ]
  },
  sections: [{
    children: [
      new Paragraph({ heading: HeadingLevel.HEADING_1, children: [new TextRun("제목")] }),
    ]
  }]
});
```

### 목록 (유니코드 글머리 기호 절대 사용 금지)

```javascript
// ❌ 잘못됨 - 글머리 기호 문자를 수동으로 삽입하지 마세요
new Paragraph({ children: [new TextRun("• 항목")] })  // 나쁨
new Paragraph({ children: [new TextRun("\u2022 항목")] })  // 나쁨

// ✅ 올바름 - LevelFormat.BULLET과 함께 numbering config 사용
const doc = new Document({
  numbering: {
    config: [
      { reference: "bullets",
        levels: [{ level: 0, format: LevelFormat.BULLET, text: "•", alignment: AlignmentType.LEFT,
          style: { paragraph: { indent: { left: 720, hanging: 360 } } } }] },
      { reference: "numbers",
        levels: [{ level: 0, format: LevelFormat.DECIMAL, text: "%1.", alignment: AlignmentType.LEFT,
          style: { paragraph: { indent: { left: 720, hanging: 360 } } } }] },
    ]
  },
  sections: [{
    children: [
      new Paragraph({ numbering: { reference: "bullets", level: 0 },
        children: [new TextRun("글머리 기호 항목")] }),
      new Paragraph({ numbering: { reference: "numbers", level: 0 },
        children: [new TextRun("번호 매기기 항목")] }),
    ]
  }]
});

// ⚠️ 각 reference는 독립적인 번호 매기기 생성
// 같은 reference = 계속 (1,2,3 이후 4,5,6)
// 다른 reference = 재시작 (1,2,3 이후 1,2,3)
```

### 표

**중요: 표는 이중 너비가 필요** - 표의 `columnWidths`와 각 셀의 `width` 모두 설정. 둘 다 없으면 일부 플랫폼에서 표가 올바르게 렌더링되지 않음.

```javascript
// 중요: 일관된 렌더링을 위해 항상 표 너비 설정
// 중요: 검정 배경을 방지하려면 ShadingType.CLEAR (SOLID 아님) 사용
const border = { style: BorderStyle.SINGLE, size: 1, color: "CCCCCC" };
const borders = { top: border, bottom: border, left: border, right: border };

new Table({
  width: { size: 9360, type: WidthType.DXA }, // 항상 DXA 사용 (퍼센트는 Google Docs에서 깨짐)
  columnWidths: [4680, 4680], // 표 너비의 합이 되어야 함 (DXA: 1440 = 1인치)
  rows: [
    new TableRow({
      children: [
        new TableCell({
          borders,
          width: { size: 4680, type: WidthType.DXA }, // 각 셀에도 설정
          shading: { fill: "D5E8F0", type: ShadingType.CLEAR }, // SOLID 아닌 CLEAR
          margins: { top: 80, bottom: 80, left: 120, right: 120 }, // 셀 패딩 (내부, 너비에 추가되지 않음)
          children: [new Paragraph({ children: [new TextRun("셀")] })]
        })
      ]
    })
  ]
})
```

**표 너비 계산:**

항상 `WidthType.DXA` 사용 — `WidthType.PERCENTAGE`는 Google Docs에서 깨짐.

```javascript
// 표 너비 = columnWidths의 합 = 콘텐츠 너비
// 1인치 여백의 US Letter: 12240 - 2880 = 9360 DXA
width: { size: 9360, type: WidthType.DXA },
columnWidths: [7000, 2360]  // 표 너비의 합이 되어야 함
```

**너비 규칙:**
- **항상 `WidthType.DXA` 사용** — `WidthType.PERCENTAGE`는 절대 사용 금지 (Google Docs와 비호환)
- 표 너비는 `columnWidths`의 합과 같아야 함
- 셀 `width`는 해당 `columnWidth`와 일치해야 함
- 셀 `margins`는 내부 패딩 - 셀 너비가 아닌 콘텐츠 영역을 줄임
- 전체 너비 표의 경우: 콘텐츠 너비 사용 (페이지 너비 - 왼쪽 여백 - 오른쪽 여백)

### 이미지

```javascript
// 중요: type 파라미터 필수
new Paragraph({
  children: [new ImageRun({
    type: "png", // 필수: png, jpg, jpeg, gif, bmp, svg
    data: fs.readFileSync("image.png"),
    transformation: { width: 200, height: 150 },
    altText: { title: "제목", description: "설명", name: "이름" } // 세 개 모두 필수
  })]
})
```

### 페이지 나누기

```javascript
// 중요: PageBreak는 Paragraph 안에 있어야 함
new Paragraph({ children: [new PageBreak()] })

// 또는 pageBreakBefore 사용
new Paragraph({ pageBreakBefore: true, children: [new TextRun("새 페이지")] })
```

### 목차

```javascript
// 중요: 헤딩은 HeadingLevel만 사용 - 헤딩 단락에 커스텀 스타일 사용 금지
new TableOfContents("목차", { hyperlink: true, headingStyleRange: "1-3" })
```

### 헤더/푸터

```javascript
sections: [{
  properties: {
    page: { margin: { top: 1440, right: 1440, bottom: 1440, left: 1440 } } // 1440 = 1인치
  },
  headers: {
    default: new Header({ children: [new Paragraph({ children: [new TextRun("헤더")] })] })
  },
  footers: {
    default: new Footer({ children: [new Paragraph({
      children: [new TextRun("페이지 "), new TextRun({ children: [PageNumber.CURRENT] })]
    })] })
  },
  children: [/* 콘텐츠 */]
}]
```

### docx-js 필수 규칙

- **페이지 크기 명시적 설정** - docx-js는 기본적으로 A4; 미국 문서에는 US Letter (12240 x 15840 DXA) 사용
- **가로 방향: 세로 치수 전달** - docx-js는 너비/높이를 내부적으로 교환; 짧은 변을 `width`, 긴 변을 `height`로 전달하고 `orientation: PageOrientation.LANDSCAPE` 설정
- **`\n` 절대 사용 금지** - 별도의 Paragraph 요소 사용
- **유니코드 글머리 기호 절대 사용 금지** - numbering config와 함께 `LevelFormat.BULLET` 사용
- **PageBreak는 Paragraph 안에** - 독립적으로 사용하면 유효하지 않은 XML 생성
- **ImageRun에 `type` 필수** - 항상 png/jpg/etc 지정
- **항상 DXA로 표 `width` 설정** - `WidthType.PERCENTAGE` 절대 사용 금지 (Google Docs에서 깨짐)
- **표는 이중 너비 필요** - `columnWidths` 배열과 셀 `width` 모두 일치해야 함
- **표 너비 = columnWidths의 합** - DXA의 경우 정확하게 더해야 함
- **항상 셀 여백 추가** - 가독성 있는 패딩을 위해 `margins: { top: 80, bottom: 80, left: 120, right: 120 }` 사용
- **`ShadingType.CLEAR` 사용** - 표 음영에 SOLID 절대 사용 금지
- **TOC는 HeadingLevel만 필요** - 헤딩 단락에 커스텀 스타일 사용 금지
- **내장 스타일 재정의** - 정확한 ID 사용: "Heading1", "Heading2" 등
- **`outlineLevel` 포함** - TOC에 필수 (H1은 0, H2는 1 등)

---

## 기존 문서 편집

**3단계를 순서대로 모두 따르세요.**

### Step 1: 압축 해제
```bash
python scripts/office/unpack.py document.docx unpacked/
```
XML을 추출하고, 보기 좋게 출력하며, 인접한 실행을 병합하고, 스마트 따옴표를 XML 엔티티(`&#x201C;` 등)로 변환하여 편집 중 유지됩니다. 실행 병합을 건너뛰려면 `--merge-runs false` 사용.

### Step 2: XML 편집

`unpacked/word/`의 파일을 편집합니다. 패턴은 아래 XML 참조 참고.

사용자가 다른 이름을 명시적으로 요청하지 않는 한, 추적 변경 및 댓글의 작성자로 **"Claude"** 사용.

**문자열 교체에 Edit 도구를 직접 사용하세요. Python 스크립트를 작성하지 마세요.** 스크립트는 불필요한 복잡성을 도입합니다. Edit 도구는 교체되는 내용을 정확히 보여줍니다.

**중요: 새 콘텐츠에 스마트 따옴표 사용.** 아포스트로피나 따옴표가 있는 텍스트 추가 시 XML 엔티티를 사용하여 스마트 따옴표 생성:
```xml
<!-- 전문적인 타이포그래피를 위해 이 엔티티 사용 -->
<w:t>Here&#x2019;s a quote: &#x201C;Hello&#x201D;</w:t>
```
| 엔티티 | 문자 |
|--------|-----------|
| `&#x2018;` | ' (왼쪽 단일) |
| `&#x2019;` | ' (오른쪽 단일 / 아포스트로피) |
| `&#x201C;` | " (왼쪽 이중) |
| `&#x201D;` | " (오른쪽 이중) |

**댓글 추가:** 여러 XML 파일에서 보일러플레이트를 처리하려면 `comment.py` 사용 (텍스트는 미리 이스케이프된 XML이어야 함):
```bash
python scripts/comment.py unpacked/ 0 "Comment text with &amp; and &#x2019;"
python scripts/comment.py unpacked/ 1 "Reply text" --parent 0  # 댓글 0에 답글
python scripts/comment.py unpacked/ 0 "Text" --author "Custom Author"  # 커스텀 작성자
```
그런 다음 document.xml에 마커를 추가합니다 (XML 참조의 댓글 섹션 참고).

### Step 3: 재압축
```bash
python scripts/office/pack.py unpacked/ output.docx --original document.docx
```
자동 수정으로 유효성 검사하고, XML을 압축하고, DOCX를 생성합니다. 건너뛰려면 `--validate false` 사용.

**자동 수정이 수정하는 것:**
- 0x7FFFFFFF 이상의 `durableId` (유효한 ID 재생성)
- 공백이 있는 `<w:t>`의 누락된 `xml:space="preserve"`

**자동 수정이 수정하지 못하는 것:**
- 잘못된 형식의 XML, 유효하지 않은 요소 중첩, 누락된 관계, 스키마 위반

### 일반적인 함정

- **전체 `<w:r>` 요소 교체**: 추적 변경 추가 시 전체 `<w:r>...</w:r>` 블록을 형제로서 `<w:del>...<w:ins>...`으로 교체. 실행 내부에 추적 변경 태그를 주입하지 마세요.
- **`<w:rPr>` 포맷팅 유지**: 굵게, 폰트 크기 등을 유지하기 위해 원본 실행의 `<w:rPr>` 블록을 추적 변경 실행에 복사.

---

## XML 참조

### 스키마 준수

- **`<w:pPr>` 내 요소 순서**: `<w:pStyle>`, `<w:numPr>`, `<w:spacing>`, `<w:ind>`, `<w:jc>`, 마지막에 `<w:rPr>`
- **공백**: 앞뒤 공백이 있는 `<w:t>`에 `xml:space="preserve"` 추가
- **RSIDs**: 8자리 16진수여야 함 (예: `00AB1234`)

### 추적 변경

**삽입:**
```xml
<w:ins w:id="1" w:author="Claude" w:date="2025-01-01T00:00:00Z">
  <w:r><w:t>삽입된 텍스트</w:t></w:r>
</w:ins>
```

**삭제:**
```xml
<w:del w:id="2" w:author="Claude" w:date="2025-01-01T00:00:00Z">
  <w:r><w:delText>삭제된 텍스트</w:delText></w:r>
</w:del>
```

**`<w:del>` 내부**: `<w:t>` 대신 `<w:delText>`, `<w:instrText>` 대신 `<w:delInstrText>` 사용.

**최소 편집** - 변경되는 것만 표시:
```xml
<!-- "30일"을 "60일"로 변경 -->
<w:r><w:t>기간은 </w:t></w:r>
<w:del w:id="1" w:author="Claude" w:date="...">
  <w:r><w:delText>30</w:delText></w:r>
</w:del>
<w:ins w:id="2" w:author="Claude" w:date="...">
  <w:r><w:t>60</w:t></w:r>
</w:ins>
<w:r><w:t>일.</w:t></w:r>
```

**전체 단락/목록 항목 삭제** - 단락의 모든 콘텐츠를 제거할 때, 다음 단락과 병합되도록 단락 마크도 삭제로 표시. `<w:pPr><w:rPr>` 내부에 `<w:del/>` 추가:
```xml
<w:p>
  <w:pPr>
    <w:numPr>...</w:numPr>  <!-- 목록 번호 매기기 (있는 경우) -->
    <w:rPr>
      <w:del w:id="1" w:author="Claude" w:date="2025-01-01T00:00:00Z"/>
    </w:rPr>
  </w:pPr>
  <w:del w:id="2" w:author="Claude" w:date="2025-01-01T00:00:00Z">
    <w:r><w:delText>삭제되는 전체 단락 콘텐츠...</w:delText></w:r>
  </w:del>
</w:p>
```
`<w:pPr><w:rPr>`의 `<w:del/>`이 없으면 변경 사항을 수락할 때 빈 단락/목록 항목이 남습니다.

**다른 작성자의 삽입 거부** - 그들의 삽입 내부에 삭제를 중첩:
```xml
<w:ins w:author="Jane" w:id="5">
  <w:del w:author="Claude" w:id="10">
    <w:r><w:delText>그들이 삽입한 텍스트</w:delText></w:r>
  </w:del>
</w:ins>
```

**다른 작성자의 삭제 복원** - 이후에 삽입 추가 (그들의 삭제를 수정하지 마세요):
```xml
<w:del w:author="Jane" w:id="5">
  <w:r><w:delText>삭제된 텍스트</w:delText></w:r>
</w:del>
<w:ins w:author="Claude" w:id="10">
  <w:r><w:t>삭제된 텍스트</w:t></w:r>
</w:ins>
```

### 댓글

`comment.py` 실행 후 (Step 2 참고), document.xml에 마커를 추가합니다. 답글의 경우 `--parent` 플래그를 사용하고 부모 마커 내부에 중첩합니다.

**중요: `<w:commentRangeStart>`와 `<w:commentRangeEnd>`는 `<w:r>`의 형제이며, `<w:r>` 내부에 있으면 안 됩니다.**

```xml
<!-- 댓글 마커는 w:p의 직접 자식, w:r 내부에 절대 있으면 안 됨 -->
<w:commentRangeStart w:id="0"/>
<w:del w:id="1" w:author="Claude" w:date="2025-01-01T00:00:00Z">
  <w:r><w:delText>삭제됨</w:delText></w:r>
</w:del>
<w:r><w:t> 더 많은 텍스트</w:t></w:r>
<w:commentRangeEnd w:id="0"/>
<w:r><w:rPr><w:rStyle w:val="CommentReference"/></w:rPr><w:commentReference w:id="0"/></w:r>

<!-- 답글 1이 내부에 중첩된 댓글 0 -->
<w:commentRangeStart w:id="0"/>
  <w:commentRangeStart w:id="1"/>
  <w:r><w:t>텍스트</w:t></w:r>
  <w:commentRangeEnd w:id="1"/>
<w:commentRangeEnd w:id="0"/>
<w:r><w:rPr><w:rStyle w:val="CommentReference"/></w:rPr><w:commentReference w:id="0"/></w:r>
<w:r><w:rPr><w:rStyle w:val="CommentReference"/></w:rPr><w:commentReference w:id="1"/></w:r>
```

### 이미지

1. `word/media/`에 이미지 파일 추가
2. `word/_rels/document.xml.rels`에 관계 추가:
```xml
<Relationship Id="rId5" Type=".../image" Target="media/image1.png"/>
```
3. `[Content_Types].xml`에 콘텐츠 타입 추가:
```xml
<Default Extension="png" ContentType="image/png"/>
```
4. document.xml에서 참조:
```xml
<w:drawing>
  <wp:inline>
    <wp:extent cx="914400" cy="914400"/>  <!-- EMUs: 914400 = 1인치 -->
    <a:graphic>
      <a:graphicData uri=".../picture">
        <pic:pic>
          <pic:blipFill><a:blip r:embed="rId5"/></pic:blipFill>
        </pic:pic>
      </a:graphicData>
    </a:graphic>
  </wp:inline>
</w:drawing>
```

---

## 의존성

- **pandoc**: 텍스트 추출
- **docx**: `npm install -g docx` (새 문서)
- **LibreOffice**: PDF 변환 (샌드박스 환경을 위해 `scripts/office/soffice.py`를 통해 자동 설정)
- **Poppler**: 이미지용 `pdftoppm`
