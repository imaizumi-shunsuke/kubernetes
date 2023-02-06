# KubernetesのNetworkについてのメモ

## 環境設定

まず3つのノードを含むクラスタを構成する。

```
$ minikube start --nodes=3
```

立ち上がりに2,3分かかるので、Done!のメッセージが出るまで待つ。

なお、クラスタの構成がうまく行かない場合は、クラスタごと削除してやり直せる。

```
$ minikube delete
```

podの稼動状態を見る。

podはコンテナ本体に相当する部分で一つ以上のDockerコンテナを持つ。一つのIPアドレスを持つ。

```
$ minikube kubectl -- get pods -A
```

nodeの稼動状態を見る。

nodeはアプリを実行するマシンに相当する。一台の物理サーバや仮想マシンと理解すればよい。nodeはpodを一つ以上持つ。

```
$ minikube kubectl get nodes
```

3つのnodesがREADYとなっていれば良い。

この段階でDockerのコンテナをみると各ノード名でコンテナが動いていているが、これらはKubernetesでこれから動かすコンテナとは別環境なので要注意。

環境は整ったのでアプリを用意する。

コンテナ内で稼動させるアプリをDockerイメージとして用意する。

同じディレクトリのDockerfileがflaskの実行環境を作り、

app.pyはGETリクエストに対してHello World!を返すAPIになっている。

更にイメージをビルドする前にDocker環境を切り替える必要がある。

minikubeはDockerのデフォルト環境を利用せずに、独自のDocker環境を作り、そこでクラスタを生成する。

Dockerの初期設定のままimageをbuildすると、デフォルト環境にimageが作られてしまう。

切り替えるためにnodeのIPアドレスを確認する。

```
$ docker exec -it minikube hostname -i
192.168.49.2
$ docker exec -it minikube-m02 hostname -i
192.168.49.3
$ docker exec -it minikube-m03 hostname -i
192.168.49.4
```

minikube node(最初の192.168.49.2のやつ)のDocker環境に切り変える。

```
$ export DOCKER_TLS_VERIFY="1"
$ export DOCKER_HOST="tcp://192.168.49.2:2376"
$ export DOCKER_CERT_PATH="/home/simaizumi/.minikube/certs"
$ export MINIKUBE_ACTIVE_DOCKERD="minikube"
```

docker psをして、k8s_で始まるコンテナが複数表示されれば切り替え完了。

minikubeのDocker環境でビルドする。

```
$ docker build -t sample_api:1.0 ./
```

docker imagesでsample_api:1.0のイメージが出来ているか確認しておくと良い。

minikube-m02, minikube-m03についても同様にビルドする。
```
$ export DOCKER_HOST="tcp://192.168.49.3:2376"
$ docker build -t sample_api:1.0 ./
$ export DOCKER_HOST="tcp://192.168.49.4:2376"
$ docker build -t sample_api:1.0 ./
```

ビルドが完了したらデフォルトのDocker環境に戻す。

```
$ eval $(minikube docker-env -u)
```

これで環境設定は完了！

## Pod内通信
1つのpodは1つのIPアドレスをもつ、ということは、pod内のコンテナ同士はlocalhostで通信できる。

まずは1つのpod内でコンテナを2つ起動するマニフェストファイルpod.yamlを作成する。

curl-containerの方はapp.pyの実行ではなく、コンテナを起動させ続ける。

yamlファイルの設定をpodに適用する。

```
$ minikube kubectl -- apply -f pod.yaml
```

minikube kubectl get podsでsample_podが2/2 RunningになっていればPodsが正常に動いていることを確認できる。

ようやくPod内の通信を確かめられる。

minikube kubectl -- exec -it Pod名 コンテナ名 -- 実行したいコマンド

でコンテナ内でコマンドを実行することができる。どのPodの中のコンテナか指定する必要がある。

例えば、sample-pod内のcurl-container内でcurl localhost:5000を実行すると

```
$ minikube kubectl -- exec -it sample-pod curl-container -- curl localhost:5000
```

Hello World!が返ってくる。

これはapi-containerが5000番のportからのGETメソッドに対して、Hello World!を返すapp.pyが実行されているからだ。

つまり、同じネットワーク内でcurl-containerとapi-containerが実行されていることが確認できる。

これにてPod内通信の確認は終了。

最後にPodを削除しておく。

```
$ minikube kubectl -- delete -f pod.yaml
```