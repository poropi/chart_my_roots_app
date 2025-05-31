# InteractiveFamilyTree ウィジェット

## 概要

`InteractiveFamilyTree` ウィジェットは、家系図をインタラクティブに表示・操作するためのUIコンポーネントです。ズーム、パン（スクロール）、人物ノードのタップによる選択などの機能を提供します。`FamilyTreePainter` を使用して実際の描画を行い、ユーザーの操作に応じて表示を更新します。

## クラス定義

```dart
// lib/features/family_tree/presentation/widgets/interactive_family_tree.dart

class InteractiveFamilyTree extends StatefulWidget {
  final TreeLayout layout;
  final String? selectedPersonId;
  final Function(String personId) onPersonTap;
  final Function(Offset position) onPan;
  final Function(double scale) onScaleUpdate;
  final double initialScale;
  final Offset initialPosition;

  const InteractiveFamilyTree({
    Key? key,
    required this.layout,
    this.selectedPersonId,
    required this.onPersonTap,
    required this.onPan,
    required this.onScaleUpdate,
    this.initialScale = 1.0,
    this.initialPosition = Offset.zero,
  }) : super(key: key);

  @override
  State<InteractiveFamilyTree> createState() => _InteractiveFamilyTreeState();
}

class _InteractiveFamilyTreeState extends State<InteractiveFamilyTree> {
  late TransformationController _transformationController;
  late double _currentScale;
  Offset _currentPosition = Offset.zero;

  // ズームの最小/最大値
  static const double _minScale = 0.2;
  static const double _maxScale = 3.0;

  @override
  void initState() {
    super.initState();
    _currentScale = widget.initialScale.clamp(_minScale, _maxScale);
    _currentPosition = widget.initialPosition;
    _transformationController = TransformationController(
      Matrix4.identity()
        ..translate(_currentPosition.dx, _currentPosition.dy)
        ..scale(_currentScale),
    );
  }

  @override
  void didUpdateWidget(covariant InteractiveFamilyTree oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (widget.initialScale != oldWidget.initialScale || widget.initialPosition != oldWidget.initialPosition) {
      _currentScale = widget.initialScale.clamp(_minScale, _maxScale);
      _currentPosition = widget.initialPosition;
      _transformationController.value = Matrix4.identity()
        ..translate(_currentPosition.dx, _currentPosition.dy)
        ..scale(_currentScale);
      widget.onScaleUpdate(_currentScale);
      widget.onPan(_currentPosition);
    }
  }

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        return GestureDetector(
          onTapUp: (details) => _handleTap(details, constraints.biggest),
          child: InteractiveViewer(
            transformationController: _transformationController,
            minScale: _minScale,
            maxScale: _maxScale,
            constrained: false, // 子ウィジェットのサイズに依存させる
            boundaryMargin: EdgeInsets.all(double.infinity), // 無限スクロールを許可
            onInteractionUpdate: (details) {
              // 現在のスケールと位置を更新
              final matrix = _transformationController.value;
              final newScale = matrix.getMaxScaleOnAxis();
              final newPosition = Offset(matrix.row0.w, matrix.row1.w);

              if (_currentScale != newScale) {
                _currentScale = newScale;
                widget.onScaleUpdate(_currentScale);
              }
              if (_currentPosition != newPosition) {
                _currentPosition = newPosition;
                widget.onPan(_currentPosition);
              }
            },
            child: SizedBox(
              width: widget.layout.requiredSize.width,
              height: widget.layout.requiredSize.height,
              child: CustomPaint(
                painter: FamilyTreePainter(
                  layout: widget.layout,
                  selectedPersonId: widget.selectedPersonId,
                  // FamilyTreePainterは独自のスケールを持たず、InteractiveViewerのスケールに依存
                ),
                size: widget.layout.requiredSize,
              ),
            ),
          ),
        );
      },
    );
  }

  void _handleTap(TapUpDetails details, Size viewSize) {
    // タップ位置をCustomPaint内のローカル座標に変換
    final Matrix4 matrix = _transformationController.value.clone();
    final Matrix4 invertedMatrix = Matrix4.inverted(matrix);
    final Offset localTapPosition = MatrixUtils.transformPoint(invertedMatrix, details.localPosition);

    // タップ位置に存在する人物を検索
    for (int i = widget.layout.positionedPersons.length - 1; i >= 0; i--) {
      final positionedPerson = widget.layout.positionedPersons[i];
      final personRect = Rect.fromLTWH(
        positionedPerson.position.dx,
        positionedPerson.position.dy,
        positionedPerson.size.width,
        positionedPerson.size.height,
      );

      if (personRect.contains(localTapPosition)) {
        widget.onPersonTap(positionedPerson.person.id);
        return;
      }
    }
    // 人物ノード以外がタップされた場合は選択を解除（オプション）
    // widget.onPersonTap(null); 
  }
  
  // 外部から特定の位置に移動・ズームするためのメソッド
  void transformTo({
    required Offset targetPosition,
    required double targetScale,
    required Size viewSize,
  }) {
    final clampedScale = targetScale.clamp(_minScale, _maxScale);
    
    // 画面中央にターゲット位置が来るように計算
    final newX = viewSize.width / 2 - targetPosition.dx * clampedScale;
    final newY = viewSize.height / 2 - targetPosition.dy * clampedScale;
    
    _transformationController.value = Matrix4.identity()
      ..translate(newX, newY)
      ..scale(clampedScale);
      
    _currentScale = clampedScale;
    _currentPosition = Offset(newX, newY);
    widget.onScaleUpdate(_currentScale);
    widget.onPan(_currentPosition);
  }
}
```

## プロパティ

| 名前 | 型 | 説明 | デフォルト値 |
|------|------|------|------------|
| `layout` | `TreeLayout` | 表示する家系図のレイアウト情報 | なし（必須） |
| `selectedPersonId` | `String?` | 現在選択されている人物のID | `null` |
| `onPersonTap` | `Function(String personId)` | 人物ノードがタップされたときのコールバック | なし（必須） |
| `onPan` | `Function(Offset position)` | 表示位置が変更されたときのコールバック | なし（必須） |
| `onScaleUpdate` | `Function(double scale)` | ズームレベルが変更されたときのコールバック | なし（必須） |
| `initialScale` | `double` | 初期ズームレベル | `1.0` |
| `initialPosition` | `Offset` | 初期表示位置 | `Offset.zero` |

## 内部状態

| 名前 | 型 | 説明 |
|------|------|------|
| `_transformationController` | `TransformationController` | `InteractiveViewer`の変換状態を管理 |
| `_currentScale` | `double` | 現在のズームレベル |
| `_currentPosition` | `Offset` | 現在の表示位置（左上隅） |

## 定数

| 名前 | 型 | 値 | 説明 |
|------|------|----|------|
| `_minScale` | `double` | `0.2` | 最小ズームレベル |
| `_maxScale` | `double` | `3.0` | 最大ズームレベル |

## メソッド

### initState

```dart
void initState()
```

ウィジェットの初期化時に呼び出されます。`TransformationController`と現在のスケール・位置を初期化します。

### didUpdateWidget

```dart
void didUpdateWidget(covariant InteractiveFamilyTree oldWidget)
```

ウィジェットのプロパティが更新されたときに呼び出されます。`initialScale`または`initialPosition`が変更された場合、表示を更新します。

### build

```dart
Widget build(BuildContext context)
```

ウィジェットのUIを構築します。`InteractiveViewer`と`CustomPaint`を使用して、インタラクティブな家系図表示を実現します。

### _handleTap

```dart
void _handleTap(TapUpDetails details, Size viewSize)
```

タップイベントを処理します。タップされた位置にある人物ノードを検出し、`onPersonTap`コールバックを呼び出します。

**パラメータ:**
- `details`: `TapUpDetails` - タップイベントの詳細情報
- `viewSize`: `Size` - `InteractiveViewer`の表示領域サイズ

### transformTo

```dart
void transformTo({
  required Offset targetPosition,
  required double targetScale,
  required Size viewSize,
})
```

指定されたターゲット位置とスケールに家系図の表示をアニメーションなしで変更します。外部から特定の人物を中心に表示する際などに使用されます。

**パラメータ:**
- `targetPosition`: `Offset` - 中心に表示したい家系図上のローカル座標
- `targetScale`: `double` - 設定したいズームスケール
- `viewSize`: `Size` - `InteractiveViewer`の表示領域サイズ

## 使用例

```dart
// コントローラからInteractiveFamilyTreeのStateにアクセスするためのGlobalKey
final GlobalKey<_InteractiveFamilyTreeState> interactiveTreeKey = GlobalKey();

// ...

InteractiveFamilyTree(
  key: interactiveTreeKey,
  layout: treeLayout, // TreeLayoutオブジェクト
  selectedPersonId: selectedPersonId, // 選択中の人物ID
  initialScale: currentZoomLevel, // FamilyTreeViewControllerから取得
  initialPosition: currentViewPosition, // FamilyTreeViewControllerから取得
  onPersonTap: (personId) {
    ref.read(familyTreeViewControllerProvider).selectPerson(personId);
  },
  onPan: (position) {
    ref.read(familyTreeViewControllerProvider).updatePosition(position);
  },
  onScaleUpdate: (scale) {
    ref.read(familyTreeViewControllerProvider).setZoomLevel(scale);
  },
)

// 特定の人物を中心に表示する例
void centerOnPerson(String personId, Size viewSize) {
  final targetPersonNode = treeLayout.positionedPersons.firstWhereOrNull((p) => p.person.id == personId);
  if (targetPersonNode != null) {
    final targetPosition = Offset(
      targetPersonNode.position.dx + targetPersonNode.size.width / 2,
      targetPersonNode.position.dy + targetPersonNode.size.height / 2,
    );
    interactiveTreeKey.currentState?.transformTo(
      targetPosition: targetPosition,
      targetScale: 1.0, // 100%スケールで表示
      viewSize: viewSize, // InteractiveViewerのサイズ
    );
  }
}
```

## 注意事項

- このウィジェットは、Flutterの`InteractiveViewer`を内部で使用して、ズームとパンの機能を実現しています。
- `constrained`プロパティを`false`に設定し、`boundaryMargin`を`EdgeInsets.all(double.infinity)`にすることで、家系図のサイズがビューポートより大きい場合でも自由にスクロールできるようにしています。
- タップ位置の判定は、`InteractiveViewer`の変換行列を考慮して行う必要があります。
- `FamilyTreePainter`は、このウィジェットの`CustomPaint`内で使用され、実際の家系図の描画を担当します。`FamilyTreePainter`自体はスケール情報を持たず、`InteractiveViewer`によってスケーリングされたキャンバスに描画します。
- `initialScale`と`initialPosition`プロパティは、ウィジェットが最初に表示されるときのズームレベルと位置を指定します。これらの値が外部から変更された場合、`didUpdateWidget`で適切に再設定されます。
- `transformTo`メソッドは、外部からプログラム的に表示位置やスケールを変更する際に使用できます。例えば、「選択した人物を中心に表示」機能の実装に役立ちます。

## 関連するクラス

- [FamilyTreePainter](./FamilyTreePainter.md) - 家系図の描画を担当する`CustomPainter`
- [TreeLayout](../モデル/TreeLayout.md) - 家系図のレイアウト情報を保持するクラス
- [FamilyTreeViewController](../コントローラ/FamilyTreeViewController.md) - 家系図表示の操作を管理するコントローラ
