# 「今のベストアングル」空間コーチカメラ ― Flutter本格アプリ化 設計書

ブラウザ試作版（単一HTML、`index.html`）の機能を、Flutterで iOS / Android 両対応のネイティブアプリにするための設計書です。試作版で検証した「構図ガイド × 水準器 × 音声ガイド × AIコーチ」の4要素を、そのままネイティブの強み（滑らかなセンサー処理、低遅延プレビュー、端末内保存）で磨き上げる方針です。

## 0. 試作版 → Flutter 機能対応表

| 試作版の機能 | Flutterでの実装先 |
|---|---|
| カメラプレビュー（背面・environment） | `camera` パッケージ、背面レンズ指定 |
| ダミー風景モード | アセット画像＋同じガイド描画を重ねる「練習モード」として移植 |
| 水準器（gamma、±2°で緑＋音声） | `sensors_plus` の加速度センサーから傾き角を算出 |
| 音声ガイド（読み上げ＋停止） | `flutter_tts`（ja-JP）、停止は `stop()` |
| 構図ガイド3モード（SVG） | `CustomPainter` で再描画（後述） |
| 巨匠の「型」オーバーレイ5種（フェルメール型／北斎・富士型／映画・シンメトリー型／映画・額縁型／逆光シルエット型、一言ガイドの表示と読み上げ付き） | `guide/painters/kata_painters.dart` に型ごとの Painter を追加し、モードガイドの上に重ねる |
| AIコーチ（Gemini、キー未設定時はデモ） | `google_generative_ai`、キーは `flutter_secure_storage` |
| 撮影→プレビュー→保存/共有 | `camera` の `takePicture`、`gal`（保存）、`share_plus`（共有） |

## 1. 推奨パッケージ構成（pubspec.yaml）

```yaml
dependencies:
  flutter:
    sdk: flutter
  camera: ^0.11.0            # カメラ制御（公式系）
  sensors_plus: ^6.0.0       # 加速度・ジャイロ取得
  flutter_tts: ^4.2.0        # 音声読み上げ
  google_generative_ai: ^0.4.0 # Gemini API（公式SDK）
  flutter_riverpod: ^2.5.0   # 状態管理
  flutter_secure_storage: ^9.2.0 # APIキーの安全な保存
  gal: ^2.3.0                # 写真ライブラリへ保存
  share_plus: ^10.0.0        # 共有シート
  path_provider: ^2.1.0      # 一時ファイル置き場
  permission_handler: ^11.3.0 # 権限リクエストの統一管理
```

選定理由：

- **camera**：Flutterチーム系の標準。プレビュー、撮影、画像ストリーム（後述の間引き処理用 `startImageStream`）が揃う。背面カメラの `CameraLensDirection.back` を指定。
- **sensors_plus**：加速度センサーを Dart 側で購読できる定番。水準器はジャイロ単体よりドリフトのない**加速度ベース**が安定する（試作版の `deviceorientation` 相当を、重力ベクトルからロール角に変換して再現）。
- **flutter_tts**：iOS の AVSpeechSynthesizer / Android の TextToSpeech を共通APIで扱える。言語 `ja-JP`、速度・停止（`stop()`）も素直に対応。
- **google_generative_ai**：Google公式の Dart SDK。モデル `gemini-2.5-flash` に画像＋プロンプトを送り、JSON応答（score / advice）を受ける試作版の方式をそのまま移植できる。
- **flutter_riverpod**：センサー値・モード・コーチ結果など「刻々と変わる状態」が多いアプリのため、テストしやすく画面再構築を局所化できる Riverpod を採用。`StateNotifierProvider` と `StreamProvider` の組み合わせ。
- **flutter_secure_storage**：APIキーを平文保存しないため（iOS Keychain / Android Keystore）。試作版の localStorage 保存からの格上げポイント。
- **gal / share_plus / path_provider**：保存と共有は OS 標準の作法に寄せる（iOS の流儀に沿った形になる）。
- **permission_handler**：カメラ・写真・マイク等の許可ダイアログを、理由説明UIとセットで一元管理する。

## 2. ディレクトリ構成案（機能単位）

```
lib/
  main.dart                  # 起動・Riverpod スコープ
  app.dart                   # MaterialApp・テーマ（ダーク基調）
  core/
    constants.dart           # 閾値（±2°）、モデル名、プロンプト文
    result.dart              # 成功/失敗の型（任意）
  features/
    camera/
      camera_controller_provider.dart  # camera の初期化・破棄
      camera_screen.dart               # 撮影画面（土台）
      widgets/guide_overlay.dart       # ガイド枠（CustomPainter 呼び出し）
      widgets/level_indicator.dart     # 水準器ウィジェット
    level/
      tilt_provider.dart       # 加速度→傾き角ストリーム（平滑化込み）
    guide/
      guide_mode.dart          # モード定義（花/人物/風景）
      painters/
        flower_guide_painter.dart      # 日の丸＋黄金比らせん
        portrait_guide_painter.dart    # 三分割＋シルエット
        landscape_guide_painter.dart   # 水平線＋一点透視
        level_painter.dart             # 水準器バーと泡
    coach/
      coach_state.dart         # スコア・アドバイス・判定中フラグ
      coach_service.dart       # Gemini 呼び出し／デモ判定の切り替え
      demo_coach.dart          # キー未設定・失敗時のデモロジック
    voice/
      voice_service.dart       # flutter_tts のラッパ（読み上げ・停止・ON/OFF）
    capture/
      capture_service.dart     # 撮影・間引きフレーム抽出・保存/共有
  settings/
    settings_screen.dart       # APIキー入力・音声ON/OFF・練習モード
assets/
  dummy_landscape.jpg          # 練習モード用ダミー風景
```

考え方：画面（features/camera）が土台となり、水準器（level）、ガイド（guide）、コーチ（coach）、音声（voice）はそれぞれ独立した feature として差し替え・テスト可能に保つ。試作版の「デモモード」は coach/demo_coach.dart に分離し、AIなしでもアプリが完結する線引きを維持する。

## 3. ネイティブ化における技術的ポイントと解決策

### 3.1 フレーム間引き処理（AI判定と静止検知）

- カメラの画像ストリームは 30fps で届くが、全フレーム処理は電池と発熱の無駄。**AI判定は「ボタン押下時」または「静止が続いたとき」に限定**する。
- 静止検知：加速度センサーの分散が小さい状態が **1.2秒** 続いたら「構えが安定」とみなす。あわせて傾きが ±2° 以内（水平OK）になった瞬間をトリガー候補にする。
- 自動コーチを使う場合でも、フレーム抽出は **2〜3秒に1枚** まで間引く。前回送ったフレームとの差分（縮小グレースケールの平均絶対差）が小さい場合は送信をスキップし、APIコストと通信を抑える。
- 画像変換（CameraImage の YUV420 → JPEG）はメインスレッドを塞ぐため、`Isolate`（または `compute`）で実行。送信サイズは長辺 768px・JPEG品質 0.8 を目安に試作版と揃える。

### 3.2 CustomPainter によるガイド枠と水準器の描画

- ガイドは `CustomPaint` をプレビューの上に重ね、`CustomPainter` で描く。サイズ依存の座標（比率ベース）で描けば、機種ごとの画面比に自動で追随する。
- 黄金比らせん：対数らせん `r = a・φ^(2θ/π)` を `Path` の折れ線近似で描画（試作版のSVG生成と同じ数式）。日の丸は二重円＋中心点。
- 三分割：比率 1/3・2/3 の線と交点ドット。シルエットは楕円（頭）＋ベジェ曲線（肩）の破線パス（`Path.computeMetrics` で破線化）。
- 一点透視：消失点から放射する線群＋水平線（破線）。水平線は水準器の傾きに連動して微回転させると「水平線を合わせる」体験が直感的になる（任意の磨き込み）。
- 水準器：バー・泡・OK時の発光を `level_painter.dart` に分離。傾き角は `tilt_provider` がローパスフィルタ（例：`θ' = θ' + 0.25(θ − θ')`）で平滑化し、描画は `RepaintBoundary` で囲って再描画範囲を限定する。
- `shouldRepaint` は「傾き or モードが変わったとき」だけ true にし、無駄な再描画を避ける。

### 3.3 権限設定（プラットフォーム固有）

**iOS（Info.plist）**

| キー | 文言の要点 |
|---|---|
| `NSCameraUsageDescription` | 構図ガイドと撮影のためにカメラを使用します |
| `NSMotionUsageDescription` | 水平チェックのためにモーションセンサーを使用します |
| `NSPhotoLibraryAddUsageDescription` | 撮影した写真をライブラリに保存します |
| `NSMicrophoneUsageDescription` | （将来の音声操作用。MVPでは入れない） |

- モーションは `sensors_plus` 経由でも、フィットネス系APIを使う場合は上記が必要になる場合がある。文言は、利用者が許可ダイアログで判断できるよう「何のために使うか」を具体的に書く。

**Android（AndroidManifest.xml）**

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
<uses-permission android:name="android.permission.HIGH_SAMPLING_RATE_SENSORS" />
<uses-feature android:name="android.hardware.camera" android:required="true" />
```

- 保存は MediaStore 経由（gal が吸収）なので、書き込み系の広い権限は要求しない。
- `HIGH_SAMPLING_RATE_SENSORS` はセンサーを高頻度で読む場合のみ。水準器は 20〜30Hz で十分なため、実測して不要なら外す。

**APIキーの扱い（重要）**

- 試作版と同じく「利用者が自分のキーを端末内に入れる」方式を基本とし、`flutter_secure_storage` に保存する。開発者のキーをアプリに埋め込んで配布しない（漏洩・課金事故の防止）。
- 将来、キー不要で使える形にする場合は、サーバー側プロキシ（キーはサーバー保管、レート制限付き）を別途用意する。本設計の MVP には含めない。

## 4. 開発ロードマップ（3段階）

**フェーズ1：MVP（撮影体験の核）**
- プロジェクト雛形、`camera` プレビュー＋撮影、水準器（sensors_plus＋平滑化）、3モードのガイド描画（CustomPainter）、flutter_tts の音声ガイド（水平OK通知・停止ボタン）、練習モード（ダミー風景）。
- 完了条件：実機（iPhone / Android 各1台）で「構える→水平→音声→撮る」が一連で気持ちよく動くこと。

**フェーズ2：AIコーチと仕上げ**
- `google_generative_ai` 連携（設定画面でキー入力、デモモード併設）、静止検知＋間引きによる自動/手動コーチ、撮影プレビュー（スコア表示付き）、保存（gal）と共有（share_plus）。
- 完了条件：キーあり/なし両方で破綻しないこと、AI失敗時にデモ判定へ自動で落ちること、オフラインでも撮影が完結すること。

**フェーズ3：公開と紹介（無料／ストア公開はしない）**
- 公開の形は確定済み：試作版は GitHub Pages で無料公開し、note の紹介記事で案内する（いつもの形）。Flutter版を作る場合もこの方針に合わせ、GitHub でソース一式（Android は APK も）を無料で配り、note で紹介する。
- 自分の端末で日常的に使うための仕上げもこの段階：アイコン・スプラッシュ整備、水準器の閾値と音声の頻度を実使用に合わせて調整、バッテリー消費の確認。身近な人に試してもらうのもここ。
- iOS はストア外配布の制約が大きいため、Flutter版の iOS 利用は自分の端末への開発用署名インストールが中心になる（証明書の有効期限切れに注意）。誰でもすぐ試せる入口は、ブラウザで開ける試作版（GitHub Pages）が担う。
- あわせて、APIキーと写真の扱い（キーは端末内のみ、写真は判定時のみ送信し保存しない）を README と紹介記事に明記しておく。
- 完了条件：試作版が GitHub Pages で公開され、note 紹介記事から辿って誰でも無料で試せること。Flutter版は、自分の端末で起動から撮影・保存まで日常の撮影にストレスなく使えること。

---

*本書は試作版（2026-11-18）の実装をもとに作成。試作版のURL・検証結果は開発記録を参照。*
