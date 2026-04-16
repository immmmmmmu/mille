# Docker 開発環境テンプレート

dev-init スキルが Docker 開発環境を構築する際に参照するテンプレート集。Q10 で「Docker開発環境」を選択した場合に使用する。

**注意:** ビルド成果物ディレクトリ（`.next`, `dist`, `build`, `target`, `__pycache__` 等）はホストとバインドマウントしないこと。コンテナ内で完結させる。named volume で分離するか、マウント対象外にする。

## 1. 言語別 app サービステンプレート

### TypeScript (Node.js)

```yaml
  app:
    image: node:20-slim
    working_dir: /app
    volumes:
      - .:/app
      - node_modules:/app/node_modules
    ports:
      - "3000:3000"
    command: ["pnpm", "dev"]
    environment:
      - NODE_ENV=development
```

**Dockerfile.dev:**
```dockerfile
FROM node:20-slim
RUN corepack enable && corepack prepare pnpm@latest --activate
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile
COPY . .
CMD ["pnpm", "dev"]
```

Bun の場合: イメージを `oven/bun:1` に変更、`pnpm` を `bun` に置換。

### Python

```yaml
  app:
    image: python:3.12-slim
    working_dir: /app
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    command: ["uv", "run", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--reload"]
    environment:
      - PYTHONDONTWRITEBYTECODE=1
      - PYTHONUNBUFFERED=1
```

**Dockerfile.dev:**
```dockerfile
FROM python:3.12-slim
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync
COPY . .
CMD ["uv", "run", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--reload"]
```

Django の場合: コマンドを `uv run python manage.py runserver 0.0.0.0:8000` に変更。

### Rust

```yaml
  app:
    image: rust:1.82
    working_dir: /app
    volumes:
      - .:/app
      - cargo-cache:/usr/local/cargo/registry
      - target-cache:/app/target
    ports:
      - "8080:8080"
    command: ["cargo", "watch", "-w", "src", "-w", "Cargo.toml", "-x", "run"]
    environment:
      - CARGO_HOME=/usr/local/cargo
```

**Dockerfile.dev:**
```dockerfile
FROM rust:1.82
RUN cargo install cargo-watch
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs && cargo build && rm -rf src
COPY . .
CMD ["cargo", "watch", "-w", "src", "-w", "Cargo.toml", "-x", "run"]
```

### Go

```yaml
  app:
    image: golang:1.22
    working_dir: /app
    volumes:
      - .:/app
      - go-mod-cache:/go/pkg/mod
    ports:
      - "8080:8080"
    command: ["air"]
    environment:
      - CGO_ENABLED=0
```

**Dockerfile.dev:**
```dockerfile
FROM golang:1.22
RUN go install github.com/air-verse/air@latest && \
    go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
CMD ["air"]
```

### Java (Spring Boot / Quarkus)

```yaml
  app:
    image: eclipse-temurin:21
    working_dir: /app
    volumes:
      - .:/app
      - gradle-cache:/root/.gradle
    ports:
      - "8080:8080"
    command: ["./gradlew", "bootRun"]
    environment:
      - JAVA_TOOL_OPTIONS=-Dspring.devtools.restart.enabled=true
```

**Dockerfile.dev:**
```dockerfile
FROM eclipse-temurin:21
WORKDIR /app
COPY gradlew gradlew.bat ./
COPY gradle/ gradle/
COPY build.gradle.kts settings.gradle.kts ./
RUN ./gradlew dependencies --no-daemon
COPY . .
CMD ["./gradlew", "bootRun"]
```

Quarkus の場合: コマンドを `./gradlew quarkusDev` に変更。

### Kotlin (Ktor / Spring Boot)

```yaml
  app:
    image: eclipse-temurin:21
    working_dir: /app
    volumes:
      - .:/app
      - gradle-cache:/root/.gradle
    ports:
      - "8080:8080"
    command: ["./gradlew", "run", "--continuous"]
    environment:
      - JAVA_TOOL_OPTIONS=-XX:+UseContainerSupport
```

**Dockerfile.dev:**
```dockerfile
FROM eclipse-temurin:21
WORKDIR /app
COPY gradlew gradlew.bat ./
COPY gradle/ gradle/
COPY build.gradle.kts settings.gradle.kts ./
RUN ./gradlew dependencies --no-daemon
COPY . .
CMD ["./gradlew", "run", "--continuous"]
```

Spring Boot (Kotlin) の場合: コマンドを `./gradlew bootRun` に変更。

### Dart (dart_frog / shelf)

```yaml
  app:
    image: dart:stable
    working_dir: /app
    volumes:
      - .:/app
      - pub-cache:/root/.pub-cache
    ports:
      - "8080:8080"
    command: ["dart", "run", "--enable-vm-service", "bin/server.dart"]
    environment:
      - DART_VM_OPTIONS=--enable-asserts
```

**Dockerfile.dev:**
```dockerfile
FROM dart:stable
WORKDIR /app
COPY pubspec.yaml pubspec.lock ./
RUN dart pub get
COPY . .
CMD ["dart", "run", "--enable-vm-service", "bin/server.dart"]
```

dart_frog の場合: `dart pub global activate dart_frog_cli` を追加し、コマンドを `dart_frog dev` に変更。

## 2. DB サービステンプレート

### PostgreSQL

```yaml
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: app_dev
    ports:
      - "5432:5432"
    volumes:
      - db-data:/var/lib/postgresql/data
```

app サービスに追加する環境変数:
```yaml
    environment:
      - DATABASE_URL=postgresql://dev:dev@db:5432/app_dev
```

### SQLite

SQLite はファイルベースのため、DB サービス不要。app サービスの volume mount で対応。

```yaml
    environment:
      - DATABASE_URL=sqlite:///app/data/app.db
    volumes:
      - ./data:/app/data
```

### MongoDB

```yaml
  db:
    image: mongo:7
    ports:
      - "27017:27017"
    volumes:
      - db-data:/data/db
```

app サービスに追加する環境変数:
```yaml
    environment:
      - MONGODB_URL=mongodb://db:27017/app_dev
```

## 3. Playwright サービス（Webアプリ用）

Webアプリ選択時、dev-webtest スキルの Playwright サービスを docker-compose.yml に追加する。

設定の詳細は `.claude/skills/dev-webtest/references/docker-setup.md` を参照すること。

基本構成:
```yaml
  playwright:
    image: mcr.microsoft.com/playwright:v1.50.0-noble
    profiles: [test]
    working_dir: /work
    volumes:
      - .:/work
    depends_on:
      - app
    command: ["tail", "-f", "/dev/null"]
    environment:
      - PLAYWRIGHT_BROWSERS_PATH=/ms-playwright
```

- `profiles: [test]` により通常の `docker compose up -d` では起動しない。テスト時は `docker compose --profile test up playwright` で起動する
- `depends_on: [app]` でアプリ起動後に Playwright コンテナが起動する
- volume mount でスクリーンショット共有（`tmp/webtest/screenshots/`）
- `@playwright/cli` のインストールが必要（詳細は dev-webtest 参照）

## 4. 追加サービステンプレート

### Redis（キャッシュ・セッション）

```yaml
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
```

app サービスに追加する環境変数:
```yaml
    environment:
      - REDIS_URL=redis://redis:6379
```

## 5. マルチサービス構成テンプレート

### 3層Webアプリ

```yaml
services:
  frontend:
    build:
      context: ./packages/frontend
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - ./packages/frontend:/app
      - frontend_node_modules:/app/node_modules
    depends_on:
      - backend
    command: ["pnpm", "dev"]
    environment:
      - API_URL=http://backend:8080

  backend:
    # Q4c言語テンプレートを使用
    build:
      context: ./packages/backend
      dockerfile: Dockerfile.dev
    ports:
      - "8080:8080"
    volumes:
      - ./packages/backend:/app
    depends_on:
      - db
    environment:
      - DATABASE_URL=postgresql://dev:dev@db:5432/app_dev

  db:
    # Q8 DB テンプレートを使用
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: app_dev
    ports:
      - "5432:5432"
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  frontend_node_modules:
  db-data:
```

### BFF（Webのみ・単一サービス）

Q3（UIターゲット）が「Webのみ」の場合。各層の言語が異なる場合は、bff と service のテンプレートをそれぞれ Q4b/Q4c 言語に対応するセクション1の言語別テンプレートに差し替える。

```yaml
services:
  web-ui:
    # Q4a言語テンプレートを使用（Webのみ: TypeScript自動選定）
    build:
      context: ./packages/web-ui
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - ./packages/web-ui:/app
      - webui_node_modules:/app/node_modules
    depends_on:
      - bff
    command: ["pnpm", "dev"]
    environment:
      - BFF_URL=http://bff:4000

  bff:
    # Q4b言語テンプレートを使用
    build:
      context: ./packages/bff
      dockerfile: Dockerfile.dev
    ports:
      - "4000:4000"
    volumes:
      - ./packages/bff:/app
      - bff_node_modules:/app/node_modules
    depends_on:
      - service
    command: ["pnpm", "dev"]
    environment:
      - SERVICE_URL=http://service:8080

  service:
    # Q4c言語テンプレートを使用
    build:
      context: ./packages/service
      dockerfile: Dockerfile.dev
    ports:
      - "8080:8080"
    volumes:
      - ./packages/service:/app
    depends_on:
      - db
    environment:
      - DATABASE_URL=postgresql://dev:dev@db:5432/app_dev

  db:
    # Q8 DB テンプレートを使用
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: app_dev
    ports:
      - "5432:5432"
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  webui_node_modules:
  bff_node_modules:
  db-data:
```

### BFF（モバイルのみ・単一サービス）

Q3（UIターゲット）が「モバイルのみ」の場合。モバイル UI は Docker 外で実行するため web-ui サービスを省略する。bff + service + db の3サービス構成。

```yaml
services:
  # モバイル UI は Docker 外で実行（Flutter/React Native 等のネイティブ開発環境を使用）

  bff:
    # Q4b言語テンプレートを使用
    build:
      context: ./packages/bff
      dockerfile: Dockerfile.dev
    ports:
      - "4000:4000"
    volumes:
      - ./packages/bff:/app
      - bff_node_modules:/app/node_modules
    depends_on:
      - service
    command: ["pnpm", "dev"]
    environment:
      - SERVICE_URL=http://service:8080

  service:
    # Q4c言語テンプレートを使用
    build:
      context: ./packages/service
      dockerfile: Dockerfile.dev
    ports:
      - "8080:8080"
    volumes:
      - ./packages/service:/app
    depends_on:
      - db
    environment:
      - DATABASE_URL=postgresql://dev:dev@db:5432/app_dev

  db:
    # Q8 DB テンプレートを使用
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: app_dev
    ports:
      - "5432:5432"
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  bff_node_modules:
  db-data:
```

> **Note:** モバイルアプリからの BFF 接続は `http://localhost:4000`（エミュレータ）または実機のネットワーク IP を使用する。

### BFF（複数サービス）

```yaml
services:
  web-ui:
    build:
      context: ./packages/web-ui
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - ./packages/web-ui:/app
      - webui_node_modules:/app/node_modules
    depends_on:
      - bff
    command: ["pnpm", "dev"]

  bff:
    build:
      context: ./packages/bff
      dockerfile: Dockerfile.dev
    ports:
      - "4000:4000"
    volumes:
      - ./packages/bff:/app
      - bff_node_modules:/app/node_modules
    depends_on:
      - service-a
      - service-b
    command: ["pnpm", "dev"]

  service-a:
    build:
      context: ./services/service-a
      dockerfile: Dockerfile.dev
    ports:
      - "8081:8080"
    volumes:
      - ./services/service-a:/app
    depends_on:
      - db

  service-b:
    build:
      context: ./services/service-b
      dockerfile: Dockerfile.dev
    ports:
      - "8082:8080"
    volumes:
      - ./services/service-b:/app
    depends_on:
      - db

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: app_dev
    ports:
      - "5432:5432"
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  webui_node_modules:
  bff_node_modules:
  db-data:
```

### マイクロサービス

```yaml
services:
  service-a:
    build:
      context: ./services/service-a
      dockerfile: Dockerfile.dev
    ports:
      - "8081:8080"
    volumes:
      - ./services/service-a:/app

  service-b:
    build:
      context: ./services/service-b
      dockerfile: Dockerfile.dev
    ports:
      - "8082:8080"
    volumes:
      - ./services/service-b:/app

  # Q8 で DB 選択時のみ追加
  # db:
  #   ...
```

## 6. メッセージブローカーサービステンプレート

### Apache Kafka

```yaml
  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
```

app サービスに追加する環境変数:
```yaml
    environment:
      - KAFKA_BROKERS=kafka:29092
```

### RabbitMQ

```yaml
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: dev
      RABBITMQ_DEFAULT_PASS: dev
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
```

app サービスに追加する環境変数:
```yaml
    environment:
      - AMQP_URL=amqp://dev:dev@rabbitmq:5672
```

管理UI: `http://localhost:15672`（dev/dev）

### Redis Pub/Sub

Redis Pub/Sub の場合はセクション4の Redis テンプレートを再利用する。

## 7. docker-compose.yml 組み立てガイド

### 基本構造

```yaml
version: "3.8"

services:
  # 1. app サービス（言語別テンプレートから選択）
  app:
    # ...

  # 2. DB サービス（Q8 の選択に応じて追加、「なし」の場合は省略）
  db:
    # ...

  # 3. Playwright サービス（Webアプリの場合のみ追加）
  playwright:
    # ...

volumes:
  # 言語・DB に応じた named volumes
  # ...
```

### 組み立てルール

1. **app サービス**（フルスタック/APIサーバー/バッチ/CLIの場合）: Q4c（言語）に対応するテンプレートを選択
2. **DB サービス**: Q8（データベース）の選択に応じて追加。「なし」の場合は DB サービスと関連 volume を省略
3. **Playwright サービス**: Q2 が「フルスタック」「3層Web」「BFF（Q3がWebのみ/Web+モバイル）」の場合のみ追加。dev-webtest の docker-setup.md を参照
4. **app → DB 接続**: DB サービス追加時、app サービスに `depends_on: [db]` と `DATABASE_URL` 環境変数を追加
5. **volumes**: 各サービスが必要とする named volume を `volumes:` セクションにまとめる
6. **3層Web**: セクション5の3層テンプレートを使用（frontend + backend + db の3サービス構成）
7. **BFF**: セクション5のBFFテンプレートを使用。Q3（UIターゲット）の回答でサービス構成が変わる: Webのみ → web-ui + bff + service(s) + db、モバイルのみ → bff + service(s) + db（web-ui なし、モバイルは Docker 外）、Web+モバイル → web-ui + bff + service(s) + db（mobile-ui は Docker 外）。Q7（サービス層構成）の回答で単一/複数を選択
8. **マイクロサービス**: セクション5のマイクロサービステンプレートを使用。各サービスは個別ビルド
9. **イベント駆動**: app サービス + セクション6のブローカーテンプレート（Q7 の回答に応じて選択）を組み合わせる
10. **メッセージブローカー**: Q7（イベント駆動のメッセージブローカー / マイクロサービスのメッセージキュー選択時）に応じてセクション6から選択して追加

### named volumes 一覧

| 言語 | volumes |
|------|---------|
| TypeScript | `node_modules` |
| Python | （不要） |
| Rust | `cargo-cache`, `target-cache` |
| Go | `go-mod-cache` |
| Java | `gradle-cache` |
| Kotlin | `gradle-cache` |
| Dart | `pub-cache` |
| 共通（PostgreSQL） | `db-data` |
| 共通（MongoDB） | `db-data` |
| 共通（Redis） | `redis-data` |
| 共通（RabbitMQ） | `rabbitmq-data` |
| 3層Web/BFF（フロント） | `frontend_node_modules` / `webui_node_modules` |
| BFF（BFF層・TS） | `bff_node_modules` |
| BFF（BFF層・Go） | `bff_go_mod_cache` |
| BFF（BFF層・Rust） | `bff_cargo_cache`, `bff_target_cache` |
| BFF（BFF層・Java/Kotlin） | `bff_gradle_cache` |
| BFF（BFF層・Python） | （不要） |

### .dockerignore テンプレート

```
.git
.env*
*.log
tmp/
```

言語固有の除外は各 scaffolding ファイルの Docker ガイドを参照。
