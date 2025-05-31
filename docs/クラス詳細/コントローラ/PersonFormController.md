# PersonFormController クラス (`StateNotifier`)

## 概要

`PersonFormController`（実際には `PersonFormNotifier` という名前の `StateNotifier`）は、人物情報を追加または編集する際のフォームの状態 (`PersonFormState`) を管理します。ユーザーが入力した氏名、旧姓、生年月日などの値を保持し、フォームの初期化、各フィールドの更新、およびフォームデータから `Person` モデルへの変換機能を提供します。Riverpodプロバイダを通じてUIウィジェット（例: `PersonEditDialog`）から利用されます。

## クラス定義 (`PersonFormState` と `PersonFormNotifier`)

```dart
// lib/features/person_editor/application/person_form_controller.dart

// 人物編集フォームの状態を表すイミュータブルなクラス
@freezed
class PersonFormState with _$PersonFormState {
  const factory PersonFormState({
    String? id, // 編集対象の人物ID (新規作成時はnull)
    @Default('') String name,
    String? maidenName,
    DateTime? birthDate,
    DateTime? deathDate,
    Gender? gender,
    String? memo,
    // バリデーションエラーメッセージ (UI表示用)
    String? nameError,
    String? maidenNameError,
    String? birthDateError,
    String? deathDateError,
    String? datesError, // 生年月日と没年月日の整合性エラー
    String? memoError,
    // フォーム全体の送信可否 (バリデーション結果に基づく)
    @Default(false) bool canSubmit,
  }) = _PersonFormState;

  // Personモデルからフォーム状態を生成するファクトリコンストラクタ
  factory PersonFormState.fromPerson(Person person) {
    return PersonFormState(
      id: person.id,
      name: person.name,
      maidenName: person.maidenName,
      birthDate: person.birthDate,
      deathDate: person.deathDate,
      gender: person.gender,
      memo: person.memo,
    );
  }
}

// PersonFormStateを管理するStateNotifier
class PersonFormNotifier extends StateNotifier<PersonFormState> {
  PersonFormNotifier() : super(const PersonFormState());

  // フォームを初期化 (新規作成時)
  void initializeForCreate() {
    state = const PersonFormState();
    _validateForm(); // 初期バリデーション
  }

  // 既存のPersonデータでフォームを初期化 (編集時)
  void initializeForEdit(Person person) {
    state = PersonFormState.fromPerson(person);
    _validateForm(); // 初期バリデーション
  }

  // 各フィールドの更新メソッド
  void updateName(String name) {
    state = state.copyWith(name: name, nameError: PersonValidators.validateName(name));
    _validateForm();
  }

  void updateMaidenName(String? maidenName) {
    state = state.copyWith(maidenName: maidenName, maidenNameError: PersonValidators.validateMaidenName(maidenName));
    _validateForm();
  }

  void updateBirthDate(DateTime? birthDate) {
    state = state.copyWith(
      birthDate: birthDate, 
      birthDateError: PersonValidators.validateBirthDate(birthDate),
      datesError: PersonValidators.validateDates(birthDate, state.deathDate),
    );
    _validateForm();
  }

  void updateDeathDate(DateTime? deathDate) {
    state = state.copyWith(
      deathDate: deathDate, 
      deathDateError: PersonValidators.validateDeathDate(deathDate),
      datesError: PersonValidators.validateDates(state.birthDate, deathDate),
    );
    _validateForm();
  }

  void updateGender(Gender? gender) {
    state = state.copyWith(gender: gender);
    // 性別は通常バリデーション不要だが、必要なら追加
    _validateForm();
  }

  void updateMemo(String? memo) {
    state = state.copyWith(memo: memo, memoError: PersonValidators.validateMemo(memo));
    _validateForm();
  }

  // フォーム全体のバリデーションを行い、送信可否を更新
  void _validateForm() {
    final nameValid = PersonValidators.validateName(state.name) == null;
    final maidenNameValid = PersonValidators.validateMaidenName(state.maidenName) == null;
    final birthDateValid = PersonValidators.validateBirthDate(state.birthDate) == null;
    final deathDateValid = PersonValidators.validateDeathDate(state.deathDate) == null;
    final datesValid = PersonValidators.validateDates(state.birthDate, state.deathDate) == null;
    final memoValid = PersonValidators.validateMemo(state.memo) == null;

    state = state.copyWith(
      canSubmit: nameValid && maidenNameValid && birthDateValid && deathDateValid && datesValid && memoValid,
      // エラーメッセージも更新
      nameError: PersonValidators.validateName(state.name),
      maidenNameError: PersonValidators.validateMaidenName(state.maidenName),
      birthDateError: PersonValidators.validateBirthDate(state.birthDate),
      deathDateError: PersonValidators.validateDeathDate(state.deathDate),
      datesError: PersonValidators.validateDates(state.birthDate, state.deathDate),
      memoError: PersonValidators.validateMemo(state.memo),
    );
  }
  
  // フォームデータからPersonモデルを生成 (送信時用)
  Person? toPerson() {
    if (!state.canSubmit) {
      // バリデーションエラーがある場合はnullを返すか例外をスロー
      return null; 
    }
    return Person(
      id: state.id ?? '', // 新規の場合は空文字列
      name: state.name,
      maidenName: state.maidenName?.isEmpty ?? true ? null : state.maidenName,
      birthDate: state.birthDate,
      deathDate: state.deathDate,
      gender: state.gender,
      memo: state.memo?.isEmpty ?? true ? null : state.memo,
      // x, y座標はここでは設定しない (レイアウトエンジンが担当)
    );
  }
  
  // フォームをリセット (オプション)
  void resetForm() {
    state = const PersonFormState();
  }
}
```

## 関連するプロバイダ

```dart
// lib/features/person_editor/application/person_form_providers.dart (または適切な場所)

// PersonFormNotifierのプロバイダ
final personFormProvider = StateNotifierProvider.autoDispose<PersonFormNotifier, PersonFormState>((ref) {
  return PersonFormNotifier();
});

// フォームの送信処理が実行中かどうかを示すプロバイダ
final personFormSavingProvider = StateProvider.autoDispose<bool>((ref) => false);

// フォーム全体のバリデーションエラーメッセージ (主にサーバーサイドエラーや複雑なエラー用)
final personFormOverallErrorProvider = StateProvider.autoDispose<String?>((ref) => null);
```

## `PersonFormState` プロパティ

| 名前 | 型 | 説明 | デフォルト値 |
|------|------|------|------------|
| `id` | `String?` | 編集対象の人物ID。新規作成時は`null`。 | `null` |
| `name` | `String` | 氏名 | `''` |
| `maidenName` | `String?` | 旧姓 | `null` |
| `birthDate` | `DateTime?` | 生年月日 | `null` |
| `deathDate` | `DateTime?` | 没年月日 | `null` |
| `gender` | `Gender?` | 性別 | `null` |
| `memo` | `String?` | メモ | `null` |
| `nameError` | `String?` | 氏名フィールドのバリデーションエラーメッセージ | `null` |
| `maidenNameError` | `String?` | 旧姓フィールドのバリデーションエラーメッセージ | `null` |
| `birthDateError` | `String?` | 生年月日フィールドのバリデーションエラーメッセージ | `null` |
| `deathDateError` | `String?` | 没年月日フィールドのバリデーションエラーメッセージ | `null` |
| `datesError` | `String?` | 生年月日と没年月日の整合性エラーメッセージ | `null` |
| `memoError` | `String?` | メモフィールドのバリデーションエラーメッセージ | `null` |
| `canSubmit` | `bool` | フォームが送信可能か（全バリデーションが通っているか） | `false` |

## `PersonFormNotifier` メソッド

### initializeForCreate

```dart
void initializeForCreate()
```

新規に人物を追加するためにフォームを初期状態にリセットし、初期バリデーションを実行します。

### initializeForEdit

```dart
void initializeForEdit(Person person)
```

既存の`Person`オブジェクトの情報でフォームを初期化し、初期バリデーションを実行します。

**パラメータ:**
- `person`: `Person` - 編集対象の人物データ

### updateName / updateMaidenName / updateBirthDate / updateDeathDate / updateGender / updateMemo

```dart
void updateName(String name)
// 他のフィールドも同様
```

各フォームフィールドの値が変更されたときに呼び出され、対応する`PersonFormState`のプロパティを更新し、関連するバリデーションを実行して`canSubmit`状態を更新します。

### toPerson

```dart
Person? toPerson()
```

現在のフォーム状態 (`PersonFormState`) から`Person`モデルオブジェクトを生成します。フォームのバリデーション (`canSubmit`) が通っていない場合は`null`を返すか、例外をスローすることを検討します。

**戻り値:**
- `Person?` - 生成された`Person`オブジェクト、またはバリデーションエラー時は`null`

### resetForm (オプション)

```dart
void resetForm()
```

フォームの状態を完全に初期状態に戻します。

## 使用例

```dart
// PersonEditDialogウィジェット内での使用

// フォームの初期化 (ダイアログ表示時)
// isNewがtrueなら新規作成、falseなら編集
if (isNew) {
  ref.read(personFormProvider.notifier).initializeForCreate();
} else {
  final personToEdit = ref.read(personBeingEditedProvider); // 例: 編集対象の人物を取得
  if (personToEdit != null) {
    ref.read(personFormProvider.notifier).initializeForEdit(personToEdit);
  }
}

// TextFormFieldの例
TextFormField(
  initialValue: ref.watch(personFormProvider).name,
  decoration: InputDecoration(
    labelText: '氏名',
    errorText: ref.watch(personFormProvider).nameError,
  ),
  onChanged: (value) => ref.read(personFormProvider.notifier).updateName(value),
  validator: (_) => ref.read(personFormProvider).nameError, // State内のエラーを直接利用
)

// 保存ボタンの処理
ElevatedButton(
  onPressed: ref.watch(personFormProvider).canSubmit 
    ? () async {
        final person = ref.read(personFormProvider.notifier).toPerson();
        if (person != null) {
          ref.read(personFormSavingProvider.notifier).state = true;
          try {
            if (isNew) {
              await ref.read(personControllerProvider).addPerson(person);
            } else {
              await ref.read(personControllerProvider).updatePerson(person);
            }
            // 成功処理
          } catch (e) {
            // エラー処理
            ref.read(personFormOverallErrorProvider.notifier).state = e.toString();
          } finally {
            ref.read(personFormSavingProvider.notifier).state = false;
          }
        }
      }
    : null, // canSubmitがfalseなら無効化
  child: Text('保存'),
)
```

## 注意事項

- `PersonFormState`は不変オブジェクトとして`freezed`で定義されています。状態の更新は`copyWith`を通じて行われます。
- 各フィールドの更新メソッド内で、対応するフィールドのバリデーションとフォーム全体のバリデーション (`_validateForm`) が実行され、`canSubmit`フラグが更新されます。
- バリデーションロジックは`PersonValidators`クラス（別途定義想定）に集約し、各更新メソッドや`_validateForm`から呼び出します。
- `toPerson`メソッドは、フォームデータを`Person`モデルに変換する際に、空文字列のフィールドを`null`に変換するなどの整形処理も行います。
- `autoDispose`をプロバイダに付与することで、フォームが不要になった際に自動的に状態が破棄されるようにしています。
- `personFormSavingProvider`は保存処理中のUI制御（例：ボタンの無効化、ローディング表示）に使用します。
- `personFormOverallErrorProvider`は、個別のフィールドエラーではなく、フォーム送信時のサーバーエラーなど、フォーム全体に関わるエラーメッセージを表示するために使用します。

## 関連するクラス・プロバイダ

- [Person](../モデル/Person.md) - 人物データのモデルクラス
- `PersonValidators` (ユーティリティクラス) - 人物フォームのバリデーションロジックを提供 (別途定義想定)
- [PersonController](./PersonController.md) - 人物データの追加・更新・削除を行うコントローラ
- [PersonEditDialog](../ウィジェット/PersonEditDialog.md) - このフォームコントローラを利用するUIウィジェット
