---
title: "cloudflare workersのデプロイでつまずいた"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cloudflare", "bun", "hono"]
published: true 
---

# はじめに

tailwindcssをいい感じに使えるようになりたく、bun + hono + tailwindcss の環境構築をしていたところ`bun run deploy`でエラーが発生して躓いてしまったので、その解決方法をまとめておきたいと思います。

# Uncaught Error: No such module "node:async_hooks". でデプロイが失敗する

エラーの全文は以下👇️

```
vite v5.3.3 building SSR bundle for production...
✓ 60 modules transformed.
rendering chunks (2)...GET  /
dist/index.html   0.23 kB
dist/_worker.js  37.86 kB
✓ built in 608ms
▲ [WARNING] Warning: Your working directory is a git repo and has uncommitted changes

  To silence this warning, pass in --commit-dirty=true


🌍  Uploading... (1/1)

✨ Success! Uploaded 0 files (1 already uploaded) (0.32 sec)

▲ [WARNING] The package "node:async_hooks" wasn't found on the file system but is built into node.

  Your Worker may throw errors at runtime unless you enable the "nodejs_compat" compatibility flag.
  Refer to https://developers.cloudflare.com/workers/runtime-apis/nodejs/ for more details. Imported
  from:
   - _worker.js


✨ Compiled Worker successfully
✨ Uploading Worker bundle
✨ Uploading _routes.json
🌎 Deploying...

✘ [ERROR] Deployment failed!

  Failed to publish your Function. Got error: Uncaught Error: No such module "node:async_hooks".
    imported from "bundledWorker-0.7226355726507692.mjs"
```

こちらは、エラーメッセージの中に以下の記述があります。

```
Your Worker may throw errors at runtime unless you enable the "nodejs_compat" compatibility flag.
Refer to https://developers.cloudflare.com/workers/runtime-apis/nodejs/ for more details.
```

`nodejs_compat`をcompatibility flag に追加しない限りランタイムエラーが発生する可能性があると記述されています。

そのため、このエラー自体はwrangler.tomlに以下の記述を追加すれば解決できます。

```toml
compatibility_flags = [ "nodejs_compat" ]
```

`nodejs_compatibility`自体の説明は以下になります。

```
Most Workers import one or more packages of JavaScript or TypeScript code from npm as dependencies in package.json. Many of these packages rely on APIs from the Node.js runtime, and will not work unless these APIs are present.

To ensure compatibility with a wider set of npm packages, and make it easier for you to run existing applications on Cloudflare Workers, the following APIs from the Node.js runtime are available directly as Workers runtime APIs, with no need to add polyfills to your own code:

- assert
- AsyncLocalStorage
- Buffer
- Crypto
- Diagnostics Channel
- EventEmitter
- path
- process
- Streams
- StringDecoder
- test
- util
```

英語が得意ではないので、正しいのか怪しいですが`nodejs_compat`を追加すると、Cloudflare Workersで上記のnodejs APIが使えるようになるとのことです。

# Failed to publish your Function. Got error: compatibility_flags cannot be specified without a compatibility_date でデプロイが失敗する

先ほど`compatibility_flags`を足した影響で、どの日時でのnodejsとの互換性を保つかを`wrangler.toml`に記入しないといけません。


[こちら](https://developers.cloudflare.com/workers/configuration/compatibility-dates/)を確認するとcompability_datesにすればいいかの指標になると思います。  
特に実装でnodejs のAPIを使っているわけではなかったので、現時点でのデフォルト値であった`2024-06-03`を設定し、再度デプロイを行ったところ、成功しました。

