# Velero Backup & Restore PoC — Kind Cluster on Ubuntu

Velero 是一套開源的 Kubernetes 備份與復原工具，本專案在 Ubuntu x86 本機透過 Kind 叢集完整驗證 Velero 的各項備份、復原與遷移功能。

## 架構圖

```
┌─────────────────────────────────────────────────────┐
│                  Ubuntu x86 Host                    │
│                                                     │
│  ┌───────────────────────────────────────────────┐  │
│  │              Kind Cluster (k8s v1.31)         │  │
│  │                                               │  │
│  │  ┌──────────┐  ┌──────────┐  ┌────────────┐  │  │
│  │  │ velero   │  │ demo-app │  │  minio      │  │  │
│  │  │ namespace│  │ demo-db  │  │  namespace  │  │  │
│  │  │          │  │          │  │             │  │  │
│  │  │ • Server │  │ • Nginx  │  │ • S3 Store  │  │  │
│  │  │ • Kopia  │  │ • MySQL  │  │ • Console   │  │  │
│  │  │ • Plugin │  │ • PVC    │  │             │  │  │
│  │  └──────────┘  └──────────┘  └────────────┘  │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

## 技術元件版本

| 元件 | 版本 |
|------|------|
| Velero | v1.17.0 |
| Velero AWS Plugin | v1.11.0 |
| Kind | v0.24.0 |
| Kubernetes | v1.31.0 |
| MinIO | latest |
| MySQL | 8.0 |
| Nginx | 1.27 |
| 上傳器 | Kopia（檔案系統備份） |

## 快速開始

### 前置需求

- Ubuntu 22.04+ x86_64
- Docker 24.0+
- kubectl 1.28+
- Helm 3.12+

### 1. 安裝工具

```bash
# 安裝 Kind v0.24.0
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/

# 安裝 Velero CLI v1.17.0
VELERO_VERSION=v1.17.0
curl -L https://github.com/vmware-tanzu/velero/releases/download/${VELERO_VERSION}/velero-${VELERO_VERSION}-linux-amd64.tar.gz -o velero.tar.gz
tar -xzf velero.tar.gz
sudo mv velero-${VELERO_VERSION}-linux-amd64/velero /usr/local/bin/
rm -rf velero.tar.gz velero-${VELERO_VERSION}-linux-amd64

# 安裝 MinIO Client
curl -LO https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc && sudo mv mc /usr/local/bin/
```

### 2. 建立 Kind 叢集

```bash
kind create cluster --config kind-config.yaml --wait 300s
kubectl cluster-info --context kind-velero-poc
```

### 3. 部署 MinIO

```bash
kubectl apply -f minio-deployment.yaml
kubectl wait --for=condition=ready pod -l app=minio -n minio --timeout=120s

# 建立 Bucket
kubectl port-forward svc/minio -n minio 9000:9000 &
sleep 3
mc alias set myminio http://localhost:9000 minioadmin minioadmin123
mc mb myminio/velero-backup
mc mb myminio/velero-backup-secondary
kill %1
```

### 4. 安裝 Velero

```bash
# 建立憑證檔
cat <<EOF > credentials-velero
[default]
aws_access_key_id=minioadmin
aws_secret_access_key=minioadmin123
EOF

# 安裝 Velero
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.11.0 \
  --bucket velero-backup \
  --secret-file ./credentials-velero \
  --backup-location-config \
    region=minio,s3ForcePathStyle="true",s3Url=http://minio.minio.svc:9000 \
  --snapshot-location-config region=minio \
  --use-node-agent \
  --uploader-type=kopia \
  --wait

# 驗證
velero backup-location get
```

### 5. 部署示範應用程式

```bash
kubectl apply -f demo-nginx.yaml
kubectl apply -f demo-mysql.yaml
kubectl wait --for=condition=ready pod -l app=nginx -n demo-app --timeout=120s
kubectl wait --for=condition=ready pod -l app=mysql -n demo-db --timeout=180s
```

## PoC 測試案例與結果

| # | 功能 | 指令範例 | 結果 |
|---|------|---------|------|
| 1 | On-Demand 全叢集備份 | `velero backup create full-backup --exclude-namespaces kube-system,velero,minio --default-volumes-to-fs-backup=true --wait` | Completed |
| 2 | Namespace 層級備份 | `velero backup create ns-backup --include-namespaces demo-app --wait` | Completed |
| 3 | Label Selector 備份 | `velero backup create label-backup --include-namespaces demo-app --selector app=nginx --wait` | Completed |
| 4 | Resource Type 篩選備份 | `velero backup create res-backup --include-resources deployments,services --wait` | Completed |
| 5 | 排程備份 (Cron) | `velero schedule create my-schedule --schedule="*/5 * * * *" --ttl 1h0m0s` | Completed |
| 6 | Pre/Post Backup Hook | MySQL FLUSH TABLES / UNLOCK TABLES 透過 Pod annotation | Completed |
| 7 | TTL 過期策略 | `velero backup create ttl-test --ttl 10m0s --wait` | Completed |
| 8 | 完整 Disaster Recovery | 刪除 namespace → `velero restore create --from-backup` | Completed |
| 9 | 選擇性復原 | `velero restore create --include-resources configmaps,secrets` | Completed |
| 10 | 跨 Namespace 復原 | `velero restore create --namespace-mappings src:dst` | Completed |
| 11 | Restore Hook | 透過 Restore CR 注入 initContainer 做還原後驗證 | Completed |
| 12 | 多重 BSL | `velero backup-location create secondary --bucket ...` | Completed |
| 13 | Cluster Resource 備份 | `velero backup create --include-cluster-resources=true --include-resources clusterroles` | Completed |
| 14 | 叢集遷移模擬 | 透過 Namespace Mapping 模擬跨叢集遷移 | Completed |

## 專案檔案說明

| 檔案 | 說明 |
|------|------|
| `kind-config.yaml` | Kind 叢集配置（1 control-plane + 2 worker） |
| `minio-deployment.yaml` | MinIO S3 相容儲存部署清單 |
| `demo-nginx.yaml` | Nginx + PVC + ConfigMap + Secret 示範應用 |
| `demo-mysql.yaml` | MySQL StatefulSet 含 Velero Backup Hook |
| `restore-hook-config.yaml` | Restore Hook 設定範例（initContainer 注入） |
| `velero-poc-kind-ubuntu.md` | 完整 PoC 規劃文件與操作步驟 |

## 常用指令參考

### 備份操作

```bash
# 查看所有備份
velero backup get

# 建立備份
velero backup create <name> --include-namespaces <ns> --wait

# 查看備份詳情
velero backup describe <name> --details

# 查看備份 log
velero backup logs <name>
```

### 復原操作

```bash
# 從備份復原
velero restore create <name> --from-backup <backup-name> --wait

# 跨 namespace 復原
velero restore create <name> --from-backup <backup> --namespace-mappings old-ns:new-ns --wait

# 查看復原狀態
velero restore get
velero restore describe <name>
```

### 排程管理

```bash
# 建立排程
velero schedule create <name> --schedule="0 2 * * *" --ttl 168h0m0s

# 查看排程
velero schedule get

# 刪除排程
velero schedule delete <name> --confirm
```

### 除錯

```bash
# Velero Server Log
kubectl logs deployment/velero -n velero -f

# Node Agent (Kopia) Log
kubectl logs daemonset/node-agent -n velero -f

# BSL 狀態
velero backup-location get
```

## 常見問題排查

| 問題 | 排查方式 | 解決方案 |
|------|---------|---------|
| BSL 顯示 Unavailable | `velero backup-location get` | 檢查 MinIO 連線與憑證是否正確 |
| Backup 狀態 PartiallyFailed | `velero backup logs <name>` | 查看哪些資源備份失敗 |
| PV 資料未備份 | `velero backup describe <name>` | 確認使用 `--default-volumes-to-fs-backup` |
| Restore 資源衝突 | `velero restore logs <name>` | 加上 `--existing-resource-policy=update` |
| Hook 執行逾時 | `velero backup logs <name> \| grep hook` | 調整 Pod annotation 中的 timeout 值 |

## Velero Backup Hook 使用方式

在 Pod 的 annotation 中加入 Hook 設定：

```yaml
annotations:
  # 備份前執行（例：鎖定資料庫）
  pre.hook.backup.velero.io/command: '["/bin/bash", "-c", "mysql -u root -p$MYSQL_ROOT_PASSWORD -e \"FLUSH TABLES WITH READ LOCK;\""]'
  pre.hook.backup.velero.io/timeout: "30s"
  # 備份後執行（例：解鎖資料庫）
  post.hook.backup.velero.io/command: '["/bin/bash", "-c", "mysql -u root -p$MYSQL_ROOT_PASSWORD -e \"UNLOCK TABLES;\""]'
  post.hook.backup.velero.io/timeout: "10s"
```

## 清理環境

```bash
# 刪除排程與備份
velero schedule delete --all --confirm
velero backup delete --all --confirm

# 刪除 Kind 叢集
kind delete cluster --name velero-poc

# 清理本機檔案
rm -f credentials-velero
```

## 企業評估摘要

### 優勢

- 完全開源免費（Apache 2.0 授權），無節點數限制
- 透過 Kubernetes API 備份（非直接存取 etcd），安全性更高
- 支援 Kopia/Restic 檔案系統備份，不依賴 CSI Snapshot
- Plugin 架構可擴充，支援 AWS / Azure / GCP 等主流雲端

### 限制

- 純 CLI 操作，無內建 Web UI
- 應用層一致性備份需自行實作 Hook
- RBAC 與多租戶管理需額外配置
- 加密依賴儲存端（MinIO SSE）或 Kopia 內建加密

---

> **PoC 完成日期**：2026-02-10
> **環境**：Ubuntu x86 / Kind v0.24.0 / Velero v1.17.0
