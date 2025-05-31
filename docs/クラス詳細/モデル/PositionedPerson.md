# PositionedPerson クラス

## 概要

`PositionedPerson` クラスは、家系図のレイアウト計算後に、個々の人物ノードの位置、サイズ、および世代情報を保持するモデルクラスです。`TreeLayoutEngine` によって生成され、`TreeLayout` クラスの一部として管理されます。このクラスは、家系図の描画処理（`FamilyTreePainter`）で、各人物ノードをキャンバス上の正しい位置に描画するために使用されます。

## クラス定義

```dart
// lib/features/family_tree/layout/tree_layout_engine.dart

class PositionedPerson {
  final Person person;      // 元となる人物データ
  final Offset position;    // ノードの左上の座標 (ローカル座標系)
  final Size size;          // ノードのサイズ (幅と高さ)
  final int generation;     // 人物の世代（階層レベル）
  
  PositionedPerson({
    required this.person,
    required this.position,
    required this.size,
    required this.generation,
  });
  
  // ノードの中心座標を取得するゲッター
  Offset get center {
    return Offset(
      position.dx + size.width / 2,
      position.dy + size.height / 2,
    );
  }
  
  // ノードの矩形領域を取得するゲッター
  Rect get rect {
    return Rect.fromLTWH(position.dx, position.dy, size.width, size.height);
  }
}
```

## プロパティ

| 名前 | 型 | 説明 | デフォルト値 |
|------|------|------|------------|
| `person` | `Person` | この配置情報に対応する元の人物データ | なし（必須） |
| `position` | `Offset` | 家系図キャンバス上でのノードの左上隅の座標 | なし（必須） |
| `size` | `Size` | ノードの幅と高さ | なし（必須） |
| `generation` | `int` | この人物が属する世代（階層レベル）。通常、基準となる人物を0世代とします。 | なし（必須） |

## メソッド (ゲッター)

### center

```dart
Offset get center
```

人物ノードの中心座標を計算して返します。

**戻り値:**
- `Offset` - ノードの中心座標

### rect

```dart
Rect get rect
```

人物ノードが占める矩形領域（`Rect`）を返します。タップ判定などに使用できます。

**戻り値:**
- `Rect` - ノードの矩形領域

## 使用例

```dart
// TreeLayoutからPositionedPersonのリストを取得
final List<PositionedPerson> positionedPersons = treeLayout.positionedPersons;

// 各配置済み人物ノードの情報を利用
for (final positionedPerson in positionedPersons) {
  final Person personData = positionedPerson.person;
  final Offset nodePosition = positionedPerson.position;
  final Size nodeSize = positionedPerson.size;
  final int generationLevel = positionedPerson.generation;
  
  print('人物: ${personData.name}, 位置: $nodePosition, サイズ: $nodeSize, 世代: $generationLevel');
  
  // ノードの中心座標を取得
  final Offset centerPoint = positionedPerson.center;
  
  // ノードの矩形領域を取得
  final Rect nodeRect = positionedPerson.rect;
  
  // 描画処理 (FamilyTreePainter内での例)
  // canvas.drawRect(nodeRect, nodePaint);
  // canvas.drawText(personData.name, centerPoint, textPaint);
}
```

## 注意事項

- `position`プロパティは、家系図全体のローカル座標系におけるノードの左上隅の位置を示します。実際の画面表示時には、`InteractiveViewer`などのウィジェットによるズームやパン（スクロール）の変換を考慮する必要があります。
- `size`プロパティは、レイアウトエンジンによって決定された各人物ノードの標準的なサイズです。必要に応じて、表示する情報量やデザインに基づいて調整されることがあります。
- `generation`プロパティは、家系図の階層構造を理解するのに役立ちます。例えば、同じ世代の人物を同じ高さや列に配置する際に使用されます。
- このクラスは不変（イミュータブル）として扱われるべきですが、`freezed`は使用していません。必要に応じて`freezed`を導入することも検討できます。

## 関連するクラス

- [Person](../モデル/Person.md) - 元となる人物データを表すクラス
- [TreeLayout](./TreeLayout.md) - 家系図全体のレイアウト情報を保持するクラス
- [TreeLayoutEngine](../ウィジェット/TreeLayoutEngine.md) - このクラスのインスタンスを生成するレイアウト計算エンジン
- [FamilyTreePainter](../ウィジェット/FamilyTreePainter.md) - このクラスの情報に基づいて人物ノードを描画する`CustomPainter`
