# Velero PoC — Ubuntu x86 Kind Cluster

> **版本**：Velero v1.17.x ｜ Kind v0.24.x ｜ MinIO (S3-Compatible)
> **目標**：在本機 Ubuntu x86 上透過 Kind 叢集，完整驗證 Velero 所有備份復原功能

---

## 1. PoC 架構總覽

```
┌─────────────────────────────────────────────────────┐
│                  Ubuntu x86 Host                    │
│                                                     │
│  ┌───────────────────────────────────────────────┐  │
│  │              Kind Cluster (k8s)               │  │
│  │                                               │  │
│  │  ┌──────────┐  ┌──────────┐  ┌────────────┐  │  │
│  │  │ velero   │  │ demo-app │  │  MinIO      │  │  │
│  │  │ namespace│  │ namespace│  │  namespace  │  │  │
│  │  │          │  │          │  │             │  │  │
│  │  │ • Server │  │ • Nginx  │  │ • S3 Store  │  │  │
│  │  │ • Kopia  │  │ • MySQL  │  │ • Console   │  │  │
│  │  │ • Plugin │  │ • PVC    │  │             │  │  │
│  │  └──────────┘  └──────────┘  └────────────┘  │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

---

## 2. 環境需求

| 項目 | 最低需求 | 建議配置 |
|------|---------|---------|
| OS | Ubuntu 22.04+ x86_64 | Ubuntu 24.04 LTS |
| CPU | 4 cores | 8 cores |
| RAM | 8 GB | 16 GB |
| Disk | 40 GB | 80 GB SSD |
| Docker | 24.0+ | 27.x |
| kubectl | 1.28+ | 1.30+ |
| Helm | 3.12+ | 3.16+ |
| Kind | 0.22+ | 0.24+ |

---

## 3. 基礎環境安裝

### 3.1 安裝 Docker

```bash
# 移除舊版本
sudo apt-get remove docker docker-engine docker.io containerd runc

# 安裝依賴
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release

# 新增 Docker GPG key 與 repo
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 免 sudo 執行 docker
sudo usermod -aG docker $USER
newgrp docker
```

### 3.2 安裝 kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

### 3.3 安裝 Kind

```bash
[ $(uname -m) = x86_64 ] && \
  curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```

### 3.4 安裝 Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

### 3.5 安裝 Velero CLI

```bash
VELERO_VERSION=v1.17.0
curl -L https://github.com/vmware-tanzu/velero/releases/download/${VELERO_VERSION}/velero-${VELERO_VERSION}-linux-amd64.tar.gz -o velero.tar.gz
tar -xvf velero.tar.gz
sudo mv velero-${VELERO_VERSION}-linux-amd64/velero /usr/local/bin/
velero version --client-only
```

---

## 4. 建立 Kind Cluster

### 4.1 Kind 叢集配置檔

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: velero-poc
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 30000
        hostPort: 9001
        protocol: TCP
      - containerPort: 30001
        hostPort: 9000
        protocol: TCP
  - role: worker
  - role: worker
```

### 4.2 建立叢集

```bash
kind create cluster --config kind-config.yaml --wait 300s
kubectl cluster-info --context kind-velero-poc
kubectl get nodes
```

---

## 5. 部署 MinIO（S3 相容儲存）

### 5.1 MinIO 部署清單

```yaml
# minio-deployment.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: minio
---
apiVersion: v1
kind: Secret
metadata:
  name: minio-credentials
  namespace: minio
type: Opaque
stringData:
  MINIO_ROOT_USER: "minioadmin"
  MINIO_ROOT_PASSWORD: "minioadmin123"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: minio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
        - name: minio
          image: minio/minio:latest
          command: ["minio", "server", "/data", "--console-address", ":9001"]
          envFrom:
            - secretRef:
                name: minio-credentials
          ports:
            - containerPort: 9000
              name: api
            - containerPort: 9001
              name: console
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: minio
spec:
  type: NodePort
  selector:
    app: minio
  ports:
    - name: api
      port: 9000
      targetPort: 9000
      nodePort: 30001
    - name: console
      port: 9001
      targetPort: 9001
      nodePort: 30000
```

### 5.2 建立 MinIO Bucket

```bash
kubectl apply -f minio-deployment.yaml
kubectl wait --for=condition=ready pod -l app=minio -n minio --timeout=120s

# 安裝 mc (MinIO Client)
curl -LO https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc && sudo mv mc /usr/local/bin/

# 透過 port-forward 建立 bucket
kubectl port-forward svc/minio -n minio 9000:9000 &
sleep 3

mc alias set myminio http://localhost:9000 minioadmin minioadmin123
mc mb myminio/velero-backup
mc mb myminio/velero-backup-secondary
mc ls myminio

# 停止 port-forward
kill %1
```

---

## 6. 安裝 Velero

### 6.1 建立憑證檔案

```bash
cat <<EOF > credentials-velero
[default]
aws_access_key_id=minioadmin
aws_secret_access_key=minioadmin123
EOF
```

### 6.2 使用 Velero CLI 安裝

```bash
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
```

### 6.3 驗證安裝

```bash
kubectl get pods -n velero
kubectl get backupstoragelocation -n velero
velero backup-location get
```

預期輸出：

```
NAME      PROVIDER   BUCKET/PREFIX    PHASE       LAST VALIDATED   ACCESS MODE   DEFAULT
default   aws        velero-backup    Available   <timestamp>      ReadWrite     true
```

---

## 7. 部署示範應用程式

### 7.1 Nginx + PVC 應用

```yaml
# demo-nginx.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-app
  labels:
    app: demo
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-logs
  namespace: demo-app
  labels:
    app: nginx
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: demo-app
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          volumeMounts:
            - name: logs
              mountPath: /var/log/nginx
      volumes:
        - name: logs
          persistentVolumeClaim:
            claimName: nginx-logs
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: demo-app
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: demo-app
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: demo-app
type: Opaque
stringData:
  DB_PASSWORD: "s3cret-passw0rd"
  API_KEY: "poc-api-key-12345"
```

### 7.2 MySQL StatefulSet（用於 Hook 測試）

```yaml
# demo-mysql.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-db
  labels:
    app: database
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: demo-db
stringData:
  MYSQL_ROOT_PASSWORD: "rootpass123"
  MYSQL_DATABASE: "pocdb"
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: demo-db
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
      annotations:
        # Velero Backup Hook: Pre-backup flush & lock
        pre.hook.backup.velero.io/command: '["/bin/bash", "-c", "mysql -u root -p$MYSQL_ROOT_PASSWORD -e \"FLUSH TABLES WITH READ LOCK; SELECT SLEEP(5);\""]'
        pre.hook.backup.velero.io/timeout: "30s"
        # Velero Backup Hook: Post-backup unlock
        post.hook.backup.velero.io/command: '["/bin/bash", "-c", "mysql -u root -p$MYSQL_ROOT_PASSWORD -e \"UNLOCK TABLES;\""]'
        post.hook.backup.velero.io/timeout: "10s"
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          envFrom:
            - secretRef:
                name: mysql-secret
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 2Gi
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: demo-db
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
    - port: 3306
```

### 7.3 部署應用

```bash
kubectl apply -f demo-nginx.yaml
kubectl apply -f demo-mysql.yaml

# 等待所有 Pod 就緒
kubectl wait --for=condition=ready pod -l app=nginx -n demo-app --timeout=120s
kubectl wait --for=condition=ready pod -l app=mysql -n demo-db --timeout=180s

# 寫入測試資料
kubectl exec -n demo-app deploy/nginx -- \
  sh -c 'echo "PoC test data - $(date)" > /var/log/nginx/poc-test.log'

kubectl exec -n demo-db sts/mysql -- \
  mysql -u root -prootpass123 -e \
  "USE pocdb; CREATE TABLE IF NOT EXISTS poc_data (id INT AUTO_INCREMENT PRIMARY KEY, msg VARCHAR(255), ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP); INSERT INTO poc_data (msg) VALUES ('Velero PoC initial data');"
```

---

## 8. PoC 測試案例

### 測試案例 1：On-Demand 全叢集備份

```bash
# 備份所有 namespace（排除系統 namespace）
velero backup create full-backup-01 \
  --exclude-namespaces kube-system,kube-public,kube-node-lease,velero,minio \
  --include-cluster-resources=true \
  --default-volumes-to-fs-backup=true \
  --wait

# 查看備份狀態
velero backup describe full-backup-01 --details
velero backup logs full-backup-01
```

**預期結果**：Phase = Completed，Items backed up > 0

---

### 測試案例 2：Namespace 層級備份

```bash
# 僅備份 demo-app namespace
velero backup create ns-backup-demo-app \
  --include-namespaces demo-app \
  --default-volumes-to-fs-backup=true \
  --wait

velero backup describe ns-backup-demo-app
```

---

### 測試案例 3：Label Selector 備份

```bash
# 僅備份帶有 app=nginx label 的資源
velero backup create label-backup-nginx \
  --include-namespaces demo-app \
  --selector app=nginx \
  --wait

velero backup describe label-backup-nginx
```

---

### 測試案例 4：Resource Type 篩選備份

```bash
# 僅備份特定資源類型
velero backup create resource-backup \
  --include-namespaces demo-app \
  --include-resources deployments,services,configmaps,secrets \
  --wait

velero backup describe resource-backup
```

---

### 測試案例 5：排程備份（Scheduled Backup）

```bash
# 每 5 分鐘執行一次（PoC 測試用，正式環境建議更長間隔）
velero schedule create poc-schedule-5min \
  --schedule="*/5 * * * *" \
  --include-namespaces demo-app,demo-db \
  --default-volumes-to-fs-backup=true \
  --ttl 1h0m0s

# 每日凌晨 2 點備份
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces demo-app,demo-db \
  --default-volumes-to-fs-backup=true \
  --ttl 168h0m0s

# 查看排程
velero schedule get

# 等待排程觸發，檢查備份
sleep 360
velero backup get | grep poc-schedule
```

---

### 測試案例 6：Pre/Post Backup Hook（MySQL）

```bash
# 帶 Hook 的備份（MySQL 已透過 annotation 設定 Hook）
velero backup create db-backup-with-hook \
  --include-namespaces demo-db \
  --default-volumes-to-fs-backup=true \
  --wait

# 查看 Hook 執行紀錄
velero backup describe db-backup-with-hook --details
velero backup logs db-backup-with-hook | grep -i hook
```

**預期結果**：Log 中可看到 pre-hook 與 post-hook 執行成功

---

### 測試案例 7：備份 TTL（過期策略）

```bash
# TTL = 10 分鐘（測試用）
velero backup create ttl-test-backup \
  --include-namespaces demo-app \
  --ttl 10m0s \
  --wait

# 查看過期時間
velero backup describe ttl-test-backup | grep -E "TTL|Expiration"

# 等待過期後確認被清理
sleep 660
velero backup get | grep ttl-test
```

---

### 測試案例 8：完整復原（Disaster Recovery）

```bash
# Step 1: 記錄目前狀態
kubectl get all -n demo-app
kubectl exec -n demo-app deploy/nginx -- cat /var/log/nginx/poc-test.log

# Step 2: 模擬災難 - 刪除整個 namespace
kubectl delete namespace demo-app

# Step 3: 確認已刪除
kubectl get ns demo-app  # 應回傳 NotFound

# Step 4: 從備份復原
velero restore create restore-demo-app \
  --from-backup ns-backup-demo-app \
  --wait

# Step 5: 驗證復原
velero restore describe restore-demo-app
kubectl get all -n demo-app
kubectl exec -n demo-app deploy/nginx -- cat /var/log/nginx/poc-test.log
```

**預期結果**：所有資源與 PV 資料完整恢復

---

### 測試案例 9：選擇性復原（部分資源）

```bash
# 僅復原 ConfigMap 和 Secret
velero restore create partial-restore \
  --from-backup full-backup-01 \
  --include-namespaces demo-app \
  --include-resources configmaps,secrets \
  --wait

velero restore describe partial-restore
```

---

### 測試案例 10：跨 Namespace 復原（Namespace Mapping）

```bash
# 將 demo-app 復原到 demo-app-clone namespace
velero restore create ns-mapping-restore \
  --from-backup ns-backup-demo-app \
  --namespace-mappings demo-app:demo-app-clone \
  --wait

# 驗證
kubectl get all -n demo-app-clone
```

---

### 測試案例 11：Restore Hook

```yaml
# restore-hook-config.yaml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: restore-with-hook
  namespace: velero
spec:
  backupName: db-backup-with-hook
  includedNamespaces:
    - demo-db
  hooks:
    resources:
      - name: mysql-post-restore
        includedNamespaces:
          - demo-db
        labelSelector:
          matchLabels:
            app: mysql
        postHooks:
          - init:
              initContainers:
                - name: post-restore-check
                  image: mysql:8.0
                  command:
                    - /bin/bash
                    - -c
                    - |
                      echo "Post-restore validation started"
                      sleep 10
                      echo "Post-restore validation completed"
```

```bash
kubectl apply -f restore-hook-config.yaml
velero restore describe restore-with-hook
```

---

### 測試案例 12：多重 Backup Storage Location

```bash
# 新增第二個 BSL
velero backup-location create secondary \
  --provider aws \
  --bucket velero-backup-secondary \
  --config region=minio,s3ForcePathStyle="true",s3Url=http://minio.minio.svc:9000 \
  --credential=secret-name=velero,key=cloud

# 備份到第二個 BSL
velero backup create secondary-bsl-backup \
  --include-namespaces demo-app \
  --storage-location secondary \
  --wait

# 驗證
velero backup-location get
velero backup get | grep secondary
```

---

### 測試案例 13：Cluster Resource 備份

```bash
# 備份 CRD、ClusterRole、ClusterRoleBinding 等
velero backup create cluster-resources-backup \
  --include-cluster-resources=true \
  --include-resources clusterroles,clusterrolebindings,customresourcedefinitions \
  --wait

velero backup describe cluster-resources-backup --details
```

---

### 測試案例 14：叢集遷移模擬

```bash
# Step 1: 在來源叢集完整備份
velero backup create migration-backup \
  --include-namespaces demo-app,demo-db \
  --include-cluster-resources=true \
  --default-volumes-to-fs-backup=true \
  --wait

# Step 2: 建立第二個 Kind Cluster（模擬目標叢集）
kind create cluster --name velero-target --wait 300s

# Step 3: 在目標叢集安裝 MinIO 與 Velero（指向相同 BSL）
# （重複步驟 5-6，但指向同一個 MinIO）

# Step 4: 在目標叢集復原
kubectl config use-context kind-velero-target
velero restore create migration-restore \
  --from-backup migration-backup \
  --wait

# Step 5: 驗證
kubectl get all -n demo-app
kubectl get all -n demo-db
```

---

## 9. 監控與除錯

### 9.1 檢查備份狀態

```bash
# 列出所有備份
velero backup get

# 詳細描述
velero backup describe <backup-name> --details

# 查看 log
velero backup logs <backup-name>

# 查看 restore 狀態
velero restore get
velero restore describe <restore-name>
velero restore logs <restore-name>
```

### 9.2 Velero Server Log

```bash
kubectl logs deployment/velero -n velero -f
```

### 9.3 Node Agent (Kopia) Log

```bash
kubectl logs daemonset/node-agent -n velero -f
```

### 9.4 常見問題排查

| 問題 | 排查指令 | 解決方案 |
|------|---------|---------|
| BSL 狀態 Unavailable | `velero backup-location get` | 檢查 MinIO 連線與憑證 |
| Backup PartiallyFailed | `velero backup logs <name>` | 查看哪些資源失敗 |
| PV 資料未備份 | `velero backup describe <name>` | 確認 `--default-volumes-to-fs-backup` |
| Restore 衝突 | `velero restore logs <name>` | 使用 `--existing-resource-policy=update` |
| Hook 逾時 | `velero backup logs <name> \| grep hook` | 調整 timeout annotation |

---

## 10. 清理環境

```bash
# 刪除排程
velero schedule delete --all --confirm

# 刪除備份
velero backup delete --all --confirm

# 刪除 Kind Cluster
kind delete cluster --name velero-poc
kind delete cluster --name velero-target

# 清理本機檔案
rm -f credentials-velero velero.tar.gz
```

---

## 11. 功能矩陣總覽

| # | 功能 | 測試案例 | 狀態 |
|---|------|---------|------|
| 1 | On-Demand 全叢集備份 | 案例 1 | ☐ |
| 2 | Namespace 層級備份 | 案例 2 | ☐ |
| 3 | Label Selector 備份 | 案例 3 | ☐ |
| 4 | Resource Type 篩選 | 案例 4 | ☐ |
| 5 | 排程備份 (Cron) | 案例 5 | ☐ |
| 6 | Pre/Post Backup Hook | 案例 6 | ☐ |
| 7 | TTL 過期策略 | 案例 7 | ☐ |
| 8 | 完整 Disaster Recovery | 案例 8 | ☐ |
| 9 | 選擇性復原 | 案例 9 | ☐ |
| 10 | 跨 Namespace 復原 | 案例 10 | ☐ |
| 11 | Restore Hook | 案例 11 | ☐ |
| 12 | 多重 BSL | 案例 12 | ☐ |
| 13 | Cluster Resource 備份 | 案例 13 | ☐ |
| 14 | 叢集遷移 | 案例 14 | ☐ |
| 15 | Kopia 檔案系統備份 | 案例 1, 5, 6, 8 | ☐ |
| 16 | MinIO S3 相容整合 | 全部案例 | ☐ |

---

## 12. 企業評估要點

### 優勢

- 完全開源免費（Apache 2.0），無節點數限制
- 社群活躍，VMware/Broadcom 持續維護
- 透過 Kubernetes API 備份（非直接存取 etcd），更安全
- 支援 Kopia/Restic 檔案系統備份，不依賴 CSI Snapshot
- Plugin 架構可擴充，支援主流雲端 Provider

### 限制

- 純 CLI 操作，無內建 Web UI（可搭配第三方 Dashboard）
- 不支援應用感知（Application-Aware）的一致性備份（需自行實作 Hook）
- RBAC 與多租戶管理需額外配置
- 無內建加密（依賴儲存端加密或 Kopia 加密）

### 金融業合規建議

- 搭配 MinIO + TLS + Server-Side Encryption 滿足資料加密需求
- 搭配 PostgreSQL WAL Archiving 或 MySQL binlog 做資料庫層級備份
- 透過 CronJob + Velero Schedule 實現 RPO < 1hr
- 建議 BSL 設定 Immutable Bucket 防止勒索軟體竄改

---

> **文件版本**：v1.0
> **建立日期**：2026-02-10
> **適用環境**：PoC / 非正式環境
