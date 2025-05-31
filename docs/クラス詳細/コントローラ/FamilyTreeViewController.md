# FamilyTreeViewController クラス

## 概要

`FamilyTreeViewController` クラスは家系図の表示に関する操作を管理するコントローラクラスです。Riverpodプロバイダを通じてアクセスされ、表示方向の切り替え、ズームレベルの調整、表示位置の更新などの操作を提供します。また、家系図のレイアウト計算を行うプロバイダとも連携します。

## クラス定義

```dart
class FamilyTreeViewController {
  final Ref _ref;
  
  FamilyTreeViewController(this._ref);
  
  // 表示方向の切り替え
  void toggleOrientation() {
    final currentOrientation = _ref.read(layoutOrientationProvider);
    final newOrientation = currentOrientation == LayoutOrientation.vertical
        ? LayoutOrientation.horizontal
        : LayoutOrientation.vertical;
    
    _ref.read(layoutOrientationProvider.notifier).state = newOrientation;
  }
  
  // 表示方向を直接設定
  void setOrientation(LayoutOrientation orientation) {
    _ref.read(layoutOrientationProvider.notifier).state = orientation;
  }
  
  // ズームレベルの設定
  void setZoomLevel(double level) {
    // 範囲を制限（25%〜200%）
    final clampedLevel = level.clamp(0.25, 2.0);
    _ref.read(zoomLevelProvider.notifier).state = clampedLevel;
  }
  
  // ズームイン
  void zoomIn() {
    final currentZoom = _ref.read(zoomLevelProvider);
    setZoomLevel(currentZoom * 1.2); // 20%ずつ拡大
  }
  
  // ズームアウト
  void zoomOut() {
    final currentZoom = _ref.read(zoomLevelProvider);
    setZoomLevel(currentZoom / 1.2); // 20%ずつ縮小
  }
  
  // 表示位置の更新
  void updatePosition(Offset position) {
    _ref.read(viewPositionProvider.notifier).state = position;
  }
  
  // 人物の選択
  void selectPerson(String? personId) {
    _ref.read(selectedPersonIdProvider.notifier).state = personId;
  }
  
  // 選択した人物を中心に表示
  void centerOnSelectedPerson() {
    final selectedPersonId = _ref.read(selectedPersonIdProvider);
    if (selectedPersonId == null) return;
    
    final treeLayout = _ref.read(treeLayoutProvider).valueOrNull;
    if (treeLayout == null) return;
    
    final positionedPerson = treeLayout.positionedPersons.firstWhereOrNull(
      (p) => p.person.id == selectedPersonId
    );
    if (positionedPerson == null) return;
    
    // 表示サイズを取得
    final screenSize = _ref.read(screenSizeProvider);
    if (screenSize == Size.zero) return;
    
    // 現在のズームレベル
    final zoomLevel = _ref.read(zoomLevelProvider);
    
    // 人物の中心座標を計算
    final personCenterX = positionedPerson.position.dx + positionedPerson.size.width / 2;
    final personCenterY = positionedPerson.position.dy + positionedPerson.size.height / 2;
    
    // 画面中央に配置するための座標を計算
    final newX = screenSize.width / 2 - personCenterX * zoomLevel;
    final newY = screenSize.height / 2 - personCenterY * zoomLevel;
    
    // 位置を更新
    updatePosition(Offset(newX, newY));
  }
  
  // 家系図全体を表示
  void showEntireTree() {
    final treeLayout = _ref.read(treeLayoutProvider).valueOrNull;
    if (treeLayout == null) return;
    
    final screenSize = _ref.read(screenSizeProvider);
    if (screenSize == Size.zero) return;
    
    // 家系図全体を表示するためのズームレベルを計算
    final requiredWidth = treeLayout.requiredSize.width;
    final requiredHeight = treeLayout.requiredSize.height;
    
    final scaleX = (screenSize.width - 40) / requiredWidth; // マージンを考慮
    final scaleY = (screenSize.height - 40) / requiredHeight; // マージンを考慮
    
    // 小さい方のスケールを採用（両方の軸で収まるように）
    final newZoom = min(scaleX, scaleY).clamp(0.25, 2.0);
    
    // ズームレベルを設定
    setZoomLevel(newZoom);
    
    // 中央に配置
    final newX = (screenSize.width - requiredWidth * newZoom) / 2;
    final newY = (screenSize.height - requiredHeight * newZoom) / 2;
    
    updatePosition(Offset(newX, newY));
  }
}
```

## 関連するプロバイダ

```dart
// 表示方向の状態
final layoutOrientationProvider = StateProvider<LayoutOrientation>((ref) {
  return LayoutOrientation.vertical; // デフォルトは縦型表示
});

// ズームレベルの状態
final zoomLevelProvider = StateProvider<double>((ref) {
  return 1.0; // 初期値: 100%
});

// 表示位置の状態
final viewPositionProvider = StateProvider<Offset>((ref) {
  return Offset.zero;
});

// 画面サイズの状態
final screenSizeProvider = StateProvider<Size>((ref) {
  return Size.zero; // 初期化時は0、実際のサイズは後から設定
});

// 選択中の人物IDの状態 (personControllerから共有)
final selectedPersonIdProvider = StateProvider<String?>((ref) => null);

// レイアウト計算結果の提供
final treeLayoutProvider = Provider<AsyncValue<TreeLayout>>((ref) {
  final persons = ref.watch(personsStreamProvider);
  final relationships = ref.watch(relationshipsStreamProvider);
  final orientation = ref.watch(layoutOrientationProvider);
  
  // 人物・関係性データがロード中またはエラーの場合はその状態を返す
  if (persons is AsyncLoading) {
    return const AsyncLoading();
  }
  if (persons is AsyncError) {
    return AsyncError(persons.error, persons.stackTrace);
  }
  if (relationships is AsyncLoading) {
    return const AsyncLoading();
  }
  if (relationships is AsyncError) {
    return AsyncError(relationships.error, relationships.stackTrace);
  }
  
  // データが揃っている場合はレイアウトを計算
  final personsData = persons.value ?? [];
  final relationshipsData = relationships.value ?? [];
  
  // 選択中の人物を取得
  final selectedPersonId = ref.watch(selectedPersonIdProvider);
  Person? focusedPerson;
  if (selectedPersonId != null) {
    focusedPerson = personsData.firstWhereOrNull(
      (p) => p.id == selectedPersonId,
    );
  }
  
  try {
    // レイアウトエンジンでノード位置を計算
    final layout = TreeLayoutEngine.calculateLayout(
      persons: personsData,
      relationships: relationshipsData,
      orientation: orientation,
      focusedPerson: focusedPerson,
    );
    return AsyncData(layout);
  } catch (e, stackTrace) {
    return AsyncError(e, stackTrace);
  }
});

// 家系図表示コントローラのプロバイダ
final familyTreeViewControllerProvider = Provider<FamilyTreeViewController>((ref) {
  return FamilyTreeViewController(ref);
});
```

## メソッド

### toggleOrientation

```dart
void toggleOrientation()
```

家系図の表示方向（縦型/横型）を切り替えます。

**戻り値:**
- なし

### setOrientation

```dart
void setOrientation(LayoutOrientation orientation)
```

家系図の表示方向を直接指定します。

**パラメータ:**
- `orientation`: `LayoutOrientation` - 設定する表示方向（縦型/横型）

**戻り値:**
- なし

### setZoomLevel

```dart
void setZoomLevel(double level)
```

家系図のズームレベルを設定します。値は0.25（25%）〜2.0（200%）の範囲に制限されます。

**パラメータ:**
- `level`: `double` - 設定するズームレベル（1.0 = 100%）

**戻り値:**
- なし

### zoomIn

```dart
void zoomIn()
```

家系図を拡大します（現在のズームレベルから20%拡大）。

**戻り値:**
- なし

### zoomOut

```dart
void zoomOut()
```

家系図を縮小します（現在のズームレベルから20%縮小）。

**戻り値:**
- なし

### updatePosition

```dart
void updatePosition(Offset position)
```

家系図の表示位置を更新します。

**パラメータ:**
- `position`: `Offset` - 新しい表示位置

**戻り値:**
- なし

### selectPerson

```dart
void selectPerson(String? personId)
```

指定された人物を選択状態にします。`null`を指定すると選択を解除します。

**パラメータ:**
- `personId`: `String?` - 選択する人物のID、または選択解除の場合は`null`

**戻り値:**
- なし

### centerOnSelectedPerson

```dart
void centerOnSelectedPerson()
```

現在選択されている人物を画面中央に表示します。選択されている人物がない場合は何もしません。

**戻り値:**
- なし

### showEntireTree

```dart
void showEntireTree()
```

家系図全体が画面内に収まるようにズームレベルと位置を調整します。

**戻り値:**
- なし

## 使用例

```dart
// コントローラの取得
final controller = ref.read(familyTreeViewControllerProvider);

// 表示方向の切り替え
controller.toggleOrientation();

// 縦型表示に設定
controller.setOrientation(LayoutOrientation.vertical);

// ズームレベルの調整
controller.setZoomLevel(1.5); // 150%
controller.zoomIn();  // 20%拡大
controller.zoomOut(); // 20%縮小

// 人物の選択
controller.selectPerson('person-id');

// 選択した人物を中心に表示
controller.centerOnSelectedPerson();

// 家系図全体を表示
controller.showEntireTree();
```

## 注意事項

- このコントローラは、家系図の表示に関する操作を管理します。実際の描画処理は別のコンポーネント（`InteractiveFamilyTree` ウィジェット）で行われます。
- 表示方向やズームレベルを変更すると、`treeLayoutProvider`を通じて家系図のレイアウトが再計算されます。
- ズームレベルは最小0.25（25%）から最大2.0（200%）の範囲に制限されています。この範囲は必要に応じて調整可能です。
- 画面サイズ情報は、`screenSizeProvider`を通じて提供される必要があります。通常、メイン画面のビルド時に`MediaQuery`から取得して設定します。
- レイアウト計算は比較的重い処理であるため、実際の実装では計算をバックグラウンドスレッド（`compute`関数）で行うことも検討してください。

## 関連するクラス・インターフェース

- [TreeLayout](../モデル/TreeLayout.md) - 家系図のレイアウト計算結果を表すクラス
- [TreeLayoutEngine](../ウィジェット/TreeLayoutEngine.md) - 家系図のレイアウトを計算するクラス
- [InteractiveFamilyTree](../ウィジェット/InteractiveFamilyTree.md) - 家系図を表示・操作するウィジェット
- [Person](../モデル/Person.md) - 家系図内の人物を表すモデルクラス
- [Relationship](../モデル/Relationship.md) - 人物間の関係性を表すモデルクラス

## 関連する列挙型

### LayoutOrientation 列挙型

```dart
enum LayoutOrientation {
  vertical,    // 縦型表示（上から下）
  horizontal,  // 横型表示（左から右）
}
```
