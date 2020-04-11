# Lab 0 - Docker をインストールする

このラボでは、ラボで使用する Docker をインストールします。
ハンズオン事前準備として実施済であればスキップして、[Lab1](lab-1/README.md)を進めてください。


## 前提条件

Docker IDが下記で必要です
- Docker のインストール
- Docker をインストールせずに [Play with Docker](http://play-with-docker.com) を使用する場合
- Lab2 ([Docker Hub](https://hub.docker.com/) を使用するため)
- Lab3 ([Play with Docker](http://play-with-docker.com) を使用するため)

お持ちでない場合は https://hub.docker.com/ より [[Sign up for Docker Hub]](https://hub.docker.com/signup) をクリックして作成してください。登録にはメールアドレスが必要です。
Web での登録作業完了後、登録したメールアドレスにメールが送付されますので、そのメールにある確認用のリンクをクリックしてアカウント登録が完了します。

> Dockerのインストール要件:
> https://docs.docker.com/install/#supported-platforms より利用するOSをクリックし、確認できます(英語)。インストールできない場合は 下記オプションの [Play with Docker](http://play-with-docker.com) をご使用ください。

# ステップ 1: Docker をインストールする

1. https://hub.docker.com/ の `Sign in` をクリックし、サインインします(Docker IDとパスワードが必要です)。

2. `[Get Started with Docker Desktop]` のボタンをクリックして、モジュールをダウンロードし、インストールします。

> Docker には 2 つのインストール・バージョンが用意されています。「有償」バージョンは Docker Enterprise Edition (Docker EE)、「無償」バージョンは Docker Community Edition (Docker CE) です。このコースでは、Docker Community Edition で提供される無償の機能のみを使用します。

# **オプション:** Play with Docker を使用する
Docker をインストールしたくない場合は、代わりの方法として [Play with Docker](http://play-with-docker.com) を使用できます。Play with Docker は、Docker がインストールされているターミナルを、ブラウザーから直接実行できる Web サイトです。

このコースのすべてのラボは Play with Docker 上で実行できますが、このコースを完了した後も Docker を学べるよう、ローカル・ホストに Docker をインストールすることをお勧めします。

Play with Docker を使用する場合は、ブラウザー内で http://play-with-docker.com にアクセスします。Docker Hub にサインインしていない場合は `Login` ボタンが表示されます。`Login` ボタンをクリックしてDocker IDでサインインし、`Start` ボタンでアクセスできることを確認しておいてください（既にサインイン済みの場合は最初から `Start` ボタンが表示されます）。

# まとめ

Docker をインストールするか、http://play-with-docker.com の使い方を把握した後は、このコースの残りのラボに取り組んでください。

[Lab1](lab-1/README-ja.md)を進めてください。
