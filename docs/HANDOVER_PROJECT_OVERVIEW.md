# Time Logger Pro - プロジェクト概要書

**プロジェクト名:** Time Logger Pro (Permanent)  
**バージョン:** v10 (チェックポイント: fddfb3c1)  
**最終更新:** 2026年3月9日  
**ステータス:** 本番運用中

---

## 1. プロジェクト概要

Time Logger Proは、時間記録・時間管理を目的としたWebアプリケーションです。ユーザーは日々の活動を「カテゴリ」ごとに記録し、時間の使い方を可視化できます。

### 主な特徴

- **リアルタイムタイマー** — ブラウザを閉じてもタイマーが動作し続ける（LocalStorage + Service Worker）
- **クラウド永久保存** — 全記録がMySQLデータベースに永久保存
- **スマホ・PC統一UI** — レスポンシブ対応で同じコードベースで両デバイスに対応
- **24時間タイムライン** — 1日の時間配分を視覚的に確認
- **カテゴリ別集計** — 勉強、学校、移動時間など9つのカテゴリで時間を分類
- **スマート時間表示** — 60分以内は「分」メイン、60分超は「時間」メインで表示

---

## 2. 技術スタック

| レイヤー | 技術 | バージョン |
|---------|------|-----------|
| **フロントエンド** | React 19 + Tailwind CSS 4 | 最新 |
| **バックエンド** | Express 4 + tRPC 11 | 最新 |
| **データベース** | MySQL / TiDB | - |
| **ORM** | Drizzle ORM | 最新 |
| **認証** | Manus OAuth | 組み込み |
| **テスト** | Vitest | 最新 |
| **ビルド** | Vite | 最新 |

---

## 3. ファイル構成

```
time-logger-pro/
├── client/                          # フロントエンド
│   ├── src/
│   │   ├── pages/
│   │   │   ├── Home.tsx            # ランディングページ
│   │   │   └── TimeLogger.tsx       # メインアプリケーション
│   │   ├── components/
│   │   │   ├── TimerDisplay.tsx    # タイマー表示
│   │   │   ├── TimelineView.tsx    # 24時間タイムライン（タップ詳細表示対応）
│   │   │   ├── SummaryView.tsx     # 時間集計表示
│   │   │   ├── EditSessionModal.tsx # セッション編集モーダル
│   │   │   ├── DashboardLayout.tsx # ダッシュボードレイアウト
│   │   │   └── ui/                 # shadcn/ui コンポーネント
│   │   ├── lib/
│   │   │   ├── formatDuration.ts   # 時間表示ユーティリティ（60分判定ロジック）
│   │   │   ├── formatDuration.test.ts # 時間表示テスト（13テスト）
│   │   │   └── trpc.ts             # tRPCクライアント設定
│   │   ├── App.tsx                 # ルーティング・レイアウト
│   │   ├── main.tsx                # エントリーポイント
│   │   └── index.css               # グローバルスタイル
│   └── public/
│       └── sw.js                   # Service Worker
├── server/
│   ├── routers.ts                  # tRPC procedures（セッション CRUD）
│   ├── db.ts                       # データベースクエリヘルパー
│   ├── sessions.test.ts            # セッション機能テスト（5テスト）
│   ├── auth.logout.test.ts         # 認証テスト（1テスト）
│   └── _core/                      # フレームワーク内部
│       ├── index.ts                # サーバー起動
│       ├── context.ts              # tRPC context（ユーザー情報）
│       ├── oauth.ts                # OAuth処理
│       ├── env.ts                  # 環境変数定義
│       └── ...
├── drizzle/
│   └── schema.ts                   # データベーススキーマ
├── shared/
│   └── types.ts                    # 共有型定義
├── todo.md                         # 機能チェックリスト
├── vitest.config.ts                # Vitest設定
├── package.json
└── README.md
```

---

## 4. 主要な実装済み機能

### 4.1 時間記録機能
- **タイマー計測** — Start/Stop/Clearボタンで時間を計測
- **カテゴリ選択** — 9つのカテゴリから選択可能（勉強、学校、移動時間、ご飯、休憩、筋トレ、遊び、睡眠、その他）
- **メモ入力** — 何をしたかの詳細をテキストで記録
- **セッション保存** — 計測終了時にデータベースに自動保存

### 4.2 時間表示（v10新機能）
- **スマート時間表示** — 60分以内は「45分」、60分超は「1.5h」のように自動切り替え
- **詳細表示** — ホバー時に分単位での詳細表示（例: 1.8h → 105分）
- **複数フォーマット** — smart（スマート）、detailed（詳細）、compact（コンパクト）、full（完全）の4種類

### 4.3 データ表示
- **本日の集計** — 本日のカテゴリ別時間配分をカード表示
- **集計（本日）** — 全カテゴリの合計時間と内訳
- **24時間タイムライン** — 1日の時間配分を視覚的に表示
- **記録一覧** — 本日のセッション一覧（クリック/タップで編集可能）

### 4.4 タイムラインインタラクション（v10新機能）
- **PC: クリック編集** — タイムラインのセッションをクリックすると編集モーダルが開く
- **スマホ: タップ詳細表示** — タイムラインのセッションをタップするとポップオーバーが表示
  - カテゴリ名と色
  - 開始・終了時刻
  - 所要時間（詳細形式）
  - メモ内容
  - 「編集する」ボタン

### 4.5 セッション管理
- **新規作成** — `trpc.sessions.create`
- **更新** — `trpc.sessions.update`（時間、カテゴリ、メモを編集可能）
- **削除** — `trpc.sessions.delete`
- **一覧取得** — `trpc.sessions.list`（日付範囲フィルタ対応）

### 4.6 認証・ユーザー管理
- **Manus OAuth** — ログイン・ログアウト機能
- **セッション管理** — JWT Cookie自動管理
- **ユーザー情報** — メールアドレス表示

### 4.7 レスポンシブデザイン（v10新機能）
- **スマホ対応** — 全コンポーネントがモバイルサイズ（320px〜）で最適化
- **タブレット対応** — 中サイズ画面（768px〜）での2段レイアウト
- **PC対応** — 大画面（1024px〜）での3段レイアウト
- **フォントサイズ** — `text-xs sm:text-sm` のようにレスポンシブ指定

---

## 5. データベーススキーマ

### `users` テーブル
```sql
CREATE TABLE users (
  id VARCHAR(255) PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  name VARCHAR(255),
  role ENUM('user', 'admin') DEFAULT 'user',
  createdAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### `sessions` テーブル
```sql
CREATE TABLE sessions (
  id VARCHAR(36) PRIMARY KEY,
  userId VARCHAR(255) NOT NULL,
  startAt DATETIME NOT NULL,
  endAt DATETIME NOT NULL,
  durationMin INT NOT NULL,
  category VARCHAR(50) NOT NULL,
  note TEXT,
  createdAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updatedAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (userId) REFERENCES users(id) ON DELETE CASCADE,
  INDEX (userId, startAt)
);
```

---

## 6. tRPC API仕様

### セッション操作

**`trpc.sessions.list`** — セッション一覧取得
```typescript
Input: { from: Date, to: Date }
Output: Session[]
```

**`trpc.sessions.create`** — セッション作成
```typescript
Input: {
  startAt: Date,
  endAt: Date,
  durationMin: number,
  category: string,
  note?: string
}
Output: Session
```

**`trpc.sessions.update`** — セッション更新
```typescript
Input: {
  id: string,
  startAt: Date,
  endAt: Date,
  durationMin: number,
  category: string,
  note?: string
}
Output: Session
```

**`trpc.sessions.delete`** — セッション削除
```typescript
Input: { id: string }
Output: void
```

### 認証操作

**`trpc.auth.me`** — 現在のユーザー情報取得
```typescript
Output: { id: string, email: string, role: 'user' | 'admin' } | null
```

**`trpc.auth.logout`** — ログアウト
```typescript
Output: void
```

---

## 7. 環境変数

以下の環境変数は自動的に注入されます：

| 変数名 | 説明 |
|--------|------|
| `DATABASE_URL` | MySQL/TiDB接続文字列 |
| `JWT_SECRET` | セッション署名用シークレット |
| `VITE_APP_ID` | Manus OAuth アプリケーションID |
| `OAUTH_SERVER_URL` | OAuth認証サーバーURL |
| `VITE_OAUTH_PORTAL_URL` | ログインポータルURL |
| `OWNER_OPEN_ID` | プロジェクトオーナーのID |
| `OWNER_NAME` | プロジェクトオーナーの名前 |

---

## 8. テスト

### テストコマンド
```bash
pnpm test
```

### テストカバレッジ
- **formatDuration.test.ts** — 13テスト（時間表示ロジック）
- **sessions.test.ts** — 5テスト（CRUD操作）
- **auth.logout.test.ts** — 1テスト（ログアウト）
- **合計** — 19テスト（全てパス）

### テスト実行例
```bash
$ pnpm test
 ✓ client/src/lib/formatDuration.test.ts (13 tests) 8ms
 ✓ server/sessions.test.ts (5 tests) 3813ms
 ✓ server/auth.logout.test.ts (1 test) 5ms
 Test Files  3 passed (3)
      Tests  19 passed (19)
```

---

## 9. デプロイ

### 本番環境への公開
1. Manus UI の「Publish」ボタンをクリック
2. カスタムドメイン設定（オプション）
3. 自動デプロイ開始

### チェックポイント管理
```bash
# 現在のチェックポイント
manus-webdev://fddfb3c1

# 前のバージョン
manus-webdev://8b7b0a69  # v9 初期版
```

---

## 10. 開発ワークフロー

### 新機能追加の手順

1. **スキーマ更新** — `drizzle/schema.ts` を編集
2. **マイグレーション** — `pnpm db:push` を実行
3. **DB ヘルパー追加** — `server/db.ts` にクエリ関数を追加
4. **tRPC Procedure作成** — `server/routers.ts` に手続きを追加
5. **フロントエンド実装** — `client/src/pages/` または `client/src/components/` に UI を追加
6. **テスト作成** — `*.test.ts` ファイルにテストを記述
7. **テスト実行** — `pnpm test` で全テストを実行
8. **チェックポイント保存** — `webdev_save_checkpoint` で状態を保存

### 時間表示ロジックの使用例

```typescript
import { formatDuration } from "@/lib/formatDuration";

// スマート表示（60分判定自動）
formatDuration(45, "smart");   // "45分"
formatDuration(90, "smart");   // "1.5h"

// 詳細表示
formatDuration(90, "detailed"); // "1時間30分"

// 合計表示用
const { main, sub } = formatTotalDuration(150);
// main: "2.5h", sub: "(150分)"
```

---

## 11. 既知の制限事項

- **Service Worker対応ブラウザのみ** — Safari Private Browsing、Firefox Strict ETP では動作しない可能性
- **タイムゾーン** — 全時刻は UTC で保存、表示時にローカルタイムゾーンに変換
- **オフライン機能** — 現在、オンライン時のみ記録保存が可能

---

## 12. 今後の拡張案

1. **カスタムカテゴリ** — ユーザーが自分でカテゴリを作成・編集可能に
2. **グラフ表示** — 週・月ビューで棒グラフ・円グラフ表示
3. **目標設定** — カテゴリごとに時間目標を設定し、達成度を表示
4. **エクスポート** — CSV/PDF形式で記録をダウンロード
5. **複数デバイス同期** — リアルタイム同期の強化
6. **通知機能** — 特定時刻に通知を送信

---

**作成日:** 2026年3月9日  
**最終更新:** 2026年3月9日  
**次の開発者へ:** このドキュメントと `HANDOVER_CHANGELOG.md`、`HANDOVER_DEVELOPMENT_GUIDE.md` を参照して開発を進めてください。
