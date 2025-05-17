# RabbitMQ Integration Code Samples (TypeScript) — **シンプル同期版**

Micropost が作成されたら、**他サービスがただちにイベントを受信し、自分のデータベースへ Micropost 情報をコピーする**──それだけ、という最小構成を示します。

---

## 1. 概要

```
┌────────────────┐             ┌────────────────┐
│ micropost‑svc   │ ──▶ AMQP ─▶ │ category‑svc    │ (INSERT INTO category_db.microposts)
└────────────────┘             └────────────────┘
        │                           ▲
        │                           │
        │             ┌──────────────────────────┐
        └──▶ AMQP ───▶ │ user‑svc (timeline)      │ (INSERT INTO user_db.timelines)
                      └──────────────────────────┘
```

* **Micropost Service** … Micropost を自サービス DB に保存し *micropost.created.v1* を発行。
* **Category / User Service** … イベントを受信して自サービスのテーブルへ `INSERT` するだけ。
* **一方向**・**1 イベント** だけで完結。Outbox や二段階 Sync は不要。

---

## 2. ディレクトリレイアウト

```
shared/
  mq.ts                        # シンプル AMQP ユーティリティ
services/
  micropost-service/
    src/events/publish.ts      # MicropostCreated 発行
  category-service/
    src/events/micropost.listener.ts  # 受信してINSERT
  user-service/
    src/events/micropost.listener.ts  # 受信してINSERT
```

---

## 3. `shared/mq.ts` — 最小実装

```ts
import amqplib from "amqplib";
import crypto from "crypto";

const URL = process.env.RABBITMQ_URL || "amqp://localhost";

export async function publish(exchange: string, payload: unknown) {
  const conn = await amqplib.connect(URL);
  const ch = await conn.createChannel();
  await ch.assertExchange(exchange, "fanout", { durable: true });
  ch.publish(exchange, "", Buffer.from(JSON.stringify(payload)), {
    contentType: "application/json",
    messageId: (payload as any).messageId ?? crypto.randomUUID(),
  });
  await ch.close();
  await conn.close();
}

export async function subscribe(exchange: string, onMessage: (data: any) => Promise<void>) {
  const conn = await amqplib.connect(URL);
  const ch = await conn.createChannel();
  await ch.assertExchange(exchange, "fanout", { durable: true });
  const { queue } = await ch.assertQueue(""); // auto‑delete, non‑durable
  await ch.bindQueue(queue, exchange, "");
  ch.consume(queue, async msg => {
    if (!msg) return;
    await onMessage(JSON.parse(msg.content.toString()));
    ch.ack(msg);
  });
}
```

---

## 4. 発行コード — `micropost-service/src/events/publish.ts`

```ts
import crypto from "crypto";
import { publish } from "@org/shared/mq";
import { Micropost } from "../models/micropost";

export async function emitMicropostCreated(post: Micropost) {
  await publish("micropost.created.v1", {
    messageId: crypto.randomUUID(),
    id: post.id,
    userId: post.userId,
    content: post.content,
    createdAt: post.createdAt,
  });
}
```

* Micropost 作成 API 内で DB 保存後に呼び出すのみ。

---

## 5. 受信コード — `category-service` 例

```ts
import { subscribe } from "@org/shared/mq";
import { prisma } from "../utils/prisma";

export async function startMicropostListener() {
  await subscribe("micropost.created.v1", async (evt) => {
    // 1️⃣ Micropost をローカルに複写
    await prisma.micropost.create({
      data: {
        id: evt.id,
        content: evt.content,
        authorId: evt.userId,
        createdAt: new Date(evt.createdAt),
      },
      skipDuplicates: true, // PRIMARY KEY(id) ならワンライナーで重複回避
    });

    // 2️⃣ ハッシュタグを Category テーブルに upsert
    const tags = (evt.content.match(/#\w+/g) || []).map((t: string) => t.slice(1));
    if (tags.length === 0) return; // タグ無しなら終了

    // bulk upsert
    await prisma.category.createMany({
      data: tags.map((tag: string) => ({ name: tag })),
      skipDuplicates: true,
    });

    // 3️⃣ Micropost と Category の関連付け (中間テーブル)
    await prisma.micropost.update({
      where: { id: evt.id },
      data: {
        categories: {
          connect: tags.map((tag: string) => ({ name: tag })),
        },
      },
    });
  });
}
```

* この軽量ロジックで **Micropost 複写 → カテゴリ追加/更新 → 関連付け** が完了します。

---

## 6. 最小限の信頼性オプション 最小限の信頼性オプション

| リスク           | シンプル対処法                                                      |
| ------------- | ------------------------------------------------------------ |
| ランタイム中のサービス停止 | RabbitMQ にメッセージが残る（Queue は自動生成だが durable exchange を使うため）     |
| 重複配信          | `PRIMARY KEY (id)` で INSERT 時に無視 or `ON CONFLICT DO NOTHING` |
| メッセージロス       | プロデューサ再送 or ログ確認で手動復旧（シンプル構成ゆえ許容）                            |

---

## 7. 動作確認スクリプト（Dev 用）

```bash
# ターミナル1: RabbitMQ
docker run -p 5672:5672 -p 15672:15672 rabbitmq:3.13-management

# ターミナル2: リスナー起動（Category）
node dist/category-service/index.js

# ターミナル3: 投稿イベント発行
curl -X POST localhost:3000/microposts -d '{"content":"Hello #tech"}' -H "Content-Type: application/json"
```

---
