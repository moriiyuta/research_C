# CircleCI検証用リポジトリ


検証ソースは下記リポジトリよりクローン  
https://github.com/W1LLY/spring-boot-postgresql-hibernate-thymeleaf-gradle-example.git


## 【前提事項】  
・JavaによるWebアプリケーションを対象とする

・開発フレームワークは「SpringBoot」を利用

・ビルドツールは「Gradle」を利用

・gradleのcheckstylプラグインを利用する

・gradleのcom.github.spotbugsプラグインを利用する

・gradleのorg.sonarqubeプラグインを利用する

・gradleのmavenプラグインを利用する

・mavenのuploadArchivesタスクを利用する

・ビルドツールは「Gradle」を利用

・ビルド成果物は単独のjarファイル

・APサーバ上では、"java -jar"コマンドでjarを実行する処理をShellに記載し、  
　サービス登録することでWebアプリケーションが動いている


## 【CircleCI設定ファイルの基本】
①初めに利用するCircleCIのバージョンを指定する。

②"jobs:"を記述して「パイプラインブロック」を作成する。  
　※パイプライン上の各処理は「パイプラインブロック」の中に作成する。

③「パイプラインブロック」の中に処理単位ごとに「処理ブロック」を作成する。

④処理ブロックの中に"docker"として「実行環境ブロック」を指定する。  
　※実行環境はDockerイメージを指定する

⑤処理ブロックの中に"steps:"として「処理実行ブロック」を記述する。  
　※処理実行ブロックに実際に実行する処理を記述していく。  
　※Werckerと異なり、対象コードのチェックアウトは手動（"- checkout"記述）で行う必要がある点を留意。

⑥"workflows:"を記述して「ワークフローブロック」を作成する。

⑦「ワークフローブロック」内に、"workflow:"を記述して「フローブロック」を作成する。  
　※「フローブロック」内に、各処理の実行順序・処理対象ブランチを含む種々の依存関係を記述する。

⑧"workflows:"を記述して「ワークフローブロック」を作成する。

⑨「ワークフローブロック」内に、"nightly:"を記述して「定時実行フローブロック」を作成する。  
　※「定時実行フローブロック」内に、cron設定を含む種々の依存関係を記述する。


### 基本系
#CircleCIのバージョン指定  
version: 2.1  

#以下パイプラインの記述  
jobs:  
　build:  
 　　#実行環境  
 　　docker:  
 　　　- image: Dockerイメージ  
       
 　　#以下、処理「build」の実処理  
 　　steps:  
 　　　- checkout ※対象コードチェックアウト  
 　　　- 処理②...  
　deploy:  
 　　docker:  
 　　　- image: Dockerイメージ  
 　　steps:  
 　　　- checkout  
 　　　- 処理②...  

#以下ワークフローの記載  
workflows:  
　version: 2  
　workflow:  
　　jobs:  
　　　- build:  
  　　　　filters:  
  　　　　　branches:  
  　　　　　　only:  
  　　　　　　　- 対象とするブランチ名  
　　　　　post-steps:  
　　　　　　- 定常実行処理  
　　　- deploy:  
  　　　　requires: ※先行処理  
　　　　　　- build  

#以下、定時実行ワークフローの記載  
　nightly:  
　　triggers:  
　　　- schedule:  
　　　　　cron: "25 * * * *"  
　　　　　　filters:  
　　　　　　　branches:  
　　　　　　　　only:  
  　　　　　　　　- master  
　　jobs:  
　　　- build:  
　　　　　filters:  
　　　　　　branches:  
　　　　　　　only:  
　　　　　　　　- /^issue\/.+/  
　　　- deploy:  
　　　　　requires:  
　　　　　　- build  


# 【このymlファイル記述でやっている処理】
①build			：GitHubから連携された対象コードに対して、"./gradlew build"コマンドでビルド・静的検証・Junitテストを行う

②sonerqube		："./gradlew sonarqube"コマンドでJunitテスト結果をSonerQubeに連携しSonerQubeで指定した静的検証を行う

③nexus-snapshot	："./gradlew upload"コマンドでNexusのMaven2（snapshot）リポジトリにビルド資材を登録する。

④nexus-release	："./gradlew upload"コマンドでNexusのMaven2（release）リポジトリにビルド資材を登録する。

⑤deploy			：APサーバにビルド成果物のjarファイルをscp転送して、jar起動Shellをsystemctlコマンドで再起動する。


# 【実運用における設定変更箇所】
①環境変数※CircleCIサイトの⚙＞BUILD SETTINGS>Enviroment Variablesで設定する。  
・SONAR_HOST_URL　　　　：連携先SonerQubeのURL  
・SONAR_JDBC_URL　　　　：連携先SonerQubeの接続先JDBCのURL  
・SONAR_JDBC_DRIVER　　 ：連携先SonerQubeが利用するJDBCのドライバクラス名  
・SONAR_JDBC_USERNAME　 ：連携先SonerQubeが利用するDBのユーザ名  
・SONAR_JDBC_PASSWORD　 ：連携先SonerQubeが利用するDBのパスワード  
・NEXUS_URL_SNAPSHOT　　：連携先Nexusのリポジトリ（SNAPSHOT）のURL  
・NEXUS_URL_RELEASE　　 ：連携先Nexusのリポジトリ（RELEASE）のURL  
・NEXUS_USERNAME　　　　：連携先Nexusのユーザ名  
・NEXUS_USERPASSWORD　　：連携先Nexusのパスワード  
・PKEY_PRIVATE　　　　　 ：連携先ECSの秘密鍵に対する公開鍵  
　　　　　　　　　　　　　　※下記サイトを参考にESC上で作成する  
　　　　　　　　　　　　　　https://webkaru.net/linux/ssh-keygen-command/  
・STAGING_SERVER_IP　　　：連携先ECSのIPアドレス  
・FINGERPRINT　　　　　　：連携先ECSにSSHするためのフィンガープリント  

②定義ファイル内の記載の変更  
・RESOURCE  
　⇒ECS上のHOMEディレクトリ直下のjarファイル格納ディレクトリ名なので適宜置き換える。

・alibaba_research.sh  
　⇒jarファイルを起動するShellファイル名なので適宜名称を変更する。

・BK_SHELL  
　⇒ECS上のHOMEディレクトリ直下のShellファイルBK格納ディレクトリ名なので適宜置き換える。
