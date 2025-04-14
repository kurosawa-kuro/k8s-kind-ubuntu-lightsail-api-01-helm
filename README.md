以下の内容は、「**Helm Chart を使って最小構成の Deployment/Service をデプロイする**」手順を、**再現性を高める**ために**1ステップずつ丁寧にリファクタリング**したものです。  
「とりあえずこれを順番にコピペ・実行すれば同じ状態が作れる」という流れを目指しています。

---

# Helm Chart 最小構成リファクタリング手順

## 事前準備
- ローカルマシンに **Helm** がインストール済みであること  
- **kubectl** で Kubernetes クラスター（Kind, Minikube, etc.）にアクセスできる状態であること  
- 以下ではパス `~/dev/k8s-kind-ubuntu-lightsail-api-01-helm` を作業ディレクトリとします

---

## 1. 作業ディレクトリの初期化

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

## 2. Helm Chart 雛形を作成

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

## 3. 不要ファイル・不要フォルダの削除

最小構成のチュートリアルには **Ingress や HPA、テスト機能** は必要ないため、以下を削除します。

```bash
# Ingress / HPA / tests用のファイル・フォルダを削除
rm -f container-nodejs-api-chart/templates/ingress.yaml
rm -f container-nodejs-api-chart/templates/hpa.yaml
rm -rf container-nodejs-api-chart/templates/tests
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

## 4. values.yaml を最小構成向けに修正

`container-nodejs-api-chart/values.yaml` を開き、**Ingress や HPA** 関連を削除／コメントアウトして、**Deployment/Service** に必要な設定だけを残します。  
下記は例ですので、**そのまま上書き**してもOKです（`vim` / `nano` / お好きなエディタで編集）。  

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

> - 今回は例として ECR リポジトリ `986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8080:latest` を使用  
> - もし別のイメージを使う場合は `repository` / `tag` を変更してください  

---

## 5. deployment.yaml の不要記述を削除／整理

`container-nodejs-api-chart/templates/deployment.yaml` の初期状態には、`helm create` によるコメントや HPA 対応の残りなどが混在している場合があります。  
**最小構成** では下記のように整理してください。

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

> - `serviceAccountName` や `autoscaling.enabled` など不要なら削除  
> - `ports` の `name: "http"` などは任意。最低限 `containerPort` があればOK  

---

## 6. service.yaml の不要記述を削除／整理

`container-nodejs-api-chart/templates/service.yaml` も同様に整理しましょう。

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

## 7. (オプション) _helpers.tpl の確認

`templates/_helpers.tpl` には、リリース名やラベルを命名する関数が定義されています。  
`helm create` が生成するデフォルト内容でも問題なく動作しますが、**命名規則をカスタマイズしたい場合**は編集してください。  

最低限、`fullname` / `selectorLabels` / `labels` の 3つの定義があれば OK です。

---

## 8. テンプレート (YAML) の出力確認

```bash
cd container-nodejs-api-chart

# Chart のテンプレートをローカルで確認
helm template . --values values.yaml
```

ここでデプロイされるはずの YAML がコンソールに出力されます。  
エラーが出る場合は、ファイル削除ミスや typo を再度チェックしてください。

---

## 9. Helm install でデプロイ

```bash
# Release 名 "api" でインストール
helm install api . --values values.yaml

# Pod / Svc / Helm Release が作成されたか確認
kubectl get pods
kubectl get svc
helm list
```

Pod が `Running` かつ `READY 1/1` になれば OK です。

---

## 10. 動作確認 (port-forward)

**NodePort や Ingress を作成しない**構成のため、`port-forward` でアクセス確認します。

```bash
# 別ターミナルでも可。Svc名: api-container-nodejs-api-chart
#   ※ "api-<chart名>" は _helpers.tpl の定義による
kubectl port-forward service/api-container-nodejs-api-chart 8080:8080

# これで localhost:8080 とコンテナの 8080 ポートが繋がる
curl -v http://localhost:8080/
# => API のレスポンスが返ってくれば成功
```

---

## 11. アップグレード例

**replicaCount** を変えて再デプロイする例です。

```bash
helm upgrade api . --set replicaCount=2

# Pod が2つに増えたか確認
kubectl get pods
```

---

## 12. アンインストール

```bash
helm uninstall api
```

これで作成した Deployment / Service などのリソースが削除されます。

---

# まとめ

1. **`helm create`** で雛形を作成  
2. **不要ファイル** (`ingress.yaml`, `hpa.yaml`, `tests`) を削除  
3. **`values.yaml` / `deployment.yaml` / `service.yaml`** を最小構成に編集  
4. **`helm template`** で YAML が正しいか確認  
5. **`helm install / upgrade / uninstall`** で実機テスト  

以上の手順を順番にコピペ・実行すれば、**Ingress 不要・NodePort 不要** の最小構成で Helm Chart をデプロイできることが確認できます。