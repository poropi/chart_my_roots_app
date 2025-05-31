# PersonDetailPanel ウィジェット

## 概要

`PersonDetailPanel` ウィジェットは、家系図上で選択された人物の詳細情報を表示するためのUIコンポーネントです。氏名、旧姓、生年月日、没年月日、性別、メモなどの情報を整理して表示し、関連する操作（編集、関係追加など）への導線を提供します。このパネルは、デスクトップレイアウトではサイドパネルとして、モバイル/タブレットレイアウトではボトムシートとして表示されることを想定しています。

## クラス定義

```dart
// lib/features/family_tree/presentation/widgets/person_detail_panel.dart

class PersonDetailPanel extends ConsumerWidget {
  const PersonDetailPanel({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // 選択中の人物情報を取得
    final selectedPerson = ref.watch(selectedPersonProvider);

    if (selectedPerson == null) {
      return _buildEmptyState(context);
    }

    // 日付フォーマッタ
    final dateFormat = DateFormat('yyyy年MM月dd日');

    return Padding(
      padding: const EdgeInsets.all(16.0),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        mainAxisSize: MainAxisSize.min, // ボトムシートで高さを適切に保つため
        children: [
          // 人物名
          Text(
            selectedPerson.name,
            style: Theme.of(context).textTheme.headlineSmall?.copyWith(
              fontWeight: FontWeight.bold,
            ),
          ),
          if (selectedPerson.maidenName != null && selectedPerson.maidenName!.isNotEmpty)
            Padding(
              padding: const EdgeInsets.only(top: 2.0),
              child: Text(
                '(旧姓: ${selectedPerson.maidenName})',
                style: Theme.of(context).textTheme.titleMedium?.copyWith(
                  color: Colors.grey.shade700,
                ),
              ),
            ),
          
          Divider(height: 24, thickness: 1),
          
          // 基本情報セクション
          _buildInfoRow(context, '生年月日', 
            selectedPerson.birthDate != null 
              ? dateFormat.format(selectedPerson.birthDate!) 
              : '未設定'),
          _buildInfoRow(context, '没年月日', 
            selectedPerson.deathDate != null 
              ? dateFormat.format(selectedPerson.deathDate!) 
              : (selectedPerson.birthDate != null ? '存命中' : '未設定')),
          _buildInfoRow(context, '性別', 
            selectedPerson.gender != null 
              ? selectedPerson.gender!.displayName 
              : '未設定'),
          
          // メモセクション
          if (selectedPerson.memo != null && selectedPerson.memo!.isNotEmpty) ...[
            SizedBox(height: 16),
            Text(
              'メモ',
              style: Theme.of(context).textTheme.titleSmall?.copyWith(
                fontWeight: FontWeight.bold,
                color: Colors.blueGrey.shade700,
              ),
            ),
            SizedBox(height: 4),
            Container(
              width: double.infinity,
              padding: EdgeInsets.all(10),
              decoration: BoxDecoration(
                color: Colors.grey.shade100,
                borderRadius: BorderRadius.circular(6),
                border: Border.all(color: Colors.grey.shade300)
              ),
              child: Text(
                selectedPerson.memo!,
                style: Theme.of(context).textTheme.bodyMedium,
              ),
            ),
          ],
          
          SizedBox(height: 24),
          
          // 操作ボタンセクション
          _buildActionButtons(context, ref, selectedPerson),
        ],
      ),
    );
  }

  // 人物が選択されていない場合の表示
  Widget _buildEmptyState(BuildContext context) {
    return Center(
      child: Padding(
        padding: const EdgeInsets.symmetric(vertical: 48.0, horizontal: 16.0),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(
              Icons.person_search_outlined,
              size: 64,
              color: Colors.grey.shade400,
            ),
            SizedBox(height: 16),
            Text(
              '人物を選択すると詳細情報が表示されます。',
              style: Theme.of(context).textTheme.titleMedium?.copyWith(
                color: Colors.grey.shade600,
              ),
              textAlign: TextAlign.center,
            ),
          ],
        ),
      ),
    );
  }

  // 情報行を構築するヘルパー
  Widget _buildInfoRow(BuildContext context, String label, String value) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 6.0),
      child: Row(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          SizedBox(
            width: 80,
            child: Text(
              label,
              style: Theme.of(context).textTheme.bodyMedium?.copyWith(
                fontWeight: FontWeight.bold,
                color: Colors.blueGrey.shade800,
              ),
            ),
          ),
          SizedBox(width: 12),
          Expanded(
            child: Text(
              value,
              style: Theme.of(context).textTheme.bodyMedium,
            ),
          ),
        ],
      ),
    );
  }

  // 操作ボタンを構築するヘルパー
  Widget _buildActionButtons(BuildContext context, WidgetRef ref, Person selectedPerson) {
    return Column(
      children: [
        Row(
          children: [
            Expanded(
              child: ElevatedButton.icon(
                icon: Icon(Icons.edit, size: 18),
                label: Text('編集'),
                style: ElevatedButton.styleFrom(
                  padding: EdgeInsets.symmetric(vertical: 12),
                ),
                onPressed: () {
                  // 人物編集ダイアログを表示
                  ref.read(personFormProvider.notifier).initWithPerson(selectedPerson);
                  showDialog(
                    context: context,
                    builder: (_) => ResponsivePersonEditDialog(
                      child: PersonEditDialog(isNew: false),
                    ),
                  );
                },
              ),
            ),
            SizedBox(width: 12),
            Expanded(
              child: ElevatedButton.icon(
                icon: Icon(Icons.link, size: 18),
                label: Text('関係追加'),
                style: ElevatedButton.styleFrom(
                  padding: EdgeInsets.symmetric(vertical: 12),
                ),
                onPressed: () {
                  // 関係性追加ダイアログを表示
                  showDialog(
                    context: context,
                    builder: (_) => ResponsiveRelationshipDialog(
                      child: RelationshipDialog(fromPersonId: selectedPerson.id),
                    ),
                  );
                },
              ),
            ),
          ],
        ),
        SizedBox(height: 12),
        SizedBox(
          width: double.infinity,
          child: OutlinedButton.icon(
            icon: Icon(Icons.center_focus_strong, size: 18),
            label: Text('中心に表示'),
            style: OutlinedButton.styleFrom(
              padding: EdgeInsets.symmetric(vertical: 12),
            ),
            onPressed: () {
              // 選択中の人物を中心に表示
              final controller = ref.read(familyTreeViewControllerProvider);
              final viewSize = MediaQuery.of(context).size; // 現在のビューサイズ
              controller.centerOnPerson(selectedPerson.id, viewSize);
              
              // モバイル/タブレットでボトムシートが表示されている場合は閉じる
              if (Navigator.canPop(context)) {
                 final deviceType = ref.read(deviceTypeProvider);
                 if(deviceType == DeviceType.mobile || deviceType == DeviceType.tablet){
                    Navigator.pop(context);
                 }
              }
            },
          ),
        ),
      ],
    );
  }
}
```

## プロパティ

なし（`ConsumerWidget`のため、プロパティは持たず、Riverpodプロバイダから状態を取得します）

## 内部状態

なし（状態はRiverpodプロバイダによって管理されます）

## メソッド

### build

```dart
Widget build(BuildContext context, WidgetRef ref)
```

ウィジェットのUIを構築します。`selectedPersonProvider`を監視し、選択された人物がいればその詳細情報を表示し、いなければ空の状態を表示します。

### _buildEmptyState (プライベートヘルパー)

```dart
Widget _buildEmptyState(BuildContext context)
```

人物が選択されていない場合に表示するプレースホルダーUIを構築します。

### _buildInfoRow (プライベートヘルパー)

```dart
Widget _buildInfoRow(BuildContext context, String label, String value)
```

ラベルと値のペアで情報を表示する行ウィジェットを構築します。

### _buildActionButtons (プライベートヘルパー)

```dart
Widget _buildActionButtons(BuildContext context, WidgetRef ref, Person selectedPerson)
```

選択された人物に対する操作ボタン（編集、関係追加、中心に表示）を構築します。

## 使用例

`PersonDetailPanel`は、メイン画面のサイドパネルやボトムシート内に配置されて使用されます。

```dart
// メイン画面のデスクトップレイアウトでの使用例
Row(
  children: [
    SizedBox(
      width: 300,
      child: Card(
        margin: EdgeInsets.all(8),
        elevation: 4,
        child: PersonDetailPanel(), // ここで使用
      ),
    ),
    Expanded(
      child: InteractiveFamilyTree(...),
    ),
  ],
)

// メイン画面のモバイル/タブレットレイアウトでのボトムシート表示例
void _showPersonDetailBottomSheet(BuildContext context, WidgetRef ref) {
  showModalBottomSheet(
    context: context,
    builder: (context) {
      return DraggableScrollableSheet(
        builder: (context, scrollController) {
          return SingleChildScrollView(
            controller: scrollController,
            child: PersonDetailPanel(), // ここで使用
          );
        },
      );
    },
  );
}
```

## 注意事項

- このウィジェットは、`selectedPersonProvider`を通じて現在選択されている人物の情報を取得します。このプロバイダは、ユーザーが家系図上の人物ノードをタップした際に更新されることを想定しています。
- 表示される情報は、`Person`モデルの各プロパティに基づきます。日付は`intl`パッケージの`DateFormat`を使用して整形されます。
- 「編集」ボタンは`PersonEditDialog`を、「関係追加」ボタンは`RelationshipDialog`をそれぞれ表示します。これらのダイアログは、選択された人物の情報を元に初期化されます。
- 「中心に表示」ボタンは、`FamilyTreeViewController`の`centerOnPerson`メソッドを呼び出して、家系図表示エリアで選択された人物が中央に来るように調整します。
- レスポンシブ対応として、このパネル自体は情報を表示する責務に集中し、表示方法（サイドパネルかボトムシートか）は親ウィジェット（例：`MainScreen`）が決定します。
- `Gender`列挙型には、表示名を取得するための`displayName`拡張メソッドが定義されていることを前提としています（例：`enum Gender { male, female, other; String get displayName => ...; }`）。

## 関連するクラス・プロバイダ

- [Person](../モデル/Person.md) - 表示する人物データのモデルクラス
- `selectedPersonProvider` (Riverpodプロバイダ) - 現在選択されている`Person`オブジェクトを提供するプロバイダ
- `personFormProvider` (Riverpodプロバイダ) - 人物編集フォームの状態を管理するプロバイダ
- [PersonEditDialog](./PersonEditDialog.md) - 人物情報を編集するためのダイアログウィジェット
- [RelationshipDialog](./RelationshipDialog.md) - 関係性を設定するためのダイアログウィジェット
- [FamilyTreeViewController](../コントローラ/FamilyTreeViewController.md) - 家系図表示の操作を管理するコントローラ
- `deviceTypeProvider` (Riverpodプロバイダ) - 現在のデバイスタイプ（モバイル/タブレット/デスクトップ）を提供するプロバイダ
