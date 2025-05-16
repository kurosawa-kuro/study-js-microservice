# Node.js（TypeScript）+ Express マイクロサービスアーキテクチャ仕様書

## 1. 目的

本ドキュメントは **Node.js（TypeScript）+ Express** を用いたマイクロサービス基盤の正式仕様書である。サービス責務・通信方式・インフラ構成・開発運用手順を明確化し、実装とデプロイの一貫性を保証する。

---

## 2. アーキテクチャ概要

```
┌──────────────┐         ┌────────────────┐
│  API Gateway │ <HTTP> │  user-service   │
│   (Nginx)    │  →     └──────┬─────────┘
└──────┬─────────┘            │RabbitMQ
       │                      │events
       │                      ▼
       │            ┌────────────────┐
       │   <HTTP>   │ micropost‑svc  │
       │   →        └──────┬─────────┘
       │                   │RabbitMQ
       │                   ▼
       │            ┌────────────────┐
       │   <HTTP>   │ category‑svc   │
       │   →        └────────────────┘
       ▼
┌───────────────────────┐
│   RabbitMQ (fan‑out)  │
└───────────────────────┘
```

* **Gateway** が外部トラフィックを受け、サービス名パスでルーティング。
* **RabbitMQ** によりサービス間を疎結合かつ非同期化。
* 各サービスは専用 **PostgreSQL** を保持し、**Prisma** ORM で操作する。

---

## 3. サービス一覧

| サービス                  | ポート  | 主責務                                   | 主要エンドポイント                                 | 依存リソース                 |
| --------------------- | ---- | ------------------------------------- | ----------------------------------------- | ---------------------- |
| **user-service**      | 3000 | ユーザ CRUD, UserCreated イベント発行          | `POST /users`, `GET /users/:id`           | user-db, RabbitMQ      |
| **micropost-service** | 3000 | マイクロポスト CRUD, MicropostCreated イベント発行 | `POST /microposts`, `GET /microposts/:id` | micropost-db, RabbitMQ |
| **category-service**  | 3000 | カテゴリ CRUD, カテゴリと Micropost の関連付け      | `POST /categories`, `GET /categories/:id` | category-db, RabbitMQ  |
| **api-gateway**       | 80   | ルーティング／LB／CORS                        | `/user-service/*` など                      | ‑                      |

> すべて **TypeScript + Express + express‑async‑errors** 実装。非同期例外はグローバルハンドラへ集約。

---

## 4. データストア設計

### 4.1 PostgreSQL（サービス毎）

* **バージョン**: 16‑alpine
* **接続文字列例**: `postgresql://user:user@user-db:5432/user`
* **マイグレーション**: Prisma Migrate
* **バックアップ**: 本番では Amazon RDS/Aurora のスナップショットに委譲

### 4.2 Prisma スキーマ雛形

```prisma
model User {
  id          String       @id @default(uuid())
  name        String
  email       String       @unique
  microposts  Micropost[]
  createdAt   DateTime     @default(now())
}

model Micropost {
  id        String       @id @default(uuid())
  content   String
  userId    String
  user      User         @relation(fields: [userId], references: [id])
  categories Category[]  @relation("MicropostCategories", references: [id])
  createdAt DateTime     @default(now())
}

model Category {
  id         String        @id @default(uuid())
  name       String        @unique
  microposts Micropost[]   @relation("MicropostCategories")
}
```

（多対多は Prisma の IMPLICIT 方式を採用）

---

## 5. 非同期メッセージング

| イベント                | Exchange            | Type   | 発行元               | サブスクライバ                     |
| ------------------- | ------------------- | ------ | ----------------- | --------------------------- |
| `user.created`      | `user.created`      | fanout | user-service      | micropost-service           |
| `micropost.created` | `micropost.created` | fanout | micropost-service | category-service (optional) |

* **ライブラリ**: `amqplib` を `shared/mq.ts` にラップ。
* **メッセージフォーマット**: JSON + `messageId`, `timestamp`, `payload`。

---

## 6. API Gateway (Nginx) 設定例

```nginx
location /user-service/ {
  rewrite /user-service/(.*) /$1 break;
  proxy_pass http://user-service:3000;
}
location /micropost-service/ {
  rewrite /micropost-service/(.*) /$1 break;
  proxy_pass http://micropost-service:3000;
}
location /category-service/ {
  rewrite /category-service/(.*) /$1 break;
  proxy_pass http://category-service:3000;
}
```

---

## 7. コンテナ & Docker Compose（抜粋）

```yaml
services:
  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    ports: ["5672:5672", "15672:15672"]

  user-service:
    build: ./services/user-service
    environment:
      DATABASE_URL: ${USER_DATABASE_URL}
      RABBITMQ_URL: amqp://rabbitmq
    depends_on: [user-db, rabbitmq]

  user-db:
    image: postgres:16-alpine
    env_file: .env
    volumes: [user-data:/var/lib/postgresql/data]

  # micropost-service, category-service 同様

volumes:
  user-data:
```

---

## 9. コーディングガイドライン（抜粋）

1. `strict` TypeScript。
2. DTO 検証は `zod`。
3. ロギングは `pino`（JSON 出力）。
4. テストは `Jest + SuperTest`。

---

## 10. 開発ワークフロー

```mermaid
graph LR
A[git clone] --> B[npm install]
B --> C[prisma migrate dev]
C --> D[docker compose up -d]
D --> E[ts-node-dev src/index.ts]
```

---

## 11. 今後の拡張候補

Kafka 移行・gRPC 採用・CQRS/ES・DDD 等、前版と同一方針。

---

## 12. 署名

*ドキュメント作成者*: 黒澤 俊文（リモートウェブプログラマー）

この Node.js + Express 版マイクロサービス基盤の仕様を確定する。実装フェーズ開始前に関係者レビューを実施し、承認を得ること。
