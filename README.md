以下では、**Helm** の標準雛形（`helm create`）で生成されるテンプレートファイルを **できるだけ弄らず** に、  
- **Ingress**  
- **ServiceAccount**  
- **HPA**  

のそれぞれを **`values.yaml` 上のフラグで無効化（enabled=false）** する方法を中心としたチュートリアルをまとめます。  
最終的に **Kind クラスター** 上でテストし、**Deployment/Service** のみが有効になったアプリケーションをデプロイします。

---

# 目次

1. [事前準備](#1-事前準備)  
2. [Kind クラスターの作成](#2-kind-クラスターの作成)  
3. [Helm Chart の雛形作成 (`helm create`)](#3-helm-chart-の雛形作成-helm-create)  
4. [values.yaml による無効化設定（最重要ポイント）](#4-valuesyaml-による無効化設定最重要ポイント)  
5. [テンプレートへの最小限の確認作業 (任意)](#5-テンプレートへの最小限の確認作業-任意)  
6. [`helm template` で検証](#6-helm-template-で検証)  
7. [`helm install` でデプロイ](#7-helm-install-でデプロイ)  
8. [動作確認 (port-forward)](#8-動作確認-port-forward)  
9. [アップグレード例 (replicaCount 変更)](#9-アップグレード例-replicacount-変更)  
10. [アンインストール](#10-アンインストール)  
11. [まとめ](#11-まとめ)  

---

## 1. 事前準備

- **Docker** がインストールされ、`docker ps` などが実行可能
- **Kind** (Kubernetes in Docker) がインストール済み
- **Helm** がインストール済み
- **AWS CLI** がインストールされ、`aws ecr get-login-password` が実行可能 (ECR ログインに使用)
- `kubectl` で Kubernetes にアクセスできる環境

---

## 2. Kind クラスターの作成

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

> - `503561449641.dkr.ecr.ap-northeast-1.amazonaws.com` の部分は適宜ご自身の ECR レジストリに書き換えてください。

3) **kind クラスター起動**

```bash
kind create cluster --config kind-cluster.yaml
kind get clusters
# => "kind" が表示されればOK
```

---

## 3. Helm Chart の雛形作成 (`helm create`)

```bash
# 任意の作業ディレクトリで
mkdir -p ~/dev/k8s-kind-helm-tutorial
cd ~/dev/k8s-kind-helm-tutorial

# chart名: my-app-chart とする例
helm create my-app-chart
```

実行すると、以下の構成が自動生成されます。

```
my-app-chart/
├── Chart.yaml
├── charts/
├── .helmignore
├── templates/
│   ├── _helpers.tpl
│   ├── tests/
│   ├── serviceaccount.yaml
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   └── service.yaml
└── values.yaml
```

> **ポイント**: ここでは **ファイルを削除しません**  
> → Ingress/HPA/ServiceAccount 等も残したまま、`values.yaml` の設定で無効化を行います。

---

## 4. values.yaml による無効化設定（最重要ポイント）

`my-app-chart/values.yaml` を開き、**Ingress, ServiceAccount, HPA** を `enabled: false` に設定します。  
（標準生成された雛形には一部 `ingress.enabled` / `autoscaling.enabled` が用意されていない場合があります。その場合は下記を追加してください）

```yaml
# replicas
replicaCount: 1

# image
image:
  repository: 111111111111.dkr.ecr.ap-northeast-1.amazonaws.com/my-app
  tag: "latest"
  pullPolicy: Always

# Ingress (無効化)
ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts: []
  tls: []

# ServiceAccount (無効化)
serviceAccount:
  create: false
  name: ""
  annotations: {}

# HPA / Autoscaling (無効化)
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 80

# Service
service:
  type: ClusterIP
  port: 80

# (アプリ内部でListenするポート例)
containerPort: 8080

# リソース設定例
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

> - `image.repository` / `image.tag` を **ECR 上のご自身のイメージ**に変更してください。  
> - `containerPort` は使用するアプリのポートに合わせます（例: 8080）。  
> - `service.port` は Kubernetes 上で公開するポートとして合わせます（例: 80 or 8080）。  

### 重要:
- `ingress.enabled: false`  
- `serviceAccount.create: false`  
- `autoscaling.enabled: false`  

これらをしっかり設定することで、後述のテンプレートが **無効** 扱いになります。

---

## 5. テンプレートへの最小限の確認作業 (任意)

`helm create` の標準テンプレートには、`{{- if .Values.ingress.enabled }}` や `{{- if .Values.autoscaling.enabled }}` のような条件分岐が **既に** 書かれている場合が多いです。

**例:** `ingress.yaml` に

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}
```

のような記述があれば OK。  
もし入っていないなら、先頭に `{{- if .Values.ingress.enabled }}` / `{{- end }}` を追加してください。  

同様に、`hpa.yaml` や `serviceaccount.yaml` にも `{{ if .Values.autoscaling.enabled }}` や `{{ if .Values.serviceAccount.create }}` があるかを確認し、必要なら追加します。

> - **「できるだけテンプレートは弄らない」** という方針なので、標準雛形に if 分岐が無い場合は最低限そこだけ足してください。  
> - それ以外の大規模な改変やファイル削除はせずに済みます。

---

## 6. `helm template` で検証

```bash
cd my-app-chart
helm template . --values values.yaml
```

- `ingress.enabled=false` → `ingress.yaml` は **出力されない**  
- `autoscaling.enabled=false` → `hpa.yaml` は **出力されない**  
- `serviceAccount.create=false` → `serviceaccount.yaml` は **出力されない**  

これで、Deployment / Service のみが出力されれば成功です。  
もし無効化されずに出力される場合は、テンプレート側に if 分岐が無いなどの原因が考えられます。

---

## 7. `helm install` でデプロイ

```bash
# リリース名: my-app
helm install my-app . --values values.yaml

kubectl get pods
kubectl get svc
helm list
```

Pod が `Running` になっていれば成功。  
`kubectl describe pod` などで image が正しくPullできているか確認すると安心です。

---

## 8. 動作確認 (port-forward)

Ingress や NodePort を使わない構成なら、`port-forward` でローカルアクセスします。

```bash
# Service名は "my-app-my-app-chart" (helm createの命名規則による)
kubectl port-forward service/my-app-my-app-chart 8080:80

# コンテナの中では 8080 でリッスンしている場合
# => "service port 80" → "targetPort 8080" というマッピング
curl -v http://localhost:8080/
```

---

## 9. アップグレード例 (replicaCount 変更)

```bash
helm upgrade my-app . --set replicaCount=2
kubectl get pods
```

Pod が 2つに増えれば OK。

---

## 10. アンインストール

```bash
helm uninstall my-app
```

これで作成したリソース（Deployment, Service, etc.）が削除されます。

---

## 11. まとめ

1. **`helm create`** で雛形を作成し、Ingress / HPA / ServiceAccount などの **テンプレートファイルを削除せず** に残す  
2. **`values.yaml`** で `ingress.enabled = false`, `autoscaling.enabled = false`, `serviceAccount.create = false` を設定  
3. テンプレートに `{{ if .Values.xxx.enabled }}` があるか最小限だけ確認（なければ追加）  
4. **`helm template`** → 出力を確認  
5. **`helm install`** → Kubernetes にデプロイ  
6. **`kubectl port-forward`** などで動作確認  

これにより「Ingress/HPA/ServiceAccount が無効化された最小構成の Helm チャート」がデプロイされます。  
**将来的に使いたくなったら** `ingress.enabled=true` などとするだけでリソースを有効化でき、柔軟に拡張可能です。

> - テンプレートを削除しなくてよいので、あとで機能をオンにするだけで簡単に使える点がメリットです。  