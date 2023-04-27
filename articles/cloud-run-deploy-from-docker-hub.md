---
title: "Artifact Registry リモートリポジトリを使って Docker Hub から Cloud Run へのお手軽デプロイ"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [artifactregistry, cloudrun, gcp, docker]
published: false
---

これまで、Cloud Run にコンテナイメージをデプロイするには、先に Google Cloud の Artifact Registry や Container Registry などにプライベートリポジトリを作成し、必ずそこにコンテナイメージを Push する必要がありました。そのため、Docker Hub で公開されている既存のコンテナイメージであっても、デプロイ前にローカルの作業環境などを経由してコンテナイメージをアップロードする必要があり、それが個人的にはちょっと面倒なポイントでした（AWS だと、Amazon ECR Public がそのあたりカバーしてくれていると思います）。

しかし、[2023年2月の機能アップデート](https://cloud.google.com/artifact-registry/docs/release-notes#February_14_2023) で、[リモートリポジトリ](https://cloud.google.com/artifact-registry/docs/repositories/remote-repo) という機能の Preview 提供がはじまり、この点が改善されました。

Artifact Registry にリモートリポジトリを作成すると、そのリモートリポジトリを経由して、Docker Hub 上の公開イメージを Pull することが可能になります。リモートリポジトリがプロキシとして働いてくれるイメージですね（`Docker Client <-> Artifact Registry リモートリポジトリ <-> Docker Hub`）。

使い方はシンプルですが、一応順を追って見てみましょう。


## リモートリポジトリの作成

まずは、[公式ドキュメント](https://cloud.google.com/artifact-registry/docs/repositories/remote-repo?hl=ja) に従って Cloud Console からリモートリポジトリを作成してみます。Artifact Registry の画面を開いて API を有効化したあと、`リポジトリの作成` から、「形式」を `Docker`、「モード」を `リモート` と指定してリポジトリを作成します。

![リモートリポジトリの作成](/images/cloud-run-deploy-from-docker-hub/create-remote-repo.png)

現状では、リモートリポジトリの接続先としては Docker Hub のみに対応しているので、自動的に `Docker Hub` が選択されていますね。リポジトリの作成が終わったあと、詳細画面を開くと以下のように表示されます。

![リモートリポジトリの詳細](/images/cloud-run-deploy-from-docker-hub/remote-repo-details.png)

なお、Cloud Console の画面ではなく Terraform を利用する場合には、Google Cloud 用 Provider の [google_artifact_registry_repository](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/artifact_registry_repository) を使って、`mode = "REMOTE_REPOSITORY"` を指定することで同様にリモートリポジトリを作成できます。

```
resource "google_artifact_registry_repository" "repo" {
  location      = "asia-northeast1"
  repository_id = "docker"
  description   = "docker remote repository for Docker Hub"
  format        = "DOCKER"
  mode          = "REMOTE_REPOSITORY"
  remote_repository_config {
    docker_repository {
      public_repository = "DOCKER_HUB"
    }
  }
}
```


## リモートリポジトリの利用

では早速、作成したリモートリポジトリを経由して Docker Hub のコンテナイメージを Pull してみましょう。

ここでは、Docker Hub で公開されている [hello-world](https://hub.docker.com/_/hello-world) というサンプルのコンテナイメージを Pull してみます。

通常、Docker Hub から直接イメージを Pull する場合は、`docker pull hello-world` といったように、イメージ名のみで指定することができます。一方、Artifact Registry のリモートリポジトリ経由で Pull する場合には、Artifact Registry の URL 形式にしたがって、Pull 対象を以下のように指定します。

`<REGION_NAME>-docker.pkg.dev/<PROJECT_ID>/<REPOSITORY_NAME>/<DOCKER_HUB_IMAGE_NAME>(:<TAG>)`

`<DOCKER_HUB_IMAGE_NAME>` というのが、今回で言うと `hello-world` なので、例えば以下のようになります。
`asia-northeast1-docker.pkg.dev/nosu-sandbox/docker/hello-world`

あるいは、タグを明示的に指定する場合にはこうなります。
`asia-northeast1-docker.pkg.dev/nosu-sandbox/docker/hello-world:latest`

それでは、実際に Cloud Shell を開き、Docker Hub のイメージを Pull できるか見てみましょう。
最初に認証設定を行います。

```bash
gcloud auth configure-docker asia-northeast1-docker.pkg.dev 
```

これによって、Docker の構成ファイル（.docker/config.json）に Artifact Registry 用の認証設定が追加されます。認証設定ができたら、あとは `docker pull` するだけです。

```bash
docker pull asia-northeast1-docker.pkg.dev/${PROJECT_ID}/docker/hello-world
# ...
# => Status: Downloaded newer image for asia-northeast1-docker.pkg.dev/<PROJECT_ID>/docker/hello-world:latest
# => asia-northeast1-docker.pkg.dev/${PROJECT_ID}/docker/hello-world:latest
```

うまくいきましたね。


### Cloud Run へデプロイする

では、いよいよ Docker Hub のイメージを Cloud Run にデプロイしてみましょう。

ここでは、Docker Hub で公開されている [dockersamples/static-site](https://hub.docker.com/r/dockersamples/static-site) というサンプルの Web アプリをデプロイしてみます。

まず、Cloud Console から Cloud Run のサービスデプロイ画面を開いてみると、先ほど Pull した `hello-world` のみが表示されているのがわかります。少なくとも現状では、一度 Pull してキャッシュされたイメージしか指定できないようです。

![コンテナイメージの選択](/images/cloud-run-deploy-from-docker-hub/container-image-select.png)

一方で、gcloud コマンドを使えばキャッシュの有無に関わらず Docker Hub 上のコンテナイメージを何でもデプロイできるので、ここでは gcloud コマンドを使ってデプロイしてみます。

```bash
gcloud run deploy image-from-docker-hub \
  --image=asia-northeast1-docker.pkg.dev/nosu-sandbox/docker/dockersamples/static-site \
  --port=80 --allow-unauthenticated --region=asia-northeast1
# Deploying container to Cloud Run service [image-from-docker-hub] in project [nosu-sandbox] region [asia-northeast1]
# OK Deploying new service... Done.                                                                   
#   OK Creating Revision... Creating Service.
#   OK Routing traffic...
#   OK Setting IAM Policy...
# Done.
# Service [image-from-docker-hub] revision [image-from-docker-hub-00001-kuc] has been deployed and is serving 100 percent of traffic.
# Service URL: https://image-from-docker-hub-laqxooze7a-an.a.run.app
```

`Service URL` を Ctrl + クリックすると、デプロイした Web サイトが表示されます。
デプロイが正常に完了して、Cloud Run でホストされた Web アプリが正常に動作していることが確認できました。

![dockersamples/static-site](/images/cloud-run-deploy-from-docker-hub/hello-docker.png)


### （おまけ）他のプロジェクトの Cloud Run からリモートリポジトリを利用する

検証用途などで、Docker Hub 上のコンテナイメージをサクッと Cloud Run にデプロイしたいことは非常によくあると思いますが（？）、そのためだけにプロジェクトごとにリモートリポジトリを作成するのは面倒です。

その場合、どこかの共有プロジェクトにリモートリポジトリを一個作っておいて、（本番環境など細かい権限管理が必要なケースを除いた）雑多な用途で共通利用する、ということもできます。

Cloud Run が Artifact Registry からイメージを Pull する際には、Cloud Run の属するプロジェクトにおける Cloud Run の Service Agent と呼ばれるサービスアカウント（`service-<PROJECT_NUMBER>@serverless-robot-prod.iam.gserviceaccount.com`）の権限が利用されます。この Service Agent がリモートリポジトリにアクセスできるようにするには、以下のようにしてリモートリポジトリが存在しているプロジェクトにおける `Artifact Registry 読み取り（roles/artifactregistry.reader）` 権限を付与します。

`<PROJECT_NUMBER>` は、Cloud Run をデプロイするプロジェクトのものです。よく利用する Project ID ではなく Project Number が必要なのでご注意ください。（参考：[プロジェクト番号の確認方法](https://cloud.google.com/resource-manager/docs/creating-managing-projects?hl=ja#identifying_projects)）

```
gcloud artifacts repositories add-iam-policy-binding docker \
   --location=asia-northeast1 \
   --member="serviceAccount:service-<PROJECT_NUMBER>@serverless-robot-prod.iam.gserviceaccount.com" \
   --role=roles/artifactregistry.reader
```

Terraform の場合はこんな感じです。

```
resource "google_artifact_registry_repository_iam_member" "repo-iam" {
  location = "asia-northeast1"
  repository = "docker"
  role   = "roles/artifactregistry.reader"
  member = "serviceAccount:service-<PROJECT_NUMBER>@serverless-robot-prod.iam.gserviceaccount.com"
}
```


### 感想

というわけで、リモートリポジトリを利用すると、Docker Hub から Cloud Run へのデプロイがかなりやりやすくなったので、Docker Hub に公開されているイメージや、あるいは自分で開発したアプリケーションでも、OSS として Docker Hub に公開しても良いものであれば、リモートリポジトリ経由でのデプロイを検討してみると良さそうです。

なお、リモートリポジトリに加えて、[仮想リポジトリ（Virtual Repository）](https://cloud.google.com/artifact-registry/docs/repositories/virtual-repo?hl=ja) という複数のリポジトリを一つのエンドポイントに集約するような機能も追加されています。これについても時間があるとき書きます。
