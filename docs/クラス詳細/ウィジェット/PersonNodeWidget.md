# PersonNodeWidget クラス

## 概要

`PersonNodeWidget` クラスは、家系図上に表示される個々の人物ノードを描画・管理するウィジェットです。人物の基本情報（氏名、生没年など）を表示し、選択状態や性別に応じた視覚的なフィードバックを提供します。ユーザーがノードをタップした際のインタラクションも処理します。

## クラス定義

```dart
// lib/features/family_tree/presentation/widgets/person_node_widget.dart

class PersonNodeWidget extends StatelessWidget {
  final PositionedPerson positionedPerson;
  final bool isSelected;
  final VoidCallback? onTap;
  final double scale; // 現在の家系図全体の表示スケール

  const PersonNodeWidget({
    Key? key,
    required this.positionedPerson,
    this.isSelected = false,
    this.onTap,
    this.scale = 1.0,
  }) : super(key: key);

  // ノードの基本サイズ (スケール1.0の時)
  static const double baseWidth = 160.0;
  static const double baseHeight = 70.0;
  static const double cornerRadius = 8.0;
  static const double borderWidth = 1.5;
  static const double selectedBorderWidth = 2.5;
  static const double padding = 8.0;

  @override
  Widget build(BuildContext context) {
    final person = positionedPerson.person;
    
    // スケールに応じたフォントサイズとパディングの調整
    final double currentFontSize = max(8.0, 14.0 * scale);
    final double currentDateFontSize = max(6.0, 11.0 * scale);
    final double currentPadding = max(4.0, padding * scale);
    final double currentCornerRadius = max(4.0, cornerRadius * scale);

    return Positioned(
      left: positionedPerson.position.dx,
      top: positionedPerson.position.dy,
      width: positionedPerson.size.width,
      height: positionedPerson.size.height,
      child: GestureDetector(
        onTap: onTap,
        child: Semantics(
          label: '人物: ${person.name}', // アクセシビリティ用ラベル
          selected: isSelected,
          button: true,
          child: Container(
            decoration: BoxDecoration(
              color: Colors.white,
              borderRadius: BorderRadius.circular(currentCornerRadius),
              border: Border.all(
                color: _getNodeBorderColor(person.gender, isSelected),
                width: isSelected ? selectedBorderWidth : borderWidth,
              ),
              boxShadow: [
                BoxShadow(
                  color: Colors.black.withOpacity(0.15),
                  blurRadius: 3.0 * scale,
                  offset: Offset(1.0 * scale, 1.0 * scale),
                ),
              ],
            ),
            padding: EdgeInsets.all(currentPadding),
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              crossAxisAlignment: CrossAxisAlignment.center,
              children: [
                // 氏名
                Text(
                  person.name,
                  textAlign: TextAlign.center,
                  style: TextStyle(
                    fontSize: currentFontSize,
                    fontWeight: FontWeight.bold,
                    color: Colors.black87,
                  ),
                  maxLines: 2,
                  overflow: TextOverflow.ellipsis,
                ),
                // 生没年
                if (person.birthDate != null || person.deathDate != null)
                  Padding(
                    padding: EdgeInsets.only(top: 2.0 * scale),
                    child: Text(
                      _getLifeSpanText(person),
                      textAlign: TextAlign.center,
                      style: TextStyle(
                        fontSize: currentDateFontSize,
                        color: Colors.grey.shade700,
                      ),
                      maxLines: 1,
                      overflow: TextOverflow.ellipsis,
                    ),
                  ),
              ],
            ),
          ),
        ),
      ),
    );
  }

  // ノードの枠線の色を取得
  Color _getNodeBorderColor(Gender? gender, bool selected) {
    if (selected) {
      return Colors.amber.shade700; // 選択時はアンバー
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

  // 生没年のテキストを生成
  String _getLifeSpanText(Person person) {
    String text = '';
    if (person.birthDate != null) {
      text = '${person.birthDate!.year}年';
      if (person.deathDate != null) {
        text += ' 〜 ${person.deathDate!.year}年';
      } else {
        text += ' 〜 存命'; // または単に ' 〜'
      }
    } else if (person.deathDate != null) {
      text = '? 〜 ${person.deathDate!.year}年';
    }
    return text;
  }
  
  // スケールに応じた最小フォントサイズを保証するヘルパー
  double max(double a, double b) => math.max(a,b);
}
```

## プロパティ

| 名前 | 型 | 説明 | デフォルト値 |
|------|------|------|------------|
| `positionedPerson` | `PositionedPerson` | 表示する人物の位置情報と元データ | なし（必須） |
| `isSelected` | `bool` | このノードが現在選択されているかどうか | `false` |
| `onTap` | `VoidCallback?` | ノードがタップされたときに呼び出されるコールバック | `null` |
| `scale` | `double` | 現在の家系図全体の表示スケール。フォントサイズなどの調整に使用。 | `1.0` |

## 静的定数

| 名前 | 型 | 値 | 説明 |
|------|------|----|------|
| `baseWidth` | `double` | `160.0` | スケール1.0時のノードの基本幅 |
| `baseHeight` | `double` | `70.0` | スケール1.0時のノードの基本高さ |
| `cornerRadius` | `double` | `8.0` | ノードの角丸半径 |
| `borderWidth` | `double` | `1.5` | 通常時の枠線太さ |
| `selectedBorderWidth` | `double` | `2.5` | 選択時の枠線太さ |
| `padding` | `double` | `8.0` | ノード内部のパディング |

## メソッド

### build

```dart
Widget build(BuildContext context)
```

ウィジェットのUIを構築します。`Positioned`ウィジェットを使用してキャンバス上の指定された位置に配置され、`GestureDetector`でタップイベントを処理します。内部の表示は`Container`と`Column`、`Text`ウィジェットで構成されます。

### _getNodeBorderColor (プライベートヘルパー)

```dart
Color _getNodeBorderColor(Gender? gender, bool selected)
```

人物の性別と選択状態に基づいて、ノードの枠線の色を決定します。

### _getLifeSpanText (プライベートヘルパー)

```dart
String _getLifeSpanText(Person person)
```

人物の生年月日と没年月日から表示用の文字列（例: "1980年 〜 2020年"）を生成します。

## 使用例

`PersonNodeWidget`は通常、`InteractiveFamilyTree`ウィジェット内の`Stack`または`CustomPaint`と組み合わせて、`TreeLayout`から得られる`PositionedPerson`のリストに基づいて動的に生成・配置されます。

```dart
// InteractiveFamilyTreeウィジェット内での使用イメージ (簡略化)
Stack(
  children: treeLayout.positionedPersons.map((pPerson) {
    return PersonNodeWidget(
      key: ValueKey(pPerson.person.id), // リビルド最適化のためのキー
      positionedPerson: pPerson,
      isSelected: pPerson.person.id == selectedPersonId,
      onTap: () => onPersonTap(pPerson.person.id),
      scale: currentGlobalScale, // InteractiveViewerなどから取得したスケール値
    );
  }).toList(),
)
```

## 注意事項

- このウィジェットは、`Positioned`ウィジェットを使用して絶対位置に配置されることを想定しています。
- `scale`プロパティは、家系図全体のズームレベルに応じてフォントサイズやパディングを動的に調整するために重要です。これにより、ズームアウトしてもテキストが読める範囲で小さくなり、ズームインすれば詳細がはっきり見えるようになります。
- `baseWidth`と`baseHeight`は、`TreeLayoutEngine`がノードサイズを計算する際の基準値としても使用されることを想定しています。`positionedPerson.size`には、この基本サイズにレイアウトエンジンによる調整が加わった値が入ります。
- アクセシビリティ向上のため、`Semantics`ウィジェットで適切なラベルを設定しています。
- `math.max` を使用するために `dart:math` のインポートが必要です。

## 関連するクラス

- [PositionedPerson](../モデル/PositionedPerson.md) - このウィジェットが表示する人物の位置情報と元データを保持するクラス
- [Person](../モデル/Person.md) - 人物の詳細情報を保持するクラス
- [Gender](../モデル/Person.md) - 性別を表す列挙型
- [InteractiveFamilyTree](./InteractiveFamilyTree.md) - このウィジェットを多数配置してインタラクティブな家系図を構成する親ウィジェット
