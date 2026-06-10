# 第5章：機能統合の実践

> 執筆者：内村 隼輔
> 最終更新：2006-06-03

## この章で学ぶこと
この章では、これまでに学んだカメラ・地図・データ保存の各機能を組み合わせて、「フォトマップ」アプリを実装する方法を学ぶ。具体的には撮影した写真をGPS位置情報と一緒に保存し、地図上に表示し、永続化したデータを検索・編集するアプリを題材にする。複数機能を統合するためのアーキテクチャ設計が重要になる。

## 模範コードの全体像

```swift
// ============================================
// 第5章：写真 + 地図 + データ保存の統合アプリ
// ============================================
// 写真を選択し、選択時の現在地を地図上に記録する
// 「フォトマップ」アプリです。
// 第2〜4章で学んだ技術を組み合わせて使います。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSLocationWhenInUseUsageDescription
//   - NSPhotoLibraryAddUsageDescription
// ============================================

import SwiftUI
import SwiftData
import MapKit
import PhotosUI

// MARK: - データモデル

@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date

    init(title: String, memo: String = "", latitude: Double, longitude: Double, imageData: Data? = nil) {
        self.title = title
        self.memo = memo
        self.latitude = latitude
        self.longitude = longitude
        self.imageData = imageData
        self.createdAt = .now
    }

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}

// MARK: - 位置情報マネージャー

@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    var currentLocation: CLLocationCoordinate2D?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        currentLocation = locations.last?.coordinate
    }
}

// MARK: - アプリエントリポイント
// ※ App ファイルに以下を記述：
//
// @main
// struct PhotoMapApp: App {
//     var body: some Scene {
//         WindowGroup {
//             ContentView()
//         }
//         .modelContainer(for: PhotoRecord.self)
//     }
// }

// MARK: - メインビュー（タブ構成）

struct ContentView: View {
    var body: some View {
        TabView {
            MapTab()
                .tabItem {
                    Label("マップ", systemImage: "map")
                }

            ListTab()
                .tabItem {
                    Label("一覧", systemImage: "list.bullet")
                }
        }
    }
}

// MARK: - マップタブ

struct MapTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query private var records: [PhotoRecord]
    @State private var locationManager = LocationManager()
    @State private var cameraPosition: MapCameraPosition = .automatic
    @State private var isShowingAddSheet = false
    @State private var selectedRecord: PhotoRecord?

    var body: some View {
        NavigationStack {
            ZStack(alignment: .bottomTrailing) {
                Map(position: $cameraPosition) {
                    UserAnnotation()

                    ForEach(records) { record in
                        Annotation(record.title, coordinate: record.coordinate) {
                            Button {
                                selectedRecord = record
                            } label: {
                                if let uiImage = record.uiImage {
                                    Image(uiImage: uiImage)
                                        .resizable()
                                        .aspectRatio(contentMode: .fill)
                                        .frame(width: 40, height: 40)
                                        .clipShape(Circle())
                                        .overlay(Circle().stroke(.white, lineWidth: 2))
                                        .shadow(radius: 2)
                                } else {
                                    Image(systemName: "photo.circle.fill")
                                        .font(.title)
                                        .foregroundStyle(.blue)
                                }
                            }
                        }
                    }
                }
                .mapControls {
                    MapUserLocationButton()
                }

                // 追加ボタン
                Button {
                    isShowingAddSheet = true
                } label: {
                    Image(systemName: "plus.circle.fill")
                        .font(.system(size: 56))
                        .foregroundStyle(.blue)
                        .background(Circle().fill(.white))
                        .shadow(radius: 4)
                }
                .padding(24)
            }
            .navigationTitle("フォトマップ")
            .sheet(isPresented: $isShowingAddSheet) {
                AddRecordView(locationManager: locationManager)
            }
            .sheet(item: $selectedRecord) { record in
                RecordDetailView(record: record)
            }
        }
    }
}

// MARK: - 一覧タブ

struct ListTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \PhotoRecord.createdAt, order: .reverse) private var records: [PhotoRecord]

    var body: some View {
        NavigationStack {
            List {
                ForEach(records) { record in
                    HStack(spacing: 12) {
                        if let uiImage = record.uiImage {
                            Image(uiImage: uiImage)
                                .resizable()
                                .aspectRatio(contentMode: .fill)
                                .frame(width: 50, height: 50)
                                .clipShape(RoundedRectangle(cornerRadius: 8))
                        }

                        VStack(alignment: .leading, spacing: 4) {
                            Text(record.title)
                                .font(.headline)
                            Text(record.createdAt, style: .date)
                                .font(.caption)
                                .foregroundStyle(.secondary)
                        }
                    }
                }
                .onDelete { offsets in
                    for index in offsets {
                        modelContext.delete(records[index])
                    }
                }
            }
            .navigationTitle("記録一覧")
        }
    }
}

// MARK: - 記録追加画面

struct AddRecordView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    let locationManager: LocationManager

    @State private var title = ""
    @State private var memo = ""
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImageData: Data?
    @State private var previewImage: Image?

    var body: some View {
        NavigationStack {
            Form {
                Section("写真") {
                    if let image = previewImage {
                        image
                            .resizable()
                            .aspectRatio(contentMode: .fit)
                            .frame(maxHeight: 200)
                            .clipShape(RoundedRectangle(cornerRadius: 8))
                    }

                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("写真を選択", systemImage: "photo")
                    }
                }

                Section("情報") {
                    TextField("タイトル", text: $title)
                    TextField("メモ（任意）", text: $memo, axis: .vertical)
                        .lineLimit(3...6)
                }

                Section("位置情報") {
                    if let location = locationManager.currentLocation {
                        Text("緯度: \(location.latitude, specifier: "%.4f")")
                        Text("経度: \(location.longitude, specifier: "%.4f")")
                    } else {
                        Text("位置情報を取得中...")
                            .foregroundStyle(.secondary)
                    }
                }
            }
            .navigationTitle("新しい記録")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        saveRecord()
                    }
                    .disabled(title.isEmpty || locationManager.currentLocation == nil)
                }
            }
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    if let data = try? await newItem?.loadTransferable(type: Data.self) {
                        selectedImageData = data
                        if let uiImage = UIImage(data: data) {
                            previewImage = Image(uiImage: uiImage)
                        }
                    }
                }
            }
        }
    }

    func saveRecord() {
        guard let location = locationManager.currentLocation else { return }

        let record = PhotoRecord(
            title: title,
            memo: memo,
            latitude: location.latitude,
            longitude: location.longitude,
            imageData: selectedImageData
        )
        modelContext.insert(record)
        dismiss()
    }
}

// MARK: - 記録詳細画面

struct RecordDetailView: View {
    let record: PhotoRecord

    var body: some View {
        ScrollView {
            VStack(spacing: 16) {
                if let uiImage = record.uiImage {
                    Image(uiImage: uiImage)
                        .resizable()
                        .aspectRatio(contentMode: .fit)
                        .clipShape(RoundedRectangle(cornerRadius: 12))
                }

                VStack(alignment: .leading, spacing: 8) {
                    Text(record.title)
                        .font(.title2)
                        .bold()

                    if !record.memo.isEmpty {
                        Text(record.memo)
                            .foregroundStyle(.secondary)
                    }

                    Text(record.createdAt, style: .date)
                        .font(.caption)
                        .foregroundStyle(.tertiary)
                }
                .frame(maxWidth: .infinity, alignment: .leading)

                // ミニマップ
                Map {
                    Marker(record.title, coordinate: record.coordinate)
                }
                .frame(height: 200)
                .clipShape(RoundedRectangle(cornerRadius: 12))
            }
            .padding()
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: PhotoRecord.self, inMemory: true)
}
```

**このアプリは何をするものか：**

自分の現在地の位置情報を付属した写真を登録するアプリ

## コードの詳細解説

### データモデルの設計
```swift
@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date

    init(title: String, memo: String = "", latitude: Double, longitude: Double, imageData: Data? = nil) {
        self.title = title
        self.memo = memo
        self.latitude = latitude
        self.longitude = longitude
        self.imageData = imageData
        self.createdAt = .now
    }

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}
```

**何をしているか：**
写真にの情報と位置情報を@Modelで定義している。

**なぜこう書くのか：**
写真を登録し、その情報を端末に記録するのにSwiftDataを利用するため＠Modelをつけている

**もしこう書かなかったら：**
自分でデータの永続化や変更の自動検知など@Modelが自動で行っているコードを自前で実装する必要になり、ミスによるバグの可能性が上がる。

---

### タブ構成の設計

```swift
struct ContentView: View {
    var body: some View {
        TabView {
            MapTab()
                .tabItem {
                    Label("マップ", systemImage: "map")
                }

            ListTab()
                .tabItem {
                    Label("一覧", systemImage: "list.bullet")
                }
        }
    }
}
```

**何をしているか：**
画面下のNavigationBarで遷移する画面を定義している。

**なぜこう書くのか：**
Tabで切り替える必要がある画面は二つしかなく、selectionで開く画面をtagで決められるが未指定で最初にTabViewに入れたViewが最初に表示されるようになっているため。

**もしこう書かなかったら：**
Tabarを利用した画面切り替えをiosで実装するにはこのような書き方しかないため、こう書くしかない


---

### カメラと位置情報の連携

```swift
@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    var currentLocation: CLLocationCoordinate2D?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        currentLocation = locations.last?.coordinate
    }
}

@State private var locationManager = LocationManager()
@State private var cameraPosition: MapCameraPosition = .automatic


```

**何をしているか：**
位置情報を管理するクラスをLocationManagerに切り出している。現在の位置情報をcurrentLocationで持っており、@StateでMapTabに渡し、画像に位置情報を付与している。

**なぜこう書くのか：**
Viewに直接CLLocationManagerを書くことも可能。しかしその場合、画面表示と位置情報取得という異なる責務が一つのViewに集中し、単一責任の原則に反する。また、位置情報機能を他の画面で再利用しにくくなり、保守性や拡張性も低下する。そのため本アプリではLocationManagerクラスとして切り出している。

**もしこう書かなかったら：**
Viewに責務が集中し、"なぜこう書くのか"で説明している通りの状況になる。


---

### SwiftDataでの画像保存

```swift
func saveRecord() {
        guard let location = locationManager.currentLocation else { return }

        let record = PhotoRecord(
            title: title,
            memo: memo,
            latitude: location.latitude,
            longitude: location.longitude,
            imageData: selectedImageData
        )
        modelContext.insert(record)
        dismiss()
    }
```

**何をしているか：**
modelContext.insert(record) により、作成した PhotoRecord をSwiftDataの管理対象として登録している。これによりデータの永続化や @Query による取得が可能になる。

**なぜこう書くのか：**
UserDefaultsでも永続化は可能だが、UserDefaultsで保存するデータは構造化されていない単純なデータ向けのもののため、SwiftDataを利用している。

**もしこう書かなかったら：**
UserDefaultsを使って構造化されているデータを保存する場合、Jsonで保存することになるため、encode decode処理が必要になってしまう。そのためSQliteのDBで保存できるSwiftDataを利用している。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`TabView` | 複数のビューをタブで切り替えるコンポーネント | `TabView { ... }.tabViewStyle(.page)` |
| 例：`CLLocationManager` | GPS位置情報を取得するAPIManager | `let location = manager.location?.coordinate` |
| MapCameraPosition = .automatic　| Map内に入っているAnotationが画面内に入るように地図の拡大縮小を自動で切り替えている　| Map(position: $cameraPosition) |
| | | |
| | | |

## 自分の実験メモ

**実験1：**
- やったこと： selectionにListTabのtagを指定してみる
- 結果：最初にListTabが表示されるようになった
- わかったこと：selectionにtagの値を指定するとそのTabが最初に表示されるようになる。Tabのコンポーネントに入っているViewの並び順はTabViewの中で定義した順番になる

**実験2：**
- やったこと：
- 結果：
- わかったこと：

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** @Observableをつける理由
   **得られた理解：** 位置情報の変更をViewが監視し、実際に状態が変更されたらViewを再描写するため。

2. **質問：** なぜ位置情報管理をViewで行わず、LocationManagerクラスとして分離しているのか。
   **得られた理解：**　Viewの責務は画面表示であり、位置情報取得などの処理の責務を分離することで、他のViewからも利用しやすく、保守性も向上し、単一責任の原則にも従った設計になるため。

3. **質問：** SwiftDataとUserDefaultsの違い
   **得られた理解：**　SwiftDataは構造化されたデータをSQLiteベースのDBに保存するためのもの、UserDefaultsはOn/offなど単純なデータを保存するためのもの。

## この章のまとめ
今まで学んできた内容を復習する形でSwiftDataをなぜ利用するのか、位置情報をどう取得するのか、StructとClassを切り替える理由などを学ぶことができた
