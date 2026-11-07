---
title: "SafeLine WAF 管理コンソールをプロキシ配下に置く運用ガイド"
emoji: "🔒"
type: "tech"
topics: ["waf", "security", "linux"]
published: true
---

SafeLine WAF は、保護対象となる Web アプリケーションの手前にリバースプロキシとして立ち、クライアントからの HTTP トラフィックを検査・監視するセルフホスト型の WAF です。ここで重要なのは、管理コンソールと保護対象サイトは別の入口だという点です。管理コンソールは https://<safeline-ip>:9443/ で開き、.env の MGT_PORT がそのポートを、MGT_PROXY がコンソール前段のプロキシ層数を表します。

本記事では、管理コンソールを追加のリバースプロキシ配下に公開する際の実務手順を整理します。公開ドキュメントには外部プロキシ製品向けの具体的な設定は含まれていないため、本手順は「層数を正しく記録する」ことと「ドキュメントに存在しない構文を持ち込まない」ことを軸にします。

事前確認
導入前にホストが公開要件を満たすか確認します。Linux、x86_64 または arm64、Docker 20.10.14 以上、Docker Compose 2.0.0 以上が必要です。最小構成は CPU 1 コア、メモリ 1 GB、ディスク 5 GB で、CPU は ssse3 をサポートする必要があります。
- uname -m でアーキテクチャを確認
- docker version / docker compose version でバージョンを確認
- cat /proc/cpuinfo と lscpu で ssse3 の有無を確認
- free -h / df -h でメモリとディスクを確認
ARM の場合は Pro ライセンスが必要で、Personal Edition は ARM 非対応とされています。

手順
1. ホストを公開要件に合わせる
上記コマンドで Linux の種別、x86_64 または arm64、Docker と Compose のバージョン、ssse3、メモリ、ディスクを確認します。ARM なら Pro ライセンス要否を先に判断します。

2. 導入方式を選ぶ
自動導入コマンド bash -c "$(curl -fsSLk https://waf.chaitin.com/release/latest/manager.sh)" -- --en は MGT_PROXY を設定しません。コンソールのプロキシ層数を明示的に記録する必要がある初回導入では、.env を直接書ける手動手順を選びます。

3. SafeLine ディレクトリを作成し compose ファイルを取得する
mkdir -p "/data/safeline" を実行し、/data/safeline へ移動して wget "https://waf.chaitin.com/release/latest/compose.yaml" を実行します。このディレクトリに設定とデータが書き込まれ、5 GB 以上のディスクが必要です。

4. 起動前に .env を作成する
/data/safeline/.env に、SAFELINE_DIR=/data/safeline、IMAGE_TAG=latest、MGT_PORT=9443、任意の POSTGRES_PASSWORD、SUBNET_PREFIX=172.22.222、IMAGE_PREFIX=chaitin、非 ARM 例では ARCH_SUFFIX 空欄、RELEASE 空欄（LTS を選ぶ場合のみ -lts）、REGION=-g、MGT_PROXY=0 を書きます。あわせて Postgres イメージ名と detector 用のトークン欄も設定します。DETECTOR_MGMT_TOKEN と MGT_INTERNAL_TOKEN は /dev/urandom を使った od で 32 バイトの 16 進値として生成し、DETECTOR_AES_KEY は 16 文字の英数字にします。書き込み後 chmod 600 .env を実行します。ARM では ARCH_SUFFIX=-arm とし、Postgres イメージも ARM 用にします。RELEASE=-lts を選ぶ場合、9.1.0-LTS（2025 年 8 月 14 日）以降 LTS は保守されず、最新版へのアップグレードが強く推奨される点に注意します。

5. コンソールのプロキシ層数は実際に置く場合だけ設定する
コンソールの前段にプロキシを置かないなら MGT_PROXY は 0 のままにします。置く場合はその層数を整数で設定します。この値を攻撃検知やボット対策、サイトのレート制限に流用してはいけません。ドキュメント上、MGT_PROXY はコンソールのプロキシ層数としてのみ定義されています。

6. ディレクトリ準備を仕上げて起動する
/data/safeline/resources/run を作成し、run、mgt、sock、postgres データディレクトリに chmod 700、.env に chmod 600 を適用します。/data/safeline/resources/run/safeline.sock を /var/run/safeline.sock へリンクし、SafeLine ディレクトリから docker compose up -d を実行します。完了まで数分かかることがあります。

7. 直接アクセスで管理者を作成する
https://<safeline-ip>:9443/ にブラウザでアクセスし、ホスト上で docker exec safeline-mgt resetadmin を実行します。成功すると初期ユーザー名 admin と初期パスワードが表示されるため、両方を控えてログインします。この直接 URL がドキュメントに記載された起動確認です。

8. サイト保護はアプリケーション経路で行う
管理コンソールにログインし、Applications から Add Application を選びます。元のドメイン・ホスト名・IP、待受ポート（80 や 443）、実アプリの upstream アドレスを入力します。HTTPS なら SSL を確認し、ドメインは SafeLine サーバの IP に解決します。SafeLine がサイトのトラフィックを受け、悪意あるリクエストを除去して残りを転送します。これは MGT_PROXY とは別のプロキシ役割です。

9. 前段プロキシの前にチェックリストを回す
数えるのは「管理コンソールの手前に置いた層」だけです。保護対象アプリへ公開トラフィックを送るプロキシはこの数に含めません。フィールドは層数なので、整数を設定します。直接ログインが成功してから、後段のプロキシの宛先をホストと MGT_PORT に向けます。ドキュメントには転送ヘッダー名やコンソールのヘルスパス、upstream 設定例は公開されていないため、これらを製品仕様として扱いません。認証チャレンジ、CAPTCHA、レート制限、動的な HTML/JavaScript 暗号化は Web アプリ向けに紹介された機能であり、MGT_PROXY を編集してもコンソールで有効になるとはドキュメントに書かれていません。アンインストール時は SafeLine ディレクトリで docker compose down を実行し、ディレクトリを削除します。外側のコンソール用プロキシは、それを管理する側のシステムで停止・削除します。

失敗例
MGT_PROXY を「サイト保護の有効化」と誤解して 1 に変更し、実際にはコンソール前段にプロキシを置かずに https://<safeline-ip>:9443/ へアクセスしたケースを考えます。この場合、直接アクセスそのものは設定したポートに依存するため、まず直接 URL の疎通と docker exec safeline-mgt resetadmin の出力を確認します。層数を記録しているだけで、外部プロキシの導入や設定が自動で行われるわけではありません。外側のプロキシを構成する作業はドキュメントに含まれないため、独自に用意する必要があります。問題を切り分ける際は、コンソール公開（Web UI・MGT_PORT・MGT_PROXY）とサイト保護（リバースプロキシのアプリケーション設定）を分けて確認してください。

要点
- MGT_PROXY はコンソール前段のプロキシ層数。0 は「プロキシ未構成」の既定値。実際に置く場合のみ層数に変更。
- MGT_PROXY は外部プロキシを導入・設定するものではない。外部プロキシの構築は別途自身で行う。
- コンソールを Add Application に追加する手順はドキュメントにない。Add Application は元の Web アプリ用。
- プロキシの前に成功すべき URL は https://<safeline-ip>:9443/（設定した MGT_PORT に応じた形式）。
- 証明書レイアウト、クライアント IP ヘッダー、タイムアウト、プロキシ製品はドキュメントで定義されていないため、この手順では約束しない。


---

- Demo: https://demo.waf.chaitin.com:9443/statistics
- Official docs: https://docs.waf.chaitin.com/en/home
- GitHub: https://github.com/chaitin/SafeLine
