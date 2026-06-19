# IPM 磁束マップ multi-speed 同定 戦略書（A＋C＋E）

## この文書の位置づけ

- **戦略の入り口**。後続の手順書（`../plan_opus/*_procedure.md`）・スクリプト・取得計画へ落とすための上位設計。実務工程は本書の Phase 表とゲートから派生させる。
- 対象: IPM 順磁束マップ `psi_d(id,iq)`, `psi_q(id,iq)` の**同定方法**。現行の単一速度同定（`flux_identify.py`）を置き換える。
- 姉妹文書:
  - [`dq2trq_vi_flux_map_plan.md`](dq2trq_vi_flux_map_plan.md) — v1 単一速度・ペア平均の磁束マップ企画
  - [`../plan_opus/plan_dq2trq_flux_loss.md`](../plan_opus/plan_dq2trq_flux_loss.md) — C案（磁束＋損失分離＋ΔT）
  - [`../plan_opus/flux_map_creation_procedure.md`](../plan_opus/flux_map_creation_procedure.md) — 現行（単一速度）手順書

## 背景・真因（なぜ作り直すか）

- **現状**: `private/dq2trq_vi_data.csv` は機械 1000rpm の**単一速度**（`omega_e = 418.879 rad/s`）で 579 点。
- **限界**: 単一 `omega_e` では定常電圧式が ψ と「Rs 降下＋インバータ電圧誤差」を**分離できない**（identifiability 不足）。C案の同定で `Vdt`/`Rs` が「4param 半縮退」したのはこの構造が原因。
- 力行/回生ペア（`iq` 符号反転）は `Rs·i` と インバータ誤差 `e` を**両方とも**反転させるので差分で分離できず、和を取る方法は `psi_d(id,+iq) = psi_d(id,-iq)` という**クロス飽和の対称性仮定**を要求する（C案が明示的に避けたかったもの）。
- **真因**: 観測情報が電流軸に偏り、**速度軸（`omega_e`）に分散していない**。データを 600 点重ねても次元が増えない。

## 核心原理（A: speed-sweep regression）

定常電圧式（電力不変 dq、`Te = p(psi_d·iq − psi_q·id)` 規約）にインバータ電圧誤差 `e_d, e_q` を入れる:

```text
vd = Rs·id − omega_e·psi_q(id,iq) + e_d(id,iq)
vq = Rs·iq + omega_e·psi_d(id,iq) + e_q(id,iq)
```

各項の依存軸:

| 項 | omega_e 依存 | 電流依存 |
|---|---|---|
| `omega_e·psi`（磁束 EMF） | **∝ omega_e** | あり |
| `Rs·i`（抵抗降下） | なし | あり |
| `e_d, e_q`（デッドタイム＋デバイス降下） | なし | あり（主に符号） |

固定 `(id,iq)` で `omega_e` を振ると:

```text
vq = (Rs·iq + e_q) + psi_d·omega_e      ← 傾き = psi_d, 切片 = Rs·iq + e_q
vd = (Rs·id + e_d) − psi_q·omega_e      ← 傾き = −psi_q, 切片 = Rs·id + e_d
```

→ **磁束は「傾き」、Rs 降下とインバータ誤差は全部「切片」に落ちる**。各動作点で `v` vs `omega_e` を回帰すれば、**対称性仮定なし・ペアリングなし**で `psi_d, psi_q` を直接得る。**副産物として切片 = `Rs·i + e(i)` も同時に得る**（柱 C の入力）。

`omega_e` 軸は電流符号・電流振幅のどちらとも直交するので、ψ を一意化する唯一の独立レバーになる。これが単一速度に対する本質的アップグレード。

## 戦略の3本柱

```text
        ┌──────────── A: speed-sweep regression ────────────┐
入力VI →│ 各(id,iq)で v–omega_e 回帰                          │→ psi サンプル（傾き）
        │                                                    │→ 切片 = Rs·i + e(i)
        └────────────────────────────────────────────────────┘
                                  │ 切片
                                  ▼
        ┌──────────── C: インバータ誤差の分離 ───────────────┐
独立Rs →│ e(i) = 切片 − Rs·i  →  e(i) を恒久 LUT 化           │→ 較正済み電圧モデル
        └────────────────────────────────────────────────────┘
                                  │ psi サンプル
                                  ▼
        ┌──────────── E: 構造化飽和フィット ─────────────────┐
        │ 疎な psi サンプルに飽和バックボーン＋クロス飽和を回帰 │→ 順磁束マップ（全面・物理外挿）
        └────────────────────────────────────────────────────┘
```

### A. Speed-sweep regression（同定本体）

- 各 `(id,iq)` で 3–5 速度 → `v` vs `omega_e` を回帰。傾きから `psi_d, psi_q`。
- **鉄損の扱い**: 鉄損は `v` に `omega_e`（ヒステリシス）・`omega_e²`（渦電流）依存成分を加える。`v = a + b·omega_e + c·omega_e²` で当て、`b` が主に ψ、`c` が渦電流を吸収する。厳密分離が要るなら鉄損抵抗 `Rc` を陽にモデル化（柱 B 相当＝Armando 2013 の鉄損補正に倣う）。
- 最小構成は 2 速度（縮退解消の必要十分）、推奨は 4–5 速度（鉄損 2 次項＋回帰の頑健性）。

### C. インバータ非線形性の分離

- A の切片 `= Rs·i + e(i)`。**独立に Rs を測れば**（DC 4 端子、既知温度）`e(i) = 切片 − Rs·i` に分解できる。
- 交差検証として、standstill での d 軸ランプ電流注入で `e(i)` を独立取得（指令電圧 vs 必要電圧）。
- 確定した `e(i)` を**恒久 LUT** 化し、以後の全同定・将来のオブザーバから電圧誤差を除去。これで「電圧誤差」側を根絶する。

### E. 構造化飽和モデル当てはめ

- A で得た**疎な** ψ サンプルに、物理構造（飽和バックボーン＋クロス飽和項）を回帰してマップ化。
- **既存 `flux_map_build.py` の飽和バックボーン＋RBF パッチをほぼ再利用**。入力を「単一速度・ペア平均の ψ サンプル」から「multi-speed 回帰由来の ψ サンプル」に差し替えるのが主変更。
- 構造が正則化として効くので、~20–40 点の疎データでも identifiable かつ物理的に外挿できる（ユーザーの「100 点案」の理論的裏付け）。

## 取得計画の骨子

| 設計変数 | 案 | 根拠 |
|---|---|---|
| 電流動作点 | 代表 ~25 点（MTPA 線＋数本の FW 線＋off-MTPA 数点） | E の構造化フィットが疎点を許容。運転域 `id<0` を重点 |
| 速度段／点 | 最小 2 / 推奨 4–5 | 2 で縮退解消、4–5 で鉄損 2 次項＋頑健性 |
| 速度上限 | 各点の変調限界まで | `vq≈omega_e·psi_d` が DC リンク上限に当たる手前 |
| 速度下限 | SNR が許す最低速 | 低速は切片（Rs+e）同定に好適、傾きは高速で精度 |
| 取得列 | `id_sw, iq_sw, vd, vq, omega_e(or rpm), torque, 温度, I_probe_rms` | `I_probe_rms`＝電流校正の真振幅。温度は熱ドリフト監視（ψm, Rs） |
| 総点数の目安 | ~100（25 点 × 4 速度） | 単一速度 600 点より情報量が桁違い |

- **電圧上限の事前見積り**: 既存の単一速度データ（`vd, vq @ 418.879`）から `v ∝ omega_e` で各点の速度別電圧を外挿し、変調限界内に収まる安全速度を**取得前に**決める（既存資産の活用）。`746V` 系の電圧バジェットに対し 1000rpm は基底速度よりかなり下＝上下に振れる余地が大きい。
- **熱の固定**: 1 動作点のスイープは短時間で、または恒温下で行い、スイープ中に `psi_m, Rs` がドリフトして傾きを汚さないようにする。プロジェクトの ΔT 適応目的（2Hz、アモルファスコア ΔT=50K→Δトルク 8%）と整合。

## Phase 0 詳細: 電流計測校正（振幅専任＋ECU角度＋τ）

**目的**: ソフト電流 `(id_sw, iq_sw)` を校正済み設備プローブ（真値）へ合わせ込み、同定が使う `(id, iq)` 軸のゲイン誤差・遅延誤差を除く。プローブには相電流の**振幅スカラ**だけ担わせ、角度は ECU、遅延は別途同定するスカラ `tau` で補正する。**プローブ側にロータ角同期は不要**。

### 誤差の層別

| 誤差源 | 振幅 | 位相 `β` | 校正手段 |
|---|---|---|---|
| 共通ゲイン `K`（ECU 入力ゲイン・センサ倍率） | 汚す | **不変** | プローブ振幅で伸縮 |
| 相間アンバランス・オフセット | 小（1ω/2ω リプル） | 小 | 定常マップでは小。要時のみ相別回帰 |
| チェーン遅延 `tau`（取得＋演算） | ほぼ無し | 汚す（`omega_e·tau`、速度依存） | スカラ `tau` で角度補正 |

**鍵**: 共通ゲインは `β` を動かさない（`i_abc → K·i_abc` を Clarke/Park しても向き不変、大きさのみ ×K）。ゆえに角度は ECU 由来のまま使え、プローブは振幅専任でよい。`K` はリプルを出さないのでオンライン自己補正法（相間ゲイン比・ESO 等）では**原理的に検出不能** → 外部プローブが唯一の手段。

### 手順（各定常動作点 = dq は DC、相電流は純正弦波）

```text
# 1. 真の振幅（プローブ）
|I|_true = sqrt(3) * I_probe_rms        # 電力不変 dq 規約: sqrt(id^2+iq^2)=sqrt(3)*Irms
|I|_sw   = sqrt(id_sw^2 + iq_sw^2)

# 2. 角度（ECU）
beta_sw = atan2(-id_sw, iq_sw)

# 3. 遅延補正（スカラ tau、別途同定）
beta_true = beta_sw + omega_e * tau

# 4. 真電流の再構成
id_true = -|I|_true * sin(beta_true)
iq_true =  |I|_true * cos(beta_true)

# tau 補正不要なら純伸縮（向き保持）:
# f = |I|_true / |I|_sw ;  id_true = f*id_sw ;  iq_true = f*iq_sw
```

### `tau`（チェーン遅延）の同定

- `tau` は**ロータ角ではなく取得チェーンの時間遅れ**（センサ応答 ~5µs ＋ AAF ＋ ADC ＋ サンプリング ＋ 演算）。ベンチで「センサのアナログ入力 vs ECU がログした値」を**トリガ時刻同期**で時間比較して取る（角度同期ではない＝実現容易）。
- 5µs 単体は基本波（1000rpm = 66.7Hz）で 0.12°＝無視可。主因は総遅延（例: 10kHz PWM で ~150µs → 1000rpm で 3.6° のベクトル回転）。**5µs スペックを当てにせず実測**する。
- `tau` はプローブと独立な定数。一度確定すれば全速度・全点に `omega_e·tau` で適用。

### 成果物・受け入れ条件

- 成果物: `current_calib.json`（`K`＝共通ゲイン, `tau`, 任意で相別 `K_x, O_x`）。
- 受け入れ: 校正後に複数振幅で `|I|_sw ≈ |I|_true`（残差 < 目標%）／`tau` 補正後の `β` が速度に対しフラット（残留 `omega_e·tau` 勾配が消える）。→ ゲート G1b。

### multi-speed 同定への効き方（なぜ Phase 0 か）

- 未補正の `tau` は `β` を `omega_e·tau` だけ**速度依存**で回す → 固定動作点のはずが速度ごとに実 `(id,iq)` がずれ、`v–omega_e` 回帰の傾き（= ψ）に直接エイリアスする。よって `tau` 補正は **Phase 2 の前提**。
- 共通ゲイン `K` は ψ・トルクの**振幅**を一様バイアスし、マップ精度の床を決める。Phase 1 取得時に各点 `I_probe_rms` を併記し、Phase 2 入口で真 `(id,iq)` に変換してから回帰する。

## 検証ゲート

- **G0 単位・符号・速度多様性**: `p=4`、`omega_e` の符号、電力不変 dq。**各電流点に複数 `omega_e` が存在**することを確認（単一速度データでは G2 以降が成立しない）。
- **G1 取得 feasibility**: 取得計画の各 (点, 速度) が変調限界内（既存データからの `v∝omega_e` 外挿で事前判定）。
- **G1b 電流校正**: 校正後に複数振幅で `|I|_sw ≈ |I|_true`（残差小）、`tau` 補正後の `β` が速度に対しフラット（残留 `omega_e·tau` 勾配が消える）。詳細は「Phase 0 詳細: 電流計測校正」節。
- **G2 回帰品質**: 点ごとの `v`–`omega_e` 当てはめ `R²`・残差、傾き（ψ）の安定性、鉄損 `omega_e²` 項の大きさ。
- **G3 分離の健全性**: 切片を `Rs·i + e(i)` に分解。`e(i)` の符号・振幅がデッドタイム物理と整合するか、`Rs` が DC 実測値と一致するか。**C案の `Vdt`/`Rs` 半縮退が解消したことを定量で示す**。
- **G4 ψ サンプル物理性**: `psi_d, psi_q` の単調性・クロス飽和が自然、`Te = p(psi_d·iq − psi_q·id)` で実測トルクを再現。`psi_d<0`（強め界磁弱め域）は許容。
- **G5 マップフィット品質**: 構造化フィットの RMS、外挿健全性、v1 単一速度マップとの差（v1 は真値ではなく sanity 比較）。
- **G6 熱整合**: 各スイープ中の `psi_m, Rs` が安定（温度ログで確認）。傾きへの熱バイアスがないこと。

## 作業工程（Phase 表）

| Phase | 内容 | 主成果物 | 依存 | 状態 |
|---|---|---|---|---|
| 0 | 取得計画確定（電流点・速度段・変調限界 feasibility・符号/単位 lock・ゲート定義）＋**電流計測校正**（`K`・`tau` 同定、振幅専任＋ECU角度＋τ。詳細節参照） | acquisition plan note, `current_calib.json` | 既存 VI データ、設備プローブ、トリガ同期 | 未 |
| 1 | **ベンチ取得**（ダイナモ定速駆動、各電流点で速度スイープ、定常 VI＋温度ログ） | `multispeed_vi_data.csv` | Phase 0、ベンチ・ダイナモ | 未（要ベンチ） |
| 2 | A 同定: `v`–`omega_e` 回帰 → ψ サンプル＋切片 | `flux_samples_ms.csv`, `intercepts.csv` | Phase 1 | 未 |
| 3 | C 分離: 独立 Rs で切片を `Rs·i + e(i)` に分解、`e(i)` LUT 化＋standstill 交差検証 | `inverter_error_lut.csv`, `rs_note` | Phase 2、DC Rs 実測 | 未 |
| 4 | E 当てはめ: 構造化飽和フィット → 順磁束マップ（`flux_map_build.py` 再利用） | `forward_flux_map_psi_{d,q}.csv` | Phase 2 | 未 |
| 5 | 下流再生成: 逆マップ・EM マップ・validity・QA、v1 比較 | `inverse_flux_map_*`, `em_torque_map`, `qa_*.md` | Phase 4、既存 build スクリプト | 未 |
| 6 | 採用判断（ゲート総合、plant variant / rollout 判断） | decision note | Phase 5 | 未 |

- **Phase 1 が律速依存**: 本戦略は**新規 multi-speed データ取得**を前提とする（既存データは単一速度）。ベンチが直ちに使えない場合の de-risk として、**Phase 2 のドライラン**（2 速度の合成データ or 既存 1 速度＋仮想 1 速度）で回帰・分離パイプラインを先に検証しておく。
- 各 Phase は独立に手順書（`plan_opus/*_procedure.md`）へ展開できる粒度。

## 既存資産との関係

| 既存 | 本戦略での扱い |
|---|---|
| `flux_identify.py`（単一速度・ペア平均・Vdt/Rs 半縮退） | **置換**。multi-speed 回帰スクリプトへ |
| `flux_map_build.py`（飽和バックボーン＋RBF パッチ） | **再利用**（柱 E）。入力 ψ サンプルを差し替え |
| `inverse_flux_map_build.py`, EM/validity 生成 | **そのまま**（下流不変） |
| C案 ΔT・軸トルク（`delta_shaft_*`） | **不変**。クリーンな ψ が入れば EM 精度が上がり ΔT 残差も改善見込み |

## プロジェクト最上位目的との接続

multi-speed 同定は `psi(id,iq)` と `Rs` と `e(i)` を**同時に clean 分離**する。これは「磁束オブザーバ式トルク推定を補助する Rs 推定器」と「ΔT 適応」というプロジェクト最上位目的に直接寄与する:

- 切片から得る `Rs` は、まさにプロジェクトが欲しい Rs の独立基準（較正・検証用の真値）。
- `e(i)` LUT は将来のオブザーバの電圧前処理にそのまま使える恒久資産。
- つまり本戦略は「より良い磁束マップ」に留まらず、**ψ / Rs / インバータ誤差の三者分離フレーム**そのものの確立。

## リスク・未決事項

- **ベンチ依存**: Phase 1 の新規取得が前提。代替として最小 2 速度でも縮退は解消する（精度は劣る）。
- **鉄損の傾きバイアス**: `omega_e²` 項の分離が甘いと ψ にバイアス。速度範囲を絞る／`Rc` モデル化で対処（G2）。
- **熱ドリフト**: スイープ時間 vs 熱時定数。恒温化が要るか取得前に判断（G6）。
- **独立 Rs の温度整合**: DC 実測 Rs の測定温度を取得時温度に合わせる（C の分解精度に直結）。
- **電圧の素性**: `vd, vq` が実測相電圧の dq 変換か指令電圧か（v1 企画の未決事項を継承）。
- **取得点配置**: MTPA／FW 線の本数と off-MTPA 点の最適配置は Phase 0 で確定。
- **電流校正の前提**: 共通ゲインは厳密補正だが、相間アンバランス・オフセット由来の微小バイアスは残る（定常マップでは小）。`tau` 同定にはプローブと ECU ログの**トリガ時刻同期**が要る（ロータ角同期は不要）。単相 RMS 代用は三相平衡前提。

## 参考文献

- E. Armando, R. Bojoi, P. Guglielmi, G. Pellegrino, M. Pastorelli, "Experimental Identification of the Magnetic Model of Synchronous Machines," IEEE Trans. Ind. Appl., 49(5):2116–2125, 2013. — 定速・電流制御グリッド＋Rs/インバータ/鉄損/温度の明示補正（手順の原典、柱 B）。
- "Identification of Flux Linkage Map of PMSM Under Uncertain Circuit Resistance and Inverter Nonlinearity," ~2017. — R とインバータ非線形性を相殺（柱 A/C の親戚）。
- "Obtaining the Current-Flux Relations of the Saturated PMSM by Signal Injection," arXiv:1705.04605. — 静止時 信号注入で微分インダクタンス→磁束（柱 D、相補的）。
- Hinkkanen 系, "Stand-still self identification of flux characteristics using novel saturation approximating function and multiple linear regression." — 構造化飽和関数＋線形回帰（柱 E）。
- "UKF-based Offline Estimation of PMSM Magnet Flux Linkage Considering Inverter Dead-time Voltage Error," 2022. — 低速走行試験で ψm・Rs・インバータ歪みを同時推定（柱 F）。
