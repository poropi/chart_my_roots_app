# RelationshipDialog ウィジェット

## 概要

`RelationshipDialog` ウィジェットは、家系図内の人物間の関係性（親子関係や配偶者関係）を設定または編集するためのモーダルダイアログUIコンポーネントです。ユーザーは、基準となる人物（`fromPersonId`）に対して、関係タイプ（親子/配偶者）と関係を結ぶ相手の人物を選択できます。既存の関係性を編集するシナリオにも対応します。

## クラス定義

```dart
// lib/features/relationship_editor/presentation/dialogs/relationship_dialog.dart

class RelationshipDialog extends ConsumerStatefulWidget {
  final String fromPersonId; // 関係の起点となる人物のID
  final String? relationshipId; // 編集対象の関係性ID (新規作成時はnull)

  const RelationshipDialog({
    Key? key,
    required this.fromPersonId,
    this.relationshipId,
  }) : super(key: key);

  @override
  ConsumerState<RelationshipDialog> createState() => _RelationshipDialogState();
}

class _RelationshipDialogState extends ConsumerState<RelationshipDialog> {
  @override
  void initState() {
    super.initState();
    // ダイアログ表示時にフォームを初期化
    WidgetsBinding.instance.addPostFrameCallback((_) {
      final notifier = ref.read(relationshipFormProvider.notifier);
      if (widget.relationshipId == null) {
        notifier.initNew(widget.fromPersonId);
      } else {
        notifier.initWithRelationship(widget.relationshipId!, widget.fromPersonId);
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    final formState = ref.watch(relationshipFormProvider);
    final formNotifier = ref.read(relationshipFormProvider.notifier);
    final isSaving = ref.watch(relationshipSavingProvider);
    final errorMessage = ref.watch(relationshipFormErrorProvider);

    final isNew = widget.relationshipId == null;
    final title = isNew ? '関係性の追加' : '関係性の編集';

    // 関係元の人物情報を取得
    final fromPersonAsync = ref.watch(personByIdProvider(widget.fromPersonId));
    // 関係先候補の人物リストを取得
    final availablePersonsAsync = ref.watch(availablePersonsProvider(widget.fromPersonId));
    // 関係先の人物情報を取得 (選択されている場合)
    final toPersonAsync = formState.toPersonId != null 
        ? ref.watch(personByIdProvider(formState.toPersonId!)) 
        : const AsyncValue.data(null);

    return AlertDialog(
      title: Text(title),
      content: SingleChildScrollView(
        child: ConstrainedBox(
          constraints: const BoxConstraints(maxWidth: 500),
          child: Column(
            mainAxisSize: MainAxisSize.min,
            crossAxisAlignment: CrossAxisAlignment.stretch,
            children: [
              // 関係元の人物表示
              fromPersonAsync.when(
                data: (person) => person != null 
                    ? _buildStaticPersonDisplay('関係元', person) 
                    : const Text('関係元の人物情報を読み込めません'),
                loading: () => const Center(child: CircularProgressIndicator(strokeWidth: 2)),
                error: (e, st) => Text('エラー: $e'),
              ),
              const SizedBox(height: 16),
              // 関係タイプ選択
              _buildRelationTypeSelector(ref, formState.type),
              const SizedBox(height: 16),
              // 関係先人物選択
              availablePersonsAsync.when(
                data: (persons) => _buildPersonSelector(
                  context,
                  ref,
                  persons,
                  formState.toPersonId,
                ),
                loading: () => const Center(child: CircularProgressIndicator(strokeWidth: 2)),
                error: (e, st) => Text('関係先候補の読み込みエラー: $e'),
              ),
              const SizedBox(height: 16),
              // 関係性プレビュー
              if (fromPersonAsync.hasValue && fromPersonAsync.value != null && 
                  toPersonAsync.hasValue && toPersonAsync.value != null)
                _buildRelationshipPreview(
                  context,
                  fromPersonAsync.value!,
                  toPersonAsync.value!,
                  formState.type,
                ),
              // エラーメッセージ表示
              if (errorMessage != null)
                Padding(
                  padding: const EdgeInsets.only(top: 12.0),
                  child: Text(
                    errorMessage,
                    style: TextStyle(color: Theme.of(context).colorScheme.error),
                  ),
                ),
            ],
          ),
        ),
      ),
      actions: <Widget>[
        // 削除ボタン (編集時のみ)
        if (!isNew && widget.relationshipId != null)
          TextButton(
            style: TextButton.styleFrom(foregroundColor: Theme.of(context).colorScheme.error),
            onPressed: isSaving ? null : () => _confirmDelete(context, ref, widget.relationshipId!),
            child: const Text('関係削除'),
          ),
        const Spacer(),
        TextButton(
          onPressed: isSaving ? null : () => Navigator.of(context).pop(),
          child: const Text('キャンセル'),
        ),
        ElevatedButton(
          onPressed: (isSaving || formState.toPersonId == null) 
              ? null 
              : () => _saveRelationship(context, ref, isNew),
          child: isSaving
              ? const SizedBox(width: 20, height: 20, child: CircularProgressIndicator(strokeWidth: 2))
              : const Text('設定'),
        ),
      ],
    );
  }

  Widget _buildStaticPersonDisplay(String label, Person person) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text(label, style: Theme.of(context).textTheme.labelMedium?.copyWith(color: Colors.blueGrey.shade700)),
        Card(
          elevation: 1,
          margin: const EdgeInsets.only(top: 4),
          child: ListTile(
            leading: Icon(
              person.gender == Gender.male ? Icons.male : person.gender == Gender.female ? Icons.female : Icons.person,
              color: person.gender == Gender.male ? Colors.blue : person.gender == Gender.female ? Colors.pink : Colors.grey,
            ),
            title: Text(person.name, style: const TextStyle(fontWeight: FontWeight.bold)),
            subtitle: person.birthDate != null ? Text('${person.birthDate!.year}年生まれ') : null,
            dense: true,
          ),
        ),
      ],
    );
  }

  Widget _buildRelationTypeSelector(WidgetRef ref, RelationType currentType) {
    final notifier = ref.read(relationshipFormProvider.notifier);
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text('関係タイプ', style: Theme.of(context).textTheme.labelMedium?.copyWith(color: Colors.blueGrey.shade700)),
        RadioListTile<RelationType>(
          title: const Text('親子関係'),
          subtitle: const Text('「関係元」が「関係先」の親になります'),
          value: RelationType.parentChild,
          groupValue: currentType,
          onChanged: (value) => value != null ? notifier.updateType(value) : null,
          dense: true,
          contentPadding: EdgeInsets.zero,
        ),
        RadioListTile<RelationType>(
          title: const Text('配偶者関係'),
          subtitle: const Text('「関係元」と「関係先」が配偶者になります'),
          value: RelationType.spouse,
          groupValue: currentType,
          onChanged: (value) => value != null ? notifier.updateType(value) : null,
          dense: true,
          contentPadding: EdgeInsets.zero,
        ),
      ],
    );
  }

  Widget _buildPersonSelector(
    BuildContext context,
    WidgetRef ref,
    List<Person> availablePersons,
    String? selectedToPersonId,
  ) {
    final notifier = ref.read(relationshipFormProvider.notifier);
    if (availablePersons.isEmpty) {
      return const Text('関係を設定できる他の人物がいません。', style: TextStyle(color: Colors.orange));
    }
    return DropdownButtonFormField<String>(
      decoration: const InputDecoration(
        labelText: '関係先の人物',
        border: OutlineInputBorder(),
        contentPadding: EdgeInsets.symmetric(horizontal: 12, vertical: 12),
      ),
      value: selectedToPersonId,
      hint: const Text('人物を選択'),
      isExpanded: true,
      items: availablePersons.map((person) {
        return DropdownMenuItem<String>(
          value: person.id,
          child: Text(person.name),
        );
      }).toList(),
      onChanged: (value) => value != null ? notifier.updateToPersonId(value) : null,
      validator: (value) => value == null ? '関係先の人物を選択してください' : null,
    );
  }

  Widget _buildRelationshipPreview(
    BuildContext context,
    Person fromPerson,
    Person toPerson,
    RelationType type,
  ) {
    String description;
    if (type == RelationType.parentChild) {
      description = '${fromPerson.name} (親) → ${toPerson.name} (子)';
    } else {
      description = '${fromPerson.name} ⇔ ${toPerson.name} (配偶者)';
    }
    return Card(
      elevation: 0,
      color: Theme.of(context).colorScheme.surfaceVariant.withOpacity(0.5),
      child: Padding(
        padding: const EdgeInsets.all(12.0),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('プレビュー', style: Theme.of(context).textTheme.labelMedium),
            const SizedBox(height: 8),
            Text(description, style: Theme.of(context).textTheme.bodyMedium),
          ],
        ),
      ),
    );
  }

  Future<void> _saveRelationship(BuildContext context, WidgetRef ref, bool isNew) async {
    ref.read(relationshipFormErrorProvider.notifier).state = null;
    final formState = ref.read(relationshipFormProvider);
    final allRelationships = ref.read(relationshipsStreamProvider).value ?? [];
    
    final validationError = RelationshipValidators.validateRelationshipForm(formState, allRelationships);
    if (validationError != null) {
      ref.read(relationshipFormErrorProvider.notifier).state = validationError;
      return;
    }

    ref.read(relationshipSavingProvider.notifier).state = true;
    try {
      final controller = ref.read(relationshipControllerProvider);
      final relationshipToSave = formState.toRelationship();
      if (isNew) {
        await controller.addRelationship(relationshipToSave);
      } else {
        await controller.updateRelationship(relationshipToSave);
      }
      if (context.mounted) {
        Navigator.of(context).pop();
        RelationshipEditorErrorHandler.showSuccessSnackBar(context, isNew ? '関係性を追加しました' : '関係性を更新しました');
      }
    } catch (e) {
      if (context.mounted) {
        ref.read(relationshipFormErrorProvider.notifier).state = RelationshipEditorErrorHandler.getErrorMessage(e);
      }
    } finally {
      ref.read(relationshipSavingProvider.notifier).state = false;
    }
  }

  Future<void> _confirmDelete(BuildContext context, WidgetRef ref, String relationshipId) async {
    final bool? confirmed = await showDialog<bool>(
      context: context,
      builder: (BuildContext dialogContext) {
        return AlertDialog(
          title: const Text('関係性の削除'),
          content: const Text('この関係性を削除してもよろしいですか？'),
          actions: <Widget>[
            TextButton(
              child: const Text('キャンセル'),
              onPressed: () => Navigator.of(dialogContext).pop(false),
            ),
            TextButton(
              style: TextButton.styleFrom(foregroundColor: Theme.of(context).colorScheme.error),
              child: const Text('削除'),
              onPressed: () => Navigator.of(dialogContext).pop(true),
            ),
          ],
        );
      },
    );

    if (confirmed == true) {
      ref.read(relationshipSavingProvider.notifier).state = true;
      try {
        await ref.read(relationshipControllerProvider).deleteRelationship(relationshipId);
        if (context.mounted) {
          Navigator.of(context).pop(); // 関係性ダイアログを閉じる
          RelationshipEditorErrorHandler.showSuccessSnackBar(context, '関係性を削除しました');
        }
      } catch (e) {
        if (context.mounted) {
          ref.read(relationshipFormErrorProvider.notifier).state = RelationshipEditorErrorHandler.getErrorMessage(e);
        }
      } finally {
        ref.read(relationshipSavingProvider.notifier).state = false;
      }
    }
  }
}
```

## プロパティ

| 名前 | 型 | 説明 | デフォルト値 |
|------|------|------|------------|
| `fromPersonId` | `String` | 関係の起点となる人物のID | なし（必須） |
| `relationshipId` | `String?` | 編集対象の関係性のID。新規作成の場合は`null`。 | `null` |

## 内部状態

なし（状態はRiverpodプロバイダによって管理されます。`initState`でフォームの初期化を行っています。）

## メソッド

### initState

```dart
void initState()
```

ウィジェットの初期化時に呼び出されます。`relationshipId`の有無に応じて、`relationshipFormProvider`を新規作成用または編集用に初期化します。

### build

```dart
Widget build(BuildContext context)
```

ウィジェットのUIを構築します。`AlertDialog`内に、関係元の人物情報表示、関係タイプ選択（親子/配偶者）、関係先人物選択（ドロップダウン）、関係性プレビュー、操作ボタン（設定、キャンセル、関係削除）を表示します。

### _buildStaticPersonDisplay (プライベートヘルパー)

```dart
Widget _buildStaticPersonDisplay(String label, Person person)
```

関係元の人物情報を静的に表示するウィジェットを構築します。

### _buildRelationTypeSelector (プライベートヘルパー)

```dart
Widget _buildRelationTypeSelector(WidgetRef ref, RelationType currentType)
```

関係タイプ（親子関係/配偶者関係）を選択するためのラジオボタンリストを構築します。

### _buildPersonSelector (プライベートヘルパー)

```dart
Widget _buildPersonSelector(
  BuildContext context,
  WidgetRef ref,
  List<Person> availablePersons,
  String? selectedToPersonId,
)
```

関係を結ぶ相手の人物を選択するためのドロップダウンメニューを構築します。候補リストには、`fromPersonId`の人物自身は含まれません。

### _buildRelationshipPreview (プライベートヘルパー)

```dart
Widget _buildRelationshipPreview(
  BuildContext context,
  Person fromPerson,
  Person toPerson,
  RelationType type,
)
```

設定しようとしている関係性を視覚的にプレビュー表示するウィジェットを構築します。

### _saveRelationship (プライベートヘルパー)

```dart
Future<void> _saveRelationship(BuildContext context, WidgetRef ref, bool isNew) async
```

入力された関係性情報を保存（追加または更新）します。フォームのバリデーションを行い、問題がなければ`RelationshipController`を通じてデータを永続化します。

### _confirmDelete (プライベートヘルパー)

```dart
Future<void> _confirmDelete(BuildContext context, WidgetRef ref, String relationshipId) async
```

関係性を削除する前に確認ダイアログを表示し、ユーザーの確認が得られた場合に`RelationshipController`を通じて関係性を削除します。

## 使用例

`RelationshipDialog`は、人物詳細パネルの「関係追加」ボタンや、既存の関係性を編集するアクションから呼び出されます。

```dart
// 関係性追加ダイアログの表示
void _showAddRelationshipDialog(BuildContext context, WidgetRef ref, String fromPersonId) {
  showDialog(
    context: context,
    builder: (_) => ResponsiveRelationshipDialog( // レスポンシブ対応ラッパー
      child: RelationshipDialog(fromPersonId: fromPersonId),
    ),
  );
}

// 関係性編集ダイアログの表示
void _showEditRelationshipDialog(BuildContext context, WidgetRef ref, String fromPersonId, String relationshipId) {
  showDialog(
    context: context,
    builder: (_) => ResponsiveRelationshipDialog(
      child: RelationshipDialog(fromPersonId: fromPersonId, relationshipId: relationshipId),
    ),
  );
}
```

## 注意事項

- このウィジェットは、`relationshipFormProvider`を通じてフォームの状態を管理し、`RelationshipController`を通じて実際のデータ操作を行います。
- 関係先の人物選択ドロップダウンには、`availablePersonsProvider`から取得した、関係元の人物自身を除いた人物リストが表示されます。
- バリデーションは、`RelationshipValidators`クラス（別途定義想定）によって行われます。特に、自己参照関係や循環する親子関係のチェックが重要です。
- 保存処理や削除処理中は、ボタンを無効化し、`CircularProgressIndicator`を表示してユーザーに処理中であることを伝えます。
- エラーメッセージは`relationshipFormErrorProvider`を通じて表示されます。
- `ResponsiveRelationshipDialog`は、このダイアログをラップして画面サイズに応じた表示調整を行うウィジェットです（別途定義想定）。

## 関連するクラス・プロバイダ

- [Relationship](../モデル/Relationship.md) - 関係性データのモデルクラス
- [Person](../モデル/Person.md) - 人物データのモデルクラス
- `relationshipFormProvider` (Riverpodプロバイダ) - 関係性編集フォームの状態を管理
- `relationshipSavingProvider` (Riverpodプロバイダ) - 保存処理中かどうかを示す状態を管理
- `relationshipFormErrorProvider` (Riverpodプロバイダ) - フォームのエラーメッセージを管理
- `personByIdProvider` (Riverpodプロバイダ) - 指定されたIDの人物情報を取得
- `availablePersonsProvider` (Riverpodプロバイダ) - 関係設定可能な人物リストを取得
- [RelationshipController](../コントローラ/RelationshipController.md) - 関係性データの追加・更新・削除を行うコントローラ
- `RelationshipValidators` (ユーティリティクラス) - 関係性フォームのバリデーションロジックを提供 (別途定義想定)
- `RelationshipEditorErrorHandler` (ユーティリティクラス) - エラーメッセージの整形や表示を行う (別途定義想定)
- `ResponsiveRelationshipDialog` (ウィジェット) - このダイアログのレスポンシブ対応ラッパー (別途定義想定)
