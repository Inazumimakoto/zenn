---
title: "約49.7日で起きた“TCP時計の停止”：実体験と分析の記録"
emoji: "🕰️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["macos", "tcp", "network", "cloudflare"]
published: true
---

## 第1章 自宅Mac miniで起きた不可解な障害

### はじめに

2026年春、筆者は自宅のMac miniを使って簡単なポートフォリオサイトを公開していた。ドメイン「inazu.me」を取得したのは1月で、その際にCloudflare Tunnelの設定も行い、静的なindex.htmlを配置するだけのシンプルな構成で外部公開を始めた。自宅回線でも安全にサイトが公開でき、特にトラブルもなく運用を続けていた。

数か月後のある日、予想外のトラブルが起こる。出張先からサイトにアクセスすると突然つながらず、帰宅後に調べても原因がすぐには分からなかった。調査の末に再起動で復旧したものの、その背後にはmacOS固有の「49.7日問題」が潜んでいたことを後に知る。

### 沖縄出張中に発覚した異常

2026年3月、筆者は学会参加のため沖縄に滞在していた。滞在中にふと自分のサイトを開いてみるとページがまったく表示されない。サーバーの電源が落ちたのかと思い、家に泊まっていた友人に連絡してMac miniの状況を確認してもらった。ほどなくして届いた返信には、次のような内容が含まれていた。

![友人にMac miniの状態を確認してもらったときのやり取り](/images/mac-tcp-report/okinawa-mac-mini-check.jpg)
*友人にMac miniの状態を確認してもらったときのやり取り*

- Mac miniの電源は入っており、YouTubeでNew Jeansの動画が再生されている。
- 画面はロックされていないが、スクリーンタイムの履歴は0秒のままで更新されていない。
- スクリーンショットの右上には「プライベートリレーがオフになっています」という通知が出ていた。

つまり、Mac自体は動いているのにサイトだけが落ちている。ネットワークの問題なのか、サーバープロセスが止まっているのか判断できないまま、沖縄からできる対応は限られていた。

### 帰宅後の調査と再起動

帰宅したのは深夜だった。Macの画面を見ると、友人の報告通り動画再生は続いているが、ターミナルで`curl inazu.me`を叩いても応答がない。AIと相談しながらメモリやログを確認したものの、特に決定的な手がかりは見つからなかった。しばらく試行錯誤した後、最後の手段として再起動を試すことになった。再起動後は驚くほどあっさりとサイトが復旧し、その日は体調も優れなかったため調査を一旦終えることにした。

このとき、AIとの相談結果を踏まえて「一時的なメモリリークやリソース枯渇ではないか」と推測したものの、原因ははっきりしなかった。

---

## 第2章 49.7日バグへの気付きと検証

### 偶然の出会い：TechFeedの記事

3月のトラブルからおよそ1か月後の4月初旬、SNS上で「macOSのTCPスタックに49日問題がある」という[TechFeedの記事](https://techfeed.io/entries/69d8131c3a9f4a0e4f11b0aa)を偶然目にしました。そこには次のような内容が書かれていました。

macOS/XNUではTCP用タイムスタンプ`tcp_now`が32ビットの符号なしミリ秒カウンタとして実装されており、約49.7日、より正確には49日17時間2分47秒でオーバーフローする。さらに、`tcp_now`は値が前回より大きいときにだけ更新されるため、オーバーフローすると値が小さくなって更新されなくなり、内部時計が停止するというのです。

この記事を読んだ瞬間、3月に経験した不可解な障害がこのバグに起因しているのではないかと思い当たり、次にいつオーバーフローが起こるのか確認してみることにしました。

### 次の「オーバーフロー時刻」を予測する

自宅のMac miniで`uptime`を確認すると、前回再起動した日から49.7日目にあたるのは4月26日朝6時頃だと分かりました。これは2^32ミリ秒後のタイミングであり、理論上同じ現象が再び発生するはずです。そこで、あえて再起動せずにMacを連続稼働させ、問題が本当に起こるかどうかを観測することにしました。

### 監視体制の構築

再現実験では、長時間監視用のスクリプトを用意し、次の項目をTSVファイルに記録しました。

- TCP状態の数：`TIME_WAIT`、`SYN_SENT`、`ESTABLISHED`、`CLOSE_WAIT`などを数え、閉じたはずの接続が残り続けていないかを見ます。
- 一時ポート（ephemeral ports）の使用状況：使用中のポート数、空きポート数、`TIME_WAIT`が握っている一時ポート数を記録します。
- `TIME_WAIT`接続の詳細：対象となるTCP 4-tupleを別ファイルに記録し、同じ接続が何秒残り続けているかを追跡します。
- ICMP ping：`1.1.1.1`へのpingを記録し、IPレベルの到達性が残っているかを確認します。
- `curl`によるHTTP疎通試験：`127.0.0.1:3000`、`inazu.me`、`chat.inazu.me`、`www.gstatic.com/generate_204`にアクセスし、HTTPステータス、接続時間、合計時間、終了コード、接続先IPを記録します。名前解決の失敗も`curl`のエラーとして残ります。

長時間監視では、TCP状態やポート数、pingは10秒ごとに記録し、HTTP疎通試験は30秒ごとに実行しました。結果はTSVファイルに残し、あとからHTMLレポートとして可視化しました。

実験中、監視データを`~/Documents`配下に置いていたため、ネットワークが不調になった際にiCloudとの同期が止まり、ファイルが読めなくなるトラブルに遭遇しました。これは普通に痛恨のミスでした。重要なログやスクリプトは`/var/tmp/tcp49`のようなローカル専用ディレクトリに保存するべきでした。

### 実験結果の概要

#### 4月24日午後〜26日早朝

長時間監視は4月24日午後に開始しました。25日夜はTWICEのライブに友人と出かけており、そのまま夜更かししていたため、26日朝6時頃にオーバーフロー時刻を迎える頃には起きていませんでした。ログを確認すると、6時過ぎから`TIME_WAIT`の値が目に見えて増え始めていることが確認できました。

![HTMLレポート上のTIME_WAIT数と一時ポート数のグラフ](/images/mac-tcp-report/tcp49-report-timewait-ports.png)
*HTMLレポート上の`TIME_WAIT`数と一時ポート数のグラフ。点線が2^32ミリ秒のオーバーフロー時刻を示す。*

#### 4月26日午前

オーバーフロー直後もすぐに障害が発生するわけではなく、午前中はまだ`curl localhost:3000`や`inazu.me`へのアクセスが成功することもありました。しかし`TIME_WAIT`はじわじわと増え続け、一時ポートの残りが減少していく様子がデータから読み取れました。外部HTTPSへの`curl`では接続時間が伸びることもあり、新規通信が少しずつ不安定になり始めます。

#### 4月27日午前

翌27日の10時過ぎになると、問題は一気に表面化します。`curl`で`localhost:3000`も`inazu.me`も`www.gstatic.com/generate_204`も失敗し、ログには名前解決のタイムアウトやホスト解決失敗も記録されるようになりました。`TIME_WAIT`の数は2万前後まで増え、一時ポートの空きも大きく減り、新しいTCP接続が軒並み失敗しました。

この頃には、普段使っているアプリにも影響が出ていました。Codexは再接続を繰り返した末に応答が切れ、Teamsもつながらず、Xcodeのビルドも外部通信を必要とするところで進まなくなりました。新しいTCP接続を張れなくなったMac miniは、外の世界から少しずつ切り離されていきます。

![Codexが再接続に失敗している画面](/images/mac-tcp-report/codex-reconnecting.png)
*Codexが再接続を繰り返した後、ストリーム切断で止まった画面*

この実験は、ずっとCodexと相談しながら進めていました。ログの見方も、次に何を確認するかも、ほとんど会話しながら決めていたので、そのCodexが急に切れた瞬間はかなり怖かったです。Teamsもサインアウトされ、Xcodeもビルドエラーになり、接続がひとつずつ落ちていくのを目の前で見ていると、もう今すぐ再起動したいという気持ちでいっぱいになりました。

さらに、プライベートリレーがオフになったという通知も届きました。これは沖縄出張中に友人から送られてきたスクリーンショットにも写っていた通知で、この時点でようやく「あの通知も同じ現象の一部だったのかもしれない」とつながります。

さらに悪いことに、監視スクリプトやログを`~/Documents`配下に置いていたため、iCloud同期が詰まると手元のファイルまで雲の上に行ったまま戻ってこないような状態になりました。壊れていく様子を観測するための実験なのに、その観測道具まで触れなくなっていく。あまりにも恐ろしすぎて大号泣していました。目の前のMacのTCP接続よりも不安定だったのは、もはや自分の心の方でした。

TCP接続が新しく作れなくなり、作業用のアプリが次々と沈黙していく中で、その孤立したMac miniの画面に流れ続けていたのはYouTubeのNew Jeansの動画だけでした。そのときは「なぜYouTubeだけ生きているのか」と不思議に思っていましたが、後から考えるとYouTubeはQUIC、つまりUDPで通信していた可能性が高い。TCPの問題を追っている横で、UDPの動画だけが平然と流れ続けていたわけです。

### 再起動による復旧

すべての新規通信がほぼ不能になったところで再起動を実施しました。再起動後、`TIME_WAIT`は瞬時に0となり、一時ポートは完全に復元され、DNSやHTTPの疎通も通常に戻りました。今回の実験により、49.7日目のオーバーフロー直後から`TIME_WAIT`の掃除が止まり、最終的にポート枯渇と新規通信の失敗を引き起こすことが実証できました。

### この章のまとめ

TechFeedの記事に偶然出会ったことから「49.7日問題」の存在を知り、自分のMac miniで実際に再現実験を行ったところ、予想通り`TIME_WAIT`が大幅に増加し、新規TCP接続が失敗する現象を確認できました。次章では、なぜこのようなことが起きるのか、XNUの実装のどこに問題があるのかを詳しく見ていきます。

---

## 第3章 なぜ49.7日で壊れるのか — 技術的背景と教訓

### 32ビットの`tcp_now`とその限界

ここからは、実際に公開されているXNUのコードを見ていきます。参照したのはApple OSS Distributionsの[`apple-oss-distributions/xnu`](https://github.com/apple-oss-distributions/xnu)です。手元のmacOSに入っているバイナリと公開リポジトリの特定ブランチが完全に一致するとは限りませんが、今回の挙動を理解するにはかなり直接的な手がかりになります。

まず、TCP内部の時刻として使われる`tcp_now`は、`uint32_t`として宣言されています。

```c
extern uint32_t tcp_now;
```

出典: [`bsd/netinet/tcp_var.h`](https://github.com/apple-oss-distributions/xnu/blob/main/bsd/netinet/tcp_var.h#L1757)

TCPのタイムスタンプ粒度は1msです。

```c
#define TCP_RETRANSHZ   1000
```

出典: [`bsd/netinet/tcp_var.h`](https://github.com/apple-oss-distributions/xnu/blob/main/bsd/netinet/tcp_var.h#L91)

つまり、`tcp_now`は「32ビット符号なし整数で表現される、ミリ秒単位のTCP時計」として扱われます。

32ビットで表現できる最大値は4,294,967,295、つまり2^32 - 1です。その次の値に進む瞬間、カウンタは0へ戻ります。2^32ミリ秒は49日17時間2分47秒なので、これが約49.7日で起きるオーバーフローの正体です。

問題の中心は、`tcp_now`を更新する`calculate_tcp_clock()`です。Apple OSSの`main`ブランチでは、該当箇所は次のようになっています。

```c
current_tcp_now = (uint32_t)now.tv_sec * 1000 + now.tv_usec / TCP_RETRANSHZ_TO_USEC;

tmp = os_atomic_load(&tcp_now, relaxed);
if (tmp < current_tcp_now) {
    os_atomic_cmpxchg(&tcp_now, tmp, current_tcp_now, relaxed);
}
```

出典: [`bsd/netinet/tcp_subr.c`](https://github.com/apple-oss-distributions/xnu/blob/main/bsd/netinet/tcp_subr.c#L3745-L3749)

`current_tcp_now`は、起動後の秒数をミリ秒に変換した値です。ここで`uint32_t`に落としているため、約49.7日を超えると0付近に戻ります。

そのうえで、更新条件は`tmp < current_tcp_now`です。つまり、現在の値が前回より大きいときだけ`tcp_now`を進める、という単純な数値比較になっています。

オーバーフロー前ならこれは自然に動きます。たとえば`tcp_now`が1000で、新しい値が1010なら更新されます。しかし、オーバーフロー直後は話が変わります。`tcp_now`が4,294,967,000付近にいる状態で、現在値が10に戻ると、単純な数値比較では「10は4,294,967,000より小さい」と判定されます。その結果、`tcp_now`は更新されません。

その結果、`tcp_now`はオーバーフロー直前の大きな値のまま更新されなくなります。つまり、TCPスタックの中だけで使われている時計が止まったような状態になります。

なお、同じ公開リポジトリの`rel/xnu-12377`ブランチでは、この比較がラップアラウンドを考慮した形に変わっていました。

```c
if (TSTMP_LT(tmp, current_tcp_now)) {
    os_atomic_cmpxchg(&tcp_now, tmp, current_tcp_now, relaxed);
}
```

出典: [`bsd/netinet/tcp_subr.c` in `rel/xnu-12377`](https://github.com/apple-oss-distributions/xnu/blob/rel/xnu-12377/bsd/netinet/tcp_subr.c#L3903-L3907)

`TSTMP_LT`は32ビットタイムスタンプの比較用マクロです。

```c
#define TSTMP_LT(a, b)   ((int)((a)-(b)) < 0)
```

出典: [`bsd/netinet/tcp_seq.h`](https://github.com/apple-oss-distributions/xnu/blob/main/bsd/netinet/tcp_seq.h#L80-L84)

この形なら、単純な大小比較ではなく、32ビット値が循環する前提で「どちらが時刻として先か」を判定できます。少なくとも公開コード上では、`tmp < current_tcp_now`から`TSTMP_LT(tmp, current_tcp_now)`への変更が、この問題の修正方向そのものに見えます。

### TIME_WAIT掃除の停止とポート枯渇

TCPでは接続を閉じたあと、同じ通信の古いパケットと新しい通信が混ざらないようにするため、ソケットを一定時間`TIME_WAIT`状態で保持します。今回の観測では、macOS上の`TIME_WAIT`は通常30秒程度で掃除されていました。

コード上でも、MSLは15秒として定義されています。

```c
#define TCPTV_MSL       ( 15*TCP_RETRANSHZ)
```

出典: [`bsd/netinet/tcp_timer.h`](https://github.com/apple-oss-distributions/xnu/blob/main/bsd/netinet/tcp_timer.h#L175)

`TIME_WAIT`に入る箇所では、`2 * tcp_msl`が渡されています。

```c
add_to_time_wait(tp, 2 * tcp_msl);
```

出典: [`bsd/netinet/tcp_input.c`](https://github.com/apple-oss-distributions/xnu/blob/main/bsd/netinet/tcp_input.c#L5802-L5805)

`tcp_msl`は`TCPTV_MSL`なので、15秒の2倍で30秒です。自分の観測で`TIME_WAIT`が通常30秒程度で掃除されていたこととも一致します。

では、その30秒後という期限はどう持たれているのか。`add_to_time_wait_locked()`を見ると、期限は`tcp_now + delay`で計算され、`TCPT_2MSL`タイマーに保存されています。

```c
timer = tcp_now + delay;
tp->t_timer[TCPT_2MSL] = timer;
```

出典: [`bsd/netinet/tcp_timer.c`](https://github.com/apple-oss-distributions/xnu/blob/main/bsd/netinet/tcp_timer.c#L563-L571)

そして掃除側の`tcp_gc()`では、現在の`tcp_now`がその期限に到達したかどうかを見ています。

```c
if (tw_tp->t_state == TCPS_CLOSED ||
    TSTMP_GEQ(tcp_now, tw_tp->t_timer[TCPT_2MSL])) {
```

出典: [`bsd/netinet/tcp_timer.c`](https://github.com/apple-oss-distributions/xnu/blob/main/bsd/netinet/tcp_timer.c#L828-L842)

ここで重要なのは、`TIME_WAIT`掃除側の比較は`TSTMP_GEQ`を使っており、ラップアラウンドを考慮する形になっていることです。つまり、掃除側の比較そのものが単純に壊れているというより、比較に使う`tcp_now`が止まることが致命傷になります。

`tcp_now`が正常に0へ回って進み続けるなら、`tcp_now + delay`で作った期限にもいつか到達します。しかし、`tcp_now`がオーバーフロー直前の値で止まると、期限判定が進みません。新しいTCP接続を作って閉じるたびに`TIME_WAIT`のソケットは増えていく一方で、いつまで待っても解放されなくなります。

その結果、一時ポートが`TIME_WAIT`に占有され続けます。外向きのTCP接続を作るにはローカル側の一時ポートが必要なので、空きポートが減っていくと、新しいTCP接続を張ることがだんだん難しくなります。最終的には、`curl`によるHTTPアクセスだけでなく、名前解決を含むアプリ側の通信処理まで失敗するようになります。

今回の実験で見えた「4月26日の朝に`TIME_WAIT`が増え始め、4月27日の午前に新規通信がほぼ壊滅した」という流れは、この説明とよく一致します。オーバーフローした瞬間にすべてが落ちるのではなく、掃除されない`TIME_WAIT`が積み上がり、翌日にかけて一時ポートを食いつぶしていく、という壊れ方でした。

### 教訓と対策

今回の経験から得られた教訓を整理します。

1. **macOSを長期間運用するなら49.7日を意識する。** 49日を超えて連続稼働させる必要がある場合は、事前に再起動する計画を立てるか、`TIME_WAIT`や一時ポートの空き数を監視して兆候を早めに捉える必要があります。今回のように、オーバーフロー直後はまだ動いているように見えても、翌日にかけて致命的な状態へ進むことがあります。

2. **重要なログやスクリプトはiCloud同期下に置かない。** ネットワーク不調時にiCloud同期が詰まり、ファイルがdataless状態になると、レポート生成や復旧に必要なスクリプトまで実行できなくなります。今回これは本当に痛恨のミスでした。監視ツールや緊急用のコマンドは、`/var/tmp/tcp49`のようなローカル専用のディレクトリに置くべきでした。

3. **「既存の接続が動いているから大丈夫」とは判断しない。** YouTubeの動画が流れていたり、一部のアプリがまだ動いていたりしても、新しいTCP接続が作れるとは限りません。特にYouTubeはQUIC、つまりUDPで通信している可能性があり、TCPの状態を判断する材料としてはかなり紛らわしい存在でした。

4. **観測環境そのものを壊れにくくしておく。** 障害の原因を調べるにはログが必要ですが、そのログを読むための通信や同期が同じ障害に巻き込まれると、一気に身動きが取れなくなります。ネットワーク障害を観測するなら、ローカル保存、ローカル実行、ローカル表示で完結できる形にしておくべきでした。

この章では、観測結果とXNUの公開コードを照らし合わせながら、`tcp_now`の停止がどのように`TIME_WAIT`の滞留とポート枯渇につながるのかを整理しました。今回の現象は「49.7日で突然TCPが全部死ぬ」というより、「TCP内部の時計が止まり、掃除されない`TIME_WAIT`が積み上がり、翌日にかけて新規接続が詰んでいく」という壊れ方だったと考えるのが自然です。

---

## まとめ

今回の障害は、最初はただの「自宅Mac miniで公開していたサイトが突然つながらなくなった」という出来事でした。沖縄出張中に発覚し、帰宅後に再起動したら直ったため、その時点ではメモリリークや一時的なリソース枯渇のようなものだと思っていました。

しかし、その後に49.7日問題の記事を読み、自分のMac miniで再現実験をしたことで、見え方が大きく変わりました。問題は単にWebサーバーやCloudflare Tunnelが落ちたという話ではなく、macOSのTCPスタック内部で使われる`tcp_now`が約49.7日でオーバーフローし、`TIME_WAIT`の掃除が止まり、一時ポートが消費され続けるという現象でした。

実際のログでも、オーバーフロー時刻を境に`TIME_WAIT`が増え始め、翌日にかけて新しいTCP接続が作れなくなっていく様子を確認できました。Codex、Teams、Xcodeといった普段の作業環境が次々と沈黙していく一方で、YouTubeだけが流れ続けていたのは、後から考えるとQUIC/UDPだったからだろうというオチまでつきました。

XNUの公開コードを見ると、`tcp_now`は32ビットのミリ秒カウンタとして扱われており、問題の本質は「32ビット整数がオーバーフローすること」そのものではなく、それを単純な大小比較で更新していたことにあります。時刻が0へ戻ったあとも循環する値として扱えればよかったのに、単調に増え続ける値として扱ってしまったことで、TCP内部の時計が止まってしまったわけです。

この記事で一番伝えたかったのは、「再起動したら直った」で終わる障害の裏にも、ちゃんと観測すれば説明できる構造がある、ということです。Macをサーバー的に長期間動かすなら、49.7日という数字を頭の片隅に置いておく。ログや監視スクリプトはiCloud配下ではなくローカルに置く。pingや動画再生だけでネットワークが生きていると判断しない。このあたりは、今回かなり痛い形で学んだ教訓でした。

そして何より、TCP接続が次々と死んでいくMacの前で、頼みのCodexまで切断されると普通に怖いです。目の前のMacのTCP接続よりも自分の心の方が不安定になる前に、長期稼働Macには定期再起動か監視を入れておきましょう。

## 参考資料

- [We Found a Ticking Time Bomb in macOS TCP Networking - It Detonates After Exactly 49 Days - Photon Blog](https://photon.codes/blog/we-found-a-ticking-time-bomb-in-macos-tcp-networking)
- [すべてのMacに49.7日バグ—長期稼働でTCP接続不能になる脆弱性が発覚 - TechFeed](https://techfeed.io/entries/69d8131c3a9f4a0e4f11b0aa)
- [apple-oss-distributions/xnu - GitHub](https://github.com/apple-oss-distributions/xnu)
- [`calculate_tcp_clock()` in `bsd/netinet/tcp_subr.c`](https://github.com/apple-oss-distributions/xnu/blob/main/bsd/netinet/tcp_subr.c#L3745-L3749)
- [`calculate_tcp_clock()` in `rel/xnu-12377`](https://github.com/apple-oss-distributions/xnu/blob/rel/xnu-12377/bsd/netinet/tcp_subr.c#L3903-L3907)
- [`TIME_WAIT`の期限計算 in `bsd/netinet/tcp_timer.c`](https://github.com/apple-oss-distributions/xnu/blob/main/bsd/netinet/tcp_timer.c#L563-L571)
- [`TIME_WAIT`掃除条件 in `bsd/netinet/tcp_timer.c`](https://github.com/apple-oss-distributions/xnu/blob/main/bsd/netinet/tcp_timer.c#L828-L842)

長くなりましたが、ここまで読んでくれてありがとうございました。
