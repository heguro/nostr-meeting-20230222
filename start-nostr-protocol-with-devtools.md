---
marp: true
---
<!-- markdownlint-disable-file single-h1 -->
<!-- paginate: true -->

# DevToolsではじめる簡単Nostrプロトコル

あすらも / heguro

- `npub1jw4e8qh6vmyq0n2tkupv7wlfu5h59luk98dcfedf03anh5ek5jkq936u57`
- <https://iris.to/heguro@heguro.com>

**※WIP** 現在編集中で、全面的に差し替える可能性もあります

<!--
あすらも または heguro と申します。

今回は「DevToolsではじめる簡単Nostrプロトコル」ということで、ブラウザの開発ツールを使ってNostrプロトコルに触れていきたいと思います。
-->

---

## 今回の資料・ソースコード

<https://github.com/heguro/nostr-connpass-20230222>

PCでご覧のかたは、是非一緒にDevToolsを開いて動かしてみてください！

※YouTubeのページはコンソールのログが流れやすいので、\
新しいタブで「<about:blank>」というURLを開いてからコンソールを開いて\
作業することをオススメします！

---
<!-- footer: ソースコードは `https://github.com/heguro/nostr-connpass-20230222` から -->

## Nostr プロトコルでの開発がいかに簡単か

始めるのは非常に簡単

なぜか

- すべてがWebSocketで動いているので、WebSocketさえ分かれば作れる（基本は）
- Twitterや各種SNSで定番の「APIキーを取得する」作業すら不要
- 仕様が一カ所にまとまっていて、単純明快

---

## 例: ある人の最新の投稿を 10 件取得する

1. WebSocketでリレーサーバーに接続
2. 以下のメッセージを送る

```json
[ "REQ", "購読ID", { "pubkey": "その人の公開鍵", "kinds": [1], "limit": 10 } ]
```

3. 投稿内容を含んだメッセージが返ってくる

```json
[ "EVENT", "購読ID", { <投稿内容> } ]
```

```json
[ "EVENT", "購読ID", { <投稿内容> } ]
```

...

```json
[ "EOSE", "購読ID" ]
```

<!--
めっちゃ簡単
-->

---

WebSocketのみで書いてみる

```typescript
pubkey = "93ab9382fa66c807cd4bb702cf3be9e52f42ff9629db84e5a97c7b3bd336a4ac"; // @heguroの公開鍵(hex)
subscriptionId = Math.random().toString().slice(2); // 購読IDはランダム

ws = new WebSocket("wss://nostrja-kari.heguro.com"); // 接続して

ws.addEventListener("open", () => {
  ws.send(JSON.stringify(
    [ "REQ", subscriptionId, { authors: [pubkey], kinds: [1], limit: 10 } ]
  ));  // 取得条件を指定して取得開始
});
ws.addEventListener("message", (event) => {
  const message = JSON.parse(event.data);
  switch (message[0]) {
    case "EVENT":  // 取得したイベントを表示
      if (message[1] === subscriptionId) {
        const event = message[2];
        console.log(new Date(event.created_at * 1000), event.content);
      }
      break;
    case "EOSE":  // End of Stored Events
      if (message[1] === subscriptionId) ws.send(JSON.stringify(["CLOSE", subscriptionId]));
      ws.close();
      break;
    case "NOTICE": console.log("error:", message[1]); break;
  }
})
```

---

## Nostr 特有の概念: リレーとイベント

- リレーサーバーは、基本的には送られてきたイベントを（署名の検証をしつつ）
  保存し、クライアントに配信するだけ
- クライアントは複数のリレーサーバーに対して、投稿・リアクション（ふぁぼ/いいね）・プロフィール・登録リレー・フォローなどのイベントを秘密鍵で署名し配信する

イベントの例

```json
{
  "TODO": "TODO"
}
```

---

TODO: 署名について。nostr-toolsを使うと全部やってくれる

---

nostr-toolsライブラリを使って同等の記述

```typescript
// お試し用
NostrTools = await import("https://esm.sh/nostr-tools@1.4.1");

pubkey = "93ab9382fa66c807cd4bb702cf3be9e52f42ff9629db84e5a97c7b3bd336a4ac";

relay = NostrTools.relayInit("wss://nostrja-kari.heguro.com");
await relay.connect();  // 接続  (本来はtry-catchで囲むべき)

events = await relay.list({ // 取得条件を指定して取得開始
  authors: [pubkey], // イベント発行者
  kinds: [1],        // kind1は投稿(ノート)
  limit: 10,         // 過去10個分を取る
});

for (const event of events) {  // 取得したイベントを表示
  console.log(
    new Date(event.created_at * 1000), // created_at: 投稿日時 Unix time (秒)
    event.content                      // content: 本文
  )
}
```

---

## まとめ
