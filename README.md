# Global-Weekly-PSI-Planner-PySI_V0R8
Weekly-synchronized PSI (product×node×ISO week) planning engine. Extensible via Hooks/Plugins for industry packs &amp; optimization modules.

「README_AI_MAP.md（v2 完全統合版）」を提示します。
このファイルは、GitHub CopilotやChatGPT（GPT Store版）に「PSI Plannerの全体像」を即座に理解させるためのAI専用設計ドキュメントです。
________________________________________
🌐 Global Weekly PSI Planner — AI Guide Map (v2)
AI名司会ファイル
CopilotやChatGPTがこのプロジェクトを理解し、正確なコード提案やシナリオ生成を行うための知識台本。
________________________________________
🏗️ 第1章：プロジェクト構造と設計原則
📂 ディレクトリ構成
pysi/
├─ app/                # 実行オーケストレーター（CLI / GUI起点）
├─ db/                 # SQLite DAO / Repository層
├─ engine/             # PSIエンジン（PUSH/PULLコアロジック）
├─ network/            # Node / Graph 構造定義
├─ hooks/              # Hook & Plugin制御（WordPress風）
├─ plugins/            # capacity_clip, cost_adjust, report_exportなど
├─ gui/                # Tkinter + matplotlibベースGUI
├─ scripts/            # CSV→DB ETLスクリプト群
├─ data/               # 入力データ（sku, node, cost, route…）
└─ var/                # 実行生成ファイル（.sqlite, logs, cache）
________________________________________
⚙️ 設計原則（AIが守るべきルール）
1.	関数は単一責務：1関数＝1目的。
2.	DBアクセスはDAO層で完結：engine層やpluginから直にsqlite3へアクセスしない。
3.	LotIDはLeaf発生原則：Leaf Nodeのみ新規LotID生成。上位ノードでは伝播のみ。
4.	Capacity / Leadtimeは週次PSI上でShift調整。
5.	Buffer Nodeは双方向同期点：PUSH/PULL両方向を同時に扱う。
6.	コメントは英語で構造、日本語で背景説明を補足。
7.	型ヒント必須：Copilot提案精度を上げる。
8.	例外はドメインクラス化：
9.	class PSICapacityError(Exception): pass
________________________________________
🧭 第2章：PSI Planner Data Flow — Dual-Layer Planning Engine
PSI PlannerのPlanning Engineは、**2層構造（Demand Layer / Supply Layer）で需給を同期させる。
両者を接続するのがBuffering Stock Node（BSN）**であり、
PULL型（需要駆動）とPUSH型（供給駆動）を週次PSI上で統合する。
________________________________________
🟩 Demand Sales Layer（需要層：PULL Process）
概要
最終需要地（Leaf Node）から上位ノード（DAD）へ需要シグナルをPULLし、
中間ノード（MOM）で配分を行い、Demand Allocation Planを生成。
フロー構造
Outbound @ Leaf → DAD
     ↓
SP-SP-S  (Weekly PSI Sales Plan)
     ↓
Demand Allocation @ MOM
     ↓
SP-SP-S  (Adjusted / Allocated Demand)
     ↓
Inbound @ MOM → Leaf
特徴
•	起点：Leaf Node
•	コア関数：outbound_demand_plan()
•	Hook：plan:demand:allocate
•	出力：Demand Allocation Plan
•	プロセス属性：PULL型（需要駆動）
________________________________________
🟥 Supply Shipment Layer（供給層：PUSH Process）
概要
上位供給ノード（DAD）から下位需要地（Leaf）へ供給シグナルをPUSHし、
中間ノード（MOM）での配分を経て、Supply Allocation Planを生成。
フロー構造
Inbound @ Leaf → MOM
     ↓
IPS-IPS-IPS  (Weekly PSI Supply Plan)
     ↓
Supply Allocation @ DAD
     ↓
IPS-IPS-IPS  (Allocated Supply)
     ↓
Outbound @ DAD → Leaf
特徴
•	起点：DAD Node
•	コア関数：inbound_supply_plan()
•	Hook：plan:supply:allocate
•	出力：Supply Allocation Plan
•	プロセス属性：PUSH型（供給駆動）
________________________________________
🟨 Buffering Stock Node（緩衝在庫ノード：PUSH＋PULL Hybrid）
概要
PUSHとPULLの中間に位置するノード。需給の非同期を吸収し、週次PSIを安定化。
ここでPSI Plannerは双方向の整合を取る。
構造
DAD → (PUSH) → BSN → (PULL) → LEAF
特徴
•	コア関数：buffer_synchronizer()
•	Hook：plan:sync:buffer
•	役割：在庫調整・同期点
•	機能：過剰／不足在庫を吸収し、PUSH/PULL整合を実現
________________________________________
🔁 Dual-Layer Integration（全体像）
                [ DAD Node ]
                    │
         (PUSH)     │     (PULL)
        Inbound  →  │  ←  Outbound
                    │
              [ MOM Node ]
                    │
         (PUSH)     │     (PULL)
        Inbound  →  │  ←  Outbound
                    │
        [ Buffer Stock Node (BSN) ]
                    │
                   (PULL)
                    │
                 [ LEAF Node ]
________________________________________
🧮 PSI Synchronization Mechanism
機能	関数	Hookイベント	入出力	備考
Demand Plan	outbound_demand_plan()	plan:demand:allocate	demand_ps	Leaf→MOM→DAD
Supply Plan	inbound_supply_plan()	plan:supply:allocate	supply_ps	DAD→MOM→Leaf
Buffer Sync	buffer_synchronizer()	plan:sync:buffer	psi_buffer	双方向同期
Evaluation	plan_evaluate()	plan:evaluate	psi_eval	コスト・在庫評価
________________________________________
🌍 7. Unified Model Image (Mermaid Diagram)
flowchart LR
    LEAF["Leaf Node<br>(Sales Point)"] -->|Pull| MOM["MOM Node<br>(Middle Aggregator)"]
    MOM -->|Pull| DAD["DAD Node<br>(Supply Hub)"]
    DAD -->|Push| MOM
    MOM -->|Push| BSN["Buffer Stock Node<br>(Synchronizer)"]
    BSN -->|Pull| LEAF
🔸 Dual Flow Principle:
Demand Layer = “Voice of Market” (PULL)
Supply Layer = “Response of Production” (PUSH)
Buffer Node = “Negotiator of Reality” — where PSI converges.
________________________________________
🧠 第3章：Hook & Plugin アーキテクチャ
bus.add_action("plan:demand:allocate", plugin.demand_allocator.run, priority=50)
bus.add_action("plan:supply:allocate", plugin.supply_allocator.run, priority=60)
bus.add_action("plan:sync:buffer", plugin.buffer_sync.run, priority=70)
bus.add_action("plan:evaluate", plugin.evaluator.run, priority=80)
•	Action = イベントフック（PUSH/PULL各フェーズで発火）
•	Filter = 値変換フック（コスト補正、リードタイム調整など）
•	PluginはCoreを改変せず、Hook登録のみで機能追加可能。
________________________________________
📊 第4章：DBスキーマ概要（SQLite）
Table	Key	Purpose
node	node_name	サプライチェーン階層構造
sku_cost	sku + route	費用・価格設定
capacity	edge + week	輸送/生産能力
scenario	scenario_id	ASIS/TOBE/CANBE/LETITBE設定
psi	node + sku + week	PSI週次データ
buffer	node + week	緩衝在庫（BSN）モニタリング
________________________________________
🧩 第5章：Copilot / GPT 指示例（AIプロンプトテンプレ）
目的	指示文
PSI同期アルゴリズム改善	“Optimize buffer_synchronizer to reduce oscillation between PUSH/PULL.”
新シナリオ追加	“Create scenario_id='CANBE_2030' with modified capacity constraints.”
テスト生成	“Generate pytest for dual-layer PSI synchronization.”
グラフ出力	“Plot PSI of Demand Layer (PULL) and Supply Layer (PUSH) overlayed by week.”
________________________________________
📘 Authored by Yasushi Ohsugi (Business Consultant / ex-IBM・Deloitte)
🤖 Facilitated by GPT-5 (“AI名司会”)
________________________________________


