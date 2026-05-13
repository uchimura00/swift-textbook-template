# 第2章：地図アプリの基本

> 執筆者：内村 隼輔
> 最終更新：2026/5/8

## この章で学ぶこと

MapKitの概要から地図アプリの作成方法を学ぶ。
必要最低限動くコードの実装を正しく理解することで後半の応用で学ぶ内容をより深く理解できるようにする

## 模範コードの全体像
```swift
// ============================================
// 第2章（基本）：MapKitで地図を表示するアプリ
// ============================================
// 東京の観光スポットを地図上にマーカーで表示します。
// マーカーをタップすると詳細情報が表示されます。
// ============================================

import SwiftUI
import MapKit

// MARK: - データモデル

struct Landmark: Identifiable, Hashable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    static func == (lhs: Landmark, rhs: Landmark) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }

    enum Category: String, CaseIterable {
        case temple = "寺社"
        case tower = "タワー"
        case park = "公園"

        var iconName: String {
            switch self {
            case .temple: return "building.columns"
            case .tower: return "antenna.radiowaves.left.and.right"
            case .park: return "leaf"
            }
        }

        var color: Color {
            switch self {
            case .temple: return .red
            case .tower: return .blue
            case .park: return .green
            }
        }
    }
}

// MARK: - サンプルデータ

extension Landmark {
    static let sampleData: [Landmark] = [
        Landmark(
            name: "浅草寺",
            description: "東京都内最古の寺院。雷門が有名。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7148, longitude: 139.7967),
            category: .temple
        ),
        Landmark(
            name: "東京タワー",
            description: "1958年に完成した高さ333mの電波塔。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6586, longitude: 139.7454),
            category: .tower
        ),
        Landmark(
            name: "東京スカイツリー",
            description: "高さ634mの世界一高い自立式電波塔。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7101, longitude: 139.8107),
            category: .tower
        ),
        Landmark(
            name: "明治神宮",
            description: "明治天皇と昭憲皇太后を祀る神社。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6764, longitude: 139.6993),
            category: .temple
        ),
        Landmark(
            name: "上野恩賜公園",
            description: "美術館や動物園がある広大な公園。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7146, longitude: 139.7732),
            category: .park
        ),
        Landmark(
            name: "新宿御苑",
            description: "都心にある広さ58.3ヘクタールの庭園。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6852, longitude: 139.7100),
            category: .park
        ),
    ]
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var cameraPosition: MapCameraPosition = .region(
        MKCoordinateRegion(
            center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671),
            span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)
        )
    )
    @State private var selectedLandmark: Landmark?
    @State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

    var filteredLandmarks: [Landmark] {
        Landmark.sampleData.filter { selectedCategories.contains($0.category) }
    }

    var body: some View {
        ZStack(alignment: .bottom) {
            // 地図
            Map(position: $cameraPosition, selection: $selectedLandmark) {
                ForEach(filteredLandmarks) { landmark in
                    Marker(
                        landmark.name,
                        systemImage: landmark.category.iconName,
                        coordinate: landmark.coordinate
                    )
                    .tint(landmark.category.color)
                    .tag(landmark)
                }
            }
            .mapStyle(.standard(elevation: .realistic))

            // カテゴリフィルター
            VStack(spacing: 8) {
                if let landmark = selectedLandmark {
                    LandmarkCard(landmark: landmark)
                        .transition(.move(edge: .bottom))
                }

                CategoryFilter(selectedCategories: $selectedCategories)
            }
            .padding()
        }
        .onMapCameraChange { context in
            // 地図の操作に応じた処理を追加できる
        }
    }
}

// MARK: - カテゴリフィルター

struct CategoryFilter: View {
    @Binding var selectedCategories: Set<Landmark.Category>

    var body: some View {
        HStack(spacing: 8) {
            ForEach(Landmark.Category.allCases, id: \.self) { category in
                Button {
                    if selectedCategories.contains(category) {
                        selectedCategories.remove(category)
                    } else {
                        selectedCategories.insert(category)
                    }
                } label: {
                    HStack(spacing: 4) {
                        Image(systemName: category.iconName)
                        Text(category.rawValue)
                    }
                    .font(.caption)
                    .padding(.horizontal, 10)
                    .padding(.vertical, 6)
                    .background(
                        selectedCategories.contains(category)
                            ? category.color.opacity(0.2)
                            : Color.gray.opacity(0.1)
                    )
                    .foregroundStyle(
                        selectedCategories.contains(category)
                            ? category.color
                            : .gray
                    )
                    .clipShape(Capsule())
                }
            }
        }
        .padding(8)
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 16))
    }
}

// MARK: - ランドマーク詳細カード

struct LandmarkCard: View {
    let landmark: Landmark

    var body: some View {
        VStack(alignment: .leading, spacing: 6) {
            HStack {
                Image(systemName: landmark.category.iconName)
                    .foregroundStyle(landmark.category.color)
                Text(landmark.name)
                    .font(.headline)
                Spacer()
            }
            Text(landmark.description)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .padding()
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 12))
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

寺社　タワー 公園と三つのカテゴリ分けされたスポットをMapにマーカーを置いて表示するアプリ。
カテゴリのフィルタリングも可能。

## コードの詳細解説

### データモデル（ランドマーク構造体）

```swift
// MARK: - データモデル

struct Landmark: Identifiable, Hashable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    static func == (lhs: Landmark, rhs: Landmark) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }

    enum Category: String, CaseIterable {
        case temple = "寺社"
        case tower = "タワー"
        case park = "公園"

        var iconName: String {
            switch self {
            case .temple: return "building.columns"
            case .tower: return "antenna.radiowaves.left.and.right"
            case .park: return "leaf"
            }
        }

        var color: Color {
            switch self {
            case .temple: return .red
            case .tower: return .blue
            case .park: return .green
            }
        }
    }
}
```

**何をしているか：**
観光スポットの構造体をIdentifiableとHashableに準拠して定義し、enumでカテゴリ分けとUI用のプロパティを定義している。

**なぜこう書くのか：**
Mapのselectionや.tagに値を渡す際にHashableに準拠している必要があるが,coordinate が Equatableに準拠していないため、手動で == とhashを定義しないといけないため。
またidだけで同一性を判定したいという理由もあると思う。
またenumを構造台の中にネストして定義することで名前空間がLandmark.Categoryと整理される。

**もしこう書かなかったら：**
selection,.tagが使えなくなるため、タップしてする機能が使えなくなる。
またLandmark.Categoryと名前空間を整理できなくなる。

---

### 地図の表示とカメラ制御

```swift
// MARK: - メインビュー

struct ContentView: View {
    @State private var cameraPosition: MapCameraPosition = .region(
        MKCoordinateRegion(
            center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671),
            span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)
        )
    )
    @State private var selectedLandmark: Landmark?
    @State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

    var filteredLandmarks: [Landmark] {
        Landmark.sampleData.filter { selectedCategories.contains($0.category) }
    }

    var body: some View {
        ZStack(alignment: .bottom) {
            // 地図
            Map(position: $cameraPosition, selection: $selectedLandmark) {
                ForEach(filteredLandmarks) { landmark in
                    Marker(
                        landmark.name,
                        systemImage: landmark.category.iconName,
                        coordinate: landmark.coordinate
                    )
                    .tint(landmark.category.color)
                    .tag(landmark)
                }
            }
            .mapStyle(.standard(elevation: .realistic))

            // カテゴリフィルター
            VStack(spacing: 8) {
                if let landmark = selectedLandmark {
                    LandmarkCard(landmark: landmark)
                        .transition(.move(edge: .bottom))
                }

                CategoryFilter(selectedCategories: $selectedCategories)
            }
            .padding()
        }
        .onMapCameraChange { context in
            // 地図の操作に応じた処理を追加できる
        }
    }
}
```

**何をしているか：**
camerapositionでcenter(緯度経度から中心点)とspan(ズーム量)を設定している
その値をMapに渡し、styleはStandardで表示している

**なぜこう書くのか：**
Mapアプリはユーザーの操作によって位置座標などが変わるのでBindingで渡す必要がある。

**もしこう書かなかったら：**
ユーザー操作後の位置座標を取得できずに固まる。


---

### マーカーの表示

```swift
ForEach(filteredLandmarks) { landmark in
  Marker(
    landmark.name,
    systemImage: landmark.category.iconName,
    coordinate: landmark.coordinate
  )
  .tint(landmark.category.color)
  .tag(landmark)
}
                    
```

**何をしているか：**
選択されているカテゴリーの種類に属しているスポットをMaKerに渡して表示している。
Landmark構造体から位置情報やCategoryからUI描写に必要な値を受け取りそれを表示している。

**なぜこう書くのか：**
スポットをMap内に表示するという要件にMapkitが用意しているMakerを使用するのがあっているから。

**もしこう書かなかったら：**
Annotationなどを使用してデザインなど自分で実装しなければならなくなる

---

### フィルター機能

```swift
// MARK: - カテゴリフィルター

struct CategoryFilter: View {
    @Binding var selectedCategories: Set<Landmark.Category>

    var body: some View {
        HStack(spacing: 8) {
            ForEach(Landmark.Category.allCases, id: \.self) { category in
                Button {
                    if selectedCategories.contains(category) {
                        selectedCategories.remove(category)
                    } else {
                        selectedCategories.insert(category)
                    }
                } label: {
                    HStack(spacing: 4) {
                        Image(systemName: category.iconName)
                        Text(category.rawValue)
                    }
                    .font(.caption)
                    .padding(.horizontal, 10)
                    .padding(.vertical, 6)
                    .background(
                        selectedCategories.contains(category)
                            ? category.color.opacity(0.2)
                            : Color.gray.opacity(0.1)
                    )
                    .foregroundStyle(
                        selectedCategories.contains(category)
                            ? category.color
                            : .gray
                    )
                    .clipShape(Capsule())
                }
            }
        }
        .padding(8)
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 16))
    }
}
```

**何をしているか：**
selectedCategoriesに含まれているカテゴリーをそれぞれのButtonのクリックによって増減させ,removeされたカテゴリーはMakerから消えるように連動する

**なぜこう書くのか：**
コンポーネントとその起こすアクションを切り分けることで可読性をあげている。

**もしこう書かなかったら：**
一つのViewの中に複数のコンポーネントと複数のアクションが混在してしまうため可読性が下がる。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`Map` | SwiftUIで地図を表示するビューコンポーネント | `Map(position: .constant(.region(region)))` |
| 例：`Marker` | 地図上に位置をマーキングするコンポーネント | `Marker("名前", coordinate: coordinate)` |
| Implicit Member Expression| 型が文脈から推論できる場合に、型名を省略して .メンバー名 だけでアクセスできるSwiftの構文| @State private var cameraPosition: MapCameraPosition = .region(|
| | | |
| | | |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：spanの値を変えて思いっきり遠くする
- 結果： makerはズーム量に関係なく表示され続けた
- わかったこと：makerはズーム量に関係なく表示される

**実験2：**
- やったこと：Hashableを消す
- 結果：Mapやtagなどでコンパイルエラーが発生した
- わかったこと：Hashableに準拠していなければならないものがわかった

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** Hashableって何
   **得られた理解：** 値をハッシュ値に変換するプロトコル。Setなど内部にハッシュテーブルを持っている箇所に使うために必要

2. **質問：** .regionで始まる書き方に違和感ある
   **得られた理解：**　Swiftでは代入先の型がわかっている場合、型名を省略して.から書ける

3. **質問：** == と hash絶対いるの
   **得られた理解：** 全プロパティがEquatableに準拠していないと手動で書かないといけない

## この章のまとめ

MapKitで必要な構造体が準拠していなければならないprotocolを理解し、Mapアプリに最低限必要なプロパティなどを知ることができた。

# 応用編

## 模範コードの全体像
```swift
// ============================================
// 第2章（応用）：現在地を表示し、周辺検索する地図アプリ
// ============================================
// ユーザーの現在地を取得して地図上に表示し、
// 周辺のコンビニやカフェなどを検索する機能を追加します。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSLocationWhenInUseUsageDescription
//     値: "現在地を地図に表示するために位置情報を使用します"
// ============================================

import SwiftUI
import MapKit

// MARK: - 位置情報マネージャー

@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    let manager = CLLocationManager()
    var userLocation: CLLocationCoordinate2D?
    var authorizationStatus: CLAuthorizationStatus = .notDetermined

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
    }

    func requestPermission() {
        manager.requestWhenInUseAuthorization()
    }

    func startUpdating() {
        manager.startUpdatingLocation()
    }

    func stopUpdating() {
        manager.stopUpdatingLocation()
    }

    // MARK: - CLLocationManagerDelegate

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        userLocation = locations.last?.coordinate
    }

    func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
        authorizationStatus = manager.authorizationStatus

        switch authorizationStatus {
        case .authorizedWhenInUse, .authorizedAlways:
            startUpdating()
        default:
            break
        }
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var locationManager = LocationManager()
    @State private var cameraPosition: MapCameraPosition = .automatic
    @State private var searchResults: [MKMapItem] = []
    @State private var selectedCategory: String = "コンビニ"
    @State private var hasInitialSearched = false

    let searchCategories = ["コンビニ", "カフェ", "レストラン", "駅"]

    // CLLocationCoordinate2D は Equatable に準拠していないため、
    // onChange で監視できる文字列キーを派生させる
    private var userLocationKey: String? {
        locationManager.userLocation.map { "\($0.latitude),\($0.longitude)" }
    }

    var body: some View {
        ZStack(alignment: .top) {
            Map(position: $cameraPosition) {
                // 現在地のマーカー
                UserAnnotation()

                // 検索結果のマーカー
                ForEach(searchResults, id: \.self) { item in
                    if let name = item.name {
                        Marker(name, coordinate: item.placemark.coordinate)
                            .tint(.orange)
                    }
                }
            }
            .mapControls {
                MapUserLocationButton()
                MapCompass()
                MapScaleView()
            }

            // 検索カテゴリボタン
            VStack {
                categoryButtons
                    .padding(.top, 8)
                Spacer()
            }
        }
        .onAppear {
            locationManager.requestPermission()
        }
        .onChange(of: userLocationKey) { _, _ in
            guard let location = locationManager.userLocation else { return }
            cameraPosition = .region(
                MKCoordinateRegion(
                    center: location,
                    span: MKCoordinateSpan(latitudeDelta: 0.01, longitudeDelta: 0.01)
                )
            )
            // 初回の位置取得時に、選択中カテゴリで自動的に周辺検索を行う
            if !hasInitialSearched {
                hasInitialSearched = true
                Task { await searchNearby(query: selectedCategory) }
            }
        }
    }

    // MARK: - カテゴリボタン

    private var categoryButtons: some View {
        ScrollView(.horizontal, showsIndicators: false) {
            HStack(spacing: 8) {
                ForEach(searchCategories, id: \.self) { category in
                    Button {
                        selectedCategory = category
                        Task { await searchNearby(query: category) }
                    } label: {
                        Text(category)
                            .font(.subheadline)
                            .padding(.horizontal, 14)
                            .padding(.vertical, 8)
                            .background(
                                selectedCategory == category
                                    ? Color.blue
                                    : Color(.systemBackground)
                            )
                            .foregroundStyle(
                                selectedCategory == category
                                    ? .white
                                    : .primary
                            )
                            .clipShape(Capsule())
                            .shadow(color: .black.opacity(0.1), radius: 2)
                    }
                }
            }
            .padding(.horizontal)
        }
    }

    // MARK: - 周辺検索

    func searchNearby(query: String) async {
        guard let userLocation = locationManager.userLocation else { return }

        let request = MKLocalSearch.Request()
        request.naturalLanguageQuery = query
        request.region = MKCoordinateRegion(
            center: userLocation,
            span: MKCoordinateSpan(latitudeDelta: 0.02, longitudeDelta: 0.02)
        )

        do {
            let search = MKLocalSearch(request: request)
            let response = try await search.start()
            searchResults = response.mapItems
        } catch {
            print("検索エラー: \(error.localizedDescription)")
            searchResults = []
        }
    }
}

#Preview {
    ContentView()
}
```

**基本編との違い:**
- 位置情報を取得して表示している
  
**位置情報の許可フロー:**
- onAppearで画面表示時にlocationManager.requestPermission()が呼ばれ、ダイアログを出す。その結果によってlocationManagerDidChangeAuthorizationのswitch分によって分岐し、許可時にstartUpdatingLocation()が呼ばれ位置情報の取得を開始する。

**なぜ delegate パターン？:**
- CLLocationManagerの位置情報の通知方法がdelegateを使って通知するやり方だから

**MKLocalSearch とは:**
-  requestに自然言語でカテゴリ、エリアを指定し非同期でApple Mapsから情報を取得するAPI。

**もしこう書かなかったら:**
- 位置情報を取得できなくなるため、地図を遠く話した状態から始まり、周辺検索もできなくなる。

**AIとのやりとりで学んだこと:**
- CLLocationManagerDelegateに準拠するにはNSObjectのサブクラスである必要があって、NSObjectはclassなのでstructでは継承できない


