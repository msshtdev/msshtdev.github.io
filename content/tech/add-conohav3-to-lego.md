---
# Common-Defined params
title: "legoを使ってConohaでワイルドカード証明書を取得する"
date: "2025-07-10T00:26:36+09:00"
description: "ConoHa VPS Ver3.0 を利用して、legoからConoHa VPSへのDNS-01チャレンジを自動化できるようにしました。"
categories:
  - "tech"
  - "ssl/tls"
tags:
  - "tech"
  - "ssl/tls"

# Theme-Defined params
thumbnail: "img/lego-logo.min.svg" # Thumbnail image
lead: "dns01-challengeを利用した証明書取得" # Lead text
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

## 経緯

先日、go-acmeの[lego](https://github.com/go-acme/lego)で新しいバージョンである`v4.24.0`がリリースされた。

このリリースに私が作成した[ConohaV3](https://go-acme.github.io/lego/dns/conohav3/)が組み込まれたので、宣伝も兼ねて紹介する。

作成した経緯としてはConohaのVPSの新バージョン(Version 3.0)の追加に伴い、新バージョンではapiの仕様が変更になり、既存のdnsモジュールでは、Version3.0のConohaを利用しているユーザーがlegoを使えなくなっていたことに気づいたためである。

## legoとConoha

### legoの利点とCertbotとの違い(主観)
Let's encryptでの証明書の取得でよく使うのは、Certbotだと思う。といか、大半の場合Certbotで事足りる。
Certbotとlegoの違いについてはメンテナの方が[discussionsでの質問](https://github.com/go-acme/lego/discussions/1914)に答える形で明らかにしている。

引用すると、
```
The main difference is the language: we use Go and Certbot uses Python.
lego is not a drop-in replacement for certbot because we don't have the same options, there are some other minor differences but both tools are here to generate certificates with the same approach.
```

deeplが翻訳すると、
```
主な違いは言語です。私たちはGoを使い、CertbotはPythonを使います。
legoは、同じオプションがないので、certbotの置き換えにはなりません。他にも細かい違いはありますが、どちらも同じアプローチで証明書を生成するツールです。
```
とのこと。

個人的にlegoの最大の利点は、DNS-01チャレンジの扱いやすさにあると思う。
CertbotでもDNS-01チャレンジは可能であるが、専用プラグインの導入が必要だったり、環境によっては設定が煩雑になりがちである。

それに対してlegoは、多くのDNSプロバイダに対応したドライバがあらかじめ用意されており、環境変数を設定するだけでDNS-01チャレンジを簡単に自動化できるという点が非常に便利である。
Goで実装されているため軽量であり、コンテナとの親和性も高く、CI/CDパイプラインにも組み込みやすい。

### なぜ他のVPSではなくConoHaなのか

DNS-01チャレンジでは、証明書の検証中にDNSゾーンにTXTレコードを動的に挿入する必要がある。
そのため、APIなどを通じてDNSレコードを自動操作できることが前提となる。

ConoHaは日本国内向けのVPSとしては珍しく、DNSゾーンの操作に対応したAPIを提供している。
これにより、legoと組み合わせることで証明書の取得・更新を完全に自動化できるという強みがある。

ただし、従来のlegoではConoHa API v2までしか対応しておらず、新しくリリースされたAPI v3では仕様変更により利用できない状態が続いていたっぽい。
今回、自分が作成したconohav3ドライバは、このギャップを埋めるためのものであり、ConoHa v3を使っているユーザーがlegoを再び利用できるようにすることを目的としている。

## 利用方法

利用するには、パッケージマネージャを使ったり、ソースからビルドしたり色々ある[^1] と思う が、今回はコンテナイメージを使う方法をで行う。

まずは.envファイル等にConohaのクレデンシャルを記述する。
```bash
$ cat .env
CONOHAV3_TENANT_ID=487727e3921d44e3bfe7ebb337bf085e                                                                                                
CONOHAV3_API_USER_ID=xxxx                                                                                                                          
CONOHAV3_API_PASSWORD=yyyy
```

その後、dockerを使ってステージング環境で正しく動作するかチェックする
```bash
docker run --env-file .env -v ./ssl:/.lego goacme/lego -m email.com --dns conohav3 -d '*.example.com' -d example.com -a -s https://acme-staging-v02.api.letsencrypt.org/directory run
```
`-s`や`--server`オプションでなど`https://acme-v02.api.letsencrypt.org/directory`を指定して実行することで、実際の証明書を発行せずに、ACMEクライアントの動作を本番と同様にテストできる。
`https://acme-v02.api.letsencrypt.org/directory`など、本番環境を使って証明書を取得する場合、Rate limitに注意する必要がある。[^2]

うまく行ったら、本番環境で実行し、有効な証明書を取得する。
```bash
docker run --env-file .env -v ./ssl:/.lego goacme/lego -m email.com --dns conohav3 -d '*.example.com' -d example.com -a run
```

なお、証明書取得の手続きにおいて以下のようなエラーが出るときがある。
```bash
propagation: time limit exceeded: last error: authoritative nameservers: NS a.conoha-dns.com.:53 returned NXDOMAIN for _acme-challenge.example.com.
```
dnsの伝播が間に合っていないようなので、`-dns.propagation-wait`などのオプションを利用して、伝播を待ってあげるとうまく行く可能性がある。

また、更新は以下のように行える。
```bash
docker run --env-file .env -v ./ssl:/.lego goacme/lego -m email.com --dns conohav3 -d '*.example.com' -d example.com -a  --renew-hook "docker restart nginx" renew
```

## ConohaでのDNS-01チャレンジとセキュリティ上の考慮

DNS-01チャレンジで実際に必要となる操作は、対象ドメインに対する_acme-challengeサブドメインへのTXTレコードの追加・削除のみである。
この点については、EFFのブログ記事[^3] でも言及されており、証明書の取得用途に限定した最小権限のAPIクレデンシャルを用意すべきであるとされている。

理想的には、DNSプロバイダがACLやスコープ付きトークンなどの仕組みを提供し、TXTレコード以外へのアクセスや、ゾーン全体への書き込み権限を制限できることが望ましい。

しかし、ConoHaのAPIクレデンシャルにはそのような権限制御の仕組みが存在しない。
ひとたびAPIクレデンシャルが漏洩すれば、DNSのすべてのレコードを編集できるだけでなく、セキュリティグループの設定変更やVPSインスタンスの削除・再起動といった操作も可能となってしまう。
これは単なる証明書発行のリスクにとどまらず、インフラ全体の完全な乗っ取りにつながる非常に深刻な問題である。

このようなリスクを前提とする以上、証明書自動化に用いるAPIクレデンシャルは、証明書発行専用として用途を分離し、短期間で定期的に再生成・無効化するなど、運用側での防御策が必要だと考える。


[^1]: https://go-acme.github.io/lego/installation/
[^2]: https://letsencrypt.org/docs/rate-limits/
[^3]: https://www.eff.org/deeplinks/2018/02/technical-deep-dive-securing-automation-acme-dns-challenge-validation
