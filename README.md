# WSO:Weekly Syncronied Operation / Global-Weekly-PSI-Planner-PySI_V0R8
以下はAI(chatGPT)との会話から生成されたWSOの仕様書です。

WSO:Weekly Syncronied Operation / PySI / Weekly PSI Planner (V0R8) — Core Spec (Dual-Tree + Lot-ID Lists)
日付: 2025-10-23 (JST)
正本ポリシー: V0R7「二本のツリー × 各ノードに週×[S,CO,I,P]（lot_IDのリスト）」をそのまま正本としてV0R8に継承。高速化は実装テクニックで達成。
数量の定義: すべて「ロットは不可分」、数量 = len(その週・そのバケツの lot_ID リスト)。
________________________________________
0. 変更点サマリ（今回更新の骨子）
•	正本データ構造を V0R7 準拠（dual-tree + PSI buckets = lot_ID lists）に確定。
•	数量は常に len(list)。(lot_id, qty) ペアは不採用。
•	週次フローは「イベント→P(w) → I(w)=I(w-1)+P(w) → 出荷 S(w) を I(w) から移動」の順序を厳守。
•	CSV アダプタは8つのCSV（V0R7レイアウト）をすべてDataFrameで raw に格納（パス文字列は不可）。
•	Pipeline契約：spec/raw/root/state のデータ契約を明示。scenario:preload で spec を拡張。
•	Hooks/Plugins：割付は**lot個数（n_lots）**でやり取り／FEFO/FIFOやcapacityは mutate フックで加工。
•	高速化：lot_poolはメタ参照、Iのコピー最小化（copy-on-write 検討）、FEFO並び替え、バッチ処理など。
•	実運用パッチ（5点）：SYN IDの決定論化、I消費の正確性、CO表現方針、horizon境界のガード、二重決済回避。
________________________________________
1. 目的・スコープ
•	目的: 週次PSI（S/CO/I/P）をlot_IDリストの移動だけで成立させるV0R7正本を維持しつつ、Hooks/Pluginsによる拡張性と実用性能をV0R8で両立する。
•	スコープ:
o	データ構造（dual-tree, PSI buckets, lot_ID）
o	週次フロー（イベント反映→Iロールフォワード→出荷）
o	I/O（CSV/SQL）、Pipeline、Hooks/Pluginsの契約
o	高速化と実運用の注意点
o	GUI/CLI 整合
________________________________________
2. 正本データモデル（V0R7継承：Dual-Tree + Lot-ID Lists）
2.1 ツリーとレイヤ
•	二本のツリーを各productごとに維持する：
o	OutBound（需要層）: leaf → DAD（販売側の流れ）
o	InBound（供給層）: source → MOM/PAD → 中間 → leaf（供給側の流れ）
•	いずれも Node/Edge 概念はV0R7と同様（NodeはPSI保持、EdgeはLT/容量/コストなどの属性）。
2.2 PSIバケツ（lot_IDのリスト）
•	各ノードは各週 w に対して、需要層と供給層それぞれに4バケツを持つ：
•	psi4demand[w] = [ S_ids, CO_ids, I_ids, P_ids ]   # すべて list[str]
•	psi4supply[w] = [ S_ids, CO_ids, I_ids, P_ids ]   # すべて list[str]
ここで S, CO, I, P はバケツ種別（インデックスは後述定数）。
•	数量は len(その週のそのバケツ) で常に算出する。
•	lotの属性（exp_week, 製造週, 原価など）は外部辞書 lot_pool: dict[str, LotMeta]に保持（PSIにはIDだけ）。
________________________________________
3. 週次フロー（Canonical Flow：V0R7準拠・lot_IDリスト版）
各週 w の処理順（需要層・供給層とも同様の原理）：
1.	イベント反映 → P(w)
o	到着/完成/受入などのスケジュールイベントをその週 w の P_ids に計上。
o	CO（on-order）は発注時点でCO_idsに積む（表示ポリシーは §10.3 参照）。
2.	Iロールフォワード（I(w) ← I(w-1)）
o	I(w) = deepcopy(I(w-1)) を基本とし、
o	すぐ後の P(w) 取り込みで I(w) に単純 append。
3.	P取り込み（I(w) ← I(w) + P(w)）
o	I(w).extend(P(w))（Pは帳票保持のため残してOK）。
o	FEFOが必要ならこの直後に I(w) を賞味期限等で整列させる（lot_pool 参照）。
4.	出荷（消費）（I(w) → S(w)）
o	割付結果に従い、I(w) から lot_ID を n_lots個 移し、S(w) に積む。
o	不足分は**合成ID（決定論的）**で S(w) を補填（Iには積まない）。
注：週頭のIロールフォワードにより、I(w) は自然に過去の持越しを含むため、遡り消費のために過去週へ戻る必要は通常ない。I(w)のみから消費すれば良い。
________________________________________
4. 定数と最小ユーティリティ（lot_IDリスト専用）
# バケツ定数
PSI_S, PSI_CO, PSI_I, PSI_P = 0, 1, 2, 3

def deterministic_syn_ids(node: str, sku: str, week: int, count: int) -> list[str]:
    """再現性ある合成lot_IDを生成（決定論）"""
    base = f"SYN:{node}:{sku}:{week}:"
    return [f"{base}{i:04d}" for i in range(count)]

def settle_events_to_P(state: dict, week: int):
    """state['scheduled'] のイベントを週wのPに反映（到着/完成）"""
    rest = []
    for ev in state.get("scheduled", []):
        if int(ev.get("to_week", -1)) != week:
            rest.append(ev); continue
        kind = ev.get("type")
        if kind == "receive_demand":
            psi = state["psi_demand"].get((ev["dst"], ev["sku"]))
            if psi:
                # ここでCO→Iの整合やCOからの除去が必要なら実装（表示方針次第）
                psi[week][PSI_P].extend(list(ev.get("lots") or []))
        elif kind == "produce_supply":
            psi = state["psi_supply"].get((ev["node"], ev["sku"]))
            if psi:
                psi[week][PSI_P].extend(list(ev.get("lots") or []))
    state["scheduled"] = rest

import copy

def roll_and_merge_I(psi_map: dict, week: int, fefo=False, lot_pool=None):
    """I(w) ← I(w-1) ロールフォワード → P(w)取り込み（FEFO整列は任意）"""
    if week == 0:
        for psi in psi_map.values():
            if psi[0][PSI_P]:
                psi[0][PSI_I].extend(psi[0][PSI_P])
        return
    for psi in psi_map.values():
        psi[week][PSI_I] = copy.deepcopy(psi[week-1][PSI_I])
        if psi[week][PSI_P]:
            psi[week][PSI_I].extend(psi[week][PSI_P])
        if fefo and lot_pool is not None:
            psi[week][PSI_I].sort(key=lambda lid: (getattr(lot_pool.get(lid), "exp_week", None) or 10**9))

def consume_S_from_I_ids(psi_map: dict, shipments: list, week: int):
    """
    shipments: [{"src":node, "sku":sku, "week":w, "n_lots":int, "wanted":[lid,...]}]
    I(w)から n_lots 個を取り出して S(w) に移す。不足はSYNで補填。
    """
    for s in shipments:
        if int(s.get("week", -1)) != week:
            continue
        node, sku = s["src"], s["sku"]
        n = int(s.get("n_lots", 0))
        wanted = set(s.get("wanted") or [])
        psi = psi_map.get((node, sku))
        if not psi or n <= 0:
            continue

        Iw = psi[week][PSI_I]
        Sw = psi[week][PSI_S]

        # 1) wanted優先
        if wanted:
            i = 0
            while i < len(Iw) and n > 0:
                if Iw[i] in wanted:
                    Sw.append(Iw.pop(i)); n -= 1
                else:
                    i += 1

        # 2) 残りは先頭から
        take = min(n, len(Iw))
        if take > 0:
            Sw.extend(Iw[:take])
            del Iw[:take]
            n -= take

        # 3) 不足は合成lot_ID（決定論）
        if n > 0:
            Sw.extend(deterministic_syn_ids(node, sku, week, n))
________________________________________
5. パフォーマンス（構造不変のまま速くする）
•	lot_pool：lot_id -> meta（期限・原価など）。PSI内はIDのみ → メモリ削減＋アクセス高速。
•	Iロールの最適化：deepcopy は安全だが重い。将来的にcopy-on-write（参照共有→書き込み時コピー）を導入可。
•	FEFO：I(w) を exp_week で整列。挿入時に bisect で順序維持すると週次の sort が不要に。
•	バッチ処理：同一 (node, sku, week) の出荷はまとめて一度に処理。
•	コンテナ：頻繁な先頭取り出しなら deque、FEFO整列なら list ＋ del Iw[:take] で十分高速。
•	並列：product/地域クラスタ単位でプロセス並列が相性良（読み取り共有、PSIは分割）。
________________________________________
6. コア契約（spec / raw / root / state）
6.1 spec（Pipeline入力仕様）
{
  "db_path": "examples/scenarios",
  "scenario_id": "v0r7_rice",
  "calendar": { "weeks": 5, "iso_year_start": 2025, "iso_week_start": 1 },
  "plugins_dir": "./plugins",
  "meta": {
    "csv_layout": "v0r7",          // レイアウト選択（拡張余地）
    "preferred_root": "outbound",  // 初期表示等に用いるヒント
    "root_dir": "..."              // GUI由来のパス（任意）
  },
  "scenario": { /* scenario:preloadが注入（後述） */ }
}
6.2 raw（I/Oアダプタの出力：DataFrame 実体のみ）
•	V0R7 CSVレイアウトの場合、最低限この8個を入れる（存在しない場合は空DFでも可だが警告）：
•	raw["tree_inbound"]           = DataFrame(product_tree_inbound.csv)
•	raw["tree_outbound"]          = DataFrame(product_tree_outbound.csv)
•	raw["sku_cost_inbound"]       = DataFrame(sku_cost_table_inbound.csv)
•	raw["sku_cost_outbound"]      = DataFrame(sku_cost_table_outbound.csv)
•	raw["node_geo"]               = DataFrame(node_geo.csv)
•	raw["S_month"]                = DataFrame(sku_S_month_data.csv)   # 任意
•	raw["P_month"]                = DataFrame(sku_P_month_data.csv)   # 任意
•	raw["tariff_table"]           = DataFrame(tariff_table.csv)       # 任意
禁止："tree_inbound_path": "..." のようなパス文字列だけを raw に入れること。必ず read_df 済み DataFrameにする。
6.3 root/state（計算と可視化の正本）
•	root は二本のツリーと状態を持つラッパ：
•	root = {
•	  "root_outbound": NodeTree(...),    # 需要層ツリー
•	  "root_inbound":  NodeTree(...),    # 供給層ツリー
•	  "state": {
•	    "psi_demand": { (node, sku): [ [S_ids], [CO_ids], [I_ids], [P_ids] ] for each week },
•	    "psi_supply": { ... 同上 ... },
•	    "scheduled": [...],  # イベント
•	    "hist": [ {"week_idx": int, "inventory": float}, ... ],  # 可視化用履歴
•	    # 任意: "ops_log": [...], "tickets": [...], ...
•	  },
•	  "raw": raw  # 何が入力だったかを後工程で参照可能に
•	}
•	数量ビュー：互換のため state["inventory"] = {node: float} を持ち続けても良い（派生情報）。
________________________________________
7. CSVアダプタ（V0R7対応の責務）
•	load_all(spec) -> raw
o	spec["scenario_id"] からディレクトリを特定し、8つのCSVを読み込んで raw に DataFrame として格納。
o	パスではなく DataFrame。キー名は上記固定（tree_inbound, tree_outbound, ...）。
o	読み込めなければ空DF＋警告（logger.warn）。
•	build_tree(raw) -> root
o	raw["tree_inbound"] / raw["tree_outbound"] の DataFrame を使ってツリー構築。
o	コスト・地理情報は set_attribute_from_csv(...) でノード・エッジへ付与。
o	root["state"]["psi_*"] の空配列（週×4バケツ）を初期化（horizon分）。
o	V0R7構造（二本ツリー＋PSI）で返す。
•	collect_result(root, params) -> result
o	KPI、root、demand_total_series（週次合計Sなど）を返す。
o	to_series_df(result) は state["hist"] を最優先し、無い場合にフォールバック。
________________________________________
8. Pipeline（背骨）とイベント順序
class Pipeline:
    def __init__(self, hooks: HookBus, io, logger=None): ...

    def run(self, db_path: str, scenario_id: str, calendar: dict, out_dir="out"):
        run_id = calendar.get("run_id")

        # 1) カレンダ（timebase）
        calendar = hooks.apply_filters("timebase:calendar:build", calendar, ...)

        # 2) データロード
        hooks.do_action("before_data_load", ...)
        spec = {"db_path": db_path, "scenario_id": scenario_id}
        spec = hooks.apply_filters("scenario:preload", spec, ...)   # ★ここで scenario を spec に注入
        raw  = io.load_all(spec)
        hooks.do_action("after_data_load", raw=raw, ...)

        # 3) ツリー構築（V0R7正本）
        hooks.do_action("before_tree_build", raw=raw, ...)
        root = io.build_tree(raw)
        root = hooks.apply_filters("plan:graph:build", root, raw=raw, ...)

        # 4) 需要チケット化（任意：after_tree_build）
        hooks.do_action("after_tree_build", root=root, raw=raw, calendar=calendar, ...)

        # 5) Allocator の取得（lotベース）
        alloc_fn = hooks.apply_filters("plan:allocate:capacity", None, root=root, calendar=calendar, ...)

        # 6) 週ループ
        for w in range(int(calendar["weeks"])):
            # (a) イベント→P(w)
            settle_events_to_P(root["state"], week=w)
            # (b) Iロール＋P取り込み
            roll_and_merge_I(root["state"]["psi_demand"], w, ...)
            roll_and_merge_I(root["state"]["psi_supply"], w, ...)
            # (c) 割付
            shipments = alloc_fn(root=root, week=w, calendar=calendar, ...)
            # 期待形: [{"src":node, "sku":sku, "week":w, "n_lots":int, "wanted":[...]}, ...]
            # (d) S消費（需要層のIから）
            consume_S_from_I_ids(root["state"]["psi_demand"], shipments, week=w)
            # 供給層側の処理が必要なら同様に
            # (e) 履歴記録
            #   histにinventoryを派生計算（len合計など）して追記

        # 7) 出力
        result = io.collect_result(root, params={})
        series = io.to_series_df(result, horizon=int(calendar["weeks"]))
        hooks.do_action("export:report", result=result, series=series, out_dir=out_dir, ...)
        return result
イベント順序（Hook標準）
timebase → load → tree → after_tree_build(需要化) → plan(alloc) → export
________________________________________
9. Hooks / Plugins（APIと推奨順序）
9.1 主なHook名
•	scenario:preload (filter)：specに scenario JSONを注入（plugins ON/OFF、パラメータ、目的関数など）。
•	timebase:calendar:build (filter)：休暇週、収穫期などのカレンダ加工。
•	after_tree_build (action)：demand.csv → 需要チケット化（S正本のセット）等。
•	plan:allocate:capacity (filter)：allocator関数の差替。戻り値はlot個数ベース（n_lots）。
•	plan:allocation:mutate (filter)：capacity clip、有効期限クリップ、優先度ソートなどの加工。
•	cost:calc:mutate (filter)：保管費・通関費・設備時間などのコスト追加。
•	viz:series (filter)：系列のビュー列追加。
•	export:report (action)：CSV/PDF/画像などの出力。
9.2 scenario:preload サンプル
# pysi/plugins/scenario_loader/plugin.py
import json, os
from pysi.core.hooks.core import filter

@filter("scenario:preload", priority=30)
def preload_scenario(spec, **ctx):
    base = spec.get("db_path") or "."
    scn  = spec.get("scenario_id") or spec.get("scenario") or ""
    # JSONファイル例: examples/scenarios/<scn>/scenario.json
    path = os.path.join(base, scn, "scenario.json")
    try:
        with open(path, "r", encoding="utf-8") as f:
            spec["scenario"] = json.load(f)
    except Exception:
        spec["scenario"] = {}
    # 互換: meta/csv_layout の既定
    meta = spec.setdefault("meta", {})
    meta.setdefault("csv_layout", "v0r7")
    return spec
9.3 plan:allocate:capacity（出力は lot数）
# 期待: alloc_fn(root, week, calendar, ...) -> list[shipments]
# shipments = [{"src":..., "sku":..., "week":w, "n_lots":int, "wanted":[...]}]
既存のqtyベースのプラグインは、LOT_SIZE[sku] 等で n_lots = ceil(qty / LOT_SIZE[sku]) に変換。
________________________________________
10. KPI / CO表現 / エクスポータ
10.1 KPI（例）
•	fill_rate: 出荷実績ロット数 / 要求ロット数
•	end_inventory: 期末 sum(len(I(w_last))) の合計（対象ノード/skuの合計）
•	expiry_scrap_cost: FEFOで期限切れ分を任意にコスト化（cost:calc:mutate で集計）
10.2 履歴と系列
•	state["hist"] を最優先に to_series_df で weekly series を生成。
•	無ければフォールバック（raw由来に計算）に落とす。
10.3 COの見せ方（選択）
•	イベント駆動（発注週にだけ積む）
o	現在の仕様はこの方式。到着週にCOを消し込み＋Pへ計上。
•	連続表示（発注～到着の間、各週のCOにコピー）
o	表示用ポリシーとして viz:series で加工するか、専用関数でコピーする。
________________________________________
11. GUI/CLI 整合（entry_csv / entry_gui / run_once）
•	run_once(cfg, bus=None, io=None, logger=None) を推奨：
o	HookBus を set_global(bus) で差し替え後にプラグインをautoload。
o	GUIでは、scenario_id, calendar, plugins_dir, root（CSVルート）を cfg に渡す。
o	Pipeline.run(db_path, scenario_id, calendar, out_dir) に分解して渡す（cfgから抽出）。
•	CSVAdapter.load_all は spec["scenario_id"] から raw に DataFrameを格納。
o	必ず raw["tree_inbound"], raw["tree_outbound"] など8キーを作成。
________________________________________
12. 高速化プラン（まとめ）
•	lot_pool（属性辞書）を参照にし、PSI内部はIDのみ。
•	Iロールは最初は deepcopy で安全運用、将来的にcopy-on-write導入。
•	FEFO は週頭に sort もしくは挿入時 bisect。
•	バッチ：(node, sku, week) 単位で一括処理。
•	並列：製品/地域クラスタ並行。
________________________________________
13. 実運用パッチ（重要 5点）
1.	合成lot_IDは決定論的に
o	hash() はプロセスごとにランダム化 → deterministic_syn_ids() を採用（例示済み）。
2.	Iの消費は週頭ロール後の I(w) から
o	I(w) は I(w-1) を含むため通常遡及は不要。
o	（遡及消費が要るように見えたのは、ロール漏れ時の症状。ロール順守で解決）
3.	COの表現方針を固定
o	本仕様では「イベント駆動（発注週のみCO）」を正とし、表示連続は viz:series で対応。
4.	horizon境界のガード
o	to_week/arrive_week が地平外なら、クリップ or ロールオーバーをルール化（まずはクリップ）。
5.	二重決済の回避
o	settle_events_to_P() はpipeline週頭の1回のみで呼ぶ。commit系フックでは呼ばない。
________________________________________
14. シナリオ（plugins ON/OFF＋設定）
14.1 シナリオJSON例（医薬：冷凍＋有効期限＋一過性需要）
{
  "id": "PHARMA_FLU_5W",
  "timebase": { "weeks": 5, "iso_year_start": 2025, "iso_week_start": 1 },
  "plugins": {
    "enable": [
      "pysi.plugins.cold_chain",
      "pysi.plugins.expiry_lot",
      "pysi.plugins.urgency_tickets",
      "pysi.plugins.capacity_clip"
    ],
    "config": {
      "pysi.plugins.cold_chain": { "default_temp": "frozen" },
      "pysi.plugins.expiry_lot": { "fefo": true, "discard_cost_per_unit": 2.0 },
      "pysi.plugins.capacity_clip": { "clip_mode": "hard" }
    }
  },
  "objective": { "weights": { "fill_rate": 1.0, "expiry_scrap_cost": 0.5, "inventory": 0.2 } },
  "allocation": { "policy": "tickets", "ticket_fields": ["urgency","expiry_bias"] },
  "experiments": [
    { "name": "mild_spike", "demand_multiplier": 1.2, "weeks": [2,3] },
    { "name": "hard_spike", "demand_multiplier": 1.8, "weeks": [2] }
  ]
}
•	scenario:preload で spec["scenario"] に注入。
•	各プラグインは ctx["scenario"]["plugins"]["config"][<plugin>] から読む。
14.2 業種別プラグイン例
•	医薬: cold_chain（温度帯）、expiry_lot（FEFO/期限）、flu_spike（需要スパイク）
•	米: harvest_window（収穫週の在庫目標）、storage_cost_curve（在庫週齢コスト）、import_substitute
•	半導体: bottleneck_scanner（拘束工程検出）、yield_model（歩留）、expedite_policy（LT短縮と費用）
________________________________________
15. エラーメッセージ／バリデーション
•	CSVAdapter.load_all：
o	8ファイル中、キーが欠けるとWARN。"tree_inbound"/"tree_outbound"が空ならERROR。
•	build_tree：
o	raw["tree_inbound"]/raw["tree_outbound"] がDataFrame以外なら ERROR（パス文字列等を拒否）。
•	Pipeline：
o	scenario_id/calendar が欠落時は ERROR。
o	Allocator の戻り（n_lots）が負/非整数は ERROR。
________________________________________
16. 移行ノート（V0R7 → V0R8）
•	構造は完全互換（dual-tree + PSI buckets = lot_ID lists）。
•	高速化は実装最適化で行い、データ契約は不変。
•	qtyベースのプラグインはlot個数に変換（ceil(qty / LOT_SIZE[sku])）。
•	CSVは8ファイルを DataFrame で raw に格納（ファイルパスは不可）。
________________________________________
17. 既知の落とし穴（再掲）
•	ローカル plugins/diagnostics/plugin.py と中核 diagnostics の二重登録に注意（プロジェクト側はローカルに集約）。
•	to_series_df() はhist最優先。無ければフォールバック計算に落ちる。
•	settle_events_to_P() は週頭の1回だけ。ダブり呼び出し注意。
________________________________________
18. 次アクション（推奨）
1.	CSVAdapter を 8 DataFrame版に確定（rawにパス文字列を入れない）。
2.	scenario:preload と サンプルシナリオJSON（医薬・米・半導体）を3本用意。
3.	FEFO/Capacity のmutateプラグイン（plan:allocation:mutate）を最小到達版で束ねる。
4.	state["hist"] を週ループで確実に追記 → GUIプレビューが安定。
5.	SYN ID決定論化・horizonガードをPipelineに反映。
________________________________________
以上が**統合版 正本仕様（V0R8）です。
ご要望に合わせ、コードの差分パッチ（csv_adapter.py の load_all() を DataFrame化、build_tree() の入力契約、entry_gui.py/run_once.py の引数整合、Pipeline.run() の週次オーダなど）も準備できます。必要でしたら、該当ファイルごとに適用可能な diff**をお出しします。
________________________________________
📎 付録（同梱ファイル）
•	📄 SEED_PROMPT_251023_V0R8_CORE_SPEC.docx
ダウンロード
Word 版はこの仕様の全文を章立てで整理済みです。今後の差分もこのファイルを「正本」として継続更新していけます。
必要に応じて、GUIプレビュー用の hist 追記テンプレや、FEFOプラグイン最小実装も続けてお渡しします。
あなた:
確認です。「GUIプレビュー用の hist 追記テンプレ」の"hist"は、どのような意味、内容ですか? また、histとはhistoryの事ですか?
ChatGPT:
思考時間: 6s
はい、そのとおりです。ここで言う hist は “history（履歴）” の略で、
週ごとの可視化（GUIプレビューやレポート）のために蓄積する軽量な時系列スナップショットを指します。
________________________________________
hist の目的と位置づけ
•	目的：GUIの折れ線グラフや表ですぐに描ける軽量ビューを、週ループ中に逐次貯めるための履歴バッファ
•	位置：root["state"]["hist"]（＝Coreの正本 spec/raw/root/state のうち、state に属する「ビュー層」データ）
•	中身：
o	最小構成では週インデックスと集計在庫など
o	拡張で需要総数、出荷数、受入数、欠品（合成SYN）数、容量使用率、平均優先度などを付加
重要：
hist は**計算正本（PSIのlot_IDリスト）**ではありません。
表示用・分析用の派生ビューとして、GUI/レポートのレスポンスを良くするために持ちます。
（なので、大量のlot_IDを hist に入れない＝軽量に保つのがポイント）
________________________________________
最小スキーマ（推奨）
# 1レコード = 1週のスナップショット
{
  "week_idx": int,           # 0-based
  "inventory": float,        # 集計在庫（例：全ノード×全SKUの I(w) の総数）
  "demand_total": float,     # その週の需要総数（S要求ロット数）
  # 任意拡張：
  "shipped": float,          # 実出荷（I→Sに動いたロット数）
  "received": float,         # 受入/完成（Pに載せたロット数）
  "backlog": float,          # 未充足（SYNで補ったロット数）
  "avg_urgency": float|None, # プラグインが付与する平均緊急度など
  "capacity_used": float|None
}
数量はlot個数なので float ではなく int でも構いません。
ただし将来、加重平均や係数を入れる可能性を見て float で扱っても実害はありません。
________________________________________
どのタイミングで記録する？
週ループ内の最後（または最後に近いタイミング）で追記するのが一番明快です。
•	週頭：イベント決済 → P(w)
•	I(w) ← I(w-1) + P(w)
•	割付 → S(w) に動かす
•	ここで hist.append({...}) を記録
•	（書き方は下のテンプレ参照）
________________________________________
集計の考え方（PSI = lot_ID リストから数量へ）
•	inventory は、対象範囲の psi[*][week][I] のリスト長の合計
•	demand_total は、その週のS要求数（方法は2通り）
o	① 要求正本（tickets/要求S） を別管理しているなら、その個数を集計
o	② この段階では S(w) は「出荷実績」なので、要求総数は別に持っておく（raw["demand"]やstate["s_demand"]など）
•	shipped は、その週に I→S へ実際に動いたロット数（割付結果の合計）
•	received は、その週に P(w) に積んだロット数
•	backlog は不足分を合成ID（SYN）で補ったロット数の合計
________________________________________
✅ 最小テンプレ（そのまま使える）
def record_hist_snapshot(root, week: int):
    """
    root["state"]["psi_demand"] / psi_supply は次の形を想定：
      { (node, sku): [ [S_ids], [CO_ids], [I_ids], [P_ids] ]  (週ごとに配列) }
    ※ ここでは「需要層の I(w)」を在庫とみなす例。供給層に切替えてもOK。
    """
    st = root["state"]
    psiD = st.get("psi_demand", {})

    # 1) inventory（全node×skuの I(w) の総ロット数）
    inv_total = 0
    for psi in psiD.values():
        Iw = psi[week][2]  # PSI_I
        inv_total += len(Iw)

    # 2) demand_total（要求ロット総数）:
    #    もし state["s_demand"] に leaf起点の要求チケットがあればそれを使う。
    #    なければフォールバックで「当週にSへ積まれた数」を「実需要代理」として記録しても良い。
    demand_total = 0
    s_demand = st.get("s_demand")  # {(node, sku): {week: [ticket_id...]}}
    if s_demand:
        for (_, _sku), by_week in s_demand.items():
            demand_total += len(by_week.get(week, []))
    else:
        # フォールバック：実出荷を需要代理とする（厳密でないがビュー用途としてOK）
        for psi in psiD.values():
            Sw = psi[week][0]  # PSI_S
            demand_total += len(Sw)

    # 3) shipped, received, backlog（任意）
    shipped = 0
    received = 0
    backlog = 0

    # shipped = 当週に S(w) に追加された実ロット（Iから移動した分）
    #  → 週内でmoveログを取っていればそれが一番正確。なければ Sw のうち SYN以外をカウントする運用も可。
    for psi in psiD.values():
        Sw = psi[week][0]
        shipped += sum(1 for lid in Sw if not str(lid).startswith("SYN:"))
        backlog += sum(1 for lid in Sw if str(lid).startswith("SYN:"))

        Pw = psi[week][3]
        received += len(Pw)

    # 4) 任意：avg_urgency（プラグインが state に算出済みなら拾う）
    avg_urgency = None
    if "avg_urgency_by_week" in st:
        avg_urgency = st["avg_urgency_by_week"].get(week)

    # 5) 追記
    st.setdefault("hist", []).append({
        "week_idx": week,
        "inventory": float(inv_total),
        "demand_total": float(demand_total),
        "shipped": float(shipped),
        "received": float(received),
        "backlog": float(backlog),
        "avg_urgency": avg_urgency,
    })
使い方：Pipelineの週ループ末尾で record_hist_snapshot(root, w) を呼ぶだけです。
こうしておくと、CSVAdapter.to_series_df(result) が state["hist"] を最優先で DataFrame 化できます。
________________________________________
GUI（matplotlib/Tkinter）とのつなぎ
•	to_series_df(result) は state["hist"] があればそれを優先して DataFrame を返す実装（既に反映済みの方針）
•	GUI側はこの df[["week_idx", "inventory", "demand_total", ...]] から折れ線を描画
•	例：inventory（青）、demand_total（緑）、backlog（赤の棒）など
________________________________________
よくある拡張バリエーション
用途に応じて hist の粒度を調整できます：
1.	グローバル集計（上のテンプレ）
o	全ノード×SKUの合計 → GUIでの全体傾向把握用（軽い）
2.	ノード別（またはSKU別）
o	hist_node / hist_sku を別配列にし、キー付き（node, sku）で追記
o	可視化時にプルダウンで切り替え
3.	需要層／供給層の分離
o	inventory_demand と inventory_supply を別列で出す
o	バランスや滞留位置の見える化に有効
注意：hist はあくまでビュー用なので、大量の明細やlot_ID本体は入れないでください。
hist のレコードは「1週 1行」×「少数列」という設計が、GUIをカクつかせないコツです。
________________________________________
まとめ
•	hist = history（履歴）の略で、GUI/レポートの表示用に週次で積み上げる軽量スナップショットです。
•	記録は週ループの最後でOK。lot_IDリスト → len(list) で数量化して集計します。
•	to_series_df が hist を読めるので、GUIはそのままプロット可能（TATが速い）。
•	ビュー用なので軽量を維持（lot明細は入れない）。必要なら別に ops_log や stock_snapshot を用意。
必要でしたら、record_hist_snapshot を **Hook（例：viz:hist:record）**として外出しする形や、
backlog（合成SYN）のカウントを確実に取るための ops_log 記録テンプレも続けてご用意できます。

