# PowerPoint용 Office Open XML 기술 참조

**중요: 시작하기 전에 이 문서 전체를 읽으세요.** 중요한 XML 스키마 규칙과 포맷팅 요구사항이 전체에 걸쳐 설명됩니다. 잘못된 구현은 PowerPoint가 열 수 없는 유효하지 않은 PPTX 파일을 만들 수 있습니다.

## 기술 가이드라인

### 스키마 준수
- **`<p:txBody>`의 요소 순서**: `<a:bodyPr>`, `<a:lstStyle>`, `<a:p>`
- **공백**: 앞/뒤에 공백이 있는 `<a:t>` 요소에 `xml:space='preserve'` 추가
- **Unicode**: ASCII 콘텐츠에서 문자 이스케이프: `"` → `&#8220;`
- **이미지**: `ppt/media/`에 추가하고 슬라이드 XML에서 참조, 치수를 슬라이드 범위에 맞게 설정
- **관계**: 각 슬라이드의 리소스에 대해 `ppt/slides/_rels/slideN.xml.rels` 업데이트
- **dirty 속성**: 깨끗한 상태를 나타내기 위해 `<a:rPr>` 및 `<a:endParaRPr>` 요소에 `dirty="0"` 추가

## 프레젠테이션 구조

### 기본 슬라이드 구조
```xml
<!-- ppt/slides/slide1.xml -->
<p:sld>
  <p:cSld>
    <p:spTree>
      <p:nvGrpSpPr>...</p:nvGrpSpPr>
      <p:grpSpPr>...</p:grpSpPr>
      <!-- 도형이 여기에 위치 -->
    </p:spTree>
  </p:cSld>
</p:sld>
```

### 텍스트가 있는 텍스트 박스 / 도형
```xml
<p:sp>
  <p:nvSpPr>
    <p:cNvPr id="2" name="Title"/>
    <p:cNvSpPr>
      <a:spLocks noGrp="1"/>
    </p:cNvSpPr>
    <p:nvPr>
      <p:ph type="ctrTitle"/>
    </p:nvPr>
  </p:nvSpPr>
  <p:spPr>
    <a:xfrm>
      <a:off x="838200" y="365125"/>
      <a:ext cx="7772400" cy="1470025"/>
    </a:xfrm>
  </p:spPr>
  <p:txBody>
    <a:bodyPr/>
    <a:lstStyle/>
    <a:p>
      <a:r>
        <a:t>Slide Title</a:t>
      </a:r>
    </a:p>
  </p:txBody>
</p:sp>
```

### 텍스트 포맷팅
```xml
<!-- 굵게 -->
<a:r>
  <a:rPr b="1"/>
  <a:t>Bold Text</a:t>
</a:r>

<!-- 기울임 -->
<a:r>
  <a:rPr i="1"/>
  <a:t>Italic Text</a:t>
</a:r>

<!-- 밑줄 -->
<a:r>
  <a:rPr u="sng"/>
  <a:t>Underlined</a:t>
</a:r>

<!-- 하이라이트 -->
<a:r>
  <a:rPr>
    <a:highlight>
      <a:srgbClr val="FFFF00"/>
    </a:highlight>
  </a:rPr>
  <a:t>Highlighted Text</a:t>
</a:r>

<!-- 폰트 및 크기 -->
<a:r>
  <a:rPr sz="2400" typeface="Arial">
    <a:solidFill>
      <a:srgbClr val="FF0000"/>
    </a:solidFill>
  </a:rPr>
  <a:t>Colored Arial 24pt</a:t>
</a:r>

<!-- 완전한 포맷팅 예시 -->
<a:r>
  <a:rPr lang="en-US" sz="1400" b="1" dirty="0">
    <a:solidFill>
      <a:srgbClr val="FAFAFA"/>
    </a:solidFill>
  </a:rPr>
  <a:t>Formatted text</a:t>
</a:r>
```

### 목록
```xml
<!-- 불릿 목록 -->
<a:p>
  <a:pPr lvl="0">
    <a:buChar char="•"/>
  </a:pPr>
  <a:r>
    <a:t>First bullet point</a:t>
  </a:r>
</a:p>

<!-- 번호 목록 -->
<a:p>
  <a:pPr lvl="0">
    <a:buAutoNum type="arabicPeriod"/>
  </a:pPr>
  <a:r>
    <a:t>First numbered item</a:t>
  </a:r>
</a:p>

<!-- 두 번째 수준 들여쓰기 -->
<a:p>
  <a:pPr lvl="1">
    <a:buChar char="•"/>
  </a:pPr>
  <a:r>
    <a:t>Indented bullet</a:t>
  </a:r>
</a:p>
```

### 도형
```xml
<!-- 사각형 -->
<p:sp>
  <p:nvSpPr>
    <p:cNvPr id="3" name="Rectangle"/>
    <p:cNvSpPr/>
    <p:nvPr/>
  </p:nvSpPr>
  <p:spPr>
    <a:xfrm>
      <a:off x="1000000" y="1000000"/>
      <a:ext cx="3000000" cy="2000000"/>
    </a:xfrm>
    <a:prstGeom prst="rect">
      <a:avLst/>
    </a:prstGeom>
    <a:solidFill>
      <a:srgbClr val="FF0000"/>
    </a:solidFill>
    <a:ln w="25400">
      <a:solidFill>
        <a:srgbClr val="000000"/>
      </a:solidFill>
    </a:ln>
  </p:spPr>
</p:sp>

<!-- 둥근 사각형 -->
<p:sp>
  <p:spPr>
    <a:prstGeom prst="roundRect">
      <a:avLst/>
    </a:prstGeom>
  </p:spPr>
</p:sp>

<!-- 원/타원 -->
<p:sp>
  <p:spPr>
    <a:prstGeom prst="ellipse">
      <a:avLst/>
    </a:prstGeom>
  </p:spPr>
</p:sp>
```

### 이미지
```xml
<p:pic>
  <p:nvPicPr>
    <p:cNvPr id="4" name="Picture">
      <a:hlinkClick r:id="" action="ppaction://media"/>
    </p:cNvPr>
    <p:cNvPicPr>
      <a:picLocks noChangeAspect="1"/>
    </p:cNvPicPr>
    <p:nvPr/>
  </p:nvPicPr>
  <p:blipFill>
    <a:blip r:embed="rId2"/>
    <a:stretch>
      <a:fillRect/>
    </a:stretch>
  </p:blipFill>
  <p:spPr>
    <a:xfrm>
      <a:off x="1000000" y="1000000"/>
      <a:ext cx="3000000" cy="2000000"/>
    </a:xfrm>
    <a:prstGeom prst="rect">
      <a:avLst/>
    </a:prstGeom>
  </p:spPr>
</p:pic>
```

### 테이블
```xml
<p:graphicFrame>
  <p:nvGraphicFramePr>
    <p:cNvPr id="5" name="Table"/>
    <p:cNvGraphicFramePr>
      <a:graphicFrameLocks noGrp="1"/>
    </p:cNvGraphicFramePr>
    <p:nvPr/>
  </p:nvGraphicFramePr>
  <p:xfrm>
    <a:off x="1000000" y="1000000"/>
    <a:ext cx="6000000" cy="2000000"/>
  </p:xfrm>
  <a:graphic>
    <a:graphicData uri="http://schemas.openxmlformats.org/drawingml/2006/table">
      <a:tbl>
        <a:tblGrid>
          <a:gridCol w="3000000"/>
          <a:gridCol w="3000000"/>
        </a:tblGrid>
        <a:tr h="500000">
          <a:tc>
            <a:txBody>
              <a:bodyPr/>
              <a:lstStyle/>
              <a:p>
                <a:r>
                  <a:t>Cell 1</a:t>
                </a:r>
              </a:p>
            </a:txBody>
          </a:tc>
          <a:tc>
            <a:txBody>
              <a:bodyPr/>
              <a:lstStyle/>
              <a:p>
                <a:r>
                  <a:t>Cell 2</a:t>
                </a:r>
              </a:p>
            </a:txBody>
          </a:tc>
        </a:tr>
      </a:tbl>
    </a:graphicData>
  </a:graphic>
</p:graphicFrame>
```

### 슬라이드 레이아웃

```xml
<!-- 타이틀 슬라이드 레이아웃 -->
<p:sp>
  <p:nvSpPr>
    <p:nvPr>
      <p:ph type="ctrTitle"/>
    </p:nvPr>
  </p:nvSpPr>
  <!-- 타이틀 콘텐츠 -->
</p:sp>

<p:sp>
  <p:nvSpPr>
    <p:nvPr>
      <p:ph type="subTitle" idx="1"/>
    </p:nvPr>
  </p:nvSpPr>
  <!-- 부제목 콘텐츠 -->
</p:sp>

<!-- 콘텐츠 슬라이드 레이아웃 -->
<p:sp>
  <p:nvSpPr>
    <p:nvPr>
      <p:ph type="title"/>
    </p:nvPr>
  </p:nvSpPr>
  <!-- 슬라이드 타이틀 -->
</p:sp>

<p:sp>
  <p:nvSpPr>
    <p:nvPr>
      <p:ph type="body" idx="1"/>
    </p:nvPr>
  </p:nvSpPr>
  <!-- 콘텐츠 본문 -->
</p:sp>
```

## 파일 업데이트

콘텐츠 추가 시 다음 파일들을 업데이트합니다:

**`ppt/_rels/presentation.xml.rels`:**
```xml
<Relationship Id="rId1" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/slide" Target="slides/slide1.xml"/>
<Relationship Id="rId2" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/slideMaster" Target="slideMasters/slideMaster1.xml"/>
```

**`ppt/slides/_rels/slide1.xml.rels`:**
```xml
<Relationship Id="rId1" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/slideLayout" Target="../slideLayouts/slideLayout1.xml"/>
<Relationship Id="rId2" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/image" Target="../media/image1.png"/>
```

**`[Content_Types].xml`:**
```xml
<Default Extension="png" ContentType="image/png"/>
<Default Extension="jpg" ContentType="image/jpeg"/>
<Override PartName="/ppt/slides/slide1.xml" ContentType="application/vnd.openxmlformats-officedocument.presentationml.slide+xml"/>
```

**`ppt/presentation.xml`:**
```xml
<p:sldIdLst>
  <p:sldId id="256" r:id="rId1"/>
  <p:sldId id="257" r:id="rId2"/>
</p:sldIdLst>
```

**`docProps/app.xml`:** 슬라이드 수 및 통계 업데이트
```xml
<Slides>2</Slides>
<Paragraphs>10</Paragraphs>
<Words>50</Words>
```

## 슬라이드 작업

### 새 슬라이드 추가
프레젠테이션 끝에 슬라이드를 추가할 때:

1. **슬라이드 파일 생성** (`ppt/slides/slideN.xml`)
2. **`[Content_Types].xml` 업데이트**: 새 슬라이드에 대한 Override 추가
3. **`ppt/_rels/presentation.xml.rels` 업데이트**: 새 슬라이드에 대한 관계 추가
4. **`ppt/presentation.xml` 업데이트**: `<p:sldIdLst>`에 슬라이드 ID 추가
5. **슬라이드 관계 생성** (`ppt/slides/_rels/slideN.xml.rels`) 필요한 경우
6. **`docProps/app.xml` 업데이트**: 슬라이드 수 증가 및 통계 업데이트 (있는 경우)

### 슬라이드 복제
1. 원본 슬라이드 XML 파일을 새 이름으로 복사
2. 새 슬라이드의 모든 ID를 고유하게 업데이트
3. 위의 "새 슬라이드 추가" 단계 따르기
4. **중요**: `_rels` 파일에서 노트 슬라이드 참조 제거 또는 업데이트
5. 사용하지 않는 미디어 파일 참조 제거

### 슬라이드 재정렬
1. **`ppt/presentation.xml` 업데이트**: `<p:sldIdLst>`의 `<p:sldId>` 요소 재정렬
2. `<p:sldId>` 요소의 순서가 슬라이드 순서를 결정
3. 슬라이드 ID와 관계 ID는 변경하지 않음

예시:
```xml
<!-- 원래 순서 -->
<p:sldIdLst>
  <p:sldId id="256" r:id="rId2"/>
  <p:sldId id="257" r:id="rId3"/>
  <p:sldId id="258" r:id="rId4"/>
</p:sldIdLst>

<!-- 슬라이드 3을 위치 2로 이동 후 -->
<p:sldIdLst>
  <p:sldId id="256" r:id="rId2"/>
  <p:sldId id="258" r:id="rId4"/>
  <p:sldId id="257" r:id="rId3"/>
</p:sldIdLst>
```

### 슬라이드 삭제
1. **`ppt/presentation.xml`에서 제거**: `<p:sldId>` 항목 삭제
2. **`ppt/_rels/presentation.xml.rels`에서 제거**: 관계 삭제
3. **`[Content_Types].xml`에서 제거**: Override 항목 삭제
4. **파일 삭제**: `ppt/slides/slideN.xml` 및 `ppt/slides/_rels/slideN.xml.rels` 제거
5. **`docProps/app.xml` 업데이트**: 슬라이드 수 감소 및 통계 업데이트
6. **사용하지 않는 미디어 정리**: `ppt/media/`에서 고아 이미지 제거

참고: 나머지 슬라이드의 번호를 다시 매기지 말 것 - 원래 ID와 파일명 유지.


## 피해야 할 일반적인 오류

- **인코딩**: ASCII 콘텐츠에서 유니코드 문자 이스케이프: `"` → `&#8220;`
- **이미지**: `ppt/media/`에 추가하고 관계 파일 업데이트
- **목록**: 목록 헤더에서 불릿 제외
- **ID**: UUID에 유효한 16진수 값 사용
- **테마**: 색상에 대해 `theme` 디렉토리의 모든 테마 확인

## 템플릿 기반 프레젠테이션 검증 체크리스트

### 패킹 전 항상 확인:
- **사용하지 않는 리소스 정리**: 참조되지 않는 미디어, 폰트, 노트 디렉토리 제거
- **Content_Types.xml 수정**: 패키지에 있는 모든 슬라이드, 레이아웃, 테마 선언
- **관계 ID 수정**:
   - 내장 폰트를 사용하지 않는 경우 폰트 임베드 참조 제거
- **깨진 참조 제거**: 삭제된 리소스에 대한 참조를 위해 모든 `_rels` 파일 확인

### 일반적인 템플릿 복제 함정:
- 복제 후 여러 슬라이드가 같은 노트 슬라이드 참조
- 더 이상 존재하지 않는 템플릿 슬라이드의 이미지/미디어 참조
- 폰트가 포함되지 않은 경우의 폰트 임베딩 참조
- 레이아웃 12-25에 대한 slideLayout 선언 누락
- docProps 디렉토리가 압축 해제되지 않을 수 있음 - 선택 사항
