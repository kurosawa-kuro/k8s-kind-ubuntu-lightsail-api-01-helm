以下は、**Kind クラスターを作成**したうえで、**Helm Chart を使って最小構成の Deployment/Service をデプロイ**するまでの手順を、**1ステップずつ丁寧に**まとめた手順書です。  
「とりあえずこれを順番にコピペ・実行すれば同じ状態が作れる」という再現性を重視しています。

---

# 目次
1. **事前準備**  
2. **Kind クラスターの作成**  
3. **作業ディレクトリの初期化**  
4. **Helm Chart 雛形を作成**  
5. **不要ファイル・不要フォルダの削除**  
6. **values.yaml を最小構成向けに修正**  
7. **deployment.yaml / service.yaml の調整**  
8. **(オプション) \_helpers.tpl の確認**  
9. **テンプレート (YAML) の出力確認**  
10. **Helm install でデプロイ**  
11. **動作確認 (port-forward)**  
12. **アップグレード例 (replicaCount 変更)**  
13. **アンインストール**  
14. **まとめ**  

---

## 1. 事前準備

- **Docker** がインストールされ、`docker ps` などが実行可能
- **kind** (Kubernetes in Docker) がインストール済み
- **Helm** がインストール済み
- **AWS CLI** がインストールされ、`aws ecr get-login-password` が実行可能 (ECR ログインに使用)
- `kubectl` で Kubernetes にアクセスできる環境

---

## 2. Kind クラスターの作成

ここでは、**ECR のイメージ**を Pull できるように、`containerd` の認証設定を含んだ `kind-cluster.yaml` を用意します。

1) **ECR へのログイン用トークンを取得**

```bash
ECR_TOKEN=$(aws ecr get-login-password --region ap-northeast-1)
```

2) **kind クラスター設定ファイルを作成**

```bash
cat <<EOF > kind-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
containerdConfigPatches:
  - |-
    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.auths."503561449641.dkr.ecr.ap-northeast-1.amazonaws.com"]
        username = "AWS"
        password = "$ECR_TOKEN"
EOF
```

> - `503561449641.dkr.ecr.ap-northeast-1.amazonaws.com` の部分は、実際に使用する ECR レジストリに合わせて書き換えてください。  

3) **kind クラスター起動**

```bash
kind create cluster --config kind-cluster.yaml
kind get clusters
# => "kind" など、クラスター名が表示されればOK
```

これでローカルに Kind クラスターが起動しました。

---

## 3. 作業ディレクトリの初期化

```bash
# ホームディレクトリに移動
cd ~

# 作業用フォルダを作成（既にあればOK）
mkdir -p dev/k8s-kind-ubuntu-lightsail-api-01-helm

# 作業ディレクトリに移動
cd dev/k8s-kind-ubuntu-lightsail-api-01-helm

# 一応、想定どおりの場所か確認
pwd
# => /home/xxxx/dev/k8s-kind-ubuntu-lightsail-api-01-helm
```

> - **前提**：このフォルダは **空** または `README.md` のみの状態とします  

---

## 4. Helm Chart 雛形を作成

```bash
# Chart 名: container-nodejs-api-chart
helm create container-nodejs-api-chart
```

実行すると、以下のディレクトリ／ファイルが生成されます。

```
container-nodejs-api-chart/
├── Chart.yaml
├── charts/
├── .helmignore
├── templates/
│   ├── _helpers.tpl
│   ├── tests/
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   └── service.yaml
└── values.yaml
```

---

## 5. 不要ファイル・不要フォルダの削除

最小構成のチュートリアルには **Ingress や HPA、テスト機能** は必要ないため、以下を削除します。

```bash
cd container-nodejs-api-chart

# Ingress / HPA / tests用のファイル・フォルダを削除
rm -f templates/ingress.yaml
rm -f templates/hpa.yaml
rm -rf templates/tests
```

削除後の構成は以下のようになります。

```
container-nodejs-api-chart/
├── Chart.yaml
├── charts/
├── .helmignore
├── templates/
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   └── service.yaml
└── values.yaml
```

---

## 6. values.yaml を最小構成向けに修正

`container-nodejs-api-chart/values.yaml` を開き、**Ingress や HPA** 関連を削除／コメントアウトして、**Deployment/Service** に必要な設定だけを残します。  
下記は例ですので、**そのまま上書き**してもOKです。（`vim` / `nano` / お好きなエディタで編集）

```yaml
# container-nodejs-api-chart/values.yaml

# replicas
replicaCount: 1

# イメージ設定
image:
  repository: 986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8080
  tag: "latest"
  pullPolicy: Always

# Service設定
service:
  type: ClusterIP
  port: 8080

# コンテナのポート
containerPort: 8080

# リソース設定
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

> - 今回は例として `986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8080:latest` を使用  
> - もし別のイメージを使う場合は `image.repository` / `image.tag` を変更してください  

---

## 7. deployment.yaml / service.yaml の調整

### 7-1. deployment.yaml

`templates/deployment.yaml` は、`helm create` による不要コメント等を削除し、下記のように最小構成にします。

```yaml
# container-nodejs-api-chart/templates/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "container-nodejs-api-chart.fullname" . }}
  labels:
    {{- include "container-nodejs-api-chart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "container-nodejs-api-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "container-nodejs-api-chart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.containerPort }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### 7-2. service.yaml

`templates/service.yaml` も同様に不要コメント等を削除します。

```yaml
# container-nodejs-api-chart/templates/service.yaml

apiVersion: v1
kind: Service
metadata:
  name: {{ include "container-nodejs-api-chart.fullname" . }}
  labels:
    {{- include "container-nodejs-api-chart.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  selector:
    {{- include "container-nodejs-api-chart.selectorLabels" . | nindent 4 }}
  ports:
    - name: http
      port: {{ .Values.service.port }}
      targetPort: {{ .Values.containerPort }}
      protocol: TCP
```

---

## 8. (オプション) _helpers.tpl の確認

`templates/_helpers.tpl` には、リリース名やラベルを命名する関数が定義されています。  
`helm create` が生成したデフォルト内容でも問題なく動作しますが、**命名規則をカスタマイズ**したい場合は編集してください。  

最低限、`fullname` / `selectorLabels` / `labels` の 3つの定義があれば OK です。

---

## 9. テンプレート (YAML) の出力確認

```bash
cd ~/dev/k8s-kind-ubuntu-lightsail-api-01-helm/container-nodejs-api-chart

# Chart のテンプレートをローカルで確認
helm template . --values values.yaml
```

ここでデプロイされるはずの YAML がコンソールに出力されます。  
もしエラーが出る場合は、削除ファイルや typo を再度チェックしてください。

---

## 10. Helm install でデプロイ

```bash
# Release 名 "api" でインストール
helm install api . --values values.yaml

# Pod / Service / Helm Release が作成されたか確認
kubectl get pods
kubectl get svc
helm list
```

Pod が `Running` かつ `READY 1/1` になれば OK です。

---

## 11. 動作確認 (port-forward)

**NodePort や Ingress を作成しない**構成のため、`port-forward` でアクセス確認します。

```bash
# Service名は "api-container-nodejs-api-chart" (デフォルト命名)
# ※ "api" + "-" + (chart.name) など
kubectl port-forward service/api-container-nodejs-api-chart 8080:8080

# これで localhost:8080 とコンテナの 8080 ポートが繋がる
curl -v http://localhost:8080/
# => API のレスポンスが返ってくれば成功
```

---

## 12. アップグレード例 (replicaCount 変更)

**replicaCount** を変えて再デプロイする例です。

```bash
helm upgrade api . --set replicaCount=2

# Pod が2つに増えたか確認
kubectl get pods
```

---

## 13. アンインストール

```bash
helm uninstall api
```

これで作成した Deployment / Service などのリソースが削除されます。

---

## 14. まとめ

1. **Kind クラスター** で Kubernetes を起動 (ECR 認証付き)
2. **`helm create`** で雛形を作成  
3. **不要ファイル** (`ingress.yaml`, `hpa.yaml`, `tests`) を削除  
4. **`values.yaml` / `deployment.yaml` / `service.yaml`** を最小構成に編集  
5. **`helm template`** で YAML が正しいか確認  
6. **`helm install / upgrade / uninstall`** で実機テスト  

以上の手順を順番にコピペ・実行すれば、**Ingress 不要・NodePort 不要** の最小構成で Helm Chart をデプロイし、Kind クラスター上でアプリケーションを確認できます。  