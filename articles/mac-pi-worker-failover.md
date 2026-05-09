---
title: "自宅Macで動く個人サイトを友人宅のRaspberry Piにフェイルオーバーする"
emoji: "🛟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cloudflare", "raspberrypi", "tailscale", "network", "nodejs"]
published: true
---

## はじめに

自分で取得した `inazu.me` というドメインで，個人サイトを公開しています．

サイト本体はVPSやホスティングサービスではなく，自宅に置いているMac mini上で動かしています．Mac側ではNode.js + Expressを起動し，Cloudflare Tunnelを使ってインターネットへ公開しています．

同じMacでは `chat.inazu.me` も動かしています．これはMac上のローカルLLMと対話するためのWebアプリです．Cloudflare Turnstileで人間確認をしたあと，ExpressがMac上のOllamaへリクエストを流し，返ってきた応答をブラウザへstreamingします．

この構成では，Macが単なる静的サイトのoriginではなく，ローカルLLMの実行環境にもなっています．そのため，サイトの可用性を考えるときに「HTMLだけどこかへ置けばよい」という話ではなく，`inazu.me` と `chat.inazu.me` を含む全体がMacに依存している，という前提があります．

この構成は手軽です．自宅ルーターのポートを開けずに済み，Mac側の `localhost:3000` をそのまま公開できます．一方で，originはあくまで自宅のMacです．Macを再起動すれば落ちますし，スリープやネットワーク不調の影響も受けます．

個人サイトなので，商用サービスのような高可用性は必要ありません．ただ，Macが応答できないときに完全に何も返らないよりは，最低限のページくらいは返したい．そこで，Raspberry PiとCloudflare Workerを使って，まずは `inazu.me` の簡易的なフェイルオーバー構成を作りました．

最終的な構成は次の通りです．

```text
User
  -> https://inazu.me/*
  -> Cloudflare Worker
      -> Mac origin tunnel
      -> Raspberry Pi origin tunnel
      -> Worker built-in fallback HTML
```

通常時は自宅Macから本番サイトを返します．Macが落ちた場合はRaspberry Piの簡易ページへ切り替えます．Piにも届かない場合は，Cloudflare Workerが内蔵HTMLを返します．

## 要件

今回の目的は，厳密な高可用性構成を作ることではありません．個人サイトとして，次の条件を満たすことを目標にしました．

- 通常時は自宅Macをprimary originにする
- Macが応答しないときだけRaspberry Piへfallbackする
- Piにも届かない場合はCloudflare Workerだけで最小ページを返す
- 友人宅ルーターのポート開放は不要にする
- できるだけ無料枠の範囲で構成する
- `chat.inazu.me` は今回の対象から外す

`chat.inazu.me` はローカルLLMと対話するWebアプリなので，Raspberry Pi Zero 2 Wに同じものを載せるのは現実的ではありません．モデルを動かす計算資源が足りませんし，仮に固定文だけを返すfallbackを作るとしても，Turnstile，cookie session，Ollama，SSE streamingの扱いを別に考える必要があります．

そのため，今回はサイト全体の入口にフェイルオーバーの仕組みを置きつつ，実際にfallback対象にするのは静的サイトに近い `inazu.me` だけにしました．`chat.inazu.me` はあとで別設計として扱います．

## もとの構成

変更前は，Cloudflare TunnelでMacのローカル3000番を公開していました．

```text
User
  -> Cloudflare
  -> Cloudflare Tunnel
  -> Mac mini
  -> http://localhost:3000
```

Express側ではHost名を見て，返す内容を切り替えています．

- `inazu.me`: ポートフォリオサイト
- `chat.inazu.me`: ローカルLLMチャット
- `/mas`: ローカルMAS実験ページ

今回の変更では，`inazu.me` 宛のリクエストをCloudflare Workerに通し，Workerがoriginを選ぶ構成に変えました．

## 採用しなかった案

### Cloudflare Load Balancing

もっとも素直なのはCloudflare Load Balancingです．primary poolとfallback poolを作り，health checkで落ちたoriginを外す構成にできます．

ただ，Cloudflare Load Balancingは有料です．個人サイトの実験としては，まず無料で構成したかったため，今回は採用しませんでした．

本番業務で同じことをするなら，Load Balancingを使う方が自然です．今回の構成は，あくまで個人運用向けの簡易版です．

### Cloudflare Tunnelのreplica

Cloudflare Tunnelでは，同じtunnelのconnectorを複数台で動かせます．MacとPiで同じtunnelを動かせば，一見フェイルオーバーに使えそうです．

しかし，Tunnel replicaは「Macを優先し，Macが落ちたときだけPiへ流す」ための仕組みではありません．Cloudflare側は近いconnectorへ流すため，Macが生きていてもPiに流れる可能性があります．

今回必要だったのはactive/passiveの構成です．通常時はMac，MacがだめなときだけPi．この優先順位を明示したかったため，Tunnel replicaは見送りました．

## 採用した構成

Cloudflare Workerをリバースプロキシとして使い，Worker内でoriginを選びます．

公開URLは変えません．ユーザーは常に `https://inazu.me/` にアクセスします．裏側にorigin用のサブドメインを用意しました．

```text
inazu.me                 -> Workerが受ける公開入口
mac-site-origin.inazu.me -> Mac tunnel -> http://localhost:3000
pi-site-origin.inazu.me  -> Pi tunnel  -> http://localhost:8080
```

実際のURLは次の通りです．

- 公開入口: [https://inazu.me](https://inazu.me)
- Mac origin: [https://mac-site-origin.inazu.me](https://mac-site-origin.inazu.me)
- Pi origin: [https://pi-site-origin.inazu.me](https://pi-site-origin.inazu.me)

これはリダイレクトではありません．ブラウザのアドレスバーは `inazu.me` のままです．Workerが内部で `mac-site-origin.inazu.me` や `pi-site-origin.inazu.me` にfetchし，そのレスポンスを `inazu.me` のレスポンスとして返します．

なお，`mac-site-origin.inazu.me` と `pi-site-origin.inazu.me` は隠していません．直接叩けばそれぞれのoriginにアクセスできます．

ただし，直接叩いた場合はfallbackしません．たとえばPiの電源が切れている状態で `https://pi-site-origin.inazu.me` にアクセスすると，そのoriginが落ちているだけなのでBad Gatewayになります．MacやWorker内蔵HTMLへ切り替わるのは，あくまで公開入口の `https://inazu.me` をWorker経由で叩いた場合だけです．

本来はWorkerからだけ通すために，Workerが付ける秘密ヘッダーをorigin側で検証する，という作りにもできます．ただ，今回は個人サイトの実験なので，origin用サブドメインを直叩きされても問題ないものとして割り切りました．

どのoriginから返ったかを確認するため，レスポンスには `x-inazu-origin` ヘッダーを付けました．

```text
x-inazu-origin: mac
x-inazu-origin: pi
x-inazu-origin: worker-fallback
```

## Raspberry Pi側

Raspberry Piには，fallback用の小さなNode.jsサーバーを置きました．

Pi側の役割は次の2つだけです．

- `/` で簡易HTMLを返す
- `/healthz/site` で `ok` を返す

`/healthz/site` は生存確認用のendpointです．人間が見るページではなく，originが最低限応答できるかを確認するために用意しています．

```js
if (req.url === '/healthz/site') {
  res.writeHead(200, {
    'content-type': 'text/plain; charset=utf-8',
    'cache-control': 'no-store'
  });
  res.end('ok\n');
  return;
}
```

fallback用HTMLは，外部CSS，外部フォント，画像，JavaScriptを使わない1枚HTMLにしました．非常用ページなので，依存するリクエストを増やさないためです．

Piのプロセスはsystemdで管理しています．

```text
inazu-fallback.service
  -> /opt/inazu-fallback/server.js
  -> http://127.0.0.1:8080
```

Cloudflare Tunnelのpublic hostnameは次のように設定しました．

```text
pi-site-origin.inazu.me -> http://localhost:8080
```

## Tailscaleの位置づけ

PiにはTailscaleも入れています．ただし，公開アクセスの経路には使っていません．

Tailscaleの役割は，設置後にPiへSSHするための管理経路です．

```text
公開アクセス:
User -> Cloudflare Worker -> cloudflared on Mac/Pi -> local server

管理アクセス:
自分の端末 -> Tailscale -> PiへSSH
```

公開経路をMac経由のTailscaleにしてしまうと，Macが死んだときにPiへの経路も死にます．そのため，PiはPi自身でCloudflare Tunnelを張ります．Tailscaleはあくまで管理用です．

## Workerの実装

Workerのコードは，実際にはもう少しあります．ただし，中心は次の4つです．

- 同じpathでMac/Piのorigin URLを組み立てる
- originへのfetchにtimeoutを付ける
- `5xx` またはnetwork errorのときだけfallbackする
- どのoriginから返したかを `x-inazu-origin` で付ける

まず，originとtimeoutを定義します．

```js
const DEFAULT_MAC_ORIGIN = 'https://mac-site-origin.inazu.me';
const DEFAULT_PI_ORIGIN = 'https://pi-site-origin.inazu.me';
const DEFAULT_ORIGIN_TIMEOUT_MS = 1500;
```

リクエストのpathとqueryはそのままoriginへ渡します．たとえば `https://inazu.me/portfolio.css` に来たリクエストは，Mac originでは `https://mac-site-origin.inazu.me/portfolio.css` として取りに行きます．

```js
function buildOriginUrl(origin, requestUrl) {
  const base = origin.endsWith('/') ? origin : `${origin}/`;
  const url = new URL(requestUrl.pathname.replace(/^\//, ''), base);
  url.search = requestUrl.search;
  return url;
}
```

originへのfetchにはtimeoutを付けています．このtimeoutを超えたら，そのoriginは使えないものとして次へ進みます．

```js
async function fetchOrigin({ request, originUrl, timeoutMs }) {
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const headers = new Headers(request.headers);
    headers.set('x-inazu-worker', 'site-failover');

    const response = await fetch(originUrl, {
      method: request.method,
      headers,
      redirect: 'manual',
      signal: controller.signal
    });

    return { response, error: null };
  } catch (error) {
    return { response: null, error };
  } finally {
    clearTimeout(timeout);
  }
}
```

この `fetchOrigin()` を使って，まずMac originへ投げます．

```js
const macResult = await fetchOrigin({
  request,
  originUrl: buildOriginUrl(env.MAC_ORIGIN || DEFAULT_MAC_ORIGIN, requestUrl),
  timeoutMs
});

if (isUsableOriginResponse(macResult.response)) {
  return withOriginHeader(macResult.response, 'mac');
}
```

ここで `isUsableOriginResponse()` はかなり単純です．`500` 番台だけを失敗扱いにしています．

```js
function isUsableOriginResponse(response) {
  return response && response.status < 500;
}
```

そのため，`404` や `403` はfallback対象にしていません．存在しないpathを叩いたときに，originの404をPiのHTMLで上書きすると挙動が分かりにくくなるためです．fallback対象はnetwork error，timeout，5xxに絞っています．

Macがtimeoutするか，5xxを返した場合はPi originへfallbackします．

```js
const piResult = await fetchOrigin({
  request,
  originUrl: buildOriginUrl(env.PI_ORIGIN || DEFAULT_PI_ORIGIN, requestUrl),
  timeoutMs
});

if (isUsableOriginResponse(piResult.response)) {
  return withOriginHeader(piResult.response, 'pi');
}
```

Piも使えない場合は，Worker内蔵HTMLを返します．

```js
return builtInFallback(request, 200, 'worker-fallback');
```

最後に，返すレスポンスには確認用のヘッダーを付けます．

```js
function withOriginHeader(response, origin) {
  const headers = new Headers(response.headers);
  headers.set('x-inazu-origin', origin);

  return new Response(response.body, {
    status: response.status,
    statusText: response.statusText,
    headers
  });
}
```

このヘッダーのおかげで，ブラウザで見た目が同じでも，実際にはMacから返っているのか，Piにfallbackしているのかを確認できます．

## 実装時に詰まった点

### curlがbot blockに引っかかった

Mac側のExpressには，簡単なbot blockを入れています．その中に `curl` を弾く条件がありました．

```js
/curl/i
```

そのため，`curl https://mac-site-origin.inazu.me/` で確認すると403になります．ブラウザでは表示できるので，最初はorigin側のhost判定を疑いました．

確認時はUser-Agentを付ければ通ります．

```bash
curl -I -A 'Mozilla/5.0' https://mac-site-origin.inazu.me/
```

### origin用hostnameをExpress側に追加した

Express側では，Host名を見てpublic portfolioとして扱うかを判定しています．

もともとは次の2つだけでした．

```js
const PUBLIC_PORTFOLIO_HOSTS = new Set(['inazu.me', 'www.inazu.me']);
```

Workerは裏側で `mac-site-origin.inazu.me` を叩くため，このhostnameもpublic portfolioとして扱う必要があります．

```js
const PUBLIC_PORTFOLIO_HOSTS = new Set([
  'inazu.me',
  'www.inazu.me',
  'mac-site-origin.inazu.me'
]);
```

### Raspberry Pi Zero 2 Wのネットワーク

Raspberry Pi Zero 2 Wには有線LANポートがありません．また，Wi-Fiは2.4GHz前提で考える必要があります．5GHz専用SSIDやWPA3-onlyのネットワークでは接続できません．

設置先では，次のどちらかにする予定です．

- 2.4GHz/WPA2のWi-Fiに接続する
- microUSB OTG adapter + USB Ethernet adapterで有線化する

家庭用ルーターは基本的にPoE非対応と考えた方がよいです．PoEまでやる場合は，PoE injectorやPoE splitterが別途必要になります．今回は通常電源で運用します．

## 動作確認

通常時はMacから返ります．

```bash
curl -I -A 'Mozilla/5.0' https://inazu.me/
```

```text
x-inazu-origin: mac
```

Mac側のpm2プロセスを止めると，Piへfallbackします．

```bash
pm2 stop inazu-chat
```

```text
x-inazu-origin: pi
```

さらにPi側のWebプロセスも止めると，Worker内蔵HTMLになります．

```text
x-inazu-origin: worker-fallback
```

確認できた挙動は次の通りです．

```text
Mac正常:
inazu.me -> Worker -> Mac

Mac停止:
inazu.me -> Worker -> Pi

Mac停止 + Pi Web停止:
inazu.me -> Worker内蔵HTML
```

## 残っている課題

現在のWorkerは，Macが落ちている間も毎回Mac originへfetchします．Macがtimeoutするまで待ってからPiへfallbackするため，Mac停止中はリクエストごとに最大 `ORIGIN_TIMEOUT_MS` の遅延が入ります．

今は `ORIGIN_TIMEOUT_MS` を1500msにしています．

次に入れたいのはcircuit breakerです．Macへの接続に失敗したら，たとえば30秒間はMacをdown扱いにし，その間はPiへ直接流す．一定時間後にMacのhealth checkを試し，復帰していればMacへ戻す．

```text
1回目:
Macへfetch -> timeout -> Piへfallback

次の30秒:
Macを試さずPiへ直行

一定時間後:
Mac health checkに成功したらMacへ戻す
```

これを入れると，Mac停止中の体感遅延を抑えられます．

## chat.inazu.meについて

`chat.inazu.me` は今回の対象外です．

理由は，通常のHTMLページより状態が多いからです．

- Turnstile verification
- cookie session
- `express-session` が各ホストのメモリ内にある
- `/api/chat` がSSE streaming
- Ollamaは遅いだけで死んでいない場合がある
- 会話中にMacが死んでも，途中からPiへ引き継ぐことはできない

やるなら，Mac正常時はOllamaへ投げ，Mac down時は固定文のSSEを返す形になりそうです．ただし，これは `inazu.me` の経路制御が安定してから考えます．

## まとめ

Cloudflare Workerを使って，自宅Macで動く個人サイトに簡易フェイルオーバーを追加しました．

構成としては，Cloudflare Load Balancerの代替ではありません．health checkやoriginの状態管理も，現時点ではまだ簡素です．

それでも，個人サイトで次のような3段構成を作るには十分でした．

```text
User
  -> Cloudflare Worker
      -> home Mac
      -> Raspberry Pi
      -> Worker built-in HTML
```

自宅サーバーを公開していると，originが自分の生活にかなり近い場所にあります．その不安定さも含めておもしろいのですが，最低限の逃げ道を用意しておくと安心できます．今回の構成は，そのための小さな仕組みとしてちょうどよいものでした．
