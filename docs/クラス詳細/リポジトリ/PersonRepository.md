# PersonRepository クラス

## 概要

`PersonRepository` インターフェースと `FirestorePersonRepository` 実装クラスは、人物（Person）データのCRUD操作を提供します。人物データの永続化と取得を担当し、Firestoreとのデータやりとりをカプセル化します。

## インターフェース定義

```dart
// lib/repositories/person_repository.dart

abstract class PersonRepository {
  // 全ての人物情報を取得
  Stream<List<Person>> getPersons();
  
  // 特定の人物情報を取得
  Stream<Person?> getPerson(String id);
  
  // 一度だけ全ての人物情報を取得
  Future<List<Person>> getPersonsOnce();
  
  // 人物情報を追加
  Future<String> addPerson(Person person);
  
  // 人物情報を更新
  Future<void> updatePerson(Person person);
  
  // 人物情報を削除
  Future<void> deletePerson(String id);
}
```

## 実装クラス

```dart
// lib/repositories/firestore_person_repository.dart

class FirestorePersonRepository implements PersonRepository {
  final FirebaseFirestore _firestore;
  final String _userId;
  final String _treeId;
  
  FirestorePersonRepository({
    required FirebaseFirestore firestore,
    required String userId,
    required String treeId,
  }) : _firestore = firestore,
       _userId = userId,
       _treeId = treeId;
  
  // コレクションへの参照を取得
  CollectionReference<Map<String, dynamic>> get _collection =>
      _firestore.collection('users/$_userId/familyTrees/$_treeId/persons');
  
  @override
  Stream<List<Person>> getPersons() {
    return _collection
        .snapshots()
        .map((snapshot) => snapshot.docs
            .map((doc) => Person.fromFirestore(doc))
            .toList());
  }
  
  @override
  Stream<Person?> getPerson(String id) {
    return _collection
        .doc(id)
        .snapshots()
        .map((doc) => doc.exists ? Person.fromFirestore(doc) : null);
  }
  
  @override
  Future<List<Person>> getPersonsOnce() async {
    final snapshot = await _collection.get();
    return snapshot.docs
        .map((doc) => Person.fromFirestore(doc))
        .toList();
  }
  
  @override
  Future<String> addPerson(Person person) async {
    // バリデーション
    _validatePerson(person);
    
    // 新規ドキュメントを追加
    final docRef = await _collection.add(person.toFirestore());
    return docRef.id;
  }
  
  @override
  Future<void> updatePerson(Person person) async {
    // バリデーション
    _validatePerson(person);
    
    // ドキュメントの存在確認
    final docRef = _collection.doc(person.id);
    final doc = await docRef.get();
    if (!doc.exists) {
      throw Exception('更新対象の人物が見つかりません: ${person.id}');
    }
    
    // ドキュメントを更新
    await docRef.update(person.toFirestore());
  }
  
  @override
  Future<void> deletePerson(String id) async {
    // ドキュメントの存在確認
    final docRef = _collection.doc(id);
    final doc = await docRef.get();
    if (!doc.exists) {
      throw Exception('削除対象の人物が見つかりません: $id');
    }
    
    // ドキュメントを削除
    await docRef.delete();
  }
  
  // バリデーション
  void _validatePerson(Person person) {
    if (person.name.trim().isEmpty) {
      throw Exception('氏名は必須です');
    }
    
    // 生年月日と没年月日の整合性チェック
    if (person.birthDate != null && person.deathDate != null) {
      if (person.birthDate!.isAfter(person.deathDate!)) {
        throw Exception('生年月日は没年月日より前の日付である必要があります');
      }
      
      final now = DateTime.now();
      if (person.deathDate!.isAfter(now)) {
        throw Exception('没年月日は現在より前の日付である必要があります');
      }
    }
    
    if (person.birthDate != null) {
      final now = DateTime.now();
      if (person.birthDate!.isAfter(now)) {
        throw Exception('生年月日は現在より前の日付である必要があります');
      }
      
      final minDate = DateTime(1800, 1, 1);
      if (person.birthDate!.isBefore(minDate)) {
        throw Exception('生年月日は1800年以降である必要があります');
      }
    }
  }
}
```

## プロバイダ

```dart
// リポジトリプロバイダ
final personRepositoryProvider = Provider<PersonRepository>((ref) {
  final firestore = ref.watch(firestoreProvider);
  final userId = ref.watch(userIdProvider);
  final treeId = ref.watch(selectedTreeIdProvider);
  
  if (userId == null || treeId == null) {
    throw UnimplementedError('リポジトリの初期化に必要な情報が不足しています');
  }
  
  return FirestorePersonRepository(
    firestore: firestore,
    userId: userId,
    treeId: treeId,
  );
});
```

## メソッド詳細

### getPersons

```dart
Stream<List<Person>> getPersons()
```

家系図に登録されているすべての人物情報をリアルタイムで取得します。

**戻り値:**
- `Stream<List<Person>>` - 人物リストのストリーム。Firestoreのデータが更新されると自動的に新しい値が流れます。

### getPerson

```dart
Stream<Person?> getPerson(String id)
```

指定されたIDの人物情報をリアルタイムで取得します。

**パラメータ:**
- `id`: `String` - 取得する人物のID

**戻り値:**
- `Stream<Person?>` - 人物情報のストリーム。該当する人物が存在しない場合は`null`が流れます。

### getPersonsOnce

```dart
Future<List<Person>> getPersonsOnce()
```

家系図に登録されているすべての人物情報を一度だけ取得します。

**戻り値:**
- `Future<List<Person>>` - 人物リストを含むFuture

### addPerson

```dart
Future<String> addPerson(Person person)
```

新しい人物情報を追加します。

**パラメータ:**
- `person`: `Person` - 追加する人物データ。`id`プロパティは空文字列にしておき、Firestoreが自動生成したIDが返されます。

**戻り値:**
- `Future<String>` - 追加された人物の新しいID

**例外:**
- 氏名が空の場合
- 生年月日/没年月日の整合性が取れない場合

### updatePerson

```dart
Future<void> updatePerson(Person person)
```

既存の人物情報を更新します。

**パラメータ:**
- `person`: `Person` - 更新する人物データ。`id`プロパティには既存の人物IDを指定する必要があります。

**戻り値:**
- `Future<void>` - 更新が完了したことを示すFuture

**例外:**
- 氏名が空の場合
- 生年月日/没年月日の整合性が取れない場合
- 指定されたIDの人物が存在しない場合

### deletePerson

```dart
Future<void> deletePerson(String id)
```

指定されたIDの人物を削除します。

**パラメータ:**
- `id`: `String` - 削除する人物のID

**戻り値:**
- `Future<void>` - 削除が完了したことを示すFuture

**例外:**
- 指定されたIDの人物が存在しない場合

## 使用例

```dart
// リポジトリの取得
final repository = ref.read(personRepositoryProvider);

// 人物リストの監視
final personsStream = repository.getPersons();
personsStream.listen((persons) {
  print('人物リストが更新されました: ${persons.length}人');
});

// 新しい人物の追加
final person = Person(
  id: '',
  name: '山田 太郎',
  birthDate: DateTime(1980, 4, 1),
  gender: Gender.male,
);
final newId = await repository.addPerson(person);

// 人物情報の更新
final updatedPerson = person.copyWith(
  id: newId,
  memo: '会社役員。趣味は釣り。',
);
await repository.updatePerson(updatedPerson);

// 人物情報の取得
final personStream = repository.getPerson(newId);
personStream.listen((person) {
  if (person != null) {
    print('人物情報: ${person.name}');
  }
});

// 人物の削除
await repository.deletePerson(newId);
```

## 注意事項

- このリポジトリは、人物データの永続化と取得に関する操作をカプセル化します。アプリケーションの他の部分はFirestoreの実装詳細を知る必要がありません。
- リポジトリインターフェースを使うことで、将来的にデータストアを変更する際の影響を最小限に抑えることができます。
- バリデーションは、データの整合性を保つために重要です。必要に応じてバリデーションルールを追加してください。
- Firestoreのセキュリティルールと組み合わせて、適切なアクセス制御を実装してください。
- リアルタイム更新（Stream）と一度きりの取得（Future）の両方を提供し、用途に応じて使い分けられるようにしています。

## 関連するクラス・インターフェース

- [Person](../モデル/Person.md) - 家系図内の人物を表すモデルクラス
- [PersonController](../コントローラ/PersonController.md) - 人物操作を管理するコントローラクラス
- [RelationshipRepository](./RelationshipRepository.md) - 関係性データを管理するリポジトリインターフェース
