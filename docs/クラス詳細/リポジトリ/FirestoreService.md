# FirestoreService クラス

## 概要

`FirestoreService` クラスは、Firebase Firestoreとのデータアクセスを抽象化し、共通の機能を提供するサービスクラスです。このクラスは、各リポジトリクラスから利用され、データの永続化に関する共通のロジックをカプセル化します。

## クラス定義

```dart
// lib/services/firestore_service.dart

class FirestoreService {
  final FirebaseFirestore _firestore;
  final String _userId;
  final String _treeId;
  
  FirestoreService({
    required FirebaseFirestore firestore,
    required String userId,
    required String treeId,
  }) : _firestore = firestore,
       _userId = userId,
       _treeId = treeId;
  
  // コレクションへの参照を取得
  CollectionReference<Map<String, dynamic>> getCollection(String collectionName) {
    return _firestore.collection('users/$_userId/familyTrees/$_treeId/$collectionName');
  }
  
  // データのストリームを取得
  Stream<List<T>> getDataStream<T>({
    required String collectionName,
    required T Function(DocumentSnapshot<Map<String, dynamic>>) fromFirestore,
    Query<Map<String, dynamic>>? query,
  }) {
    final collection = getCollection(collectionName);
    final streamSource = query ?? collection;
    
    return streamSource
        .snapshots()
        .map((snapshot) => snapshot.docs
            .map((doc) => fromFirestore(doc))
            .toList());
  }
  
  // 特定のドキュメントのストリームを取得
  Stream<T?> getDocumentStream<T>({
    required String collectionName,
    required String documentId,
    required T Function(DocumentSnapshot<Map<String, dynamic>>) fromFirestore,
  }) {
    return getCollection(collectionName)
        .doc(documentId)
        .snapshots()
        .map((doc) => doc.exists ? fromFirestore(doc) : null);
  }
  
  // 一度だけデータを取得
  Future<List<T>> getDataOnce<T>({
    required String collectionName,
    required T Function(DocumentSnapshot<Map<String, dynamic>>) fromFirestore,
    Query<Map<String, dynamic>>? query,
  }) async {
    final collection = getCollection(collectionName);
    final querySource = query ?? collection;
    
    final snapshot = await querySource.get();
    return snapshot.docs
        .map((doc) => fromFirestore(doc))
        .toList();
  }
  
  // ドキュメントを追加
  Future<String> addDocument({
    required String collectionName,
    required Map<String, dynamic> data,
  }) async {
    final docRef = await getCollection(collectionName).add(data);
    return docRef.id;
  }
  
  // ドキュメントを更新
  Future<void> updateDocument({
    required String collectionName,
    required String documentId,
    required Map<String, dynamic> data,
  }) async {
    await getCollection(collectionName)
        .doc(documentId)
        .update(data);
  }
  
  // ドキュメントを削除
  Future<void> deleteDocument({
    required String collectionName,
    required String documentId,
  }) async {
    await getCollection(collectionName)
        .doc(documentId)
        .delete();
  }
  
  // バッチ操作の実行
  Future<void> runBatch(void Function(WriteBatch batch) operations) async {
    final batch = _firestore.batch();
    operations(batch);
    await batch.commit();
  }
  
  // トランザクションの実行
  Future<T> runTransaction<T>(
    Future<T> Function(Transaction transaction) operations
  ) async {
    return await _firestore.runTransaction(operations);
  }
  
  // 条件に一致するドキュメントをバッチ削除
  Future<void> batchDeleteDocuments({
    required String collectionName,
    required Query<Map<String, dynamic>> query,
  }) async {
    final snapshot = await query.get();
    
    if (snapshot.docs.isEmpty) {
      return;
    }
    
    // Firestoreのバッチ制限（500）を考慮して分割処理
    const batchSize = 400;
    final batches = <List<QueryDocumentSnapshot<Map<String, dynamic>>>>[];
    
    for (var i = 0; i < snapshot.docs.length; i += batchSize) {
      final end = (i + batchSize < snapshot.docs.length) 
          ? i + batchSize 
          : snapshot.docs.length;
      batches.add(snapshot.docs.sublist(i, end));
    }
    
    // 各バッチを順次実行
    for (final batchDocs in batches) {
      await runBatch((batch) {
        for (final doc in batchDocs) {
          batch.delete(doc.reference);
        }
      });
    }
  }
}
```

## プロバイダ

```dart
// lib/services/firestore_service_provider.dart

final firestoreServiceProvider = Provider<FirestoreService>((ref) {
  final firestore = ref.watch(firestoreProvider);
  final userId = ref.watch(userIdProvider);
  final treeId = ref.watch(selectedTreeIdProvider);
  
  if (userId == null || treeId == null) {
    throw UnimplementedError('サービスの初期化に必要な情報が不足しています');
  }
  
  return FirestoreService(
    firestore: firestore,
    userId: userId,
    treeId: treeId,
  );
});
```

## メソッド詳細

### getCollection

```dart
CollectionReference<Map<String, dynamic>> getCollection(String collectionName)
```

指定されたコレクション名に対応するFirestoreのコレクション参照を取得します。

**パラメータ:**
- `collectionName`: `String` - コレクション名

**戻り値:**
- `CollectionReference<Map<String, dynamic>>` - コレクション参照

### getDataStream

```dart
Stream<List<T>> getDataStream<T>({
  required String collectionName,
  required T Function(DocumentSnapshot<Map<String, dynamic>>) fromFirestore,
  Query<Map<String, dynamic>>? query,
})
```

コレクションまたはクエリに基づいてデータのストリームを取得します。

**パラメータ:**
- `collectionName`: `String` - コレクション名
- `fromFirestore`: `T Function(DocumentSnapshot)` - ドキュメントから型Tのオブジェクトを生成する関数
- `query`: `Query?` - オプションのクエリ（フィルタリングなど）

**戻り値:**
- `Stream<List<T>>` - 型Tのオブジェクトリストのストリーム

### getDocumentStream

```dart
Stream<T?> getDocumentStream<T>({
  required String collectionName,
  required String documentId,
  required T Function(DocumentSnapshot<Map<String, dynamic>>) fromFirestore,
})
```

特定のドキュメントのストリームを取得します。

**パラメータ:**
- `collectionName`: `String` - コレクション名
- `documentId`: `String` - ドキュメントID
- `fromFirestore`: `T Function(DocumentSnapshot)` - ドキュメントから型Tのオブジェクトを生成する関数

**戻り値:**
- `Stream<T?>` - 型Tのオブジェクトのストリーム（存在しない場合は`null`）

### addDocument

```dart
Future<String> addDocument({
  required String collectionName,
  required Map<String, dynamic> data,
})
```

新しいドキュメントを追加します。

**パラメータ:**
- `collectionName`: `String` - コレクション名
- `data`: `Map<String, dynamic>` - 保存するデータ

**戻り値:**
- `Future<String>` - 新しく生成されたドキュメントID

### updateDocument

```dart
Future<void> updateDocument({
  required String collectionName,
  required String documentId,
  required Map<String, dynamic> data,
})
```

既存のドキュメントを更新します。

**パラメータ:**
- `collectionName`: `String` - コレクション名
- `documentId`: `String` - 更新するドキュメントのID
- `data`: `Map<String, dynamic>` - 更新するデータ

### deleteDocument

```dart
Future<void> deleteDocument({
  required String collectionName,
  required String documentId,
})
```

ドキュメントを削除します。

**パラメータ:**
- `collectionName`: `String` - コレクション名
- `documentId`: `String` - 削除するドキュメントのID

### runBatch

```dart
Future<void> runBatch(void Function(WriteBatch batch) operations)
```

バッチ操作を実行します。

**パラメータ:**
- `operations`: `void Function(WriteBatch)` - バッチに対して実行する操作

### runTransaction

```dart
Future<T> runTransaction<T>(Future<T> Function(Transaction) operations)
```

トランザクションを実行します。

**パラメータ:**
- `operations`: `Future<T> Function(Transaction)` - トランザクション内で実行する操作

**戻り値:**
- `Future<T>` - トランザクションの実行結果

## 使用例

```dart
// サービスの取得
final service = ref.read(firestoreServiceProvider);

// データのストリーム取得
final personsStream = service.getDataStream<Person>(
  collectionName: 'persons',
  fromFirestore: (doc) => Person.fromFirestore(doc),
);

// フィルタリングを使用したデータ取得
final query = service.getCollection('persons')
    .where('gender', isEqualTo: Gender.male.toString());
final malePersonsStream = service.getDataStream<Person>(
  collectionName: 'persons',
  fromFirestore: (doc) => Person.fromFirestore(doc),
  query: query,
);

// ドキュメントの追加
final person = Person(id: '', name: '山田 太郎');
final newId = await service.addDocument(
  collectionName: 'persons',
  data: person.toFirestore(),
);

// ドキュメントの更新
final updatedPerson = person.copyWith(id: newId, memo: '会社役員');
await service.updateDocument(
  collectionName: 'persons',
  documentId: newId,
  data: updatedPerson.toFirestore(),
);

// バッチ操作の例（複数の更新を一括で実行）
await service.runBatch((batch) {
  final collection = service.getCollection('persons');
  batch.set(collection.doc('id1'), {'name': '山田 一郎'});
  batch.set(collection.doc('id2'), {'name': '山田 二郎'});
  batch.delete(collection.doc('id3'));
});

// トランザクションの例（読み取りと書き込みを原子的に実行）
final newCount = await service.runTransaction((transaction) async {
  final docRef = service.getCollection('counters').doc('personCount');
  final snapshot = await transaction.get(docRef);
  final currentCount = snapshot.data()?['count'] as int? ?? 0;
  transaction.update(docRef, {'count': currentCount + 1});
  return currentCount + 1;
});
```

## 注意事項

- このサービスクラスは、Firebase Firestoreへのアクセスを抽象化し、共通のデータアクセスパターンを提供します。
- バッチ操作とトランザクションは、データの整合性を保つために重要です。
- バッチ操作は最大500の操作に制限されているため、大量のデータを処理する場合は分割して実行する必要があります。
- エラーハンドリングは呼び出し元で適切に行う必要があります。
- Firestoreのセキュリティルールと組み合わせて、適切なアクセス制御を実装してください。

## 関連するクラス・インターフェース

- [PersonRepository](./PersonRepository.md) - 人物データを管理するリポジトリ
- [RelationshipRepository](./RelationshipRepository.md) - 関係性データを管理するリポジトリ
- [Person](../モデル/Person.md) - 人物を表すモデルクラス
- [Relationship](../モデル/Relationship.md) - 関係性を表すモデルクラス
