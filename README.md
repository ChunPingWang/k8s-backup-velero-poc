# Velero Backup & Restore PoC — Kind Cluster on Ubuntu

Velero 是一套開源的 Kubernetes 備份與復原工具，本專案在 Ubuntu x86 本機透過 Kind 叢集完整驗證 Velero 的各項備份、復原與遷移功能。

---

## 目錄

- [Kubernetes 基礎架構](#kubernetes-基礎架構)
- [為什麼需要 K8s 備份](#為什麼需要-k8s-備份)
- [Velero 架構與運作原理](#velero-架構與運作原理)
- [PoC 架構](#poc-架構)
- [技術元件版本](#技術元件版本)
- [快速開始](#快速開始)
- [PoC 測試案例與結果](#poc-測試案例與結果)
- [備份與復原策略指南](#備份與復原策略指南)
- [常用指令參考](#常用指令參考)
- [常見問題排查](#常見問題排查)
- [企業評估摘要](#企業評估摘要)
- [清理環境](#清理環境)

---

## Kubernetes 基礎架構

如果你是 Kubernetes 初學者，以下是 K8s 的核心架構說明。

### K8s 叢集架構

```mermaid
graph TB
    subgraph CP["Control Plane（控制平面）"]
        API["API Server<br/>叢集的入口閘道"]
        ETCD["etcd<br/>分散式鍵值資料庫<br/>儲存所有叢集狀態"]
        SCHED["Scheduler<br/>決定 Pod 在哪個 Node 執行"]
        CM["Controller Manager<br/>維持資源的期望狀態"]
    end

    subgraph W1["Worker Node 1"]
        KL1["kubelet<br/>管理 Pod 生命週期"]
        KP1["kube-proxy<br/>處理網路規則"]
        POD1["Pod A"]
        POD2["Pod B"]
    end

    subgraph W2["Worker Node 2"]
        KL2["kubelet"]
        KP2["kube-proxy"]
        POD3["Pod C"]
        POD4["Pod D"]
    end

    USER["kubectl / 使用者"] -->|REST API| API
    API --> ETCD
    API --> SCHED
    API --> CM
    SCHED --> KL1
    SCHED --> KL2
    CM --> API
    KL1 --> POD1
    KL1 --> POD2
    KL2 --> POD3
    KL2 --> POD4
```

### K8s 核心資源關係

```mermaid
graph LR
    subgraph "Namespace（命名空間）"
        DEP["Deployment<br/>定義應用的期望狀態"]
        RS["ReplicaSet<br/>確保 Pod 副本數"]
        POD["Pod<br/>最小部署單元"]
        SVC["Service<br/>穩定的網路端點"]
        CM["ConfigMap<br/>設定資料"]
        SEC["Secret<br/>敏感資訊"]
        PVC["PVC<br/>持久化儲存請求"]
    end

    PV["PersistentVolume<br/>實際儲存資源"]

    DEP -->|管理| RS
    RS -->|管理| POD
    SVC -->|選取| POD
    POD -->|掛載| CM
    POD -->|掛載| SEC
    POD -->|綁定| PVC
    PVC -->|綁定| PV
```

### K8s 資源分類

```mermaid
graph TB
    subgraph CL["Cluster-Scoped（叢集層級）"]
        NODE["Node"]
        NS["Namespace"]
        CR["ClusterRole"]
        CRB["ClusterRoleBinding"]
        PV2["PersistentVolume"]
        CRD["CustomResourceDefinition"]
    end

    subgraph NP["Namespace-Scoped（命名空間層級）"]
        DEP2["Deployment"]
        STS["StatefulSet"]
        SVC2["Service"]
        CM2["ConfigMap"]
        SEC2["Secret"]
        PVC2["PVC"]
        POD2["Pod"]
    end

    CL ---|包含| NP
```

> **初學者筆記**：
> - **Pod** 是 K8s 最小部署單元，通常包含一個容器
> - **Deployment** 管理無狀態應用（如 Web Server）
> - **StatefulSet** 管理有狀態應用（如資料庫）
> - **PVC/PV** 提供持久化儲存，Pod 刪除後資料仍保留
> - **Namespace** 用來隔離不同環境或團隊的資源

---

## 為什麼需要 K8s 備份

### K8s 中需要備份的資料

```mermaid
graph TB
    subgraph BACKUP["K8s 備份範圍"]
        subgraph META["Kubernetes 資源（Metadata）"]
            D1["Deployment / StatefulSet"]
            D2["Service / Ingress"]
            D3["ConfigMap / Secret"]
            D4["RBAC / NetworkPolicy"]
            D5["CRD / Custom Resources"]
        end

        subgraph DATA["持久化資料（Volume Data）"]
            V1["資料庫檔案<br/>MySQL / PostgreSQL"]
            V2["應用程式 Log"]
            V3["上傳檔案 / 靜態資源"]
            V4["快取資料"]
        end

        subgraph CLUSTER["叢集層級資源"]
            C1["ClusterRole / Binding"]
            C2["Namespace"]
            C3["StorageClass"]
            C4["CustomResourceDefinition"]
        end
    end

    style META fill:#e1f5fe
    style DATA fill:#fff3e0
    style CLUSTER fill:#f3e5f5
```

### 常見災難場景

```mermaid
graph LR
    subgraph DISASTERS["災難場景"]
        H1["人為誤操作<br/>kubectl delete ns"]
        H2["應用程式 Bug<br/>資料損毀"]
        H3["叢集升級失敗<br/>版本不相容"]
        H4["硬體故障<br/>Node 損壞"]
        H5["資安事件<br/>勒索軟體攻擊"]
    end

    subgraph SOLUTIONS["備份復原策略"]
        S1["On-Demand 備份<br/>重大變更前"]
        S2["排程備份<br/>定期自動執行"]
        S3["跨叢集遷移<br/>DR Site 復原"]
    end

    H1 --> S1
    H2 --> S1
    H3 --> S2
    H4 --> S3
    H5 --> S3
```

---

## Velero 架構與運作原理

### Velero 元件架構

```mermaid
graph TB
    subgraph CLIENT["使用者端"]
        CLI["Velero CLI<br/>命令列工具"]
    end

    subgraph CLUSTER["Kubernetes Cluster"]
        subgraph VNS["velero namespace"]
            VS["Velero Server<br/>（Deployment）<br/>核心控制器"]
            NA["Node Agent<br/>（DaemonSet）<br/>Kopia 檔案備份"]
            BSL["BackupStorageLocation<br/>（CR）指向 S3"]
            VSL["VolumeSnapshotLocation<br/>（CR）快照設定"]
        end

        subgraph APP["應用 namespace"]
            APOD["Application Pods"]
            APVC["PersistentVolumeClaim"]
        end
    end

    subgraph STORAGE["外部儲存"]
        S3["S3 / MinIO<br/>物件儲存"]
    end

    CLI -->|"kubectl API"| VS
    VS -->|"備份 K8s 資源"| APP
    NA -->|"Kopia 備份 PV 資料"| APVC
    VS -->|"上傳備份"| S3
    NA -->|"上傳 PV 資料"| S3
    BSL -.->|"指向"| S3
```

### 備份流程（Backup Workflow）

```mermaid
sequenceDiagram
    participant User as 使用者
    participant CLI as Velero CLI
    participant API as K8s API Server
    participant VS as Velero Server
    participant NA as Node Agent (Kopia)
    participant S3 as MinIO (S3)

    User->>CLI: velero backup create my-backup
    CLI->>API: 建立 Backup CR
    API->>VS: Watch 到新 Backup 資源

    rect rgb(230, 245, 255)
        Note over VS: Phase 1: 備份 K8s 資源
        VS->>API: 列出所有指定資源
        VS->>VS: 序列化為 JSON/tarball
    end

    rect rgb(255, 243, 224)
        Note over VS,NA: Phase 2: 執行 Backup Hook
        VS->>API: 執行 Pre-hook（如 DB Lock）
        VS->>NA: 觸發 PV 檔案備份
        NA->>S3: Kopia 上傳 PV 資料
        VS->>API: 執行 Post-hook（如 DB Unlock）
    end

    rect rgb(232, 245, 233)
        Note over VS,S3: Phase 3: 上傳備份
        VS->>S3: 上傳 K8s 資源 tarball
        VS->>API: 更新 Backup CR 狀態 = Completed
    end

    VS-->>CLI: 備份完成
    CLI-->>User: Backup completed ✅
```

### 復原流程（Restore Workflow）

```mermaid
sequenceDiagram
    participant User as 使用者
    participant CLI as Velero CLI
    participant API as K8s API Server
    participant VS as Velero Server
    participant NA as Node Agent (Kopia)
    participant S3 as MinIO (S3)

    User->>CLI: velero restore create --from-backup my-backup
    CLI->>API: 建立 Restore CR
    API->>VS: Watch 到新 Restore 資源

    rect rgb(230, 245, 255)
        Note over VS,S3: Phase 1: 下載備份
        VS->>S3: 下載 K8s 資源 tarball
        VS->>VS: 解析備份內容
    end

    rect rgb(255, 243, 224)
        Note over VS: Phase 2: 還原 K8s 資源
        VS->>API: 依序建立 Namespace
        VS->>API: 建立 PVC → PV
        VS->>API: 建立 Deployment / StatefulSet
        VS->>API: 建立 Service / ConfigMap / Secret
    end

    rect rgb(232, 245, 233)
        Note over NA,S3: Phase 3: 還原 PV 資料
        NA->>S3: Kopia 下載 PV 資料
        NA->>API: 寫入 PV
    end

    rect rgb(243, 229, 245)
        Note over VS: Phase 4: 執行 Restore Hook
        VS->>API: 注入 initContainer
        Note over VS: 執行還原後驗證
    end

    VS->>API: 更新 Restore CR 狀態 = Completed
    VS-->>CLI: 復原完成
    CLI-->>User: Restore completed ✅
```

### Velero 備份層級

```mermaid
graph TB
    subgraph LEVELS["Velero 備份粒度"]
        L1["全叢集備份<br/>--include-cluster-resources=true<br/>所有 Namespace + Cluster 資源"]
        L2["Namespace 層級<br/>--include-namespaces ns1,ns2<br/>指定 Namespace 內所有資源"]
        L3["Label 篩選<br/>--selector app=nginx<br/>符合 Label 的資源"]
        L4["Resource Type<br/>--include-resources deployments<br/>指定資源類型"]
    end

    L1 -->|範圍最大| L2
    L2 -->|更精確| L3
    L3 -->|最精確| L4

    style L1 fill:#e8eaf6
    style L2 fill:#e3f2fd
    style L3 fill:#e0f7fa
    style L4 fill:#e8f5e9
```

---

## PoC 架構

```mermaid
graph TB
    subgraph HOST["Ubuntu x86 Host"]
        subgraph KIND["Kind Cluster — k8s v1.31"]
            subgraph VEL["velero namespace"]
                VSRV["Velero Server"]
                NAGENT["Node Agent<br/>（Kopia）"]
                AWSP["AWS Plugin"]
            end

            subgraph APP["demo-app namespace"]
                NGX["Nginx Deployment<br/>× 2 replicas"]
                NPVC["PVC: nginx-logs<br/>1Gi"]
                ACFG["ConfigMap: app-config"]
                ASEC["Secret: app-secret"]
            end

            subgraph DB["demo-db namespace"]
                MYSQL["MySQL StatefulSet<br/>含 Backup Hook"]
                MPVC["PVC: mysql-data<br/>2Gi"]
            end

            subgraph MINIO["minio namespace"]
                MS3["MinIO Server<br/>S3 API :9000"]
                MCO["MinIO Console<br/>:9001"]
            end
        end
    end

    VSRV -->|"備份 K8s 資源"| APP
    VSRV -->|"備份 K8s 資源"| DB
    NAGENT -->|"Kopia 備份"| NPVC
    NAGENT -->|"Kopia 備份"| MPVC
    VSRV -->|"S3 協定"| MS3

    style VEL fill:#e8eaf6
    style APP fill:#e3f2fd
    style DB fill:#fff3e0
    style MINIO fill:#e8f5e9
```

## 技術元件版本

| 元件 | 版本 | 用途 |
|------|------|------|
| Velero | v1.17.0 | K8s 備份復原核心 |
| Velero AWS Plugin | v1.11.0 | S3 相容儲存介面 |
| Kind | v0.24.0 | 本機 K8s 叢集 |
| Kubernetes | v1.31.0 | 容器編排平台 |
| MinIO | latest | S3 相容物件儲存 |
| Kopia | (內建) | 檔案系統層級備份引擎 |
| MySQL | 8.0 | 有狀態示範應用（含 Hook） |
| Nginx | 1.27 | 無狀態示範應用（含 PVC） |

---

## 快速開始

### 前置需求

| 項目 | 最低需求 | 建議配置 |
|------|---------|---------|
| OS | Ubuntu 22.04+ x86_64 | Ubuntu 24.04 LTS |
| CPU | 4 cores | 8 cores |
| RAM | 8 GB | 16 GB |
| Disk | 40 GB | 80 GB SSD |
| Docker | 24.0+ | 27.x |
| kubectl | 1.28+ | 1.30+ |
| Helm | 3.12+ | 3.16+ |

### Step 1 — 安裝工具

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

### Step 2 — 建立 Kind 叢集

```bash
kind create cluster --config kind-config.yaml --wait 300s
kubectl cluster-info --context kind-velero-poc
kubectl get nodes
```

### Step 3 — 部署 MinIO

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

### Step 4 — 安裝 Velero

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

# 驗證安裝
kubectl get pods -n velero
velero backup-location get
# 預期：Phase = Available
```

### Step 5 — 部署示範應用程式

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

## PoC 測試案例與結果

### 測試結果總覽

| # | 功能 | 測試狀態 |
|---|------|---------|
| 1 | On-Demand 全叢集備份 | Completed |
| 2 | Namespace 層級備份 | Completed |
| 3 | Label Selector 備份 | Completed |
| 4 | Resource Type 篩選備份 | Completed |
| 5 | 排程備份 (Cron) | Completed |
| 6 | Pre/Post Backup Hook (MySQL) | Completed |
| 7 | TTL 過期策略 | Completed |
| 8 | 完整 Disaster Recovery | Completed |
| 9 | 選擇性復原 | Completed |
| 10 | 跨 Namespace 復原 | Completed |
| 11 | Restore Hook | Completed |
| 12 | 多重 Backup Storage Location | Completed |
| 13 | Cluster Resource 備份 | Completed |
| 14 | 叢集遷移模擬 | Completed |

### 測試案例 1 — On-Demand 全叢集備份

備份所有 Namespace（排除系統 Namespace）。

```bash
velero backup create full-backup-01 \
  --exclude-namespaces kube-system,kube-public,kube-node-lease,velero,minio \
  --include-cluster-resources=true \
  --default-volumes-to-fs-backup=true \
  --wait

velero backup describe full-backup-01 --details
```

### 測試案例 2 — Namespace 層級備份

僅備份 `demo-app` namespace。

```bash
velero backup create ns-backup-demo-app \
  --include-namespaces demo-app \
  --default-volumes-to-fs-backup=true \
  --wait
```

### 測試案例 3 — Label Selector 備份

僅備份帶有 `app=nginx` label 的資源。

```bash
velero backup create label-backup-nginx \
  --include-namespaces demo-app \
  --selector app=nginx \
  --wait
```

### 測試案例 4 — Resource Type 篩選備份

僅備份特定資源類型。

```bash
velero backup create resource-backup \
  --include-namespaces demo-app \
  --include-resources deployments,services,configmaps,secrets \
  --wait
```

### 測試案例 5 — 排程備份 (Cron)

```bash
# 每 5 分鐘執行備份（PoC 測試用）
velero schedule create poc-schedule-5min \
  --schedule="*/5 * * * *" \
  --include-namespaces demo-app,demo-db \
  --default-volumes-to-fs-backup=true \
  --ttl 1h0m0s

# 每日凌晨 2 點備份（正式建議）
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces demo-app,demo-db \
  --default-volumes-to-fs-backup=true \
  --ttl 168h0m0s

velero schedule get
```

### 測試案例 6 — Pre/Post Backup Hook (MySQL)

MySQL 透過 Pod annotation 設定 Backup Hook，在備份前鎖定資料表、備份後解鎖。

```yaml
# demo-mysql.yaml 中的 annotation 片段
annotations:
  pre.hook.backup.velero.io/command: '["/bin/bash", "-c", "mysql -u root -p$MYSQL_ROOT_PASSWORD -e \"FLUSH TABLES WITH READ LOCK; SELECT SLEEP(5);\""]'
  pre.hook.backup.velero.io/timeout: "30s"
  post.hook.backup.velero.io/command: '["/bin/bash", "-c", "mysql -u root -p$MYSQL_ROOT_PASSWORD -e \"UNLOCK TABLES;\""]'
  post.hook.backup.velero.io/timeout: "10s"
```

```bash
velero backup create db-backup-with-hook \
  --include-namespaces demo-db \
  --default-volumes-to-fs-backup=true \
  --wait
```

Hook 執行流程：

```mermaid
sequenceDiagram
    participant VS as Velero Server
    participant MySQL as MySQL Pod

    VS->>MySQL: Pre-hook: FLUSH TABLES WITH READ LOCK
    Note over MySQL: 資料表鎖定，確保一致性
    VS->>VS: 備份 K8s 資源 + PV 資料
    VS->>MySQL: Post-hook: UNLOCK TABLES
    Note over MySQL: 資料表解鎖，恢復正常服務
```

### 測試案例 7 — TTL 過期策略

```bash
velero backup create ttl-test-backup \
  --include-namespaces demo-app \
  --ttl 10m0s \
  --wait

# 確認 TTL
velero backup describe ttl-test-backup | grep -E "TTL|Expiration"
# 10 分鐘後備份會自動被 GC 清理
```

### 測試案例 8 — 完整 Disaster Recovery

```mermaid
sequenceDiagram
    participant User as 操作者
    participant K8s as Kubernetes
    participant Velero as Velero

    Note over User,Velero: Step 1: 確認現有狀態
    User->>K8s: kubectl get all -n demo-app ✅

    Note over User,K8s: Step 2: 模擬災難
    User->>K8s: kubectl delete namespace demo-app 💥
    User->>K8s: kubectl get ns demo-app → NotFound

    Note over User,Velero: Step 3: 從備份復原
    User->>Velero: velero restore create --from-backup ns-backup-demo-app
    Velero->>K8s: 還原 Namespace + 所有資源

    Note over User,K8s: Step 4: 驗證復原
    User->>K8s: kubectl get all -n demo-app ✅
    Note over K8s: 所有 Deployment, Service, PVC 復原完成
```

```bash
# 記錄目前狀態
kubectl get all -n demo-app

# 模擬災難 - 刪除 namespace
kubectl delete namespace demo-app

# 確認已刪除
kubectl get ns demo-app  # NotFound

# 從備份復原
velero restore create restore-demo-app \
  --from-backup ns-backup-demo-app \
  --wait

# 驗證復原
kubectl get all -n demo-app
```

### 測試案例 9 — 選擇性復原

僅復原 ConfigMap 和 Secret。

```bash
velero restore create partial-restore \
  --from-backup full-backup-01 \
  --include-namespaces demo-app \
  --include-resources configmaps,secrets \
  --wait
```

### 測試案例 10 — 跨 Namespace 復原

將 `demo-app` 復原到 `demo-app-clone` namespace。

```bash
velero restore create ns-mapping-restore \
  --from-backup ns-backup-demo-app \
  --namespace-mappings demo-app:demo-app-clone \
  --wait

kubectl get all -n demo-app-clone
```

### 測試案例 11 — Restore Hook

透過 Restore CR 注入 initContainer，在還原後執行驗證腳本。

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
        includedNamespaces: [demo-db]
        labelSelector:
          matchLabels:
            app: mysql
        postHooks:
          - init:
              initContainers:
                - name: post-restore-check
                  image: mysql:8.0
                  command: ["/bin/bash", "-c", "echo 'Post-restore validation completed'"]
```

### 測試案例 12 — 多重 Backup Storage Location

```bash
# 新增第二個 BSL
velero backup-location create secondary \
  --provider aws \
  --bucket velero-backup-secondary \
  --config region=minio,s3ForcePathStyle="true",s3Url=http://minio.minio.svc:9000 \
  --credential cloud-credentials=cloud

# 備份到第二個 BSL
velero backup create secondary-bsl-backup \
  --include-namespaces demo-app \
  --storage-location secondary \
  --wait

velero backup-location get
```

### 測試案例 13 — Cluster Resource 備份

```bash
velero backup create cluster-resources-backup \
  --include-cluster-resources=true \
  --include-resources clusterroles,clusterrolebindings,customresourcedefinitions \
  --wait
```

### 測試案例 14 — 叢集遷移模擬

透過 Namespace Mapping 模擬跨叢集遷移。

```bash
# 建立遷移備份
velero backup create migration-backup \
  --include-namespaces demo-app,demo-db \
  --include-cluster-resources=true \
  --default-volumes-to-fs-backup=true \
  --wait

# 模擬遷移到新環境（透過 Namespace Mapping）
velero restore create migration-restore \
  --from-backup migration-backup \
  --namespace-mappings demo-app:migrated-app,demo-db:migrated-db \
  --wait

kubectl get all -n migrated-app
kubectl get all -n migrated-db
```

---

## 備份與復原策略指南

### RPO / RTO 概念

```mermaid
graph LR
    subgraph TIMELINE["時間軸"]
        direction LR
        LB["最後一次備份<br/>Last Backup"]
        DIS["災難發生<br/>Disaster"]
        REC["系統復原<br/>Recovery"]
    end

    LB -->|"RPO<br/>Recovery Point Objective<br/>可容忍的資料遺失時間"| DIS
    DIS -->|"RTO<br/>Recovery Time Objective<br/>系統恢復所需時間"| REC

    style LB fill:#c8e6c9
    style DIS fill:#ffcdd2
    style REC fill:#c8e6c9
```

| 指標 | 說明 | 本 PoC 驗證結果 |
|------|------|----------------|
| **RPO** | 從災難發生到最後一次備份的時間差 | Schedule 每 5 分鐘 → RPO < 5 min |
| **RTO** | 從災難發生到服務恢復的時間 | Restore 約 5~30 秒 → RTO < 1 min |

### 建議的備份策略

```mermaid
graph TB
    subgraph STRATEGY["3-2-1 備份策略"]
        THREE["3 份備份"]
        TWO["2 種儲存媒介"]
        ONE["1 份異地備援"]
    end

    THREE --> BSL1["Primary BSL<br/>MinIO Cluster A"]
    THREE --> BSL2["Secondary BSL<br/>MinIO Cluster B"]
    THREE --> BSL3["Cloud BSL<br/>AWS S3 / GCS"]

    TWO --> LOCAL["本地物件儲存"]
    TWO --> CLOUD["雲端物件儲存"]

    ONE --> DR["異地 DR Site"]

    style STRATEGY fill:#e8eaf6
```

---

## 專案檔案說明

| 檔案 | 說明 |
|------|------|
| `kind-config.yaml` | Kind 叢集配置（1 control-plane + 2 worker） |
| `minio-deployment.yaml` | MinIO S3 相容儲存部署清單 |
| `demo-nginx.yaml` | Nginx + PVC + ConfigMap + Secret 示範應用 |
| `demo-mysql.yaml` | MySQL StatefulSet 含 Velero Backup Hook |
| `restore-hook-config.yaml` | Restore Hook 設定範例（initContainer 注入） |
| `velero-poc-kind-ubuntu.md` | 完整 PoC 規劃文件與操作步驟 |

---

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

---

## 常見問題排查

| 問題 | 排查方式 | 解決方案 |
|------|---------|---------|
| BSL 顯示 Unavailable | `velero backup-location get` | 檢查 MinIO 連線與憑證是否正確 |
| Backup 狀態 PartiallyFailed | `velero backup logs <name>` | 查看哪些資源備份失敗 |
| PV 資料未備份 | `velero backup describe <name>` | 確認使用 `--default-volumes-to-fs-backup` |
| Restore 資源衝突 | `velero restore logs <name>` | 加上 `--existing-resource-policy=update` |
| Hook 執行逾時 | `velero backup logs <name> \| grep hook` | 調整 Pod annotation 中的 timeout 值 |
| CLI 無法取得 log | CLI 連不到叢集內 MinIO | 使用 `kubectl logs deployment/velero -n velero` 替代 |
| Schedule 沒觸發 | `velero schedule get` 確認 Status | 檢查 Cron 表達式是否正確 |

---

## 企業評估摘要

### Velero vs 其他備份方案

```mermaid
graph TB
    subgraph COMPARE["K8s 備份方案比較"]
        VEL2["Velero<br/>開源免費<br/>Apache 2.0"]
        KAS["Kasten K10<br/>商業授權<br/>Veeam"]
        TK8["TrilioVault<br/>商業授權"]
        ETCD["etcd snapshot<br/>僅備份 etcd<br/>無法選擇性復原"]
    end

    VEL2 -->|"適合"| USE1["中小型企業<br/>PoC / Dev 環境"]
    KAS -->|"適合"| USE2["大型企業<br/>需要 Web UI"]
    TK8 -->|"適合"| USE3["多租戶環境"]
    ETCD -->|"適合"| USE4["簡單 DR<br/>整叢集還原"]

    style VEL2 fill:#c8e6c9
    style KAS fill:#e3f2fd
    style TK8 fill:#fff3e0
    style ETCD fill:#f3e5f5
```

### 優勢

- 完全開源免費（Apache 2.0 授權），無節點數限制
- 透過 Kubernetes API 備份（非直接存取 etcd），安全性更高
- 支援 Kopia/Restic 檔案系統備份，不依賴 CSI Snapshot
- Plugin 架構可擴充，支援 AWS / Azure / GCP 等主流雲端
- 社群活躍，VMware/Broadcom 持續維護

### 限制

- 純 CLI 操作，無內建 Web UI（可搭配第三方 Dashboard）
- 應用層一致性備份需自行實作 Hook
- RBAC 與多租戶管理需額外配置
- 加密依賴儲存端（MinIO SSE）或 Kopia 內建加密

### 金融業合規建議

- 搭配 MinIO + TLS + Server-Side Encryption 滿足資料加密需求
- 搭配 PostgreSQL WAL Archiving 或 MySQL binlog 做資料庫層級備份
- 透過 CronJob + Velero Schedule 實現 RPO < 1hr
- 建議 BSL 設定 Immutable Bucket 防止勒索軟體竄改

---

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

---

## 延伸學習資源

- [Velero 官方文件](https://velero.io/docs/)
- [Kubernetes 官方文件](https://kubernetes.io/docs/home/)
- [MinIO 官方文件](https://min.io/docs/minio/linux/index.html)
- [Kind 官方文件](https://kind.sigs.k8s.io/)

---

> **PoC 完成日期**：2026-02-10
> **環境**：Ubuntu x86 / Kind v0.24.0 / Velero v1.17.0
> **測試結果**：14/14 測試案例全數通過 Completed
