---
title: "SafeLine WAF 導入前の Docker Compose と ssse3 確認ガイド"
emoji: "🔒"
type: "tech"
topics: ["waf", "security", "linux"]
published: true
---

SafeLine WAF を自前のサーバに載せる場合、コンテナを作成しただけでは保護は始まりません。まずホストが公開済みの前提条件を満たしているかを確認します。SafeLine は Web アプリケーションの手前にリバースプロキシとして配置され、HTTP トラフィックを監視・フィルタする WAF です。導入ガイド(https://docs.waf.chaitin.com/en/home)に記載された要件は、Linux、x86_64 または arm64、Docker 20.10.14 以上、Docker Compose 2.0.0 以上、CPU 1 コア以上、メモリ 1 GB 以上、ディスク 5 GB 以上、ssse3 対応 CPU です。

Docker Compose と ssse3 の確認は、コンテナ作成後の後始末ではなく、導入前の最初のゲートです。以下、実際にホストで実行する手順に沿って説明します。

## 1. 公開された確認コマンドを順に実行する

導入先ホストのシェルで、次のコマンドを実行します。

```bash
uname -m
docker version
docker compose version
cat /proc/cpuinfo | grep "processor"
free -h
df -h
lscpu | grep ssse3
```

それぞれの出力は要件表の行と一対一で対応します。uname -m は命令セット、docker version は Docker 本体、docker compose version は Compose、processor は CPU コア、free -h はメモリ、df -h はディスク、lscpu | grep ssse3 は CPU 命令の確認です。別の資産棚卸しスクリプトに置き換えず、このコマンドで確認してください。

## 2. Docker と Docker Compose を先に確認する

Docker は 20.10.14 以上、Docker Compose は 2.0.0 以上が条件です。Compose はスペース入りの docker compose version で確認します。コマンド自体がない場合、または表示されたバージョンが下限より古い場合は、その場で作業を止めます。docker version が下限より古い場合も同様です。

自動導入スクリプトも手動導入も、この下限が満たされている前提で進みます。新しい VM では先にバージョン確認を行い、不足していれば Docker と Compose の更新を済ませてから次へ進みます。

## 3. アーキテクチャと ssse3 を別々に扱う

対応する命令セットは x86_64 と arm64 です。uname -m がこれ以外を返したら、公開された対応表の外です。コンテナイメージが差を吸収してくれることを期待して compose ファイルの取得や起動に進まないでください。

ssse3 は別の行です。lscpu | grep ssse3 で該当行が出れば、そのフラグが lscpu から見えていることを確認できます。何も返らない場合は要件未達です。導入ガイドには免除、ソフトウェア代替、Compose の設定による回避策は記載されていません。どちらの結果も出力を保存してください。

arm64 だからといって ssse3 の確認を省略できません。手動で ARM に導入する場合は、ARCH_SUFFIX=-arm と POSTGRES_IMAGE=chaitin/safeline-postgres-arm:15.18 を設定します。ARM 上では Pro ライセンスが必要で、Personal Edition はサポートされません。まず uname -m を確認し、その後に記載された ARM 向け設定だけを適用します。arm64 のホストを既定の x86 イメージ名に向けないでください。

## 4. 容量を公表された下限と比較する

最小構成は CPU 1 コア、メモリ 1 GB、ディスク 5 GB です。cat /proc/cpuinfo | grep "processor" でプロセッサ、free -h でメモリ、df -h でファイルシステム容量を確認します。手動導入では SafeLine が書き込むディレクトリに 5 GB 以上が必要と繰り返し示されています。

下限を下回る場合は、容量を追加するか別のホストを選びます。想定コア数や想定メモリ量を勝手に上乗せしないでください。コンテナの起動が不足したメモリやディスクを補う手段として記載されているわけでもありません。

## 5. 失敗例

新規 VM で docker compose version が 1.29 を返していたのに、先に自動導入スクリプトを実行したケースがあります。イメージ取得の途中で Compose プラグインの差に起因するエラーが発生し、作業がどの段階まで進んだか分かりにくくなりました。また、lscpu | grep ssse3 が何も返さない旧い仮想化基盤では、コンテナを立ち上げてから初めて CPU フラグの未達に気づくことになります。事前にコマンド出力を控えておけば、このような手戻りを避けられます。

## 6. ゲートを通過してから導入する

アーキテクチャ、Docker、Compose、CPU、メモリ、ディスク、ssse3 をすべて確認できたら、導入パスを選びます。自動導入は推奨されており root が必要です。手動導入は最後に docker compose up -d を実行します。MGT_PORT を既定の 9443 のままにした場合、コンソールは https://<ip>:9443/ で開きます。初期管理者アカウントは docker exec safeline-mgt resetadmin で確認します。

アプリケーション保護はコンソール上の別の作業です。ドメイン、待受ポート、アップストリームを指定して追加し、そのドメインを SafeLine サーバへ向けます。ただし、これらの作業が Compose 2.0.0 や ssse3 を検証してくれるわけではありません。先にホスト側の確認を済ませ、コマンド出力を保存してから、インストーラまたは compose ファイルを開いてください。


---

- Demo: https://demo.waf.chaitin.com:9443/statistics
- Official docs: https://docs.waf.chaitin.com/en/home
- GitHub: https://github.com/chaitin/SafeLine
