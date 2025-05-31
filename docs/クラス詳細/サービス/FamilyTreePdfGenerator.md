# FamilyTreePdfGenerator クラス

## 概要

`FamilyTreePdfGenerator` クラスは、家系図データ（`FamilyTreeData`）とPDF設定（`PdfSettings`）に基づいて、実際にPDFドキュメントを生成するコアロジックを担当します。`pdf`パッケージを利用して、人物ノードや関係線を配置し、指定されたフォーマットでPDFを構築します。日本語フォントの組み込みや、ページ分割、レイアウト調整などの複雑な処理もこのクラスで行います。

## クラス定義

```dart
// lib/features/pdf_export/services/family_tree_pdf_generator.dart

class FamilyTreePdfGenerator {
  final Uint8List? _japaneseFontData; // 日本語フォントデータ

  FamilyTreePdfGenerator({Uint8List? japaneseFontData})
      : _japaneseFontData = japaneseFontData;

  // PDF生成メイン関数
  Future<Uint8List> generatePdf({
    required FamilyTreeData familyTreeData,
    required PdfSettings settings,
  }) async {
    final pdf = pw.Document();

    final font = _japaneseFontData != null ? pw.Font.ttf(_japaneseFontData!) : null;
    final pageFormat = settings.orientation == PdfOrientation.portrait
        ? settings.pageFormat
        : settings.pageFormat.landscape;

    // 家系図のレイアウト計算 (TreeLayoutEngineを使用)
    final layoutOrientation = settings.orientation == PdfOrientation.portrait
        ? LayoutOrientation.vertical
        : LayoutOrientation.horizontal;
    final treeLayout = TreeLayoutEngine.calculateLayout(
      persons: familyTreeData.persons,
      relationships: familyTreeData.relationships,
      orientation: layoutOrientation,
    );

    // PDF用にレイアウトを調整 (1ページに収める場合のスケーリングなど)
    final adjustedLayout = _adjustLayoutForPdf(
      treeLayout,
      pageFormat,
      settings.fitToSinglePage,
    );

    // ページ分割の計算
    final pages = _calculatePages(adjustedLayout, pageFormat);

    for (int pageIndex = 0; pageIndex < pages.length; pageIndex++) {
      final pageArea = pages[pageIndex]; // 現在のページが担当する描画領域

      pdf.addPage(
        pw.Page(
          pageFormat: pageFormat,
          build: (pw.Context context) {
            return pw.Stack(
              children: [
                // ヘッダー（タイトル）
                if (settings.title.isNotEmpty)
                  pw.Positioned(
                    top: 20,
                    left: 0,
                    right: 0,
                    child: pw.Center(
                      child: pw.Text(
                        settings.title,
                        style: pw.TextStyle(
                          font: font,
                          fontSize: 18,
                          fontWeight: pw.FontWeight.bold,
                        ),
                      ),
                    ),
                  ),
                // フッター（日付）
                if (settings.showDate)
                  pw.Positioned(
                    bottom: 20,
                    left: 0,
                    right: 0,
                    child: pw.Center(
                      child: pw.Text(
                        '出力日: ${DateFormat('yyyy年MM月dd日').format(DateTime.now())}',
                        style: pw.TextStyle(
                          font: font,
                          fontSize: 9,
                          color: PdfColors.grey700,
                        ),
                      ),
                    ),
                  ),
                // 家系図本体の描画領域
                pw.Positioned.fill(
                  child: pw.Padding(
                    padding: const pw.EdgeInsets.only(top: 50, bottom: 40, left: 20, right: 20),
                    child: _buildTreeContent(
                      adjustedLayout,
                      pageArea, // このページが描画するべき範囲
                      font,
                    ),
                  ),
                ),
                // ページ番号
                if (pages.length > 1)
                  pw.Positioned(
                    bottom: 20,
                    right: 20,
                    child: pw.Text(
                      '${pageIndex + 1}/${pages.length}',
                      style: pw.TextStyle(font: font, fontSize: 9, color: PdfColors.grey700),
                    ),
                  ),
                // 凡例 (最初のページのみ)
                if (settings.showLegend && pageIndex == 0)
                  pw.Positioned(
                    bottom: 40,
                    left: 20,
                    child: _buildLegend(font),
                  ),
              ],
            );
          },
        ),
      );
    }
    return pdf.save();
  }

  // PDF用にレイアウトを調整するヘルパー
  TreeLayout _adjustLayoutForPdf(
    TreeLayout originalLayout,
    PdfPageFormat pageFormat,
    bool fitToSinglePage,
  ) {
    if (originalLayout.isEmpty) return originalLayout;

    final availableWidth = pageFormat.width - 40; // 左右マージン
    final availableHeight = pageFormat.height - 90; // 上下マージン (ヘッダー・フッター分)

    if (!fitToSinglePage) {
      // 1ページに収めない場合は、ノードサイズを一定に保つ
      // ただし、ノードがページ幅を超える場合はスケールダウンを検討
      double scale = 1.0;
      if (TreeLayoutEngine.defaultNodeWidth > availableWidth) {
        scale = availableWidth / TreeLayoutEngine.defaultNodeWidth;
      }
      if (scale < 1.0) {
         return _scaleLayout(originalLayout, scale);
      }
      return originalLayout;
    }

    // 1ページに収める場合のスケーリング
    final originalWidth = originalLayout.requiredSize.width;
    final originalHeight = originalLayout.requiredSize.height;

    if (originalWidth <= 0 || originalHeight <= 0) return originalLayout;

    final scaleX = availableWidth / originalWidth;
    final scaleY = availableHeight / originalHeight;
    final scale = min(scaleX, scaleY);

    return _scaleLayout(originalLayout, scale);
  }
  
  // レイアウトを指定されたスケールで縮小/拡大するヘルパー
  TreeLayout _scaleLayout(TreeLayout layout, double scale) {
    if (scale == 1.0) return layout; // スケール変更なし

    final scaledPersons = layout.positionedPersons.map((p) {
      return PositionedPerson(
        person: p.person,
        position: p.position * scale,
        size: p.size * scale,
        generation: p.generation,
      );
    }).toList();

    final scaledRelationships = layout.positionedRelationships.map((r) {
      return PositionedRelationship(
        relationship: r.relationship,
        pathPoints: r.pathPoints.map((point) => point * scale).toList(),
      );
    }).toList();

    return TreeLayout(
      positionedPersons: scaledPersons,
      positionedRelationships: scaledRelationships,
      requiredSize: layout.requiredSize * scale,
    );
  }

  // ページ分割を計算するヘルパー
  List<Rect> _calculatePages(TreeLayout layout, PdfPageFormat pageFormat) {
    if (layout.isEmpty) return [Rect.zero];

    final contentWidth = layout.requiredSize.width;
    final contentHeight = layout.requiredSize.height;

    final pageContentWidth = pageFormat.width - 40; // 左右マージン
    final pageContentHeight = pageFormat.height - 90; // 上下マージン

    if (contentWidth <= 0 || contentHeight <= 0 || pageContentWidth <=0 || pageContentHeight <= 0) {
        return [Rect.fromLTWH(0, 0, pageContentWidth, pageContentHeight)];
    }

    final numHorizontalPages = (contentWidth / pageContentWidth).ceil();
    final numVerticalPages = (contentHeight / pageContentHeight).ceil();

    final List<Rect> pages = [];
    for (int y = 0; y < numVerticalPages; y++) {
      for (int x = 0; x < numHorizontalPages; x++) {
        final left = x * pageContentWidth;
        final top = y * pageContentHeight;
        pages.add(Rect.fromLTWH(left, top, pageContentWidth, pageContentHeight));
      }
    }
    return pages.isNotEmpty ? pages : [Rect.fromLTWH(0, 0, pageContentWidth, pageContentHeight)];
  }

  // 1ページ分の家系図コンテンツを構築するヘルパー
  pw.Widget _buildTreeContent(
    TreeLayout layout,
    Rect pageArea, // このページが描画するべき家系図全体のローカル座標系での範囲
    pw.Font? font,
  ) {
    return pw.CustomPaint(
      painter: (pw.PdfGraphics canvas, pw.Size size) {
        // 描画の原点を現在のページが表示すべきエリアの左上にオフセット
        canvas.translate(-pageArea.left, -pageArea.top);

        // 関係線の描画
        for (final positionedRel in layout.positionedRelationships) {
          // 関係線が現在のページエリアと交差するか大まかにチェック (最適化)
          // (ここでは簡略化のため省略。厳密には線の各セグメントとpageAreaの交差判定が必要)
          _drawPdfRelationship(canvas, positionedRel, font);
        }
        // 人物ノードの描画
        for (final positionedPerson in layout.positionedPersons) {
          // ノードが現在のページエリア内に存在するかチェック
          if (positionedPerson.rect.overlaps(pageArea)) {
            _drawPdfPersonNode(canvas, positionedPerson, font);
          }
        }
      },
      size: layout.requiredSize, // CustomPaintのサイズは家系図全体のサイズ
    );
  }
  
  // PDFに人物ノードを描画
  void _drawPdfPersonNode(pw.PdfGraphics canvas, PositionedPerson pPerson, pw.Font? font) {
    final rect = PdfRect.fromRect(pPerson.rect);
    final person = pPerson.person;

    canvas.setFillColor(PdfColors.white);
    canvas.drawRRect(rect.x, rect.y, rect.width, rect.height, 2, 2);
    canvas.fillPath();

    final borderColor = person.gender == Gender.male
        ? PdfColors.blue700
        : person.gender == Gender.female
            ? PdfColors.pink700
            : PdfColors.grey700;
    canvas.setStrokeColor(borderColor);
    canvas.setLineWidth(0.8);
    canvas.drawRRect(rect.x, rect.y, rect.width, rect.height, 2, 2);
    canvas.strokePath();

    // 名前テキスト (フォントサイズはノードサイズに応じて調整)
    final nameFontSize = max(6.0, pPerson.size.height * 0.15);
    _drawCenteredText(
      canvas,
      person.name,
      pPerson.center.dx,
      pPerson.position.dy + pPerson.size.height * 0.25, // 上寄せ
      font: font,
      fontSize: nameFontSize,
      fontWeight: pw.FontWeight.bold,
      maxWidth: pPerson.size.width * 0.9,
    );

    // 生没年テキスト
    String dateText = '';
    if (person.birthDate != null) {
      dateText = '${person.birthDate!.year}年';
      if (person.deathDate != null) {
        dateText += ' 〜 ${person.deathDate!.year}年';
      } else {
        dateText += ' 〜';
      }
    } else if (person.deathDate != null) {
      dateText = '? 〜 ${person.deathDate!.year}年';
    }
    if (dateText.isNotEmpty) {
      final dateFontSize = max(5.0, pPerson.size.height * 0.12);
      _drawCenteredText(
        canvas,
        dateText,
        pPerson.center.dx,
        pPerson.position.dy + pPerson.size.height * 0.65, // 下寄せ
        font: font,
        fontSize: dateFontSize,
        color: PdfColors.grey700,
        maxWidth: pPerson.size.width * 0.9,
      );
    }
  }

  // PDFに関係線を描画
  void _drawPdfRelationship(pw.PdfGraphics canvas, PositionedRelationship pRel, pw.Font? font) {
    final points = pRel.pathPoints;
    if (points.length < 2) return;

    canvas.setStrokeColor(
      pRel.type == RelationType.parentChild ? PdfColors.blue700 : PdfColors.pink700,
    );
    canvas.setLineWidth(0.7);

    if (pRel.type == RelationType.parentChild) {
      canvas.moveTo(points.first.dx, points.first.dy);
      for (int i = 1; i < points.length; i++) {
        canvas.lineTo(points[i].dx, points[i].dy);
      }
      canvas.strokePath();
    } else {
      for (int i = 0; i < points.length - 1; i++) {
        _drawDashedLine(canvas, points[i], points[i + 1], dashLength: 2, gapLength: 2);
      }
    }
  }

  // 中央揃えテキスト描画ヘルパー
  void _drawCenteredText(
    pw.PdfGraphics canvas,
    String text,
    double centerX,
    double centerY,
    {
      pw.Font? font,
      double fontSize = 10,
      pw.FontWeight fontWeight = pw.FontWeight.normal,
      PdfColor color = PdfColors.black,
      double maxWidth = double.infinity,
  }) {
    final textSpan = pw.TextSpan(
      text: text,
      style: pw.TextStyle(font: font, fontSize: fontSize, fontWeight: fontWeight, color: color),
    );
    final textPainter = pw.TextPainter(text: textSpan, textAlign: pw.TextAlign.center, textDirection: pw.TextDirection.ltr);
    textPainter.layout(maxWidth: maxWidth);
    textPainter.paint(canvas, pw.Offset(centerX - textPainter.width / 2, centerY - textPainter.height / 2));
  }

  // 点線描画ヘルパー
  void _drawDashedLine(pw.PdfGraphics canvas, Offset p1, Offset p2, {double dashLength = 3, double gapLength = 3}) {
    final dx = p2.dx - p1.dx;
    final dy = p2.dy - p1.dy;
    final distance = sqrt(dx * dx + dy * dy);
    if (distance == 0) return;
    final unitX = dx / distance;
    final unitY = dy / distance;
    var currentDistance = 0.0;
    while (currentDistance < distance) {
      final startX = p1.dx + unitX * currentDistance;
      final startY = p1.dy + unitY * currentDistance;
      currentDistance += dashLength;
      if (currentDistance > distance) currentDistance = distance;
      final endX = p1.dx + unitX * currentDistance;
      final endY = p1.dy + unitY * currentDistance;
      canvas.moveTo(startX, startY);
      canvas.lineTo(endX, endY);
      canvas.strokePath();
      currentDistance += gapLength;
    }
  }

  // 凡例ウィジェット構築ヘルパー
  pw.Widget _buildLegend(pw.Font? font) {
    // (詳細はPDF出力機能 詳細設計書を参照)
    return pw.Container( /* ... 凡例の実装 ... */ );
  }
  
  // DartのMathライブラリのsqrtを使用するためのヘルパー
  double sqrt(double value) => math.sqrt(value);
  double min(double a, double b) => math.min(a,b);
  double max(double a, double b) => math.max(a,b);
}
```

## プロパティ

| 名前 | 型 | 説明 |
|------|------|------|
| `_japaneseFontData` | `Uint8List?` | PDFに埋め込む日本語フォントのバイナリデータ。`null`の場合はデフォルトフォントが使用されます。 |

## メソッド

### generatePdf

```dart
Future<Uint8List> generatePdf({
  required FamilyTreeData familyTreeData,
  required PdfSettings settings,
})
```

指定された家系図データと設定に基づいてPDFドキュメントを生成します。

**パラメータ:**
- `familyTreeData`: `FamilyTreeData` - PDF化する家系図データ
- `settings`: `PdfSettings` - PDFの出力設定（タイトル、用紙サイズ、向きなど）

**戻り値:**
- `Future<Uint8List>` - 生成されたPDFデータのバイナリ

**処理フロー:**
1. `pw.Document`インスタンスを作成します。
2. 日本語フォントが提供されていればロードします。
3. `PdfSettings`に基づいてページフォーマットとレイアウト方向を決定します。
4. `TreeLayoutEngine.calculateLayout`を呼び出して家系図の基本レイアウトを計算します。
5. `_adjustLayoutForPdf`メソッドを呼び出して、PDFのページサイズと`fitToSinglePage`設定に基づいてレイアウトをスケーリング・調整します。
6. `_calculatePages`メソッドを呼び出して、調整後のレイアウトが何ページに分割されるか、各ページが描画すべき領域（`Rect`）を計算します。
7. 計算されたページごとに`pw.Page`を追加します。
   - 各ページには、タイトル、日付、ページ番号、凡例（初回ページのみ）などの共通要素を`pw.Stack`と`pw.Positioned`で配置します。
   - 家系図本体は`_buildTreeContent`メソッドで描画されます。このメソッドは、現在のページが担当する描画領域（`pageArea`）を考慮して、`pw.CustomPaint`内で必要な部分のみを描画します。
8. `pdf.save()`を呼び出してPDFデータを生成し、返します。

### _adjustLayoutForPdf (プライベートヘルパー)

```dart
TreeLayout _adjustLayoutForPdf(
  TreeLayout originalLayout,
  PdfPageFormat pageFormat,
  bool fitToSinglePage,
)
```

`TreeLayoutEngine`によって計算された元のレイアウトを、PDFのページサイズと「1ページに収める」設定に基づいて調整（主にスケーリング）します。

### _scaleLayout (プライベートヘルパー)

```dart
TreeLayout _scaleLayout(TreeLayout layout, double scale)
```

指定されたスケール値に基づいて、レイアウト内のすべての要素（人物ノードの位置とサイズ、関係線の経路点）をスケーリングします。

### _calculatePages (プライベートヘルパー)

```dart
List<Rect> _calculatePages(TreeLayout layout, PdfPageFormat pageFormat)
```

調整（スケーリング）後の家系図レイアウトが、指定されたページフォーマットで何ページに分割されるか、そして各ページが家系図全体のどの領域を描画すべきかを計算します。

### _buildTreeContent (プライベートヘルパー)

```dart
pw.Widget _buildTreeContent(
  TreeLayout layout,
  Rect pageArea,
  pw.Font? font,
)
```

1ページ分の家系図コンテンツを`pw.CustomPaint`を使用して構築します。`pageArea`パラメータは、このページが家系図全体のどの部分を描画すべきかを示します。`canvas.translate`を使用して描画の原点を調整し、`_drawPdfPersonNode`と`_drawPdfRelationship`を呼び出して実際の描画を行います。

### _drawPdfPersonNode (プライベートヘルパー)

```dart
void _drawPdfPersonNode(pw.PdfGraphics canvas, PositionedPerson pPerson, pw.Font? font)
```

PDFキャンバスに個々の人物ノード（矩形とテキスト）を描画します。

### _drawPdfRelationship (プライベートヘルパー)

```dart
void _drawPdfRelationship(pw.PdfGraphics canvas, PositionedRelationship pRel, pw.Font? font)
```

PDFキャンバスに関係線（親子関係は実線、配偶者関係は点線）を描画します。

### _drawCenteredText (プライベートヘルパー)

```dart
void _drawCenteredText(pw.PdfGraphics canvas, String text, ...)
```

指定された位置に中央揃えでテキストを描画するユーティリティメソッドです。

### _drawDashedLine (プライベートヘルパー)

```dart
void _drawDashedLine(pw.PdfGraphics canvas, Offset p1, Offset p2, ...)
```

2点間に点線を描画するユーティリティメソッドです。

### _buildLegend (プライベートヘルパー)

```dart
pw.Widget _buildLegend(pw.Font? font)
```

PDFに表示する凡例ウィジェットを構築します。

## 使用例

```dart
// サービスのインスタンス化 (フォントデータは事前にロードしておく)
final Uint8List? fontData = await loadMyJapaneseFont(); 
final pdfGenerator = FamilyTreePdfGenerator(japaneseFontData: fontData);

// 家系図データと設定の準備
final familyTree = ref.read(selectedFamilyTreeStreamProvider).value;
final settings = ref.read(pdfSettingsProvider);

if (familyTree != null) {
  try {
    final Uint8List pdfBytes = await pdfGenerator.generatePdf(
      familyTreeData: familyTree,
      settings: settings,
    );
    // pdfBytesを保存または表示
    // await Printing.layoutPdf(onLayout: (PdfPageFormat format) async => pdfBytes);
  } catch (e) {
    print('PDF生成エラー: $e');
  }
}
```

## 注意事項

- このクラスは、`pdf`パッケージのウィジェット（`pw.Widget`）を使用してPDFドキュメントを構築します。
- 日本語を表示するためには、日本語対応フォント（例：NotoSansJP-Regular.ttf）をアセットとしてプロジェクトに含め、`_japaneseFontData`としてコンストラクタに渡す必要があります。
- レイアウト計算は`TreeLayoutEngine`に依存します。
- ページ分割ロジック（`_calculatePages`）と、各ページへの描画割り当て（`_buildTreeContent`内の`pageArea`の使用）が、大規模な家系図を複数ページに正しく出力するための鍵となります。
- `_adjustLayoutForPdf`メソッドは、特に「1ページに収める」オプションが有効な場合に、家系図全体を指定されたページサイズに収まるようにスケーリングします。この際、ノードやテキストが非常に小さくなる可能性があるため、ユーザーへのフィードバックが重要です。
- エラーハンドリングは呼び出し元（主に`PdfExportService`）で行うことを想定していますが、このクラス内で発生しうる特定の例外（フォント関連など）は適切に処理または伝播させる必要があります。
- `math.sqrt`, `math.min`, `math.max` を使用するために `dart:math` のインポートが必要です（コードスニペット内では `math.` プレフィックスで示唆）。

## 関連するクラス

- [PdfExportService](./PdfExportService.md) - このジェネレータを使用してPDF出力サービスを提供するクラス
- [PdfSettings](../モデル/PdfSettings.md) - PDFの出力設定を保持するモデルクラス
- [FamilyTreeData](../モデル/FamilyTreeData.md) - PDF化する家系図データを保持するモデルクラス
- [TreeLayout](../モデル/TreeLayout.md) - 家系図のレイアウト情報を保持するクラス
- [TreeLayoutEngine](../ウィジェット/TreeLayoutEngine.md) - 家系図のレイアウト計算を行うクラス
