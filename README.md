# 目次

1. **Chart の構造・ファイル内容**  
2. **Helm install / upgrade / uninstall**  
3. **port-forward で動作確認**  

> - 引き続き **Ingress 不要・禁止**  
> - **NodePort も使わない**  
> - **サーバ内部でのみアクセス**  
> - イメージ: `986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8080:latest`  
> - コンテナポート: **8080**  

---

## 1️⃣ Chart の構造・ファイル内容

### 1-1. Chart.yaml

```yaml
apiVersion: v2
name: my-app-chart
description: A Helm chart for my-app
version: 0.1.0
appVersion: "1.0"
```

### 1-2. values.yaml

```yaml
# replicas (Deployment)
replicaCount: 1

# イメージ設定
image:
  repository: 986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8080
  tag: "latest"
  pullPolicy: Always

# コンテナのポート
containerPort: 8080

# Service設定
service:
  type: ClusterIP
  port: 8080

# ServiceAccount設定
serviceAccount:
  create: false
  name: ""
  annotations: {}

# Ingress設定（無効化）
ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts: []
  tls: []

# オートスケーリング設定（無効化）
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 80

# リソース制限
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

### 1-3. templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app-chart.fullname" . }}
  labels:
    {{- include "my-app-chart.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app-chart.selectorLabels" . | nindent 8 }}
    spec:
      {{- if .Values.serviceAccount.create }}
      serviceAccountName: {{ include "my-app-chart.serviceAccountName" . }}
      {{- end }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.containerPort }}
              protocol: TCP
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### 1-4. templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "my-app-chart.fullname" . }}
  labels:
    {{- include "my-app-chart.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  selector:
    {{- include "my-app-chart.selectorLabels" . | nindent 4 }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.containerPort }}
      protocol: TCP
      name: http
```

---

## 2️⃣ Helm install / upgrade / uninstall

### 2-1. テンプレートの検証

```bash
cd ~/dev/k8s-kind-ubuntu-lightsail-api-01-helm/my-app-chart

# YAML出力を確認
helm template . --values values.yaml
```

### 2-2. インストール

```bash
# リリース名 container-api でインストール
helm install container-api . --values values.yaml
```

実行後、Kubernetes リソースが作成されたことを確認します。

```bash
kubectl get pods
kubectl get svc
helm list
```

### 2-3. アップグレード

```bash
# 例: replicas を 2 に増やす場合
helm upgrade container-api . --set replicaCount=2
```

### 2-4. アンインストール

```bash
helm uninstall container-api
```

---

## 3️⃣ port-forward で動作確認

```bash
# Service名を確認
kubectl get svc

# 正しいService名でport-forward
kubectl port-forward service/container-api-my-app-chart 8080:8080

# 別ターミナルで確認
curl -v http://localhost:8080/
# => {"status":"ok","timestamp":"2025-04-14T03:40:12.137Z"}
```

> **重要**: Service名は `container-api-my-app-chart` となります。これは Helm のリリース名とチャート名から自動生成されます。

---

# まとめ

- **Helm Chart** を用いることで、Kubernetes リソースをまとめてパッケージ化し、一括デプロイできます。
- **Ingress, NodePort** 不要の構成を維持しながら、Helm の利点を活用できます。
- ServiceAccount や HPA など、将来の拡張に備えた設定も含まれています。
- **ポート設定** は、コンテナポート、サービスポート共に **8080** を使用します。

主な修正ポイント：
1. チャート名とリリース名の一貫性を保持
2. ポート番号を8080に統一
3. Service名の正確な記述
4. 手順の明確化と詳細な説明の追加
