# CLAUDE.md テンプレートガイド

dev-init Phase 6 で CLAUDE.md を生成する際のテンプレート構造、コマンド導出ロジック、スタイルガイドを定義する。

## 基本方針

- **80行以内** に収める（Claude Code が毎セッション読み込むため、簡潔さが最重要）
- **指示的な言語** を使う（「TypeScriptを使う」「pnpmで管理する」）— 記述的ではなく命令的
- **コピペ可能なコマンド** を記載する — 説明ではなく実行可能な形式
- **信号機システムは使わない** — CLAUDE.md は検出結果ではなく指示書
- **context.md と重複させない** — 詳細は context.md に委ね、CLAUDE.md はルールとコマンドに集中

## context.md との棲み分け

| 項目 | CLAUDE.md | context.md |
|------|-----------|------------|
| 目的 | Claude への行動指示 | プロジェクト情報の記録 |
| 読者 | Claude Code（毎セッション自動読み込み） | dev-* スキル群 |
| 内容 | コマンド、コードスタイル、プロジェクトルール | 技術スタック詳細、ディレクトリ構造、信号機付き情報 |
| トーン | 命令文（「〜を使う」「〜しない」） | 記述文（「〜を使用している」） |
| 行数 | 80行以内 | 500行以内 |

## ルート CLAUDE.md vs サブパッケージ CLAUDE.md の棲み分け

モノレポ構成（3層Web / BFF / マイクロサービス）では、ルートとサブパッケージの両方に CLAUDE.md を配置する。

| 項目 | ルート CLAUDE.md | サブパッケージ CLAUDE.md |
|------|-----------------|----------------------|
| 配置場所 | プロジェクトルート | `packages/<name>/` または `services/<name>/` |
| 行数制限 | 80行以内 | 30行以内 |
| コマンド | ワークスペースコマンド (`pnpm -r build` 等) | パッケージローカルコマンド (`pnpm build` 等) |
| Code Style | 共通コードスタイル（全パッケージ共通） | 混合言語時のみ記載（パッケージ固有の言語スタイル） |
| Project Rules | アーキテクチャルール（依存方向等） | パッケージ固有ルール（役割・責務） |
| Language | 記載する | 記載しない（ルートで定義済み） |
| 生成対象外 | — | `packages/shared/`（型定義パッケージ） |

## テンプレート構造

```markdown
# Project Overview

{Q1/Q2 プロジェクトタイプ} + {Q4 言語} + {Q6 フレームワーク} の1-2行要約。

## Development Commands

{言語×PM別のコマンド一覧}

## Code Style

{言語別のコーディング規約}

## Project Rules

{フレームワーク固有ルール + テスト規約 + Docker + 追加ツール}

## Language

日本語でコミュニケーションする。
```

## サブパッケージ CLAUDE.md テンプレート構造

モノレポのサブパッケージ（`packages/frontend/`, `packages/bff/`, `services/<name>/` 等）に配置する CLAUDE.md のテンプレート。30行以内に収める。

```markdown
# {package-name}

{パッケージの役割を1行で記述}

## Development Commands

{パッケージローカルのコマンド一覧}

## Code Style

{混合言語時のみ記載 — ルートと異なる言語の場合にパッケージ固有のスタイルを記載}

## Package Rules

{パッケージ固有のルール — 役割・責務・制約}
```

> **Note:** 同一言語のモノレポでは Code Style セクションを省略する（ルート CLAUDE.md に記載済みのため）。混合言語（例: TS frontend + Go backend）の場合のみ、ルートと異なる言語のパッケージに Code Style を追加する。

## Development Commands 導出テーブル

### TypeScript

| PM | ビルド | テスト | Lint | フォーマット | 開発サーバー |
|----|--------|--------|------|------------|------------|
| pnpm | `pnpm build` | `pnpm test` | `pnpm lint` | `pnpm format` | `pnpm dev` |
| bun | `bun run build` | `bun test` | `bun run lint` | `bun run format` | `bun dev` |
| yarn | `yarn build` | `yarn test` | `yarn lint` | `yarn format` | `yarn dev` |
| npm | `npm run build` | `npm test` | `npm run lint` | `npm run format` | `npm run dev` |

Linter/Formatter:
- Biome: `{pm} biome check --write .`
- ESLint + Prettier: `{pm} eslint . && {pm} prettier --write .`

### Python

| PM | ビルド | テスト | Lint | フォーマット | 開発サーバー |
|----|--------|--------|------|------------|------------|
| uv | — | `uv run pytest` | `uv run ruff check .` | `uv run ruff format .` | `uv run uvicorn app.main:app --reload` |
| pip | — | `pytest` | `ruff check .` | `ruff format .` | `uvicorn app.main:app --reload` |
| poetry | — | `poetry run pytest` | `poetry run ruff check .` | `poetry run ruff format .` | `poetry run uvicorn app.main:app --reload` |

Django の場合: 開発サーバーは `{pm-run} python manage.py runserver`

### Rust

| コマンド | 実行 |
|---------|------|
| ビルド | `cargo build` |
| テスト | `cargo test` |
| Lint | `cargo clippy -- -D warnings` |
| フォーマット | `cargo fmt` |
| 開発実行 | `cargo run` |
| Watch | `cargo watch -x run` |

### Go

| コマンド | 実行 |
|---------|------|
| ビルド | `go build ./...` |
| テスト | `go test ./...` |
| Lint | `golangci-lint run` |
| フォーマット | `gofmt -w .` |
| 開発サーバー | `air`（要 air インストール） |

### Java (Gradle)

| コマンド | 実行 |
|---------|------|
| ビルド | `./gradlew build` |
| テスト | `./gradlew test` |
| Lint | `./gradlew checkstyleMain` |
| フォーマット | `./gradlew spotlessApply` |
| 開発サーバー | `./gradlew bootRun` |

Maven の場合: `./gradlew` → `./mvnw` に置換

### Kotlin (Gradle)

| コマンド | 実行 |
|---------|------|
| ビルド | `./gradlew build` |
| テスト | `./gradlew test` |
| Lint | `./gradlew ktlintCheck` |
| フォーマット | `./gradlew ktlintFormat` |
| 開発サーバー | `./gradlew run` |

### Dart

| コマンド | 実行 |
|---------|------|
| ビルド | `dart compile exe` / `flutter build` |
| テスト | `dart test` / `flutter test` |
| Lint | `dart analyze` |
| フォーマット | `dart format .` |
| 開発サーバー | `flutter run` / `dart run` |

## Docker 時のコマンドプレフィクス

Q10 で Docker 開発環境を選択した場合、Development Commands のコマンドに `docker compose exec app` プレフィクスを付与する。

**適用ルール:**
- ビルド、テスト、Lint、フォーマットコマンド → プレフィクス付与
- 開発サーバー → `docker compose up` に置換（docker-compose.yml で定義済みのため）
- Docker 自体の操作コマンドは別セクションに記載:

```
docker compose up -d     # 開発環境起動
docker compose down      # 停止
docker compose logs -f   # ログ確認
```

## Code Style デフォルト

### TypeScript
- 命名: camelCase（変数・関数）、PascalCase（クラス・型・コンポーネント）
- エラーハンドリング: try-catch、カスタムエラークラス
- ファイル命名: kebab-case.ts（コンポーネントは PascalCase.tsx）

### Python
- 命名: snake_case（変数・関数）、PascalCase（クラス）
- エラーハンドリング: 例外、カスタム例外クラス
- ファイル命名: snake_case.py

### Rust
- 命名: snake_case（変数・関数）、PascalCase（型・トレイト）、SCREAMING_SNAKE_CASE（定数）
- エラーハンドリング: Result/Option、thiserror/anyhow
- ファイル命名: snake_case.rs

### Go
- 命名: camelCase（非公開）、PascalCase（公開）
- エラーハンドリング: error 返却、errors.New/fmt.Errorf
- ファイル命名: snake_case.go

### Java
- 命名: camelCase（変数・メソッド）、PascalCase（クラス）
- エラーハンドリング: チェック例外/非チェック例外
- ファイル命名: PascalCase.java

### Kotlin
- 命名: camelCase（変数・関数）、PascalCase（クラス）
- エラーハンドリング: runCatching、sealed class Result
- ファイル命名: PascalCase.kt

### Dart
- 命名: camelCase（変数・関数）、PascalCase（クラス）、snake_case（ライブラリ）
- エラーハンドリング: try-catch、カスタム例外
- ファイル命名: snake_case.dart

## チーム規模による調整（Q5）

| 項目 | 個人・プロトタイプ | 小規模（2-5人） | 中〜大規模（6人+） |
|------|------------------|----------------|-------------------|
| Code Style の記載量 | 最小限（3-5行） | 標準（5-8行） | 詳細（8-12行） |
| Project Rules | 基本のみ | + コードレビュー指針 | + 命名規約詳細・ドキュメント義務 |
| テスト規約 | 「テストを書く」 | 「単体+結合テストを書く」 | 「単体+結合+E2E、カバレッジ80%以上」 |
| Lint 厳格度 | warn レベル | error レベル | error + カスタムルール |

## 80行制限の圧縮優先順位

80行を超える場合、以下の優先順位で圧縮する（上ほど残す）:

1. **Development Commands** — 最重要。コマンドがないと開発できない
2. **Code Style** — 一貫性に直結。最低限の命名規則は残す
3. **Project Rules** — 圧縮可能。フレームワーク固有ルールを要約
4. **Project Overview** — 1行に短縮可能
5. **Language** — 1行のため圧縮不要

圧縮テクニック:
- コマンドをインラインで記載: `ビルド: \`pnpm build\` | テスト: \`pnpm test\` | Lint: \`pnpm lint\``
- ルールをリスト化して1行1ルール
- 自明なデフォルト（言語標準の命名規則等）は省略

## タイプ別 Project Rules テンプレート

### 3層Webアプリ

- packages/frontend, packages/backend, packages/shared に配置する
- 共有型定義は packages/shared に集約し、各パッケージから参照する
- 各パッケージは独立ビルド可能に保つ
- フロントエンドからバックエンドへの直接 import は禁止（shared 経由）
- API通信は型安全なクライアントを shared で定義する

### BFF

- UI層は表示ロジックのみ、BFF層はAPI集約とデータ整形のみ、サービス層がビジネスロジック担当
- UI→BFF→サービスの方向でのみ依存する（逆方向禁止）
- BFFはサービス層のAPIを集約・整形してUI向けに最適化したレスポンスを返す
- BFFにビジネスロジックを置かない
- **全層同一言語の場合**: サービス層のAPI契約は packages/shared で型定義する
- **混合言語の場合**: API契約は OpenAPI スキーマ or Protocol Buffers で定義し、packages/shared に配置する。各層は生成された型定義を使用する

### バッチ処理

- 全ジョブは冪等に実装する（再実行で同一結果）
- リトライ回数とタイムアウトを各ジョブに明示する
- 構造化ロギングを使用し、ジョブID・開始/終了時刻を必ず記録する
- 長時間ジョブは進捗ログを定期出力する

### マイクロサービス

- 各サービスは独立デプロイ可能に保つ
- サービス間のAPI契約は packages/shared で定義し、破壊的変更時はバージョニングする
- 共有コードは最小限にする（型定義・プロトコル定義のみ）
- サービス固有のデータストアは他サービスから直接アクセスしない

### イベント駆動

- イベントスキーマは src/events/ に集約し、バージョン管理する
- プロデューサーとコンシューマーは明確に分離する
- デッドレターキューを設定し、処理失敗イベントを捕捉する
- コンシューマーは冪等に実装する

### GAS

- clasp でデプロイする。手動でスクリプトエディタを編集しない
- TypeScript で記述し、esbuild でビルドする
- GAS の実行制限（6分/実行、30分/日等）を考慮して設計する
- グローバル関数のみが GAS から呼び出し可能。エクスポートは global オブジェクトに代入する
- テストは Vitest でローカル実行する（GAS API のモックが必要）

## GAS 用 Development Commands テーブル

| コマンド | 実行 |
|---------|------|
| ビルド | `pnpm build` |
| テスト | `pnpm test` |
| Lint | `pnpm lint` |
| デプロイ | `pnpm run push` (clasp push) |
| ログ確認 | `clasp logs` |
| ブラウザで開く | `clasp open` |

## 3層Web 用モノレポ Development Commands テーブル

| コマンド | 実行 |
|---------|------|
| 全体ビルド | `pnpm -r build` |
| 全体テスト | `pnpm -r test` |
| フロント開発 | `pnpm --filter frontend dev` |
| バックエンド開発 | `pnpm --filter backend dev` |
| Lint | `pnpm -r lint` |

## BFF 用モノレポ Development Commands テーブル

### 全層 TypeScript の場合

| コマンド | 実行 |
|---------|------|
| 全体ビルド | `pnpm -r build` |
| 全体テスト | `pnpm -r test` |
| UI開発 | `pnpm --filter web-ui dev` |
| BFF開発 | `pnpm --filter bff dev` |
| サービス開発 | `pnpm --filter service dev` / `pnpm --filter service-a dev` |
| Lint | `pnpm -r lint` |

### 混合言語の場合

全層が同一言語でない場合は、ワークスペース一括コマンド（`pnpm -r`）は使用せず、各パッケージのコマンドを個別に列挙する。各層の言語に対応するコマンドテーブル（TypeScript / Python / Go / Rust / Java / Kotlin / Dart）から導出する。

```
# UI層（例: TypeScript）
cd packages/web-ui && pnpm build / pnpm test / pnpm dev

# BFF層（例: Go）
cd packages/bff && go build ./... / go test ./...

# サービス層（例: Python）
cd packages/service && uv run pytest / uv run uvicorn ...
```

> Makefile + Docker Compose ベースのモノレポでは `make build-all`, `make test-all` 等のターゲットを定義してルート CLAUDE.md に記載する。

## サブパッケージ Development Commands 導出テーブル

サブパッケージの CLAUDE.md に記載するローカルコマンドは、ルートのワークスペースコマンドではなくパッケージ内で直接実行するコマンドを記載する。

### TypeScript パッケージ

| PM | ビルド | テスト | Lint | 開発サーバー |
|----|--------|--------|------|------------|
| pnpm | `pnpm build` | `pnpm test` | `pnpm lint` | `pnpm dev` |
| bun | `bun run build` | `bun test` | `bun run lint` | `bun dev` |
| yarn | `yarn build` | `yarn test` | `yarn lint` | `yarn dev` |
| npm | `npm run build` | `npm test` | `npm run lint` | `npm run dev` |

### 非 TypeScript パッケージ（混合言語時）

ルートの言語別コマンドテーブル（Python / Rust / Go / Java / Kotlin）をそのまま適用する。Docker 選択時は `docker compose exec app` プレフィクスを付与する。

## サブパッケージ Package Rules テンプレート

パッケージの役割に応じたルールテンプレート。各パッケージに該当するものを1-3行で記載する。

### frontend / web-ui

- UIコンポーネントと表示ロジックのみ配置する
- ビジネスロジックは配置しない
- API通信は shared の型定義を使用する

### mobile-ui

- モバイルアプリのUIとナビゲーションロジックのみ配置する
- BFF の API を呼び出してデータを取得する
- BFF の URL は環境変数で管理する

### bff

- API集約とデータ整形のみ行う
- ビジネスロジックは配置しない
- サービス層の API を集約して UI 向けに最適化する

### backend / service

- ビジネスロジックとデータアクセスを配置する
- API契約は shared の型定義に従う
- 他パッケージの内部実装に依存しない

### microservice（各 services/<name>）

- 独立デプロイ可能に保つ
- 他サービスのデータストアに直接アクセスしない
- API契約は shared で定義する

## 30行制限の圧縮ガイド

サブパッケージ CLAUDE.md が30行を超える場合の圧縮優先順位（上ほど残す）:

1. **Development Commands** — パッケージ内で実行するコマンドは必須
2. **Package Rules** — パッケージの役割・制約は必須（1-3行に圧縮）
3. **Code Style** — 混合言語時のみ。同一言語なら省略
4. **パッケージ説明** — 1行で十分

圧縮テクニック:
- コマンドをインラインで記載: `ビルド: \`pnpm build\` | テスト: \`pnpm test\` | Lint: \`pnpm lint\``
- ルートと重複するルールは省略（ルート CLAUDE.md で定義済み）
- Code Style はルートと異なる部分のみ記載

## 生成例: TypeScript + Hono + Docker（個人プロジェクト）

```markdown
# Project Overview

TypeScript + Hono の API サーバー。Docker 開発環境で PostgreSQL を使用。

## Development Commands

docker compose up -d     # 開発環境起動
docker compose down      # 停止
docker compose logs -f   # ログ確認

docker compose exec app pnpm build     # ビルド
docker compose exec app pnpm test      # テスト
docker compose exec app pnpm lint      # Lint
docker compose exec app pnpm format    # フォーマット

## Code Style

- 命名: camelCase（変数・関数）、PascalCase（型・クラス）
- ファイル命名: kebab-case.ts
- Biome でフォーマット・Lint を統合管理

## Project Rules

- ルーティングは src/routes/ に配置
- ビジネスロジックは src/services/ に分離
- テストは Vitest で tests/ 配下に配置
- 環境変数は .env を使用、.env はコミットしない

## Language

日本語でコミュニケーションする。
```

上記例: 約25行（80行制限内）

## 生成例: GAS（スプレッドシート連携・個人プロジェクト）

```markdown
# Project Overview

Google Apps Script（TypeScript）のスプレッドシート連携スクリプト。clasp + esbuild でビルド・デプロイ。

## Development Commands

pnpm build          # esbuild でビルド
pnpm test           # Vitest テスト実行
pnpm lint           # Biome lint
pnpm run push       # clasp push（GASへデプロイ）
clasp logs          # 実行ログ確認
clasp open          # ブラウザで開く

## Code Style

- 命名: camelCase（変数・関数）、PascalCase（型・クラス）
- ファイル命名: kebab-case.ts
- Biome でフォーマット・Lint を統合管理

## Project Rules

- GAS から呼び出す関数は global オブジェクトに代入する
- GAS の実行制限（6分/実行）を考慮し、大量データは分割処理する
- GAS API のモックを使ってローカルテストする
- clasp でデプロイする。手動でスクリプトエディタを編集しない

## Language

日本語でコミュニケーションする。
```

上記例: 約25行（80行制限内）

## 生成例: BFF（TypeScript + Next.js + Hono + Docker・小規模チーム）

```markdown
# Project Overview

BFFアーキテクチャの Webアプリ。Next.js（UI）+ Hono（BFF）+ Hono（サービス）+ PostgreSQL。Docker 開発環境。

## Development Commands

docker compose up -d       # 開発環境起動
docker compose down        # 停止
docker compose logs -f     # ログ確認

pnpm -r build              # 全体ビルド
pnpm -r test               # 全体テスト
pnpm --filter web-ui dev   # UI開発
pnpm --filter bff dev      # BFF開発
pnpm --filter service dev  # サービス開発
pnpm -r lint               # Lint

## Code Style

- 命名: camelCase（変数・関数）、PascalCase（型・クラス・コンポーネント）
- ファイル命名: kebab-case.ts（コンポーネントは PascalCase.tsx）
- Biome でフォーマット・Lint を統合管理
- 共有型定義は packages/shared に配置

## Project Rules

- UI層は表示ロジックのみ、BFF層はAPI集約・整形のみ、サービス層がビジネスロジック担当
- UI→BFF→サービスの方向でのみ依存する（逆方向禁止）
- サービス層のAPI契約は packages/shared で型定義する
- テストは Vitest で各パッケージに配置
- 環境変数は .env を使用、.env はコミットしない

## Language

日本語でコミュニケーションする。
```

上記例: 約35行（80行制限内）

## サブパッケージ生成例: 同一言語 BFF（TypeScript + Hono）

### packages/web-ui/CLAUDE.md (~12行)

```markdown
# web-ui

Next.js App Router のフロントエンド UI パッケージ。

## Development Commands

pnpm build          # ビルド
pnpm test           # テスト
pnpm dev            # 開発サーバー
pnpm lint           # Lint

## Package Rules

- UIコンポーネントと表示ロジックのみ配置する
- ビジネスロジックは配置しない
- API通信は shared の型定義を使用する
```

### packages/bff/CLAUDE.md (~12行)

```markdown
# bff

Hono の BFF（Backend for Frontend）パッケージ。サービス層の API を集約・整形する。

## Development Commands

pnpm build          # ビルド
pnpm test           # テスト
pnpm dev            # 開発サーバー
pnpm lint           # Lint

## Package Rules

- API集約とデータ整形のみ行う
- ビジネスロジックは配置しない
- サービス層の API を集約して UI 向けに最適化する
```

## サブパッケージ生成例: 混合言語 3層Web（TypeScript + Go）

### packages/frontend/CLAUDE.md (~10行)

```markdown
# frontend

Next.js App Router のフロントエンドパッケージ。

## Development Commands

pnpm build          # ビルド
pnpm test           # テスト
pnpm dev            # 開発サーバー
pnpm lint           # Lint

## Package Rules

- UIコンポーネントと表示ロジックのみ配置する
- API通信は shared の型定義を使用する
```

### packages/backend/CLAUDE.md (~22行, Code Style 付き)

```markdown
# backend

Go + Echo のバックエンド API パッケージ。

## Development Commands

go build ./...          # ビルド
go test ./...           # テスト
golangci-lint run       # Lint
gofmt -w .              # フォーマット
air                     # 開発サーバー

## Code Style

- 命名: camelCase（非公開）、PascalCase（公開）
- エラーハンドリング: error 返却、errors.New/fmt.Errorf
- ファイル命名: snake_case.go

## Package Rules

- ビジネスロジックとデータアクセスを配置する
- API契約は shared の型定義に従う
- 他パッケージの内部実装に依存しない
```
