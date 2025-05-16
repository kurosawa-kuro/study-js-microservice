# study-js-microservice以下では、既存の **Spring Boot＋RabbitMQ＋PostgreSQL** マイクロサービスを
**Node.js（TypeScript）＋Express＋RabbitMQ＋PostgreSQL** へ置き換えるための
調整案をまとめました。⚙️✨
（学習用のスタブとして動く → 実戦で拡張できる、を念頭に置いています）

---

## 1. 全体方針

| 項目         | Spring Boot 版     | Node.js 版（提案）                                                  |
| ---------- | ----------------- | -------------------------------------------------------------- |
| 言語         | Java              | TypeScript（js でも可）                                             |
| フレームワーク    | Spring Boot       | Express + express-async-errors                                 |
| ORM / クエリ  | JPA (Hibernate)   | **Prisma** もしくは **TypeORM**<br>（PostgreSQL に最適・型安全・マイグレーション内蔵） |
| 非同期メッセージ   | Spring AMQP       | **amqplib**（生 AMQP）または **RabbitMQ Streams** SDK                |
| ログ         | SLF4J + Logback   | **pino**（高速 JSON ロガー）                                          |
| バリデーション    | Bean Validation   | **class-validator** / **zod**                                  |
| テスト        | JUnit             | **Jest + SuperTest**                                           |
| ビルド        | Maven / Gradle    | **ts-node-dev**（開発用）＋`tsc`（本番）                                 |
| API ドキュメント | SpringDoc OpenAPI | **Swagger-UI Express** + `openapi.json` 生成                     |

---

## 2. ディレクトリ構成（例）

```
spring-to-node-ms/
├─ services/
│  ├─ author-service/
│  │  ├─ src/
│  │  │  ├─ controllers/
│  │  │  ├─ routes/
│  │  │  ├─ models/
│  │  │  ├─ mq/            # RabbitMQ publish/subscribe
│  │  │  ├─ index.ts       # app エントリ
│  │  ├─ prisma/           # schema.prisma / migrations
│  │  └─ Dockerfile
│  ├─ book-service/
│  └─ comment-service/
├─ api-gateway/            # nginx.conf だけ変更
├─ shared/                 # 共通ライブラリ（logger, dto, errors…）
└─ docker-compose.yml
```

* **shared/** は npm workspace にして `@org/common` として各サービスへ依存。
* TypeScript 設定（`tsconfig.base.json`）もここで一元管理。

---

## 3. docker-compose（抜粋）

```yaml
version: "3.9"
services:

  # ---------- RabbitMQ ----------
  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    ports: ["5672:5672", "15672:15672"]

  # ---------- Author Service ----------
  author-service:
    build: ./services/author-service
    environment:
      - DATABASE_URL=postgresql://author:author@author-db:5432/author
      - RABBITMQ_URL=amqp://rabbitmq
    depends_on: [author-db, rabbitmq]
    ports: ["8081:3000"]

  author-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: author
      POSTGRES_PASSWORD: author
      POSTGRES_DB: author
    volumes: [author-data:/var/lib/postgresql/data]
    ports: ["5433:5432"]

  # …… book / comment も同様に定義 ……

volumes:
  author-data:
  # book-data:
  # comment-data:
```

> **📝 ポイント**
>
> * 各マイクロサービスが **独立 DB** を持つ 12-Factor 構成を維持。
> * 本番は DB だけマネージド（RDS/Aurora）に切り替える。

---

## 4. API ゲートウェイ（Nginx）修正

```nginx
location /author-service/ {
    rewrite /author-service/(.*) /$1 break;
    proxy_pass http://author-service:3000;
}
# /book-service, /comment-service も同様に 3000 ポートへ
```

*サービス名 or ポートのみ変更* で済むため、既存ルールは再利用できます。

---

## 5. サービス実装サンプル  — Book Service `createBook`

### 5.1 ルーター `routes/book.routes.ts`

```ts
import { Router } from "express";
import { createBook } from "../controllers/book.controller";

export const bookRouter = Router();

bookRouter.post("/", createBook);
```

### 5.2 コントローラ `controllers/book.controller.ts`

```ts
import { Request, Response } from "express";
import { prisma } from "../utils/prisma";          // Prisma Client
import { publishBookCreated } from "../mq/publisher";
import { z } from "zod";

const CreateBookDto = z.object({
  title: z.string().min(1),
  description: z.string().optional(),
  authors: z.array(z.string().uuid()).optional(),
});

export async function createBook(req: Request, res: Response) {
  const dto = CreateBookDto.parse(req.body);

  const book = await prisma.book.create({
    data: {
      title: dto.title,
      description: dto.description,
      authors: { set: dto.authors ?? [] },
    },
  });

  await publishBookCreated(book);                 // RabbitMQ へ送信

  return res.status(201).json(book);
}
```

### 5.3 RabbitMQ パブリッシュ `mq/publisher.ts`

```ts
import amqplib from "amqplib";
const EXCHANGE = "book.created";

export async function publishBookCreated(payload: unknown) {
  const conn = await amqplib.connect(process.env.RABBITMQ_URL!);
  const ch = await conn.createChannel();
  await ch.assertExchange(EXCHANGE, "fanout", { durable: true });
  ch.publish(EXCHANGE, "", Buffer.from(JSON.stringify(payload)), {
    contentType: "application/json",
    messageId: crypto.randomUUID(),
    timestamp: Date.now() / 1000,
  });
  await ch.close();
  await conn.close();
}
```

> **検討:**
> ❶ 接続プール化／チャネル再利用を行い、パフォーマンス最適化
> ❷ [RabbitMQ Streams](https://rabbitmq.com/streams.html) 採用で堅牢性向上

---

## 6. Kubernetes マニフェスト差分

* **Deployment**

  * `image: ghcr.io/your-org/book-service:1.0.0`
  * `ports.containerPort: 3000`
  * `livenessProbe.httpGet.path: /healthz`
* **ConfigMap / Secret**

  * `DATABASE_URL`, `RABBITMQ_URL`
* **HorizontalPodAutoscaler**

  * CPU 50 % でスケールアウトなど

Helm Chart を一本作り、値ファイルで book/author/comment を切り替えると運用が楽です。
（Argo CD へは Helm 推奨）

---

## 7. 学習・実装ステップ案

1. **共通テンプレート**

   * `npm init -w services/author-service` でワークスペース化
   * ESLint / Prettier / ts-node-dev 設定

2. **DB スキーマ → Prisma**

   * `npx prisma init --datasource-provider postgresql`
   * `prisma migrate dev` でローカル DB に反映

3. **RabbitMQ 接続ユーティリティ** を shared に切り出し

   * publish/subscribe をジェネリック型で包む

4. **各サービス** を順に実装

   1. REST（CRUD）
   2. メッセージ送信
   3. メッセージ受信→集約（Book←Author など）

5. **コンテナ化**

   * multi-stage Dockerfile（alpine, non-root）
   * `docker compose up` で E2E 確認

6. **CI/CD**

   * GitHub Actions → Docker build & push (ghcr)
   * Argo CD で自動同期

---

## 8. 参考リソース

* **公式 Doc**

  * Express  [https://expressjs.com/](https://expressjs.com/)
  * Prisma   [https://www.prisma.io/docs](https://www.prisma.io/docs)
  * RabbitMQ  [https://rabbitmq.com/tutorials/](https://rabbitmq.com/tutorials/)
* **教材**

  * “Microservices with Node.js & RabbitMQ” (Udemy)
  * **Prisma Data Platform** のサンプルリポジトリ
* **ベストプラクティス**

  * [Node.js Production Best Practices](https://github.com/goldbergyoni/nodebestpractices)

---

### まとめ

* **サービス境界・RabbitMQ イベント・API ゲートウェイ** はそのまま流用し、
  コンテナ／コードだけを **Spring → Express** に置き換えるのが最小コスト。
* TypeScript ＋ Prisma で **型安全かつ高速** な CRUD 開発が可能。
* 将来の **Kafka** や **gRPC** への発展も、このレイヤ分割なら容易です。

ご不明点や追加のサンプルコードが必要な箇所があれば、お気軽にお知らせください。
