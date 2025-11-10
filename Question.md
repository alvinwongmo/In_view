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
        │            │              │         [ Access via SSH/RDP
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

即使兩台主機在同一 VLAN，也可以禁止互通
可以隨著工作負載搬移、擴展自動更新規則


🧠 微分段（Microsegmentation）延伸設計

在上圖的每個 VLAN 內，再使用 微分段 (Microsegmentation) 進行細化控制，例如：

[Server VLAN]
 ├─ App01  ↔  App02  (Blocked)
 ├─ App01  ↔  DB01   (Allowed: TCP 1433)
 ├─ App02  ↔  DB02   (Allowed: TCP 1521)

從「每一台主機 / VM / Container」的角度定義通訊規則
以 應用角色或標籤（App Role Tag） 決定誰能與誰通訊

這樣即使攻擊者入侵一台伺服器，也無法直接掃描或連線同 VLAN 內其他主機
即使兩台主機在同一 VLAN，也可以禁止互通。這是防止橫向移動最關鍵的最後一道牆
 
🧩 微分段（在每個主機 / 應用層面加上細粒度控制）

                  ┌──────────────────────────────┐
                  │        Server VLAN           │
                  │     (e.g. 10.10.2.0/24)      │
                  └──────────────────────────────┘
                             │
          ┌──────────────────┼───────────────────┐
          │                  │                   │
   ┌──────▼──────┐     ┌─────▼──────┐      ┌─────▼──────┐
   │ App01 (Tag:frontend)│ │ App02 (Tag:api)│ │ DB01 (Tag:db)│
   └──────────────┘     └────────────┘      └────────────┘

   🔹 微分段策略：
       - App01 → DB01 : ALLOW (TCP 3306)
       - App02 → DB01 : DENY
       - App01 ↔ App02 : DENY
       - 其他所有流量：DENY by default

   🔸 結果：
       ✅ 即使在同一 VLAN，也有獨立防火牆規則。
       ✅ 攻擊者入侵 App02，無法連 App01 或 DB01。
       ✅ 完全阻斷橫向移動（East-West Traffic）。


🧠 舉實際例子比較

假設你有以下三層架構（10 台伺服器）：

類別	伺服器	                IP
Web	  Web01, Web02	        10.10.1.10 / 11
App	  App01, App02, App03	  10.10.2.10~12
DB	  DB01, DB02	          10.10.3.10 / 11

🔹 傳統防火牆做法：
你可能要寫這樣的 ACL：

permit 10.10.1.10 443 -> 10.10.2.10
permit 10.10.1.10 443 -> 10.10.2.11
permit 10.10.1.11 443 -> 10.10.2.10
permit 10.10.1.11 443 -> 10.10.2.11
permit 10.10.2.10 3306 -> 10.10.3.10
permit 10.10.2.11 3306 -> 10.10.3.10

➡️ 每多一台主機就多幾條規則；若 IP 改變，就要重新部署 ACL。
這是「靜態綁定的 1 對 1 控制」。

🔹 微分段做法：
你只要設定 2 條規則：

ALLOW tag=Web TO tag=App ON tcp/443
ALLOW tag=App TO tag=DB ON tcp/3306

系統會自動比對帶有該標籤的所有主機（Web01, Web02, App01...）並下放 Host-based policy。

➡️ 新增 Web03 時，只要給它 tag=Web，就自動繼承所有相同策略，完全不需修改任何 ACL。


🧩 為什麼防火牆更適合「外部 ↔ 內部」流量

🔹 1. 防火牆具備邊界防護的設計優勢

位於 網絡邊界（edge / perimeter），天然是流量進出的必經點。

能進行：
封包過濾（Packet Filtering）
深層檢查（DPI / IPS / Anti-DDoS）
NAT / Port Forwarding / VPN Termination
適合抵禦外部攻擊，如 DDoS、惡意掃描、入侵嘗試。

👉 微分段沒有這些功能，也不在外部流量的路徑上

🔹2. 防火牆支援複雜協定與 Proxy 功能

外部進來的流量通常包含多種應用協定（HTTP, HTTPS, FTP, SMTP...）
防火牆可解析協定內容、支援 WAF（Web Application Firewall）、SSL Inspection
可與 IDS/IPS、Proxy、Sandbox 整合
微分段只針對已授權的內部通信，不負責協定層檢查

🔹 3. 防火牆可集中管理外部信任區域

典型企業網絡會有分區：
[Internet]
   ↓
[Firewall DMZ Zone]
   ↓
[Internal Network]

DMZ：放公開服務（Web / Email / Reverse Proxy）
防火牆可同時控制：
外部 → DMZ
DMZ → 內部
外部 → 內部（直接禁止）

這種邊界概念在微分段中不存在（微分段是內部細粒度控管）

🔹 4. 防火牆適合高流量與異質環境

防火牆的硬體/虛擬化設備通常具備高速 ASIC 或專用芯片
能承受數十 Gbps 的外部流量
微分段則偏向軟體層（Host Agent / vSwitch Policy），不適合直接面對高強度外部流量

🧠 為什麼微分段不適合外部通信
限制	                    原因
不在外部路徑上	            微分段通常部署於資料中心內部、雲端 VPC、或虛擬化環境中。外部流量不會經過它
設計目的不同	              微分段的目的在限制內部主機間的通信，而非分析或抵禦外部攻擊
缺乏流量代理與協定解析能力	沒有 NAT、Proxy、SSL Inspection 等能力
以「內部信任區域」為前提	  它假設外部流量已被邊界防火牆清理過

總結:
防火牆位於邊界，設計上能處理大量外部流量、協定檢查、NAT 與入侵防禦，非常適合外部與內部之間的通信。
微分段則部署在內部環境，專注於主機與應用間的東西向控制，用於防止橫向移動。
因此兩者互補：防火牆守外，微分段守內。
