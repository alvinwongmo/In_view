🧠 一、什麼是「橫向移動 (Lateral Movement)」

定義說法：
橫向移動是攻擊者在入侵網絡後，為了擴大控制權或尋找更高價值目標（例如 Domain Controller 或財務伺服器）而在內部網絡中橫向探索、嘗試登入其他系統的過程

常見手法包括：
利用被竊取的憑證進行 Pass-the-Hash / Pass-the-Ticket
濫用遠端協定（RDP、SMB、WMI、PsExec）
植入惡意程式或後門到橫向節點
利用共用檔案夾或共用帳號傳播

為什麼這很危險？
因為現代企業多為扁平網絡（Flat network），一旦攻入任何主機，攻擊者可輕易存取整個企業的關鍵資產
因此「網絡分段」就是阻斷橫向移動最有效的結構性防禦手段。


🧩 什麼是 Flat Network（扁平網絡）
定義：
Flat network 指的是整個內部網絡中沒有明確的區域劃分（Segmentation），
所有設備（伺服器、用戶端、IoT、管理端）都在同一層網段或同一個信任區域中，
可以彼此直接通信，不需經過防火牆或ACL控制。

🔥 為什麼 Flat Network 無法阻擋橫向攻擊（Lateral Movement）
攻擊者一旦入侵一台主機，就能橫向掃描整個網段
攻擊者可用工具如 nmap 掃描整個子網，找出其他主機與開放埠
例如：nmap -p 445,3389 10.0.0.0/24
若有未修補漏洞或弱密碼主機，即可繼續入侵

🔥監控可見性極低
沒有分區就無法設定特定區域流量監控。
IDS / NDR 難以分辨「正常內部流量」與「異常橫向流量」。
攻擊活動（例如從一台電腦到另一台大量登入嘗試）容易被忽略。


🧱 二、網絡分段的核心目的與原則

目的：
限制攻擊範圍與影響面。
控制不同信任區之間的流量與通訊方向。
提升事件偵測與應變效率（縮小可疑活動範圍）。

三大原則：
1️⃣ 最小信任區原則 (Least Trust Zone)
→ 每個區域僅允許必要流量通行
2️⃣ 明確邊界原則 (Explicit Boundary)
→ 使用 VLAN、防火牆、SDN 或微分段工具建立明確邊界
3️⃣ 可監控原則 (Visibility & Monitoring)
→ 對跨區流量進行持續監控與日誌審計


🧩 三、網絡分段的實施層級

下面我分二個層級詳細說明：

🔹 1. 網絡層級分段（Network-Level Segmentation）
技術手段：
使用 VLAN / Subnet 將業務功能或信任層分離，例如：
User VLAN、Server VLAN、Database VLAN、Management VLAN、DMZ
於 L3 Switch 或防火牆設定 ACL / Zone Policy：
僅允許明確流量（例如 Web → App → DB），拒絕預設所有其他流量
關鍵管理操作僅允許從專用 Jump Server 經多因素驗證（MFA）

\
🔹 2. 微分段（Microsegmentation）
進階技術，用於防止內部橫向擴散
實現方式：
使用 SDN / Overlay Network 工具（如 VMware NSX、Cisco ACI、Illumio、Guardicore）
在 VM / Workload 層面設定安全策略（每台主機只允許指定流量）
以標籤（Tag-based Policy）管理，減少依賴IP

中文摘要版：
「橫向移動指攻擊者入侵後利用憑證或內部工具橫向擴散
我會透過多層網絡分段防止此行為，包括VLAN與防火牆劃分使用者、伺服器、管理及DMZ區
在應用層限制跨系統流量；進階環境中使用微分段隔離VM
最後結合SIEM監控與流量分析，實現持續偵測與防禦」

🧱 網絡分段示意拓撲圖

                         ┌────────────────────────────┐
                         │          Internet          │
                         └──────────────┬─────────────┘
                                        │
                                [ External Firewall ]
                                        │
                            ┌───────────┴───────────┐
                            │                       │
                  ┌─────────▼─────────┐     ┌──────▼──────┐
                  │      DMZ Zone      │     │ VPN Gateway │
                  │ (Web, Reverse Proxy│     │  (MFA Access)│
                  └─────────┬─────────┘     └──────┬──────┘
                            │                       │
                            │                       │
                [ Internal Firewall / Segmentation Policy ]
                            │
        ┌───────────┬──────────────┬───────────────┐
        │           │              │               │
┌───────▼───────┐┌──▼────────┐┌───▼─────────┐┌────▼─────────┐
│ User VLAN     ││ Server VLAN││ Database VLAN││ Mgmt VLAN   │
│ (Workstations)││ (App, API) ││ (DB servers) ││ (Jump Hosts)│
└───────┬───────┘└──┬────────┘└───┬─────────┘└────┬─────────┘
        │            │              │               │
        │            │              │               │
        │            │              │      [ Access via SSH/RDP
        │            │              │         from Jump Host Only ]
        │            │              │
        └────────────┴──────────────┴───────────────────────┘
                        │
              [ East-West Traffic Monitored by NDR / IDS ]
                        │
                        ▼
             ┌─────────────────────────────┐
             │   SIEM / SOC Monitoring     │
             │ (Correlate logs & alerts)   │
             └─────────────────────────────┘


🔹 微分段（Microsegmentation）

從「每一台主機 / VM / Container」的角度定義通訊規則
以 應用角色或標籤（App Role Tag） 決定誰能與誰通訊

例如：
Allow App01 (role=frontend) → DB01 (role=database) on TCP 3306
Deny  App01 → App02 (same VLAN)

即使兩台主機在同一 VLAN，也可以禁止互通。
可以隨著工作負載搬移、擴展自動更新規則。


🧠 微分段（Microsegmentation）延伸設計

在上圖的每個 VLAN 內，再使用 微分段 (Microsegmentation) 進行細化控制，例如：

[Server VLAN]
 ├─ App01  ↔  App02  (Blocked)
 ├─ App01  ↔  DB01   (Allowed: TCP 1433)
 ├─ App02  ↔  DB02   (Allowed: TCP 1521)

這樣即使攻擊者入侵一台伺服器，也無法直接掃描或連線同 VLAN 內其他主機，
這是防止橫向移動最關鍵的最後一道牆。
 
