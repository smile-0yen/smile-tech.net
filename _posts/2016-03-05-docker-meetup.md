---
layout: post
title: Docker Meetup Tokyo#6へ行ってきた
---

## Docker Meetup Tokyo
運良く抽選に当たり[Docker Meetup Tokyo #6](http://dockerjp.connpass.com/event/26538/)へ参加して来ました。 #5にも参加したので２回目。
Dockerも1.10になり、プロダクションで使う方法が容易に想像できるようになってきました。

## Opening + Docker 1.10の新機能
Speaker: [@ianmlewis](https://twitter.com/ianmlewis)  
Developer Advocate - Google Cloud Platform

### The Docker mission
DockerのミッションはBuild - Ship - Runをどこでもできるようにすること。

### Docker 1.10 Launch

- composeなどの周辺アプリケーションもバージョンアップ。
- **新機能**
  - Docker compose が今回バージョンアップしてservicesに加えて volumes,networkが柔軟に定義できるようになった。
  - Docker Swarmもより安定した。
  - Securityによりコンテナがどのsystem callを呼べるかまで制御できるようになった。
  - オーケストレーションでは内部DNSでコンテナの名前解決ができるようになるなど機能が追加された。

##  Docker活用パターンの整理：Docker/OpenStack/Ansible/Kubernetes/OpenShift/etc... どう組み合わせるのが正解？

Speaker: [@enakai00](https://twitter.com/enakai00)

### Dockerが提供する基本機能
- コンテナを使うことが目的では**ない**
- アプリケーションをポータブルにしてどこでも実行できるようにするもの。

### 本番環境でやると嬉しいこと
- デプロイが安全に行える(Chef,Puppetと比較して)。＊安全＝開発済で確実に動く環境を配布できる。
- その観点で考えるとIaaS基盤とDockerを組み合わせると良いと考えられる。
- Chef,Puppetは設定手順をファイルに落としてるだけ。普及に伴って設定のメンテナンスができなくなって困ってしまうということをよく聞くようになった。
- DockerではChefのような心配はない。
- DockerはOpenStack、Ansibleと組み合わせて使うのは相性が良さそう。
- AnsibleでもChefのようなことができるが、そうではなくあくまでOpenStackAPIを叩くようにする。そのように自動化の住み分けを行うと良い。
- ポイントは1仮想マシン、1コンテナ。単純にデプロイを安全、簡単にするということを目的にする。
- 同じマシンに複数コンテナを入れるとコンテナ間の運用監視には独自のノウハウが必要になってしまう。それが導入の壁になる。  
参考情報) http://jp-redhat.com/openeye_online/column/nakai/3077/

### 複数ホストのDockerを管理する仕組み
- GoogleがBorgを開発したのは1台1台のマシンを意識したくないため。
- Dockerを用いて複数のホストを束ねて1つのコンピュータとして活用する
  - 眺めていると1つのマシンの中に複数プロセスがあるのと同じものに見えてくる。
  - 最近の流行り言葉で言うとマイクロサービス。この言葉を最初に聞いた時はどういう単位でサービスを設計すれば良いのか疑問を持っていた。
  - 1コンテナは1プロセスに相当するのだという観点で考えると見えてくる。

### 今後
- 機能単位でコンテナを入れ替えることにより稼働中のアプリケーションの動的な機能変更も可能になる。
- 例えばオンラインゲームのアプリであればフロントエンドはマイクロサービス化してアップデートしていく。
- 一方でバックエンドで使っている課金機能などは頻繁なバージョンアップは不要。
- 以下のような使い分けが良いのではないか。
  - 頻繁なバージョンアップをする部分はベアメタル状のコンテナプール。
  - 基幹システムは1VM1コンテナ

## DockerとPodの概念。様々なコンテナ環境のあり方

Speaker: [@ianmlewis](https://twitter.com/IanMLewis)

### What are containers?
- DockerはLinuxの2つの機能をラッピングしている
  - Linux cgroup
  - Linux Namespace(IPC, Network, Mount PID, User, UTS)
- Container Image = Metadata + File System

### Volumes
- 複数のコンテナでデータを連動したい場合=> Volumeを使えば良い
- ClusterでVolumeを使うときは？
 - 1つのクラスタ上にデータを共有するコンテナを寄せるべき？
 - 1つのクラスタに寄せた上でNFSで共有すべき？
 - NASに入れてどこからでもアクセスできるようにするべき？
- nginx confの場合は？
 - 通常はetcdでIPを管理して変更があればそれをconfdが見てnginxの設定ファイルを書き換えた上でnginxをHUPするような設計にする。
 - このような場合を全てコンテナで行うときにどういったクラスタ構成、コンテナ設計にすべきかは自明ではない。
 - よくある解決策は1つのContainer内にconfd,nginxをまとめる方法。
 - Dockerにはエントリポイントが1つしかないのでforeman(or superviserd)を用意する。結構Bashスクリプトでごりごりやってる人とかもいる。
 - しかしこの方法には問題がある。例えばnginxがconfの異常でCrashループに陥っていても、エントリポイントのforemanが動いていればDockerは全てが正常だと判断してしまう。

### Pods & Docker
- Googleは10年以上という長いコンテナの運用経験がある。
- Kubernetes
 - Kubernetesでは同じサービスを提供するコンテナをPodという単位で管理する。Pod内のコンテナはお互いを見ることができる。
 - 共通のNamespaceで動かすためにKubernetesのschedulerは同じPodのコンテナを同じサーバへ割り当てる。
 - 複数のPodsを管理するReplication Controllerの存在によってスケールが簡単にできるようになっている。

## Docker Usercase in Production(using CoreOS and Docker compose)
Speaker: [@tn961ir](https://twitter.com/tn961ir)

### 開発上の課題
- Cost,Ops,QA & Security
- 開発プロセス
- マルチクラウド対応(AWS, GCP, CloudStack, OpenStack, etc.)

### Cost,Ops問題
- インフラエンジニアが1人もいなかった。GCP利用で解決。
- 自社事例）GCPが支えるクラウドソリューション  
[http://googlecloudplatform-japan.blogspot.jp/2015/04/google-cloud-platform-o2o.html](http://googlecloudplatform-japan.blogspot.jp/2015/04/google-cloud-platform-o2o.html)

### 開発
- GitLabを採用
 - 開発からデプロイまでを一気通貫に。
 - GitLabCI(Docker利用)を回してSlackに通知。
 - 同社エンジニアの記事"GitLab CIでXcodeプロジェクトをCIする手順"  [http://qiita.com/enmtknt/items/83935c1137203e1c2c0f](http://qiita.com/enmtknt/items/83935c1137203e1c2c0f)

### OS/Platform
- footprintの小さいOSとしてCoreOSを採用
- ただしcluster, etcd, fleetなどはOFF
- 代わりにdocker-compose + Ansibleで頑張っている。
- イメージの管理はGoogle Container Repositoryを使っている。
- 1VM1Containerのプロジェクトと1VM2Containerの2つのPJがある。それ以上複雑にはしないようにしている。
- 監視はGoogle Cloud Monitoring + Munin.

### 今後
- インフラエンジニアがいないのでServerless architechtureに移行していきたい。
- AWS LambdaもあるがGCPを使ってきたしGoogle Cloud functions(alpha)に移行したい。

## Microsoft AzureでのDocker対応状況の紹介

Speaker: [@yuya_lush](https://twitter.com/yuya_lush)  

### 手軽にDockerを始めたい方へ

- AzureにはDocker on Ubuntu Serverというイメージが用意されている。
- OSはCanonical、追加部分(Docker Extension)をMSがメンテナンスしてます。  
[https://github.com/Azure/azure-docker-extension](https://github.com/Azure/azure-docker-extension)
- dockerは最新のstableが入るようになっているので今なら1.10が入る。

### Docker + Azure ちゃんと使いたい人向け
- 上の方法はプロダクションには向かない。
- quickstartというものが用意されている(他にもswarm用のテンプレなどが用意されている)  
[https://github.com/Azure/azure-quickstart-templates/tree/master/docker-simple-on-ubuntu](https://github.com/Azure/azure-quickstart-templates/tree/master/docker-simple-on-ubuntu)
- テンプレートからの自動作成
  - 一番基本的な構成でデプロイしてくれる
  - 後からセキュリティ系リソースを追加可能(defaultは全開放)
  - IaaS v2のデプロイなので最新機能を使える

### がっちりDockerで環境を作りたい。
- Azure Container Service - Swarm
  - Ross Gardler(Program Manager, Azure)が責任者。
- Docker Swarm が Apache Mesos を使ったクラスター構成を作ってくれる。
- Trusted RegistryもBYOLで利用可能。
- Visual Studio OnlineでGit + イメージ作成が可能。

### おまけ
元MSで現Docker社のPatrickさんが作ったチュートリアル  
[https://github.com/chanezon/azure-linux](https://github.com/chanezon/azure-linux)

## Docker Best Practice

Speaker: [@spesnova](https://twitter.com/spesnova)

### 個人的なベストプラクティスを紹介する。
- 基本原則
 - コンテナは短命なものとして扱う
   - 頻繁に起動してはすぐに終了する。
   - スケール、入れ替え、移動などどれもやりやすくなる。
 - できるだけコンテナでやる
   - ホストに直接インストールしない。ホストの運用が楽になる。

### プラクティス1) コンテナはGracefulに止めよう
- コンテナはライフサイクルが短く、頻繁に起動/停止される。
 - 問題が起きないようにGracefulに止める。  
 `docker kill --signal=<SIGNAL> container`
- `CMD nginx`(シェルがエントリポイントになるのでシェルにシグナルが送られる)ではなく`CMD["nginx"]`にする。

### プラクティス2) すべてコンテナでやる
- 1コンテナでは1つのことだけ行う。
 - Dockup(バックアップだけをやるコンテナ)
 - Datadog
 - Fluentd

### プラクティス3) 設定は環境変数に入れる
- 実行時に設定したいものをハードコードしてるとビルドし直す必要が出る。
- ミドルウェア等に依存しないように設定値を渡せる。

### プラクティス4) どうやってホストマシンをメンテナンスするか考えておく
- ホストマシンの運用を忘れずに
- 1度コンテナの運用を始めたらホスト情報が圧倒的にメンテナンスしづらい
- kubernetes Drainというのを使うとホスト上のコンテナを別ホストに全部移すことができる。

### プラクティス5) 脆弱性を継続的にスキャンする
- 再現性が高いゆえに、更新頻度が下がり脆弱性対応も放置されがち。
- Clairを使う。
- Quay.io脆弱性対応をしたらSlack通知してくれる

### プラクティス6) 公式のセキュリティベンチマークを流す
- Dockerは周辺ツールも含めて急速に進化しており何かと選択肢が沢山ありすぎる。
- 機械的にベスト・プラクティスをチェックするようにしておく。

## リンク
- [ぼへぼへさんがまとめた当日資料](http://bohelabo.jp/docker-meetup-6/)
