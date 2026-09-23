# M2 VCF 9.1 架構規劃與 vSAN 上 DB 效能調校建議 — 正式回覆稿

> **檔案說明（內部，寄出前請刪除本段）**
> 本文件是回覆客戶 Ethan 來信的正式回覆稿。「主旨」到「附錄 A」為客戶可見內容，可直接貼入郵件；「附錄 B」為內部備註，列出每一項技術主張的驗證狀態與待原廠確認事項，寄出前請刪除。完整的主張驗證紀錄（含官方文件 URL、審查者修正措辭）見同目錄的 `M2_VCF9.1_claims_verification.md`。

---

**主旨：RE: M2 VCF 9.1 架構規劃與 vSAN 上 DB 效能調校建議**

Dear Ethan,

感謝來信，也謝謝您把 M2 的規劃方向與 DBA 的測試狀況說明得很清楚。以下依您信中的順序分兩部分回覆：第一部分是 VCF 9.1 最小建置架構（Management Domain 的儲存選項、6 台主機的配置建議、若採 vSAN 的硬體規格與網路前置需求），第二部分是 vSAN 上 DB VM 的效能調校（vSAN Policy、VM 層設定、Redo/Log File Sync 的判讀方式與再測試方法）。文中的技術依據均附上 Broadcom / VMware / Oracle 官方文件出處（見附錄 A）；凡屬我們的規劃建議而非官方硬性要求者，會以「建議」標示，尚待原廠書面確認的事項則標示「待原廠確認」。

---

## 一、VCF 9.1 最小建置架構

### 1.1 六台主機 + SAN 是否可建置為符合 Broadcom Support 標準的 VCF 9.1 架構？

**可以。** 依 Broadcom 官方 VCF 9.1 FAQ，全新部署（greenfield）的 VCF 9 Management Cluster 最少需要 4 台主機，且可使用 vSAN、NFS 或 VMFS on FC 作為儲存 [1]：

> "A new VCF 9 deployment requires a minimum of 4 hosts for the management cluster which is deployed using vSAN, NFS or VMFS on FC."

Broadcom KB 416270 亦明確指出，greenfield 部署的 Management Domain 現已支援 VMFS on Fibre Channel 與 NFSv3 作為 principal storage [2]。因此 M2 規劃的 6 台主機 + FC SAN，在主機數量與儲存類型上都符合官方支援條件。

需要留意的兩個前提：

1. **SAN 的連線協定必須是 Fibre Channel（VMFS on FC）或 NFSv3。** KB 416270 說明 iSCSI、NFS v4.1、FCoE 與 NVMe over Fabrics（FC / TCP / RDMA）目前不在 greenfield 流程內，只能透過先建置 vSphere 再 converge / import 的方式納入 VCF [2]。請協助確認 M2 SAN 的協定與型號，若是 iSCSI 或 NVMe-oF，架構規劃需另行調整。
2. **主機、FC HBA、NIC 需在 Broadcom Compatibility Guide 內**，且 FC zoning、LUN 與 VMFS datastore 需在主機加入 VCF 前先建好並掛載到所有主機（使用相同的 datastore 名稱），否則 VCF Installer / SDDC Manager 在建立 domain 或 cluster 時不會列出這些主機 [5][49]。

### 1.2 以「先建 VCF 基礎架構、再逐步擴充」為方向的建議最小架構

VCF 9.x 有兩種佈局：**Standard**（管理元件與 workload 分屬不同 domain）與 **Consolidated**（同一個 Management Domain 內同時承載管理元件與 workload）。VCF 9.1 Design 文件提供「VCF Fleet in a Single Site with Minimal Footprint」設計藍圖，將所有管理元件與 workload 部署在單一 vSphere cluster 內 [7]；VCF 9.0 Design Library 的 Tenancy Deployment Model 1 亦描述同一 domain 內管理元件與 workload 共用 cluster 的模式，並提醒共用時需注意資源競爭、效能與可用性 [8]。以 6 台主機 + FC SAN 為前提，我們比較兩個方案：

| 項目 | 方案 A：6 台主機、單一 Management Domain（Consolidated / Minimal Footprint） | 方案 B：4 台 Management Domain + 2 台 VI Workload Domain |
|---|---|---|
| 主機配置 | 6 台全部在 Management Domain 的 default cluster | Management Domain 4 台（官方下限 [1]）+ 一個 2 台的 VI Workload Domain cluster |
| Principal storage | VMFS on FC | 兩個 cluster 皆 VMFS on FC |
| 官方依據 | Minimal Footprint 藍圖 [7]；Management Cluster 最少 4 台 [1] | VI Workload Domain 使用 NFS / VMFS on FC 且以 vSphere Lifecycle Manager image 管理時最少 2 台，否則最少 3 台 [6] |
| 優點 | 資源利用率最高、管理最單純；日後加主機、加 cluster、加 domain 的路徑都保留 | 管理元件與 workload 實體隔離（Standard 架構） |
| 缺點 | workload 與管理元件共用 cluster，需以 Resource Pool 與資源預留保護管理元件 [8] | 2 台的 Workload Domain 只能容忍單一主機故障、無法啟用 Workload Management（需 3 台）[6]；整體資源被切碎 |
| 後續擴充 | 直接新增主機到既有 cluster；在 Management Domain 新增 vSAN ESA cluster 或新建 vSAN Workload Domain；將 M1 converge / import 為 Workload Domain | 同左 |
| 我們的建議 | **建議採用**，作為「先建 VCF 基礎、再逐步擴充」的起點 | 若貴司政策要求管理與 workload 實體隔離，建議將 Workload Domain 補到 3 台（合計 7 台） |

擴充路徑上有三點請特別留意：

1. **Cluster 的 principal storage 類型在建立後不可更改** [5]。KB 435850 說明 VCF 9.1 Management Domain 支援的 principal storage 轉換僅有同類型（NFSv3 → NFSv3、VMFS FC → VMFS FC）與 vVol → vSAN，並沒有 FC → vSAN 的路徑 [4]。因此未來導入 vSAN 的做法是**在 Management Domain 新增一個 vSAN cluster，或新建 vSAN 的 VI Workload Domain**（同一 domain 內不同 cluster 可以使用不同的 principal storage [6]），而不是把既有 FC cluster 原地轉成 vSAN。
2. 由此延伸的採購建議：**若希望 M2 這 6 台主機日後能轉作 vSAN ESA 使用，請在採購時就選擇 vSAN ESA ReadyNode 認證機型，並預留 NVMe 槽位與 25GbE 網路**（規格見 1.4）；否則日後只能另購主機組成新的 vSAN cluster。
3. M1（vSphere 8.0 U3 + SAN）日後可透過 VCF 9.1 的 converge / import 納入為 Workload Domain。官方支援 vCenter 8.0 Update 3a 以上且已有 NSX Manager 4.2 以上的環境直接納入；若 M1 沒有 NSX，則需先將 vCenter 升級至 9.1 再納入 [10]。屆時請提供 M1 的 vCenter 確切版本，我們再確認適用路徑。

### 1.3 Management Domain 是否一定需要 vSAN？可否直接使用既有的 External SAN？

**不一定需要 vSAN。** 自 VCF 9.0.x 起，透過 VCF Installer 全新部署的 Management Domain 可以選擇 vSAN ESA、vSAN OSA、NFSv3 或 VMFS on FC 作為 principal storage [2][3][5]。VMware Cloud Foundation 官方部落格（2025-11-11）原文：

> "VMware Cloud Foundation 9 now allows the use of NFSv3 or Fibre Channel VMFS datastores as principal storage for the management workload domain during greenfield deployments." [3]

vSAN ReadyNode 仍是 Broadcom 建議的首選，但不是必要條件。以 External SAN 建置 Management Domain 時，請留意：

1. **協定與前置作業限制**：見 1.1（僅 FC 與 NFSv3；zoning / LUN / datastore 需事先完成）。
2. **只有 vSAN 才有的功能**：vSAN Data Protection 與原生 Snapshot Service 需 vSAN ESA（vSAN OSA 也不支援）[11]；SDDC Manager 的 vSAN stretched cluster 流程僅適用 vSAN cluster；FC / NFS 的 Management Domain 若要跨站點，需依 KB 417356 採 vSphere Metro Storage Cluster 的方式 [12]。若這些功能是 M2 的目標，日後需另建 vSAN ESA cluster。
3. **官方文件仍有版本落差**：VCF 9.1 Design 的「Storage Models」頁面 [48] 與 KB 392993 [47] 仍保留較舊的敘述（例如 Management Domain 的 FC 僅作 supplemental storage、或僅限 converge 的環境）。KB 416270、VCF 9.1 FAQ、VCF Installer 文件與官方部落格則是較新且明確的說明，實務上以後者為準。**為避免日後 Support 認定上的爭議，我們建議在硬體採購定案前，由我們協助向 Broadcom Support 就「6 台主機 + VMFS on FC 的 Management Domain greenfield 部署」取得書面確認。**
4. **授權面**：VCF 授權每個 core 內含 1 TiB 的 vSAN 容量，不足時可加購 vSAN TiB add-on [13][14]。M2 初期使用 SAN 不會影響授權，日後導入 vSAN 時可直接使用內含容量。

### 1.4 若 Management Domain 採用 vSAN：每台主機最少幾顆 SSD？網路是否至少 10GbE？建議規格？

VCF 9.1 新建 vSAN cluster 同時支援 vSAN ESA 與 vSAN OSA，主機需分別在 HCL 上列為 vSAN ESA ReadyNode 或 vSAN OSA（hybrid / all-flash），且 ESA 與 OSA 建立後不可互轉 [15]。Broadcom 將 ESA 定位為高效能、低延遲的首選，也是 DB workload 的建議方向；以下規格以 ESA 為主。

| 項目 | 官方最低要求 | 建議採購規格（M2；兼顧日後 vSAN ESA 與 DB workload） | 依據 |
|---|---|---|---|
| 主機機型 | vSAN ESA ReadyNode（或 vSAN OSA ReadyNode）認證機型 | ESA ReadyNode 認證機型，同一 cluster 使用同型號 | [15][17] |
| CPU | ESA-AF-0 基準為 16 cores；VCF 授權每顆 CPU 最少計 16 cores | 2 顆 CPU，每顆 24 至 32 cores 以上（consolidated 加 DB workload） | [16][14] |
| 記憶體 | vSAN ESA 最少 128 GB | 512 GB 以上（DB VM 需大量記憶體預留） | [18][16] |
| 開機裝置 | 32 GB 以上持久性儲存，新機建議 128 GB；避免 USB / SD | 2 顆 M.2 NVMe（RAID-1）128 GB 以上 | [19] |
| vSAN ESA 儲存裝置（SSD 顆數） | NVMe TLC、每顆 1.6 TB 以上、1 DWPD 以上、效能等級 Class F 以上；入門的 ESA-AF-0 profile 可少至 1 至 2 顆 | 每台 4 顆以上 NVMe（效能與重建彈性），並預留槽位 | [16][17][20][22] |
| vSAN OSA 儲存裝置（若採 OSA） | 每個 disk group 1 顆 cache 加 1 顆以上 capacity；all-flash 需 10GbE | 新採購不建議走 OSA | [15][18] |
| 網路（vSAN） | ESA 最低 10GbE（僅 ESA-AF-0 profile 允許），建議 25GbE 以上；網路延遲 1 ms 以內 | 每台 2 個 25GbE port（vSAN / vMotion / Overlay）加 2 個 FC HBA port（SAN） | [18][21][22] |
| 網路（VCF 一般） | 所有 pNIC 需 10 Gbps 以上 | 同上 | [23] |
| MTU | vSAN / NFS 儲存網路預設 9000，實體交換器需端對端支援 | 9000 | [23][24] |

針對「網路是否至少需要 10GbE」的直接回答：**10GbE 是 vSAN ESA 的最低支援門檻，且只有入門的 ESA-AF-0 profile 允許 [22]**；vSAN Skyline Health 的「Physical NIC link speed」檢查對 ESA 以每台 25 Gbps 彙總頻寬為基準，10GbE 會出現提示 [21]；KB 372311 亦指出使用 10Gb 網路的 ESA cluster 容量使用率需維持在 75% 以下，resync 時需額外設定 [25][26]。以 DB 低延遲需求來看，**我們強烈建議直接採 25GbE**。

### 1.5 網路與 NSX 前置需求（採購交換器前請一併確認）

- **NSX**：NSX Manager 是每個 VCF 9.x Management Domain 的必要元件，由 VCF Installer 部署；VCF 9.0 起可選單節點（資源精簡）或三節點 cluster（高可用）[9]。NSX Edge cluster 不是 bring-up 的必要項目，日後需要 VPC / VCF Automation 網路時再部署；因此初期不需要 BGP 與 Edge uplink VLAN，但建議在設計中預留。
- **VLAN**：主機管理、VM 管理、vMotion、NSX Host Overlay（TEP）；若使用 vSAN 需另加 vSAN VLAN，使用 NFS 需另加 NFS VLAN。實際清單以 VCF 9.1 Installer 的 planning 步驟產出為準。
- **MTU**：儲存與 vMotion 網路 9000 [23]；Overlay 網路需 1600 以上（建議一併使用 9000）。
- **IP / DNS**：VCF 9.1 新增 VCF Management Services（Fleet Services、Instance Services、License Server、Identity Broker 等）及其 IP pool，Simple 模式約需 6 個以上 IP、HA 模式約 8 個以上，另加 SDDC Manager、vCenter、NSX Manager、VCF Operations 等各自的 IP 與 DNS 記錄；確切數量以 VCF 9.1 Installer planning 產出的清單為準（待原廠確認）。
- **實體交換器**：建議 2 台 ToR，每台主機的 pNIC 分別接到兩台 ToR，所有 VLAN trunk 到主機 port；LACP 為選配（VCF 9.1 Installer 已支援）。

---

## 二、vSAN 上 DB VM 的效能調校

### 2.1 先釐清測試環境（請 DBA 與基礎架構同仁協助提供）

為了讓建議能對應到實際環境，以及正確判讀「Redo / Commit Path 與 Log File Sync Latency 偏高」的原因，麻煩先提供：

1. vSAN 架構（ESA 或 OSA）與版本（vSAN 8.x / 9.x）；若已在 VCF 9.1 上，Auto-RAID 是否啟用。
2. 主機是否為 ReadyNode、NVMe / SSD 型號與韌體、NIC 型號與速度（10GbE 或 25GbE）、MTU、vSAN 是否使用獨立 VLAN 與 VMkernel。
3. DB VM 套用的 Storage Policy 內容（FTT / RAID、Stripe Width、IOPS Limit、Object Space Reservation、Checksum、壓縮）；Redo Log 與 Data File 是否使用同一個 policy。
4. Cluster 層的 Deduplication / Compression（OSA）或 policy 層壓縮（ESA）狀態。
5. DB VM 的虛擬硬體版本、vCPU / 記憶體與預留設定、虛擬儲存控制器種類與數量、各 VMDK 的配置、Guest OS 與 I/O scheduler、Oracle 版本、redo log 大小與組數、平均 commit 頻率。
6. 測試工具與 profile、AWR 報表（log file sync 與 log file parallel write 的平均等待時間、commits/s、redo MB/s），以及同一時段 vSAN Performance Service 的 VM 層與 Backend 寫入延遲、Skyline Health 狀態。

### 2.2 vSAN Storage Policy 建議

先說明原理：vSAN 的每一筆寫入都要等網路對側的副本寫入完成才會回覆 ack，Redo / Transaction Log 這類小型、同步、低 outstanding I/O 的寫入，因此對網路延遲與副本配置最敏感。ESA 與 OSA 的寫入路徑不同，建議也不同：

| Policy 項目 | vSAN OSA 建議 | vSAN ESA 建議 | 說明與依據 |
|---|---|---|---|
| Failures to tolerate / RAID | Redo / Log VMDK 使用 RAID-1、FTT=1（Oracle on vSAN 參考架構即對 online redo log 採 RAID-1 [27]）；Data File 可依容量需求評估 RAID-5 | RAID-5（FTT=1）或 RAID-6（FTT=2）即可：官方文件與測試指出 ESA 的 RAID-5/6 效能與 RAID-1 相當 [28][29][30][46]；VCF 9.1 新建的 cluster 預設由 Auto-RAID 系統管理 RAID 層級 [31] | OSA 的 RAID-5/6 需要額外的 parity 讀寫，寫入延遲高於 RAID-1（我們的實務經驗）；ESA 的 log-structured 寫入路徑消除了這個差異 [28] |
| Number of disk stripes per object | 保留預設 1；官方僅建議在效能除錯時作為實驗項目 [32] | 保留預設 1，ESA 不需要調整 [32] | |
| IOPS limit for object | 0（不限制） | 0（不限制） | 設定後 vSAN 以 32 KB 為單位計算並節流，會拉高延遲 [33] |
| Object space reservation | 預設 0（thin）；官方建議 thin [34] | 0（thin） | OSR=100 只是容量預留，對 all-flash 寫入延遲沒有實質幫助（我們的實務經驗） |
| Disable object checksum | 保留預設（checksum 啟用） | 保留預設（checksum 啟用） | 一般建議；即使 DB 有自身 checksum 亦不建議關閉 |
| Compression / Deduplication | Cluster 層的 Dedup 與 Compression 會增加寫入路徑處理；低延遲 DB cluster 建議先以未啟用狀態做 baseline 比較（待 Lab 驗證） | 屬 policy 層設定；建議先以預設做 baseline，再用關閉壓縮的獨立 policy 測 Redo VMDK（待 Lab 驗證） | 以實測為準 |
| Flash read cache reservation | 0（僅 hybrid 有意義） | 不適用 | |
| Force provisioning | 否 | 否 | |

**具體作法：為 Redo / Transaction Log VMDK 建立獨立的 Storage Policy**（例如 `DB-Log-Policy`：OSA 用 RAID-1 FTT=1，ESA 用 RAID-5 FTT=1；stripe 1、IOPS limit 0、OSR 0、checksum 啟用），Data / Temp 另用一個 policy，方便分開量測與調整。若 cluster 是 VCF 9.1 且啟用 Auto-RAID，RAID 層級由系統管理，請先確認實際套用的 policy，再決定是否對特定 VM 使用自訂 policy（Auto-RAID 下自訂 policy 的建議作法待原廠確認）。

### 2.3 VM 層建議

| 項目 | 建議 | 說明與依據 |
|---|---|---|
| 虛擬儲存控制器 | 使用 PVSCSI，並用滿 4 個 PVSCSI controller；Redo / Log VMDK 掛在獨立的 controller，與 Data File 分開 | Oracle on VMware 最佳實務 [35][36]；vSphere 文件亦建議使用 NVMe 或 PVSCSI controller 並將磁碟平均分散到多個 controller [37]。虛擬 NVMe controller 可作為對照測試項目，但 Oracle RAC 的 multi-writer 磁碟不可掛在虛擬 NVMe controller [38] |
| PVSCSI queue depth | Windows：RequestRingPages=32、MaxQueueDepth=254；Linux：vmw_pvscsi 模組 ring_pages=32、cmd_per_lun=254（依 KB 343323） | 預設為每個磁碟 64、每個 adapter 254，可提高至 254 / 1024 [39] |
| VMDK 配置 | OS、Data、Redo、Temp、Archive 各自獨立 VMDK；Redo 至少 2 顆 VMDK，分置於不同 controller | [35][36] |
| VMDK 佈建方式 | vSAN 上由 Storage Policy 的 OSR 決定，採 thin 即可，不需要 Eager Zeroed Thick | [34] |
| 記憶體 | 預留量 = SGA + PGA + Oracle 背景程序 + OS 使用量；啟用 Large Pages / HugePages | Oracle Databases on VMware Best Practices Guide [36] |
| CPU / NUMA | vCPU 數量以不跨 NUMA node 為佳；高負載 DB 可評估 CPU 預留 | 一般建議 |
| Latency Sensitivity | 一般不建議設為 High（需 100% CPU 與記憶體預留、降低整併率），可作為實驗項目 | 一般建議 |
| 虛擬硬體版本 / VMware Tools | 升到對應 vSphere 版本的最新硬體版本與 Tools | 一般建議 |
| Guest OS | Linux I/O scheduler 使用 none 或 mq-deadline、分割區 4K 對齊、使用 ASM 或 XFS；redo log blocksize 512B 與 4K 的選擇依 [40] 評估 | 一般建議 |

### 2.4 vSAN 網路與 cluster 層

- vSAN 使用專用的 VMkernel 與獨立 VLAN；若與其他流量共用 pNIC，使用 vSphere Distributed Switch 並以 Network I/O Control 給 vSAN 較高的 shares [41]。
- 網路速度 25GbE 以上；若目前是 10GbE 的 ESA cluster，請確認容量使用率低於 75%，並依 KB 372309 設定 resync 節流參數 [21][25][26]。
- MTU 9000 端對端一致，Skyline Health 的網路與 MTU 檢查需全綠。
- 測試期間確認沒有 resync / rebalance 進行中；NIC 與 NVMe 的韌體與驅動需為 HCL 上的版本。
- ESA 可評估啟用 RDMA（RoCE v2），需 NIC 與交換器支援（一般建議，待 Lab 驗證）。

### 2.5 Redo / Log File Sync 偏高的判讀方式

依 Oracle 官方文件對等待事件的定義 [42]：

- **log file sync**：使用者 session commit 後，等待 LGWR 把 redo 寫入 redo log file 並回覆的時間。它包含 LGWR 的 I/O 時間、LGWR 被喚醒與回覆的 CPU 排程時間，以及 commit 頻率過高時的排隊時間。
- **log file parallel write**：LGWR 實際把 redo 記錄寫到 redo log file 的 I/O 時間。

判讀原則：

1. **若 log file parallel write 的平均等待也偏高**（與 log file sync 接近）：瓶頸在儲存路徑，對應 2.2 / 2.3 / 2.4 的調整。請同時比對 vSAN Performance Service 的 VM 層寫入延遲與 Backend 延遲，並使用 vSAN I/O Trip Analyzer 檢視是哪一層（虛擬 SCSI、DOM client、網路、DOM owner、磁碟層）貢獻最多延遲 [43]。
2. **若 log file sync 遠大於 log file parallel write**：瓶頸在 CPU 排程或 commit 行為（vCPU %RDY 偏高、LGWR 排程延遲、應用程式 commit 過於頻繁），應從 VM 的 CPU 預留 / NUMA 配置與應用端 commit 行為著手，而不是儲存。
3. Oracle 文件對 LGWR 的一般建議是把 redo log 放在專用磁碟、避免與其他 I/O 競爭 [42]；文件中「不要把 redo log 放在 RAID 5」的說法是針對傳統陣列，vSAN ESA 的 RAID-5 寫入行為不同（見 2.2）。
4. SQL Server 的對應等待事件為 WRITELOG，判讀原則相同。

### 2.6 建議 DBA 再測試的方法

1. **前置檢查**：vSAN Skyline Health 全綠、HCL 核對（NIC / NVMe 韌體與驅動）、MTU 一致、沒有 resync 進行中、記錄 2.1 的環境資訊。
2. **vSAN 層 baseline**：以 HCIBench（fio 引擎）用 8K 隨機寫、outstanding I/O 1 至 4 的 profile 量測 VM 層寫入延遲，作為儲存層基準 [44]。
3. **I/O 特徵**：用 vSAN I/O Insight 擷取 DB VM 的 Redo VMDK 的 I/O 特徵（block size、outstanding I/O、循序比）[45]。
4. **DB 層測試**：使用 SLOB（Broadcom 的 vSAN ESA Oracle 效能測試即採用 SLOB [30]）或 Swingbench / HammerDB，固定相同資料集與測試時間。
5. **一次只改一個變數**，建議順序：(a) Redo 使用獨立 policy（OSA 用 RAID-1、ESA 用 RAID-5）→ (b) controller 配置與 queue depth → (c) 壓縮 / Dedup 開關 → (d) 記憶體預留與 HugePages → (e) 網路 25GbE 與 RDMA。
6. **每輪擷取**：AWR（log file sync、log file parallel write、commits/s、redo MB/s）、vSAN VM 層與 Backend 延遲、I/O Trip Analyzer、esxtop（%RDY、DAVG / KAVG）。
7. **判定方式**：以相對於 baseline 的改善幅度為準。官方文件並未提供特定硬體下的保證延遲數值，我們不會承諾特定數字，但會協助對照 Broadcom 公開的 Oracle on vSAN ESA 測試結果 [30] 判斷是否合理。

---

## 三、後續建議與討論

1. **請貴司提供**：M2 SAN 的協定與型號、候選主機機型（若已有）、2.1 所列的 DB 測試環境資訊、DBA 的測試圖表與 AWR 報表。
2. **我們提供**：候選硬體 BOM 對照 Broadcom Compatibility Guide 與 vSAN ESA ReadyNode 的檢核結果、VCF 9.1 Installer 前置需求清單（IP / DNS / VLAN / MTU）、DB 調校測試計畫書（含 2.6 的步驟與量測表格）。
3. **原廠確認**：由我們協助向 Broadcom Support 就「6 台主機 + VMFS on FC 的 Management Domain greenfield 部署」取得書面確認，避免日後 Support 認定上的落差。
4. **會議**：建議安排 60 至 90 分鐘的討論（前半段架構與硬體規格，後半段與 DBA 討論測試計畫）。麻煩您提供方便的時段，我們會配合安排。

再次感謝您的信任，若有任何問題，隨時與我聯繫。

Best regards,

Jonathan
【職稱】
【公司】
【電話】

---

## 附錄 A、官方文件參考

1. VMware Cloud Foundation 9.1 General FAQs — https://www.vmware.com/docs/vmware-cloud-foundation-9-1-general-faqs
2. Broadcom KB 416270, Supporting all Principal Storage Options in VMware Cloud Foundation 9 — https://knowledge.broadcom.com/external/article/416270/supporting-all-principal-storage-options.html
3. VMware Cloud Foundation Blog (2025-11-11), VMware Cloud Foundation 9: Now Ready For All Storage — https://blogs.vmware.com/cloud-foundation/2025/11/11/vmware-cloud-foundation-9-now-ready-for-all-storage/
4. Broadcom KB 435850, Supported Principal Storage Migration Paths for Management Domains in VCF 9.1 — https://knowledge.broadcom.com/external/article/435850/supported-principal-storage-migration-pa.html
5. VCF 9.1 Deployment Guide, Deploy a New VCF Fleet or a New VCF Instance — https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/deployment/deploying-a-new-vmware-cloud-foundation-or-vmware-vsphere-foundation-private-cloud-/deploy-a-new-vcf-fleet-or-a-new-vcf-instance.html
6. VCF 9.1, Create a New Workload Domain — https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/building-your-private-cloud-infrastructure/working-with-workload-domains/deploy-a-vi-workload-domain-using-the-sddc-manager-ui.html
7. VCF 9.1 Design Blueprints, Minimum Requirements for VCF Fleet in a Single Site with Minimal Footprint — https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/design/design-blueprints-for/infrastructure-modernization/vcf-fleet-in-a-single-site-with-minimal-footprint-blueprint/minimaltemo.html
8. VCF 9.0 Design Library, Tenancy Deployment Model 1: Consolidated Management and Organization Workloads — https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-0/design/design-library/vcf-automation-deployment-models(1)/tenancy-deployment-models-with-vmware-cloud-foundation/model.html
9. VCF 9.0 Release Notes, What's New: NSX（單節點 NSX Manager） — https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-0/release-notes/vmware-cloud-foundation-90-release-notes/platform-whats-new/whats-new-nsx.html
10. VCF 9.1, Supported Scenarios to Converge to VCF — https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/deployment/converging-your-existing-vsphere-infrastructure-to-a-vcf-or-vvf-platform-/supported-scenarios-to-converge-to-vcf.html
11. VMware Cloud Foundation Blog (2025-11-04), vSAN Data Protection in VMware Cloud Foundation — https://blogs.vmware.com/cloud-foundation/2025/11/04/vsan-data-protection-in-vmware-cloud-foundation-the-solution-you-already-own/
12. Broadcom KB 417356, Considerations for Implementing vSphere Metro Storage Cluster (vMSC) on VMware Cloud Foundation 9.x — https://knowledge.broadcom.com/external/article/417356/considerations-for-implementing-vsphere.html
13. Broadcom KB 313548, Counting Cores for VMware Cloud Foundation and vSphere Foundation and TiBs for vSAN — https://knowledge.broadcom.com/external/article/313548/counting-cores-for-vmware-cloud-foundati.html
14. VCF 9.1 Licensing, Licensing Model — https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/licensing/licensing-overview/licensing-model.html
15. VCF 9.1, Commission Hosts（vSAN ESA / OSA HCL 要求） — https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/building-your-private-cloud-infrastructure/host-management/commission-hosts.html
16. VMware Cloud Foundation Blog (2025-11-14), Driving Down Storage Costs with Lower Hardware Requirements for vSAN — https://blogs.vmware.com/cloud-foundation/2025/11/14/driving-down-storage-costs-with-lower-hardware-requirements-for-vsan/
17. Broadcom KB 326717, Supported Hardware Changes: vSAN ESA and vSAN Storage Clusters ReadyNodes — https://knowledge.broadcom.com/external/article/326717/what-you-can-and-cannot-change-in-a-vsan.html
18. VCF 9.1 vSAN, Hardware Requirements for vSAN — https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/vsan-deployment-administration-and-monitoring/vsan-planning-and-deployment/requirements-for-creating-a-virtual-san-cluster/hardware-requirements-for-virtual-san.html
19. ESX 9.0, ESX Hardware Requirements — https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere/9-0/esx-installation-and-setup/installing-and-setting-up-esxi/esxi-requirements/esxi-hardware-requirements.html
20. VMware Cloud Foundation Blog (2024-03-01), Smaller vSAN ESA ReadyNodes — https://blogs.vmware.com/cloud-foundation/2024/03/01/smaller-vsan-esa-readynodes-to-accommodate-vmware-vsphere-foundations-trial-capacity-capability/
21. Broadcom KB 317674, vSAN Health Service: Physical NIC link speed meets minimum requirements — https://knowledge.broadcom.com/external/article/317674/vsan-health-service-hardware-compatibil.html
22. VMware Cloud Foundation Blog (2023-08-01), ESA for All Platforms and All Workloads with ESA-AF-0 — https://blogs.vmware.com/cloud-foundation/2023/08/01/esa-for-all-platforms-and-all-workloads-with-esa-af-0/
23. VCF 9.1, Planning and Preparation — https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/planning-and-preparation.html
24. Broadcom KB 446573, VCF Installer pre-validation fails at NFS Datastore Configuration in VCF 9.1（MTU） — https://knowledge.broadcom.com/external/article/446573/vcf-installer-prevalidation-fails-at-nfs.html
25. Broadcom KB 372311, System performance may degrade when cluster capacity utilization exceeds 75% and vSAN ESA is configured with 10Gb networking — https://knowledge.broadcom.com/external/article/372311/system-performance-may-degrade-when-clus.html
26. Broadcom KB 372309, Workaround to reduce impact of resync traffic in vSAN ESA clusters utilizing a 10G network — https://knowledge.broadcom.com/external/article/372309/workaround-to-reduce-impact-of-resync-tr.html
27. Oracle Database on VMware vSAN 6.7 Reference Architecture — https://core.vmware.com/resource/oracle-database-vmware-vsan-67
28. VMware Cloud Foundation Blog (2022-09-02), RAID-5/6 with the Performance of RAID-1 using the vSAN Express Storage Architecture — https://blogs.vmware.com/cloud-foundation/2022/09/02/raid-5-6-with-the-performance-of-raid-1-using-the-vsan-express-storage-architecture/
29. VCF 9.1 vSAN, Selecting the Best RAID Configuration for a vSAN Storage Cluster — https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/vsan-deployment-administration-and-monitoring/administering-vmware-vsan/increasing-space-efficiency-in-a-vsan-cluster/using-raid-5-6-erasure-coding-in-vsan-cluster.html
30. VMware Cloud Foundation Blog (2024-03-15), VMware vSAN 8 Express Storage Architecture (ESA) for Oracle Workloads – Performance — https://blogs.vmware.com/cloud-foundation/2024/03/15/vsa8-esa-oracle-performance/
31. VMware Cloud Foundation Blog (2026-05-08), Auto-RAID in VMware vSAN for VCF 9.1 — https://blogs.vmware.com/cloud-foundation/2026/05/08/auto-raid-in-vsan-for-vcf-9-1/
32. VMware Cloud Foundation Blog (2022-11-30), Stripe Width Storage Policy Rule in the vSAN ESA — https://blogs.vmware.com/cloud-foundation/2022/11/30/stripe-width-storage-policy-rule-in-the-vsan-esa/
33. vSAN 8, What are vSAN Policies（IOPS limit 以 32 KB 為單位） — https://techdocs.broadcom.com/us/en/vmware-cis/vsan/vsan/8-0/vsan-administration/using-vsan-policies/about-vsan-policies.html
34. VMware Cloud Foundation Blog (2020-11-16), Thick vs. Thin on vSAN and VMFS – 2020 Edition — https://blogs.vmware.com/cloud-foundation/2020/11/16/thick-vs-thin-on-vsan-2020-edition/
35. VMware Cloud Foundation Blog (2020-09-16), PVSCSI Controllers and Queue Depth – Accelerating performance for Oracle Workloads — https://blogs.vmware.com/cloud-foundation/2020/09/16/pvscsi_queue_oracle_asm_disk/
36. Oracle Databases on VMware Best Practices Guide — https://www.vmware.com/docs/vmware-oracle-databases-on-vmware-best-practices-guide
37. vSphere 8, SCSI, SATA, and NVMe Storage Controller Conditions, Limitations, and Compatibility — https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere/8-0/vsphere-virtual-machine-administration/configuring-virtual-machine-hardwarevsphere-vm-admin/scsi-controller-configurationvsphere-vm-admin.html
38. Broadcom KB 327037, Using Oracle RAC on a vSAN Datastore — https://knowledge.broadcom.com/external/article/327037/using-oracle-rac-on-a-vsan-datastore.html
39. Broadcom KB 343323, Large-scale workloads with intensive I/O patterns might require queue depths significantly greater than Paravirtual SCSI default values — https://knowledge.broadcom.com/external/article/343323/largescale-workloads-with-intensive-io-p.html
40. VMware Apps Blog (2022-01), Oracle Workloads and Redo Log Blocksize – 512 bytes or 4k — https://blogs.vmware.com/apps/2022/01/oracle-redo-blocksize-512b-4k.html
41. VCF 9.0 vSAN, Designing and Sizing vSAN Hosts（網路與 NIOC） — https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-0/vsan-deployment-administration-and-monitoring/vsan-planning-and-deployment/designing-and-sizing-a-virtual-san-cluster/designing-and-sizing-virtual-san-hosts.html
42. Oracle Database Reference, Descriptions of Wait Events — https://docs.oracle.com/en/database/oracle/oracle-database/26/refrn/descriptions-of-wait-events.html
43. vSAN 8, Use vSAN I/O Trip Analyzer — https://techdocs.broadcom.com/us/en/vmware-cis/vsan/vsan/8-0/vsan-monitoring/monitor-virtual-san-performance/use-vsan-io-trip-analyzer.html
44. HCIBench（Broadcom Developer Portal） — https://developer.broadcom.com/tools/hcibench/latest
45. vSAN 8, Use vSAN I/O Insight — https://techdocs.broadcom.com/us/en/vmware-cis/vsan/vsan/8-0/vsan-monitoring/monitor-virtual-san-performance/use-vsan-i-o-insight.html
46. VMware Cloud Foundation Blog (2023-01-01), Performance Recommendations for vSAN ESA — https://blogs.vmware.com/cloud-foundation/2023/01/01/performance-recommendations-for-vsan-esa/
47. Broadcom KB 392993, Minimum number of ESXi hosts required on vSAN clusters for deployment of VCF Management Domain — https://knowledge.broadcom.com/external/article/392993/minimum-number-of-esxi-hosts-required-on.html
48. VCF 9.1 Design, Storage Models — https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/design/vmware-cloud-foundation-concepts/storage-models.html
49. Broadcom KB 432318, Commissioned hosts are not visible during Workload Domain or Cluster creation when using VMFS_FC storage in VCF 9.x — https://knowledge.broadcom.com/external/article/432318/commissioned-hosts-are-not-visible-durin.html

---

## 附錄 B、內部備註（寄出前請刪除）

### B-1 主要技術主張的驗證狀態

| 回覆段落 | 主張 | 驗證狀態 | 依據 / 備註 |
|---|---|---|---|
| 1.1 / 1.3 | VCF 9.x greenfield 的 Management Domain 可用 VMFS on FC / NFSv3 作為 principal storage | 官方文件已確認 | KB 416270、VCF 9.1 FAQ、Installer 文件 [5]、官方部落格 [3]。注意 Storage Models 頁 [48] 與 KB 392993 [47] 仍有舊敘述，已在 1.3 對客戶說明並建議開 Support case |
| 1.1 | Management Cluster 最少 4 台主機（不分 vSAN / NFS / FC） | 官方文件已確認 | FAQ [1]、部落格 [3]。審查者發現 VCF Installer 本身可能允許更少主機（vSAN 3 台、外部儲存 2 台）但官方指引為 4 台，回覆採 4 台 |
| 1.1 | iSCSI / NFS v4.1 / FCoE / NVMe-oF 不在 greenfield 流程 | 官方文件已確認 | KB 416270 [2] |
| 1.1 | FC datastore 需事先建好並掛載所有主機、同名 | 官方文件已確認 | Installer 文件 [5]；KB 432318 [49]（主機未列出）僅標題層級確認，寄出前請開啟 KB 核對內容 |
| 1.2 | VI Workload Domain 使用 NFS / VMFS on FC 且用 vLCM image 時最少 2 台；Workload Management 需 3 台 | 官方文件已確認 | 9.1 techdocs [6] |
| 1.2 | Minimal Footprint 藍圖：管理元件與 workload 同一 cluster | 官方文件已確認 | 9.1 design blueprint [7]；藍圖表格中的最低主機數未取得引句，未寫入正文 |
| 1.2 | Cluster principal storage 建立後不可更改；同 domain 不同 cluster 可用不同 principal storage | 官方文件已確認 | [5][6] |
| 1.2 | KB 435850：Management Domain 僅支援同類型轉換與 vVol→vSAN | 官方文件已確認 | [4]（2026-05 更新） |
| 1.2 | M1 converge / import 條件（vCenter 8.0 U3a+ 且 NSX 4.2+；無 NSX 需先升 vCenter 9.1） | 官方文件已確認 | 9.1 release notes / converge 文件 [10]。審查者補充：若 M1 已是 8.0 U3j（2026-05）以上，目標需為 VCF 9.1.1 而非 9.1.0，待原廠確認 |
| 1.3 | vSAN Data Protection / Snapshot Service 需 ESA | 官方文件已確認 | 部落格 [11] |
| 1.3 | VCF 授權每 core 含 1 TiB vSAN；每 CPU 最少 16 cores | 官方文件已確認 | KB 313548 [13]、9.1 licensing [14] |
| 1.4 | vSAN ESA 最少 128 GB RAM；10GbE 最低、25GbE 建議、延遲 1 ms | 官方文件已確認 | 9.1 / 9.0 vSAN 文件 [18] |
| 1.4 | ESA-AF-0 可少至 1 顆 1.6 TB NVMe（2024-03 部落格）、原始 AF-0 為 2 顆以上（2023-08 部落格）；1 DWPD、Class F | 官方文件已確認 | [20][22][17]。「每台 4 顆以上」為我們的建議，非官方要求 |
| 1.4 | 10GbE 僅 AF-0 profile 允許；Health check 以 25 Gbps 彙總為基準；KB 372311 / 372309 | 官方文件已確認 | [22][21][25][26] |
| 1.4 | ESX 9 開機裝置 32 GB 以上、新機建議 128 GB | 官方文件已確認 | [19] |
| 1.4 | CPU 24–32 cores、RAM 512 GB、2×25GbE、2×FC HBA | 我們的規劃建議 | 非官方最低要求，已在表格以「建議採購規格」欄區分 |
| 1.5 | NSX Manager 必要；VCF 9.0 起可單節點 | 官方文件已確認（9.0 release notes [9]） | 9.1 Installer 精靈是否提供單節點選項，待原廠確認 |
| 1.5 | VLAN 清單、Overlay MTU 1600 | 一般設計知識 | 未取得 9.1 引句；正文已請客戶以 Installer planning 產出為準 |
| 1.5 | VCF Management Services IP pool Simple 6 / HA 8 | 內部簡報資料 | 來自內部 VCF 9.1 升級簡報，非公開文件；正文標示「待原廠確認」 |
| 2.2 | ESA RAID-5/6 效能與 RAID-1 相當 | 官方文件已確認 | 部落格 [28][46]、9.1 techdocs [29]、Oracle ESA 測試 [30] |
| 2.2 | OSA 上 redo 用 RAID-1 | 官方文件已確認（舊版參考架構） | Oracle on vSAN 6.7 RA [27]："For online redo log disks, RAID 1 policy is used for performance"（OSA 時代文件） |
| 2.2 | OSA RAID-5/6 parity read-modify-write 增加寫入延遲 | 未取得官方引句 | 正文標示為「我們的實務經驗」 |
| 2.2 | Stripe width 保留 1（ESA 不需要；OSA 僅除錯時實驗） | 官方文件已確認 | 部落格 [32] |
| 2.2 | IOPS limit 以 32 KB 正規化、0 = 不限制 | 官方文件已確認 | vSAN 8 policies [33] |
| 2.2 | Thin（OSR 0）為官方建議 | 官方文件已確認 | 部落格 [34] |
| 2.2 | ESA 壓縮為 policy 層設定、預設值 | 未完全確認 | 搜尋結果未證實「預設啟用」；正文改以「先做 baseline、再以獨立 policy 測試」處理，標示待 Lab 驗證 |
| 2.2 | VCF 9.1 新建 cluster 預設 Auto-RAID；Auto-Policy Management 不應在 9.1 使用 | 官方文件已確認 | 部落格 [31]。Auto-RAID 下對特定 VM 用自訂 policy 的建議作法，待原廠確認 |
| 2.3 | PVSCSI 用滿 4 個、queue depth 調整值 | 官方文件已確認 | 部落格 [35]、KB 343323 [39]（Linux 模組參數名稱依 KB 內容，寄出前請再核對） |
| 2.3 | vSphere 建議 NVMe 或 PVSCSI controller；RAC multi-writer 不可用 vNVMe | 官方文件已確認 | [37][38] |
| 2.3 | 記憶體預留公式、Large Pages | 官方文件已確認 | Oracle BP Guide [36] |
| 2.3 | NUMA、Latency Sensitivity、硬體版本、Guest I/O scheduler | 一般最佳實務 | 未取得官方引句，正文標示「一般建議」 |
| 2.4 | vSAN 專用 VLAN、vDS + NIOC | 官方文件已確認 | [41] |
| 2.4 | RDMA on ESA | 一般建議 | 標示待 Lab 驗證 |
| 2.5 | log file sync / log file parallel write 定義與 LGWR 建議 | Oracle 官方文件已確認 | [42] |
| 2.6 | I/O Trip Analyzer、I/O Insight、HCIBench、SLOB | 官方文件已確認 | [43][45][30]；HCIBench 連結 [44] 取自 developer.broadcom.com，寄出前請點開確認 |
| 2.6 | 預期延遲數值 | 無官方數值 | 正文明確不承諾特定數字 |

### B-2 寄出前待辦

1. 向 Broadcom Support 開立 case：主題「VCF 9.1 greenfield Management Domain on VMFS-FC, 6 hosts, consolidated」，取得書面確認後回覆客戶。
2. 確認客戶 M2 SAN 協定（FC / iSCSI / NVMe-oF）與 M1 vCenter 確切版本。
3. 附錄 A 的 49 個連結均由 Broadcom / VMware 官方網域搜尋結果取得，但本次撰稿環境無法直接開啟網頁，寄出前請逐一點開確認可連線且內容相符（特別是 [27] core.vmware.com 與 [44] developer.broadcom.com 可能已改版或轉址）。
4. 若客戶的 vSAN 測試環境是 VCF 9.1 + Auto-RAID，2.2 的 policy 建議需依 Auto-RAID 的行為調整；請先取得 2.1 的環境資訊再定稿。
5. 簽名欄【職稱】【公司】【電話】待填；刪除本附錄 B 與檔頭的檔案說明。
