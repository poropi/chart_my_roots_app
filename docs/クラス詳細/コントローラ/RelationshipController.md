# RelationshipController クラス

## 概要

`RelationshipController` クラスは人物間の関係性（Relationship）に関する操作を管理するコントローラクラスです。Riverpodプロバイダを通じてアクセスされ、親子関係や配偶者関係の追加、更新、削除などの操作をリポジトリと連携して実行します。また、関係性の整合性チェックも行います。

## クラス定義

```dart
class RelationshipController {
  final RelationshipRepository _repository;
  
  RelationshipController(this._repository);
  
  // 関係性の追加
  Future<String> addRelationship(Relationship relationship) async {
    try {
      // 同じ人物間で同じ種類の関係が既に存在するかチェック
      final exists = await _checkDuplicateRelationship(relationship);
      if (exists) {
        throw Exception('この2人の間には既に同じ種類の関係が設定されています');
      }
      
      // 循環する親子関係をチェック
      if (relationship.type == RelationType.parentChild) {
        final hasCircular = await _checkCircularParentChildRelationship(
          relationship.fromPersonId,
          relationship.toPersonId,
        );
        if (hasCircular) {
          throw Exception('循環する親子関係は設定できません');
        }
      }
      
      final newId = await _repository.addRelationship(relationship);
      return newId;
    } catch (e) {
      // エラーログ記録
      _logError('Failed to add relationship', e);
      rethrow;
    }
  }
  
  // 関係性の更新
  Future<void> updateRelationship(Relationship relationship) async {
    try {
      // 同じ人物間で同じ種類の関係が既に存在するかチェック（自分自身は除く）
      final exists = await _checkDuplicateRelationship(relationship, excludeId: relationship.id);
      if (exists) {
        throw Exception('この2人の間には既に同じ種類の関係が設定されています');
      }
      
      // 循環する親子関係をチェック
      if (relationship.type == RelationType.parentChild) {
        final hasCircular = await _checkCircularParentChildRelationship(
          relationship.fromPersonId,
          relationship.toPersonId,
        );
        if (hasCircular) {
          throw Exception('循環する親子関係は設定できません');
        }
      }
      
      await _repository.updateRelationship(relationship);
    } catch (e) {
      // エラーログ記録
      _logError('Failed to update relationship', e);
      rethrow;
    }
  }
  
  // 関係性の削除
  Future<void> deleteRelationship(String id) async {
    try {
      await _repository.deleteRelationship(id);
    } catch (e) {
      // エラーログ記録
      _logError('Failed to delete relationship', e);
      rethrow;
    }
  }
  
  // 特定の人物に関連する関係性を取得
  Future<List<Relationship>> getRelationshipsForPerson(String personId) async {
    try {
      return await _repository.getRelationshipsForPerson(personId);
    } catch (e) {
      // エラーログ記録
      _logError('Failed to get relationships for person', e);
      rethrow;
    }
  }
  
  // 重複する関係性がないかチェック
  Future<bool> _checkDuplicateRelationship(
    Relationship relationship, {
    String? excludeId,
  }) async {
    final relationships = await _repository.getRelationshipsOnce();
    
    return relationships.any((rel) {
      // 自分自身は除外
      if (excludeId != null && rel.id == excludeId) {
        return false;
      }
      
      // 同じタイプの関係で、同じ人物同士の組み合わせをチェック
      if (rel.type != relationship.type) {
        return false;
      }
      
      // 親子関係の場合、方向が重要
      if (relationship.type == RelationType.parentChild) {
        return rel.fromPersonId == relationship.fromPersonId &&
               rel.toPersonId == relationship.toPersonId;
      }
      
      // 配偶者関係の場合、方向は関係ない
      return (rel.fromPersonId == relationship.fromPersonId &&
              rel.toPersonId == relationship.toPersonId) ||
             (rel.fromPersonId == relationship.toPersonId &&
              rel.toPersonId == relationship.fromPersonId);
    });
  }
  
  // 循環する親子関係をチェック
  Future<bool> _checkCircularParentChildRelationship(
    String fromPersonId,
    String toPersonId,
  ) async {
    // すべての親子関係を取得
    final relationships = await _repository.getRelationshipsOnce();
    final parentChildRelationships = relationships.where(
      (rel) => rel.type == RelationType.parentChild
    ).toList();
    
    // fromPersonId（親）の先祖にtoPersonId（子）が含まれるかをチェック
    // （循環する場合はtrue）
    final ancestorIds = <String>{};
    _findAncestors(fromPersonId, parentChildRelationships, ancestorIds);
    
    return ancestorIds.contains(toPersonId);
  }
  
  // 先祖を再帰的に探索する補助メソッド
  void _findAncestors(
    String personId,
    List<Relationship> parentChildRelationships,
    Set<String> ancestorIds,
  ) {
    // 親を探す
    final parentRelations = parentChildRelationships.where(
      (rel) => rel.toPersonId == personId,
    );
    
    for (final relation in parentRelations) {
      final parentId = relation.fromPersonId;
      ancestorIds.add(parentId);
      _findAncestors(parentId, parentChildRelationships, ancestorIds);
    }
  }
  
  // エラーログ記録（実際の実装ではロギングライブラリを使用）
  void _logError(String message, dynamic error) {
    print('ERROR: $message - $error');
  }
}
```

## 関連するプロバイダ

```dart
// リポジトリのプロバイダ
final relationshipRepositoryProvider = Provider<RelationshipRepository>((ref) {
  throw UnimplementedError('Provider was not initialized');
});

// 関係性データのストリームプロバイダ
final relationshipsStreamProvider = StreamProvider<List<Relationship>>((ref) {
  final repository = ref.watch(relationshipRepositoryProvider);
  return repository.getRelationships();
});

// 特定の人物に関連する関係性のプロバイダ
final personRelationshipsProvider = FutureProvider.family<List<Relationship>, String>((ref, personId) async {
  final controller = ref.watch(relationshipControllerProvider);
  return await controller.getRelationshipsForPerson(personId);
});

// 関係性操作用コントローラのプロバイダ
final relationshipControllerProvider = Provider<RelationshipController>((ref) {
  final repository = ref.watch(relationshipRepositoryProvider);
  return RelationshipController(repository);
});
```

## メソッド

### addRelationship

```dart
Future<String> addRelationship(Relationship relationship) async
```

新しい関係性を追加します。同じ人物間で同じ種類の関係が既に存在する場合や、循環する親子関係となる場合は例外をスローします。

**パラメータ:**
- `relationship`: `Relationship` - 追加する関係性データ。`id`プロパティは空文字列にしておき、Firestoreが自動生成したIDが返されます。

**戻り値:**
- `Future<String>` - 追加された関係性の新しいID

**例外:**
- 重複する関係性が存在する場合や循環する親子関係となる場合、例外がスローされます。
- リポジトリでのデータ操作に失敗した場合、例外がスローされます。

### updateRelationship

```dart
Future<void> updateRelationship(Relationship relationship) async
```

既存の関係性情報を更新します。同じ人物間で同じ種類の関係が既に存在する場合や、循環する親子関係となる場合は例外をスローします。

**パラメータ:**
- `relationship`: `Relationship` - 更新する関係性データ。`id`プロパティには既存の関係性IDを指定する必要があります。

**戻り値:**
- `Future<void>` - 更新が完了したことを示すFuture

**例外:**
- 重複する関係性が存在する場合や循環する親子関係となる場合、例外がスローされます。
- リポジトリでのデータ操作に失敗した場合、例外がスローされます。

### deleteRelationship

```dart
Future<void> deleteRelationship(String id) async
```

指定されたIDの関係性を削除します。

**パラメータ:**
- `id`: `String` - 削除する関係性のID

**戻り値:**
- `Future<void>` - 削除が完了したことを示すFuture

**例外:**
- リポジトリでのデータ操作に失敗した場合、例外がスローされます。

### getRelationshipsForPerson

```dart
Future<List<Relationship>> getRelationshipsForPerson(String personId) async
```

特定の人物に関連するすべての関係性を取得します。

**パラメータ:**
- `personId`: `String` - 関連する関係性を取得したい人物のID

**戻り値:**
- `Future<List<Relationship>>` - 人物に関連する関係性のリスト

**例外:**
- リポジトリでのデータ操作に失敗した場合、例外がスローされます。

## 使用例

```dart
// コントローラの取得
final controller = ref.read(relationshipControllerProvider);

// 親子関係の追加
final parentChildRelationship = Relationship(
  id: '',
  fromPersonId: 'parent-id',  // 親の人物ID
  toPersonId: 'child-id',     // 子の人物ID
  type: RelationType.parentChild,
);
final newId = await controller.addRelationship(parentChildRelationship);

// 配偶者関係の追加
final spouseRelationship = Relationship(
  id: '',
  fromPersonId: 'person1-id',
  toPersonId: 'person2-id',
  type: RelationType.spouse,
);
await controller.addRelationship(spouseRelationship);

// 関係性の更新
final updatedRelationship = parentChildRelationship.copyWith(
  id: newId,
);
await controller.updateRelationship(updatedRelationship);

// 特定の人物に関連する関係性を取得
final relationships = await controller.getRelationshipsForPerson('person-id');

// 関係性の削除
await controller.deleteRelationship(newId);
```

## 注意事項

- このコントローラは、人物間の関係性に関するビジネスロジックを実装しています。特に重要なのは、データの整合性を確保するためのチェックです。
- 重複する関係性のチェックにより、同じ人物間に同じ種類の関係が複数存在することを防止しています。
- 循環する親子関係のチェックにより、A→B→C→Aのような無限ループになる関係設定を防止しています。
- 親子関係では方向性が重要ですが、配偶者関係では方向性は重要ではありません。
- すべての操作でエラーハンドリングを行い、エラーログを記録します。実際の実装では適切なロギングライブラリを使用してください。

## 関連するクラス・インターフェース

- [Relationship](../モデル/Relationship.md) - 人物間の関係性を表すモデルクラス
- [RelationshipRepository](../リポジトリ/RelationshipRepository.md) - 関係性データのCRUD操作を定義するインターフェース
- [Person](../モデル/Person.md) - 家系図内の人物を表すモデルクラス
