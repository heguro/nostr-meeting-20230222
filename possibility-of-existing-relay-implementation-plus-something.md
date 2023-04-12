---
marp: true
---
<!-- markdownlint-disable-file single-h1 no-duplicate-header -->
<!-- paginate: true -->

# 「既存のリレー実装 + なにか」の可能性

![width:60px](https://heguro.com/public/nostr/heguro-icon-20230206.jpg)
あすらも / heguro

- `npub1jw4e8qh6vmyq0n2tkupv7wlfu5h59luk98dcfedf03anh5ek5jkq936u57`
- <https://snort.social/p/heguro@heguro.com>
- つくったもの: NostrFlu, ？？（予定）

<!-- _footer: 2023/4/12「Nostr勉強会 #2」 <https://428lab.connpass.com/event/276333/> -->

---

## Nostr リレーの汎用性と可能性

- Nostrのリレーはシンプルで、普通に立てると独自性がほとんどない
- ActivityPubなどに比べて、リレーを立てるうまみが小さい？
  - → 小さなプログラムと組み合わせて可能性を広げてみない？

---

## Nostr リレーの汎用性と可能性

- リレー実装はミニマム
  - WebUIなどが存在しない
  - プログラムフレンドリー
- 無責任分散の仕様のおかげもあり雑にリレー立てて雑に落とせる
- 既存のリレー実装と組み合わせる小さなプログラムを書くことで、*無限の可能性*が生まれるかも

たとえば・・・

- aggregator relay
- Nostrの仕様にない手段での認証

---

## aggregator relay ?

- 複数のリレーへ代わりに接続してくれるリレー
- 機能例
  - 複数のリレーからの投稿を受信してくれたり
  - 投稿を送信したら複数のリレーに配信してくれたり
  - スパムなどを弾いてくれる機能があったり
- モバイル回線の通信量節約に

※ aggregate: 〜を集める、まとめる。　フィルターリレー、集約リレー？

稼働中（どちらも要課金）

- **filter.nostr.wine** - <https://nostr-wine.github.io/filter-relay/>
- **nostrich.land** - <https://nostrich.land>

---

## 自前の aggregator relay をつくってみよう

- nostream（普通のリレー）
- 自前のプログラム
  - Node.js
  - nostr-tools

~~つくってみた~~ つくっている（多分まだ動作しません）
- <https://github.com/heguro/nostr-aggregator-relay-test>

---

## つくりかた

これだけ！

- 既存のリレーに接続し、全投稿を受信
  - filter: `{kinds: [1]}`
- 受信した投稿を、ローカルに立ち上げたリレーへそのまま送信
  - 「日本語の投稿である」のような送信条件を付けてもよい

独自仕様として

- 一度でも日本語判定されたアカウント（公開鍵）は、以降全投稿を送信する

---

## 利点: データベース構築が不要

- ローカルのNostrリレーをそのまま簡易データベースとしても使える！
- NIP-33の「（パラメータつき）**上書き可能イベント**」を活用
- 名前をつけてデータを送信すると過去のイベントを上書きしてくれる
  - データの内容は（テキストであれば）自由
  - key-valueデータベースとして使える

今回は、「日本語の投稿である」と判定したことがある公開鍵のリスト保存に使用

---

## 「既存リレー実装 + なにか」 の可能性

ほかの可能性（検討中）

- リレーの手前にシンプルなプロキシをつくってみる
  - フィルターにより別のリレーに送信するとか

など・・・
