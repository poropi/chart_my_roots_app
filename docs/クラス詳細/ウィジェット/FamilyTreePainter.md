# FamilyTreePainter クラス

## 概要

`FamilyTreePainter` クラスは、`CustomPainter` を継承し、家系図の実際の描画処理を担当します。`TreeLayout` オブジェクトから得られる配置情報（人物ノードの位置や関係線の経路）に基づいて、キャンバス上に家系図をレンダリングします。人物ノードのスタイル（色、枠線）、関係線のスタイル（実線、点線）、選択状態のハイライトなどを描画ロジックとして実装します。

## クラス定義

```dart
// lib/features/family_tree/presentation/painters/family_tree_painter.dart

class FamilyTreePainter extends CustomPainter {
  final TreeLayout layout;
  final String? selectedPersonId;
  final double scale; // InteractiveViewerからの現在のスケール
  final Paint _linePaint;
  final Paint _nodeBackgroundPaint;
  final Paint _nodeBorderPaint;
  final Paint _selectedNodeBorderPaint;
  final TextPainter _textPainter;

  FamilyTreePainter({
    required this.layout,
    this.selectedPersonId,
    this.scale = 1.0,
  }) : _linePaint = Paint()..style = PaintingStyle.stroke,
       _nodeBackgroundPaint = Paint()..style = PaintingStyle.fill,
       _nodeBorderPaint = Paint()..style = PaintingStyle.stroke,
       _selectedNodeBorderPaint = Paint()
         ..style = PaintingStyle.stroke
         ..color = Colors.amber.shade700, // 選択時のハイライト色
       _textPainter = TextPainter(textAlign: TextAlign.center, textDirection: TextDirection.ltr);

  @override
  void paint(Canvas canvas, Size size) {
    // 注意: このPainterはInteractiveViewerの子として使われるため、
    // InteractiveViewerがスケーリングと平行移動を処理します。
    // ここでの描画座標は、layout内のローカル座標をそのまま使用します。
    // スケールに応じた見た目の調整（線幅、フォントサイズなど）はここで行います。

    // 関係線の描画
    _drawRelationships(canvas);

    // 人物ノードの描画
    _drawPersonNodes(canvas);
  }

  void _drawRelationships(Canvas canvas) {
    _linePaint.strokeWidth = max(0.5, 1.5 * scale); // スケールに応じた線幅

    for (final positionedRel in layout.positionedRelationships) {
      final points = positionedRel.pathPoints;
      if (points.length < 2) continue;

      _linePaint.color = positionedRel.type == RelationType.parentChild
          ? Colors.blueGrey.shade400
          : Colors.pink.shade300;

      final path = Path()..moveTo(points.first.dx, points.first.dy);
      for (int i = 1; i < points.length; i++) {
        path.lineTo(points[i].dx, points[i].dy);
      }

      if (positionedRel.type == RelationType.spouse) {
        // 配偶者関係は点線
        final dashWidth = 4.0 * scale;
        final dashSpace = 2.0 * scale;
        canvas.drawPath(
          dashPath(path, dashArray: CircularIntervalList<double>([dashWidth, dashSpace])),
          _linePaint,
        );
      } else {
        // 親子関係は実線
        canvas.drawPath(path, _linePaint);
      }
    }
  }

  void _drawPersonNodes(Canvas canvas) {
    _nodeBorderPaint.strokeWidth = max(0.8, PersonNodeWidget.borderWidth * scale);
    _selectedNodeBorderPaint.strokeWidth = max(1.0, PersonNodeWidget.selectedBorderWidth * scale);

    for (final positionedPerson in layout.positionedPersons) {
      final person = positionedPerson.person;
      final rect = positionedPerson.rect;
      final bool isSelected = person.id == selectedPersonId;

      // 背景色
      _nodeBackgroundPaint.color = Colors.white;
      final rRect = RRect.fromRectAndRadius(rect, Radius.circular(max(3.0, PersonNodeWidget.cornerRadius * scale)));
      canvas.drawRRect(rRect, _nodeBackgroundPaint);

      // 枠線
      final currentBorderPaint = isSelected ? _selectedNodeBorderPaint : _nodeBorderPaint;
      currentBorderPaint.color = _getNodeBorderColor(person.gender, isSelected);
      canvas.drawRRect(rRect, currentBorderPaint);

      // テキスト描画
      _drawPersonText(canvas, positionedPerson, person);
    }
  }

  void _drawPersonText(Canvas canvas, PositionedPerson positionedPerson, Person person) {
    final rect = positionedPerson.rect;
    final baseFontSize = PersonNodeWidget.baseHeight * 0.2; // 基本フォントサイズ
    final currentFontSize = max(7.0, baseFontSize * scale); // スケール適用、最小フォントサイズ保証

    // 氏名
    _textPainter.text = TextSpan(
      text: person.name,
      style: TextStyle(
        fontSize: currentFontSize,
        fontWeight: FontWeight.bold,
        color: Colors.black87,
      ),
    );
    _textPainter.layout(minWidth: 0, maxWidth: rect.width - (PersonNodeWidget.padding * 2 * scale));
    _textPainter.paint(
      canvas,
      Offset(
        rect.left + (rect.width - _textPainter.width) / 2,
        rect.top + (rect.height / 2) - _textPainter.height, // 上半分の中央やや上
      ),
    );

    // 生没年
    final lifeSpanText = _getLifeSpanText(person);
    if (lifeSpanText.isNotEmpty) {
      final dateFontSize = max(6.0, baseFontSize * 0.8 * scale);
      _textPainter.text = TextSpan(
        text: lifeSpanText,
        style: TextStyle(
          fontSize: dateFontSize,
          color: Colors.grey.shade700,
        ),
      );
      _textPainter.layout(minWidth: 0, maxWidth: rect.width - (PersonNodeWidget.padding * 2 * scale));
      _textPainter.paint(
        canvas,
        Offset(
          rect.left + (rect.width - _textPainter.width) / 2,
          rect.top + (rect.height / 2) + (_textPainter.height * 0.2), // 下半分の中央やや下
        ),
      );
    }
  }

  Color _getNodeBorderColor(Gender? gender, bool selected) {
    if (selected) {
      return Colors.amber.shade700;
    }
    switch (gender) {
      case Gender.male:
        return Colors.blue.shade600;
      case Gender.female:
        return Colors.pink.shade500;
      default:
        return Colors.grey.shade500;
    }
  }

  String _getLifeSpanText(Person person) {
    String text = '';
    if (person.birthDate != null) {
      text = '${person.birthDate!.year}年';
      if (person.deathDate != null) {
        text += ' 〜 ${person.deathDate!.year}年';
      } else {
        text += ' 〜'; // 存命中の表現
      }
    } else if (person.deathDate != null) {
      text = '? 〜 ${person.deathDate!.year}年';
    }
    return text;
  }

  @override
  bool shouldRepaint(covariant FamilyTreePainter oldDelegate) {
    return oldDelegate.layout != layout ||
           oldDelegate.selectedPersonId != selectedPersonId ||
           oldDelegate.scale != scale;
  }
  
  // スケールに応じた最小値を保証するヘルパー
  double max(double a, double b) => math.max(a,b);
}
```

## プロパティ

| 名前 | 型 | 説明 | デフォルト値 |
|------|------|------|------------|
| `layout` | `TreeLayout` | 描画する家系図のレイアウト情報 | なし（必須） |
| `selectedPersonId` | `String?` | 現在選択されている人物のID。選択ハイライトに使用。 | `null` |
| `scale` | `double` | 現在の家系図全体の表示スケール。線幅やフォントサイズの調整に使用。 | `1.0` |
| `_linePaint` | `Paint` | 関係線描画用のPaintオブジェクト（内部使用） | - |
| `_nodeBackgroundPaint` | `Paint` | 人物ノード背景描画用のPaintオブジェクト（内部使用） | - |
| `_nodeBorderPaint` | `Paint` | 人物ノード枠線描画用のPaintオブジェクト（内部使用） | - |
| `_selectedNodeBorderPaint` | `Paint` | 選択された人物ノード枠線描画用のPaintオブジェクト（内部使用） | - |
| `_textPainter` | `TextPainter` | テキスト描画用のTextPainterオブジェクト（内部使用） | - |

## メソッド

### paint

```dart
void paint(Canvas canvas, Size size)
```

`CustomPainter`のメインメソッド。キャンバスに家系図を描画します。内部で`_drawRelationships`と`_drawPersonNodes`を呼び出します。

**パラメータ:**
- `canvas`: `Canvas` - 描画対象のキャンバス
- `size`: `Size` - `CustomPaint`ウィジェットのサイズ（このPainterでは主に`layout.requiredSize`が基準となる）

### _drawRelationships (プライベートヘルパー)

```dart
void _drawRelationships(Canvas canvas)
```

`layout.positionedRelationships`に基づいて、人物間の関係線（親子関係、配偶者関係）をキャンバスに描画します。線のスタイル（色、太さ、点線/実線）は関係タイプと現在のスケールに応じて調整されます。

### _drawPersonNodes (プライベートヘルパー)

```dart
void _drawPersonNodes(Canvas canvas)
```

`layout.positionedPersons`に基づいて、個々の人物ノードをキャンバスに描画します。ノードの背景、枠線（選択状態や性別で変化）、内部のテキスト（氏名、生没年）を描画します。フォントサイズや枠線の太さも現在のスケールに応じて調整されます。

### _drawPersonText (プライベートヘルパー)

```dart
void _drawPersonText(Canvas canvas, PositionedPerson positionedPerson, Person person)
```

人物ノード内に氏名と生没年テキストを描画します。テキストは中央揃えで、ノードサイズとスケールに応じてフォントサイズが調整されます。

### _getNodeBorderColor (プライベートヘルパー)

```dart
Color _getNodeBorderColor(Gender? gender, bool selected)
```

人物の性別と選択状態に基づいて、ノードの枠線の色を決定します。

### _getLifeSpanText (プライベートヘルパー)

```dart
String _getLifeSpanText(Person person)
```

人物の生年月日と没年月日から表示用の文字列を生成します。

### shouldRepaint

```dart
bool shouldRepaint(covariant FamilyTreePainter oldDelegate)
```

`CustomPainter`のメソッド。ウィジェットが再描画されるべきかどうかを判断します。`layout`、`selectedPersonId`、または`scale`が変更された場合に再描画します。

## 使用例

`FamilyTreePainter`は、`InteractiveFamilyTree`ウィジェット内の`CustomPaint`ウィジェットの`painter`プロパティに指定されて使用されます。

```dart
// InteractiveFamilyTreeウィジェット内での使用イメージ
InteractiveViewer(
  // ... InteractiveViewerの設定 ...
  child: SizedBox(
    width: layout.requiredSize.width,
    height: layout.requiredSize.height,
    child: CustomPaint(
      painter: FamilyTreePainter(
        layout: layout, // TreeLayoutオブジェクト
        selectedPersonId: selectedPersonId, // 選択中の人物ID
        scale: currentInteractiveViewerScale, // InteractiveViewerから取得したスケール
      ),
      size: layout.requiredSize, // 描画全体のサイズを指定
    ),
  ),
)
```

## 注意事項

- このPainterは、`InteractiveViewer`のような親ウィジェットによってスケーリングと平行移動が管理されることを前提としています。そのため、`paint`メソッド内での座標は`layout`オブジェクトが提供するローカル座標を直接使用します。
- `scale`プロパティは非常に重要で、線幅、フォントサイズ、パディングなど、描画要素の見た目をズームレベルに応じて動的に調整するために使用されます。これにより、ズームアウトしても要素が細かくなりすぎず、ズームインすれば詳細がクリアに見えるようになります。
- `TextPainter`はテキストの描画と測定に効率的ですが、各描画サイクルで再利用するためにインスタンス変数として保持しています。
- 点線の描画には`path_drawing`パッケージの`dashPath`関数を利用しています（`import 'package:path_drawing/path_drawing.dart';` が必要）。
- `math.max` を使用するために `dart:math` のインポートが必要です。
- パフォーマンス向上のため、`shouldRepaint`メソッドで不要な再描画を避けるようにしています。

## 関連するクラス

- [InteractiveFamilyTree](./InteractiveFamilyTree.md) - このPainterを使用する親ウィジェット
- [TreeLayout](../モデル/TreeLayout.md) - 描画する家系図のレイアウト情報を保持するクラス
- [PositionedPerson](../モデル/PositionedPerson.md) - 配置された人物ノードの情報を保持するクラス
- [PositionedRelationship](../モデル/PositionedRelationship.md) - 配置された関係線の情報を保持するクラス
- [PersonNodeWidget](./PersonNodeWidget.md) - このPainterが描画する内容のウィジェット版の参考。ただし、Painterはより低レベルな描画を行います。
