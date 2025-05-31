# MainScreen ウィジェット

## 概要

`MainScreen` ウィジェットは、家系図アプリケーションのメインとなる画面UIコンポーネントです。家系図のインタラクティブな表示エリア、選択された人物の詳細情報を表示するパネル（レスポンシブ対応）、アプリケーションバー、フローティングアクションボタン、およびナビゲーションドロワー（モバイル/タブレット時）など、主要なUI要素を統合します。

## クラス定義

```dart
// lib/features/family_tree/presentation/screens/main_screen.dart

class MainScreen extends ConsumerStatefulWidget {
  const MainScreen({Key? key}) : super(key: key);

  @override
  ConsumerState<MainScreen> createState() => _MainScreenState();
}

class _MainScreenState extends ConsumerState<MainScreen> {
  final GlobalKey<ScaffoldState> _scaffoldKey = GlobalKey<ScaffoldState>();
  final GlobalKey<_InteractiveFamilyTreeState> _interactiveTreeKey = GlobalKey<_InteractiveFamilyTreeState>();

  @override
  void initState() {
    super.initState();
    // アプリ起動時に初期化処理を実行
    WidgetsBinding.instance.addPostFrameCallback((_) {
      ref.read(mainScreenInitializationProvider.future).catchError((e, st) {
        // 初期化エラー時の処理
        if (mounted) {
          ScaffoldMessenger.of(context).showSnackBar(
            SnackBar(content: Text('アプリの初期化に失敗しました: $e'), backgroundColor: Colors.red),
          );
        }
      });
    });
  }

  @override
  Widget build(BuildContext context) {
    // 画面幅の更新
    // LayoutBuilderやMediaQuery.of(context).size.widthを適切な場所で監視し、
    // screenWidthProviderを更新するロジックが必要 (例: initStateやdidChangeDependencies)
    // ここでは簡略化のため、ビルド時に毎回取得する形とするが、最適化の余地あり
    final screenWidth = MediaQuery.of(context).size.width;
    // ref.read(mainScreenControllerProvider).updateScreenSize(screenWidth); // ビルド時の直接更新は避ける
    
    final deviceType = ref.watch(deviceTypeProvider);
    final selectedPersonId = ref.watch(selectedPersonIdProvider);

    return Scaffold(
      key: _scaffoldKey,
      appBar: _buildAppBar(context, ref, deviceType),
      body: _buildBody(context, ref, deviceType),
      floatingActionButton: _buildFloatingActionButton(context, ref, selectedPersonId),
      floatingActionButtonLocation: FloatingActionButtonLocation.centerDocked,
      bottomNavigationBar: deviceType != DeviceType.desktop ? _buildBottomAppBar(context, ref) : null,
      drawer: deviceType != DeviceType.desktop ? _buildAppDrawer(context, ref) : null,
    );
  }

  AppBar _buildAppBar(BuildContext context, WidgetRef ref, DeviceType deviceType) {
    return AppBar(
      title: const Text('ChartMyRoots'),
      leading: deviceType != DeviceType.desktop 
        ? IconButton(
            icon: const Icon(Icons.menu),
            onPressed: () => _scaffoldKey.currentState?.openDrawer(),
          )
        : null,
      actions: [
        const SyncStatusIndicator(),
        IconButton(
          icon: const Icon(Icons.swap_horiz_outlined),
          tooltip: '表示方向切替',
          onPressed: () => ref.read(familyTreeViewControllerProvider).toggleOrientation(),
        ),
        IconButton(
          icon: const Icon(Icons.zoom_out_map_outlined),
          tooltip: '全体表示',
          onPressed: () => ref.read(familyTreeViewControllerProvider).showEntireTree(_interactiveTreeKey.currentState?.context),
        ),
        if (deviceType == DeviceType.desktop) ...[
          IconButton(
            icon: const Icon(Icons.picture_as_pdf_outlined),
            tooltip: 'PDF出力',
            onPressed: () => _showPdfSettingsDialog(context, ref),
          ),
          IconButton(
            icon: const Icon(Icons.file_download_outlined),
            tooltip: 'エクスポート',
            onPressed: () => _showExportImportDialog(context, ref, isExport: true),
          ),
          IconButton(
            icon: const Icon(Icons.file_upload_outlined),
            tooltip: 'インポート',
            onPressed: () => _showExportImportDialog(context, ref, isExport: false),
          ),
        ],
        // TODO: 設定画面への導線など
      ],
    );
  }

  Widget _buildBody(BuildContext context, WidgetRef ref, DeviceType deviceType) {
    final treeLayoutAsync = ref.watch(treeLayoutProvider);
    final selectedPersonId = ref.watch(selectedPersonIdProvider);
    final viewPosition = ref.watch(viewPositionProvider);
    final zoomLevel = ref.watch(zoomLevelProvider);

    Widget treeView = treeLayoutAsync.when(
      data: (layout) {
        if (layout.isEmpty) {
          return const Center(child: Text('家系図データがありません。人物を追加してください。'));
        }
        return InteractiveFamilyTree(
          key: _interactiveTreeKey, // GlobalKeyを渡す
          layout: layout,
          selectedPersonId: selectedPersonId,
          initialPosition: viewPosition,
          initialScale: zoomLevel,
          onPersonTap: (personId) {
            ref.read(familyTreeViewControllerProvider).selectPerson(personId);
            if (deviceType != DeviceType.desktop) {
              _showPersonDetailBottomSheet(context, ref);
            }
          },
          onPan: (position) => ref.read(familyTreeViewControllerProvider).updatePosition(position),
          onScaleUpdate: (scale) => ref.read(familyTreeViewControllerProvider).setZoomLevel(scale),
        );
      },
      loading: () => const Center(child: CircularProgressIndicator()),
      error: (error, stack) => Center(child: Text('家系図の読み込みエラー: $error')),
    );

    if (deviceType == DeviceType.desktop) {
      return Row(
        children: [
          Expanded(child: treeView),
          SizedBox(
            width: 320, // サイドパネルの幅
            child: Card(
              elevation: 2,
              margin: const EdgeInsets.all(8.0).copyWith(left:0),
              child: const PersonDetailPanel(),
            ),
          ),
        ],
      );
    }
    return treeView; // モバイル・タブレットでは家系図表示のみ
  }

  Widget _buildFloatingActionButton(BuildContext context, WidgetRef ref, String? selectedPersonId) {
    return FloatingActionButton(
      tooltip: selectedPersonId != null ? '選択中の人物を編集' : '人物を追加',
      onPressed: () {
        if (selectedPersonId != null) {
          final person = ref.read(personByIdProvider(selectedPersonId)).value;
          if (person != null) {
            _showPersonEditDialog(context, ref, person: person, isNew: false);
          }
        } else {
          _showPersonEditDialog(context, ref, isNew: true);
        }
      },
      child: Icon(selectedPersonId != null ? Icons.edit_outlined : Icons.person_add_alt_1_outlined),
    );
  }

  BottomAppBar? _buildBottomAppBar(BuildContext context, WidgetRef ref) {
    return BottomAppBar(
      shape: const CircularNotchedRectangle(),
      notchMargin: 6.0,
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceAround,
        children: <Widget>[
          IconButton(icon: const Icon(Icons.add_circle_outline), tooltip: '人物追加', onPressed: () => _showPersonEditDialog(context, ref, isNew: true)),
          IconButton(icon: const Icon(Icons.picture_as_pdf_outlined), tooltip: 'PDF出力', onPressed: () => _showPdfSettingsDialog(context, ref)),
          const SizedBox(width: 48), // FABのためのスペース
          IconButton(icon: const Icon(Icons.ios_share_outlined), tooltip: 'エクスポート/インポート', onPressed: () => _showExportImportOptionsBottomSheet(context, ref)),
          IconButton(icon: const Icon(Icons.settings_outlined), tooltip: '設定', onPressed: () { /* TODO: 設定画面へ */ }),
        ],
      ),
    );
  }

  Drawer _buildAppDrawer(BuildContext context, WidgetRef ref) {
    return Drawer(
      child: ListView(
        padding: EdgeInsets.zero,
        children: <Widget>[
          DrawerHeader(
            decoration: BoxDecoration(color: Theme.of(context).primaryColor),
            child: Text('ChartMyRoots', style: Theme.of(context).primaryTextTheme.headlineSmall),
          ),
          ListTile(leading: const Icon(Icons.file_download_outlined), title: const Text('エクスポート'), onTap: () {
            Navigator.pop(context);
            _showExportImportDialog(context, ref, isExport: true);
          }),
          ListTile(leading: const Icon(Icons.file_upload_outlined), title: const Text('インポート'), onTap: () {
            Navigator.pop(context);
            _showExportImportDialog(context, ref, isExport: false);
          }),
          const Divider(),
          ListTile(leading: const Icon(Icons.info_outline), title: const Text('このアプリについて'), onTap: () { /* TODO: アプリ情報画面へ */ }),
        ],
      ),
    );
  }

  void _showPersonDetailBottomSheet(BuildContext context, WidgetRef ref) {
    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      shape: const RoundedRectangleBorder(borderRadius: BorderRadius.vertical(top: Radius.circular(16))),
      builder: (ctx) => DraggableScrollableSheet(
        initialChildSize: 0.5, minChildSize: 0.3, maxChildSize: 0.9, expand: false,
        builder: (_, controller) => SingleChildScrollView(controller: controller, child: const PersonDetailPanel()),
      ),
    );
  }

  void _showPersonEditDialog(BuildContext context, WidgetRef ref, {Person? person, required bool isNew}) {
    ref.read(personFormProvider.notifier).initWithPerson(person);
    showDialog(
      context: context,
      builder: (_) => ResponsivePersonEditDialog(child: PersonEditDialog(isNew: isNew)),
    );
  }

  void _showPdfSettingsDialog(BuildContext context, WidgetRef ref) {
    ref.read(pdfGenerationStateProvider.notifier).reset();
    final currentTree = ref.read(selectedFamilyTreeStreamProvider).value;
    ref.read(pdfSettingsProvider.notifier).reset(currentTree);
    showDialog(
      context: context,
      builder: (_) => ResponsivePdfSettingsDialog(child: PdfSettingsDialog()),
    );
  }
  
  void _showExportImportDialog(BuildContext context, WidgetRef ref, {required bool isExport}) {
    ref.read(exportImportStateProvider.notifier).reset();
    showDialog(
      context: context,
      builder: (_) => ResponsiveExportImportDialog( // レスポンシブ対応ラッパーを想定
        child: ExportImportDialog(isExport: isExport)
      ),
    );
  }

  void _showExportImportOptionsBottomSheet(BuildContext context, WidgetRef ref) {
    showModalBottomSheet(
      context: context,
      builder: (ctx) => Wrap(
        children: <Widget>[
          ListTile(leading: const Icon(Icons.file_download_outlined), title: const Text('エクスポート'), onTap: () {
            Navigator.pop(ctx);
            _showExportImportDialog(context, ref, isExport: true);
          }),
          ListTile(leading: const Icon(Icons.file_upload_outlined), title: const Text('インポート'), onTap: () {
            Navigator.pop(ctx);
            _showExportImportDialog(context, ref, isExport: false);
          }),
        ],
      ),
    );
  }
}
```

## プロパティ

なし（`ConsumerStatefulWidget`のため、プロパティは持たず、Riverpodプロバイダから状態を取得します）

## 内部状態

| 名前 | 型 | 説明 |
|------|------|------|
| `_scaffoldKey` | `GlobalKey<ScaffoldState>` | `Scaffold`の状態を管理し、ドロワーを開くために使用 |
| `_interactiveTreeKey` | `GlobalKey<_InteractiveFamilyTreeState>` | `InteractiveFamilyTree`ウィジェットのStateにアクセスし、外部から操作（例：全体表示）するために使用 |

## メソッド

### initState

```dart
void initState()
```

ウィジェットの初期化時に呼び出されます。アプリケーションの初期化処理（`mainScreenInitializationProvider`）を実行します。

### build

```dart
Widget build(BuildContext context)
```

ウィジェットのUIを構築します。`Scaffold`をルートとし、デバイスタイプ（デスクトップ、タブレット、モバイル）に応じてレスポンシブなレイアウトを構成します。

### _buildAppBar (プライベートヘルパー)

```dart
AppBar _buildAppBar(BuildContext context, WidgetRef ref, DeviceType deviceType)
```

アプリケーションバーを構築します。タイトル、ナビゲーションドロワーを開くボタン（モバイル/タブレット時）、各種アクションボタン（同期状態、表示方向切替、全体表示、PDF出力など）を含みます。

### _buildBody (プライベートヘルパー)

```dart
Widget _buildBody(BuildContext context, WidgetRef ref, DeviceType deviceType)
```

画面のメインコンテンツエリアを構築します。`InteractiveFamilyTree`ウィジェットで家系図を表示し、デスクトップの場合は右側に`PersonDetailPanel`を配置します。

### _buildFloatingActionButton (プライベートヘルパー)

```dart
Widget _buildFloatingActionButton(BuildContext context, WidgetRef ref, String? selectedPersonId)
```

フローティングアクションボタンを構築します。人物が選択されていれば編集アイコン、されていなければ人物追加アイコンを表示します。

### _buildBottomAppBar (プライベートヘルパー)

```dart
BottomAppBar? _buildBottomAppBar(BuildContext context, WidgetRef ref)
```

モバイル/タブレット時に表示されるボトムナビゲーションバーを構築します。主要なアクションへのショートカットを提供します。

### _buildAppDrawer (プライベートヘルパー)

```dart
Drawer _buildAppDrawer(BuildContext context, WidgetRef ref)
```

モバイル/タブレット時に表示されるナビゲーションドロワーを構築します。エクスポート/インポート機能やアプリ情報へのリンクを提供します。

### _showPersonDetailBottomSheet (プライベートヘルパー)

```dart
void _showPersonDetailBottomSheet(BuildContext context, WidgetRef ref)
```

モバイル/タブレット時に、選択された人物の詳細情報を表示するためのボトムシートを表示します。

### _showPersonEditDialog (プライベートヘルパー)

```dart
void _showPersonEditDialog(BuildContext context, WidgetRef ref, {Person? person, required bool isNew})
```

人物追加または編集ダイアログ（`PersonEditDialog`）を表示します。

### _showPdfSettingsDialog (プライベートヘルパー)

```dart
void _showPdfSettingsDialog(BuildContext context, WidgetRef ref)
```

PDF出力設定ダイアログ（`PdfSettingsDialog`）を表示します。

### _showExportImportDialog (プライベートヘルパー)

```dart
void _showExportImportDialog(BuildContext context, WidgetRef ref, {required bool isExport})
```

データのエクスポートまたはインポートダイアログ（`ExportImportDialog`）を表示します。

### _showExportImportOptionsBottomSheet (プライベートヘルパー)

```dart
void _showExportImportOptionsBottomSheet(BuildContext context, WidgetRef ref)
```

モバイル/タブレット時に、エクスポートとインポートの選択肢をボトムシートで表示します。

## 使用例

`MainScreen`は、アプリケーションのエントリポイント（例: `main.dart`内の`MaterialApp`の`home`プロパティ）として使用されます。

```dart
// main.dart
void main() {
  // Firebase初期化など
  // await Firebase.initializeApp(
  //   options: DefaultFirebaseOptions.currentPlatform,
  // );
  runApp(const ProviderScope(child: MyApp()));
}

class MyApp extends StatelessWidget {
  const MyApp({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'ChartMyRoots',
      theme: AppTheme.lightTheme, // AppThemeでテーマを定義
      darkTheme: AppTheme.darkTheme,
      themeMode: ThemeMode.system, // システム設定に追従
      home: const MainScreen(),
      // navigatorObservers: [ ... ], // 必要に応じて
      // localizationsDelegates: [ ... ], // 国際化対応
      // supportedLocales: [ ... ],
    );
  }
}
```

## 注意事項

- このウィジェットはアプリケーションの主要なUI構造を定義し、多くの機能を統合します。
- レスポンシブデザインは、`deviceTypeProvider`から取得した現在のデバイスタイプ（モバイル、タブレット、デスクトップ）に基づいてUI要素の表示/非表示やレイアウトを切り替えることで実現されます。
- 各種ダイアログ（人物編集、PDF設定など）は、このウィジェット内の適切なアクションから呼び出されます。ダイアログ自体はレスポンシブ対応ラッパーウィジェット（例: `ResponsivePersonEditDialog`）でラップされることを想定しています。
- `InteractiveFamilyTree`ウィジェットの`GlobalKey` (`_interactiveTreeKey`) は、外部から`InteractiveFamilyTree`のStateメソッド（例：全体表示のための`transformTo`）を呼び出すために使用されます。
- アプリケーションの初期化処理（`mainScreenInitializationProvider`）は、`initState`で非同期に実行され、エラーハンドリングも含まれます。
- 画面幅の変更をリアルタイムに検知し`screenWidthProvider`を更新するロジックは、`LayoutBuilder`や`WidgetsBinding.instance.addObserver`と`didChangeMetrics`などを組み合わせて実装することを検討してください。

## 関連するクラス・プロバイダ

- `deviceTypeProvider` (Riverpodプロバイダ) - 現在のデバイスタイプを提供
- `selectedPersonIdProvider` (Riverpodプロバイダ) - 選択中の人物IDを管理
- `treeLayoutProvider` (Riverpodプロバイダ) - 家系図のレイアウト情報を提供
- `familyTreeViewControllerProvider` (Riverpodプロバイダ) - 家系図表示操作を管理
- [InteractiveFamilyTree](./InteractiveFamilyTree.md) - 家系図のインタラクティブ表示ウィジェット
- [PersonDetailPanel](./PersonDetailPanel.md) - 人物詳細表示パネルウィジェット
- [SyncStatusIndicator](./SyncStatusIndicator.md) - データ同期状態表示ウィジェット
- [PersonEditDialog](./PersonEditDialog.md) - 人物編集ダイアログ
- [PdfSettingsDialog](./PdfSettingsDialog.md) - PDF設定ダイアログ
- `ExportImportDialog` (ウィジェット) - データエクスポート/インポートダイアログ (別途定義想定)
- `AppTheme` (ユーティリティクラス) - アプリのテーマ定義 (別途定義想定)
