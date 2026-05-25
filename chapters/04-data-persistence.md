# 第4章：データの永続化

> 執筆者：内村 隼輔
> 最終更新：2026-05-20

## この章で学ぶこと
iosアプリのデータの永続化についてMemoアプリを通して学ぶ

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第4章：データの永続化（AppStorage + SwiftData）
// ============================================
// シンプルなメモアプリで、2つの永続化方法を学びます。
// - AppStorage：アプリ設定の保存
// - SwiftData：構造化データの保存
// ============================================

import SwiftUI
import SwiftData

// MARK: - SwiftDataモデル

@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
    @AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
    @AppStorage("userName") private var userName: String = ""
    @State private var isShowingAddSheet = false
    @State private var isShowingSettings = false

    var displayedMemos: [Memo] {
        if sortByFavorite {
            return memos.sorted { $0.isFavorite && !$1.isFavorite }
        }
        return memos
    }

    var body: some View {
        NavigationStack {
            Group {
                if memos.isEmpty {
                    ContentUnavailableView(
                        "メモがありません",
                        systemImage: "note.text",
                        description: Text("右上の＋ボタンからメモを追加してください")
                    )
                } else {
                    List {
                        ForEach(displayedMemos) { memo in
                            NavigationLink(destination: MemoEditView(memo: memo)) {
                                MemoRow(memo: memo)
                            }
                        }
                        .onDelete(perform: deleteMemos)
                    }
                }
            }
            .navigationTitle(userName.isEmpty ? "メモ帳" : "\(userName)のメモ帳")
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    Button {
                        isShowingSettings = true
                    } label: {
                        Image(systemName: "gear")
                    }
                }
                ToolbarItem(placement: .topBarTrailing) {
                    Button {
                        isShowingAddSheet = true
                    } label: {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $isShowingAddSheet) {
                MemoAddView()
            }
            .sheet(isPresented: $isShowingSettings) {
                SettingsView(userName: $userName, sortByFavorite: $sortByFavorite)
            }
        }
    }

    func deleteMemos(at offsets: IndexSet) {
        for index in offsets {
            let memo = displayedMemos[index]
            modelContext.delete(memo)
        }
    }
}

// MARK: - メモの行

struct MemoRow: View {
    let memo: Memo

    var body: some View {
        HStack {
            VStack(alignment: .leading, spacing: 4) {
                Text(memo.title)
                    .font(.headline)

                Text(memo.content)
                    .font(.caption)
                    .foregroundStyle(.secondary)
                    .lineLimit(2)

                Text(memo.createdAt, style: .date)
                    .font(.caption2)
                    .foregroundStyle(.tertiary)
            }

            Spacer()

            if memo.isFavorite {
                Image(systemName: "star.fill")
                    .foregroundStyle(.yellow)
            }
        }
        .padding(.vertical, 2)
    }
}

// MARK: - メモ追加画面

struct MemoAddView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    @State private var title = ""
    @State private var content = ""

    var body: some View {
        NavigationStack {
            Form {
                Section("タイトル") {
                    TextField("メモのタイトル", text: $title)
                }
                Section("内容") {
                    TextEditor(text: $content)
                        .frame(minHeight: 200)
                }
            }
            .navigationTitle("新しいメモ")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        let memo = Memo(title: title, content: content)
                        modelContext.insert(memo)
                        dismiss()
                    }
                    .disabled(title.isEmpty)
                }
            }
        }
    }
}

// MARK: - メモ編集画面

struct MemoEditView: View {
    @Bindable var memo: Memo

    var body: some View {
        Form {
            Section("タイトル") {
                TextField("タイトル", text: $memo.title)
            }
            Section("内容") {
                TextEditor(text: $memo.content)
                    .frame(minHeight: 200)
            }
            Section {
                Toggle("お気に入り", isOn: $memo.isFavorite)
            }
        }
        .navigationTitle("メモを編集")
        .navigationBarTitleDisplayMode(.inline)
    }
}

// MARK: - 設定画面（AppStorageの活用）

struct SettingsView: View {
    @Binding var userName: String
    @Binding var sortByFavorite: Bool
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            Form {
                Section("ユーザー設定") {
                    TextField("あなたの名前", text: $userName)
                }
                Section("表示設定") {
                    Toggle("お気に入りを上に表示", isOn: $sortByFavorite)
                }
                Section {
                    Text("設定はアプリを閉じても保存されます")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .navigationTitle("設定")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .confirmationAction) {
                    Button("完了") { dismiss() }
                }
            }
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: Memo.self, inMemory: true)
}
```

**このアプリは何をするものか：**
メモをタイトルと内容だけ記入し保存し、保存した内容はHome画面と編集画面で確認できるシンプルなアプリ。
メモごとにお気に入りを登録することが可能で、ToolbarのSettingsViewでお気に入りを上に表示する並び替えをすることもできる。

## コードの詳細解説

### SwiftDataモデル（@Model）

```swift
@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}
```

**何をしているか：**
@ModelでSwiftDataに保存するデータの型を定義している。
＠Modelをつけることでこのクラスにデータの永続化・変更の自動検知・@queryでのSQL実行対象になる　機能を加える。
Androidと違いdaoやdatabaseクラスの定義が必要ない。
UI用のmodelと保村用のmodelを共通化するのが標準(?)らしい。

**なぜこう書くのか：**
データの永続化,変更の自動検知,SQLでの実行対象にするために必要なコードを自前で実装するのはボイラープレートコードが増えるため

**もしこう書かなかったら：**
自前でボイラープレートコードの実装をすることになる。

---

### データの追加・削除（modelContext）

```swift
// ── 追加（MemoAddView 内の保存ボタン）──
@Environment(\.modelContext) private var modelContext
 
Button("保存") {
    let memo = Memo(title: title, content: content)
    modelContext.insert(memo)
    dismiss()
}
.disabled(title.isEmpty)
 
// ── 削除（ContentView 内）──
func deleteMemos(at offsets: IndexSet) {
    for index in offsets {
        let memo = displayedMemos[index]
        modelContext.delete(memo)
    }
}
 
// ── リストのスワイプ削除（ContentView 内）──
.onDelete(perform: deleteMemos)
```

**何をしているか：**
@Environment(\.modelContext) でSwiftDataのデータベース窓口（modelContext）を取得して、@Model で定義したデータを insert（追加）/ delete（削除）している

**なぜこう書くのか：**
SwiftDataはViewから直接DB操作ができるため

**もしこう書かなかったら：**
modelContext経由でしかDBにアクセスできないため、データを保存したつもりでもできていなかったり、削除の操作をしても削除できていないことになる。

---

### @Queryによるデータ取得

```swift
@Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]

var displayedMemos: [Memo] {
    if sortByFavorite {
         return memos.sorted { $0.isFavorite && !$1.isFavorite }
    }
    return memos
}
```

**何をしているか：**
@QueryがSwiftDataのdbからmemoの配列を.reverseで降順で作成日(createdAt)の新しい順で自動取得し、その結果をViewと同期させている。
displayMemosはその結果をfavorite(お気に入り)が前に来るようにsortしている。

**なぜこう書くのか：**
手動でfetchを呼んだり、変更通知を監視するコードを書くのは無駄であり、swiftDataの機能で賄えるから。

**もしこう書かなかったら：**
データが変わるたびに値をfetchできずにviewとの同期もできなくなる

---

### @AppStorageによる設定保存

```swift
// ── 宣言（ContentView 内）──
@AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
@AppStorage("userName") private var userName: String = ""
 
// ── 使用例①：ナビゲーションタイトルへの反映 ──
.navigationTitle(userName.isEmpty ? "メモ帳" : "\(userName)のメモ帳")
 
// ── 使用例②：SettingsView で値を変更 ──
struct SettingsView: View {
    @Binding var userName: String
    @Binding var sortByFavorite: Bool
 
    var body: some View {
        NavigationStack {
            Form {
                Section("ユーザー設定") {
                    TextField("あなたの名前", text: $userName)
                }
                Section("表示設定") {
                    Toggle("お気に入りを上に表示", isOn: $sortByFavorite)
                }
            }
        }
    }
}
```

**何をしているか：**

**なぜこう書くのか：**

**もしこう書かなかったら：**

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`@Model` | SwiftDataでオブジェクトを永続化するためのマクロ | `@Model final class Memo { ... }` |
| 例：`@Query` | データベースからデータを取得し、変更を自動で反映するプロパティラッパー | `@Query var memos: [Memo]` |
| | | |
| | | |
| | | |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：
- 結果：
- わかったこと：

**実験2：**
- やったこと：
- 結果：
- わかったこと：

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   **得られた理解：**

2. **質問：**
   **得られた理解：**

3. **質問：**
   **得られた理解：**

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
