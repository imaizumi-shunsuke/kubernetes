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
$ kubectl get pod
```

nodeの稼動状態を見る。

nodeはアプリを実行するマシンに相当する。一台の物理サーバや仮想マシンと理解すればよい。nodeはpodを一つ以上持つ。

```
$ kubectl get nodes
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
$ kubectl apply -f pod.yaml
```

kubectl get podでsample_podが2/2 RunningになっていればPodsが正常に動いていることを確認できる。

ようやくPod内の通信を確かめられる。

kubectl exec -it Pod名 コンテナ名 -- 実行したいコマンド

でコンテナ内でコマンドを実行することができる。どのPodの中のコンテナか指定する必要がある。

例えば、sample-pod内のcurl-container内でcurl localhost:5000を実行すると

```
$ kubectl exec -it sample-pod curl-container -- curl localhost:5000
```

Hello World!が返ってくる。

これはapi-containerが5000番のportからのGETメソッドに対して、Hello World!を返すapp.pyが実行されているからだ。

つまり、同じネットワーク内でcurl-containerとapi-containerが実行されていることが確認できる。

これにてPod内通信の確認は終了。

最後にPodを削除しておく。

```
$ kubectl delete -f pod.yaml
```

## Pod間通信
次にPodとPodの間での通信を確認する。

Dockerだけで異なるネットワークにあるコンテナ間通信を行う場合、NAT(ポートフォワーディング）などの設定が必要となるが、KubernetesではNATなしでPod間通信が可能。

簡単に異なるネットワーク間の通信が可能なことを確認しよう。

今回は先ほどの３つのnodeを利用して、それぞれで同じPodを起動する。

nodeに一つずつ同じPodを配置する場合、DaemonSetという設定ができる。（ちなみに先ほどはPodという方法でコンテナやコマンドを別々に指定した。）

daemonset.yamlに設定内容があるので中身はそちらを確認してほしい。

それぞれのnodeでsample_api:1.0のイメージからコンテナを作り、python app.pyを実行している。

```
$ kubectl apply -f daemonset.yaml
```

無事実行されれば、以下のコマンドでIPアドレスなどが確認できる。

```
$ kubectl get pod -o wide
```

最後にPodから別のPodにアクセスできることを確認する。

例えば、このケースではminikubeで動いているPodからminikube-m02にアクセスできる。ただし、Podの名前は実行環境によって違うので注意。

```
$ kubectl exec -it sample-daemonset-rbdk9 -- curl 10.244.1.2:5000
```

Hello world!が帰ってくればok。

## nodeとPodの通信
最後にnodeで稼動するプロセスでもPodにアクセスできることを確認する。

このケースではminikube-m02のノードにssh接続し、そこからminikube-m03のPodにアクセスする。

```
$ minikube ssh -n minikube-m02
docker@minikube-m02:~$ curl 10.244.2.3:5000
```

こちらでもHello world!が返ってくれば完了。

以上でkubernetesのネットワーク基礎については終了。