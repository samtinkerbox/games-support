# DAU計測 横断設計

全アプリ横断でDAU（日次アクティブユーザー）を計測するための設計。DL数は伸びているがアクティブ度が見えていない課題への対応。

## 目的
- DL → DAU → 継続率 → 広告収益(ARPDAU) を1本の日次通知で追えるようにする
- 運用工数ゼロ（既存のDiscord日報基盤に相乗り）
- 量産ゲームに横展開できるテンプレ化

## 1. アーキテクチャ：1 Firebaseプロジェクトに全アプリ集約（確定）
- 全アプリ（iOS/Android各バンドル）を **1つのFirebaseプロジェクト → 1 GA4プロパティ** に登録
- 横断集計・Discord通知が **1スクリプトで完結**（GA4 Data APIを1回叩くだけ）
- 既存のGA4認証（memory: reference_ga4_auth）をそのまま流用
- アプリ別の内訳は user property `app_id` でbreakdown
- トレードオフ: AdMobリンクはアプリ単位。ARPDAUはGA4の `ad_impression` から算出するので実害小

## 2. 指標を3層に分ける
| 層 | 中身 | 実装コスト |
|---|---|---|
| L0 自動収集 | DAU/WAU/MAU・継続率(D1/D7)・セッション長・`ad_impression`→ARPDAU | SDK初期化のみ・コード追加ゼロ |
| L1 共通カスタム（全アプリ必須） | `game_start` / `session_complete` / `tutorial_complete` / `first_reward` | 少数に絞る＝横断比較の軸 |
| L2 アプリ固有 | `level_complete`・`merge_done` 等 | 各ゲーム任意 |

※ DAUだけなら L0 で足りる（`session_start` が自動収集）。L1は「どのゲームが刺さっているか」を横断で見る最小共通軸。

## 3. 命名・プロパティ規約
- イベント名は `snake_case`、全アプリ共通語彙（アプリごとに別名にしない）
- ユーザープロパティ必須3点: `app_id`（統一ID） / `app_version` / `platform`

## 4. 運用パイプライン（自動化の出口）
```
各アプリ → Firebase Analytics → GA4プロパティ
                                   ↓ GA4 Data API（既存認証流用）
                        日次バッチ: アプリ別DAU/継続率/ARPDAUを集計
                                   ↓
        既存Discord日報に相乗り（DL数=scripts/app_installs 7:15 / AdMob=scripts/admob_daily_digest 7:05）
```
- 新規: `scripts/app_dau/report.py`（想定）

## 5. 実装テンプレ
- Flutter: `brainrot_core/lib` に analytics wrapper（init + L1イベント関数）→ 依存する全brainrot系Flutterゲームを一括カバー
- 単体Flutter: iromaze 等に同wrapperを適用
- Godot: Firebaseプラグイン統合（AdMob統合と同要領。1回作れば量産テンプレ化）→ arena / street_hoops 系

## ロールアウト順（レバレッジ順）
1. Firebaseプロジェクト作成 + GA4プロパティ作成（1回）
2. brainrot_core に Flutter wrapper 実装 → brainrot系Flutterゲームに一括反映（最小工数で最大カバー）
3. iromaze（単体Flutter）に適用
4. GA4 Data API → Discord日報スクリプト（DAUが数字で届くようになる）
5. Godot テンプレ（arena / street_hoops）

## 確定リソース（samtinkerbox配下）
- GCPプロジェクト: `samtinker-analytics` / project number `403465150712`
- Firebase: 有効化済み（2026-07-08）
- GA4 プロパティID: `544663182`
- GA4 Analyticsアカウント ID: `392500611`
- 認証: gcloud=samtinkerbox@gmail.com。読み取りAPIはgcloudトークンで疎通OK。
  書き込み(addFirebase等)はgcloud共有OAuthクライアントが弾かれるためコンソール手動が必要だった。

## 登録済みアプリ（Firebase iosApps・13本）
全て project `samtinker-analytics` 配下。各 `GoogleService-Info.plist` 取得済。
Flutter(11): brainrot clicker/match/runner/sort/stack/td, brainrot-hunter, merge_monsters,
iromaze, speakdrill, surf-sim → いずれも `lib/firebase_options.dart` 生成＋main.dart配線＋
Podfile `platform :ios, '13.0'` 設定済。analyze全clean。
Godot(2): brainrot_arena / street_hoops → `scripts/Analytics.gd` autoload追加（GA4
Measurement Protocolクライアント、GDScript実装、構文チェック通過）。session_start+game_start
を起動時送信。`res://analytics_secret.txt`（gitignore済）にMP APIシークレットを置くと有効化、
無い間はno-op。ネイティブFirebase SDK不要。

## 実装メモ
- プラットフォーム設定はplistをXcodeに組込む方式ではなく `firebase_options.dart` コード生成方式
  （`Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform)`）。Xcode編集不要で機械適用可
- brainrot系6本は brainrot_core 経由でfirebase依存＋`Analytics.init()`。単体5本は直接firebase依存＋inline init
- DAU読み出し: `scripts/app_dau/report.py`（GA4 Data API、streamNameでアプリ別集計、Discord投稿）
  - 認証=taskdesklabと同じADC+SA impersonation(`task-analytics@short-video-factory-492413`)流用

## 残タスク
1. ⏳ **[要ユーザー操作] GA4プロパティ544663182にSAを閲覧者追加**（下記）→ digest疎通
2. iOSフルビルド検証（Firebase pods統合）
3. 各Flutterアプリの再ビルド＆再リリース → 実DAUデータ流入開始
4. ✅ Godot 2本(arena/street_hoops)=Analytics.gd実装済。要=各ストリームのMP APIシークレット作成
   →`res://analytics_secret.txt`配置で有効化（GA4管理→データストリーム→Measurement Protocol API secrets）
5. digest launchd登録（朝7:20想定）／Android google-services.json（military-fit等）

### 要ユーザー操作: GA4にSA追加
GA4管理 → プロパティ(544663182) → プロパティのアクセス管理 → 「+」→
`task-analytics@short-video-factory-492413.iam.gserviceaccount.com` を「閲覧者」で追加。
→ `python3 scripts/app_dau/report.py --dry-run` で疎通確認。

## ステータス
- 2026-07-08: 基盤+全Flutterアプリ配線完了(analyze clean)。SA権限付与→digest疎通OK・launchd登録(7:20)。
  Godot arena/street_hoops=Analytics.gd(GA4 Measurement Protocol)実装・実イベント着弾確認。
- 2026-07-09: **計測開始リリース実行（全公開アプリ）**。crowd(Godot・LIVE)も対応
  （Firebase登録/Analytics.gd/MPシークレット=GA4 UI自動化で作成/export include_filter修正）。
  Flutter5本(merge 1.1.6/hunter 1.0.9/iromaze 1.1.3/match 1.0.3/stacks 1.0.3)+crowd 1.3を
  build→upload→App PrivacyにANALYTICS行publish→審査提出。tycoon/war=未公開のためスキップ。
- 2026-07-17: **war対応**（7/10にLIVE化していたのに計測漏れ→Discord日報カバレッジ監査で発覚）。
  Firebase登録(appId 1:403465150712:ios:8c0deacc04ecf3db7d1b0d / stream 15271478347)→Analytics.gd移植→
  MPシークレット(GA4 UI自動化)→v1.1(8)提出済み(WAITING_FOR_REVIEW)。debug validation+/mp/collect 204+
  シークレット帰属UI照合まで確認。**GA4 realtimeは新規ストリームだと即時に出ない**（arenaの実イベントは
  同時刻に見えていた）→着弾最終確認は翌日以降の標準レポート/DAU日報で。
  残りの未計測=未公開のみ（tycoon/block/screw等。公開時に下のテンプレを適用）。

## Godot新作に計測を足す手順（テンプレ）
1. Firebase iosApps登録(API可) → appId取得
2. `scripts/Analytics.gd`をコピーしFIREBASE_APP_ID/APP_ID_LABEL変更、autoload登録
3. export_presets.cfg `include_filter="analytics_secret.txt"`（.txtはデフォルトでpckに入らない）
4. GA4データストリームのMP APIシークレット作成（chrome_session+authuser=1でUI自動化可）
   → `analytics_secret.txt`(gitignore)に配置
5. debug/mp/collectでvalidation→/mp/collectで204→GA4 realtimeのeventNameで着弾確認
