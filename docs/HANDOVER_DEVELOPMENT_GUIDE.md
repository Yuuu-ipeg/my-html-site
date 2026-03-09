# Time Logger Pro - 開発ガイド

**対象者:** 次の開発者  
**更新日:** 2026年3月9日  
**バージョン:** v10 以降

---

## 1. 開発環境のセットアップ

### 前提条件

- Node.js 22.13.0 以上
- pnpm 最新版
- MySQL / TiDB データベース
- Manus プラットフォームアカウント

### 初期セットアップ

```bash
# プロジェクトをクローン
git clone <repository-url>
cd time-logger-pro

# 依存関係をインストール
pnpm install

# 環境変数を確認（自動注入）
# DATABASE_URL, JWT_SECRET, VITE_APP_ID など

# データベースをセットアップ
pnpm db:push

# 開発サーバーを起動
pnpm dev
```

### ポート確認

- **フロントエンド:** http://localhost:5173 (Vite)
- **バックエンド:** http://localhost:3000 (Express)

---

## 2. 開発ワークフロー

### 新機能追加の標準手順

#### Step 1: 要件確認

新機能の要件を明確にします：
- **機能説明** — 何ができるようになるか
- **対象ユーザー** — 誰が使うか
- **成功基準** — どうなれば完成か

#### Step 2: データモデル設計

`drizzle/schema.ts` でテーブルを設計します。

```typescript
// 例: 目標テーブルを追加
export const goals = sqliteTable('goals', {
  id: text('id').primaryKey(),
  userId: text('user_id').notNull(),
  category: text('category').notNull(),
  targetMinutes: integer('target_minutes').notNull(),
  createdAt: timestamp('created_at').defaultNow(),
});
```

#### Step 3: マイグレーション実行

```bash
pnpm db:push
```

#### Step 4: データベースヘルパー作成

`server/db.ts` にクエリ関数を追加します。

```typescript
// 例: 目標を取得
export async function getGoalByUserAndCategory(
  userId: string,
  category: string
) {
  return db
    .select()
    .from(goals)
    .where(
      and(
        eq(goals.userId, userId),
        eq(goals.category, category)
      )
    )
    .get();
}
```

#### Step 5: tRPC Procedure 作成

`server/routers.ts` に手続きを追加します。

```typescript
// 例: 目標を取得する Procedure
goals: {
  getByCategory: protectedProcedure
    .input(z.object({ category: z.string() }))
    .query(async ({ ctx, input }) => {
      return await getGoalByUserAndCategory(ctx.user.id, input.category);
    }),

  // 目標を更新する Procedure
  update: protectedProcedure
    .input(z.object({
      category: z.string(),
      targetMinutes: z.number().positive(),
    }))
    .mutation(async ({ ctx, input }) => {
      return db
        .update(goals)
        .set({ targetMinutes: input.targetMinutes })
        .where(
          and(
            eq(goals.userId, ctx.user.id),
            eq(goals.category, input.category)
          )
        )
        .run();
    }),
},
```

#### Step 6: フロントエンド実装

`client/src/pages/` または `client/src/components/` に UI を追加します。

```typescript
import { trpc } from "@/lib/trpc";

export function GoalSetting({ category }: { category: string }) {
  const { data: goal, isLoading } = trpc.goals.getByCategory.useQuery(
    { category },
    { enabled: !!category }
  );

  const updateMutation = trpc.goals.update.useMutation({
    onSuccess: () => {
      toast.success("目標を更新しました");
    },
  });

  return (
    <div>
      {isLoading ? (
        <div>読み込み中...</div>
      ) : (
        <div>
          <p>目標: {goal?.targetMinutes} 分</p>
          <Button
            onClick={() =>
              updateMutation.mutate({
                category,
                targetMinutes: 120,
              })
            }
          >
            120分に設定
          </Button>
        </div>
      )}
    </div>
  );
}
```

#### Step 7: テスト作成

`server/goals.test.ts` または `client/src/lib/goals.test.ts` にテストを記述します。

```typescript
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import { createTestUser, createTestGoal, db } from "./test-utils";

describe("goals router", () => {
  let userId: string;

  beforeAll(async () => {
    userId = await createTestUser();
  });

  afterAll(async () => {
    // クリーンアップ
  });

  it("should get goal by category", async () => {
    const goal = await createTestGoal(userId, "study", 120);
    const result = await getGoalByUserAndCategory(userId, "study");
    expect(result?.targetMinutes).toBe(120);
  });

  it("should update goal", async () => {
    // テスト実装
  });
});
```

#### Step 8: テスト実行

```bash
pnpm test
```

全テストが成功することを確認します。

#### Step 9: チェックポイント保存

```bash
# Manus UI の "Publish" ボタンをクリック前に
# または webdev_save_checkpoint コマンドで
```

---

## 3. 時間表示ロジックの使用

### formatDuration 関数

時間表示は常に `formatDuration()` を使用してください。

```typescript
import { formatDuration } from "@/lib/formatDuration";

// スマート表示（推奨）
formatDuration(45, "smart");   // "45分"
formatDuration(90, "smart");   // "1.5h"

// 詳細表示（モーダルなど）
formatDuration(90, "detailed"); // "1時間30分"

// コンパクト表示（リスト）
formatDuration(90, "compact"); // "1.5h"

// 完全表示（ツールチップなど）
formatDuration(90, "full"); // "1.5h (90分)"
```

### formatTotalDuration 関数

合計時間の表示に使用します。

```typescript
const { main, sub } = formatTotalDuration(150);
// main: "2.5h"
// sub: "(150分)"

// 表示例
<div>
  <span className="text-lg font-bold">{main}</span>
  <span className="text-xs text-gray-500">{sub}</span>
</div>
```

### formatCategoryDuration 関数

カテゴリ別表示に使用します。

```typescript
const { primary, secondary } = formatCategoryDuration(150);
// primary: "2.5h"
// secondary: "150分"

// 表示例
<div>
  <p>{category}</p>
  <p>{primary}</p>
  {secondary && <p className="text-xs">{secondary}</p>}
</div>
```

---

## 4. レスポンシブデザインの実装

### ブレークポイント

Tailwind CSS の `sm:` ブレークポイント（640px）を使用します。

```tsx
// モバイル優先で実装
<div className="text-xs sm:text-sm">テキスト</div>
<div className="p-2 sm:p-4">パディング</div>
<div className="gap-2 sm:gap-4">ギャップ</div>
```

### グリッドレイアウト

```tsx
// 1列 → lg時に3列
<div className="grid grid-cols-1 lg:grid-cols-3 gap-4 sm:gap-6">
  <div className="lg:col-span-1">左パネル</div>
  <div className="lg:col-span-2">右パネル</div>
</div>
```

### フォントサイズの統一

```tsx
// 見出し
<h1 className="text-base sm:text-2xl font-bold">...</h1>

// 本文
<p className="text-xs sm:text-sm">...</p>

// ボタン
<Button className="text-xs sm:text-sm h-9 sm:h-10">...</Button>
```

---

## 5. スマホ固有機能の実装

### デバイス判定

```typescript
const isTouchDevice = "ontouchstart" in window || navigator.maxTouchPoints > 0;

if (isTouchDevice) {
  // スマホ用の処理
} else {
  // PC用の処理
}
```

### ポップオーバー実装

```typescript
import { useState, useRef, useEffect } from "react";

export function MyComponent() {
  const [popover, setPopover] = useState<{ x: number; y: number } | null>(null);
  const popoverRef = useRef<HTMLDivElement>(null);

  // 外側クリック検出
  useEffect(() => {
    const handleClickOutside = (e: MouseEvent) => {
      if (popoverRef.current && !popoverRef.current.contains(e.target as Node)) {
        setPopover(null);
      }
    };
    if (popover) {
      document.addEventListener("mousedown", handleClickOutside);
    }
    return () => {
      document.removeEventListener("mousedown", handleClickOutside);
    };
  }, [popover]);

  return (
    <>
      <button onClick={(e) => setPopover({ x: e.clientX, y: e.clientY })}>
        開く
      </button>
      {popover && (
        <div
          ref={popoverRef}
          className="absolute z-50 bg-white rounded-lg shadow-lg p-4"
          style={{ left: popover.x, top: popover.y }}
        >
          ポップオーバー内容
        </div>
      )}
    </>
  );
}
```

---

## 6. テスト戦略

### テストの種類

| テストタイプ | 場所 | 例 |
|-------------|------|-----|
| **ユーティリティ関数** | `lib/*.test.ts` | formatDuration, 計算関数 |
| **サーバー API** | `server/*.test.ts` | tRPC procedures, DB クエリ |
| **コンポーネント** | `components/*.test.ts` | UI ロジック（オプション） |

### テスト実行

```bash
# 全テストを実行
pnpm test

# 特定のテストファイルのみ
pnpm test formatDuration.test.ts

# ウォッチモード（開発中）
pnpm test --watch
```

### テストテンプレート

```typescript
import { describe, it, expect } from "vitest";

describe("機能名", () => {
  it("正常系: 期待される動作", () => {
    const result = myFunction(input);
    expect(result).toBe(expectedOutput);
  });

  it("異常系: エラーハンドリング", () => {
    expect(() => myFunction(invalidInput)).toThrow();
  });

  it("エッジケース: 境界値", () => {
    expect(myFunction(0)).toBe(expected);
    expect(myFunction(MAX_VALUE)).toBe(expected);
  });
});
```

---

## 7. デバッグ技法

### ブラウザコンソール

```typescript
// tRPC クエリのデバッグ
const { data, isLoading, error } = trpc.sessions.list.useQuery(...);
console.log("data:", data);
console.log("isLoading:", isLoading);
console.log("error:", error);
```

### サーバーログ

```typescript
// Express ミドルウェアでログ出力
app.use((req, res, next) => {
  console.log(`[${new Date().toISOString()}] ${req.method} ${req.url}`);
  next();
});

// tRPC procedure 内でログ出力
export const myProcedure = protectedProcedure
  .input(z.object({ id: z.string() }))
  .query(async ({ ctx, input }) => {
    console.log("User:", ctx.user.id, "Input:", input);
    const result = await db.select().from(myTable).where(...);
    console.log("Result:", result);
    return result;
  });
```

### React DevTools

- **Components タブ** — コンポーネント階層の確認
- **Profiler タブ** — レンダリングパフォーマンスの測定

---

## 8. パフォーマンス最適化

### クエリの最適化

```typescript
// ❌ N+1 クエリ問題
for (const session of sessions) {
  const user = await db.select().from(users).where(eq(users.id, session.userId));
}

// ✅ JOIN で解決
const sessions = await db
  .select()
  .from(sessionTable)
  .innerJoin(users, eq(sessionTable.userId, users.id));
```

### React レンダリング最適化

```typescript
// ❌ 毎回新しい配列参照
const { data } = trpc.items.list.useQuery({ ids: [1, 2, 3] });

// ✅ useMemo で安定化
const ids = useMemo(() => [1, 2, 3], []);
const { data } = trpc.items.list.useQuery({ ids });
```

### バンドルサイズ削減

```bash
# バンドルサイズを確認
pnpm build

# 不要な依存関係を削除
pnpm remove <package-name>
```

---

## 9. よくある問題と解決策

### 問題: タイムラインのポップオーバーが画面外に出る

**原因:** ポップオーバーの位置計算が不正確

**解決策:**
```typescript
// 位置を画面内に調整
const adjustedX = Math.min(
  Math.max(8, popover.x - 130),
  containerWidth - 268
);
```

### 問題: スマホでレイアウトが崩れる

**原因:** Tailwind CSS のレスポンシブ指定が不足

**解決策:**
```tsx
// モバイル優先で実装
<div className="text-xs sm:text-sm p-2 sm:p-4">
  {/* コンテンツ */}
</div>
```

### 問題: 時間表示が統一されていない

**原因:** 異なる場所で異なる形式で表示

**解決策:**
```typescript
// 常に formatDuration を使用
import { formatDuration } from "@/lib/formatDuration";
formatDuration(minutes, "smart");
```

### 問題: テストが失敗する

**原因:** テストデータの初期化が不正確

**解決策:**
```typescript
beforeAll(async () => {
  // テストデータを準備
  await setupTestDatabase();
});

afterAll(async () => {
  // テストデータをクリーンアップ
  await cleanupTestDatabase();
});
```

---

## 10. デプロイ手順

### 本番環境へのデプロイ

```bash
# 1. 全テストが成功することを確認
pnpm test

# 2. TypeScript 型チェック
npx tsc --noEmit

# 3. ビルド確認
pnpm build

# 4. チェックポイントを保存
# Manus UI で "Publish" ボタンをクリック
```

### ロールバック手順

```bash
# 前のバージョンに戻す
manus-webdev://8b7b0a69  # v9
```

---

## 11. コードスタイルガイド

### ファイル命名規則

```
components/MyComponent.tsx      # React コンポーネント（PascalCase）
lib/myUtility.ts               # ユーティリティ関数（camelCase）
pages/MyPage.tsx               # ページコンポーネント（PascalCase）
server/routers.ts              # tRPC ルーター（camelCase）
*.test.ts                       # テストファイル（同名 + .test）
```

### コメント規則

```typescript
/**
 * 関数の説明
 * @param param1 - パラメータの説明
 * @returns 戻り値の説明
 */
export function myFunction(param1: string): string {
  // 複雑なロジックの説明
  return result;
}
```

### 型定義

```typescript
// ❌ any を使わない
function process(data: any) { }

// ✅ 明示的に型を定義
interface ProcessData {
  id: string;
  value: number;
}
function process(data: ProcessData) { }
```

---

## 12. リソース

### 公式ドキュメント
- [React 19](https://react.dev)
- [Tailwind CSS 4](https://tailwindcss.com)
- [tRPC](https://trpc.io)
- [Drizzle ORM](https://orm.drizzle.team)

### Manus プラットフォーム
- [Manus ドキュメント](https://docs.manus.im)
- [OAuth 実装ガイド](https://docs.manus.im/oauth)

### 開発ツール
- [Chrome DevTools](https://developer.chrome.com/docs/devtools/)
- [VS Code](https://code.visualstudio.com)
- [Postman](https://www.postman.com)（API テスト）

---

## 13. サポート連絡先

問題が発生した場合：

1. **ドキュメント確認** — このガイドと `HANDOVER_PROJECT_OVERVIEW.md` を確認
2. **テスト実行** — `pnpm test` でエラーを確認
3. **ログ確認** — ブラウザコンソールとサーバーログを確認
4. **Git ログ確認** — 最近の変更を確認

---

**最終更新:** 2026年3月9日  
**バージョン:** v10  
**次の開発者へ:** 質問や問題があれば、このドキュメントと関連ファイルを参照してください。
