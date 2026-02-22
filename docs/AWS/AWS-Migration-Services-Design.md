# AWS 機房遷移架構設計文件（二）：各服務角色設計

> 文件版本：v1.0  
> 撰寫日期：2026-02-22  
> 適用範圍：機房至 AWS 全面遷移專案

---

## 目錄

1. [AP Server 設計](#1-ap-server-設計)
2. [File Server 設計](#2-file-server-設計)
3. [解析服務設計](#3-解析服務設計)
4. [DB Server 設計](#4-db-server-設計)
5. [ArcGIS Cluster 設計](#5-arcgis-cluster-設計)
6. [Portal Cluster 設計](#6-portal-cluster-設計)
7. [NAS 替代方案設計](#7-nas-替代方案設計)
8. [Active Directory / 網域設計](#8-active-directory--網域設計)

---

## 服務角色對應總覽

| 原有角色 | 數量 | AWS 服務 | Multi-AZ | Load Balancer | Auto Scaling |
|---------|------|---------|----------|---------------|--------------|
| AP Server | 2 | EC2 (Windows) | ✓ | ALB | 可選 |
| File Server | 1 | FSx for Windows | ✓ | - | - |
| 解析服務 | 2 (主/備) | EC2 (Windows) | ✓ | - | - |
| DB Server (主/備) | 2 | RDS SQL Server Multi-AZ | ✓ | - | - |
| DB Server (分析) | 1 | RDS SQL Server Read Replica | - | - | - |
| ArcGIS Server | 3 | EC2 (Windows) | ✓ | ALB | 可選 |
| Portal | 3 | EC2 (Windows) | ✓ | ALB | 可選 |
| NAS | 2 | FSx + S3 | ✓ | - | - |

---

## 1. AP Server 設計

### 1.1 設計決策

| 項目 | 決策 | 理由 |
|------|------|------|
| AWS 服務 | EC2 (Windows Server 2022) | 現有應用為 Windows，需保持相容性 |
| 執行個體類型 | m6i.xlarge (4 vCPU, 16GB) | 初始配置，可依實際負載調整 |
| 數量 | 2 台，分散於不同 AZ | 維持原有架構，確保高可用 |
| 儲存 | gp3 EBS (100GB OS + 200GB Data) | gp3 成本效益優於 gp2，IOPS 可獨立調整 |

### 1.2 部署架構

```
┌─────────────────────────────────────────┐
│              ALB (HTTPS)                │
│         ┌───────────┴───────────┐       │
│         ▼                       ▼       │
│   ┌──────────┐           ┌──────────┐   │
│   │ AP-01    │           │ AP-02    │   │
│   │ AZ-a     │           │ AZ-c     │   │
│   │ m6i.xl   │           │ m6i.xl   │   │
│   └────┬─────┘           └────┬─────┘   │
│        │                      │         │
│        └──────────┬───────────┘         │
│                   ▼                     │
│        ┌──────────────────┐             │
│        │ RDS SQL Server   │             │
│        │ FSx for Windows  │             │
│        └──────────────────┘             │
└─────────────────────────────────────────┘
```

### 1.3 網路配置

| 項目 | 配置 |
|------|------|
| Subnet | private-app-subnet-a / private-app-subnet-c |
| Security Group | sg-ap |
| Inbound | 80/443 from sg-alb |
| Outbound | 1433 to sg-db, 445 to sg-fsx, 443 to 0.0.0.0/0 (via NAT) |

### 1.4 Load Balancer 設計

| 項目 | 配置 |
|------|------|
| 類型 | Application Load Balancer |
| Listener | HTTPS:443 (SSL Termination) |
| Target Group | Protocol HTTP, Port 80 |
| Health Check | /health, Interval 30s, Threshold 3 |
| Sticky Session | 啟用 (Application-based cookie) |

**為何啟用 Sticky Session：**
- AP Server 可能存在 Session State
- 遷移初期維持與原有行為一致，後續可評估 Session 外部化

### 1.5 Auto Scaling 評估

**現階段不啟用，原因如下：**
- 原有架構為固定 2 台，先維持一致性
- Windows Server 啟動時間較長（約 3-5 分鐘）
- 需先評估應用程式是否支援動態擴展

**未來啟用條件：**
- 應用程式 Stateless 化完成
- 建立 Golden AMI 自動化流程
- 負載明確呈現週期性波動

---

## 2. File Server 設計

### 2.1 設計決策

| 項目 | 決策 | 理由 |
|------|------|------|
| AWS 服務 | Amazon FSx for Windows File Server | 全託管 Windows 檔案系統，原生 SMB 支援 |
| 部署模式 | Multi-AZ | 自動故障轉移，RPO/RTO 接近零 |
| 容量 | 1TB SSD | 初始配置，可線上擴展 |
| 吞吐量 | 128 MB/s | 依實際負載調整 |

**為何選擇 FSx 而非 EC2 自建：**
- 免去 Windows Server 授權與維護
- 內建 Multi-AZ HA，無需自行設定複寫
- 支援 AD 整合、VSS 快照、DFS Namespace
- 自動備份至 S3

### 2.2 整合配置

| 項目 | 配置 |
|------|------|
| VPC | 主 VPC |
| Subnet | private-db-subnet-a (Primary), private-db-subnet-c (Standby) |
| Security Group | sg-fsx |
| AD 整合 | AWS Managed AD |
| DNS 名稱 | \\fs.internal.example.com\share |

### 2.3 存取控制

| 項目 | 配置 |
|------|------|
| Inbound | 445 (SMB), 5985 (WinRM) from sg-ap, sg-arcgis |
| AD 群組權限 | 維持原有 NTFS 權限結構 |

### 2.4 備份策略

| 項目 | 配置 |
|------|------|
| 自動備份 | 每日，保留 30 天 |
| 備份視窗 | 03:00-04:00 UTC+9 |
| 跨區域備份 | 視需求啟用（Osaka） |

---

## 3. 解析服務設計

### 3.1 設計決策

| 項目 | 決策 | 理由 |
|------|------|------|
| AWS 服務 | EC2 (Windows Server 2022) | 專屬服務，需獨立運算資源 |
| 執行個體類型 | c6i.large (2 vCPU, 4GB) | 解析服務通常為 CPU-bound |
| 數量 | 2 台 (主/備)，分散於不同 AZ | 維持原有主備架構 |
| HA 機制 | Route 53 Health Check + Failover | 無需 Load Balancer，主備切換即可 |

### 3.2 高可用設計

```
┌────────────────────────────────────────────────────┐
│                  Route 53 Private Zone             │
│              parser.internal.example.com           │
│                                                    │
│     ┌─────────────────┐   ┌─────────────────┐     │
│     │  Health Check   │   │  Health Check   │     │
│     │     (HTTP)      │   │     (HTTP)      │     │
│     └────────┬────────┘   └────────┬────────┘     │
│              │                     │               │
│              ▼                     ▼               │
│     ┌─────────────────┐   ┌─────────────────┐     │
│     │  Parser-Main    │   │  Parser-Standby │     │
│     │  (Primary)      │   │  (Secondary)    │     │
│     │  AZ-a           │   │  AZ-c           │     │
│     └─────────────────┘   └─────────────────┘     │
│                                                    │
│     Failover Policy: Primary → Secondary           │
│     TTL: 60 seconds                                │
└────────────────────────────────────────────────────┘
```

### 3.3 網路配置

| 項目 | 配置 |
|------|------|
| Subnet | private-app-subnet-a (Main), private-app-subnet-c (Standby) |
| Security Group | sg-parser |
| Inbound | 服務 Port from sg-ap |
| DNS | Route 53 Private Hosted Zone，Failover Routing |

### 3.4 Failover 機制

| 項目 | 配置 |
|------|------|
| Health Check Protocol | HTTP |
| Health Check Path | /health |
| Check Interval | 10 秒 |
| Failure Threshold | 3 次 |
| 預計 Failover 時間 | 30-60 秒 |

---

## 4. DB Server 設計

### 4.1 設計決策

| 項目 | 決策 | 理由 |
|------|------|------|
| AWS 服務 | Amazon RDS for SQL Server | 全託管，免除 OS/DB 維護 |
| 版本 | SQL Server 2019 Standard | 相容現有應用 |
| 執行個體類型 | db.r6i.xlarge (4 vCPU, 32GB) | 依現有 DB 規格對應 |
| 儲存 | gp3, 500GB, 3000 IOPS | 初始配置，可線上擴展 |

**為何選擇 RDS 而非 EC2 自建：**
- 自動化備份、Patch、升級
- 內建 Multi-AZ 自動故障轉移
- 免去 SQL Server HA 設定（Always On 等）
- 一鍵建立 Read Replica

### 4.2 架構配置

```
┌─────────────────────────────────────────────────────┐
│                     RDS Subnet Group                │
│                                                     │
│   ┌───────────────────┐   ┌───────────────────┐    │
│   │   RDS Primary     │   │   RDS Standby     │    │
│   │   (AZ-a)          │◄─►│   (AZ-c)          │    │
│   │   db.r6i.xlarge   │   │   db.r6i.xlarge   │    │
│   │                   │   │   (同步複寫)       │    │
│   └─────────┬─────────┘   └───────────────────┘    │
│             │                                       │
│             │ 非同步複寫                            │
│             ▼                                       │
│   ┌───────────────────┐                            │
│   │  RDS Read Replica │                            │
│   │  (TERIA 分析用)    │                            │
│   │  AZ-c             │                            │
│   │  db.r6i.large     │                            │
│   └───────────────────┘                            │
└─────────────────────────────────────────────────────┘
```

### 4.3 各 DB 角色配置

| 角色 | 執行個體 | AZ | 用途 | 同步方式 |
|------|---------|----|----|---------|
| Primary | db.r6i.xlarge | AZ-a | 主要讀寫 | - |
| Standby | db.r6i.xlarge | AZ-c | HA 備援 | 同步複寫 |
| Read Replica | db.r6i.large | AZ-c | TERIA 分析查詢 | 非同步複寫 |

### 4.4 網路配置

| 項目 | 配置 |
|------|------|
| Subnet Group | private-db-subnet-a + private-db-subnet-c |
| Security Group | sg-db |
| Inbound | 1433 from sg-ap, sg-arcgis, sg-portal |
| Public Access | 否 |

### 4.5 備份與維護

| 項目 | 配置 |
|------|------|
| 自動備份 | 啟用，保留 14 天 |
| 備份視窗 | 03:00-04:00 UTC+9 |
| 維護視窗 | 週日 04:00-05:00 UTC+9 |
| 加密 | AWS KMS (aws/rds) |

### 4.6 TERIA 分析用 DB 設計考量

| 項目 | 決策 | 理由 |
|------|------|------|
| 服務類型 | RDS Read Replica | 查詢負載不影響主庫 |
| 執行個體規格 | db.r6i.large | 分析用途，規格可較小 |
| 複寫延遲容許 | < 1 分鐘 | 分析報表可接受近即時資料 |

---

## 5. ArcGIS Cluster 設計

### 5.1 設計決策

| 項目 | 決策 | 理由 |
|------|------|------|
| AWS 服務 | EC2 (Windows Server 2022) | ArcGIS Enterprise 需 Windows 環境 |
| 執行個體類型 | r6i.xlarge (4 vCPU, 32GB) | ArcGIS Server 為記憶體密集型 |
| 數量 | 3 台，跨 2 個 AZ | 維持原有叢集規模 |
| 儲存 | gp3 EBS (100GB OS + 500GB Data) | 地圖快取與服務資料 |

### 5.2 叢集架構

```
┌─────────────────────────────────────────────────────────┐
│                    ALB (HTTPS:443)                      │
│           ┌────────────┼────────────┐                   │
│           ▼            ▼            ▼                   │
│   ┌─────────────┐┌─────────────┐┌─────────────┐        │
│   │ ArcGIS-01   ││ ArcGIS-02   ││ ArcGIS-03   │        │
│   │ AZ-a        ││ AZ-a        ││ AZ-c        │        │
│   │ r6i.xlarge  ││ r6i.xlarge  ││ r6i.xlarge  │        │
│   └──────┬──────┘└──────┬──────┘└──────┬──────┘        │
│          │              │              │                │
│          └──────────────┼──────────────┘                │
│                         ▼                               │
│          ┌─────────────────────────────┐               │
│          │  Shared Config Store        │               │
│          │  (FSx for Windows)          │               │
│          └─────────────────────────────┘               │
└─────────────────────────────────────────────────────────┘
```

### 5.3 網路配置

| 項目 | 配置 |
|------|------|
| Subnet | private-app-subnet-a (×2), private-app-subnet-c (×1) |
| Security Group | sg-arcgis |
| Inbound | 6443, 6080 from sg-alb; 內部通訊 from sg-arcgis |
| Outbound | 1433 to sg-db, 445 to sg-fsx |

### 5.4 Load Balancer 設計

| 項目 | 配置 |
|------|------|
| Target Group | Port 6443 (HTTPS) |
| Health Check | /arcgis/rest/info, Interval 30s |
| Sticky Session | 啟用 (Cookie-based) |
| Deregistration Delay | 60 秒 |

### 5.5 共享儲存設計

ArcGIS Server 叢集需要共享 Config Store 與 Server Directories：

| 路徑 | 儲存位置 | 用途 |
|------|---------|------|
| Config Store | FSx for Windows | 叢集設定同步 |
| Server Directories | FSx for Windows | 服務發佈、快取 |
| 地圖快取 | S3 (可選) | 大量靜態快取可改用 S3 |

---

## 6. Portal Cluster 設計

### 6.1 設計決策

| 項目 | 決策 | 理由 |
|------|------|------|
| AWS 服務 | EC2 (Windows Server 2022) | Portal for ArcGIS 需 Windows 環境 |
| 執行個體類型 | r6i.xlarge (4 vCPU, 32GB) | Portal 為記憶體密集型 |
| 數量 | 3 台，跨 2 個 AZ | 維持原有叢集規模 |
| 儲存 | gp3 EBS (100GB OS + 300GB Data) | Portal 內容儲存 |

### 6.2 叢集架構

```
┌─────────────────────────────────────────────────────────┐
│                    ALB (HTTPS:443)                      │
│           ┌────────────┼────────────┐                   │
│           ▼            ▼            ▼                   │
│   ┌─────────────┐┌─────────────┐┌─────────────┐        │
│   │ Portal-01   ││ Portal-02   ││ Portal-03   │        │
│   │ AZ-a        ││ AZ-a        ││ AZ-c        │        │
│   │ r6i.xlarge  ││ r6i.xlarge  ││ r6i.xlarge  │        │
│   └──────┬──────┘└──────┬──────┘└──────┬──────┘        │
│          │              │              │                │
│          └──────────────┼──────────────┘                │
│                         ▼                               │
│          ┌─────────────────────────────┐               │
│          │  Shared Content Directory   │               │
│          │  (FSx for Windows)          │               │
│          └─────────────────────────────┘               │
└─────────────────────────────────────────────────────────┘
```

### 6.3 網路配置

| 項目 | 配置 |
|------|------|
| Subnet | private-app-subnet-a (×2), private-app-subnet-c (×1) |
| Security Group | sg-portal |
| Inbound | 7443 from sg-alb; 內部通訊 from sg-portal |
| Outbound | 6443 to sg-arcgis, 1433 to sg-db, 445 to sg-fsx |

### 6.4 Load Balancer 設計

| 項目 | 配置 |
|------|------|
| Target Group | Port 7443 (HTTPS) |
| Health Check | /arcgis/portaladmin, Interval 30s |
| Sticky Session | 啟用 |

### 6.5 Portal 與 ArcGIS Server 整合

Portal 與 ArcGIS Server 需建立 Federation：

| 整合項目 | 配置 |
|---------|------|
| Federation URL | https://arcgis.example.com/arcgis |
| Admin URL | https://arcgis-admin.internal/arcgis |
| Hosting Server | ArcGIS Cluster 作為 Hosting Server |

---

## 7. NAS 替代方案設計

### 7.1 設計決策

原有 NAS ×2 的功能拆分至不同 AWS 服務：

| 原有用途 | AWS 服務 | 理由 |
|---------|---------|------|
| 檔案共享 (SMB) | FSx for Windows | 原生 SMB、AD 整合、Multi-AZ |
| 大量非結構化資料 | S3 Standard | 無限容量、成本低、高耐久性 |
| 備份儲存 | S3 Glacier | 長期保存、成本極低 |

### 7.2 FSx 配置（檔案共享用）

已於 File Server 章節說明，此處補充 NAS 對應：

| 項目 | 配置 |
|------|------|
| 容量 | 2TB SSD (原有 NAS 總容量對應) |
| 吞吐量 | 256 MB/s |
| 存取協定 | SMB 3.x |
| 權限管理 | AD 群組對應 |

### 7.3 S3 配置（大量資料用）

| 項目 | 配置 |
|------|------|
| Bucket 名稱 | company-data-archive |
| 儲存類別 | S3 Standard (熱資料)、S3 Glacier (冷資料) |
| 版本控制 | 啟用 |
| 生命週期 | 90 天後轉 S3-IA、365 天後轉 Glacier |
| 加密 | SSE-S3 (Server-Side Encryption) |

### 7.4 資料存取方式

| 存取情境 | 建議方式 |
|---------|---------|
| 應用程式存取 | AWS SDK (S3) / SMB (FSx) |
| 使用者存取共享資料夾 | FSx SMB 掛載 |
| 大檔案上傳 | S3 Multipart Upload |
| 資料搬遷 | AWS DataSync |

---

## 8. Active Directory / 網域設計

### 8.1 設計決策

| 項目 | 決策 | 理由 |
|------|------|------|
| AD 服務 | AWS Managed Microsoft AD | 全託管、與 AWS 服務原生整合 |
| 版本 | Standard Edition | 單一網域、最多 5 萬物件，足夠使用 |
| 部署模式 | Multi-AZ | DC 自動分散於兩個 AZ |

**為何選擇 AWS Managed AD 而非 AD Connector：**
- AD Connector 需要現有 On-Premises AD
- AWS Managed AD 提供完整 DC 功能
- 可作為獨立網域，或與現有 AD 建立 Trust

### 8.2 AD 架構

```
┌─────────────────────────────────────────────────────────┐
│                 AWS Managed Microsoft AD                │
│                 corp.example.local                      │
│                                                         │
│   ┌───────────────────┐   ┌───────────────────┐        │
│   │   Domain Controller│   │   Domain Controller│        │
│   │   (AZ-a)          │◄─►│   (AZ-c)          │        │
│   └───────────────────┘   └───────────────────┘        │
│                                                         │
│   ┌─────────────────────────────────────────────┐      │
│   │              VPC DNS Resolver                │      │
│   │     (DHCP Option Set 指向 AD DNS)           │      │
│   └─────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
   ┌──────────┐     ┌──────────┐     ┌──────────┐
   │ EC2      │     │ FSx      │     │ RDS      │
   │ Domain   │     │ AD       │     │ Windows  │
   │ Joined   │     │ Integrated│    │ Auth     │
   └──────────┘     └──────────┘     └──────────┘
```

### 8.3 網路配置

| 項目 | 配置 |
|------|------|
| Subnet | private-app-subnet-a + private-app-subnet-c |
| Security Group | sg-ad |
| DNS | VPC DHCP Option Set 指向 AD DNS IP |

### 8.4 Domain Join 設計

| 服務 | Domain Join 方式 |
|------|-----------------|
| EC2 Windows | SSM + Seamless Domain Join |
| FSx for Windows | 建立時指定 AD |
| RDS SQL Server | 建立 AD Directory Association |

**SSM Seamless Domain Join 優點：**
- 無需在 AMI 中寫入 Domain 認證
- EC2 啟動時自動加入網域
- 支援 Auto Scaling 場景

### 8.5 GPO 管理

| GPO 類型 | 管理方式 |
|---------|---------|
| 標準 GPO | 透過 Management Instance 操作 |
| Password Policy | AD Admin 設定 |
| 軟體部署 | 建議改用 SSM 或 SCCM |

**建議的 Management Instance：**
- 部署一台 Windows EC2 安裝 RSAT 工具
- 放置於 Private Subnet
- 透過 SSM Session Manager 存取

### 8.6 AD 整合服務列表

| AWS 服務 | 整合功能 |
|---------|---------|
| EC2 | Domain Join, Windows 驗證 |
| FSx for Windows | SMB 存取權限、NTFS ACL |
| RDS SQL Server | Windows Authentication |
| WorkSpaces | 使用者登入 |
| QuickSight | AD 使用者同步 |

---

## 附錄：執行個體規格對照表

| 角色 | 執行個體類型 | vCPU | RAM | 儲存 | 數量 |
|------|-------------|------|-----|------|------|
| AP Server | m6i.xlarge | 4 | 16 GB | gp3 300GB | 2 |
| 解析服務 | c6i.large | 2 | 4 GB | gp3 100GB | 2 |
| ArcGIS Server | r6i.xlarge | 4 | 32 GB | gp3 600GB | 3 |
| Portal | r6i.xlarge | 4 | 32 GB | gp3 400GB | 3 |
| RDS Primary | db.r6i.xlarge | 4 | 32 GB | gp3 500GB | 1 |
| RDS Standby | db.r6i.xlarge | 4 | 32 GB | gp3 500GB | 1 |
| RDS Replica | db.r6i.large | 2 | 16 GB | gp3 500GB | 1 |
| FSx | - | - | - | 2TB SSD | 1 |

---

> 上一份文件：[AWS 機房遷移架構設計文件（一）：架構總覽與網路設計](./AWS-Migration-Architecture-Overview.md)  
> 下一份文件：[AWS 機房遷移架構設計文件（三）：維運與安全強化設計](./AWS-Migration-Operations.md)

