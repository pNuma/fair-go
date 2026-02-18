# fair-go（フェアに行こう）

Vue.jsの練習用プロジェクトです。
複数の参加者の郵便番号を入力することで、全員の中間地点（集合場所の目安）をとってもざっくり計算し、地図上に表示するWebアプリケーションです。

なお、実装およびライブラリ選定の調査には Google Gemini を使用しています。

こちらから利用できます ⇒ https://pnuma.github.io/fair-go/

## 機能
* **中間地点の算出**: 入力された地点の重心（中間地点）を計算し、マーカーを表示
* **結果の共有**: 入力された郵便番号をURLパラメータとして保持し、リンク共有が可能

## 使用技術

* Vue.js

## 使用API・データソース / ライセンス

本アプリケーションでは以下の外部サービス・データを利用しています。

### 地図データ
* **OpenStreetMap**
    * 地図の表示にOpenStreetMapのタイルデータを使用しています。
    * Copyright: © OpenStreetMap contributors
    * License: [ODbL](https://www.openstreetmap.org/copyright)

### 地図表示ライブラリ
* **Leaflet**
    * 地図の描画エンジンとして使用しています。
    * License: BSD 2-Clause License

### 住所・駅・座標検索API
* **HeartRails Geo API / HeartRails Express**
    * 郵便番号からの座標取得、座標からの住所・最寄り駅検索に使用しています。
    * [HeartRails Geo API](http://geoapi.heartrails.com/)
    * [HeartRails Express](http://express.heartrails.com/)
