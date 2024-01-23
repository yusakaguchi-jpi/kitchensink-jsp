# Local事前作業
cd ~/work # 任意の作業用ディレクトリを作成  
cp -a $QUICKSTART_HOME/kitchensink-jsp .  
cd kitchensink-jsp  
cp $QUICKSTART_HOME/pom.xml pom-parent.xml  
vi pom.xml # "../pom.xml"を"pom-parent.xml"に書き換える  
不要なファイルの削除  
rm README.html  
rm crw-java8-maven-eap.yaml  
rm crw-java11-maven-eap.yaml  
別のターミナルでEAPを起動しておく。  
cd $JBOSS_HOME  
./bin/standalone.sh  
プロジェクト内のディレクトリで mvn コマンドによりビルドとデプロイを行う。  
mvn package # 下記の行だけでもビルドは行われる  
mvn wildfly:deploy  
http://localhost:8080/kitchensink-jsp/ にブラウザでアクセスしアプリを操作してみる。  
コンテナの起動  
Docker Hubにある 公式イメージ を利用する。  
使い方もこのページに書いてある。  
docker run -d --name mypgserver -p 5432:5432 \  
-e POSTGRES_USER=pguser -e POSTGRES_PASSWORD=pgpassword -e POSTGRES_DB=pgdatabase \  
docker.io/library/postgres:13  

JDBCドライバのインストール  
# Maven Centralより ~/.m2 以下にダウンロード  
mvn dependency:get -Dartifact=org.postgresql:postgresql:42.7.1  
既存のデータソースをいったん削除するためにアンデプロイする。  
cd ~/work/kitchensink-jsp  
mvn wildfly:undeploy # 一旦アンデプロイする  
one@localhost:9990 /] /subsystem=datasources/jdbc-driver=postgresql:add( \  
driver-name=postgresql,driver-module-name=org.postgresql, \  
driver-xa-datasource-class-name=org.postgresql.xa.PGXADataSource)  
# データソースの登録（JNDI名、ユーザ名やパスワード等を実際のものに合わせること）  
[standalone@localhost:9990 /] xa-data-source add --name=KitchensinkJSPQuickstartDS \  
--jndi-name=java:jboss/datasources/KitchensinkJSPQuickstartDS --driver-name=postgresql \  
--user-name=pguser --password=pgpassword --validate-on-match=true --background-validation=false \  
--valid-connection-checker-class-name=org.jboss.jca.adapters.jdbc.extensions.postgres.PostgreSQLValidConnectionChecker \  
--exception-sorter-class-name=org.jboss.jca.adapters.jdbc.extensions.postgres.PostgreSQLExceptionSorter \  
--xa-datasource-properties={"ServerName"=>"localhost","PortNumber"=>"5432","DatabaseName"=>"pgdatabase"}  
外部DBを利用する形で起動する  
cd ~/work/kitchensink-jsp  
# 内臓DBを使ったデータソースの定義を削除する  
rm src/main/webapp/WEB-INF/kitchensink-quickstart-ds.xml  
# 一旦cleanして再デプロイ  
mvn clean wildfly:deploy  
http://localhost:8080/kitchensink-jsp/ にアクセスし試しにデータを入力してみる  
OpenShiftにログインし自分のプロジェクト(以降 $MYPROJ で参照)を作成する。  
oc login -u <自分のユーザ名> https://api.<OpenShiftクラスタ>:6443  
oc new-project <一意なプロジェクト名>  
# プロジェクト名を変数MYPROJに保存  
MYPROJ=$(oc project -q)  
source <(oc completion bash)  
oc status  
oc whoami  
EAP公式のImageStreamとTemplateのインポート  
# ImageStreamのインポート  
oc apply -f https://raw.githubusercontent.com/jboss-container-images/jboss-eap-openshift-templates/eap74/eap74-openjdk17-image-stream.json  
# Templateのインポート  
for resource in eap74-amq-persistent-s2i.json eap74-amq-s2i.json eap74-basic-s2i.json eap74-https-s2i.json eap74-sso-s2i.json ; \  
do oc apply -f https://raw.githubusercontent.com/jboss-container-images/jboss-eap-openshift-templates/eap74/templates/${resource}; done  
.s2i の作成  
cd ~/work/kitchensink-jsp  
mkdir .s2i  
vi .s2i/environment  
以下の内容を持つテキストファイル .s2i/environment を作成する。  
CUSTOM_INSTALL_DIRECTORIES=extensions  
extensions の作成  
mkdir extensions  
以下の3つのファイルを作成する。  
extensions/install.sh  
extensions/postconfigure.sh  
extensions/config-database.cli  
extensions/install.sh の作成
以下の内容で作成する。  
#!/usr/bin/env bash  
set -x  
echo "=== Running $PWD/install.sh"  
injected_dir=$1  
# copy any needed files into the target build.  
cp -rf ${injected_dir} $JBOSS_HOME/extensions  
extensions/postconfigure.sh の作成  
以下の内容で作成する。  
#!/usr/bin/env bash  
echo "=== Executing postconfigure.sh"  
$JBOSS_HOME/bin/jboss-cli.sh --file=$JBOSS_HOME/extensions/config-database.cli  
extensions/config-database.cli の作成
以下の内容で作成する。  
embed-server --std-out=echo --server-config=standalone-openshift.xml  
# Register PostgreSQL JDBC driver  
/subsystem=datasources/jdbc-driver=postgresql:add( \  
driver-name=postgresql, \  
driver-module-name=org.postgresql, \  
driver-xa-datasource-class-name=org.postgresql.xa.PGXADataSource )  
# Create a datasource named KitchensinkJSPQuickstartDS  
# EAP内にJDBCドライバのモジュールを作成  
cd $JBOSS_HOME  
./bin/jboss-cli.sh  
[disconnected /] module add --name=org.postgresql \  
xa-data-source add \   
--name=KitchensinkJSPQuickstartDS \  
--jndi-name=java:jboss/datasources/KitchensinkJSPQuickstartDS \  
--driver-name=postgresql \  
--user-name=${env.MYDB_USERNAME} \  
--password=${env.MYDB_PASSWORD} \  
--validate-on-match=true \  
--background-validation=false \  
--valid-connection-checker-class-name=org.jboss.jca.adapters.jdbc.extensions.postgres.PostgreSQLValidConnectionChecker \  
--exception-sorter-class-name=org.jboss.jca.adapters.jdbc.extensions.postgres.PostgreSQLExceptionSorter \  
--xa-datasource-properties={ \  
"ServerName"=>"${env.MYDB_SERVER}", \  
"PortNumber"=>"${env.MYDB_PORT}", \  
"DatabaseName"=>"${env.MYDB_DATABASE}" }  
quit  
modules の作成  
ローカルのEAPにはPostgreSQLのJDBCドライバをモジュールとしてインストールして  
いるはずなので、それをコピーしてくる。  
mkdir modules  
cp -a $JBOSS_HOME/modules/org modules/  
（公式ドキュメントから module add のコマンド例をコピペした場合は org ではなく  
com になっている可能性あり。）  
以下の階層で二つのファイルが存在するはず。  
modules/org/postgresql/main/postgresql-42.7.1.jar  
modules/org/postgresql/main/module.xml  
penShift上でPostgreSQLを起動する  
openshift プロジェクトに postgresql-persistent というTemplateが用意されてい  
るのでそれを使って起動する。  
# Templateの詳細の確認  
oc describe template postgresql-persistent -n openshift  
# 必要なパラメタを指定してTemplateを適用する  
oc new-app --template=postgresql-persistent \  
-p POSTGRESQL_VERSION=13-el8 -p POSTGRESQL_USER=pguser \  
-p POSTGRESQL_PASSWORD=pgpassword -p POSTGRESQL_DATABASE=pgdatabase  
itchensink-jsp をGitHubに置く  
cd ~/work/kitchensink-jsp  
# ビルドの成果物が格納されるtargetディレクトリは無視するように設定  
echo "target" >> .gitignore  
git init  
git branch -m main  
git remote add origin git@github.com:<自分のアカウント名>/kitchensink-jsp.git   
git add .  
git commit -m "initial commit"  
git push -u origin main  
ースコード取得用のSecretの作成  
リポジトリをプライベートで作成したのでそのままでは取得に失敗する。  
後の oc new-app で使用するために自分のSSH秘密鍵をSecretとして作成しておく。  
oc create secret generic my-github-key \  
--from-file=ssh-privatekey=${HOME}/.ssh/id_rsa --type=kubernetes.io/ssh-auth  
oc new-app --template=eap74-basic-s2i \  
-p APPLICATION_NAME=myapp \  
-p IMAGE_STREAM_NAMESPACE=$(oc project -q) \  
-p EAP_IMAGE_NAME=jboss-eap74-openjdk17-openshift:latest \  
-p EAP_RUNTIME_IMAGE_NAME=jboss-eap74-openjdk17-runtime-openshift:latest \  
-p SOURCE_REPOSITORY_URL=git@github.com:<自分のアカウント名>/kitchensink-jsp.git \  
-p SOURCE_REPOSITORY_REF=main \  
-p CONTEXT_DIR="" \  
--source-secret=my-github-key \  
-e MYDB_USERNAME=pguser \  
-e MYDB_PASSWORD=pgpassword \  
-e MYDB_DATABASE=pgdatabase \  
-e MYDB_SERVER=postgresql \  
-e MYDB_PORT=5432  
# Routeの確認 ("https://"を付けてブラウザでアクセス)  
oc get route  
--resources=~/.m2/repository/org/postgresql/postgresql/42.7.1/postgresql-42.7.1.jar \
--dependencies=javaee.api,sun.jdk,ibm.jdk,javax.api,javax.transaction.api
[disconnected /] exit
