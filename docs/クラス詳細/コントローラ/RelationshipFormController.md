# RelationshipFormController クラス (`StateNotifier`)

## 概要

`RelationshipFormController`（実際には `RelationshipFormNotifier` という名前の `StateNotifier`）は、人物間の関係性（親子関係や配偶者関係）を設定または編集する際のフォームの状態 (`RelationshipFormState`) を管理します。関係の起点となる人物ID、関係タイプ、関係を結ぶ相手の人物IDを保持し、フォームの初期化、各フィールドの更新、およびフォームデータから `Relationship` モデルへの変換機能を提供します。Riverpodプロバイダを通じてUIウィジェット（例: `RelationshipDialog`）から利用されます。

## クラス定義 (`RelationshipFormState` と `RelationshipFormNotifier`)

```dart
// lib/features/relationship_editor/application/relationship_form_controller.dart

// 関係性編集フォームの状態を表すイミュータブルなクラス
@freezed
class RelationshipFormState with _$RelationshipFormState {
  const factory RelationshipFormState({
    String? id, // 編集対象の関係性ID (新規作成時はnull)
    required String fromPersonId, // 関係の起点となる人物ID
    String? toPersonId, // 関係を結ぶ相手の人物ID
    @Default(RelationType.parentChild) RelationType type, // 関係タイプ
    // バリデーションエラーメッセージ (UI表示用)
    String? toPersonIdError,
    String? circularReferenceError,
    String? duplicateRelationshipError,
    // フォーム全体の送信可否 (バリデーション結果に基づく)
    @Default(false) bool canSubmit,
  }) = _RelationshipFormState;

  // Relationshipモデルからフォーム状態を生成するファクトリコンストラクタ
  factory RelationshipFormState.fromRelationship(Relationship relationship) {
    return RelationshipFormState(
      id: relationship.id,
      fromPersonId: relationship.fromPersonId,
      toPersonId: relationship.toPersonId,
      type: relationship.type,
    );
  }
}

// RelationshipFormStateを管理するStateNotifier
class RelationshipFormNotifier extends StateNotifier<RelationshipFormState> {
  final Ref _ref; // バリデーションのために他のプロバイダを読み取るために必要

  RelationshipFormNotifier(this._ref, String initialFromPersonId)
      : super(RelationshipFormState(fromPersonId: initialFromPersonId)) {
    _validateForm(); // 初期バリデーション
  }

  // 新規作成用にフォームを初期化
  void initializeForCreate(String fromPersonId) {
    state = RelationshipFormState(fromPersonId: fromPersonId);
    _validateForm();
  }

  // 既存の関係性データでフォームを初期化 (編集時)
  void initializeForEdit(Relationship relationship) {
    state = RelationshipFormState.fromRelationship(relationship);
    _validateForm();
  }

  // 関係先の人物IDを更新
  void updateToPersonId(String? toPersonId) {
    state = state.copyWith(toPersonId: toPersonId);
    _validateForm();
  }

  // 関係タイプを更新
  void updateType(RelationType type) {
    state = state.copyWith(type: type);
    _validateForm();
  }

  // フォーム全体のバリデーションを行い、送信可否とエラーメッセージを更新
  Future<void> _validateForm() async {
    // 既存の関係性リストを取得 (循環参照チェックと重複チェックのため)
    final allRelationships = _ref.read(relationshipsStreamProvider).value ?? [];

    final toPersonIdError = RelationshipValidators.validateToPersonId(state.toPersonId);
    final selfReferenceError = RelationshipValidators.validateSelfReference(state.fromPersonId, state.toPersonId);
    final circularError = RelationshipValidators.validateCircularReference(
      state.fromPersonId,
      state.toPersonId,
      state.type,
      allRelationships,
    );
    final duplicateError = await RelationshipValidators.validateDuplicateRelationship(
      state.toRelationship(), // 現在のフォーム状態から一時的なRelationshipを作成
      allRelationships,
      excludeId: state.id, // 編集中の場合は自身を除外
    );
    
    String? combinedError;
    if (toPersonIdError != null) combinedError = toPersonIdError;
    else if (selfReferenceError != null) combinedError = selfReferenceError;
    else if (circularError != null) combinedError = circularError;
    else if (duplicateError != null) combinedError = duplicateError;

    state = state.copyWith(
      toPersonIdError: toPersonIdError, // 個別のエラーも保持するなら
      circularReferenceError: circularError,
      duplicateRelationshipError: duplicateError,
      canSubmit: combinedError == null && state.toPersonId != null, // toPersonIdがnullでないことも条件
    );
    
    // UI表示用の全体エラーメッセージ (オプション)
    // _ref.read(relationshipFormOverallErrorProvider.notifier).state = combinedError;
  }

  // フォームデータからRelationshipモデルを生成 (送信時用)
  Relationship? toRelationship() {
    // _validateFormを呼び出して最新のcanSubmit状態を保証
    // ただし、_validateFormが非同期になったため、UI側でcanSubmitを監視する形が良い
    if (!state.canSubmit || state.toPersonId == null) {
      return null;
    }
    return Relationship(
      id: state.id ?? '', // 新規の場合は空文字列
      fromPersonId: state.fromPersonId,
      toPersonId: state.toPersonId!, // canSubmitがtrueならnullでないはず
      type: state.type,
    );
  }
  
  // フォームをリセット (オプション)
  void resetForm(String fromPersonId) {
    state = RelationshipFormState(fromPersonId: fromPersonId);
    _validateForm();
  }
}
```

## 関連するプロバイダ

```dart
// lib/features/relationship_editor/application/relationship_form_providers.dart (または適切な場所)

// RelationshipFormNotifierのプロバイダ
// fromPersonIdを引数に取るため、family修飾子を使用するか、
// ダイアログ生成時にProviderをoverrideして初期値を渡すなどの工夫が必要。
// ここでは、ダイアログのinitStateで初期化することを前提とする。
final relationshipFormProvider = 
  StateNotifierProvider.autoDispose<RelationshipFormNotifier, RelationshipFormState>((ref) {
    // この初期値は実際にはダイアログの引数から渡されるべき
    // throw UnimplementedError('relationshipFormProvider must be initialized with fromPersonId');
    // ダミーの初期化。実際の初期化はNotifierのメソッドで行う。
    return RelationshipFormNotifier(ref, ''); 
  });

// フォームの送信処理が実行中かどうかを示すプロバイダ
final relationshipFormSavingProvider = StateProvider.autoDispose<bool>((ref) => false);

// フォーム全体のバリデーションエラーメッセージ (主にサーバーサイドエラーや複雑なエラー用)
final relationshipFormOverallErrorProvider = StateProvider.autoDispose<String?>((ref) => null);
```

## `RelationshipFormState` プロパティ

| 名前 | 型 | 説明 | デフォルト値 |
|------|------|------|------------|
| `id` | `String?` | 編集対象の関係性ID。新規作成時は`null`。 | `null` |
| `fromPersonId` | `String` | 関係の起点となる人物ID。 | なし（必須、コンストラクタで初期化） |
| `toPersonId` | `String?` | 関係を結ぶ相手の人物ID。 | `null` |
| `type` | `RelationType` | 関係タイプ（親子/配偶者）。 | `RelationType.parentChild` |
| `toPersonIdError` | `String?` | 関係先人物選択のバリデーションエラー。 | `null` |
| `circularReferenceError` | `String?` | 循環参照エラーメッセージ。 | `null` |
| `duplicateRelationshipError` | `String?` | 重複関係エラーメッセージ。 | `null` |
| `canSubmit` | `bool` | フォームが送信可能か（全バリデーションが通っているか）。 | `false` |

## `RelationshipFormNotifier` メソッド

### initializeForCreate

```dart
void initializeForCreate(String fromPersonId)
```

新規に関係性を追加するためにフォームを初期状態にリセットし、初期バリデーションを実行します。

**パラメータ:**
- `fromPersonId`: `String` - 関係の起点となる人物のID

### initializeForEdit

```dart
void initializeForEdit(Relationship relationship)
```

既存の`Relationship`オブジェクトの情報でフォームを初期化し、初期バリデーションを実行します。

**パラメータ:**
- `relationship`: `Relationship` - 編集対象の関係性データ

### updateToPersonId

```dart
void updateToPersonId(String? toPersonId)
```

関係を結ぶ相手の人物IDが変更されたときに呼び出され、状態を更新し、バリデーションを実行します。

### updateType

```dart
void updateType(RelationType type)
```

関係タイプが変更されたときに呼び出され、状態を更新し、バリデーションを実行します。

### toRelationship

```dart
Relationship? toRelationship()
```

現在のフォーム状態 (`RelationshipFormState`) から`Relationship`モデルオブジェクトを生成します。フォームのバリデーション (`canSubmit`) が通っていない場合は`null`を返します。

**戻り値:**
- `Relationship?` - 生成された`Relationship`オブジェクト、またはバリデーションエラー時は`null`

### resetForm (オプション)

```dart
void resetForm(String fromPersonId)
```

フォームの状態を、指定された`fromPersonId`で初期状態に戻します。

## 使用例

```dart
// RelationshipDialogウィジェット内での使用

// フォームの初期化 (ダイアログ表示時 initState内など)
// final notifier = ref.read(relationshipFormProvider.notifier);
// if (widget.relationshipId == null) { // isNew
//   notifier.initializeForCreate(widget.fromPersonId);
// } else {
//   final existingRel = ref.read(relationshipByIdProvider(widget.relationshipId!)).value;
//   if (existingRel != null) notifier.initializeForEdit(existingRel);
// }

// 関係先人物選択ドロップダウンの例
DropdownButtonFormField<String>(
  value: ref.watch(relationshipFormProvider).toPersonId,
  items: availablePersons.map((p) => DropdownMenuItem(value: p.id, child: Text(p.name))).toList(),
  onChanged: (value) => ref.read(relationshipFormProvider.notifier).updateToPersonId(value),
  decoration: InputDecoration(
    labelText: '関係先の人物',
    errorText: ref.watch(relationshipFormProvider).toPersonIdError ?? 
               ref.watch(relationshipFormProvider).circularReferenceError ?? 
               ref.watch(relationshipFormProvider).duplicateRelationshipError,
  ),
  validator: (_) => ref.read(relationshipFormProvider).toPersonIdError ?? 
                   ref.read(relationshipFormProvider).circularReferenceError ?? 
                   ref.read(relationshipFormProvider).duplicateRelationshipError,
);

// 保存ボタンの処理
ElevatedButton(
  onPressed: ref.watch(relationshipFormProvider).canSubmit
    ? () async {
        final relationship = ref.read(relationshipFormProvider.notifier).toRelationship();
        if (relationship != null) {
          ref.read(relationshipFormSavingProvider.notifier).state = true;
          try {
            if (isNew) {
              await ref.read(relationshipControllerProvider).addRelationship(relationship);
            } else {
              await ref.read(relationshipControllerProvider).updateRelationship(relationship);
            }
            // 成功処理
          } catch (e) {
            // エラー処理
            ref.read(relationshipFormOverallErrorProvider.notifier).state = e.toString();
          } finally {
            ref.read(relationshipFormSavingProvider.notifier).state = false;
          }
        }
      }
    : null, // canSubmitがfalseなら無効化
  child: Text('設定'),
)
```

## 注意事項

- `RelationshipFormState`は不変オブジェクトとして`freezed`で定義されています。
- バリデーションロジックは`RelationshipValidators`クラス（別途定義想定）に集約し、`_validateForm`メソッドから呼び出します。このバリデーションには、循環参照チェックや重複関係チェックなど、他の関係性データとの比較が必要なため、`_ref`を通じてリポジトリや関連プロバイダにアクセスします。
- `_validateForm`メソッドは非同期になる可能性があるため（例：重複チェックでDBアクセス）、UIの更新タイミングに注意が必要です。`canSubmit`の状態はリアクティブに更新されます。
- `toRelationship`メソッドは、フォームデータを`Relationship`モデルに変換します。
- `autoDispose`をプロバイダに付与することで、フォームが不要になった際に自動的に状態が破棄されるようにしています。
- `relationshipFormSavingProvider`は保存処理中のUI制御に使用します。
- `relationshipFormOverallErrorProvider`は、フォーム全体に関わるエラーメッセージの表示に使用します。

## 関連するクラス・プロバイダ

- [Relationship](../モデル/Relationship.md) - 関係性データのモデルクラス
- `RelationshipValidators` (ユーティリティクラス) - 関係性フォームのバリデーションロジックを提供 (別途定義想定)
- [RelationshipController](./RelationshipController.md) - 関係性データの追加・更新・削除を行うコントローラ
- [RelationshipDialog](../ウィジェット/RelationshipDialog.md) - このフォームコントローラを利用するUIウィジェット
- `relationshipsStreamProvider` (Riverpodプロバイダ) - 全関係性データのストリームを提供（バリデーション用）
