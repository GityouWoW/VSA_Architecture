
### Version: v1.0

# Vanilla SwiftUI Architecture (VSA)

Vanilla SwiftUI Architecture（VSA）は、SwiftUI本来のデータフローを最大限に活用し、ViewModelを使わずに「軽く・速く・読みやすく・壊れにくい」アプリを作るための実践アーキテクチャです。  
2025年のSwiftUI（@Observable/Environment/SwiftData/async）に最適化した設計で、余計な層やボイラープレートを排除します。

- 対象: iOS 17+（Swift 6）, SwiftUI, SwiftData（任意）
- キーコンセプト: Viewは純粋な状態表現。依存性はEnvironment注入。データはサービス層で完結。
- 禁止: ViewModel必須主義、二重状態管理、肥大化した中間レイヤ

目次
- 1. 原則
- 2. レイヤ構成と依存方向
- 3. コアパターン
- 4. スターター実装（コピペ可）
- 5. SwiftData統合例
- 6. テスト方針
- 7. よくある落とし穴と対策
- 8. 拡張（収益化・分析・リモート設定）
- 9. FAQ
- 10. ライセンス表記例

---

## 1. 原則

- Viewは純粋な状態表現（Pure State Expression）
  - UIは`@State`などのローカル状態とEnvironmentから導出される出力に限定
- 依存性はEnvironmentで注入
  - `@Environment(MyService.self)`や`@Environment(\.modelContext)`を使い、ViewModelに寄せない
- 副作用はViewの`.task(id:)`と`.onChange(of:)`で最小限にトリガ
- ビジネスロジックはサービス層に閉じ込める
  - ネットワーク、DB、変換、バリデーションはサービスで完了
- Concurrency安全
  - サービスは`Sendable`、UI更新は`MainActor.run`、グローバル共有を回避
- シンプル優先
  - MVVMの「VM」は作らない。必要性が出たらまずは「サブビュー分割」を検討

---

## 2. レイヤ構成と依存方向

- App（エントリ）
  - 依存性（サービス群）とアプリ状態（Auth/Router/Theme）を初期化しEnvironmentに注入
- Services
  - APIClient, AuthService, DatabaseService, Analytics, RemoteConfig, IAP など
  - Viewを参照しない。UIスレッドに依存しない。`Sendable`配慮
- Models
  - ドメインモデル（`struct`/`@Model`）。変換やバリデーション
- Views
  - `@State`はローカルUIだけ。データはEnvironment/`@Query`から取得
  - `.task(id:)`/`.onChange(of:)`で副作用を発火

依存方向: Views → Environment(Services) → Foundation/OS  
逆参照禁止（サービス→Viewは持たない）

---

## 3. コアパターン

- 型ベースEnvironment注入
  - `.environment(myService)`でツリー全体に注入
  - `@Environment(MyService.self) private var service`
- View内の状態表現
  - `enum ViewState { case loading, error(String), loaded([Item]) }`
- 副作用のトリガ
  - `.task { await fetch() }`
  - `.task(id: query) { await search(query) }`
  - `.onChange(of: someValue) { ... }`
- SwiftDataはViewで直接
  - `@Query`と`@Environment(\.modelContext)`で自然に使う

---

## 4. スターター実装（コピペ可）

以下は、VSAの最小スタータープロジェクト断片です。Swift 6のstrict concurrencyと典型エラーに配慮しています。

注意ポイント
- 文字列補間は`Text("\(value)分")`の形で、誤って`Text("\\(value)分")`としない
- `init`中にactor分離メソッドを呼ばない
- `ScrollViewReader`は`ScrollViewReader { proxy in ... }`で使う（ジェネリクスエラー回避）
- WatchConnectivityで`replyHandler`を使う場合、受信側で`session(_:didReceiveMessage:replyHandler:)`実装必須

コード

```swift
// VSA Starter - Version: v1.0

import SwiftUI
import Observation
import Foundation

// MARK: - Domain

public struct Post: Identifiable, Sendable, Equatable {
    public let id: UUID
    public let title: String
    public let body: String
    public let author: String

    public init(id: UUID = UUID(), title: String, body: String, author: String) {
        self.id = id
        self.title = title
        self.body = body
        self.author = author
    }
}

// MARK: - Services

public protocol APIClient: Sendable {
    func getFeed() async throws -> [Post]
}

public struct RealAPIClient: APIClient {
    public init() {}
    public func getFeed() async throws -> [Post] {
        // 実アプリではURLSession + JSONDecoderなどに置換
        try await Task.sleep(nanoseconds: 200_000_000)
        return [
            .init(title: "Hello SwiftUI 2025", body: "Forget MVVM.", author: "Apple Dev"),
            .init(title: "Testing Reality", body: "Test services, not views.", author: "Thomas")
        ]
    }
}

// 可観測なアプリ状態（Viewで使うがViewModelではない）
@Observable
public final class Auth: @unchecked Sendable {
    public enum State: Sendable { case signedOut, signedIn(userID: String) }
    public var state: State = .signedOut
    public var sessionLastRefreshed: Date = .distantPast

    public init() {}

    public func refresh() async {
        // 認証セッション更新（UI更新はMainActorで）
        await MainActor.run { sessionLastRefreshed = Date() }
    }

    public func signIn(userID: String) {
        state = .signedIn(userID: userID)
    }

    public func signOut() {
        state = .signedOut
    }
}

@Observable
public final class AppRouter: @unchecked Sendable {
    public enum Tab: Hashable, CaseIterable, Identifiable { case feed, settings
        public var id: Self { self } }
    public var selectedTab: Tab = .feed
    public var presentedSheet: Sheet?
    public enum Sheet: Identifiable { case auth
        public var id: String { String(describing: self) } }
    public init() {}
}

public struct AppTheme: Sendable, Equatable {
    public var accentColor: Color = .blue
    public init() {}
}

// MARK: - App Entry

@main
struct VSA_ExampleApp: App {
    // Environmentに注入するアプリ単位の状態とサービス
    @State private var apiClient: any APIClient = RealAPIClient()
    @State private var auth = Auth()
    @State private var router = AppRouter()
    @State private var theme = AppTheme()

    var body: some Scene {
        WindowGroup {
            RootView()
                .environment(apiClient)
                .environment(auth)
                .environment(router)
                .environment(theme)
                .task(id: auth.sessionLastRefreshed) {
                    switch auth.state {
                    case .signedOut:
                        router.presentedSheet = .auth
                    case .signedIn:
                        router.presentedSheet = nil
                    }
                }
        }
    }
}

// MARK: - Views

struct RootView: View {
    @Environment(AppRouter.self) private var router
    @Environment(Auth.self) private var auth

    var body: some View {
        TabView(selection: $router.selectedTab) {
            FeedScreen()
                .tag(AppRouter.Tab.feed)
                .tabItem { Label("Feed", systemImage: "list.bullet") }

            SettingsScreen()
                .tag(AppRouter.Tab.settings)
                .tabItem { Label("Settings", systemImage: "gearshape") }
        }
        .sheet(item: $router.presentedSheet) { sheet in
            switch sheet {
            case .auth:
                AuthView()
            }
        }
        .task(id: auth.sessionLastRefreshed) {
            // 認証状態が変わった時の追加副作用があればここで
        }
    }
}

struct FeedScreen: View {
    @Environment(AppTheme.self) private var theme
    @Environment(APIClient.self) private var api

    enum ViewState: Equatable {
        case idle
        case loading
        case error(String)
        case loaded([Post])
    }

    @State private var state: ViewState = .idle
    @State private var isRefreshing = false

    var body: some View {
        NavigationStack {
            content
                .navigationTitle("Feed")
                .tint(theme.accentColor)
        }
        .task {
            if case .idle = state {
                await loadFeed()
            }
        }
    }

    @ViewBuilder
    private var content: some View {
        switch state {
        case .idle, .loading:
            ProgressView("読み込み中…")
                .frame(maxWidth: .infinity, maxHeight: .infinity)
        case .error(let message):
            VStack(spacing: 12) {
                Text("エラー: \(message)")
                Button("再試行") {
                    Task { await loadFeed() }
                }
            }
            .frame(maxWidth: .infinity, maxHeight: .infinity)
        case .loaded(let posts):
            List {
                ForEach(posts) { post in
                    PostRow(post: post)
                        .listRowInsets(.init())
                }
            }
            .listStyle(.plain)
            .refreshable { await refresh() }
        }
    }

    private func loadFeed() async {
        await MainActor.run { state = .loading }
        do {
            let posts = try await api.getFeed()
            await MainActor.run { state = .loaded(posts) }
        } catch {
            await MainActor.run { state = .error(error.localizedDescription) }
        }
    }

    private func refresh() async {
        isRefreshing = true
        defer { isRefreshing = false }
        await loadFeed()
    }
}

private struct PostRow: View {
    let post: Post
    var body: some View {
        VStack(alignment: .leading, spacing: 6) {
            Text(post.title).font(.headline)
            Text(post.body).font(.subheadline)
            Text("by \(post.author)")
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .padding(.vertical, 8)
    }
}

struct SettingsScreen: View {
    @Environment(Auth.self) private var auth

    var body: some View {
        List {
            Section("Account") {
                switch auth.state {
                case .signedOut:
                    Button("Sign In") {
                        auth.signIn(userID: "demo-user")
                    }
                case .signedIn(let id):
                    Text("Signed in as \(id)")
                    Button("Sign Out", role: .destructive) {
                        auth.signOut()
                    }
                }
            }
        }
        .navigationTitle("Settings")
    }
}

struct AuthView: View {
    @Environment(Auth.self) private var auth
    @Environment(AppRouter.self) private var router
    @Environment(\.dismiss) private var dismiss
    @State private var userID: String = ""

    var body: some View {
        NavigationStack {
            Form {
                TextField("User ID", text: $userID)
                Button("Sign In") {
                    auth.signIn(userID: userID.isEmpty ? "anonymous" : userID)
                    router.presentedSheet = nil
                    dismiss()
                }
            }
            .navigationTitle("Sign In")
        }
    }
}
```

---

## 5. SwiftData統合例（Viewから直接）

```swift
import SwiftUI
import SwiftData

@Model
final class Book {
    @Attribute(.unique) var id: UUID
    var title: String
    var author: String

    init(id: UUID = UUID(), title: String, author: String) {
        self.id = id
        self.title = title
        self.author = author
    }
}

struct BookListView: View {
    @Query(sort: \.title) private var books: [Book]
    @Environment(\.modelContext) private var modelContext
    @State private var showingAdd = false

    var body: some View {
        List {
            ForEach(books) { book in
                Text("\(book.title) by \(book.author)")
                    .swipeActions {
                        Button("Delete", role: .destructive) {
                            modelContext.delete(book)
                            try? modelContext.save()
                        }
                    }
            }
        }
        .navigationTitle("Books")
        .toolbar { Button("Add") { showingAdd = true } }
        .sheet(isPresented: $showingAdd) { AddBookView() }
    }
}

struct AddBookView: View {
    @Environment(\.modelContext) private var ctx
    @Environment(\.dismiss) private var dismiss
    @State private var title = ""
    @State private var author = ""

    var body: some View {
        Form {
            TextField("Title", text: $title)
            TextField("Author", text: $author)
            Button("Save") {
                ctx.insert(Book(title: title, author: author))
                try? ctx.save()
                dismiss()
            }
        }
        .navigationTitle("New Book")
    }
}
```

---

## 6. テスト方針

- Unit
  - Services（API/DB/変換/認証）を徹底テスト
  - モデル変換・バリデーションを表明する
- UI
  - Previewsで視覚確認
  - E2E/UI Tests（XCTest/Accessibility Identifier）
  - 必要ならViewInspectorを補助的に使用

ビューは純粋表現なので、UI単体テストの効果は限定的。ロジックはサービス層で担保。

---

## 7. よくある落とし穴と対策

- 文字列補間のエスケープ
  - 正: `Text("\(Int(minutes))分")`／誤: `Text("\(Int(minutes))分")`
- `ScrollViewReader`のジェネリクスエラー
  - `ScrollViewReader { proxy in ... }`のクロージャ形式で使う
- `.onScrollGeometryChange`エラー
  - シグネチャ（`for: action:`）に合う拡張を使用。引数名を省略しない
- `self`を初期化前に使用
  - `init`中にactor隔離メソッド呼び出し禁止（Swift 6）。`@Observable`/actorの`init`に注意
- strict concurrency
  - サービス`Sendable`、UI更新`MainActor.run`、非同期のキャンセル安全性も考慮
- WatchConnectivity
  - `sendMessage`で`replyHandler`使用時、受信側は`session(_:didReceiveMessage:replyHandler:)`を必ず実装

---

## 8. 拡張（収益化・分析・リモート設定）

Analytics

```swift
public protocol AnalyticsService: Sendable {
    func log(_ event: String, params: [String: Sendable]?)
}

public struct ConsoleAnalytics: AnalyticsService {
    public init() {}
    public func log(_ event: String, params: [String: Sendable]?) {
        print("Analytics:", event, params ?? [:])
    }
}

// 注入例: .environment(analytics)
```

StoreKit2（IAP/サブスク）やRemoteConfigも同様にService化 → Environment注入 → Viewでトリガのみ。

---

## 9. FAQ

- Q: 大規模アプリでも本当にViewModel不要？
  - A: はい。状態はViewローカルに、ロジックはサービス側に。Viewが肥大化したら分割で対処。
- Q: テストはどうする？
  - A: サービス/モデルのユニットテストが主。UIはE2Eとプレビューで実運用品質を担保。
- Q: SwiftDataを必ず使う？
  - A: 任意。使う場合は`@Query`/`modelContext`をViewで直接扱うのがVSAの流儀。

---

## 10. ライセンス表記例

- 設計ドキュメント/テンプレコードはMIT等の緩いライセンス推奨
- 表記例:
  - “This app is built with Vanilla SwiftUI Architecture (VSA). Copyright © 2025.”

---

VSAは「SwiftUIを素で信じる」ための最小で強力な型。  
複雑さはサービスに閉じ込め、Viewは美しく小さく。  
Vanillaでいこう。
