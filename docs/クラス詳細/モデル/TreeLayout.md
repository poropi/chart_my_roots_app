# TreeLayout クラス

## 概要

`TreeLayout` クラスは、家系図のレイアウト計算結果を保持するモデルクラスです。`TreeLayoutEngine` によって生成され、配置された人物ノードの情報（`PositionedPerson`）、関係線の情報（`PositionedRelationship`）、およびレイアウト全体のサイズを管理します。このクラスは、家系図の描画処理（`FamilyTreePainter`）に渡され、実際の描画に使用されます。

## クラス定義

```dart
// lib/features/family_tree/layout/tree_layout_engine.dart

class TreeLayout {
  final List<PositionedPerson> positionedPersons;
  final List<PositionedRelationship> positionedRelationships;
  final Size requiredSize; // レイアウト全体のサイズ
  
  TreeLayout({
    required this.positionedPersons,
    required this.positionedRelationships,
    required this.requiredSize,
  });
  
  // 空のレイアウトを生成するファクトリメソッド
  factory TreeLayout.empty() {
    return TreeLayout(
      positionedPersons: [],
      positionedRelationships: [],
      requiredSize: Size.zero,
    );
  }
  
  // レイアウトが空かどうかを判定するゲッター
  bool get isEmpty => positionedPersons.isEmpty && requiredSize == Size.zero;
}
```

## プロパティ

| 名前 | 型 | 説明 | デフォルト値 |
|------|------|------|------------|
| `positionedPersons` | `List<PositionedPerson>` | 配置された人物ノードのリスト | なし（必須） |
| `positionedRelationships` | `List<PositionedRelationship>` | 配置された関係線のリスト | なし（必須） |
| `requiredSize` | `Size` | 家系図全体の描画に必要なサイズ | なし（必須） |

## メソッド

### TreeLayout.empty

```dart
factory TreeLayout.empty()
```

空の`TreeLayout`インスタンスを生成します。人物や関係性が存在しない場合や、初期状態として使用されます。

**戻り値:**
- `TreeLayout` - 空のレイアウト情報を持つインスタンス

### isEmpty (ゲッター)

```dart
bool get isEmpty
```

このレイアウトが空（人物ノードが存在せず、サイズもゼロ）であるかどうかを返します。

**戻り値:**
- `bool` - レイアウトが空の場合は`true`、そうでない場合は`false`

## 使用例

```dart
// TreeLayoutEngineからレイアウト計算結果を取得
final TreeLayout layout = TreeLayoutEngine.calculateLayout(
  persons: personList,
  relationships: relationshipList,
  orientation: LayoutOrientation.vertical,
);

// レイアウト情報を使用して描画
final painter = FamilyTreePainter(layout: layout);

// レイアウトが空かどうかの確認
if (layout.isEmpty) {
  print('家系図データがありません。');
}

// 配置された人物ノードの数を取得
final numberOfPersons = layout.positionedPersons.length;

// 描画に必要な全体のサイズを取得
final totalWidth = layout.requiredSize.width;
final totalHeight = layout.requiredSize.height;
```

## 注意事項

- このクラスは、`TreeLayoutEngine`の計算結果を格納するためのデータコンテナです。
- `positionedPersons`と`positionedRelationships`には、各要素の座標やサイズ、経路情報が含まれます。
- `requiredSize`は、家系図全体を描画するために必要なキャンバスのサイズを示します。`InteractiveViewer`などのウィジェットでコンテンツサイズを指定する際に使用されます。
- このクラスは不変（イミュータブル）として扱われるべきですが、`freezed`は使用していません。必要に応じて`freezed`を導入することも検討できます。

## 関連するクラス

- [PositionedPerson](./PositionedPerson.md) - 配置された人物ノードの情報を保持するクラス
- [PositionedRelationship](./PositionedRelationship.md) - 配置された関係線の情報を保持するクラス
- [TreeLayoutEngine](../ウィジェット/TreeLayoutEngine.md) - 家系図のレイアウト計算を行うクラス
- [FamilyTreePainter](../ウィジェット/FamilyTreePainter.md) - 家系図の描画を行う`CustomPainter`
