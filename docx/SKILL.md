---
name: docx
description: "当用户需要创建、读取、编辑或处理 Word 文档（.docx）时使用本技能。触发条件包括：提到「Word 文档」「word document」「.docx」，或需要带目录、标题、页码、信头等专业排版。也用于从 .docx 提取或重组内容、插入/替换图片、在 Word 中查找替换、处理修订或批注、将内容转为精美 Word 文档。若用户要以 Word/.docx 交付「报告」「备忘录」「信件」「模板」等，使用本技能。不要用于 PDF、电子表格、Google Docs，或与文档生成无关的通用编程任务。"
license: 专有许可，完整条款见 LICENSE.txt
---

# DOCX：创建、编辑与分析

（说明：下文代码、命令、XML 标签与 API 标识保持英文，以确保可运行与引用一致。）

## 概述

.docx 文件是包含多个 XML 文件的 ZIP 压缩包。

## 速查

| 任务 | 做法 |
|------|------|
| 读取/分析内容 | `pandoc`，或解包以查看原始 XML |
| 新建文档 | 使用 `docx-js` — 见下文「创建新文档」 |
| 编辑已有文档 | 解包 → 编辑 XML → 重新打包 — 见下文「编辑已有文档」 |

### 将 .doc 转为 .docx

旧版 `.doc` 须先转换再编辑：

```bash
python scripts/office/soffice.py --headless --convert-to docx document.doc
```

### 读取内容

```bash
# Text extraction with tracked changes
pandoc --track-changes=all document.docx -o output.md

# Raw XML access
python scripts/office/unpack.py document.docx unpacked/
```

### 转为图片

```bash
python scripts/office/soffice.py --headless --convert-to pdf document.docx
pdftoppm -jpeg -r 150 document.pdf page
```

### 接受所有修订

生成已接受全部修订的干净文档（需要 LibreOffice）：

```bash
python scripts/accept_changes.py input.docx output.docx
```

---

## 创建新文档

用 JavaScript 生成 .docx，然后校验。安装：`npm install -g docx`

### 初始化
```javascript
const { Document, Packer, Paragraph, TextRun, Table, TableRow, TableCell, ImageRun,
        Header, Footer, AlignmentType, PageOrientation, LevelFormat, ExternalHyperlink,
        InternalHyperlink, Bookmark, FootnoteReferenceRun, PositionalTab,
        PositionalTabAlignment, PositionalTabRelativeTo, PositionalTabLeader,
        TabStopType, TabStopPosition, Column, SectionType,
        TableOfContents, HeadingLevel, BorderStyle, WidthType, ShadingType,
        VerticalAlign, PageNumber, PageBreak } = require('docx');

const doc = new Document({ sections: [{ children: [/* content */] }] });
Packer.toBuffer(doc).then(buffer => fs.writeFileSync("doc.docx", buffer));
```

### 校验
创建文件后必须校验。若失败则解包、修复 XML 再打包。
```bash
python scripts/office/validate.py doc.docx
```

### 页面尺寸

```javascript
// CRITICAL: docx-js defaults to A4, not US Letter
// Always set page size explicitly for consistent results
sections: [{
  properties: {
    page: {
      size: {
        width: 12240,   // 8.5 inches in DXA
        height: 15840   // 11 inches in DXA
      },
      margin: { top: 1440, right: 1440, bottom: 1440, left: 1440 } // 1 inch margins
    }
  },
  children: [/* content */]
}]
```

**常见纸张尺寸（DXA，1440 DXA = 1 英寸）：**

| 纸张 | 宽 | 高 | 内容区宽（左右各 1" 边距） |
|------|-----|-----|---------------------------|
| US Letter | 12,240 | 15,840 | 9,360 |
| A4（默认） | 11,906 | 16,838 | 9,026 |

**横向（Landscape）：** docx-js 内部会交换宽高，因此传入**竖向**的 width/height 数值，由库完成交换：
```javascript
size: {
  width: 12240,   // Pass SHORT edge as width
  height: 15840,  // Pass LONG edge as height
  orientation: PageOrientation.LANDSCAPE  // docx-js swaps them in the XML
},
// Content width = 15840 - left margin - right margin (uses the long edge)
```

### 样式（覆盖内置标题）

默认字体用 Arial（兼容性好）。标题保持黑色以利阅读。

```javascript
const doc = new Document({
  styles: {
    default: { document: { run: { font: "Arial", size: 24 } } }, // 12pt default
    paragraphStyles: [
      // IMPORTANT: Use exact IDs to override built-in styles
      { id: "Heading1", name: "Heading 1", basedOn: "Normal", next: "Normal", quickFormat: true,
        run: { size: 32, bold: true, font: "Arial" },
        paragraph: { spacing: { before: 240, after: 240 }, outlineLevel: 0 } }, // outlineLevel required for TOC
      { id: "Heading2", name: "Heading 2", basedOn: "Normal", next: "Normal", quickFormat: true,
        run: { size: 28, bold: true, font: "Arial" },
        paragraph: { spacing: { before: 180, after: 180 }, outlineLevel: 1 } },
    ]
  },
  sections: [{
    children: [
      new Paragraph({ heading: HeadingLevel.HEADING_1, children: [new TextRun("Title")] }),
    ]
  }]
});
```

### 列表（切勿使用 Unicode 项目符号字符）

```javascript
// ❌ WRONG - never manually insert bullet characters
new Paragraph({ children: [new TextRun("• Item")] })  // BAD
new Paragraph({ children: [new TextRun("\u2022 Item")] })  // BAD

// ✅ CORRECT - use numbering config with LevelFormat.BULLET
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
        children: [new TextRun("Bullet item")] }),
      new Paragraph({ numbering: { reference: "numbers", level: 0 },
        children: [new TextRun("Numbered item")] }),
    ]
  }]
});

// ⚠️ Each reference creates INDEPENDENT numbering
// Same reference = continues (1,2,3 then 4,5,6)
// Different reference = restarts (1,2,3 then 1,2,3)
```

### 表格

**关键：表格须双重宽度** — 既要在 `Table` 上设置 `columnWidths`，也要在每个 `TableCell` 上设置 `width`。缺一则在部分平台上渲染错误。

```javascript
// CRITICAL: Always set table width for consistent rendering
// CRITICAL: Use ShadingType.CLEAR (not SOLID) to prevent black backgrounds
const border = { style: BorderStyle.SINGLE, size: 1, color: "CCCCCC" };
const borders = { top: border, bottom: border, left: border, right: border };

new Table({
  width: { size: 9360, type: WidthType.DXA }, // Always use DXA (percentages break in Google Docs)
  columnWidths: [4680, 4680], // Must sum to table width (DXA: 1440 = 1 inch)
  rows: [
    new TableRow({
      children: [
        new TableCell({
          borders,
          width: { size: 4680, type: WidthType.DXA }, // Also set on each cell
          shading: { fill: "D5E8F0", type: ShadingType.CLEAR }, // CLEAR not SOLID
          margins: { top: 80, bottom: 80, left: 120, right: 120 }, // Cell padding (internal, not added to width)
          children: [new Paragraph({ children: [new TextRun("Cell")] })]
        })
      ]
    })
  ]
})
```

**表格宽度计算：**

始终使用 `WidthType.DXA` — `WidthType.PERCENTAGE` 在 Google Docs 中会出问题。

```javascript
// Table width = sum of columnWidths = content width
// US Letter with 1" margins: 12240 - 2880 = 9360 DXA
width: { size: 9360, type: WidthType.DXA },
columnWidths: [7000, 2360]  // Must sum to table width
```

**宽度规则：**
- **始终 `WidthType.DXA`** — 勿用 `WidthType.PERCENTAGE`（与 Google Docs 不兼容）
- 表格总宽须等于各 `columnWidths` 之和
- 单元格 `width` 须与对应列宽一致
- 单元格 `margins` 为内边距，缩小内容区，不增加单元格总宽
- 通栏表：使用页面内容区宽度（页宽减去左右边距）

### 图片

```javascript
// CRITICAL: type parameter is REQUIRED
new Paragraph({
  children: [new ImageRun({
    type: "png", // Required: png, jpg, jpeg, gif, bmp, svg
    data: fs.readFileSync("image.png"),
    transformation: { width: 200, height: 150 },
    altText: { title: "Title", description: "Desc", name: "Name" } // All three required
  })]
})
```

### 分页

```javascript
// CRITICAL: PageBreak must be inside a Paragraph
new Paragraph({ children: [new PageBreak()] })

// Or use pageBreakBefore
new Paragraph({ pageBreakBefore: true, children: [new TextRun("New page")] })
```

### 超链接

```javascript
// External link
new Paragraph({
  children: [new ExternalHyperlink({
    children: [new TextRun({ text: "Click here", style: "Hyperlink" })],
    link: "https://example.com",
  })]
})

// Internal link (bookmark + reference)
// 1. Create bookmark at destination
new Paragraph({ heading: HeadingLevel.HEADING_1, children: [
  new Bookmark({ id: "chapter1", children: [new TextRun("Chapter 1")] }),
]})
// 2. Link to it
new Paragraph({ children: [new InternalHyperlink({
  children: [new TextRun({ text: "See Chapter 1", style: "Hyperlink" })],
  anchor: "chapter1",
})]})
```

### 脚注

```javascript
const doc = new Document({
  footnotes: {
    1: { children: [new Paragraph("Source: Annual Report 2024")] },
    2: { children: [new Paragraph("See appendix for methodology")] },
  },
  sections: [{
    children: [new Paragraph({
      children: [
        new TextRun("Revenue grew 15%"),
        new FootnoteReferenceRun(1),
        new TextRun(" using adjusted metrics"),
        new FootnoteReferenceRun(2),
      ],
    })]
  }]
});
```

### 制表位

```javascript
// Right-align text on same line (e.g., date opposite a title)
new Paragraph({
  children: [
    new TextRun("Company Name"),
    new TextRun("\tJanuary 2025"),
  ],
  tabStops: [{ type: TabStopType.RIGHT, position: TabStopPosition.MAX }],
})

// Dot leader (e.g., TOC-style)
new Paragraph({
  children: [
    new TextRun("Introduction"),
    new TextRun({ children: [
      new PositionalTab({
        alignment: PositionalTabAlignment.RIGHT,
        relativeTo: PositionalTabRelativeTo.MARGIN,
        leader: PositionalTabLeader.DOT,
      }),
      "3",
    ]}),
  ],
})
```

### 多栏布局

```javascript
// Equal-width columns
sections: [{
  properties: {
    column: {
      count: 2,          // number of columns
      space: 720,        // gap between columns in DXA (720 = 0.5 inch)
      equalWidth: true,
      separate: true,    // vertical line between columns
    },
  },
  children: [/* content flows naturally across columns */]
}]

// Custom-width columns (equalWidth must be false)
sections: [{
  properties: {
    column: {
      equalWidth: false,
      children: [
        new Column({ width: 5400, space: 720 }),
        new Column({ width: 3240 }),
      ],
    },
  },
  children: [/* content */]
}]
```

使用新节并设 `type: SectionType.NEXT_COLUMN` 可强制分栏断页。

### 目录

```javascript
// CRITICAL: Headings must use HeadingLevel ONLY - no custom styles
new TableOfContents("Table of Contents", { hyperlink: true, headingStyleRange: "1-3" })
```

### 页眉/页脚

```javascript
sections: [{
  properties: {
    page: { margin: { top: 1440, right: 1440, bottom: 1440, left: 1440 } } // 1440 = 1 inch
  },
  headers: {
    default: new Header({ children: [new Paragraph({ children: [new TextRun("Header")] })] })
  },
  footers: {
    default: new Footer({ children: [new Paragraph({
      children: [new TextRun("Page "), new TextRun({ children: [PageNumber.CURRENT] })]
    })] })
  },
  children: [/* content */]
}]
```

### docx-js 关键规则

- **显式设置页面尺寸** — docx-js 默认为 A4；美国文档请用 US Letter（12240 × 15840 DXA）
- **横向：传入竖向尺寸** — docx-js 内部交换宽高；短边作 `width`，长边作 `height`，并设 `orientation: PageOrientation.LANDSCAPE`
- **禁止使用 `\n`** — 用多个 `Paragraph` 分段
- **禁止 Unicode 项目符号** — 用 `LevelFormat.BULLET` 与 numbering 配置
- **分页须在 Paragraph 内** — 单独的 PageBreak 会产生无效 XML
- **ImageRun 必须带 `type`** — 始终指定 png/jpg 等
- **表格宽度一律 DXA** — 勿用 `WidthType.PERCENTAGE`（Google Docs 会坏）
- **表格双重宽度** — `columnWidths` 与每个 cell 的 `width` 须一致匹配
- **表格总宽 = columnWidths 之和** — DXA 下须精确相加
- **单元格边距** — 使用 `margins: { top: 80, bottom: 80, left: 120, right: 120 }` 等保证可读内边距
- **着色用 `ShadingType.CLEAR`** — 表格底纹勿用 SOLID
- **勿用表格当分隔线** — 单元格有最小高度，在页眉页脚也会像空框；改用 `Paragraph` 的 `border: { bottom: {...} }`。双栏页脚用制表位（见「制表位」），不要用表格
- **目录仅能用 HeadingLevel** — 标题段落勿用自定义样式冒充
- **覆盖内置样式** — 使用精确 ID：`"Heading1"`、`"Heading2"` 等
- **须含 `outlineLevel`** — 目录需要（H1 为 0，H2 为 1，以此类推）

---

## 编辑已有文档

**请严格按顺序执行以下 3 步。**

### 第 1 步：解包
```bash
python scripts/office/unpack.py document.docx unpacked/
```
提取 XML、美化、合并相邻 run，并将弯引号转为 XML 实体（如 `&#x201C;`）以便编辑。`--merge-runs false` 可跳过 run 合并。

### 第 2 步：编辑 XML

编辑 `unpacked/word/` 下文件。模式见下文「XML 参考」。

修订与批注的作者名默认使用 **"Claude"**，除非用户明确要求其他名称。

**请直接用编辑工具做字符串替换，不要写 Python 脚本。** 脚本徒增复杂度。编辑工具能精确展示替换内容。

**关键：新增内容使用弯引号实体。** 添加含撇号或引号的文字时，用 XML 实体表示弯引号：
```xml
<!-- Use these entities for professional typography -->
<w:t>Here&#x2019;s a quote: &#x201C;Hello&#x201D;</w:t>
```
| 实体 | 字符 |
|--------|-----------|
| `&#x2018;` | ‘ (左单引号) |
| `&#x2019;` | ’ (右单引号/撇号) |
| `&#x201C;` | “ (左双引号) |
| `&#x201D;` | ” (右双引号) |

**添加批注：** 用 `comment.py` 处理跨多个 XML 的样板（文本须预先转义为合法 XML）：
```bash
python scripts/comment.py unpacked/ 0 "Comment text with &amp; and &#x2019;"
python scripts/comment.py unpacked/ 1 "Reply text" --parent 0  # reply to comment 0
python scripts/comment.py unpacked/ 0 "Text" --author "Custom Author"  # custom author name
```
然后在 document.xml 中加入标记（见「XML 参考」中的批注部分）。

### 第 3 步：打包
```bash
python scripts/office/pack.py unpacked/ output.docx --original document.docx
```
校验并可自动修复、压缩 XML 并生成 DOCX。`--validate false` 可跳过校验。

**自动修复会处理：**
- `durableId` >= 0x7FFFFFFF（会重新生成合法 ID）
- 含空白的前后 `<w:t>` 缺少 `xml:space="preserve"`

**自动修复无法处理：**
- 畸形 XML、非法嵌套、关系缺失、违反 schema

### 常见陷阱

- **整体替换 `<w:r>`**：添加修订时，应将整个 `<w:r>...</w:r>` 替换为并列的 `<w:del>` / `<w:ins>`，勿在 run 内嵌套修订标签。
- **保留 `<w:rPr>` 格式**：将原 run 的 `<w:rPr>` 复制到修订后的 run 中，以保留粗体、字号等。

---

## XML 参考

### Schema 合规

- **`<w:pPr>` 内元素顺序**：`<w:pStyle>`、`<w:numPr>`、`<w:spacing>`、`<w:ind>`、`<w:jc>`，**最后** `<w:rPr>`
- **空白**：前后含空格的 `<w:t>` 须加 `xml:space="preserve"`
- **RSID**：须为 8 位十六进制（如 `00AB1234`）

### 修订（插入/删除）

**插入：**
```xml
<w:ins w:id="1" w:author="Claude" w:date="2025-01-01T00:00:00Z">
  <w:r><w:t>inserted text</w:t></w:r>
</w:ins>
```

**删除：**
```xml
<w:del w:id="2" w:author="Claude" w:date="2025-01-01T00:00:00Z">
  <w:r><w:delText>deleted text</w:delText></w:r>
</w:del>
```

**在 `<w:del>` 内**：使用 `<w:delText>` 代替 `<w:t>`，使用 `<w:delInstrText>` 代替 `<w:instrText>`。

**最小修订范围** — 仅标记实际变化：
```xml
<!-- Change "30 days" to "60 days" -->
<w:r><w:t>The term is </w:t></w:r>
<w:del w:id="1" w:author="Claude" w:date="...">
  <w:r><w:delText>30</w:delText></w:r>
</w:del>
<w:ins w:id="2" w:author="Claude" w:date="...">
  <w:r><w:t>60</w:t></w:r>
</w:ins>
<w:r><w:t> days.</w:t></w:r>
```

**删除整段/整个列表项** — 删除段落全部内容时，还须标记段落符号为删除，以便与下一段合并。在 `<w:pPr><w:rPr>` 内加入 `<w:del/>`：
```xml
<w:p>
  <w:pPr>
    <w:numPr>...</w:numPr>  <!-- list numbering if present -->
    <w:rPr>
      <w:del w:id="1" w:author="Claude" w:date="2025-01-01T00:00:00Z"/>
    </w:rPr>
  </w:pPr>
  <w:del w:id="2" w:author="Claude" w:date="2025-01-01T00:00:00Z">
    <w:r><w:delText>Entire paragraph content being deleted...</w:delText></w:r>
  </w:del>
</w:p>
```
若 `<w:pPr><w:rPr>` 中缺少 `<w:del/>`，接受修订后会留下空段落/空列表项。

**拒绝他人插入的内容** — 将删除嵌套在其插入之内：
```xml
<w:ins w:author="Jane" w:id="5">
  <w:del w:author="Claude" w:id="10">
    <w:r><w:delText>their inserted text</w:delText></w:r>
  </w:del>
</w:ins>
```

**恢复他人删除的内容** — 在其删除之后添加插入（勿改对方删除块）：
```xml
<w:del w:author="Jane" w:id="5">
  <w:r><w:delText>deleted text</w:delText></w:r>
</w:del>
<w:ins w:author="Claude" w:id="10">
  <w:r><w:t>deleted text</w:t></w:r>
</w:ins>
```

### 批注

运行 `comment.py`（见第 2 步）后，在 document.xml 中加入标记。回复批注时使用 `--parent`，并将标记嵌套在父批注内。

**关键：`<w:commentRangeStart>` 与 `<w:commentRangeEnd>` 与 `<w:r>` 为兄弟节点，绝不能放在 `<w:r>` 内部。**

```xml
<!-- Comment markers are direct children of w:p, never inside w:r -->
<w:commentRangeStart w:id="0"/>
<w:del w:id="1" w:author="Claude" w:date="2025-01-01T00:00:00Z">
  <w:r><w:delText>deleted</w:delText></w:r>
</w:del>
<w:r><w:t> more text</w:t></w:r>
<w:commentRangeEnd w:id="0"/>
<w:r><w:rPr><w:rStyle w:val="CommentReference"/></w:rPr><w:commentReference w:id="0"/></w:r>

<!-- Comment 0 with reply 1 nested inside -->
<w:commentRangeStart w:id="0"/>
  <w:commentRangeStart w:id="1"/>
  <w:r><w:t>text</w:t></w:r>
  <w:commentRangeEnd w:id="1"/>
<w:commentRangeEnd w:id="0"/>
<w:r><w:rPr><w:rStyle w:val="CommentReference"/></w:rPr><w:commentReference w:id="0"/></w:r>
<w:r><w:rPr><w:rStyle w:val="CommentReference"/></w:rPr><w:commentReference w:id="1"/></w:r>
```

### 图片（XML 嵌入）

1. 将图片文件放入 `word/media/`
2. 在 `word/_rels/document.xml.rels` 中添加关系：
```xml
<Relationship Id="rId5" Type=".../image" Target="media/image1.png"/>
```
3. 在 `[Content_Types].xml` 中添加内容类型：
```xml
<Default Extension="png" ContentType="image/png"/>
```
4. 在 document.xml 中引用：
```xml
<w:drawing>
  <wp:inline>
    <wp:extent cx="914400" cy="914400"/>  <!-- EMUs: 914400 = 1 inch -->
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

## 依赖

- **pandoc**：文本提取
- **docx**：`npm install -g docx`（新建文档）
- **LibreOffice**：PDF 转换（沙箱环境通过 `scripts/office/soffice.py` 自动配置）
- **Poppler**：`pdftoppm` 用于图片
