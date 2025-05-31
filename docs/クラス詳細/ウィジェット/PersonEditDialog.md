# PersonEditDialog ウィジェット

## 概要

`PersonEditDialog` ウィジェットは、人物の情報を追加または編集するためのモーダルダイアログUIコンポーネントです。氏名、旧姓、生年月日、没年月日、性別、メモなどの入力フィールドを提供し、ユーザーがこれらの情報を入力・変更できるようにします。新規追加と既存情報の編集の両方のシナリオに対応します。

## クラス定義

```dart
// lib/features/person_editor/presentation/dialogs/person_edit_dialog.dart

class PersonEditDialog extends ConsumerWidget {
  final bool isNew; // 新規追加の場合はtrue、編集の場合はfalse

  const PersonEditDialog({
    Key? key,
    required this.isNew,
  }) : super(key: key);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final formKey = GlobalKey<FormState>();
    final formState = ref.watch(personFormProvider);
    final formNotifier = ref.read(personFormProvider.notifier);
    final isSaving = ref.watch(personSavingProvider);
    final errorMessage = ref.watch(personFormErrorProvider);

    final title = isNew ? '人物の追加' : '人物情報の編集';

    return AlertDialog(
      title: Text(title),
      content: SingleChildScrollView(
        child: ConstrainedBox(
          constraints: BoxConstraints(
            maxWidth: 500, // ダイアログの最大幅
          ),
          child: Form(
            key: formKey,
            child: Column(
              mainAxisSize: MainAxisSize.min,
              crossAxisAlignment: CrossAxisAlignment.stretch,
              children: [
                // 氏名
                TextFormField(
                  initialValue: formState.name,
                  decoration: const InputDecoration(labelText: '氏名', hintText: '山田 太郎'),
                  validator: PersonValidators.validateName,
                  onChanged: formNotifier.updateName,
                  textInputAction: TextInputAction.next,
                ),
                const SizedBox(height: 16),
                // 旧姓
                TextFormField(
                  initialValue: formState.maidenName,
                  decoration: const InputDecoration(labelText: '旧姓', hintText: '佐藤'),
                  validator: PersonValidators.validateMaidenName,
                  onChanged: formNotifier.updateMaidenName,
                  textInputAction: TextInputAction.next,
                ),
                const SizedBox(height: 16),
                // 生年月日
                _buildDateField(
                  context: context,
                  label: '生年月日',
                  initialDate: formState.birthDate,
                  onDateSelected: formNotifier.updateBirthDate,
                  validator: (date) => PersonValidators.validateDates(date, formState.deathDate),
                ),
                const SizedBox(height: 16),
                // 没年月日
                _buildDateField(
                  context: context,
                  label: '没年月日',
                  initialDate: formState.deathDate,
                  onDateSelected: formNotifier.updateDeathDate,
                  validator: (date) => PersonValidators.validateDates(formState.birthDate, date),
                ),
                const SizedBox(height: 16),
                // 性別
                _buildGenderField(ref, formState.gender),
                const SizedBox(height: 16),
                // メモ
                TextFormField(
                  initialValue: formState.memo,
                  decoration: const InputDecoration(
                    labelText: 'メモ',
                    hintText: 'この人物に関する備考情報 (例: 趣味、職業など)',
                    alignLabelWithHint: true,
                  ),
                  maxLines: 3,
                  validator: PersonValidators.validateMemo,
                  onChanged: formNotifier.updateMemo,
                  textInputAction: TextInputAction.done,
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
      ),
      actions: <Widget>[
        // 削除ボタン (編集時のみ)
        if (!isNew && formState.id != null)
          TextButton(
            style: TextButton.styleFrom(foregroundColor: Theme.of(context).colorScheme.error),
            onPressed: isSaving ? null : () => _confirmDelete(context, ref, formState.id!),
            child: const Text('削除'),
          ),
        const Spacer(),
        // キャンセルボタン
        TextButton(
          onPressed: isSaving ? null : () => Navigator.of(context).pop(),
          child: const Text('キャンセル'),
        ),
        // 保存ボタン
        ElevatedButton(
          onPressed: isSaving ? null : () => _savePerson(context, ref, formKey, isNew),
          child: isSaving
              ? const SizedBox(width: 20, height: 20, child: CircularProgressIndicator(strokeWidth: 2))
              : const Text('保存'),
        ),
      ],
    );
  }

  // 日付選択フィールド構築ヘルパー
  Widget _buildDateField({
    required BuildContext context,
    required String label,
    required DateTime? initialDate,
    required ValueChanged<DateTime?> onDateSelected,
    FormFieldValidator<DateTime>? validator,
  }) {
    final dateFormat = DateFormat('yyyy年MM月dd日');
    return FormField<DateTime>(
      initialValue: initialDate,
      validator: validator,
      builder: (FormFieldState<DateTime> field) {
        return InkWell(
          onTap: () async {
            final DateTime? picked = await showDatePicker(
              context: context,
              initialDate: field.value ?? DateTime.now(),
              firstDate: DateTime(1800),
              lastDate: DateTime.now().add(const Duration(days: 365)), // 未来日も許容する場合
              locale: const Locale('ja'),
            );
            if (picked != null && picked != field.value) {
              field.didChange(picked);
              onDateSelected(picked);
            }
          },
          child: InputDecorator(
            decoration: InputDecoration(
              labelText: label,
              border: const OutlineInputBorder(),
              errorText: field.errorText,
              suffixIcon: field.value != null 
                ? IconButton(
                    icon: const Icon(Icons.clear, size: 18),
                    onPressed: () {
                      field.didChange(null);
                      onDateSelected(null);
                    },
                  )
                : const Icon(Icons.calendar_today),
            ),
            child: Text(
              field.value != null ? dateFormat.format(field.value!) : '日付を選択',
              style: TextStyle(
                color: field.value != null ? null : Theme.of(context).hintColor,
              ),
            ),
          ),
        );
      },
    );
  }

  // 性別選択フィールド構築ヘルパー
  Widget _buildGenderField(WidgetRef ref, Gender? currentGender) {
    final formNotifier = ref.read(personFormProvider.notifier);
    return InputDecorator(
      decoration: const InputDecoration(
        labelText: '性別',
        border: OutlineInputBorder(),
        contentPadding: EdgeInsets.symmetric(horizontal: 12, vertical: 8),
      ),
      child: DropdownButtonHideUnderline(
        child: DropdownButton<Gender?>(
          value: currentGender,
          isExpanded: true,
          hint: const Text('性別を選択'),
          items: [
            const DropdownMenuItem<Gender?>(
              value: null,
              child: Text('未設定'),
            ),
            ...Gender.values.map((Gender gender) {
              return DropdownMenuItem<Gender?>(
                value: gender,
                child: Text(gender.displayName),
              );
            }).toList(),
          ],
          onChanged: (Gender? newValue) {
            formNotifier.updateGender(newValue);
          },
        ),
      ),
    );
  }

  // 保存処理
  Future<void> _savePerson(BuildContext context, WidgetRef ref, GlobalKey<FormState> formKey, bool isNew) async {
    ref.read(personFormErrorProvider.notifier).state = null; // 前のエラーをクリア
    if (formKey.currentState?.validate() ?? false) {
      final formState = ref.read(personFormProvider);
      final validationError = PersonValidators.validatePersonForm(formState);
      if (validationError != null) {
        ref.read(personFormErrorProvider.notifier).state = validationError;
        return;
      }

      ref.read(personSavingProvider.notifier).state = true;
      try {
        final controller = ref.read(personControllerProvider);
        final personToSave = formState.toPerson();
        if (isNew) {
          await controller.addPerson(personToSave);
        } else {
          await controller.updatePerson(personToSave);
        }
        if (context.mounted) {
          Navigator.of(context).pop();
          PersonEditorErrorHandler.showSuccessSnackBar(context, isNew ? '人物を追加しました' : '人物情報を更新しました');
        }
      } catch (e) {
        if (context.mounted) {
          ref.read(personFormErrorProvider.notifier).state = PersonEditorErrorHandler.getErrorMessage(e);
        }
      } finally {
        ref.read(personSavingProvider.notifier).state = false;
      }
    }
  }

  // 削除確認処理
  Future<void> _confirmDelete(BuildContext context, WidgetRef ref, String personId) async {
    final bool? confirmed = await showDialog<bool>(
      context: context,
      builder: (BuildContext dialogContext) {
        return AlertDialog(
          title: const Text('人物の削除'),
          content: const Text('この人物を削除してもよろしいですか？\n関連する全ての関係性も削除されます。'),
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
      ref.read(personSavingProvider.notifier).state = true;
      try {
        await ref.read(personControllerProvider).deletePerson(personId);
        if (context.mounted) {
          Navigator.of(context).pop(); // 編集ダイアログを閉じる
          PersonEditorErrorHandler.showSuccessSnackBar(context, '人物を削除しました');
        }
      } catch (e) {
        if (context.mounted) {
          ref.read(personFormErrorProvider.notifier).state = PersonEditorErrorHandler.getErrorMessage(e);
        }
      } finally {
        ref.read(personSavingProvider.notifier).state = false;
      }
    }
  }
}
```

## プロパティ

| 名前 | 型 | 説明 | デフォルト値 |
|------|------|------|------------|
| `isNew` | `bool` | 新規追加の場合は`true`、既存の人物を編集する場合は`false` | なし（必須） |

## 内部状態

なし（状態はRiverpodプロバイダによって管理されます）

## メソッド

### build

```dart
Widget build(BuildContext context, WidgetRef ref)
```

ウィジェットのUIを構築します。`AlertDialog`内に`Form`ウィジェットを配置し、各種入力フィールド（氏名、旧姓、生年月日、没年月日、性別、メモ）と操作ボタン（保存、キャンセル、削除）を表示します。

### _buildDateField (プライベートヘルパー)

```dart
Widget _buildDateField({
  required BuildContext context,
  required String label,
  required DateTime? initialDate,
  required ValueChanged<DateTime?> onDateSelected,
  FormFieldValidator<DateTime>? validator,
})
```

日付選択フィールド（生年月日、没年月日）を構築します。タップするとネイティブの日付選択ダイアログが表示されます。

### _buildGenderField (プライベートヘルパー)

```dart
Widget _buildGenderField(WidgetRef ref, Gender? currentGender)
```

性別選択フィールド（ドロップダウン）を構築します。

### _savePerson (プライベートヘルパー)

```dart
Future<void> _savePerson(BuildContext context, WidgetRef ref, GlobalKey<FormState> formKey, bool isNew) async
```

入力された人物情報を保存（追加または更新）します。フォームのバリデーションを行い、問題がなければ`PersonController`を通じてデータを永続化します。

### _confirmDelete (プライベートヘルパー)

```dart
Future<void> _confirmDelete(BuildContext context, WidgetRef ref, String personId) async
```

人物を削除する前に確認ダイアログを表示し、ユーザーの確認が得られた場合に`PersonController`を通じて人物を削除します。

## 使用例

`PersonEditDialog`は、メイン画面のフローティングアクションボタンや人物詳細パネルの編集ボタンから呼び出されます。

```dart
// 人物追加ダイアログの表示
void _showPersonAddDialog(BuildContext context, WidgetRef ref) {
  ref.read(personFormProvider.notifier).initWithPerson(null); // フォームを初期化
  showDialog(
    context: context,
    builder: (_) => ResponsivePersonEditDialog( // レスポンシブ対応ラッパー
      child: PersonEditDialog(isNew: true),
    ),
  );
}

// 人物編集ダイアログの表示
void _showPersonEditDialog(BuildContext context, WidgetRef ref, Person personToEdit) {
  ref.read(personFormProvider.notifier).initWithPerson(personToEdit); // 既存データでフォームを初期化
  showDialog(
    context: context,
    builder: (_) => ResponsivePersonEditDialog(
      child: PersonEditDialog(isNew: false),
    ),
  );
}
```

## 注意事項

- このウィジェットは、`personFormProvider`を通じてフォームの状態を管理し、`PersonController`を通じて実際のデータ操作を行います。
- 入力バリデーションは、各`TextFormField`の`validator`プロパティと、`PersonValidators`クラス（別途定義想定）によって行われます。
- 日付選択には`showDatePicker`を使用し、日本語ロケールを設定しています。
- 性別選択は`DropdownButtonFormField`を使用しています。`Gender`列挙型には`displayName`拡張メソッドが定義されていることを前提とします。
- 保存処理や削除処理中は、ボタンを無効化し、`CircularProgressIndicator`を表示してユーザーに処理中であることを伝えます。
- エラーメッセージは`personFormErrorProvider`を通じて表示されます。
- `ResponsivePersonEditDialog`は、このダイアログをラップして画面サイズに応じた表示調整を行うウィジェットです（別途定義想定）。

## 関連するクラス・プロバイダ

- [Person](../モデル/Person.md) - 人物データのモデルクラス
- [Gender](../モデル/Person.md) - 性別を表す列挙型
- `personFormProvider` (Riverpodプロバイダ) - 人物編集フォームの状態を管理
- `personSavingProvider` (Riverpodプロバイダ) - 保存処理中かどうかを示す状態を管理
- `personFormErrorProvider` (Riverpodプロバイダ) - フォームのエラーメッセージを管理
- [PersonController](../コントローラ/PersonController.md) - 人物データの追加・更新・削除を行うコントローラ
- `PersonValidators` (ユーティリティクラス) - 人物フォームのバリデーションロジックを提供 (別途定義想定)
- `PersonEditorErrorHandler` (ユーティリティクラス) - エラーメッセージの整形や表示を行う (別途定義想定)
- `ResponsivePersonEditDialog` (ウィジェット) - このダイアログのレスポンシブ対応ラッパー (別途定義想定)
