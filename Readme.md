
# Akebi

💠 **Akebi:** **A** **ke**yless https server, and **b**ackend dns server that resolves **i**p from domain

> Sorry, the documentation is currently in Japanese only. Google Translate is available.

インターネットに公開されていないプライベート Web サイトを「正規」の Let’s Encrypt の証明書で HTTPS 化するための、HTTPS リバースプロキシサーバーです。

この HTTPS リバースプロキシサーバーは、

- **権威 DNS サーバー:** `192-168-1-11.local.example.com` のようにサブドメインとして IP アドレスを指定すると、そのまま `192.168.1.11` に名前解決するワイルドカード DNS
- **API サーバー:** 事前に Let’s Encrypt で取得した証明書と秘密鍵を保持し、TLS ハンドシェイク時の証明書の供給と、Pre-master Secret Key の生成に使う乱数に秘密鍵でデジタル署名する API
- **デーモンプロセス:** Let’s Encrypt で取得した *.local.example.com の HTTPS ワイルドカード証明書と、API サーバーの HTTPS 証明書を定期的に更新するデーモン

の3つのコンポーネントによって構成される、**Keyless Server** に依存しています。

以下、HTTPS リバースプロキシサーバーを **HTTPS Server** 、上記の3つの機能を持つサーバーを **Keyless Server** と呼称します。

Keyless Server のコードの大半と HTTPS Server の TLS ハンドシェイク処理は、[ncruces](https://github.com/ncruces) さん開発の [keyless](https://github.com/ncruces/keyless) をベースに、私の用途に合わせてカスタマイズしたものです。  
偉大な発明をしてくださった ncruces さんにこの場で心から感謝を申し上げます（私が書いたコードは 20% 程度にすぎません）。

## 開発背景

**Akebi は、オレオレ証明書以外での HTTPS 化が困難なローカル LAN 上でリッスンされるサーバーアプリケーションを、Let's Encrypt 発行の正規の HTTPS 証明書で HTTPS 化するために開発されました。**

ローカル LAN やイントラネットなどのプライベートネットワークでリッスンされている Web サーバーは、HTTP でリッスンされていることがほとんどです。

これは盗聴されるリスクが著しく低く、VPN 経由なら元々暗号化されているなどの理由で HTTPS にする必要がないこと、プライベートネットワークで信頼される HTTPS 証明書の入手が事実上難しいことなどが理由でしょう。HTTP の方が単純で簡単ですし。

### ブラウザの HTTPS 化の圧力

…ところが、最近のブラウザはインターネット上に公開されている Web サイトのみならず、**盗聴のリスクが著しく低いプライベートネットワーク上の Web サイトにも、HTTPS を要求するようになってきました。**

すでに PWA の主要機能である Service Worker や Web Push API などをはじめ、近年追加された多くの Web API の利用に（中には WebCodecs API のような HTTPS 化を必須にする必要が皆無なものも含めて）**HTTPS が必須になってしまっています。**

> 正確には **[安全なコンテキスト (Secure Contexts)](https://developer.mozilla.org/ja/docs/Web/Security/Secure_Contexts)** でないと動作しないようになっていて、特別に localhost (127.0.0.1) だけは http:// で通信しても安全なコンテキストだと認められるようになっています。

プライベート Web サイトでも、たとえばビデオチャットのために [getUserMedia()](https://developer.mozilla.org/ja/docs/Web/API/MediaDevices/getUserMedia) を、クリップボードにコピーするために [Clipboard API](https://developer.mozilla.org/ja/docs/Web/API/Clipboard_API) を使いたい要件が出てくることもあるでしょう（どちらも Secure Contexts が必須です）。  

- せっかくコードは Service Worker に対応しているのに、HTTP では Service Worker が動かないのでキャッシュが効かず、読み込みがたびたび遅くなる
- PWA でホーム画面にインストールしてもアイコンが Chrome 扱いになるし、フォームに入力すると上部に「保護されていない通信」というバナーが表示されてうざい
- Clipboard API・Storage API・SharedArrayBuffer などの強力な API が Secure Contexts でないと使えず、今後の機能開発が大きく制約される

私が開発している [KonomiTV](https://github.com/tsukumijima/KonomiTV) でも、上記のような課題を抱えていました。  

しかも、最近新たに追加された API はその性質に関わらず問答無用で [Secure Contexts が必須になっている](https://developer.mozilla.org/ja/docs/Web/Security/Secure_Contexts/features_restricted_to_secure_contexts) ことが多く、リッチなプライベート Web サイトの開発はかなりやりづらくなってきています。

さらに、Chrome 94 から適用された [Private Network Access](https://developer.chrome.com/blog/private-network-access-update/) という仕様のおかげで、**HTTP の公開 Web サイトからプライベート Web サイトにアクセスできなくなりました。** CORS ヘッダーで明示的に許可していても、です。

以前より HTTPS の公開 Web サイトから HTTP のプライベート Web サイトへのアクセスは、Mixed Content として禁止されています (localhost を除く) 。そのため、公開 Web サイトも HTTP (Public (HTTP) -> Private (HTTP)) にせざるを得なかったのですが、それすらも禁止されてしまいました。

こうした変更は、公開 Web サイトからローカル LAN 上にあるデバイスを操作する類のアプリケーションにとって、かなり厳しい制約になります。

> Chrome 101 以降では、Public (HTTPS) -> Private (HTTPS) のアクセスには、さらに `Access-Control-Allow-Private-Network` ヘッダーが必要になるようです。  
> Chrome 101 以降も公開 Web サイトからプライベート Web サイトにアクセスするには両方の HTTPS 化が必須で、加えて Preflight リクエストが飛んできたときに `Access-Control-Allow-Private-Network: true` を返せる必要が出てきます。

### プライベート Web サイトの証明書取得の困難さ

一般的な公開 Web サイトなら、Let's Encrypt を使えば無料で簡単に HTTPS 化できます。無料で HTTPS 証明書を取れるようになったこともあって、ブラウザによる HTTPS 化の圧力は年々強まっています。

しかし、プライベート Web サイトの場合、**正攻法での HTTPS 化は困難を極めます。**  
まず、インターネット上から Web サーバーにアクセスできないため、Let's Encrypt の HTTP-01 チャレンジが通りません。  
…それ以前に Let's Encrypt は元々 IP アドレス宛には証明書を発行できませんし、グローバル IP ならまだしも、世界各地で山ほど被りまくっているプライベート IP の所有権を主張するのには無理があります。

そこでよく利用されるのが、**自己署名証明書（オレオレ証明書）を使った HTTPS 化**です。

自分で HTTPS 証明書を作ってしまう方法で、プライベート IP アドレスだろうが関係なく、自由に証明書を作成できます。  
最近では [mkcert](https://github.com/FiloSottile/mkcert) のような、オレオレ証明書をかんたんに生成するツールも出てきています。

自分で作った証明書なので当然ブラウザには信頼されず、そのままではアクセスすると警告が表示されてしまいます。  
ブラウザに証明書を信頼させ「この接続ではプライバシーが保護されません」の警告をなくすには、**生成したオレオレ証明書を OS の証明書ストアに「信頼されたルート証明機関」としてインストールする必要があります。**

mkcert はそのあたりも自動化してくれますが、それはあくまで開発時の話。  
まず mkcert をインストールした以外の PC やスマホには手動でインストールしないといけませんし、インストール方法もわりと面倒です。開発者ならともかく、一般ユーザーには難易度が高い作業だと思います。  
しかも、プライベート Web サイトを閲覧するデバイスすべてにインストールしなければならず、デバイスが多ければ多いほど大変です。

…こうした背景から、**一般ユーザーに配布するアプリケーションでは、事実上オレオレ証明書は使えません。**  
もちろんユーザー体験を犠牲にすれば使えなくはありませんが、より多くの方に簡単に使っていただくためにも、できるだけそうした状態は避けたいです。

### Let's Encrypt の DNS 認証 + ワイルドカード DNS という選択肢

閑話休題。オレオレ証明書に押されてあまり知られていないのですが、**実はプライベート Web サイトでも、Let's Encrypt の DNS 認証 (DNS-01 チャレンジ) を使えば、正規の HTTPS 証明書を取ることができます。**  
詳細は [この記事](https://blog.jxck.io/entries/2020-06-29/https-for-localhost.html) が詳しいですが、軽く説明します。

通常、DNS 上の A レコードにはグローバル IP アドレスを指定します。ですが、とくにグローバル IP アドレスでないといけない制約があるわけではありません。`127.0.0.1` や `192.168.1.1` を入れることだって可能です。

たとえば、`local.example.com` の A レコードを `127.0.0.1` に設定したとします。もちろんループバックアドレスなのでインターネット上からはアクセスできませんし、Let's Encrypt の HTTP 認証は通りません。

そこで、**Let's Encrypt の DNS 認証 (DNS-01 チャレンジ) で HTTPS 証明書を取得します。**  
DNS 認証は、例でいう `local.example.com` の DNS を変更できる権限（≒ドメインの所有権）を証明することで、HTTPS 証明書を取得する方法です。  
DNS 認証ならインターネットからアクセスできる必要はなく、**DNS 認証時に `_acme-challenge.local.example.com` の TXT レコードにトークンを設定できれば、あっさり HTTPS 証明書が取得できます。**

……一見万事解決のように見えます。が、この方法はイントラネット上のサイトなどでプライベート IP アドレスが固定されている場合にはぴったりですが、**不特定多数の環境にインストールされるプライベート Web サイトでは、インストールされる PC のプライベート IP アドレスが環境ごとにバラバラなため、そのままでは使えません。**

**そこで登場するのがワイルドカード DNS サービスです。**[nip.io](https://nip.io/) や [sslip.io](https://sslip.io/) がよく知られています。  
これらは **`http://192-168-1-11.sslip.io` のようなサブドメインを `192.168.1.11` に名前解決してくれる特殊な DNS サーバー**で、sslip.io の方は自分が保有するドメインをワイルドカード DNS サーバーにすることもできます。

また、**実は Let's Encrypt ではワイルドカード証明書を取得できます。** ドメインの所有権を証明できれば、`hoge.local.example.com`・`fuga.local.example.com`・`piyo.local.example.com` いずれでも使える証明書を発行できます。

このワイルドカード DNS サービスと取得したワイルドカード証明書を組み合わせれば、**`http://192.168.1.11:7000/` の代わりに `http://192-168-1-11.local.example.com` にアクセスするだけで、魔法のように正規の証明書でリッスンされるプライベート HTTPS サイトができあがります！**

### 証明書と秘密鍵の扱い

経緯の説明がたいへん長くなってしまいましたが、ここからが本番です。

上記の手順を踏むことで、プライベート Web サイトでも HTTPS 化できる道筋はつきました。  
ですが、不特定多数の環境にインストールされるプライベート Web サイト（そう多くはないが、著名な例だと Plex Media Server などの一般ユーザーに配布されるアプリケーションが該当する）では、**HTTPS 証明書・秘密鍵の扱いをどうするかが問題になります。**

アプリケーション自体を配布しなければならないので、当然証明書と秘密鍵もアプリケーションに同梱しなければなりません。ですが、このうち秘密鍵が漏洩すると、別のアプリケーションがなりすましできたり、通信を盗聴できたりしてしまいます（中間者攻撃）。

もっとも今回はブラウザへの建前として形式上 HTTPS にしたいだけなのでその点は正直どうでもいいのですが、それよりも **「証明書と秘密鍵があれば誰でも HTTPS 証明書を失効できてしまう」「秘密鍵の公開は Let's Encrypt の利用規約で禁止されている」点が厄介です。**

アプリケーションの内部に秘密鍵を隠すこともできますが、所詮は DRM のようなもので抜本的とはいえないほか、OSS の場合は隠すこと自体が難しくなります。  
また、Let's Encrypt 発行の HTTPS 証明書は3ヶ月で有効期限が切れるため、各環境にある証明書・秘密鍵をどうアップデートするかも問題になります。

**この「秘密鍵の扱いをどうするか」問題を、TLS ハンドシェイクの内部処理をハックし秘密鍵をリモートサーバーに隠蔽することで解決させた点が、Akebi HTTPS Server の最大の特徴です。**

> 証明書も TLS ハンドシェイク毎に Keyless Server からダウンロードするため、保存した証明書の更新に悩む必要がありません。

秘密鍵をリモートサーバーに隠蔽するためには、TLS ハンドシェイク上で秘密鍵を使う処理を、サーバー上で代わりに行う API サーバーが必要になります。  
**どのみち API サーバーが要るなら、sslip.io スタイルのワイルドカード DNS と Let's Encrypt の証明書自動更新までまとめてやってくれる方が良いよね？ということで開発されたのが、[ncruces](https://github.com/ncruces) さん開発の [keyless](https://github.com/ncruces/keyless) です。**

私がこの keyless をもとに若干改良したものが Akebi Keyless Server で、Akebi HTTPS Server とペアで1つのシステムを構成しています。

> HTTPS リバースプロキシの形になっているのは、**HTTPS 化対象のアプリケーションがどんな言語で書かれていようと HTTP サーバーのリバースプロキシとして挟むだけで HTTPS 化できる汎用性の高さ**と、**そもそも TLS ハンドシェイクの細かい処理に介入できるのが Golang くらいしかなかった**のが理由です。  
> 詳細は 技術解説と注意 の項で説明しています。

## 導入

### 必要なもの

- Linux サーバー (VM・VPS)
  - Keyless Server を動かすために必要です。
  - Keyless Server は UDP 53 ポート (DNS) と TCP 443 ポート (HTTPS) を使用します。
    - それぞれ外部ネットワークからアクセスできるようにファイアウォールを設定してください。
  - Keyless Server がダウンしてしまうと、その Keyless Server に依存する HTTPS Server も起動できなくなります。安定稼働のためにも、Keyless Server は他のサイトと同居させないことをおすすめします。
  - サーバーは低スペックなものでも大丈夫です。私は Oracle Cloud Free Tier の AMD インスタンスで動かしています。
  - Ubuntu 20.04 LTS で動作を確認しています。
- 自分が所有するドメイン
  - Keyless Server のワイルドカード DNS 機能と、API サーバーのドメインに利用します。
  - ワイルドカード DNS 機能用のドメインは、たとえば `example.net` を所有している場合、`local.example.net` や `ip.example.net` などのサブドメインにすると良いでしょう。
    - IP → ドメインのための専用のドメインを用意できるなら、必ずしもサブドメインである必要はありません。
    - この例の場合、`192-168-1-11.local.example.net` が 192.168.1.11 に名前解決されるようになります。
  - もちろん、ドメインの DNS 設定を変更できることが前提です。

### Keyless Server のセットアップ

以下は Ubuntu 20.04 LTS でのインストール手順です。

#### Golang のインストール

```bash
$ sudo add-apt-repository ppa:longsleep/golang-backports
$ sudo apt install golang
```

#### systemd-resolved を止める

ワイルドカード DNS サーバーを動かすのに必要です（53番ポートがバッティングするため）。  
他にもっとスマートな回避策があるかもしれないので、参考程度に…。

```bash
$ sudo systemctl disable systemd-resolved
$ sudo systemctl stop systemd-resolved
$ sudo mv /etc/resolv.conf /etc/resolv.conf.old  # オリジナルの resolv.conf をバックアップ
$ sudo nano /etc/resolv.conf
---------------------------------------------
nameserver 1.1.1.1 1.0.0.1  # ← nameserver を 127.0.0.53 から変更する
(以下略)
---------------------------------------------
```

#### DNS 設定の変更

ここからは、Keyless Server を立てたサーバーに割り当てるドメインを **`akebi.example.com`** 、ワイルドカード DNS で使うドメインを **`local.example.com`** として説明します。

`example.com` の DNS 設定で、`akebi.example.com` の A レコードに、Keyless Server を立てたサーバーの IP アドレスを設定します。IPv6 用の AAAA レコードを設定してもいいでしょう。

次に、`local.example.com` の NS レコードに、ネームサーバー（DNSサーバー）として `akebi.example.com` を指定します。

この設定により、`192-168-1-11.local.example.com` を `192.168.1.11` に名前解決するために、`akebi.example.com` の DNS サーバー (UDP 53 番ポート) に DNS クエリが飛ぶようになります。  

#### インストール

```bash
$ sudo apt install make  # make が必要
$ git clone git@github.com:tsukumijima/Akebi.git
$ cd Akebi
$ make build-keyless-server  # Keyless Server をビルド
$ cp ./example/akebi-keyless-server.json ./akebi-keyless-server.json  # 設定ファイルをコピー
```

`akebi-keyless-server.json` が設定ファイルです。JSONC (JSON with comments) で書かれています。  
実際に変更が必要な設定は4つだけです。

- `domain`: ワイルドカード DNS で使うドメイン（この例では `local.example.com`）を設定します。
- `nameserver`: `local.example.com` の NS レコードに設定したネームサーバー（この例では `akebi.example.com`）を設定します。
- `is_private_ip_ranges_only`: ワイルドカード DNS の名前解決範囲をプライベート IP アドレスに限定するかを設定します。
  - この設定が true のとき、たとえば `192-168-1-11.local.example.com` や `10-8-0-1.local.example.com` は名前解決されますが、`142-251-42-163.local.example.com` は名前解決されず、ドメインが存在しない扱いになります。
  - プライベート IP アドレスの範囲には [Tailscale](https://tailscale.com/) の IP アドレス (100.64.0.0/10, fd7a:115c:a1e0:ab12::/64) も含まれます。
  - グローバル IP に解決できてしまうと万が一フィッシングサイトに使われないとも限らない上、用途上グローバル IP に解決できる必要性がないため、個人的には true にしておくことをおすすめします。
- `keyless_api.handler`: Keyless API サーバーの URL（https:// のような URL スキームは除外する）を設定します。
  - `akebi.example.com/` のように指定します。末尾のスラッシュは必須です。

#### セットアップ

```bash
$ sudo ./akebi-keyless-server setup
```

セットアップスクリプトを実行します。  
セットアップ途中で DNS サーバーを起動しますが、53 番ポートでのリッスンには root 権限が必要なため、sudo をつけて実行します。

```
Running setup...

Creating a new Let's Encrypt account...
Creating a new account private key...

Accept Let's Encrypt ToS? [y/n]: y
Use the Let's Encrypt production API? [y/n]: y
Enter an email address: yourmailaddress@example.com

Creating a new master private key...

Starting DNS server for domain validation...
Please, ensure that:
 - NS records for local.example.com point to akebi.example.com
 - akebi-keyless-server is reachable from the internet on UDP akebi.example.com:53
Continue? y

Obtaining a certificate for *.local.example.com...
Creating a new Keyless API private key...

Starting HTTPS server for hostname validation...
Please, ensure that:
 - akebi-keyless-server is reachable from the internet on TCP akebi.example.com:443
Continue?
Obtaining a certificate for akebi.example.com...

Done!
```

```bash
$ sudo chown -R $USER:$USER ./
```

これで Keyless Server を起動できる状態になりました！  
root 権限で作られたファイル類の所有者をログイン中の一般ユーザーに設定しておきましょう。

certificates/ フォルダには、Let's Encrypt から取得した HTTPS ワイルドカード証明書/秘密鍵と、API サーバーの HTTPS 証明書/秘密鍵が格納されています。  
letsencrypt/ フォルダには、Let's Encrypt のアカウント情報が格納されています。

#### Systemd サービスの設定

Keyless Server は Systemd サービスとして動作します。  
Systemd に Keyless Server サービスをインストールし、有効化します。

```bash
# サービスファイルをコピー
$ sudo cp ./example/akebi-keyless-server.service /etc/systemd/system/akebi-keyless-server.service

# /home/ubuntu/Akebi の部分を Akebi を配置したディレクトリのパスに変更する
$ sudo nano /etc/systemd/system/akebi-keyless-server.service

# ソケットファイルをコピー
$ sudo cp ./example/akebi-keyless-server.socket /etc/systemd/system/akebi-keyless-server.socket

# サービスを有効化
$ sudo systemctl daemon-reload
$ sudo systemctl enable akebi-keyless-server.service
$ sudo systemctl enable akebi-keyless-server.socket

# サービスを起動
# akebi-keyless-server.socket は自動で起動される
$ sudo systemctl start akebi-keyless-server.service
```

**`https://akebi.example.com` にアクセスして 404 ページが表示されれば、Keyless Server のセットアップは完了です！** お疲れ様でした。

**Keyless Server が起動している間、Let's Encrypt から取得した HTTPS 証明書は自動的に更新されます。** 一度セットアップすれば、基本的にメンテナンスフリーで動作します。

```
● akebi-keyless-server.service - Akebi Keyless Server Service
     Loaded: loaded (/etc/systemd/system/akebi-keyless-server.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2022-05-21 07:31:34 UTC; 2h 59min ago
TriggeredBy: ● akebi-keyless-server.socket
   Main PID: 767 (akebi-keyless-s)
      Tasks: 7 (limit: 1112)
     Memory: 7.8M
     CGroup: /system.slice/akebi-keyless-server.service
             └─767 /home/ubuntu/Akebi/akebi-keyless-server
```

`systemctl status akebi-keyless-server.service` がこのようになっていれば、正しく Keyless Server を起動できています。

```
$ sudo systemctl stop akebi-keyless-server.service
$ sudo systemctl stop akebi-keyless-server.socket
```

Keyless Server サービスを終了したい際は、以上のコマンドを実行してください。

### HTTPS Server のセットアップ

#### ビルド

HTTPS Server のビルドには、Golang と make がインストールされている環境が必要です。ここではすでにインストールされているものとして説明します。  

> Windows 版の make は [こちら](http://gnuwin32.sourceforge.net/packages/make.htm) からインストールできます。  
> 2006 年から更新されていませんが、Windows 10 でも普通に動作します。それだけ完成されたアプリケーションなのでしょう。

```bash
$ git clone git@github.com:tsukumijima/Akebi.git
$ cd Akebi

# 現在のプラットフォーム向けにビルド
$ make build-https-server

# すべてのプラットフォーム向けにビルド
# Windows (64bit), Linux (amd64), Linux (arm64) 向けの実行ファイルを一度にクロスコンパイルする
$ make build-https-server
```

#### セットアップ

HTTPS Server は、設定を同じフォルダ内の `akebi-keyless-server.json` から読み込みます。Keyless Server 同様、JSONC (JSON with comments) で書かれています。  

設定はコマンドライン引数からも行えます。引数はそれぞれ設定ファイルの項目に対応しています。  
設定ファイルが配置されているときにコマンドライン引数を指定した場合は、コマンドライン引数の方の設定が優先されます。

- `listen_address`: HTTPS リバースプロキシをリッスンするアドレスを指定します。
  - コマンドライン引数では `--listen-address` に対応します。
  - 基本的には `0.0.0.0:(ポート番号)` のようにしておけば OK です。
- `proxy_pass_url`: リバースプロキシする HTTP サーバーの URL を指定します。
  - コマンドライン引数では `--proxy-pass-url` に対応します。
- `keyless_server_url`: Keyless Server の URL を指定します。 
  - コマンドライン引数では `--keyless-server-url` に対応します。
- `custom_certificate`: Keyless Server を使わず、カスタムの HTTPS 証明書/秘密鍵を使う場合に設定します。
  - コマンドライン引数では `--custom-certificate` `--custom-private-key` に対応します。
  - 普通に HTTPS でリッスンするのと変わりませんが、Keyless Server を使うときと HTTPS サーバーを共通化できること、HTTP/2 に対応できることがメリットです。**

#### HTTPS リバースプロキシの起動

HTTPS Server はビルド後の実行ファイル単体で動作します。  
`akebi-keyless-server.json` を実行ファイルと同じフォルダに配置しない場合は、実行時にコマンドライン引数を指定する必要があります。

```bash
$ ./akebi-https-server
2022/05/22 03:49:36 Info:  Starting HTTPS reverse proxy server...
2022/05/22 03:49:36 Info:  Listening on 0.0.0.0:7000, Proxing http://your-http-server-url:8080/.
```

**この状態で https://local.local.example.com:7000/ にアクセスしてプロキシ元のサイトが表示されれば、正しく HTTPS 化できています！！**

HTTPS Server は Ctrl + C で終了できます。  
設定内容にエラーがあるときはログが表示されるので、それを確認してみてください。

ドメインの本来 IP アドレスを入れる部分に **`local` または `localhost` と入れると、特別に 127.0.0.1（ループバックアドレス）に名前解決されように設定しています。**  
`127-0-0-1.local.example.com` よりもわかりやすいと思います。ローカルで開発する際にお使いください。

**HTTPS Server は HTTP/2 に対応しています。** HTTP/2 は HTTPS でしか使えませんが、サイトを HTTPS 化することで、同時に HTTP/2 に対応できます。

> どちらかと言えば、Golang の標準 HTTP サーバー ([http.Server](https://pkg.go.dev/net/http#Server)) が何も設定しなくても HTTP/2 に対応していることによるものです。

カスタムの証明書/秘密鍵を指定できるのも、HTTPS 化に Keyless Server を使わない場合と実装を共通化できるのもありますが、HTTPS Server を間に挟むだけでかんたんに HTTP/2 に対応できることが大きいです。

[Uvicorn](https://github.com/encode/uvicorn) など、HTTP/2 に対応していないアプリケーションサーバーはそれなりにあります。本来は NGINX などを挟むべきでしょうけど、一般ユーザーに配布するアプリケーションでは、簡易な HTTP サーバーにせざるを得ないことも多々あります。  
そうした場合でも、**アプリケーション本体の実装に手を加えることなく、アプリケーション本体の起動と同時に HTTPS Server を起動するだけで、HTTPS 化と HTTP/2 対応を同時に行えます。**

```bash
$ ./akebi-https-server --listen-address 0.0.0.0:8080 --proxy-pass-url http://192.168.1.11:8000
2022/05/22 03:56:50 Info:  Starting HTTPS reverse proxy server...
2022/05/22 03:56:50 Info:  Listening on 0.0.0.0:8080, Proxing http://192.168.1.11:8000.
```

`--listen-address` や `--proxy-pass-url` オプションを指定して、リッスンポートやプロキシ対象の HTTP サーバーの URL を上書きできます。

```bash
$ ./akebi-https-server -h
Usage of C:\Develop\Akebi\akebi-https-server.exe:
  -custom-certificate string
        Optional: Use your own HTTPS certificate instead of Akebi Keyless Server.
  -custom-private-key string
        Optional: Use your own HTTPS private key instead of Akebi Keyless Server.
  -keyless-server-url string
        URL of HTTP server to reverse proxy.
  -listen-address string
        Address that HTTPS server listens on.
        Specify 0.0.0.0:port to listen on all interfaces.
  -mtls-client-certificate string
        Optional: Client certificate of mTLS for akebi.example.com (Keyless API).
  -mtls-client-certificate-key string
        Optional: Client private key of mTLS for akebi.example.com (Keyless API).
  -proxy-pass-url string
        URL of HTTP server to reverse proxy.
```

`-h` オプションでヘルプが表示されます。

## 技術解説と注意

![](https://blog.cloudflare.com/content/images/2014/Sep/cloudflare_keyless_ssl_handshake_diffie_hellman.jpg)

Akebi の最大の特徴である、**秘密鍵をユーザーに公開せずに正規の HTTPS サーバーを立てられる**というトリックは（”Keyless” の由来）、Cloudflare の [Keyless SSL](https://blog.cloudflare.com/keyless-ssl-the-nitty-gritty-technical-details/) で用いられた手法が参考にされています。

『ワイルドカード DNS と Let's Encrypt のワイルドカード証明書を組み合わせてローカル LAN で HTTPS サーバーを実現する』というアイデアは、[Corollarium](https://github.com/Corollarium) 社開発の [localtls](https://github.com/Corollarium/localtls) を参考にしたものだそうです。

[原作者の ncruces 氏によれば](https://github.com/cunnie/sslip.io/issues/6#issuecomment-778914231)、Cloudflare の Keyless SSL で使われている RSA 暗号ではなく、実装を単純化するために ECDSA 暗号で実装したとのこと。Keyless Server のセットアップで生成された秘密鍵のサイズが小さいのはそのためです。  
Keyless SSL と大まかな手法は同じですが、Key Server との通信プロトコルは異なるため（大幅に簡略化されている）、Keyless SSL と互換性があるわけではありません。

この手法は非常に優れていますが、**中間者攻撃 (MitM) のリスクがないわけではありません。**  
証明書と秘密鍵がそのまま公開されている状態からすると遥かに難易度が跳ね上がりますが、Keyless API にはどこからでもアクセスできるため、秘密鍵が必要な処理を Keyless API にやらせてしまえば、中間者攻撃ができてしまうかもしれません（セキュリティエンジニアではないので詳しいことはわからない…）。

（執筆中…）

## License

[MIT License](License.txt)
