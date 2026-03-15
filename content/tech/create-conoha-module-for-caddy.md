---
# Common-Defined params
title: "ConoHaでCaddy"
date: "2025-08-12T01:02:33+09:00"
description: "ConoHa VPS Ver3.0 を利用して、CaddyからConoHa VPSへのDNS-01チャレンジを自動化できるようにしました。"
categories:
  - "tech"
  - "ssl/tls"
  - "caddy"
tags:
  - "tech"
  - "ssl/tls"
  - "caddy"

# Theme-Defined params
thumbnail: "img/caddy.svg" # Thumbnail image
lead: "caddy-dns moduleの開発と解説" # Lead text
authorbox: true # Enable authorbox for specific page
pager: true # Enable pager navigation (prev/next) for specific page
toc: true # Enable Table of Contents for specific page
mathjax: true # Enable MathJax for specific page
sidebar: "right" # Enable sidebar (on the right side) per page
widgets: # Enable sidebar widgets in given order per page
  - "search"
  - "recent"
  - "taglist"
---

## はじめに

この記事はワイルドカード証明書の取得を推奨するものではありません。
また、ワイルドカード証明書はむやみにつかって良いものでもありません。
ワイルドカードの不適切な運用は[ALPACA](https://www.nsa.gov/Press-Room/News-Highlights/Article/Article/2804293/avoid-dangers-of-wildcard-tls-certificates-the-alpaca-technique/)のような攻撃を可能にし、単一のサーバーの侵害によって、その証明書に紐づけられた他のすべてのサーバーが危険にさらされる可能性があります。
利用する際は、その利便性と危険性を理解し、上記のリスクが適切に軽減されたスコープと組み合わせで利用する必要があります。

## Caddy使ってますか?

[Caddy](https://caddyserver.com/)は、Golangで書かれた軽量かつ高機能なWebサーバーです。
Webサーバーと聞くと、NginxやApacheを思い浮かべる方が多いかもしれません。

w3techsによると、2025年8月11日現在、Webサイトの約33%がNginxを、約25%がApacheを利用しています。[^1]
一方でCaddyのシェアは0.3%ほど。NginxやApacheに比べると、知名度はそこまで高くないように感じます。

Caddyの初版は2015年と比較的新しく、NginxはCaddyより約10年先輩にあたります。
ですが、この若さこそがCaddyの魅力の一つで、メモリ安全性、高速なコンパイル、シンプルな設定ファイルなど、メジャーなWebサーバーにはない多くの利点があると思います。

## CaddyとHTTPS

Caddyの大きな特徴の一つはHTTPSの自動化だと思います。
これは[CertMagic](https://github.com/caddyserver/certmagic)によって実現され、デフォルトでHTTPSを有効化し、証明書の取得・更新も自動で行ってくれます。詳しくは[Automatic HTTPS](https://caddyserver.com/docs/automatic-https)を見ると、その手軽さがわかります。

ただし、ワイルドカード証明書を取得する場合は少し手間がかかります。
これは以前の記事でも書きましたが、ワイルドカード証明書を取得する場合や、何らかの理由でHTTP ポートが使えない、対象サーバーが外部からアクセスできない場合には[dns-01-challenge](https://letsencrypt.org/docs/challenge-types/#dns-01-challenge)が必要であり、この方式では検証中にDNSへ特定のレコードを追加しなければなりません。
そのため、自動化するにはDNSプロバイダーのAPIを使う必要があり、プロバイダーごとに仕様が異なるため追加作業が必要になるのです。

## CaddyでConoHaのDNSにdns-01-challengeができるようにした

今回は[ConoHa用のCaddyモジュール](https://github.com/caddy-dns/conoha)を作成したので、それを使ってワイルドカード証明書を取得してみます。

他のDNSプロバイダーでのやり方は[CaddyのWiki](https://caddy.community/t/how-to-use-dns-provider-modules-in-caddy-2/8148)にまとまっていますが、作業の流れはほぼ同じだと思います。

作業は大きく分けて次の2ステップです。

1. `xcaddy`をつかってConoHaのモジュールと一緒にをcaddyをビルド
2. `Caddyfile`の記述

今回はdockerをつかって作業していきます。
今回はdockerで進めますが、podmanなど他のコンテナでも同じ要領でできます。

### 1. xcaddyでbuild

まずは以下のような`Containerfile`を書きます。(各versionはそれぞれ最新のものに更新してください)
```
FROM  caddy:2.10.0-builder-alpine AS builder

RUN xcaddy build --with github.com/caddy-dns/conoha@v0.1.0

FROM caddy:2.10.0

COPY --from=builder /usr/bin/caddy /usr/bin/caddy

```

### 2. Caddyfileの記述
次に`Caddyfile`を書きます。

Caddyfileでは、ConohaのAPIをCaddy側から操作できるようにAPIクレデンシャルを入力する必要があります。
必要な情報は`APIテナントID`、`APIユーザーID`、`APIパスワード`で、いずれも[ConoHa VPS](https://www.conoha.jp)のAPIページから取得できます。
Caddyfileへの記述の仕方は、[github](https://github.com/caddy-dns/conoha)に書いて置いたので、こちらを参考にしてください。

以下のような感じになると思います。
```
mssht.net, *.mssht.net {
	tls {
		dns conoha {
			api_tenant_id {env.API_TENANT_ID}
			api_user_id {env.API_USER_ID}
			api_password {env.API_PASSWORD}
		}
		propagation_timeout 10m
		propagation_delay 5m
    }
...
```

今回はAPIクレデンシャルを環境変数から値を読んでいますが、Caddyfileに直接記述しても良いと思います。

あとは[Basic Usage, docker.com](https://hub.docker.com/_/caddy/#basic-usage)を参考にdockerなり、podmanなりで起動するだけです。
`docker-compose.yml`は以下のような感じに書けるかと思います。

```
services:
  caddy:
    build:
      context: ./caddy/
      dockerfile: Containerfile
    container_name: caddy
    restart: always
    ports:
      - "80:80"
      - "443:443"
    env_file:
      - ".env_caddy"
    volumes:
      - ./caddy/conf:/etc/caddy:ro
      - ./caddy/contents/public:/usr/share/caddy:ro

```

## 最後に

今回の開発によって以下2モジュールのメンテナになりました。なにかしらの提案だったり、要望等あれば、issue等に書き込んでいただけると助かります。
- https://github.com/libdns/conoha
- https://github.com/caddy-dns/conoha

そういえば、先日ConoHaのAPIが更新されましたね。[^2]
具体的には「サブユーザー」と「ロール」という概念が追加され、APIユーザーが自分のクレデンシャルを使ってロールを作成し、そのロールをサブユーザーにアタッチする形で利用できるようになりました。

これにより、実際にトークンを取得して操作するユーザーの権限を、より細かく設定できるようになりました。
[以前のポスト](/tech/add-conohav3-to-lego)でも書いたように、最小権限でないとAPIクレデンシャルが流出した際の被害が大きくなってしまうため、これは非常に良い変更だと思います。

ただし、残念ながらロールにアタッチできるパーミッションはVolumeやImageなど一部のAPIに限られ、DNSのAPIはまだ含まれていませんでした。
同機能が今後DNS APIにも適応されるのかどうかわかりませんが、追加されしだい、上記のモジュールは修正しようと思います。


[^1]: [Usage statistics and market shares of web servers, W3Techs](https://w3techs.com/technologies/overview/web_server)
[^2]: [APIでAPIサブユーザーを作成する, ConoHa](https://doc.conoha.jp/reference/api-vps3/api-utilization-vps3/api-create_subuser-v3/)
