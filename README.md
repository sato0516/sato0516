## 🌸 ご挨拶
ご覧いただきありがとうございます。<br>
Webエンジニアを目指して一から学習中です。<br>
ベテランエンジニアの夫に見守ってもらいながら、<br>
ChatGPT（GPT-5有料プラン）を活用して独学に励んでいます。

---

## 📂 これまでの学習内容
- 書籍「1冊ですべて身につくHTML&CSSとWebデザイン入門講座」をもとにWebサイト制作
- Progate（有料プラン）でJavaScript / SQL / React / Command Line のレッスンを修了
- paizaラーニング問題演習～GitHubでプルリクエスト運用 / 夫からレビューを受け改善 / マージする開発サイクルを実践 
- ToDoリスト作成（Reactで実装）
- お問い合わせフォーム作成（Next.jsで環境構築 / バリデーション設計 / ServerActionで送信処理 / PrismaでDB操作）
- 簡易ブログシステム作成（Next.jsで環境構築 / ログイン機能 / 検索機能 / 画像UL）
- デプロイ関連（Vercel / GitHub Actions × AWS App Runner）

---

## ⚙️ 使用技術
- **フロントエンド**：HTML / CSS / JavaScript / TypeScript  
- **フレームワーク**：React / Next.js / Node.js  
- **データベース・ORM**：Prisma ORM / SQLite / PostgreSQL / Neon  
- **スタイリング・UI**：Tailwind CSS / shadcn/ui
- **バリデーション**：Zod  
- **開発環境・ツール**：GitHub / VSCode / Node.js / Docker  
- **デプロイ・インフラ**：Vercel / GitHub Actions / AWS（IAM, ECR, App Runner）  

---

## 💻 デプロイ実践記録（手順メモ）
### ① Vercel × Neon  
✅ [公開ページはこちら（Vercel）](https://nextjs-2-hfx2.vercel.app)

#### 0. 前提（アカウント & 環境）

- GitHub アカウント（コードは `nextjs-2` リポに push 済み）
- Vercel アカウント（GitHub 連携でOK）
- Neon アカウント（無料で作成可）
- Node.js 18 以上を推奨（`node -v` で確認）
- 端末に Git が入っていること（`git --version`）

> **ゴールのイメージ**  
> 1) ローカルで Postgres に切り替え → マイグレーション適用  
> 2) GitHub に push  
> 3) Vercel で GitHub と連携 → 環境変数 `DATABASE_URL` をセット → デプロイ  
> 4) 公開URLで “書き込み” を含む動作ができることを確認

---

#### 1. リポジトリの取得・準備

```bash
# まだならクローン
git clone https://github.com/sato0516/nextjs-2.git
cd nextjs-2

# 依存パッケージ導入（npm）
npm ci || npm install
```

> ※ リポジトリに `prisma` ディレクトリが無い場合は、後述の「Prisma の導入（無い場合）」に従って作成。

---

#### 2. Prisma の導入・確認

この手順では ORM として **Prisma** を使う。既に導入済みでも、バージョンや依存の整合のために実行してOK。

```bash
# Prisma CLI（開発依存）と Prisma Client（本番でも使用）
npm i -D prisma
npm i @prisma/client
```

`package.json` の `scripts` に以下を追加または統合。

```jsonc
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "postinstall": "prisma generate",
    "migrate:deploy": "prisma migrate deploy",
    "vercel-build": "prisma generate && prisma migrate deploy && next build"
  }
}
```

- **postinstall**：Vercel のビルド時に Prisma Client を自動生成するために必須級
- **migrate:deploy**：本番データベースにマイグレーションを適用
- **vercel-build**：Vercel が優先して使うビルドコマンド（定義しておくと安心）

---

#### 3. Prisma スキーマを SQLite → Postgres に変更

`prisma/schema.prisma` を開き、`datasource` を **postgresql** に変更。

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// 以降、既存の model 定義はそのままでOK（必要に応じて調整）
```

> 以前の `provider = "sqlite"` や `url = "file:./dev.db"` などの記述があれば削除またはコメントアウトしてください。

---

#### 4. Neon（無料 Postgres）を用意して接続文字列を取得

##### A. Neon 側から Vercel と連携する場合（おすすめ）
1. Neon Console にログイン → **Integrations** → **Vercel**  
2. 「Install from Vercel Marketplace」を選んで指示に従い接続  
3. データベースを作成 → 対象の **Vercel プロジェクト** とリンク  
4. `DATABASE_URL` が **Vercel の環境変数**に自動登録される（後で確認）

##### B. Vercel 側から Neon を追加する場合
1. Vercel Dashboard → **Integrations** → **Neon Postgres** をインストール  
2. プロジェクトを選び、Region・プラン（無料）などを選択 → DB 作成  
3. `DATABASE_URL` が **Vercel の環境変数**に自動登録される（後で確認）

**どちらでもOK。** ポイントは「**Vercel の環境変数に `DATABASE_URL` が入る**」こと。

---

#### 5. ローカル開発用 `.env` を作成

リポジトリ直下に `.env` を作成して、Neon の接続文字列を入れる：

```env
# .env（ローカル専用。絶対にコミットしない）
DATABASE_URL="postgresql://<user>:<password>@<host>:<port>/<database>?sslmode=require"
```

> `.env` は **秘密情報** のため Git にコミットしない（`.gitignore` されていることを確認）。

---

#### 6. マイグレーションの適用（ローカル）

既存の Prisma モデルに基づいて、Postgres にテーブルを作成。

```bash
# 初回はマイグレーション名（例: init）を聞かれる
npx prisma migrate dev

# Prisma Client を生成（postinstall でも生成されるがローカル確認用）
npx prisma generate
```

アプリのローカル動作確認：

```bash
npm run dev
# http://localhost:3000 にアクセスし、 CRUD（書き込み）が動くか軽く確認
```

> もし Seed データを入れたい場合は、`prisma/seed.ts` を作成し、`npm run seed` などのスクリプトを用意。

---

#### 7. GitHub に push（変更を反映）

```bash
git add .
git commit -m "feat: migrate DB from SQLite to Postgres (Neon)"
git push origin main
```

> 以降は Vercel が GitHub の変更をフックして自動デプロイしてくれる。

---

#### 8. Vercel プロジェクトの作成 & 連携

1. Vercel にログイン → **Add New… → Project**  
2. GitHub 上の `sato0516/nextjs-2` を選択  
3. Framework は自動で **Next.js** と判定される（設定は基本そのままでOK）  
4. **Project Settings → Environment Variables** で `DATABASE_URL` が入っているか確認  
   - 連携で自動登録されていなければ、**手動で追加**（Development / Preview / Production すべてに同じ値を入れるのが楽）
5. Deploy を実行（初回は数分かかる）

> **ビルドコマンドについて**  
> `vercel-build` を定義していれば、Vercel は以下の順で実行：  
> ① `prisma generate` → ② `prisma migrate deploy` → ③ `next build`。  
> これで **本番DBに自動でマイグレーションが適用** され、Prisma Client も生成済みとなる。

---

#### 9. 動作確認（本番 URL）

- デプロイ完了後、Vercel が `https://<project-name>.vercel.app` の URL を表示。  
- 画面の “作成・更新” など **書き込み操作** が本番でも成功するかを確認。  
- もし 404/500 が出る場合は、**Logs（Vercel のプロジェクト画面）** を参照。

---

#### 10. よくあるつまずきと対処

##### 10-1. 「`@prisma/client` did not initialize yet」
- **原因**：本番ビルド時に Prisma Client が生成されていない
- **対処**：`package.json` に `"postinstall": "prisma generate"` を入れて再デプロイ

##### 10-2. 「`DATABASE_URL` が無い／読み取れない」
- **原因**：Vercel の環境変数に `DATABASE_URL` が未設定、または Preview/Production のどちらかが未設定
- **対処**：**Project Settings → Environment Variables** で `DATABASE_URL` を **Development / Preview / Production** すべてに追加して再デプロイ

##### 10-3. マイグレーションが反映されない
- **原因**：本番で `prisma migrate deploy` が走っていない
- **対処**：`"migrate:deploy": "prisma migrate deploy"` を scripts に追加し、`vercel-build` または Build Command に含める

##### 10-4. 型エラー / Client 不一致
- **原因**：スキーマ変更後に生成された Prisma Client が古い
- **対処**：`npx prisma generate` を再実行 → コミット → デプロイをやり直す

##### 10-5. 接続数やパフォーマンス（規模が大きくなったら）
- Neon の無料プランは接続数などに上限があります。ポートフォリオ規模なら多くの場合 OK。必要に応じてプランや接続プールの導入を検討。

---

#### 11. 片付け（SQLite を使わなくなる場合）

- リポ内の `prisma/dev.db` 等の SQLite ファイルを削除
- README の記述を “Postgres（Neon）” に更新
- `.env` は **コミットしない**（Vercel 側の環境変数に本番用 `DATABASE_URL` を設定しておく）

---

#### 12. チェックリスト

- `schema.prisma` の `provider = "postgresql"` になっている  
- ローカル `.env` の `DATABASE_URL` は Neon の接続文字列  
- `package.json` に `postinstall` と `migrate:deploy`、（あれば）`vercel-build` がある  
- Vercel の **Project Settings → Environment Variables** に `DATABASE_URL` がある（Dev/Preview/Prod）  
- 本番 URL で **書き込み操作** が成功する

---

### ② GitHub Actions × AWS App Runner 
✅ [公開ページはこちら（App Runner）](https://あなたの-apprunner-URL.awsapprunner.com)

---
