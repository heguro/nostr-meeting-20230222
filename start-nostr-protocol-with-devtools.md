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

<https://github.com/heguro/nostr-meeting-20230222>

PCでご覧のかたは、是非一緒にDevToolsを開いて動かしてみてください！

※YouTubeのページはコンソールのログが流れやすいので、\
新しいタブで「<about:blank>」というURLを開いてからコンソールを開いて\
作業することをオススメします！

---
<!-- footer: ソースコードは `https://github.com/heguro/nostr-meeting-20230222` から -->

## Nostr プロトコルでの開発がいかに簡単か

始めるのは非常に簡単

なぜか

- すべてがWebSocketで動いてる！！！
  - （WebSocket: ブラウザなどで使えるリアルタイム双方向通信の規格）
- Twitterや各種SNSで定番の「APIキーを取得する」作業すら不要
- 仕様が一カ所にまとまっていて、単純明快
  - <https://github.com/nostr-protocol/nips>
  - 日本語 → <https://scrapbox.io/nostr/NIP>

---

## 例: ある人の最新の投稿を 10 件取得する

1. WebSocketでリレーサーバーに接続
2. 以下のメッセージを送る

```json
[ "REQ", "購読ID", { "pubkey": "その人の公開鍵", "kinds": [1], "limit": 10 } ]
```

3. 投稿内容を含んだメッセージが返ってくる！！

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

ws.addEventListener("open", () => { // 接続オープンできたら
  ws.send(JSON.stringify(  // 取得条件を指定してリクエスト
    [ "REQ", subscriptionId, { authors: [pubkey], kinds: [1], limit: 10 } ]
  ));
});
ws.addEventListener("message", (event) => { // メッセージが来たら
  const message = JSON.parse(event.data);
  switch (message[0]) {
    case "EVENT":  // イベント
      if (message[1] === subscriptionId) {
        const event = message[2];
        console.log(new Date(event.created_at * 1000), event.content);
      }
      break;
    case "EOSE":  // End of Stored Events. 投稿を全部返したよ
      if (message[1] === subscriptionId) ws.send(JSON.stringify(["CLOSE", subscriptionId]));
      ws.close();
      break;
    case "NOTICE": console.log("error:", message[1]); break; // エラーなど
  }
})
```

---

## Nostr 特有の概念: リレーとイベント

- Nostrでの行動はすべて「イベント」で表現される
  - 投稿、リアクション（ふぁぼ・いいね）、フォロー、プロフィール変更…
- リレーサーバーは、基本的には送られてきたイベントを（署名の検証をしつつ）
  保存し、クライアントに配信するだけ
- クライアントは複数のリレーサーバーに対して、ユーザーの行動に応じたイベントを秘密鍵で署名し配信する

<!--
先ほど「投稿内容が返る」と言いましたが、実際は「イベント」というものが返ってきてます。
Nostrでは投稿、リアクション、またプロフィール変更などがすべて「イベント」で表現されます。
-->

---

イベントの例

```json
{
  "id": "<64文字のhex>",      // 作成時刻・投稿内容から生成される
  "pubkey": "93ab9382fa66c807cd4bb702cf3be9e52f42ff9629db84e5a97c7b3bd336a4ac",
  "created_at": 1676227868,  // イベント作成時刻（＝投稿時刻）
  "kind": 1,                 // 投稿はkind(種類):1
  "tags": [],                // リプライ先などを指定
  "content": "本文\nだよ〜",　 // 改行は`\n`
  "sig": "<128文字のhex>",   // 作成時刻・投稿内容・ID・秘密鍵から生成される
}
```

（投稿イベントにはプロフィール情報が含まれてないので、別にREQしよう！）

---

## ここで問題

既存の投稿だけでなく、リアルタイムに新しい投稿を取得したいときはどうする？

---

再掲

```typescript
pubkey = "93ab9382fa66c807cd4bb702cf3be9e52f42ff9629db84e5a97c7b3bd336a4ac"; // @heguroの公開鍵(hex)
subscriptionId = Math.random().toString().slice(2); // 購読IDはランダム

ws = new WebSocket("wss://nostrja-kari.heguro.com"); // 接続して

ws.addEventListener("open", () => { // 接続オープンできたら
  ws.send(JSON.stringify(  // 取得条件を指定してリクエスト
    [ "REQ", subscriptionId, { authors: [pubkey], kinds: [1], limit: 10 } ]
  ));
});
ws.addEventListener("message", (event) => { // メッセージが来たら
  const message = JSON.parse(event.data);
  switch (message[0]) {
    case "EVENT":  // イベント
      if (message[1] === subscriptionId) {
        const event = message[2];
        console.log(new Date(event.created_at * 1000), event.content);
      }
      break;
    case "EOSE":  // End of Stored Events. 投稿を全部返したよ
      if (message[1] === subscriptionId) ws.send(JSON.stringify(["CLOSE", subscriptionId]));
      ws.close();
      break;
    case "NOTICE": console.log("error:", message[1]); break; // エラーなど
  }
})
```

---

正解は・・・

```typescript
pubkey = "93ab9382fa66c807cd4bb702cf3be9e52f42ff9629db84e5a97c7b3bd336a4ac"; // @heguroの公開鍵(hex)
subscriptionId = Math.random().toString().slice(2); // 購読IDはランダム

ws = new WebSocket("wss://nostrja-kari.heguro.com"); // 接続して

ws.addEventListener("open", () => { // 接続オープンできたら
  ws.send(JSON.stringify(  // 取得条件を指定してリクエスト
    [ "REQ", subscriptionId, { authors: [pubkey], kinds: [1], limit: 10 } ]
  ));
});
ws.addEventListener("message", (event) => { // メッセージが来たら
  const message = JSON.parse(event.data);
  switch (message[0]) {
    case "EVENT":  // イベント
      if (message[1] === subscriptionId) {
        const event = message[2];
        console.log(new Date(event.created_at * 1000), event.content);
      }
      break;
    case "EOSE":  /////  ↓ この2行をコメントアウト ↓   /////
      // if (message[1] === subscriptionId) ws.send(JSON.stringify(["CLOSE", subscriptionId]));
      // ws.close();    
      break;
    case "NOTICE": console.log("error:", message[1]); break; // エラーなど
  }
})
```

---

## これで実行すると・・・

---

## 実際は・・・

- 署名検証の作業が必要
  - 投稿されてからリレーを経由して受信するまでにイベントを改ざんされていないか
- ライブラリーを使うのが手軽
- JavaScriptなら`nostr-tools`
  - 取得時自動で署名検証してくれる（手動でもできる）

---

`nostr-tools`ライブラリを使って同等の記述

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

投稿してみる

```typescript
NostrTools = await import("https://esm.sh/nostr-tools@1.4.1"); // 時間かかる

pubkey = "93ab9382fa66c807cd4bb702cf3be9e52f42ff9629db84e5a97c7b3bd336a4ac";

relay = NostrTools.relayInit("wss://nostrja-kari.heguro.com");
await relay.connect();  // 接続  (本来はtry-catchで囲むべき)

event = {
  "pubkey": pubkey,
  "created_at": Math.floor(Date.now() / 1000),  // 現在時刻(秒)
  "kind": 1,                 // 投稿はkind(種類):1
  "tags": [],                // リプライ先があれば指定
  "content": "本文\nだよ〜", // 改行は`\n`
}

event.id = NostrTools.getEventHash(event); // IDを生成
event.sig = NostrTools.signEvent(event, "秘密鍵(hex)"); // sigを生成

pub = relay.publish(event);
pub.on('ok', () => {
  console.log('投稿完了');
});
pub.on('failed', (error) => {
  console.log('投稿失敗:', error);
});
```

---

## 実際にクライアントがやってること

- 複数リレーに同時に接続する
- リンク・画像埋め込みの表示
  - 画像をやりとりする機能はNostrにはないので、外部アップローダーを使う
- リプライ・引用投稿などがあれば元投稿を見にいって表示を置き換える

など・・・

***あなたもLet's Nostr!!!!!!!!!!!!!***

---

## 宣伝

NostrFlu
<https://heguro.github.io/nostr-following-list-util/>
