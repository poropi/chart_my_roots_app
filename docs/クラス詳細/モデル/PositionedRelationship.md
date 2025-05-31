# PositionedRelationship クラス

## 概要

`PositionedRelationship` クラスは、家系図のレイアウト計算後に、人物間の関係線（親子関係や配偶者関係）の経路情報を保持するモデルクラスです。`TreeLayoutEngine` によって生成され、`TreeLayout` クラスの一部として管理されます。このクラスは、家系図の描画処理（`FamilyTreePainter`）で、関係線をキャンバス上に正しく描画するために使用されます。

## クラス定義

```dart
// lib/features/family_tree/layout/tree_layout_engine.dart

class PositionedRelationship {
  final Relationship relationship; // 元となる関係性データ
  final List<Offset> pathPoints;  // 関係線を描画するための経路点のリスト
  
  PositionedRelationship({
    required this.relationship,
    required this.pathPoints,
  });
  
  // 関係線の始点を取得するゲッター
  Offset? get startPoint => pathPoints.isNotEmpty ? pathPoints.first : null;
  
  // 関係線の終点を取得するゲッター
  Offset? get endPoint => pathPoints.isNotEmpty ? pathPoints.last : null;
  
  // 関係線の種類を取得するゲッター
  RelationType get type => relationship.type;
}
```

## プロパティ

| 名前 | 型 | 説明 | デフォルト値 |
|------|------|------|------------|
| `relationship` | `Relationship` | この配置情報に対応する元の関係性データ | なし（必須） |
| `pathPoints` | `List<Offset>` | 関係線を描画するための経路点のリスト。通常、始点、中間点（必要な場合）、終点が含まれます。 | なし（必須） |

## メソッド (ゲッター)

### startPoint

```dart
Offset? get startPoint
```

関係線の始点を返します。`pathPoints`が空の場合は`null`を返します。

**戻り値:**
- `Offset?` - 関係線の始点、または`null`

### endPoint

```dart
Offset? get endPoint
```

関係線の終点を返します。`pathPoints`が空の場合は`null`を返します。

**戻り値:**
- `Offset?` - 関係線の終点、または`null`

### type

```dart
RelationType get type
```

この関係線の種類（親子関係または配偶者関係）を返します。

**戻り値:**
- `RelationType` - 関係線の種類

## 使用例

```dart
// TreeLayoutからPositionedRelationshipのリストを取得
final List<PositionedRelationship> positionedRelationships = treeLayout.positionedRelationships;

// 各配置済み関係線の情報を利用
for (final positionedRel in positionedRelationships) {
  final Relationship originalRelData = positionedRel.relationship;
  final List<Offset> linePath = positionedRel.pathPoints;
  final RelationType relType = positionedRel.type;
  
  print('関係タイプ: $relType, 経路点数: ${linePath.length}');
  
  if (linePath.isNotEmpty) {
    final Offset start = positionedRel.startPoint!;
    final Offset end = positionedRel.endPoint!;
    print('  始点: $start, 終点: $end');
    
    // 描画処理 (FamilyTreePainter内での例)
    // final path = Path();
    // path.moveTo(start.dx, start.dy);
    // for (int i = 1; i < linePath.length; i++) {
    //   path.lineTo(linePath[i].dx, linePath[i].dy);
    // }
    // canvas.drawPath(path, linePaint);
  }
}
```

## 注意事項

- `pathPoints`プロパティは、家系図全体のローカル座標系における関係線の経路を示します。直線だけでなく、角を曲がる線などを表現するために複数の点を持つことがあります。
- 実際の画面表示時には、`InteractiveViewer`などのウィジェットによるズームやパン（スクロール）の変換を考慮して描画する必要があります。
- 関係線のスタイル（色、太さ、実線/点線など）は、`relationship.type`に基づいて`FamilyTreePainter`内で決定されます。
- このクラスは不変（イミュータブル）として扱われるべきですが、`freezed`は使用していません。必要に応じて`freezed`を導入することも検討できます。

## 関連するクラス

- [Relationship](../モデル/Relationship.md) - 元となる関係性データを表すクラス
- [TreeLayout](./TreeLayout.md) - 家系図全体のレイアウト情報を保持するクラス
- [TreeLayoutEngine](../ウィジェット/TreeLayoutEngine.md) - このクラスのインスタンスを生成するレイアウト計算エンジン
- [FamilyTreePainter](../ウィジェット/FamilyTreePainter.md) - このクラスの情報に基づいて関係線を描画する`CustomPainter`
