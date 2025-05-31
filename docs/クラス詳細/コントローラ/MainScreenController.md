# MainScreenController クラス

## 概要

`MainScreenController` クラスは、メイン画面 (`MainScreen`) の状態管理とUIロジックの一部を担当するコントローラクラスです。主に、画面サイズに基づいて現在のデバイスタイプ（モバイル、タブレット、デスクトップ）を判定し、それに応じたUIの調整（例：サイドパネルの表示/非表示）を制御します。Riverpodプロバイダを通じてアクセスされます。

## クラス定義

```dart
// lib/features/family_tree/application/main_screen_controller.dart

class MainScreenController {
  final Ref _ref;

  MainScreenController(this._ref);

  // 画面幅に基づいてデバイスタイプを更新
  // このメソッドは、MediaQueryやLayoutBuilderから画面幅が変更された際に呼び出されることを想定
  void updateDeviceType(double screenWidth) {
    final newDeviceType = _getDeviceTypeFromWidth(screenWidth);
    final currentDeviceType = _ref.read(deviceTypeProvider);

    if (newDeviceType != currentDeviceType) {
      _ref.read(deviceTypeProvider.notifier).state = newDeviceType;
    }
    // 画面幅プロバイダも更新 (必要に応じて)
    // _ref.read(screenWidthProvider.notifier).state = screenWidth;
  }

  // サイドパネルの表示/非表示を切り替える (主にデバッグ用や将来的な手動制御用)
  void toggleSidePanelVisibility() {
    final currentVisibility = _ref.read(isSidePanelVisibleProvider);
    _ref.read(isSidePanelVisibleProvider.notifier).state = !currentVisibility;
  }
  
  // 画面幅からデバイスタイプを判定するロジック
  DeviceType _getDeviceTypeFromWidth(double screenWidth) {
    if (screenWidth >= Breakpoints.desktop) {
      return DeviceType.desktop;
    } else if (screenWidth >= Breakpoints.tablet) {
      return DeviceType.tablet;
    } else {
      return DeviceType.mobile;
    }
  }
  
  // アプリケーションの初期化処理をトリガー (オプション)
  // mainScreenInitializationProviderが自動実行するため、通常は不要
  Future<void> initializeApp() async {
    await _ref.read(mainScreenInitializationProvider.future);
  }
}
```

## 関連するプロバイダ

```dart
// lib/features/family_tree/application/main_screen_providers.dart (または適切な場所)

// デバイスタイプを定義する列挙型
enum DeviceType {
  mobile,
  tablet,
  desktop,
}

// ブレイクポイントを定義するクラス
class Breakpoints {
  static const double mobile = 600;  // モバイルとタブレットの境界
  static const double tablet = 840;  // タブレットとデスクトップの境界 (Material Design 3 ガイドライン参考)
  static const double desktop = 1024; // より大きなデスクトップの境界 (プロジェクトの要件に応じて調整)
}

// 現在の画面幅を保持するプロバイダ (MainScreenのbuildメソッドなどで更新)
final screenWidthProvider = StateProvider<double>((ref) => 0.0);

// 現在のデバイスタイプを保持するプロバイダ
final deviceTypeProvider = StateProvider<DeviceType>((ref) {
  // screenWidthProviderを監視して動的に変更することも可能だが、
  // MainScreenControllerのupdateDeviceType経由での更新を主とする
  final screenWidth = ref.watch(screenWidthProvider);
  if (screenWidth >= Breakpoints.desktop) {
    return DeviceType.desktop;
  } else if (screenWidth >= Breakpoints.tablet) {
    return DeviceType.tablet;
  } else {
    return DeviceType.mobile;
  }
});

// サイドパネルが表示されているかどうかを管理するプロバイダ
// デスクトップではデフォルトで表示、それ以外では非表示
final isSidePanelVisibleProvider = StateProvider<bool>((ref) {
  final deviceType = ref.watch(deviceTypeProvider);
  return deviceType == DeviceType.desktop;
});

// メイン画面コントローラのプロバイダ
final mainScreenControllerProvider = Provider<MainScreenController>((ref) {
  return MainScreenController(ref);
});

// メイン画面の初期化処理を行うプロバイダ
final mainScreenInitializationProvider = FutureProvider.autoDispose<void>((ref) async {
  // 匿名認証状態を確認し、必要であればサインイン
  final authService = ref.watch(anonymousAuthServiceProvider);
  if (authService.getCurrentUser() == null) {
    try {
      await authService.signInAnonymously();
      print('匿名サインイン成功 (初期化時)');
    } catch (e) {
      print('初期化時の匿名サインイン失敗: $e');
      // 必要に応じてエラーハンドリング
    }
  }
  
  // ユーザーIDが確定したら、最初の家系図を選択または作成を促す
  final userId = ref.watch(currentUserIdProvider);
  if (userId != null) {
    final treeRepo = ref.watch(familyTreeStorageRepositoryProvider);
    final trees = await treeRepo.getFamilyTrees().first; // 最初のリストを取得
    
    if (trees.isNotEmpty) {
      // 既存の家系図があれば最初のものを選択
      ref.read(selectedFamilyTreeIdProvider.notifier).state = trees.first.id;
    } else {
      // 家系図がない場合は、UI側で新規作成を促すなどの処理が必要
      print('家系図が存在しません。新規作成を促してください。');
      // ref.read(showCreateTreeDialogProvider.notifier).state = true; (例)
    }
  } else {
    print('ユーザーIDが取得できませんでした。データアクセスに問題がある可能性があります。');
  }
  
  // その他の初期化処理 (例: 設定の読み込みなど)
});
```

## メソッド詳細

### updateDeviceType

```dart
void updateDeviceType(double screenWidth)
```

指定された画面幅に基づいて現在のデバイスタイプを判定し、`deviceTypeProvider`の状態を更新します。このメソッドは、`MainScreen`ウィジェットが画面サイズの変更を検知した際に呼び出されることを想定しています（例：`LayoutBuilder`や`MediaQuery`を使用）。

**パラメータ:**
- `screenWidth`: `double` - 現在の画面幅

### toggleSidePanelVisibility

```dart
void toggleSidePanelVisibility()
```

サイドパネルの表示/非表示状態を切り替えます。主にデバッグ目的や、将来的にユーザーが手動でサイドパネルの表示を制御する機能を追加する場合に使用されます。

### _getDeviceTypeFromWidth (プライベートヘルパー)

```dart
DeviceType _getDeviceTypeFromWidth(double screenWidth)
```

画面幅を受け取り、定義されたブレイクポイントに基づいて`DeviceType`（mobile, tablet, desktop）を返します。

### initializeApp (オプション)

```dart
Future<void> initializeApp() async
```

アプリケーションの初期化処理（`mainScreenInitializationProvider`）を明示的にトリガーします。通常、`mainScreenInitializationProvider`は自動的に実行されるため、このメソッドの直接呼び出しは限定的なケースでのみ使用されます。

## 使用例

```dart
// MainScreenウィジェットのbuildメソッド内やinitStateでの画面幅監視と更新
// (ただし、buildメソッド内での直接的な状態更新は避けるべき。リスナーやコールバックを使用)

// 例: LayoutBuilderを使用する場合
LayoutBuilder(
  builder: (context, constraints) {
    // このコールバックは頻繁に呼ばれるため、直接コントローラを呼ぶのではなく
    // 差分がある場合のみ更新するなどの工夫が必要
    WidgetsBinding.instance.addPostFrameCallback((_) {
        // 画面幅が実際に変更された場合のみ更新するなどの制御を入れる
        final currentScreenWidth = ref.read(screenWidthProvider);
        if ((currentScreenWidth - constraints.maxWidth).abs() > 1.0) { // 閾値を設ける
            ref.read(screenWidthProvider.notifier).state = constraints.maxWidth;
            ref.read(mainScreenControllerProvider).updateDeviceType(constraints.maxWidth);
        }
    });
    return YourMainScreenContent();
  }
)

// デバイスタイプに応じたUIの分岐
final deviceType = ref.watch(deviceTypeProvider);
if (deviceType == DeviceType.desktop) {
  // デスクトップ用レイアウト
} else {
  // モバイル/タブレット用レイアウト
}

// サイドパネルの表示制御
final isSidePanelVisible = ref.watch(isSidePanelVisibleProvider);
if (isSidePanelVisible) {
  // サイドパネルを表示
}
```

## 注意事項

- `updateDeviceType`メソッドは、画面サイズの変更を検知するウィジェット（例：`MainScreen`内の`LayoutBuilder`や、`WidgetsBindingObserver`を利用した`didChangeMetrics`）から呼び出されることを想定しています。ビルドメソッド内で直接呼び出すと無限ループを引き起こす可能性があるため注意が必要です。
- `screenWidthProvider`は、`deviceTypeProvider`が画面幅の変更に反応できるようにするために存在しますが、更新頻度が高い場合はパフォーマンスへの影響を考慮する必要があります。
- `mainScreenInitializationProvider`は、アプリ起動時に必要な非同期処理（認証状態の確認、初期データのロードなど）をまとめて行います。このプロバイダが完了するまでは、UI側でローディング表示を行うことが推奨されます。
- ブレイクポイント（`Breakpoints`クラス）の値は、アプリケーションのデザイン要件やターゲットデバイスに応じて調整してください。

## 関連するクラス・プロバイダ

- `DeviceType` (列挙型) - モバイル、タブレット、デスクトップのデバイスタイプを定義
- `Breakpoints` (クラス) - 画面幅のブレイクポイントを定義
- `screenWidthProvider` (Riverpodプロバイダ) - 現在の画面幅を保持
- `deviceTypeProvider` (Riverpodプロバイダ) - 現在のデバイスタイプを保持
- `isSidePanelVisibleProvider` (Riverpodプロバイダ) - サイドパネルの表示状態を管理
- `mainScreenInitializationProvider` (Riverpodプロバイダ) - アプリケーションの初期化処理を実行
- [MainScreen](../ウィジェット/MainScreen.md) - このコントローラを利用するメイン画面ウィジェット
- [AnonymousAuthService](../サービス/AnonymousAuthService.md) - 匿名認証を行うサービス
- [FamilyTreeStorageRepository](../リポジトリ/FamilyTreeStorageRepository.md) - 家系図データのストレージアクセス
