# HTML을 PowerPoint로 변환 가이드

`html2pptx.js` 라이브러리를 사용하여 HTML 슬라이드를 정확한 위치 지정으로 PowerPoint 프레젠테이션으로 변환합니다.

## 목차

1. [HTML 슬라이드 생성](#html-슬라이드-생성)
2. [html2pptx 라이브러리 사용](#html2pptx-라이브러리-사용)
3. [PptxGenJS 사용](#pptxgenjs-사용)

---

## HTML 슬라이드 생성

모든 HTML 슬라이드에는 적절한 body 치수가 포함되어야 합니다:

### 레이아웃 치수

- **16:9** (기본값): `width: 720pt; height: 405pt`
- **4:3**: `width: 720pt; height: 540pt`
- **16:10**: `width: 720pt; height: 450pt`

### 지원 요소

- `<p>`, `<h1>`-`<h6>` - 스타일이 있는 텍스트
- `<ul>`, `<ol>` - 목록 (수동 불릿 •, -, * 절대 사용 금지)
- `<b>`, `<strong>` - 굵은 텍스트 (인라인 포맷팅)
- `<i>`, `<em>` - 기울임 텍스트 (인라인 포맷팅)
- `<u>` - 밑줄 텍스트 (인라인 포맷팅)
- `<span>` - CSS 스타일이 있는 인라인 포맷팅 (굵기, 기울임, 밑줄, 색상)
- `<br>` - 줄바꿈
- bg/border가 있는 `<div>` - 도형이 됨
- `<img>` - 이미지
- `class="placeholder"` - 차트를 위한 예약 공간 (`{ id, x, y, w, h }` 반환)

### 중요 텍스트 규칙

**모든 텍스트는 반드시 `<p>`, `<h1>`-`<h6>`, `<ul>`, 또는 `<ol>` 태그 안에 있어야 함:**
- 올바른 예: `<div><p>Text here</p></div>`
- 잘못된 예: `<div>Text here</div>` - **텍스트가 PowerPoint에 나타나지 않음**
- 잘못된 예: `<span>Text</span>` - **텍스트가 PowerPoint에 나타나지 않음**
- 텍스트 태그 없이 `<div>` 또는 `<span>` 안의 텍스트는 조용히 무시됨

**수동 불릿 기호 (•, -, *, 등) 절대 사용 금지** - 대신 `<ul>` 또는 `<ol>` 목록 사용

**보편적으로 사용 가능한 웹 안전 폰트만 사용:**
- 올바른 폰트: `Arial`, `Helvetica`, `Times New Roman`, `Georgia`, `Courier New`, `Verdana`, `Tahoma`, `Trebuchet MS`, `Impact`, `Comic Sans MS`
- 잘못된 폰트: `'Segoe UI'`, `'SF Pro'`, `'Roboto'`, 커스텀 폰트 - **렌더링 문제 발생 가능**

### 스타일링

- 오버플로우 검증에서 마진 붕괴를 방지하기 위해 body에 `display: flex` 사용
- 간격에는 `margin` 사용 (패딩은 크기에 포함됨)
- 인라인 포맷팅: `<b>`, `<i>`, `<u>` 태그 또는 CSS 스타일이 있는 `<span>` 사용
  - `<span>` 지원: `font-weight: bold`, `font-style: italic`, `text-decoration: underline`, `color: #rrggbb`
  - `<span>` 미지원: `margin`, `padding` (PowerPoint 텍스트 런에서 미지원)
  - 예시: `<span style="font-weight: bold; color: #667eea;">Bold blue text</span>`
- Flexbox 사용 가능 - 렌더링된 레이아웃에서 위치 계산
- CSS에서 `#` 접두사와 함께 hex 색상 사용
- **텍스트 정렬**: 텍스트 길이가 약간 맞지 않을 때 PptxGenJS에 텍스트 포맷팅 힌트로 CSS `text-align` (`center`, `right` 등) 사용

### 도형 스타일링 (DIV 요소만)

**중요: 배경, 테두리, 그림자는 `<div>` 요소에서만 작동하고, 텍스트 요소 (`<p>`, `<h1>`-`<h6>`, `<ul>`, `<ol>`)에서는 작동하지 않음**

- **배경**: `<div>` 요소에만 CSS `background` 또는 `background-color`
  - 예시: `<div style="background: #f0f0f0;">` - 배경이 있는 도형 생성
- **테두리**: `<div>` 요소의 CSS `border`가 PowerPoint 도형 테두리로 변환
  - 균일한 테두리 지원: `border: 2px solid #333333`
  - 부분 테두리 지원: `border-left`, `border-right`, `border-top`, `border-bottom` (선 도형으로 렌더링)
  - 예시: `<div style="border-left: 8pt solid #E76F51;">`
- **테두리 반경**: `<div>` 요소의 CSS `border-radius`로 둥근 모서리
  - `border-radius: 50%` 이상은 원형 도형 생성
  - 50% 미만 백분율은 도형의 작은 치수에 상대적으로 계산
  - px 및 pt 단위 지원 (예: `border-radius: 8pt;`, `border-radius: 12px;`)
  - 예시: 100x200px 박스에서 `<div style="border-radius: 25%;">` = 100px의 25% = 25px 반경
- **박스 그림자**: `<div>` 요소의 CSS `box-shadow`가 PowerPoint 그림자로 변환
  - 외부 그림자만 지원 (안쪽 그림자는 손상 방지를 위해 무시)
  - 예시: `<div style="box-shadow: 2px 2px 8px rgba(0, 0, 0, 0.3);">`
  - 참고: 인셋/내부 그림자는 PowerPoint에서 미지원이므로 건너뜀

### 아이콘 & 그라디언트

- **중요: CSS 그라디언트 (`linear-gradient`, `radial-gradient`) 절대 사용 금지** - PowerPoint로 변환되지 않음
- **항상 Sharp를 사용하여 그라디언트/아이콘 PNG를 먼저 생성한 후 HTML에서 참조**
- 그라디언트의 경우: SVG를 PNG 배경 이미지로 래스터화
- 아이콘의 경우: react-icons SVG를 PNG 이미지로 래스터화
- 모든 시각적 효과는 HTML 렌더링 전에 래스터 이미지로 미리 렌더링되어야 함

**Sharp를 사용한 아이콘 래스터화:**

```javascript
const React = require('react');
const ReactDOMServer = require('react-dom/server');
const sharp = require('sharp');
const { FaHome } = require('react-icons/fa');

async function rasterizeIconPng(IconComponent, color, size = "256", filename) {
  const svgString = ReactDOMServer.renderToStaticMarkup(
    React.createElement(IconComponent, { color: `#${color}`, size: size })
  );

  // Sharp를 사용하여 SVG를 PNG로 변환
  await sharp(Buffer.from(svgString))
    .png()
    .toFile(filename);

  return filename;
}

// 사용법: HTML에 사용하기 전에 아이콘 래스터화
const iconPath = await rasterizeIconPng(FaHome, "4472c4", "256", "home-icon.png");
// 그런 다음 HTML에서 참조: <img src="home-icon.png" style="width: 40pt; height: 40pt;">
```

**Sharp를 사용한 그라디언트 래스터화:**

```javascript
const sharp = require('sharp');

async function createGradientBackground(filename) {
  const svg = `<svg xmlns="http://www.w3.org/2000/svg" width="1000" height="562.5">
    <defs>
      <linearGradient id="g" x1="0%" y1="0%" x2="100%" y2="100%">
        <stop offset="0%" style="stop-color:#COLOR1"/>
        <stop offset="100%" style="stop-color:#COLOR2"/>
      </linearGradient>
    </defs>
    <rect width="100%" height="100%" fill="url(#g)"/>
  </svg>`;

  await sharp(Buffer.from(svg))
    .png()
    .toFile(filename);

  return filename;
}

// 사용법: HTML 전에 그라디언트 배경 생성
const bgPath = await createGradientBackground("gradient-bg.png");
// 그런 다음 HTML에서: <body style="background-image: url('gradient-bg.png');">
```

### 예시

```html
<!DOCTYPE html>
<html>
<head>
<style>
html { background: #ffffff; }
body {
  width: 720pt; height: 405pt; margin: 0; padding: 0;
  background: #f5f5f5; font-family: Arial, sans-serif;
  display: flex;
}
.content { margin: 30pt; padding: 40pt; background: #ffffff; border-radius: 8pt; }
h1 { color: #2d3748; font-size: 32pt; }
.box {
  background: #70ad47; padding: 20pt; border: 3px solid #5a8f37;
  border-radius: 12pt; box-shadow: 3px 3px 10px rgba(0, 0, 0, 0.25);
}
</style>
</head>
<body>
<div class="content">
  <h1>Recipe Title</h1>
  <ul>
    <li><b>Item:</b> Description</li>
  </ul>
  <p>Text with <b>bold</b>, <i>italic</i>, <u>underline</u>.</p>
  <div id="chart" class="placeholder" style="width: 350pt; height: 200pt;"></div>

  <!-- 텍스트는 반드시 <p> 태그 안에 있어야 함 -->
  <div class="box">
    <p>5</p>
  </div>
</div>
</body>
</html>
```

## html2pptx 라이브러리 사용

### 의존성

이 라이브러리들은 전역 설치되어 있으며 사용 가능합니다:
- `pptxgenjs`
- `playwright`
- `sharp`

### 기본 사용법

```javascript
const pptxgen = require('pptxgenjs');
const html2pptx = require('./html2pptx');

const pptx = new pptxgen();
pptx.layout = 'LAYOUT_16x9';  // HTML body 치수와 일치해야 함

const { slide, placeholders } = await html2pptx('slide1.html', pptx);

// 플레이스홀더 영역에 차트 추가
if (placeholders.length > 0) {
    slide.addChart(pptx.charts.LINE, chartData, placeholders[0]);
}

await pptx.writeFile('output.pptx');
```

### API 참조

#### 함수 시그니처
```javascript
await html2pptx(htmlFile, pres, options)
```

#### 파라미터
- `htmlFile` (string): HTML 파일 경로 (절대 또는 상대)
- `pres` (pptxgen): 레이아웃이 이미 설정된 PptxGenJS 프레젠테이션 인스턴스
- `options` (object, 선택사항):
  - `tmpDir` (string): 생성된 파일을 위한 임시 디렉토리 (기본값: `process.env.TMPDIR || '/tmp'`)
  - `slide` (object): 재사용할 기존 슬라이드 (기본값: 새 슬라이드 생성)

#### 반환값
```javascript
{
    slide: pptxgenSlide,           // 생성/업데이트된 슬라이드
    placeholders: [                 // 플레이스홀더 위치 배열
        { id: string, x: number, y: number, w: number, h: number },
        ...
    ]
}
```

### 검증

라이브러리는 모든 오류를 수집한 후 throw합니다:

1. **HTML 치수가 프레젠테이션 레이아웃과 일치해야 함** - 치수 불일치 보고
2. **콘텐츠가 body를 넘치면 안 됨** - 정확한 측정값과 함께 오버플로우 보고
3. **CSS 그라디언트** - 미지원 그라디언트 사용 보고
4. **텍스트 요소 스타일링** - 텍스트 요소의 배경/테두리/그림자 보고 (div에서만 허용)

**모든 검증 오류는 수집되어 단일 오류 메시지로 함께 보고**되므로 한 번에 모든 문제 수정 가능

### 플레이스홀더 작업

```javascript
const { slide, placeholders } = await html2pptx('slide.html', pptx);

// 첫 번째 플레이스홀더 사용
slide.addChart(pptx.charts.BAR, data, placeholders[0]);

// ID로 찾기
const chartArea = placeholders.find(p => p.id === 'chart-area');
slide.addChart(pptx.charts.LINE, data, chartArea);
```

### 완전한 예시

```javascript
const pptxgen = require('pptxgenjs');
const html2pptx = require('./html2pptx');

async function createPresentation() {
    const pptx = new pptxgen();
    pptx.layout = 'LAYOUT_16x9';
    pptx.author = 'Your Name';
    pptx.title = 'My Presentation';

    // 슬라이드 1: 타이틀
    const { slide: slide1 } = await html2pptx('slides/title.html', pptx);

    // 슬라이드 2: 차트가 있는 콘텐츠
    const { slide: slide2, placeholders } = await html2pptx('slides/data.html', pptx);

    const chartData = [{
        name: 'Sales',
        labels: ['Q1', 'Q2', 'Q3', 'Q4'],
        values: [4500, 5500, 6200, 7100]
    }];

    slide2.addChart(pptx.charts.BAR, chartData, {
        ...placeholders[0],
        showTitle: true,
        title: 'Quarterly Sales',
        showCatAxisTitle: true,
        catAxisTitle: 'Quarter',
        showValAxisTitle: true,
        valAxisTitle: 'Sales ($000s)'
    });

    // 저장
    await pptx.writeFile({ fileName: 'presentation.pptx' });
    console.log('Presentation created successfully!');
}

createPresentation().catch(console.error);
```

## PptxGenJS 사용

HTML을 `html2pptx`로 슬라이드로 변환한 후 PptxGenJS를 사용하여 차트, 이미지, 추가 요소 등 동적 콘텐츠를 추가합니다.

### 중요 규칙

#### 색상
- **PptxGenJS에서 hex 색상에 `#` 접두사 절대 사용 금지** - 파일 손상 원인
- 올바른 예: `color: "FF0000"`, `fill: { color: "0066CC" }`
- 잘못된 예: `color: "#FF0000"` (문서 손상)

### 이미지 추가

항상 실제 이미지 치수에서 종횡비를 계산합니다:

```javascript
// 이미지 치수 가져오기: identify image.png | grep -o '[0-9]* x [0-9]*'
const imgWidth = 1860, imgHeight = 1519;  // 실제 파일에서
const aspectRatio = imgWidth / imgHeight;

const h = 3;  // 최대 높이
const w = h * aspectRatio;
const x = (10 - w) / 2;  // 16:9 슬라이드에서 중앙 정렬

slide.addImage({ path: "chart.png", x, y: 1.5, w, h });
```

### 텍스트 추가

```javascript
// 포맷팅이 있는 리치 텍스트
slide.addText([
    { text: "Bold ", options: { bold: true } },
    { text: "Italic ", options: { italic: true } },
    { text: "Normal" }
], {
    x: 1, y: 2, w: 8, h: 1
});
```

### 도형 추가

```javascript
// 사각형
slide.addShape(pptx.shapes.RECTANGLE, {
    x: 1, y: 1, w: 3, h: 2,
    fill: { color: "4472C4" },
    line: { color: "000000", width: 2 }
});

// 원
slide.addShape(pptx.shapes.OVAL, {
    x: 5, y: 1, w: 2, h: 2,
    fill: { color: "ED7D31" }
});

// 둥근 사각형
slide.addShape(pptx.shapes.ROUNDED_RECTANGLE, {
    x: 1, y: 4, w: 3, h: 1.5,
    fill: { color: "70AD47" },
    rectRadius: 0.2
});
```

### 차트 추가

**대부분의 차트에 필수**: `catAxisTitle` (카테고리) 및 `valAxisTitle` (값)을 사용한 축 레이블.

**차트 데이터 형식:**
- 단순 막대/선 차트에는 **모든 레이블이 있는 단일 시리즈** 사용
- 각 시리즈는 별도의 범례 항목 생성
- 레이블 배열이 X축 값 정의

**시계열 데이터 - 올바른 세분성 선택:**
- **30일 미만**: 일별 그룹화 사용 (예: "10-01", "10-02") - 단일 포인트 차트를 만드는 월별 집계 지양
- **30-365일**: 월별 그룹화 사용 (예: "2024-01", "2024-02")
- **365일 초과**: 연별 그룹화 사용 (예: "2023", "2024")
- **검증**: 데이터 포인트가 1개뿐인 차트는 기간에 맞지 않는 잘못된 집계를 의미할 가능성 높음

```javascript
const { slide, placeholders } = await html2pptx('slide.html', pptx);

// 올바른 예: 모든 레이블이 있는 단일 시리즈
slide.addChart(pptx.charts.BAR, [{
    name: "Sales 2024",
    labels: ["Q1", "Q2", "Q3", "Q4"],
    values: [4500, 5500, 6200, 7100]
}], {
    ...placeholders[0],  // 플레이스홀더 위치 사용
    barDir: 'col',       // 'col' = 수직 막대, 'bar' = 수평
    showTitle: true,
    title: 'Quarterly Sales',
    showLegend: false,   // 단일 시리즈에는 범례 불필요
    // 필수 축 레이블
    showCatAxisTitle: true,
    catAxisTitle: 'Quarter',
    showValAxisTitle: true,
    valAxisTitle: 'Sales ($000s)',
    // 선택사항: 스케일 제어 (더 나은 시각화를 위해 데이터 범위에 맞게 최소값 조정)
    valAxisMaxVal: 8000,
    valAxisMinVal: 0,  // 카운트/금액에는 0 사용; 클러스터된 데이터 (예: 4500-7100)에는 최소값에 더 가깝게 시작 고려
    valAxisMajorUnit: 2000,  // 혼잡 방지를 위한 y축 레이블 간격 제어
    catAxisLabelRotate: 45,  // 혼잡할 경우 레이블 회전
    dataLabelPosition: 'outEnd',
    dataLabelColor: '000000',
    // 단일 시리즈 차트에는 단일 색상 사용
    chartColors: ["4472C4"]  // 모든 막대 같은 색상
});
```

#### 산점도

**중요**: 산점도 데이터 형식은 특이함 - 첫 번째 시리즈에는 X축 값, 이후 시리즈에는 Y값 포함:

```javascript
// 데이터 준비
const data1 = [{ x: 10, y: 20 }, { x: 15, y: 25 }, { x: 20, y: 30 }];
const data2 = [{ x: 12, y: 18 }, { x: 18, y: 22 }];

const allXValues = [...data1.map(d => d.x), ...data2.map(d => d.x)];

slide.addChart(pptx.charts.SCATTER, [
    { name: 'X-Axis', values: allXValues },  // 첫 번째 시리즈 = X 값
    { name: 'Series 1', values: data1.map(d => d.y) },  // Y 값만
    { name: 'Series 2', values: data2.map(d => d.y) }   // Y 값만
], {
    x: 1, y: 1, w: 8, h: 4,
    lineSize: 0,  // 0 = 연결선 없음
    lineDataSymbol: 'circle',
    lineDataSymbolSize: 6,
    showCatAxisTitle: true,
    catAxisTitle: 'X Axis',
    showValAxisTitle: true,
    valAxisTitle: 'Y Axis',
    chartColors: ["4472C4", "ED7D31"]
});
```

#### 선 차트

```javascript
slide.addChart(pptx.charts.LINE, [{
    name: "Temperature",
    labels: ["Jan", "Feb", "Mar", "Apr"],
    values: [32, 35, 42, 55]
}], {
    x: 1, y: 1, w: 8, h: 4,
    lineSize: 4,
    lineSmooth: true,
    // 필수 축 레이블
    showCatAxisTitle: true,
    catAxisTitle: 'Month',
    showValAxisTitle: true,
    valAxisTitle: 'Temperature (°F)',
    // 선택사항: Y축 범위 (더 나은 시각화를 위해 데이터 범위에 맞게 최소값 설정)
    valAxisMinVal: 0,     // 0에서 시작하는 범위용 (카운트, 백분율 등)
    valAxisMaxVal: 60,
    valAxisMajorUnit: 20,  // 혼잡 방지를 위한 y축 레이블 간격 제어 (예: 10, 20, 25)
    // valAxisMinVal: 30,  // 권장: 범위에 클러스터된 데이터 (예: 32-55 또는 평점 3-5)에는 변동 표시를 위해 최소값에 가깝게 시작
    // 선택사항: 차트 색상
    chartColors: ["4472C4", "ED7D31", "A5A5A5"]
});
```

#### 파이 차트 (축 레이블 불필요)

**중요**: 파이 차트는 `labels` 배열에 모든 카테고리, `values` 배열에 해당 값이 있는 **단일 데이터 시리즈**가 필요합니다.

```javascript
slide.addChart(pptx.charts.PIE, [{
    name: "Market Share",
    labels: ["Product A", "Product B", "Other"],  // 하나의 배열에 모든 카테고리
    values: [35, 45, 20]  // 하나의 배열에 모든 값
}], {
    x: 2, y: 1, w: 6, h: 4,
    showPercent: true,
    showLegend: true,
    legendPos: 'r',  // 오른쪽
    chartColors: ["4472C4", "ED7D31", "A5A5A5"]
});
```

#### 다중 데이터 시리즈

```javascript
slide.addChart(pptx.charts.LINE, [
    {
        name: "Product A",
        labels: ["Q1", "Q2", "Q3", "Q4"],
        values: [10, 20, 30, 40]
    },
    {
        name: "Product B",
        labels: ["Q1", "Q2", "Q3", "Q4"],
        values: [15, 25, 20, 35]
    }
], {
    x: 1, y: 1, w: 8, h: 4,
    showCatAxisTitle: true,
    catAxisTitle: 'Quarter',
    showValAxisTitle: true,
    valAxisTitle: 'Revenue ($M)'
});
```

### 차트 색상

**중요**: `#` 접두사 **없이** hex 색상 사용 - `#`을 포함하면 파일 손상 발생.

**선택한 디자인 팔레트에 맞게 차트 색상 조정**하여 데이터 시각화에 충분한 대비와 구별성 확보. 다음을 위해 색상 조정:
- 인접 시리즈 간 강한 대비
- 슬라이드 배경에 대한 가독성
- 접근성 (빨강-초록 조합만 사용 지양)

```javascript
// 예시: Ocean 팔레트에서 영감을 받은 차트 색상 (대비를 위해 조정됨)
const chartColors = ["16A085", "FF6B9D", "2C3E50", "F39C12", "9B59B6"];

// 단일 시리즈 차트: 모든 막대/점에 한 가지 색상 사용
slide.addChart(pptx.charts.BAR, [{
    name: "Sales",
    labels: ["Q1", "Q2", "Q3", "Q4"],
    values: [4500, 5500, 6200, 7100]
}], {
    ...placeholders[0],
    chartColors: ["16A085"],  // 모든 막대 같은 색상
    showLegend: false
});

// 다중 시리즈 차트: 각 시리즈마다 다른 색상
slide.addChart(pptx.charts.LINE, [
    { name: "Product A", labels: ["Q1", "Q2", "Q3"], values: [10, 20, 30] },
    { name: "Product B", labels: ["Q1", "Q2", "Q3"], values: [15, 25, 20] }
], {
    ...placeholders[0],
    chartColors: ["16A085", "FF6B9D"]  // 시리즈당 하나의 색상
});
```

### 테이블 추가

기본 또는 고급 포맷팅으로 테이블을 추가할 수 있습니다:

#### 기본 테이블

```javascript
slide.addTable([
    ["Header 1", "Header 2", "Header 3"],
    ["Row 1, Col 1", "Row 1, Col 2", "Row 1, Col 3"],
    ["Row 2, Col 1", "Row 2, Col 2", "Row 2, Col 3"]
], {
    x: 0.5,
    y: 1,
    w: 9,
    h: 3,
    border: { pt: 1, color: "999999" },
    fill: { color: "F1F1F1" }
});
```

#### 사용자 정의 포맷팅이 있는 테이블

```javascript
const tableData = [
    // 사용자 정의 스타일의 헤더 행
    [
        { text: "Product", options: { fill: { color: "4472C4" }, color: "FFFFFF", bold: true } },
        { text: "Revenue", options: { fill: { color: "4472C4" }, color: "FFFFFF", bold: true } },
        { text: "Growth", options: { fill: { color: "4472C4" }, color: "FFFFFF", bold: true } }
    ],
    // 데이터 행
    ["Product A", "$50M", "+15%"],
    ["Product B", "$35M", "+22%"],
    ["Product C", "$28M", "+8%"]
];

slide.addTable(tableData, {
    x: 1,
    y: 1.5,
    w: 8,
    h: 3,
    colW: [3, 2.5, 2.5],  // 열 너비
    rowH: [0.5, 0.6, 0.6, 0.6],  // 행 높이
    border: { pt: 1, color: "CCCCCC" },
    align: "center",
    valign: "middle",
    fontSize: 14
});
```

#### 셀 병합이 있는 테이블

```javascript
const mergedTableData = [
    [
        { text: "Q1 Results", options: { colspan: 3, fill: { color: "4472C4" }, color: "FFFFFF", bold: true } }
    ],
    ["Product", "Sales", "Market Share"],
    ["Product A", "$25M", "35%"],
    ["Product B", "$18M", "25%"]
];

slide.addTable(mergedTableData, {
    x: 1,
    y: 1,
    w: 8,
    h: 2.5,
    colW: [3, 2.5, 2.5],
    border: { pt: 1, color: "DDDDDD" }
});
```

### 테이블 옵션

일반적인 테이블 옵션:
- `x, y, w, h` - 위치 및 크기
- `colW` - 열 너비 배열 (인치 단위)
- `rowH` - 행 높이 배열 (인치 단위)
- `border` - 테두리 스타일: `{ pt: 1, color: "999999" }`
- `fill` - 배경 색상 (# 접두사 없음)
- `align` - 텍스트 정렬: "left", "center", "right"
- `valign` - 수직 정렬: "top", "middle", "bottom"
- `fontSize` - 텍스트 크기
- `autoPage` - 콘텐츠가 넘치면 새 슬라이드 자동 생성
