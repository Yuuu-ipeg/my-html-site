# Time Logger Pro - アーキテクチャドキュメント

**バージョン:** v10  
**最終更新:** 2026年3月9日

---

## 1. システムアーキテクチャ

```
┌─────────────────────────────────────────────────────────────┐
│                    ブラウザ (クライアント)                    │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              React 19 フロントエンド                 │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │  TimeLogger.tsx (メインページ)                       │   │
│  │  ├─ TimerDisplay (タイマー表示)                     │   │
│  │  ├─ SummaryView (時間集計)                          │   │
│  │  ├─ TimelineView (24時間タイムライン)              │   │
│  │  └─ EditSessionModal (編集モーダル)                 │   │
│  │                                                      │   │
│  │  formatDuration.ts (時間表示ユーティリティ)         │   │
│  │  └─ 60分判定ロジック                                │   │
│  └──────────────────────────────────────────────────────┘   │
│                          ↕ (tRPC)                            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Vite + Tailwind CSS                     │   │
│  │  ├─ HMR (ホットモジュール置換)                      │   │
│  │  ├─ レスポンシブ UI (sm: ブレークポイント)          │   │
│  │  └─ shadcn/ui コンポーネント                        │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              LocalStorage + Service Worker          │   │
│  │  ├─ タイマー状態の永続化                            │   │
│  │  └─ オフライン対応（準備中）                        │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
└─────────────────────────────────────────────────────────────┘
                          ↕ (HTTPS)
┌─────────────────────────────────────────────────────────────┐
│                  Express サーバー (バックエンド)              │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              tRPC ルーター                           │   │
│  │  ├─ sessions.list                                   │   │
│  │  ├─ sessions.create                                 │   │
│  │  ├─ sessions.update                                 │   │
│  │  ├─ sessions.delete                                 │   │
│  │  ├─ auth.me                                         │   │
│  │  └─ auth.logout                                     │   │
│  └──────────────────────────────────────────────────────┘   │
│                          ↕                                    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Drizzle ORM                             │   │
│  │  ├─ スキーマ定義 (schema.ts)                        │   │
│  │  ├─ クエリビルダー                                  │   │
│  │  └─ マイグレーション                                │   │
│  └──────────────────────────────────────────────────────┘   │
│                          ↕                                    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Manus OAuth                             │   │
│  │  ├─ ログイン処理                                    │   │
│  │  ├─ JWT トークン管理                                │   │
│  │  └─ セッションクッキー                              │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
└─────────────────────────────────────────────────────────────┘
                          ↕ (SQL)
┌─────────────────────────────────────────────────────────────┐
│              MySQL / TiDB データベース                        │
├─────────────────────────────────────────────────────────────┤
│  ├─ users テーブル                                          │
│  ├─ sessions テーブル                                       │
│  └─ インデックス (userId, startAt)                          │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. データフロー

### 2.1 セッション作成フロー

```
ユーザー操作
    ↓
[Start] ボタンクリック
    ↓
タイマー開始 (LocalStorage に保存)
    ↓
[Stop & Save] ボタンクリック
    ↓
trpc.sessions.create() 呼び出し
    ↓
Express サーバー受信
    ↓
JWT トークン検証 → ユーザーID 取得
    ↓
Drizzle ORM で INSERT
    ↓
MySQL に保存
    ↓
レスポンス返却
    ↓
tRPC キャッシュ更新
    ↓
UI 再レンダリング
    ↓
toast.success() 表示
```

### 2.2 セッション一覧取得フロー

```
コンポーネントマウント
    ↓
trpc.sessions.list.useQuery() 実行
    ↓
Express サーバー受信
    ↓
JWT トークン検証 → ユーザーID 取得
    ↓
Drizzle ORM で SELECT
    ↓
MySQL クエリ実行
    ↓
結果を JSON に変換（SuperJSON）
    ↓
レスポンス返却
    ↓
tRPC キャッシュに保存
    ↓
UI レンダリング
    ↓
TimelineView, SummaryView に表示
```

### 2.3 時間表示フロー

```
セッションデータ取得 (durationMin: 105)
    ↓
formatDuration(105, 'smart') 呼び出し
    ↓
105 > 60 ? → true
    ↓
時間に変換: 105 / 60 = 1.75
    ↓
小数点第1位まで: 1.8
    ↓
"1.8h" を返却
    ↓
UI に表示
```

---

## 3. コンポーネント設計

### 3.1 TimeLogger.tsx (メインコンポーネント)

**責務:**
- ユーザー認証状態の管理
- タイマー状態の管理
- 日付・モード（日/週/月）の管理
- tRPC クエリの実行
- 子コンポーネントへのデータ渡し

**状態:**
```typescript
const [running, setRunning] = useState(false);           // タイマー実行中
const [startAt, setStartAt] = useState<Date | null>(null); // 開始時刻
const [elapsedSeconds, setElapsedSeconds] = useState(0); // 経過秒数
const [selectedDate, setSelectedDate] = useState(...);    // 選択日付
const [mode, setMode] = useState<Mode>("day");           // 表示モード
const [selectedCategory, setSelectedCategory] = useState("study"); // カテゴリ
const [note, setNote] = useState("");                    // メモ
const [editingSession, setEditingSession] = useState<Session | null>(null); // 編集中セッション
```

**子コンポーネント:**
- TimerDisplay
- SummaryView
- TimelineView
- EditSessionModal

### 3.2 TimelineView.tsx (タイムライン)

**責務:**
- 24時間タイムラインの描画
- セッションの位置計算
- PC: クリック → 編集モーダル
- スマホ: タップ → ポップオーバー

**主要ロジック:**

```typescript
// 時間から Y 座標を計算
const yFromTime = (d: Date | string) => {
  const dateObj = typeof d === "string" ? new Date(d) : d;
  const minutes = (dateObj.getTime() - dayStart.getTime()) / 60000;
  return (minutes / (24 * 60)) * TOTAL_HEIGHT;
};

// デバイス判定
const isTouchDevice = "ontouchstart" in window || navigator.maxTouchPoints > 0;

// セッションタップ処理
const handleSessionTap = (session: Session, e: React.MouseEvent | React.TouchEvent) => {
  if (isTouchDevice) {
    setPopover({ session, x, y }); // ポップオーバー表示
  } else {
    onSessionClick(session); // 編集モーダル表示
  }
};
```

**ポップオーバー:**
- カテゴリ表示（色付きドット）
- 時間表示（詳細形式）
- メモ表示
- 編集ボタン

### 3.3 SummaryView.tsx (集計表示)

**責務:**
- 本日の時間集計表示
- カテゴリ別の時間表示
- スマート時間表示（60分判定）

**表示内容:**
```
本日の集計
合計: 3.5h (213分)

[勉強]     [学校]     [移動時間]
1.8h       0分        53分
105分                  0.9h

[ご飯]     [休憩]     [筋トレ]
0分        0分        0分

[遊び]     [睡眠]     [その他]
0分        0分        55分
                       0.9h
```

### 3.4 TimerDisplay.tsx (タイマー表示)

**責務:**
- タイマーの時刻表示（HH:MM:SS）
- 開始・終了時刻の表示

**表示内容:**
```
00:00:00
未計測

または

00:05:30
11:25 〜 ...（計測中）
```

### 3.5 EditSessionModal.tsx (編集モーダル)

**責務:**
- セッションの詳細編集
- 時間の変更
- カテゴリの変更
- メモの編集
- 削除機能

**フォーム:**
- 開始時刻（datetime-local）
- 終了時刻（datetime-local）
- 所要時間（自動計算）
- カテゴリ（Select）
- メモ（Textarea）

---

## 4. 状態管理戦略

### 4.1 ローカル状態 (useState)

```typescript
// TimeLogger.tsx
const [running, setRunning] = useState(false);
const [selectedDate, setSelectedDate] = useState(...);
const [editingSession, setEditingSession] = useState<Session | null>(null);

// TimelineView.tsx
const [popover, setPopover] = useState<PopoverState | null>(null);
```

### 4.2 サーバー状態 (tRPC)

```typescript
// セッション一覧
const { data: sessionList = [] } = trpc.sessions.list.useQuery(
  { from: windowDates.from, to: windowDates.to },
  { enabled: isAuthenticated }
);

// ユーザー情報
const { user } = useAuth();
```

### 4.3 永続化 (LocalStorage)

```typescript
// タイマー状態の保存
localStorage.setItem(STORAGE_KEY, JSON.stringify({
  running,
  startAt: startAt.toISOString(),
  selectedCategory,
  note,
}));

// 復元
const saved = localStorage.getItem(STORAGE_KEY);
if (saved) {
  const state = JSON.parse(saved);
  // 復元処理
}
```

---

## 5. API 仕様

### 5.1 tRPC Router 構造

```typescript
export const appRouter = router({
  sessions: {
    list: protectedProcedure
      .input(z.object({ from: z.date(), to: z.date() }))
      .query(async ({ ctx, input }) => {
        // 実装
      }),
    
    create: protectedProcedure
      .input(z.object({
        startAt: z.date(),
        endAt: z.date(),
        durationMin: z.number(),
        category: z.string(),
        note: z.string().optional(),
      }))
      .mutation(async ({ ctx, input }) => {
        // 実装
      }),
    
    update: protectedProcedure
      .input(z.object({
        id: z.string(),
        startAt: z.date(),
        endAt: z.date(),
        durationMin: z.number(),
        category: z.string(),
        note: z.string().optional(),
      }))
      .mutation(async ({ ctx, input }) => {
        // 実装
      }),
    
    delete: protectedProcedure
      .input(z.object({ id: z.string() }))
      .mutation(async ({ ctx, input }) => {
        // 実装
      }),
  },
  
  auth: {
    me: publicProcedure.query(async ({ ctx }) => {
      // 実装
    }),
    
    logout: protectedProcedure.mutation(async ({ ctx }) => {
      // 実装
    }),
  },
});
```

### 5.2 リクエスト/レスポンス例

**セッション作成:**
```json
// リクエスト
{
  "startAt": "2026-03-06T11:25:00",
  "endAt": "2026-03-06T13:09:00",
  "durationMin": 104,
  "category": "study",
  "note": "AI学習"
}

// レスポンス
{
  "id": "uuid-xxx",
  "userId": "user-id",
  "startAt": "2026-03-06T11:25:00.000Z",
  "endAt": "2026-03-06T13:09:00.000Z",
  "durationMin": 104,
  "category": "study",
  "note": "AI学習",
  "createdAt": "2026-03-06T11:25:00.000Z",
  "updatedAt": "2026-03-06T11:25:00.000Z"
}
```

---

## 6. データベーススキーマ

### 6.1 users テーブル

```sql
CREATE TABLE users (
  id VARCHAR(255) PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  name VARCHAR(255),
  role ENUM('user', 'admin') DEFAULT 'user',
  createdAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**インデックス:**
- PRIMARY KEY: id
- UNIQUE: email

### 6.2 sessions テーブル

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
  INDEX idx_user_start (userId, startAt)
);
```

**インデックス:**
- PRIMARY KEY: id
- FOREIGN KEY: userId → users.id
- INDEX: (userId, startAt) — 日付範囲クエリの最適化

---

## 7. 認証フロー

### 7.1 ログイン

```
ユーザー
    ↓
[ログイン] ボタンクリック
    ↓
getLoginUrl() → Manus OAuth ポータルにリダイレクト
    ↓
ユーザー認証
    ↓
/api/oauth/callback にリダイレクト
    ↓
Express で OAuth コード検証
    ↓
JWT トークン生成
    ↓
セッションクッキーに保存
    ↓
ホームページにリダイレクト
    ↓
useAuth() で認証状態を取得
    ↓
TimeLogger ページ表示
```

### 7.2 API リクエスト

```
フロントエンド
    ↓
tRPC 呼び出し
    ↓
Express サーバー
    ↓
context.ts で JWT 検証
    ↓
ユーザーID 取得
    ↓
protectedProcedure で権限確認
    ↓
ビジネスロジック実行
    ↓
レスポンス返却
```

---

## 8. エラーハンドリング

### 8.1 クライアント側

```typescript
const createMutation = trpc.sessions.create.useMutation({
  onSuccess: () => {
    toast.success("記録を保存しました");
  },
  onError: (err) => {
    toast.error(`保存に失敗しました: ${err.message}`);
  },
});
```

### 8.2 サーバー側

```typescript
export const createSession = protectedProcedure
  .input(sessionSchema)
  .mutation(async ({ ctx, input }) => {
    try {
      if (new Date(input.endAt) <= new Date(input.startAt)) {
        throw new TRPCError({
          code: "BAD_REQUEST",
          message: "終了時刻は開始時刻より後にしてください",
        });
      }
      
      const result = await createSessionInDb(ctx.user.id, input);
      return result;
    } catch (error) {
      if (error instanceof TRPCError) throw error;
      throw new TRPCError({
        code: "INTERNAL_SERVER_ERROR",
        message: "セッション作成に失敗しました",
      });
    }
  });
```

---

## 9. パフォーマンス最適化

### 9.1 クエリ最適化

**インデックス活用:**
```sql
-- userId と startAt でインデックス
SELECT * FROM sessions
WHERE userId = ? AND startAt >= ? AND startAt < ?
ORDER BY startAt DESC;
```

**ページネーション（将来）:**
```typescript
export const listSessions = protectedProcedure
  .input(z.object({
    from: z.date(),
    to: z.date(),
    limit: z.number().default(50),
    offset: z.number().default(0),
  }))
  .query(async ({ ctx, input }) => {
    // LIMIT と OFFSET を使用
  });
```

### 9.2 キャッシング

**tRPC キャッシュ:**
```typescript
const { data, isLoading } = trpc.sessions.list.useQuery(
  { from, to },
  { staleTime: 5 * 60 * 1000 } // 5分間キャッシュ
);
```

**無効化:**
```typescript
const utils = trpc.useUtils();
utils.sessions.list.invalidate(); // キャッシュをクリア
```

### 9.3 レンダリング最適化

**useMemo で参照を安定化:**
```typescript
const windowDates = useMemo(
  () => computeWindowDates(mode, selectedDate),
  [mode, selectedDate.getTime()]
);
```

---

## 10. セキュリティ考慮事項

### 10.1 認証

- **JWT トークン** — 署名付き、有効期限付き
- **セッションクッキー** — HttpOnly, Secure フラグ
- **CSRF 保護** — tRPC で自動処理

### 10.2 入力検証

```typescript
const sessionSchema = z.object({
  startAt: z.date(),
  endAt: z.date(),
  durationMin: z.number().positive().int(),
  category: z.enum([...CATEGORIES]),
  note: z.string().max(500).optional(),
});
```

### 10.3 データアクセス制御

```typescript
// ユーザーは自分のセッションのみ取得可能
export const listSessions = protectedProcedure
  .input(z.object({ from: z.date(), to: z.date() }))
  .query(async ({ ctx, input }) => {
    return db
      .select()
      .from(sessions)
      .where(
        and(
          eq(sessions.userId, ctx.user.id), // ← ユーザーID チェック
          gte(sessions.startAt, input.from),
          lt(sessions.startAt, input.to)
        )
      );
  });
```

---

## 11. スケーラビリティ考慮事項

### 11.1 データベース

- **インデックス** — (userId, startAt) で高速化
- **パーティショニング** — 将来、月別パーティションを検討
- **アーカイブ** — 古いデータを別テーブルに移動

### 11.2 キャッシング

- **Redis** — セッションキャッシュ（将来）
- **CDN** — 静的アセット配信

### 11.3 API レート制限

```typescript
// 将来実装
const rateLimit = createRateLimiter({
  windowMs: 15 * 60 * 1000, // 15分
  max: 100, // 100リクエスト
});
```

---

## 12. 監視とロギング

### 12.1 ログレベル

```typescript
console.log("[INFO] ユーザー認証成功:", userId);
console.warn("[WARN] クエリが遅い:", duration, "ms");
console.error("[ERROR] データベース接続失敗:", error);
```

### 12.2 メトリクス

- **API レスポンスタイム** — 目標: < 200ms
- **データベースクエリ時間** — 目標: < 50ms
- **エラー率** — 目標: < 0.1%

---

**作成日:** 2026年3月9日  
**バージョン:** v10  
**次の開発者へ:** このドキュメントを参照して、新機能追加やバグ修正を行ってください。
