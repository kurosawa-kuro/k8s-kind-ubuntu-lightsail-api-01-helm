# k8s-kind-ubuntu-lightsail-api-01-helm

以下の手順で、`~/dev/k8s-kind-ubuntu-lightsail-api-01-helm` ディレクトリに Helm Chart を最小構成で作成してください。  
（ディレクトリが空 or `README.md` のみであることを前提としています。）

---

## ディレクトリ構成

```
~/dev/k8s-kind-ubuntu-lightsail-api-01-helm
└── container-nodejs-api-chart
    ├── Chart.yaml
    ├── values.yaml
    └── templates
        ├── _helpers.tpl
        ├── deployment.yaml
        └── service.yaml
```

1. **`~/dev/k8s-kind-ubuntu-lightsail-api-01-helm` ディレクトリに移動**
2. **`container-nodejs-api-chart` ディレクトリを作成**
3. **`templates` ディレクトリを作成**

最終的に上記の構成になるように、以下のファイルを配置してください。

---

## 1) Chart.yaml

```yaml
apiVersion: v2
name: container-nodejs-api-chart
description: A Helm chart for container-nodejs-api
version: 0.1.0
appVersion: "1.0"
```

- `name:` を **`container-nodejs-api-chart`** にしています。  
  - 後述するテンプレート内で `{{ include "container-nodejs-api-chart.xxx" . }}` と参照しているためです。

---

## 2) values.yaml

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
  create: true
  name: "container-nodejs-api"
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

---

## 3) templates/\_helpers.tpl

Helm テンプレート内の `include` で呼び出す共通関数や命名規則をまとめるためのファイルです。以下をそのまま配置してください。

```yaml
{{- define "container-nodejs-api-chart.name" -}}
{{ .Chart.Name }}
{{- end }}

{{- define "container-nodejs-api-chart.chart" -}}
{{ printf "%s-%s" .Chart.Name .Chart.Version }}
{{- end }}

{{- define "container-nodejs-api-chart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else }}
{{- printf "%s-%s" .Release.Name (include "container-nodejs-api-chart.name" .) | trunc 63 | trimSuffix "-" -}}
{{- end }}
{{- end }}

{{- define "container-nodejs-api-chart.serviceAccountName" -}}
{{- if .Values.serviceAccount.create -}}
{{- default (include "container-nodejs-api-chart.fullname" .) .Values.serviceAccount.name | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- default "default" .Values.serviceAccount.name -}}
{{- end }}
{{- end }}

{{- define "container-nodejs-api-chart.labels" -}}
helm.sh/chart: {{ include "container-nodejs-api-chart.chart" . }}
app.kubernetes.io/name: {{ include "container-nodejs-api-chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion }}
app.kubernetes.io/managed-by: Helm
{{- end }}

{{- define "container-nodejs-api-chart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "container-nodejs-api-chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

---

## 4) templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "container-nodejs-api-chart.fullname" . }}
  labels:
    {{- include "container-nodejs-api-chart.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "container-nodejs-api-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "container-nodejs-api-chart.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "container-nodejs-api-chart.serviceAccountName" . }}
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

---

## 5) templates/service.yaml

```yaml
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
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.containerPort }}
      protocol: TCP
      name: http
```

---

# インストール・動作確認

上記ファイルをすべて配置後、以下のコマンドを実行して動作確認できます。

```bash
# Chartのテンプレートを確認
cd ~/dev/k8s-kind-ubuntu-lightsail-api-01-helm/container-nodejs-api-chart
helm template . --values values.yaml

# インストール (Release名: api)
helm install api . --values values.yaml

# デプロイ状態確認
kubectl get pods
kubectl get svc
helm list

# 例: replicas を増やしてアップグレード
helm upgrade api . --set replicaCount=2

# アンインストール
helm uninstall api
```

## port-forward で確認

```bash
# Service を 8080 でフォワード
kubectl port-forward service/api-container-nodejs-api-chart 8080:8080

# 別ターミナルで確認
curl -v http://localhost:8080/
# => {"status":"ok","timestamp":"..."}
```

---

以上で、最小構成の Deployment / Service を Helm Chart として管理するチュートリアルは完了です。  
Ingress や NodePort を使わずに内部アクセス専用としたまま、`helm install / upgrade / uninstall` の流れを学ぶことができます。ぜひご活用ください。
