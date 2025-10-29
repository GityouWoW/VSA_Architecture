### Version: v2.0

# Vanilla SwiftUI Architecture (VSA)

以下は、次回以降あなた（アシスタント）に正しく意図を伝えるための「アーキテクチャ・プロンプト」です。プロジェクトでこのドキュメントを参照し、実装方針・コードスタイル・エラーポイントを統一します。

## 目的
- SwiftUI 2025 のデータフローに沿った、ViewModel不要（もしくは最小限）な設計をデフォルトにする。
- VSA_Architecture を尊重しつつ、Environment/State/Observable を核に据えた設計を徹底。
- Swift 6 の strict concurrency、初期化順序、プロパティラッパー運用、通信（WatchConnectivity含む）を安全に実装。
- 私がコピペで使える、実践的でエラーが少ないコードを提示すること。

---

## 基本原則

1) View = 状態の宣言的表現  
- Viewは軽量なstruct。状態は `@State`/`@Binding` を基本とし、ローカルUI状態はView内部で完結。  
- 複雑化したら「分割」する。ViewModelに逃がさない。複合Viewへ分解。

2) 依存性は Environment に注入  
- サービス層（APIクライアント、DB、ルーター等）は `@Environment(MyService.self)` または `@Environment(\.modelContext)` で注入。  
- Appエントリで `.environment(service)` を注入する初期化フローを確立。

3) データ取得・副作用は View ライフサイクルで  
- `.task(id:)`, `.onChange(of:)`, `.refreshable` を活用し、副作用を小さく局所化。  
- 共有状態を無理にViewModel化しない。必要ならサービス側に集中させる。

4) SwiftData/Observation を優先  
- SwiftData では `@Query`, `@Environment(\.modelContext)` を直接Viewで使う。  
- `@Observable`/`@Bindable` を使う場合も、「Environmentで取れるものをViewModelに持ち込まない」が原則。

5) テスト戦略  
- 単体テストはサービス・ビジネスロジック中心。  
- UIテストはE2E/スナップショット/Preview重視。必要なら ViewInspector を併用。

---

## 禁則・アンチパターン

- 二重の状態管理  
  - 同一責務で `@StateObject` と `@ObservedObject` を混在させない。  
  - ObservableObject内にさらに `@ObservedObject` を入れない。

- 不適切な状態変更  
  - `onAppear` で安易に初期値代入しない。初期化は `init`（安全な範囲）や `.task` を使う。  
  - SwiftDataの取得・更新をViewModelに移して `@Query` の自動反映を失わせない。

- 過度な階層化  
  - 不要な中間ViewModelや複雑な依存チェーン禁止。  
  - 煩雑になったらView分割かサービス分離で解消。

---

## Swift 6 / Concurrency 安全対策

- 俯瞰
  - actorの`init`中は`self`が隔離されておらず、actor-isolatedメソッド呼び出し禁止。`setup`は`init`外（`.task`など）で。  
  - `@MainActor` をUI境界に明示。バックグラウンド処理は `Task.detached` でなく適切にサービス側で。

- 典型エラー回避
  - `self used in method call '' before all stored properties are initialized`  
    - `init`で`self`を使う前に全プロパティ初期化を完了。`Task {}`で遅延起動。  
  - `.onScrollGeometryChange` の `Missing arguments for parameters 'for', 'action'`  
    - APIシグネチャに合わせ、必須パラメータ `for:action:` を全て渡す。  
  - `Reference to generic type 'ScrollViewReader' requires arguments in <...>`  
    - `ScrollViewReader` は `ScrollViewReader { proxy in ... }` の完全形で使用。

---

## 文字列補間・表示の注意

- `Text("\(Int(minutes))分")` のような場面で、誤ってエスケープして `Text("\\\(Int(minutes))分")` にしない。  
- ローカライズを想定する場合は `Text("\(minutes, format: .number)分")` か `String.localizedStringWithFormat` を使用。

---

## ObservableObject 設計ガイド（必要最小限で）

- 原則はView＋サービス。どうしても必要な場合のみ `@Observable`/`ObservableObject` を使用。  
- `ViewModel`を使う場合の制約
  - Environmentを直接保持しない。必要情報はViewから渡すか、サービスを注入（Environment→App注入、VM→init注入）。  
  - ライフサイクル副作用はView側でトリガーし、VMは純粋なステート変換に留める。  
  - `@StateObject` は所有者最上位Viewで1回。子は `@ObservedObject` or 値受け渡し。

---

## サービス層の基本

- 例：APIクライアントは `struct` か `final class`（スレッド安全確保）。  
- 依存はAppエントリで生成し `.environment(client)` で注入。  
- メソッドは `async throws` にしてUI側の`.task`から呼ぶ。

---

## SwiftData 運用指針

- 取得は `@Query` を最優先。  
- 変更は `@Environment(\.modelContext)` 経由で `insert/delete/save`。  
- 動的クエリは `init` で `Query(filter:sort:)` をセット。

---

## 実装テンプレ（実践用・コピペ可）

### 1) Appエントリでの依存注入

```swift
import SwiftUI
import SwiftData

@main
struct MyApp: App {
    @State private var client = BlueSkyClient.live()
    @State private var router = AppRouter(initialTab: .feed)
    @State private var auth = Auth()
    @State private var currentUser: CurrentUser?
    @Environment(\.scenePhase) private var scenePhase

    var body: some Scene {
        WindowGroup {
            RootView()
                .environment(client)
                .environment(router)
                .environment(auth)
                .environment(currentUser)
                .modelContainer(for: [RecentFeedItem.self])
                .task(id: auth.sessionLastRefreshed) {
                    await handleAuthStateChange()
                }
                .task(id: scenePhase) {
                    if scenePhase == .active { await auth.refresh() }
                }
        }
    }

    @MainActor
    private func handleAuthStateChange() async {
        if let cfg = auth.configuration {
            await refreshEnv(with: cfg)
            if router.presentedSheet == .auth { router.presentedSheet = nil }
        } else if auth.configuration == nil {
            router.presentedSheet = .auth
        }
    }

    @MainActor
    private func refreshEnv(with configuration: Auth.Configuration) async {
        client = BlueSkyClient.from(configuration)
        currentUser = await CurrentUser.load(using: client)
    }
}
```

変更点の意図:
- 依存を `@State` で保持し、`environment(_:)` で注入（ViewModel禁止）。
- 認証の副作用は `.task(id:)` で管理。

### 2) 典型的なView（ステート駆動 + サービス呼び出し）

```swift
import SwiftUI

struct FeedView: View {
    @Environment(BlueSkyClient.self) private var client
    @Environment(AppTheme.self) private var theme

    enum ViewState {
        case loading
        case error(String)
        case loaded([Post])
    }

    @State private var viewState: ViewState = .loading
    @State private var isRefreshing = false

    var body: some View {
        NavigationStack {
            List {
                switch viewState {
                case .loading:
                    ProgressView("Loading feed...")
                        .frame(maxWidth: .infinity)
                        .listRowSeparator(.hidden)
                case .error(let message):
                    ErrorStateView(message: message) {
                        await loadFeed()
                    }
                    .listRowSeparator(.hidden)
                case .loaded(let posts):
                    ForEach(posts) { post in
                        PostRowView(post: post)
                            .listRowInsets(.init())
                    }
                }
            }
            .listStyle(.plain)
            .refreshable { await refreshFeed() }
            .task { await loadFeed() }
            .navigationTitle("Feed")
        }
    }

    @MainActor
    private func loadFeed() async {
        viewState = .loading
        do {
            let posts = try await client.getFeed()
            viewState = .loaded(posts)
        } catch {
            viewState = .error(error.localizedDescription)
        }
    }

    @MainActor
    private func refreshFeed() async {
        defer { isRefreshing = false }
        isRefreshing = true
        await loadFeed()
    }
}
```

変更点の意図:
- 状態はView内 `enum`。副作用は `.task`/`refreshable`。  
- サービスはEnvironmentから直接参照。

### 3) SwiftData の自然な使い方（ViewModel禁止）

```swift
import SwiftUI
import SwiftData

struct BookListView: View {
    @Query private var books: [Book]
    @Environment(\.modelContext) private var modelContext
    @State private var showingAddBook = false

    var body: some View {
        NavigationStack {
            List {
                ForEach(books) { book in
                    BookRowView(book: book)
                        .swipeActions {
                            Button(role: .destructive) {
                                delete(book)
                            } label: {
                                Text("Delete")
                            }
                        }
                }
            }
            .navigationTitle("Books")
            .toolbar {
                Button("Add Book") { showingAddBook = true }
            }
            .sheet(isPresented: $showingAddBook) {
                AddBookView()
            }
        }
    }

    @MainActor
    private func delete(_ book: Book) {
        modelContext.delete(book)
        try? modelContext.save()
    }
}
```

変更点の意図:
- 取得は `@Query`、更新は `modelContext`。再取得や手動通知を不要化。

### 4) 動的クエリ（initでQuery設定）

```swift
import SwiftUI
import SwiftData

struct AuthorBooksView: View {
    let author: Author
    @Query private var books: [Book]

    init(author: Author) {
        self.author = author
        let predicate = #Predicate<Book> { $0.author?.id == author.id }
        _books = Query(filter: predicate, sort: \.title)
    }

    var body: some View {
        List(books) { book in
            BookRowView(book: book)
        }
        .navigationTitle(author.name)
    }
}
```

---

## よくある落とし穴と回避例

- ViewModel初期化の循環依存  
  - 回避: VMではなくサービスをEnvironment注入。必要ならViewの`init`で引数を束ねる。

- `.onScrollGeometryChange` の引数不足  
  - 正: `.onScrollGeometryChange(for: axes) { geometry in ... }`（使用APIに合わせて必須引数を明示）

- `ScrollViewReader` のジェネリクス警告  
  - 正:  
    ```swift
    ScrollViewReader { proxy in
        Button("Bottom") { withAnimation { proxy.scrollTo("bottom") } }
        // ...
    }
    ```

- 文字列補間の誤エスケープ  
  - 正: `Text("\(Int(minutes))分")`  
  - 誤: `Text("\\\(Int(minutes))分")`（バックスラッシュを入れない）

- actor初期化中の隔離違反  
  - 回避: `actor` の `init` 内ではactor-isolatedメソッドを呼ばない。初期化後に `.task` で `await setup()`。

---

## WatchConnectivity（sendMessage + replyHandler）注意

- `sendMessage` の `replyHandler` を使う場合、受信側で必ず  
  - `session(_:didReceiveMessage:replyHandler:)` を実装。未実装だと受信失敗。  
- スレッド/アクター境界に注意。UI更新は `@MainActor` で。

例（受信側）:
```swift
extension WatchSessionDelegate: WCSessionDelegate {
    func session(_ session: WCSession,
                 didReceiveMessage message: [String : Any],
                 replyHandler: @escaping ([String : Any]) -> Void) {
        // 必ずreplyHandlerを呼ぶ
        let result = ["ok": true]
        replyHandler(result)
    }
}
```

---

## コードスタイルと提出ルール（アシスタント向け）

- 言語: 主に日本語。  
- 変更点は箇条書きで「何を」「なぜ」変えたか明示。  
- 実践的な完全コードを提示（コピペ可）。Swift 6 準拠・strict concurrencyチェック通過を重視。  
- 既知エラー（補間のバックスラッシュ、ViewModel初期化順、`.onScrollGeometryChange` 引数不足、`ScrollViewReader` ジェネリクス、`self`初期化前使用）を都度スキャンし指摘・修正。  
- 新しいバージョン・APIを優先（2025時点）。古い回避策より公式推奨を採用。  
- VSA_Architecture との整合性を取り、必要ならセクション毎に「VSA適用ポイント」をコメントで明示。  
- 参考リンク（要求時に提示）:  
  - Data Flow Through SwiftUI（WWDC19）  
  - Discover Observation in SwiftUI（WWDC23）  
  - Data Essentials in SwiftUI（WWDC20）

---

## 思想の要点

- MVVMはSwiftUIのデータフローと相性が悪い場面が多い。  
- `@State`/`@Environment`/`@Observable`/`@Query` を信頼し、Viewを純粋な状態表現に保つ。  
- サービスに複雑性を押し込み、Viewは「構成と合成」に集中。  
- テストはサービス・モデル中心、UIはE2E/プレビューで十分。

---

このドキュメントを以後の会話の前提プロンプトとして扱い、矛盾する要求が来た場合は本方針に照らして是正案を提示してください。
