 # 第1章：WebAPIの基本

> 執筆者：（氏名）内村 隼輔
> 最終更新：2026/04/10

## この章で学ぶこと

iTunes Search APIを使って曲を検索し、その結果を表示するアプリを通して、WebAPIについての理解を深める。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第1章（基本）：iTunes Search APIで音楽を検索するアプリ
// ============================================
// このアプリは、iTunes Search APIを使って
// 音楽（曲）を検索し、結果をリスト表示します。
// APIキーは不要で、すぐに動かすことができます。
// ============================================

import SwiftUI

// MARK: - データモデル

struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?

    var id: Int { trackId }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var songs: [Song] = []
    @State private var searchText: String = ""
    @State private var isLoading: Bool = false

    var body: some View {
        NavigationStack {
            VStack {
                // 検索バー
                HStack {
                    TextField("アーティスト名を入力", text: $searchText)
                        .textFieldStyle(.roundedBorder)

                    Button("検索") {
                        Task {
                            await searchMusic()
                        }
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(searchText.isEmpty)
                }
                .padding(.horizontal)

                // 検索結果リスト
                if isLoading {
                    ProgressView("検索中...")
                        .padding()
                    Spacer()
                } else if songs.isEmpty {
                    ContentUnavailableView(
                        "曲を検索してみよう",
                        systemImage: "music.note",
                        description: Text("アーティスト名を入力して検索ボタンを押してください")
                    )
                } else {
                    List(songs) { song in
                        SongRow(song: song)
                    }
                }
            }
            .navigationTitle("Music Search")
        }
    }

    // MARK: - API通信

    func searchMusic() async {
        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else { return }

        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

        guard let url = URL(string: urlString) else { return }

        isLoading = true

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)
            songs = response.results
        } catch {
            print("エラー: \(error.localizedDescription)")
            songs = []
        }

        isLoading = false
    }
}

// MARK: - 曲の行ビュー

struct SongRow: View {
    let song: Song

    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                image
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } placeholder: {
                Color.gray.opacity(0.3)
            }
            .frame(width: 60, height: 60)
            .clipShape(RoundedRectangle(cornerRadius: 8))

            VStack(alignment: .leading, spacing: 4) {
                Text(song.trackName)
                    .font(.headline)
                    .lineLimit(1)

                Text(song.artistName)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }
        }
        .padding(.vertical, 4)
    }
}

#Preview {
    ContentView()
}
```

**このアプリは何をするものか：**

曲をキーワードで検索しその結果をリストで表示する
詳細画面やnavigationのないシンプルなアプリ

## コードの詳細解説

### データモデル（Codable構造体）

```swift
struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?

    var id: Int { trackId }
}
```

**何をしているか：**
iTunesAPIのJSONレスポンスを構造体に変換している

**なぜこう書くのか：**
JSONレスポンスを構造体にすることで扱いやすくする

**もしこう書かなかったら：**
JSONレスポンスを上手く変換することができない

---

### API通信の処理

```swift
func searchMusic() async {
        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else { return }

        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

        guard let url = URL(string: urlString) else { return }

        isLoading = true

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)
            songs = response.results
        } catch {
            print("エラー: \(error.localizedDescription)")
            songs = []
        }

        isLoading = false
    }
```

**何をしているか：**
検索欄に入力したテキストをURLに変換しAPIを叩いて、返ってきた結果を構造体に変換する処理を非同期で行っている。

**なぜこう書くのか：**
asyncを使って非同期処理を走らせることで検索中にUIの描写が止まることを防いでいる

**もしこう書かなかったら：**
検索処理中にUIの描写が止まり、UXが悪化する。
---

### ビューの構成

```swift
var body: some View {
        NavigationStack {
            VStack {
                // 検索バー
                HStack {
                    TextField("アーティスト名を入力", text: $searchText)
                        .textFieldStyle(.roundedBorder)

                    Button("検索") {
                        Task {
                            await searchMusic()
                        }
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(searchText.isEmpty)
                }
                .padding(.horizontal)

                // 検索結果リスト
                if isLoading {
                    ProgressView("検索中...")
                        .padding()
                    Spacer()
                } else if songs.isEmpty {
                    ContentUnavailableView(
                        "曲を検索してみよう",
                        systemImage: "music.note",
                        description: Text("アーティスト名を入力して検索ボタンを押してください")
                    )
                } else {
                    List(songs) { song in
                        SongRow(song: song)
                    }
                }
            }
            .navigationTitle("Music Search")
        }
    }
```

**何をしているか：**
画面の状態を`isLoading`と`songs`で分岐させており、検索確定ボタンも`searchText`が`isEmpty`かどうかで確定できるか分岐させるようにしている。
**なぜこう書くのか：**
ユーザーに今何が起こっているのかを明示的に示すことが必要だから。
**もしこう書かなかったら：**
何が起こっているのかユーザーに明示的に示さないので何が起きているのかわからずUXが悪化する。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`Codable` | JSONデータとSwiftの構造体を相互変換するプロトコル | `struct Song: Codable { ... }` |
| 例：`async/await` | 非同期処理を同期的に書ける構文 | `let data = try await URLSession.shared.data(from: url)` |
| `Task` | 同期->非同期処理を開始するための構造体| Task{ await searchMusic() } |
| | | |
| | | |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：ContentUnavailableViewをトレイリングクロージャーで書く
- 結果：正常に動作した
- わかったこと：クロージャーはラムダ式のようなものだと再確認した

**実験2：**
- やったこと：SongRowのAsyncImageのクロージャーの引数を$0で書く
- 結果：正常に動作した
- わかったこと：クロージャーの引数を$0で書けはするが、可読性が悪化するため、名前をつけた方が良いなと感じた

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** async awaitについて解説して
   **得られた理解：** asyncはこの関数は非同期であることを示し、awaitは非同期処理の完了を待つことを示すキーワードである

2. **質問：** ContentUnavailableViewって何
   **得られた理解：** コンテンツが表示できないことを示すSwiftUIフレームワークに含まれているビュー 

3. **質問：** struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?

    var id: Int { trackId }
}
のidの書き方は何
   **得られた理解：** iTunesAPIがtrackIdを返すがIdentifiableはidプロパティを要求するがCodableでデコードしたいため、idをcomputed propertyにしてtrackIdを返すようにしている

## この章のまとめ

async await構文の読み方やそれにあたってTaskなどの構造体の意味を知ることができた

## 応用編

**基本編との違い:** MVだったのがMVVMになっている。resultCount追加してるけど使われてない?
**MVVMとは何か:** Model View ViewModelの略。ViewModelはViewとModelの橋渡しのような役割をしており、View - ViewModel - Modelの形になっている。ViewはViewModelのイベントを呼び出し、ViewModelはそのイベントを受け取って,Modelの状態を変更している。そして変更された状態をViewModelが受け取り、Viewがその状態を監視している。これによって単方向データフロー(UDF)が成立している。ただしSwiftUIにおいてはViewModelの存在はAndroidとは違い、ライフサイクルの考慮などがないのでMVでもいいのでは...?と思っている。
**エラーハンドリング:** エラーハンドリングをviewModelがやっている。エラー自信がメッセージを持つんだと思った。
**もしこう書かなかったら:** 全然問題ないと思う。これぐらい小規模なら基本編のAPI通信をModel側でやるようにしてMVで書いてもいいと思う。
**AIとのやりとりで学んだこと:** ViewBuilderは複数の異なるViewを1つのView型にまとめることができる。

