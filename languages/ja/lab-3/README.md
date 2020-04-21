# Lab3 - オーケストレーション入門

これまでのラボで、Docker を使用してローカル・マシン上でアプリケーションを実行する方法を学んできましたが、Docker でコンテナ化したアプリケーションを本番環境で実行する場合はどうなるでしょうか？本番対応のアプリケーションを構築する際にはいくつもの問題が伴います。数例を挙げるだけでも、分散された複数のノードでのサービスのスケジューリング、高可用性の維持、調整の実装、スケーリング、ロギングといった問題に対処しなければなりません。

こうした問題を解決するため、いくつかのオーケストレーション・ソリューションを利用できます。そのうちの 1 つが、[IBM Cloud Kubernetes Service](https://www.ibm.com/jp-ja/cloud/container-service) です。このサービスでは本番環境内のコンテナを、[Kubernetes](https://kubernetes.io/) を使用して実行します。

Kubernetes について紹介する前に、Docker Swarm を使用してアプリケーションをオーケストレーションする方法を説明します。Docker Swarm は、Docker Engine に組み込まれているオーケストレーション・ツールです。

このラボで使用する Docker コマンドは少数に限られています。使用できるコマンドについて詳しくは、[公式のドキュメント](http://docs.docker.jp/)を参照してください。

## 前提条件

このラボではアプリケーションを複数のホストにデプロイしてオーケストレーションする作業に取り組むので、当然、複数のホストが必要です。作業を簡単にするために、このラボでは http://play-with-docker.com で提供しているマルチノード・サポートを使用します。Docker Swarm をテストするには、マルチノード・サポートを使用するのが最も簡単な方法です。このサポートを使用する場合、複数のホストに Docker をインストールする必要はありません。

## ステップ 1: 初めての Swarm を作成する

このステップでは、Play with Docker を使用して初めての Swarm を作成します。

1. http://play-with-docker.com にアクセスします。

2. 左側にある「add new instance (新しいインスタンスを作成)」を 3 回クリックして、3 つのノードを作成します。

  初めて作成する Swarm クラスターは、この 3 つのノードで構成されます。

3. node1 上で Swarm を初期化します。

  ```sh
  $ docker swarm init --advertise-addr eth0
  Swarm initialized: current node (v0rnc435bpqjhtfuurqqms54k) is now a manager.

  To add a worker to this swarm, run the following command:

      docker swarm join --token SWMTKN-1-0ptxqjzg4633c9o1mfz4w86ri4btel2apqwuo6a42qwhhznvc3-9mawd536d0kshzaxpyet8u9cn 192.168.0.43:2377

  To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
  ```

  Docker Swarm はいわば、コマンド `docker swarm init` によってアクティブ化される特殊な「モード」です。他のノードが Swarm に参加する際は、`--advertise-addr` によって指定されたアドレスを使用します。

  上記の `docker swarm init` コマンドにより、クラスター参加用のトークンが生成されます。このトークンは、不正なノードが Swarm に参加できないようにするためのものです。このトークンがなければ、他のノードを Swarm に参加させることはできません。コピーして他のノードに貼り付けられるよう、上記の出力には完全な `docker swarm join` コマンドを記載しています。

4. node2 と node3 の両方で、前のコマンドでコンソールに出力された `docker swarm join` コマンドをコピーして実行します。

  これで、3 つのノードからなる Swarm が構成されました！

5. node1 に戻って `docker node ls` を実行し、3 つのノードからなるクラスターを確認します。

  ```sh
  $ docker node ls
  ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
  7x9s8baa79l29zdsx95i1tfjp     node3               Ready               Active
  x223z25t7y7o4np3uq45d49br     node2               Ready               Active
  zdqbsoxa6x1bubg3jyjdmrnrn *   node1               Ready               Active              Leader
  ```

  上記のコマンドにより、Swarm を構成する 3 つのノードが出力されます。ノードの ID の横に示されている * は、そのノードが、この特定のコマンド (この例では `docker node ls`) を処理したノードであることを意味します。  

  この例の場合、ノードには 1つのマネージャー・ノードと 2つのワーカー・ノードがあります。マネージャーはコマンドを処理し、Swarm の状態を管理します。ワーカーがコマンドを処理することはできません。ワーカーはコンテナを大規模に実行するためだけに使用されます。デフォルトでは、コンテナを実行するためにもマネージャーが使用されます。

  このラボの残りで使用するすべての `docker service` コマンドは、マネージャー・ノード (node1) 上で実行する必要があります。

> **注:** ここでは Swarm が実行されているノードから直接 Swarm を制御しますが、リモートから Docker Swarm を制御することもできます。それには、リモート API を介してマネージャーの Docker Engine に接続するか、ローカルの Docker インストール環境からリモート・ホストをアクティブ化します (`$DOCKER_HOST` 環境変数と `$DOCKER_CERT_PATH` 環境変数を使用)。本番サーバーに直接 SSH で接続するのではなく、リモートから本番アプリケーションを制御する必要がある場合は、この方法が役立ちます。

## ステップ 2: 初めてのサービスをデプロイする

3 つのノードからなる Swarm クラスターが初期化されたので、次は、コンテナをいくつかデプロイしましょう。Docker Swarm 上でコンテナを実行するには、サービスを作成する必要があります。サービスとは、分散型クラスターにデプロイされている、同じイメージから作成された複数のコンテナを表す抽象概念です。

Nginx を使用した単純な例に取り組みましょう。とりあえず 1 つの実行中コンテナだけでサービスを作成しますが、後でこれをスケールアップします。

1. Nginx を使用してサービスをデプロイします。

  ```sh
  $ docker service create --detach=true --name nginx1 --publish 80:80  --mount source=/etc/hostname,target=/usr/share/nginx/html/index.html,type=bind,ro nginx:1.12 pgqdxr41dpy8qwkn6qm7vke0q
  ```

  上記のステートメントは「*宣言型*」です。別の `docker service` コマンドによって明示的状態が変更されない限り、Docker Swarm はこのコマンドで宣言されている状態を積極的に維持しようとします。この動作は、例えばノードが停止状態になったときに役立ちます。その場合、この動作によってコンテナが別のノードで自動的に再スケジューリングされるためです。後で、このラボでその例を説明します。

  `--mount` フラグは、ノードが実行されているホスト名を Nginx に出力させる巧みな手段です。複数のコンテナの間で負荷を分散するようになると、このフラグが重宝します。このラボでは後で、クラスター内のさまざまなノードに分散された Nginx コンテナの間で負荷分散を行うときに、このフラグを使用して、リクエストに対応している Swarm 内のノードを確認します。

  上記のコマンドでは、nginx タグで「1.12」を指定しています。後で、このラボではバージョン 1.13 を使用してローリング・アップデートを行う例を説明します。

  `--publish` コマンドでは、Swarm の組み込み「*ルーティング・メッシュ*」機能を利用しています。この例の場合、「*Swarm 内のすべてのノード*」上でポート 80 が公開されます。ルーティング・メッシュにより、ポート 80 上の着信リクエストは、ノードのうち、コンテナを実行しているノードにルーティングされます。

2. サービスを調べます。

  先ほど作成したサービスを調べるには、`docker service ls` を使用します。

  ```sh
  $ docker service ls
  ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
  pgqdxr41dpy8        nginx1              replicated          1/1                 nginx:1.12          *:80->80/tcp
  ```

3. サービスの実行中コンテナを調べます。

  実行中のタスクを詳しく調べるには、`docker service ps` を使用します。タスクも Docker Swarm で使用されている抽象概念の 1 つであり、サービスの実行中インスタンスを表します。この例の場合、タスクとコンテナは 1 対 1 でマッピングされています。

  ```sh
  $ docker service ps nginx1
  ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
iu3ksewv7qf9        nginx1.1            nginx:1.12          node1               Running             Running 8 minutes ago
  ```

  コンテナが実行されているノードがわかっていれば (どのノードであるかは、`docker service ps` の出力を基に判断できます)、`docker container ls` を使用して、その特定のノード上で実行されているコンテナを確認できます。

4. サービスをテストします。

  ルーティング・メッシュにより、Swarm のどのノードにでもポート 80 上でリクエストを送信できます。送信したリクエストは Nginx コンテナを実行しているノードに自動的にルーティングされます。

  次のコマンドを各ノード上で試してください。

  ```sh
  $ curl localhost:80
  node1
  ```

  curl コマンドにより、コンテナが実行されているホスト名が出力されます。この例でコンテナが実行されているのは「node1」ですが、皆さんの出力では異なる場合があります。

## ステップ 3: サービスをスケーリングする

本番では、アプリケーションへの大量のトラフィックを処理しなければならないはずです。本番に備えてサービスをスケーリングしましょう！

1. レプリカの数を増やしてサービスを更新します。

  `docker service` コマンドを使用して前に作成した Nginx サービスを更新し、サービスに 5 つのレプリカが含まれるようにします。次のコードで、サービスの新しい状態を定義します。

  ```sh
  $ docker service update --replicas=5 --detach=true nginx1
  nginx1
  ```

  このコマンドを実行すると同時に、次の処理が行われます。
  1. サービスの状態が更新されてレプリカの数が 5 つに変更されます (更新後の状態は、Swarm の内部ストレージに保管されます)。
  2. Docker Swarm は、現在スケジューリングされているレプリカの数が、レプリカ 5 つとして宣言された状態と一致しないことを認識します。
  3. 宣言されたサービスの状態と一致させるために、Docker Swarm がさらに 5 つのタスク (コンテナ) をスケジューリングします。

  Swarm は目的の状態が実際の状態と等しいかどうかをアクティブにチェックし、必要に応じて調整を試みます。

2. 実行中のインスタンスを確認します。

  数秒後、Swarm がその役目を果たして、さらに 9 つのコンテナを正常に起動したことを確認できます。クラスターを構成する 3 つのノードのすべてでコンテナがスケジューリングされていることに注目してください。新しいコンテナの実行場所を決定するために使用されるデフォルトの配置戦略は「emptiest node (最も空いているノード)」ですが、必要に応じて変更できます。

  ```sh
  $ docker service ps nginx1
  ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
  iu3ksewv7qf9        nginx1.1            nginx:1.12          node1               Running             Running 17 minutes ago
  lfz1bhl6v77r        nginx1.2            nginx:1.12          node2               Running             Running 6 minutes ago
  qururb043dwh        nginx1.3            nginx:1.12          node3               Running             Running 6 minutes ago
  q53jgeeq7y1x        nginx1.4            nginx:1.12          node3               Running             Running 6 minutes ago
  xj271k2829uz        nginx1.5            nginx:1.12          node1               Running             Running 7 minutes ago
  ```

3. 一連のリクエストを http://localhost:80 に送信します。

  `docker service update` を実行したときに `--publish 80:80` は変更していないため、このサービスではこの設定が引き続き有効になっています。けれども現時点でポート 80 に対してリクエストを送信すると、ルーティング・メッシュでリクエストのルーティング先となるコンテナは 1 つではなくなっています。ルーティング・メッシュはコンテナのロード・バランサーとして機能し、リクエストのルーティング先を順に交代させます。

  curl を数回実行して、この動作を試してみましょう。リクエストの送信先とするノードはどれでも構いません。リクエストを受信するノードとリクエストがルーティングされるノードの間には、何の関係もないからです。  

  ```sh
  $ curl localhost:80
  node3
  $ curl localhost:80
  node3
  $ curl localhost:80
  node2
  $ curl localhost:80
  node1
  $ curl localhost:80
  node1
  ```

  前に使用した巧みな `--mount` コマンドにより、どのノードがリクエストに対応しているかを確認できます。

> **ルーティング・メッシュの制限事項**
ルーティング・メッシュでは、ポート 80 上で 1 つのサービスしか公開できません。複数のサービスをポート 80 上で公開する必要がある場合は、外部アプリケーションとして Swarm 外部にあるロード・バランサーを使用することで、その目的を果たせます。

4. 集約されたサービス・ログを確認する

  リクエストのルーティング先となったノードを簡単に調べるもう 1 つの方法は、集約されたログを確認することです。集約されたサービス・ログを取得するには、`docker service logs [サービス名]` を使用します。このコマンドにより、実行中のすべてのコンテナからの出力、つまり `docker container logs [コンテナ名]` による出力が集約されます。

  ```sh
  $ docker service logs nginx1
  nginx1.4.q53jgeeq7y1x@node3    | 10.255.0.2 - - [28/Jun/2017:18:59:39 +0000] "GET / HTTP/1.1" 200 6 "-" "curl/7.52.1" "-"
  nginx1.2.lfz1bhl6v77r@node2    | 10.255.0.2 - - [28/Jun/2017:18:59:40 +0000] "GET / HTTP/1.1" 200 6 "-" "curl/7.52.1" "-"
  nginx1.5.xj271k2829uz@node1    | 10.255.0.2 - - [28/Jun/2017:18:59:41 +0000] "GET / HTTP/1.1" 200 6 "-" "curl/7.52.1" "-"
  nginx1.1.iu3ksewv7qf9@node1    | 10.255.0.2 - - [28/Jun/2017:18:50:23 +0000] "GET / HTTP/1.1" 200 6 "-" "curl/7.52.1" "-"
  nginx1.1.iu3ksewv7qf9@node1    | 10.255.0.2 - - [28/Jun/2017:18:59:41 +0000] "GET / HTTP/1.1" 200 6 "-" "curl/7.52.1" "-"
  nginx1.3.qururb043dwh@node3    | 10.255.0.2 - - [28/Jun/2017:18:59:38 +0000] "GET / HTTP/1.1" 200 6 "-" "curl/7.52.1" "-"
  ```

  これらのログを基に、各リクエストに対応したコンテナはそれぞれに異なることがわかります。

  リクエストが node1、node2、node3 のどれに送信されたかだけでなく、各ノード上のどのコンテナにリクエストが送信されたのかも確認できます。例えば、`nginx1.5` は、`docker service ps nginx1` の出力に示された名前と同じ名前のコンテナにリクエストが送信されたことを意味します。

## ステップ 4: ローリング・アップデート

サービスがデプロイされた状態になったので、アプリケーションのリリースについて具体的に見ていきましょう。これから、Nginx のバージョンを「1.13」に更新します。この更新を行うために、`docker service update` コマンドを使用します。

```sh
$ docker service update --image nginx:1.13 --detach=true nginx1
```

上記のコマンドにより、Swarm のローリング・アップデートがトリガーされます。`docker service ps nginx1` と素早く何度も入力すると、リアルタイムでの更新を確認できます。

ローリング・アップデートを微調整するためのオプションとしては、まず、
`--update-parallelism` を使用して、一度に更新するコンテナの数を指定できます (デフォルトは 1 です)。
`--update-delay` を使用すると、コンテナのセットの更新を完了した後、次のコンテナのセットで更新を続行するまで待機する時間を指定できます。

数秒経ってから、`docker service ps nginx1` を実行すると、nginx:1.13 に更新されたすべてのイメージを確認できます。

```sh
$ docker service ps nginx1
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE             ERROR               PORTS
di2hmdpw0j0z        nginx1.1            nginx:1.13          node1               Running             Running 50 seconds ago
iu3ksewv7qf9         \_ nginx1.1        nginx:1.12          node1               Shutdown            Shutdown 52 seconds ago
qsk6gw43fgfr        nginx1.2            nginx:1.13          node2               Running             Running 47 seconds ago
lfz1bhl6v77r         \_ nginx1.2        nginx:1.12          node2               Shutdown            Shutdown 49 seconds ago
r4429oql42z9        nginx1.3            nginx:1.13          node3               Running             Running 41 seconds ago
qururb043dwh         \_ nginx1.3        nginx:1.12          node3               Shutdown            Shutdown 43 seconds ago
jfkepz8tqy9g        nginx1.4            nginx:1.13          node2               Running             Running 44 seconds ago
q53jgeeq7y1x         \_ nginx1.4        nginx:1.12          node3               Shutdown            Shutdown 45 seconds ago
n15o01ouv2uf        nginx1.5            nginx:1.13          node3               Running             Running 39 seconds ago
xj271k2829uz         \_ nginx1.5        nginx:1.12          node1               Shutdown            Shutdown 40 seconds ago
```

アプリが nginx の最新バージョンに更新されました！

## ステップ 5: 調整

前のステップで、`docker service update` を使用してサービスの状態を更新し、Docker Swarm が目的の状態と実際の状態との不一致を認識すると、この問題を解決しようと試みる動作を確認しました。

Docker Swarm の「検査 -> 適応」モデルでは、エラーの発生時に Docker Swarm で調整を行うことができます。例えば、Swarm に含まれるノードが停止状態になると、そのノードで実行されているコンテナも一緒に停止してしまうでしょう。Swarm はコンテナの損失を認識すると、目的とされているサービスの状態を達成するために、利用可能なノード上でコンテナの再スケジューリングを試みます。

ノードを削除して、nginx1 サービスのタスクが他のノード上で自動的に再スケジューリングされることを確認しましょう。

1. 出力を簡潔にするために、まず、以下の行をコピーしてまったく新しいサービスを作成します。既存のサービスと競合しないよう、サービスの名前と公開ポートを変更します。また、サービスをスケーリングするために、`--replicas` コマンドを追加してサービスの数を 5 インスタンスに設定します。

  ```sh
  $ docker service create --detach=true --name nginx2 --replicas=5 --publish 81:80  --mount source=/etc/hostname,target=/usr/share/nginx/html/index.html,type=bind,ro nginx:1.12 aiqdh5n9fyacgvb2g82s412js
  ```

2. node1 上で `watch` を使用して、`docker service ps` の出力で更新状況を観察します。「watch」は Linux ユーティリティーであるため、他のプラットフォームでは利用できない場合があります。

  ```sh
  $ watch -n 1 docker service ps nginx2
  ```

  上記のコマンドにより、ウィンドウに次のような内容が表示されます。

  ```
  Every 1s: docker service ps nginx1 2                                                                                              2017-05-12 15:29:20

  ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
  6koehbhsfbi7         nginx2.1        nginx:1.12          node3               Running            Running 21 seconds ago
  dou2brjfr6lt        nginx2.2            nginx:1.12          node1               Running             Running 26 seconds ago
  8jc41tgwowph        nginx2.3            nginx:1.12          node2               Running             Running 27 seconds ago
  n5n8zryzg6g6        nginx2.4            nginx:1.12          node1               Running             Running 26 seconds ago
  cnofhk1v5bd8        nginx2.5            nginx:1.12          node2               Running             Running 27 seconds ago
  [node1] (loc
  ```

3. node3 をクリックし、Swarm クラスターから離脱させるためのコマンドを入力します。

  ```sh
  docker swarm leave
  ```

  Swarm からノードを離脱させるには、このコマンドが「便利」な方法になりますが、ノードをキルするという方法でも同じ動作になります。

4. node1 をクリックして、調整が行われる様子を観察します。Swarm が宣言された状態に戻すために、node3 上で実行されていたコンテナを自動的に node1 と node2 で再スケジューリングしようとする動作を確認できるはずです。

  ```sh
  $ docker service ps nginx2
  ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
  jeq4604k1v9k        nginx2.1            nginx:1.12          node1               Running             Running 5 seconds ago
  6koehbhsfbi7         \_ nginx2.1        nginx:1.12          node3               Shutdown            Running 21 seconds ago
  dou2brjfr6lt        nginx2.2            nginx:1.12          node1               Running             Running 26 seconds ago
  8jc41tgwowph        nginx2.3            nginx:1.12          node2               Running             Running 27 seconds ago
  n5n8zryzg6g6        nginx2.4            nginx:1.12          node1               Running             Running 26 seconds ago
  cnofhk1v5bd8        nginx2.5            nginx:1.12          node2               Running             Running 27 seconds ago
  [node1] (loc
  ```

## 必要なノードの数は？

このラボでは、1 つのマスター・ノードと 2 つのワーカー・ノードからなる Docker Swarm クラスターを構成しました。この構成は、それほど可用性が高いとは言えません。マネージャー・ノードにはクラスターを管理するために必要な情報が含まれているため、このノードが停止した場合、クラスターは機能しなくなります。本番アプリケーションには、マネージャー・ノードの障害を許容できるよう、複数のマネージャー・ノードを含むクラスターをプロビジョニングする必要があります。

少なくとも 3 つのマネージャー・ノードが必要ですが、通常は 7 つを超える数のマネージャー・ノードは必要ありません。マネージャーは Raft という分散合意アルゴリズムを実装します。このアルゴリズムでは、保管されるクラスターの状態に対してノードの過半数が同意する必要があります。同意するノードが過半数に満たない場合、Swarm は正常に動作しなくなります。このことから、ノードの耐障害性は次のように前提できます。

- マネージャー・ノードが 3 つある場合、そのうち 1 つのノード障害が許容されます。
- マネージャー・ノードが 5 つある場合、そのうち 2 つのノード障害が許容されます。
- マネージャー・ノードが 7 つある場合、そのうち 3 つのノード障害が許容されます。

マネージャー・ノードの数を偶数にすることもできますが、ノード障害の数という点では何の価値ももたらしません。例えば、マネージャー・ノードが 4 つある場合でも、許容されるノード障害は 1 つだけです。これは、3 つのマネージャー・ノードが含まれるクラスターでの耐障害性と変わりありません。また、マネージャー・ノードの数が多くなればなるほど、クラスターの状態について合意に達するのが難しくなります。

一般に、マネージャー・ノードの数は 7 つ以下に制限する必要がありますが、ワーカー・ノードの数はそれよりも遥かに多くすることができます。ワーカー・ノードは、数千個にまでスケールアップできます。ワーカー・ノードはゴシップ・プロトコルを使用して通信します。このプロトコルは、大量のトラフィックでも多数のノードでも、極めて効率よく動作するように最適化されています。

http://play-with-docker.com を使用している場合は、組み込みテンプレートを使用して複数のマネージャー・ノードを含むクラスターを簡単にデプロイできます。左上にあるテンプレート・アイコンをクリックすると、利用可能なテンプレートを調べることができます。

## まとめ

このラボでは、本番でコンテナを実行する際に伴う、分散された複数のノードをまたがるサービスのスケジューリング、高可用性の維持、調整の実装、スケーリング、ロギングなどの問題について説明しました。こうした問題に対処するために、Docker Engine に組み込まれている Docker Swarm というオーケストレーション・ツールを使用しました。

このラボで学んだ重要な点
- Docker Swarm では、宣言型言語を使用してサービスをスケジューリングします。状態を宣言すると、Swarm は実際の状態が目的の状態と同じになるように調整を試みます。
- Docker Swarm はマネージャー・ノードとワーカー・ノードで構成されます。Swarm の状態を維持できるのは、マネージャー・ノードのみです。また、マネージャー・ノードだけが、状態を変更するコマンドを受け入れることができます。ワーカーは極めてスケーラブルであり、コンテナを実行するためにのみ使用されます。デフォルトでは、マネージャーがコンテナを実行することもできます。
- Swarm にはルーティング・メッシュ機能が組み込まれているため、サービス・レベルで公開されている任意のポートは Swarm に含まれるすべてのノードで公開されます。公開されているサービス・ポートに送信されたリクエストは、Sawrm で実行されているサービスのコンテナに自動的にルーティングされます。
- コンテナ化されたアプリケーションを本番環境でオーケストレーションするという問題を解決するには、Docker Swarm や [IBM Cloud Kubernetes Service](https://www.ibm.com/jp-ja/cloud/container-service) ツールをはじめ、多数のツールを利用できます。
