# OBD スクレイパー — セットアップ・操作ガイド

OBDeleven のカスタマイズデータを取得し、画像の S3 保存・日本語翻訳・WordPress 投稿までを一括管理するパイプラインツールです。

---

## 目次

1. [システム要件](#1-システム要件)
2. [セットアップ手順](#2-セットアップ手順)
3. [環境変数の設定](#3-環境変数の設定)
4. [アプリケーションの起動](#4-アプリケーションの起動)
5. [パイプラインの流れ](#5-パイプラインの流れ)
6. [各ステップの詳細](#6-各ステップの詳細)
7. [生成されるファイル一覧](#7-生成されるファイル一覧)
8. [トラブルシューティング](#8-トラブルシューティング)

---

## 1. システム要件

| 項目 | 要件 |
|------|------|
| OS | Windows 10 / 11 |
| Python | 3.10 以上 |
| ネットワーク | インターネット接続必須 |
| 外部サービス | AWS S3、Google Gemini API、WordPress サイト |

> **EXE 版を使用する場合**: EXE にはランタイムが含まれていないため、Python のインストールが必要です。

---

## 2. セットアップ手順

### 2-1. Python 環境の構築（EXE 版を使わない場合）

```bash
# 仮想環境の作成
python -m venv env

# 仮想環境の有効化
env\Scripts\activate

# 依存ライブラリのインストール
pip install -r requirements.txt
```

### 2-2. 環境変数ファイルの作成

`.env.example` をコピーして `.env` を作成します。

```bash
copy .env.example .env
```

---

## 3. 環境変数の設定

`.env` ファイルを開き、以下の値を設定してください。

```ini
# ── AWS S3（画像保存先） ──
AWS_ACCESS_KEY_ID     = "AKIAxxxxxxxxxx"
AWS_SECRET_ACCESS_KEY = "xxxxxxxxxxxxxxxxxxxxxxxx"
AWS_REGION            = "us-east-1"
S3_BUCKET_NAME        = "your-bucket-name"

# ── Google Gemini API（翻訳用） ──
GEMINI_API_KEY = "AIzaSyxxxxxxxxxxxxxxxxx"

# ── WordPress ──
WP_BASE_URL     = "https://your-site.com/wp-json/wp/v2"
WP_USERNAME     = "your-wp-username"
WP_APP_PASSWORD = "xxxx xxxx xxxx xxxx xxxx xxxx"
```

### 各項目の説明

| 変数名 | 説明 |
|--------|------|
| `AWS_ACCESS_KEY_ID` | AWS IAM ユーザーのアクセスキー ID |
| `AWS_SECRET_ACCESS_KEY` | AWS IAM ユーザーのシークレットキー |
| `AWS_REGION` | S3 バケットのリージョン（例: `us-east-1`） |
| `S3_BUCKET_NAME` | 画像をアップロードする S3 バケット名 |
| `GEMINI_API_KEY` | Google AI Studio で発行した Gemini API キー |
| `WP_BASE_URL` | WordPress REST API のベース URL（末尾に `/wp-json/wp/v2`） |
| `WP_USERNAME` | WordPress の管理者ユーザー名 |
| `WP_APP_PASSWORD` | WordPress の「アプリケーションパスワード」（WordPress 管理画面 → ユーザー → アプリケーションパスワードで発行） |

---

## 4. アプリケーションの起動

### 方法 A: GUI ダッシュボード（推奨）

```bash
python server.py
```

または、コンパイル済みの EXE を使用：

```
OBD-Scraper.exe
```

ダッシュボードが起動し、5 つのステップをボタン操作で実行できます。

![GUI イメージ] ダークテーマのデスクトップアプリが開きます。

### 方法 B: コマンドラインで個別実行

各ステップを順番に手動で実行することもできます。

```bash
python main.py              # ステップ 1: スクレイピング
python compare.py           # ステップ 2: 差分比較
python download_images.py   # ステップ 3: 画像 DL → S3
python translate.py         # ステップ 4: 日本語翻訳
python wp.py                # ステップ 5: WordPress 投稿
```

---

## 5. パイプラインの流れ

```
OBDeleven.com
      │
      ▼
 ① スクレイピング (main.py)
      │  → output.json
      ▼
 ② 差分比較 (compare.py)
      │  → output-update.json（新規・更新のみ）
      ▼
 ③ 画像 DL & S3 アップロード (download_images.py)
      │  → output-update.json に S3 URL を追記
      ▼
 ④ 日本語翻訳 (translate.py)
      │  → output_jp.json
      ▼
 ⑤ WordPress 投稿 (wp.py)
      │  → WordPress サイトに投稿
      └  → wp_progress.json（進捗記録）
```

**重要**: ステップは必ず ①→②→③→④→⑤ の順番で実行してください。各ステップは前のステップの出力ファイルを入力として使用します。

---

## 6. 各ステップの詳細

### ① データスクレイピング (`main.py`)

- OBDeleven.com からカスタマイズデータを取得
- 車種（メーカー / モデル / 年式 / 世代）、タイトル、説明、画像 URL などを抽出
- 途中停止しても再開可能（既にスクレイピング済みの URL はスキップ）
- 50 件ごとに `output.json` へ自動保存

### ② 差分比較 (`compare.py`)

- `output.json`（最新）と `output-old.json`（前回分）を比較
- 新規データ (`_status: "new"`) と更新データ (`_status: "updated"`) を抽出
- 結果を `output-update.json` に保存

> **初回実行時**: `output-old.json` が無い場合は、`output.json` の全データが新規として扱われます。

### ③ 画像 DL & S3 アップロード (`download_images.py`)

- `output-update.json` 内の画像を元サイトからダウンロード
- AWS S3 に `メーカー/モデル/年式/ファイル名` の構造でアップロード
- 既に S3 に存在する画像はスキップ
- 各レコードに `image_new`（S3 URL）を追記

### ④ 日本語翻訳 (`translate.py`)

- Google Gemini AI を使用し、タイトル・説明・手順などを日本語に翻訳
- 20 件ずつのバッチ処理
- 途中停止しても再開可能
- 結果を `output_jp.json` に保存

### ⑤ WordPress 投稿 (`wp.py`)

- `output_jp.json` のデータを WordPress に投稿
- カスタム分類（車種・モデル・年式・世代）を自動作成
- スラッグで重複チェック（既存の投稿はスキップ）
- 50 件ごとに進捗を `wp_progress.json` に保存
- 途中停止しても再開可能

**コンソール記号の意味**:
| 記号 | 意味 |
|------|------|
| `✓` | 投稿成功 |
| `⊘` | 既に存在（スキップ） |
| `✗` | エラー発生 |

---

## 7. 生成されるファイル一覧

| ファイル名 | 説明 | 生成元 |
|-----------|------|--------|
| `output.json` | スクレイピング結果（全データ） | main.py |
| `output-old.json` | 前回のスクレイピング結果（比較用） | 手動リネーム |
| `output-update.json` | 新規・更新データのみ | compare.py |
| `output_jp.json` | 日本語翻訳済みデータ | translate.py |
| `wp_progress.json` | WordPress アップロード進捗 | wp.py |
| `wp_errors.json` | アップロード失敗レコード | wp.py |
| `errors.json` | スクレイピングエラーログ | main.py |

---

## 8. トラブルシューティング

### スクレイピングが途中で止まる

- ネットワーク接続を確認してください
- 自動リトライ機能がありますが、長時間の切断後は再度 `main.py` を実行してください
- 再開時は前回の続きから処理されます

### S3 アップロードに失敗する

- `.env` の `AWS_ACCESS_KEY_ID`、`AWS_SECRET_ACCESS_KEY`、`S3_BUCKET_NAME` が正しいか確認してください
- IAM ユーザーに S3 の `PutObject` / `HeadObject` 権限があるか確認してください

### 翻訳エラー

- `.env` の `GEMINI_API_KEY` が有効か確認してください
- API の利用上限に達している場合は、しばらく時間を置いてから再実行してください

### WordPress 投稿に失敗する

- `.env` の `WP_BASE_URL` が正しいか確認してください（末尾は `/wp-json/wp/v2`）
- WordPress ユーザーに投稿の作成権限があるか確認してください
- アプリケーションパスワードが正しいか確認してください（通常のログインパスワードでは動作しません）

### 途中から再開したい場合

各ステップには再開機能があります。同じコマンドを再度実行すれば、前回の続きから処理が再開されます。進捗ファイル（`wp_progress.json` 等）を削除すると、最初からやり直しになります。

---

## 補足

- `.env` ファイルには機密情報が含まれます。第三者に共有しないでください。
- 大量データ（数万件）の処理には長時間かかる場合があります。途中停止・再開が可能なので、分割して実行することもできます。
- WordPress のアプリケーションパスワードは WordPress 管理画面の「ユーザー → プロフィール → アプリケーションパスワード」から発行できます。
