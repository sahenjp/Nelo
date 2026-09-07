<p align="center">
  <img src="./assets/nelo-icon.svg" alt="Nelo" width="128" height="128">
</p>

<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="./assets/nelo-wordmark-on-dark.svg">
    <source media="(prefers-color-scheme: light)" srcset="./assets/nelo-wordmark-on-light.svg">
    <img src="./assets/nelo-wordmark-on-light.svg" alt="Nelo — Every request owns its work." width="520">
  </picture>
</p>

<p align="center">
  <strong>リクエストに紐づく処理の寿命を管理する。</strong>
</p>

<p align="center">
  <img alt="実験的" src="https://img.shields.io/badge/status-experimental-6d7178">
  <img alt="Version 0.2.0" src="https://img.shields.io/badge/version-0.2.0-2864dc">
  <img alt="Strict TypeScript" src="https://img.shields.io/badge/TypeScript-strict-3178c6">
  <a href="./LICENSE"><img alt="Apache-2.0" src="https://img.shields.io/badge/license-Apache--2.0-5bc8ad"></a>
</p>

<p align="center">
  <a href="./README.md">English</a> · 日本語 · <a href="https://nelo.lattee.jp">Webサイト</a>
</p>

Neloは、各リクエストを、そのリクエストから始まった処理の所有者として扱うWeb Standardsベースのフレームワークです。ハンドラーが`Response`を返したあとも残るタスク、リソース、レスポンス本文、明示的に移譲されたDeferred Workの寿命を明示します。

> [!NOTE]
> **AI利用について:** Neloは、開発の大半でAIツールを利用しています。

> `Response`を返した時点で、リクエストに関係する処理がすべて終わるとは限りません。

## クイックスタート

```ts
import { Nelo } from "@lasder/nelo";
import { serve } from "@lasder/nelo/node";

const app = new Nelo();

app.get("/users/:id", async (context) => {
  const user = context.fork("load-user", (signal) =>
    fetchUser(context.params.id!, { signal })
  );
  return context.json(await user);
});

const server = serve(app, { port: 3000 });
await server.listen();
```

## ライフタイム

```text
リクエストのライフタイム
├── ハンドラースコープ
│   ├── middleware
│   ├── context.fork()
│   ├── context.forkScope() → 入れ子のタスク/リソースのライフタイム
│   ├── context.deadline()
│   └── context.use()
├── デリバリースコープ
│   ├── Response.body
│   ├── context.delivery.fork()
│   ├── context.delivery.forkScope()
│   └── context.delivery.use()
└── Deferredスコープ
    └── context.defer() → レスポンス後の明示的な所有権移譲
```

ハンドラースコープはハンドラー終了後に閉じます。デリバリースコープは、本文の完了、失敗、中断、接続切断まで残ります。Deferredスコープはレスポンス後も続ける処理を明示的に移譲し、対応adapterが追跡します。リソースは一度だけ、取得した順序と逆に解放されます。

## 主なAPI

| API | 役割 |
|---|---|
| `app.fetch(request)` | ルーティング、ミドルウェア、ハンドラー、配信を実行します。 |
| `context.fork(name, operation)` | リクエストが所有するタスクを開始します。 |
| `context.forkScope(name, operation)` | 複数のタスクやリソースを持つ入れ子のライフタイムを1つの所有タスクとして扱います。 |
| `context.signal` | リクエストの中断通知を処理へ渡します。 |
| `context.deadline(duration)` | リクエストより短い処理期限を持つSignalを作ります。 |
| `context.defer(name, operation)` | best-effort処理をレスポンス後へ明示的に移譲します。 |
| `context.use(name, acquire, cleanup?)` | ハンドラーが所有するリソースを取得・解放します。 |
| `context.delivery.fork(name, operation)` | レスポンス配信が所有する処理を開始します。 |
| `context.delivery.forkScope(...)` | 配信中に入れ子のライフタイムを所有します。 |
| `context.delivery.use(...)` | リソースや後片付けを配信終了まで保持します。 |

期限にはミリ秒の数値、または`750ms`、`2s`、`1m`、`1h`のような値を指定できます。親リクエストの中断理由を引き継ぎ、期限切れ時は型付きの`deadline`理由で中断し、ハンドラースコープ終了時に自動解放されます。

`forkScope()`は既存の`fork()`を置き換えるものではなく追加APIです。1つの処理が複数のタスク、リソース、deadlineを所有するときに使えます。子ライフタイムは親の中断を引き継ぎ、`handlerTree` / `deliveryTree`の診断にも残ります。従来の低レベル`forkChild()`は互換性のためdeprecated aliasとして残しています。

### Deferred work

`context.defer(name, operation)`は、レスポンス後も続けたいbest-effort処理を明示的に移譲します。処理には独立した`AbortSignal`が渡され、失敗は`NELO_DEFERRED_002`として観測できます。Deferred Workを提供しないruntimeでは`NELO_DEFERRED_001`で明示的に失敗し、所有者のないbackground Promiseを暗黙には作りません。

Node adapterはprocess内でDeferred Workを追跡します。`server.close()`は既存HTTP exchangeを先にdrainし、その後、残りのgrace期間でDeferred taskを待ちます。期限を超えた処理には`server_shutdown`を通知します。これは永続queue、retry、exactly-onceを保証する仕組みではありません。

## セキュリティ境界

Neloはproxyのforwarding headerを暗黙には信頼しません。Node adapterの`protocol: "https"`はFetch `Request`へ渡す公開URL schemeを指定するだけで、TLSを有効化する設定ではありません。HTTPSとして使う場合は、信頼できる外部server/proxyでTLSを終端してください。

request bodyはstreamingのまま扱われます。アップロード上限はendpointごとに要件が異なるため、アプリ側でbyte数・時間の上限を設定します。Internetへ公開する前に[SECURITY.md](./SECURITY.md)も確認してください。

## Lvau連携

[`examples/lvau-service`](./examples/lvau-service/mod.ts)は、Lvauの暗号化CLIをリクエスト所有の処理として実行します。クライアント切断時に子プロセスを終了し、一時平文を削除します。

設定方法と安全上の境界は[連携ガイド](./docs/integrations/lvau.md)を確認してください。

## ブランド素材

- [`assets/nelo-icon.svg`](./assets/nelo-icon.svg) — README、サイト、製品画面向けの基本アイコン。
- [`assets/favicon.svg`](./assets/favicon.svg) — faviconや小さい表示向け。

どちらも透明背景のフラットなSVGで、そのままWebサイトから参照できます。

## 開発

```sh
git clone https://github.com/sahenjp/Nelo.git
cd Nelo
npm ci
npm run format:check
npm run lint
npm run typecheck
npm test
npm run build
npm run check:package
npm run check:tarball
```

貢献方法とsecurity-sensitiveな変更の確認事項は[CONTRIBUTING.md](./CONTRIBUTING.md)を参照してください。

## ライセンス

[Apache License 2.0](./LICENSE)で公開しています。
