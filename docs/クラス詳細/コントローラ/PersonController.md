# PersonController クラス

## 概要

`PersonController` クラスは人物（Person）に関する操作を管理するコントローラクラスです。Riverpodプロバイダを通じてアクセスされ、人物の追加、更新、削除などの操作をリポジトリと連携して実行します。また、人物選択状態の管理も行います。

## クラス定義

```dart
class PersonController {
  final Ref _ref;
  
  PersonController(this._ref);
  
  // 人物の追加
  Future<String> addPerson(Person person) async {
    try {
      final repository = _ref.read(personRepositoryProvider);
      return await repository.addPerson(person);
    } catch (e) {
      // エラーログ記録
      _logError('Failed to add person', e);
      rethrow;
    }
  }
  
  // 人物の更新
  Future<void> updatePerson(Person person) async {
    try {
      final repository = _ref.read(personRepositoryProvider);
      await repository.updatePerson(person);
    } catch (e) {
      // エラーログ記録
      _logError('Failed to update person', e);
      rethrow;
    }
  }
  
  // 人物の削除
  Future<void> deletePerson(String id) async {
    try {
      // 人物を削除すると、関連する関係性も自動的に削除される
      final personRepo = _ref.read(personRepositoryProvider);
      final relationshipRepo = _ref.read(relationshipRepositoryProvider);
      
      // 関連する関係性を先に削除
      await relationshipRepo.deleteRelationshipsForPerson(id);
      
      // 人物を削除
      await personRepo.deletePerson(id);
      
      // 選択中の人物が削除された場合、選択を解除
      final selectedId = _ref.read(selectedPersonIdProvider);
      if (selectedId == id) {
        _ref.read(selectedPersonIdProvider.notifier).state = null;
      }
    } catch (e) {
      // エラーログ記録
      _logError('Failed to delete person', e);
      rethrow;
    }
  }
  
  // 人物の選択
  void selectPerson(String? personId) {
    _ref.read(selectedPersonIdProvider.notifier).state = personId;
  }
  
  // エラーログ記録（実際の実装ではロギングライブラリを使用）
  void _logError(String message, dynamic error) {
    print('ERROR: $message - $error');
  }
}
```

## 関連するプロバイダ

```dart
// リポジトリのプロバイダ (依存性の注入)
final personRepositoryProvider = Provider<PersonRepository>((ref) {
  throw UnimplementedError('Provider was not initialized');
});

final relationshipRepositoryProvider = Provider<RelationshipRepository>((ref) {
  throw UnimplementedError('Provider was not initialized');
});

// 人物データのストリームプロバイダ
final personsStreamProvider = StreamProvider<List<Person>>((ref) {
  final repository = ref.watch(personRepositoryProvider);
  return repository.getPersons();
});

// 選択中の人物IDを保持するプロバイダ
final selectedPersonIdProvider = StateProvider<String?>((ref) => null);

// 選択中の人物の詳細情報を提供するプロバイダ
final selectedPersonProvider = Provider<Person?>((ref) {
  final personId = ref.watch(selectedPersonIdProvider);
  final personsAsyncValue = ref.watch(personsStreamProvider);
  
  return personsAsyncValue.when(
    data: (persons) {
      if (personId == null) return null;
      return persons.firstWhere(
        (person) => person.id == personId,
        orElse: () => null,
      );
    },
    loading: () => null,
    error: (_, __) => null,
  );
});

// 人物操作用コントローラのプロバイダ
final personControllerProvider = Provider<PersonController>((ref) {
  return PersonController(ref);
});
```

## メソッド

### addPerson

```dart
Future<String> addPerson(Person person) async
```

新しい人物を追加します。

**パラメータ:**
- `person`: `Person` - 追加する人物データ。`id`プロパティは空文字列にしておき、Firestoreが自動生成したIDが返されます。

**戻り値:**
- `Future<String>` - 追加された人物の新しいID

**例外:**
- リポジトリでのデータ操作に失敗した場合、例外がスローされます。

### updatePerson

```dart
Future<void> updatePerson(Person person) async
```

既存の人物情報を更新します。

**パラメータ:**
- `person`: `Person` - 更新する人物データ。`id`プロパティには既存の人物IDを指定する必要があります。

**戻り値:**
- `Future<void>` - 更新が完了したことを示すFuture

**例外:**
- リポジトリでのデータ操作に失敗した場合、例外がスローされます。

### deletePerson

```dart
Future<void> deletePerson(String id) async
```

指定されたIDの人物を削除します。関連するすべての関係性も自動的に削除されます。

**パラメータ:**
- `id`: `String` - 削除する人物のID

**戻り値:**
- `Future<void>` - 削除が完了したことを示すFuture

**例外:**
- リポジトリでのデータ操作に失敗した場合、例外がスローされます。

### selectPerson

```dart
void selectPerson(String? personId)
```

指定された人物を選択状態にします。`null`を指定すると選択を解除します。

**パラメータ:**
- `personId`: `String?` - 選択する人物のID、または選択解除の場合は`null`

**戻り値:**
- なし

## 使用例

```dart
// コントローラの取得
final controller = ref.read(personControllerProvider);

// 新しい人物の追加
final person = Person(
  id: '',
  name: '山田 太郎',
  birthDate: DateTime(1980, 4, 1),
  gender: Gender.male,
);
final newId = await controller.addPerson(person);

// 人物の更新
final updatedPerson = person.copyWith(
  id: newId,
  memo: '会社役員。趣味は釣り。',
);
await controller.updatePerson(updatedPerson);

// 人物の選択
controller.selectPerson(newId);

// 選択中の人物の取得
final selectedPerson = ref.watch(selectedPersonProvider);

// 人物の削除
await controller.deletePerson(newId);
```

## 注意事項

- このコントローラは、ビジネスロジックとデータアクセス層を橋渡しする役割を持ちます。UIからの操作を受け取り、リポジトリを通じてデータを更新します。
- Riverpodのプロバイダを通じて依存性を注入することで、テスト容易性を高めています。
- 人物の削除時には関連する関係性も削除するようにしています。これにより、データの整合性が保たれます。
- 選択中の人物が削除された場合は、自動的に選択状態を解除します。
- すべての操作でエラーハンドリングを行い、エラーログを記録します。実際の実装では適切なロギングライブラリを使用してください。

## 関連するクラス・インターフェース

- [Person](../モデル/Person.md) - 家系図内の人物を表すモデルクラス
- [PersonRepository](../リポジトリ/PersonRepository.md) - 人物データのCRUD操作を定義するインターフェース
- [RelationshipRepository](../リポジトリ/RelationshipRepository.md) - 関係性データのCRUD操作を定義するインターフェース
