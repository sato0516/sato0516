## 🌸 ご挨拶
ご覧いただきありがとうございます。<br>
Webエンジニアを目指して一から学習中です。<br>
ベテランエンジニアの夫に見守ってもらいながら、ChatGPT（GPT-5有料プラン）を活用して独学に励んでいます。

---

## 📂 これまでの学習内容
- 書籍「1冊ですべて身につくHTML&CSSとWebデザイン入門講座」をもとにWebサイト制作
- Progate（有料プラン）でJavaScript / SQL / React / Command Line のレッスンを修了
- paizaラーニング問題演習～GitHubでプルリクエスト運用 / 夫からレビューを受け改善 / マージする開発サイクルを実践 
- ToDoリスト作成（Reactで実装）
- お問い合わせフォーム作成（Udemy講座 / Next.jsで環境構築 / バリデーション設計 / ServerActionで送信処理 / PrismaでDB操作）
- 簡易ブログシステム作成（Udemy講座 / Next.jsで環境構築 / ログイン機能 / 検索機能 / 画像UL）
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
## **① Vercel × Neon**
✅ **[公開ページはこちら（Vercel）](https://nextjs-2-hfx2.vercel.app)**

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

## **② GitHub Actions × AWS App Runner**
✅ **[公開ページはこちら（App Runner）](https://あなたの-apprunner-URL.awsapprunner.com)**

#### 0. 諸々初期設定

eslint.config.mjsに「"src/generated/**"」を追加

```yaml
import { dirname } from "path";
import { fileURLToPath } from "url";
import { FlatCompat } from "@eslint/eslintrc";

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

const compat = new FlatCompat({
  baseDirectory: __dirname,
});

const eslintConfig = [
  ...compat.extends("next/core-web-vitals", "next/typescript"),
  {
    ignores: [
      "node_modules/**",
      ".next/**",
      "out/**",
      "build/**",
      "next-env.d.ts",
      "src/generated/**", // ← 🆕 Prismaなどの自動生成コードを除外
    ],
  },
];

export default eslintConfig;
```

##### next.config.ts

```yaml
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  // ここは experimental ではなくトップレベル（Next 15）
  typedRoutes: true,

  reactStrictMode: true,

  // CI 中の ESLint 失敗で止めたくない場合は有効化（任意）
  eslint: { ignoreDuringBuilds: true },

  // 画像やファイルトレースを有効化（デフォでOKだが明示）
  outputFileTracing: true,

  // App Runner のポートは環境変数 PORT=3000 で受けるので特に不要
  // 必要なら他の設定を追記
};

export default nextConfig;

```

ファイル名: .dockerignore 追加

```yaml
node_modules
.next
.git
.gitignore
Dockerfile
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.vscode
dist
coverage
```

→GitHubActionsでビルドを軽く・速くする

##### GitHubActionsに環境変数を設定

- GitHub リポジトリ → **Settings → Secrets and variables → Actions → New repository secret**
- **Name:** `DATABASE_URL`
    
    

##### schema.prisma

```yaml
generator client {
  provider = "prisma-client-js"
  engineType    = "library" // Next.js向け
  binaryTargets = ["debian-openssl-3.0.x","native"] // bookwormはOpenSSL 3系
  output   = "../src/generated/prisma"
}
```

#### 1. プロジェクト直下にDockerfileを作成

```docker
# ===== base =====
FROM node:22-bookworm-slim AS base
ENV PNPM_HOME="/root/.local/share/pnpm"
ENV PATH="$PNPM_HOME:$PATH"
# App Runner 既定ポート（今回は 3000）
ENV PORT=3000

# ===== deps: 依存を入れる。postinstall は無効化し、devDeps も入れる =====
FROM base AS deps
WORKDIR /app

# 先に package をコピー（キャッシュ効率）
COPY package.json package-lock.json ./

# postinstall（= prisma generate）をここでは走らせない
# → prisma/ をまだコピーしていないため。後段で明示実行します。
RUN npm ci --ignore-scripts

# ===== builder: ソース投入→ビルド（devDeps 必須）=====
FROM base AS builder
WORKDIR /app

# node_modules（devDeps 付き）を引き継ぐ
COPY --from=deps /app/node_modules ./node_modules

# Prisma スキーマやソース一式をコピー
COPY . .

# ビルド前に generate を含む build を実行
# package.json の "build": "prisma generate && next build" を利用
# DATABASE_URL は generate には不要ですが、プラグインによっては欲しがることがあるので ARG で受け取り可能に
ARG DATABASE_URL
ENV DATABASE_URL=$DATABASE_URL

# Next/Tailwind のビルド（devDependencies が必要）
RUN npm run build

# 本番用に devDeps を除去（ランナーに渡す node_modules を軽量化）
RUN npm prune --omit=dev

# ===== runner: 実行用（できるだけ軽く）=====
FROM node:22-bookworm-slim AS runner
WORKDIR /app

# 本番向け
ENV NODE_ENV=production
ENV PORT=3000

# ランタイムに必要な最低限をコピー
COPY --from=builder /app/package.json /app/package-lock.json ./
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
# Prisma Client は node_modules 内に生成済み。念のためスキーマも持っておく（不要なら省略可）
COPY --from=builder /app/prisma ./prisma

# ここで migrate を実行してから立ち上げ（初回・スキーマ変更時の安全策）
# App Runner 側で DATABASE_URL を環境変数として設定済みであることが前提
CMD sh -c "npx prisma migrate deploy && node node_modules/next/dist/bin/next start -p ${PORT}"

```

#### 2. GitHub Actionsの設定ファイルを作成

##### 2-1 パス: .github/workflows/deploy-apprunner.yml

```yaml
name: Deploy to AppRunner (ECR image)

on:
  push:
    branches: [ "main" ]
  workflow_dispatch:

permissions:
  id-token: write   # OIDC AssumeRole に必須
  contents: read

env:
  AWS_REGION: ap-northeast-1
  ECR_REPOSITORY: nextjs-test
  IMAGE_TAG: ${{ github.sha }}
  # 明示的に引用符で囲む（YAML パース事故回避）
  ROLE_TO_ASSUME: "arn:aws:iam::775209358765:role/github-oidc-deploy-role"
  APP_RUNNER_SERVICE_ARN: "arn:aws:apprunner:ap-northeast-1:775209358765:service/nextjs-test/0c09533fdc5d41ddb9d5f5d446fc0e85"

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ env.ROLE_TO_ASSUME }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build image
        run: |
          ECR_URI="${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}"
          docker build \
            --build-arg DATABASE_URL="${{ secrets.DATABASE_URL }}" \
            -t "$ECR_URI:latest" \
            -t "$ECR_URI:${{ env.IMAGE_TAG }}" \
            .

      - name: Push image
        run: |
          ECR_URI="${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}"
          docker push "$ECR_URI:latest"
          docker push "$ECR_URI:${{ env.IMAGE_TAG }}"

      # App Runner（手動デプロイ）を明示トリガー
      - name: Start App Runner deployment
        run: |
          aws apprunner start-deployment --service-arn "${{ env.APP_RUNNER_SERVICE_ARN }}"

```

##### 2-2 パス: .github/workflows/push-initial-image.yml

```yaml
name: Push initial image to ECR

on:
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

env:
  AWS_REGION: ap-northeast-1
  ECR_REPOSITORY: nextjs-test
  IMAGE_TAG: ${{ github.sha }}
  ROLE_TO_ASSUME: "arn:aws:iam::775209358765:role/github-oidc-deploy-role"

jobs:
  push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ env.ROLE_TO_ASSUME }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build image
        run: |
          ECR_URI="${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}"
          docker build \
            --build-arg DATABASE_URL="${{ secrets.DATABASE_URL }}" \
            -t "$ECR_URI:latest" \
            -t "$ECR_URI:${{ env.IMAGE_TAG }}" \
            .

      - name: Push image
        run: |
          ECR_URI="${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}"
          docker push "$ECR_URI:latest"
          docker push "$ECR_URI:${{ env.IMAGE_TAG }}"

```

#### 3. AWS

##### 3-1. IAMロール(OIDC)設定

概要: GitHub → AWSを“鍵レス認証”で接続
目的: セキュアなCI/CD
GitHub Actionsが「長期キーなし」でAWSにアクセスできるようにします。

###### 3-1-1. IAMコンソールを開く

→ https://console.aws.amazon.com/iam/

左メニューから「**ID プロバイダー**」を選択。

###### 3-1-2. 新しいOIDCプロバイダーを追加

- プロバイダーのタイプ：**OpenID Connect**
- プロバイダーURL：
    
    ```
    https://token.actions.githubusercontent.com
    ```
    
- 対象者（audience）：
    
    ```
    sts.amazonaws.com
    ```
    

保存すればOK。

###### 3-1-3. ポリシーの作成

左メニューポリシー→ポリシーの作成→「JSON」タブを選択し、下記を丸ごと張り付け

```yaml
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EcrAuth",
      "Effect": "Allow",
      "Action": ["ecr:GetAuthorizationToken"],
      "Resource": "*"
    },
    {
      "Sid": "EcrPushPull",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:CompleteLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:InitiateLayerUpload",
        "ecr:PutImage",
        "ecr:BatchGetImage",
        "ecr:DescribeRepositories",
        "ecr:CreateRepository"
      ],
      "Resource": "arn:aws:ecr:ap-northeast-1:775209358765:repository/nextjs-test"
    },
    {
      "Sid": "AppRunnerDeploy",
      "Effect": "Allow",
      "Action": [
        "apprunner:StartDeployment",
        "apprunner:DescribeService"
      ],
      "Resource": "arn:aws:apprunner:ap-northeast-1:775209358765:service/nextjs-test/0c09533fdc5d41ddb9d5f5d446fc0e85"
    }
  ]
}

```

「次へ」→ 「名前を設定」

- ポリシー名：`GitHubActionsAppRunnerPolicy`
- 説明：`Allows GitHub Actions to push to ECR and update App Runner services`

「ポリシーを作成」ボタンをクリック

###### 3-1-4. IAMロール作成

1. 左メニュー「**ロール**」→「**ロールの作成**」をクリック
2. 信頼されたエンティティタイプ → **Web ID**
3. プロバイダー選択 → `token.actions.githubusercontent.com`
4. Audience → `sts.amazonaws.com`
5. GitHub organization → 自分のGitHubアカウント名 (例: ryohei0109-develop)
6. 次へ
7. さっき作った `GitHubActionsAppRunnerPolicy` にチェックを入れる
8. 「次へ」→ ロール名を `github-oidc-deploy-role` にして作成！
9. GitHub Actions の設定ファイルである `deploy-apprunner.yml` と`publish-initial-image.yml` にARNを設定

```yaml
(例)
role-to-assume: arn:aws:iam::775209358765:role/github-oidc-deploy-role
```

##### 3-2. ECRリポジトリ作成

概要: Dockerイメージの保存先
目的: App Runnerがイメージを取得

GitHub Actions がビルドしたDockerイメージを push →

App Runner が pull して実行できるようにする。

##### 手順（AWSマネジメントコンソールで）

1️⃣ **ECRコンソールを開く**

👉 https://console.aws.amazon.com/ecr/repositories

2️⃣ **「リポジトリを作成」** をクリック

3️⃣ **設定項目**

| 項目 | 設定値 |
| --- | --- |
| リポジトリ名 | `nextjs-test` ← GitHub Actionsの`ECR_REPOSITORY`と同じにする |
| 可視性設定 | 🔒 プライベート（デフォルト） |
| タグの指定 | そのまま（オプション） |
| イメージスキャン | ✅ 有効でも無効でもOK（セキュリティチェック目的） |
| 暗号化設定 | デフォルトのまま（KMS管理キーでも可） |

4️⃣ 「リポジトリを作成」ボタンを押す

→ECRのURIが生成される

![image.png](attachment:fe07944d-32dc-4771-80d0-11f9ba1c68a9:image.png)

##### 補足

- このリポジトリは**1回作ればOK**。以後、GitHub Actions が毎回自動で：
    1. `docker build`
    2. `docker tag`
    3. `docker push`
        
        を行います。
        
- ECRはリージョンごとに独立してるので、App Runnerと同じリージョン
    
    （今回は `ap-northeast-1`＝東京）で作成してください。
    

##### 3-3. App Runnerサービス作成

概要: 本番アプリを動かす環境
目的: 自動デプロイ先

##### 準備確認

✅ ECRリポジトリ名：`nextjs-test`

✅ リージョン：`ap-northeast-1`（東京）

✅ GitHub Actions → ECR が成功する想定

##### 事前準備

あらかじめECRに初回のイメージ作成をしておく必要がある

GitHubにアクセス→Actionsタブ→**“Push initial image to ECR” → Run workflow** を実行

![image.png](attachment:09fecae5-27e0-472f-9af3-1e7f757f3838:image.png)

すると、ECRにイメージが作成される

![image.png](attachment:0dfaad0f-20cb-448d-a040-83d96678ed00:image.png)

ECR
└── nextjs-test
├── bootstrap → 最初に登録したイメージ
└── latest    → 現在の最新イメージ（以降CI/CDで更新）

今後は GitHub Actions で main ブランチに push するたびに「latest」が自動更新され、App Runnerもそれをpullして更新します。

##### 🔧 手順（AWSコンソールから）

###### ① App Runnerにアクセス

👉 https://console.aws.amazon.com/apprunner/home

右上のリージョンが「**アジアパシフィック（東京） ap-northeast-1**」になっているのを確認。

→ 「**サービスの作成**」をクリック。

###### ① App Runner へアクセス

👉 https://console.aws.amazon.com/apprunner/home

右上リージョンが `アジアパシフィック（東京） ap-northeast-1` になっていることを確認してね。

---

###### ② 「サービスの作成」をクリック

---

###### ③ ソースとデプロイ設定

| 項目 | 設定内容 | 備考 |
| --- | --- | --- |
| **デプロイ元** | ✅ コンテナレジストリ |  |
| **プロバイダー** | ✅ Amazon ECR |  |
| **コンテナイメージURI** | 今ECRにできた `nextjs-test:latest` を選択 |  |
| **デプロイトリガー** | 🔘 手動（GitHub Actionsで更新するから） |  |
| ECR アクセスロール | 新しいサービスロールの作成 | App Runnerが自動で必要最小限のIAMロールを作ってくれます👌
手動で作る必要はありません（むしろ自動作成が安全）。 |

> 💡「続行」ボタンが押せるようになったら latest タグを選択して進めてOK！
> 

![image.png](attachment:20449657-5bbb-47a5-864f-cb425889bf15:image.png)

###### ④ サービス設定

| 項目 | 設定値 | 備考 |
| --- | --- | --- |
| **サービス名** | `nextjs-test`（任意でもOK） |  |
| vCPU | 0.25 vCPU |  |
| **ポート** | `3000` | next.jsはデフォルトで3000番ポートでサーバを起動するため。 |
| **ヘルスチェック** | HTTP / パス `/` |  |
| **Auto scaling** | デフォルト（1〜5）でOK |  |
| **ログ** | ✅ CloudWatchログを有効化（おすすめ） |  |
| 環境変数 | DATABASE_URL | posgreへの接続URLを指定 |

---

###### ⑥ ネットワーク設定

| 項目 | 設定値 |
| --- | --- |
| **VPCコネクタ** | 今は不要（Neon接続はインターネット経由） |
| **Egress type** | `Default`（そのまま） |

---

###### ⑦ 「次へ」→ 「作成」ボタンをクリック

デプロイが始まるので、2〜5分ほど待ちます。

ステータスが `Running` になったら成功です🎉

エラーの場合は、サービスARNの値をコピーして、deploy-apprunner.ymlのAPP_RUNNER_SERVICE_ARNに張り付ける

![image.png](attachment:da006f14-0a05-485d-9e51-7f0080ec71ed:image.png)

##### デプロイ完了後に確認するもの

App Runner サービス詳細画面に表示される：

- **ドメインURL**
    
    例）`https://xxxxxxxx.ap-northeast-1.awsapprunner.com`
    
    → これが実際にアクセスできるWebアプリURL！
    
- **サービスARN**
    
    例）
    
    ```
    arn:aws:apprunner:ap-northeast-1:775209358765:service/nextjs-test/8a1b7a6cbaef4a11a
    ```
    
    これを控えて、GitHub ActionsのYAMLの環境変数に設定👇
    
    ```yaml
    APP_RUNNER_SERVICE_ARN: arn:aws:apprunner:ap-northeast-1:775209358765:service/nextjs-test/8a1b7a6cbaef4a11a
    ```
---
