# AI質問ログ：第4章 データの永続化

## 使用した生成AIツール

Claude

## 質問と回答の記録

### Q1

**質問：**
@Modelで定義しているclassは何をしているの？

**AIの回答の要点：**
`@Model`はSwiftDataに保存するデータの型を定義するマクロ。付けるだけで永続化・変更検知・UI通知が自動で行われる。

**自分の理解：**
RoomではEntity、DAO、Databaseを別々に書く必要があるが、SwiftDataでは`@Model`一つでそれらの役割をカバーできるもの

### Q2

**質問：**
@Queryは何をしている

**AIの回答の要点：**
`@Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]`の1行で、DB全件取得・ソート・データ変更時の自動更新が行われる。Androidでは DAO定義→SQLクエリ文→ViewModel→Flowの収集と4ステップ必要な処理が1行で済む。

**自分の理解：**
＠Modelと同じくdata層まで記述を跨ぐ処理をView側で処理を行ってくれるもの

### Q3

**質問：**
@Bindingと@Bindableの違いは

**AIの回答の要点：**
`@Binding`は値型（String, Boolなど）を親子ビュー間で共有するプロパティラッパー。`@Bindable`は`@Model`オブジェクトから各プロパティのバインディング（`$memo.title`）を生成するマクロ。`@Binding`だとプロパティごとに1つずつ渡す必要があるが、`@Bindable`ならオブジェクト1つ渡すだけで済む。

**自分の理解：**
MemoなどSwiftDataのオブジェクトを渡す場合は@Bindableで、ただの値を渡す場合は@Binding

## 今日の質問を振り返って
Swiftマクロとプロパティラッパーの実装について概要を知ることができた。
