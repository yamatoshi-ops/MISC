磁束マップ同定の核心的な論点ですね。「同一動作点（同一 ωe）では ψ と電圧誤差が分離できない」というのは本質を突いた指摘です。英語圏の文献を調査してアイデアを集めます。まず検索ツールを読み込みます。

検索を並行実行します。

核心に近い文献が出てきました。特に「不確かな抵抗・インバータ非線形性の下での磁束マップ同定」はユーザーのアイデアそのものです。さらに深掘りします。

核心テクニックの詳細を、オープンアクセス文献から固めます。

調査結果がそろいました。まず結論から言うと、**あなたの直感は文献的にも理論的にも正しい**です。「同一 ωe では ψ と電圧誤差が分離不能、ωを振れば分離できる」というのは、磁束同定で identifiability を生む唯一の直交軸を突いています。順に整理します。

---

## 1. なぜ単一 ωe で ψ が決まらないか（構造的限界の数式）

定常状態の電圧方程式（あなたの電力不変 dq 規約）にインバータ電圧誤差 $e_d, e_q$ を入れると:

$$v_d = R_s i_d - \omega_e \psi_q(i_d,i_q) + e_d(i_d,i_q)$$
$$v_q = R_s i_q + \omega_e \psi_d(i_d,i_q) + e_q(i_d,i_q)$$

ここで各項の **依存軸** が決定的に違います:

| 項 | ωe 依存 | 電流依存 |
|---|---|---|
| $\omega_e\psi$（磁束 EMF） | **∝ ωe** | あり |
| $R_s i$（抵抗降下） | なし | あり（符号・振幅） |
| $e_d,e_q$（デッドタイム＋デバイス降下） | なし | あり（主に符号） |

固定 $(i_d,i_q)$ で **ωe を振って** $v$ を測れば:

$$v_q = \underbrace{(R_s i_q + e_q)}_{\text{切片}} + \underbrace{\psi_d}_{\text{傾き}}\cdot\omega_e, \qquad v_d = \underbrace{(R_s i_d + e_d)}_{\text{切片}} - \underbrace{\psi_q}_{\text{傾き}}\cdot\omega_e$$

→ **磁束は「傾き」、抵抗降下とインバータ誤差は全部「切片」に落ちる**。単一 ωe だと未知数2に対し式1本＝不定。これがまさに C 案で `Vdt` と `Rs` が「4param 半縮退」した正体です。

**重要**: あなたが C 案で使った力行/回生ペア（iq の符号反転）は、$R_s i$ と $e_q$ が**両方とも符号反転する**ため差分では分離できず（だから縮退）、和を取る方法は $\psi_d(i_d,+i_q)=\psi_d(i_d,-i_q)$ という**クロス飽和の対称性仮定**を要求します（あなたが避けたかったもの）。一方 **ωe 軸は電流符号・電流振幅のどちらとも直交**するので、対称性仮定もペアリングも不要で、各動作点ごとに傾き＝ψ を直接取れます。これが本質的なアップグレードです。

---

## 2. あなたのアイデア＝文献の「multi-speed / speed-sweep regression」

各 $(i_d,i_q)$ で **3〜5 速度**を振り $v$ vs $\omega_e$ を線形回帰 → 傾きが $\psi_d,\psi_q$。「600点を同一速度」より「**30動作点 × 4速度 ≈ 120点**」の方が情報量が桁違いに上、というあなたの見積もりは正しいです。情報は電流軸の繰り返しではなく**速度軸**にあります。

実装上の留意点（文献が口を揃える）:
- **鉄損**: ωe に比例（ヒステリシス）・ωe² に比例（渦電流）する電圧成分が傾きを汚す。$v=a+b\,\omega_e+c\,\omega_e^2$ と二次まで当てる、または鉄損抵抗 $R_c$ を明示モデル化（Armando 2013 がこれを含む理由）。
- **温度ドリフト**: スイープ中に $\psi_m,R_s$ が動くと傾きが歪む。1動作点のスイープを短時間で、または恒温下で（後述 Hwang/温度管理論文）。
- **電圧上限**: 高速ほど $v_q\!\approx\!\omega_e\psi_d$ が増え DC リンク上限に当たる。各電流点で上限内に収まる最高速度まで広く振るのが SNR 最良。746V/418rad/s なら 1000rpm は基底速度よりかなり下＝**上にも下にも振れる余地が大きい**。

---

## 3. 文献からの「ネタ」リスト（英語圏中心）

| # | 手法 | 何が使えるか | 主な出典 |
|---|---|---|---|
| **A** | **Multi-speed 回帰**（あなたの案） | 傾き=ψ／切片=Rs+インバータ誤差で完全分離。対称性仮定不要 | back-EMF定数法の負荷時一般化。下記 B/F が実装例 |
| **B** | **定速・電流制御グリッド＋明示補正**（業界標準） | ダイナモ定速駆動で (id,iq) を掃引、定常電圧式から ψ。**Rs・インバータ降下・鉄損・温度を個別補正**。手順の金字塔 | Armando, Bojoi, Guglielmi, Pellegrino (IEEE TIA 2013) |
| **C** | **インバータ非線形性の事前自己同定** | d軸ランプ電流注入で $e(i)$ を LUT 化 → 以後全同定から差し引く。「電圧誤差」側を根絶 | Bedetti/Calligaro/Petrella 系、arctan 2-param モデル |
| **D** | **静止時・信号注入で電流-磁束関係** | ωe=0 で微分インダクタンス $L_{dd},L_{dq},L_{qq}(i_d,i_q)$ を測り積分→磁束マップ。**クロス飽和が直接出る／ダイナモ不要** | "Obtaining the Current-Flux Relations of the Saturated PMSM by Signal Injection" (arXiv 1705.04605) |
| **E** | **構造化飽和関数＋線形回帰** | ノンパラ・マップでなく物理形（べき/有理）を回帰。**疎データでも ψ が identifiable**＝あなたの「100点」案の正則化 | Hinkkanen ら「novel saturation approximating function + multiple linear regression」 |
| **F** | **UKF/EKF オフライン多パラ推定** | ψm・Rs誤差・L誤差・インバータ歪み振幅を**同時**推定。低速走行試験のみ | "UKF-based Offline Estimation of PMSM Magnet Flux Linkage Considering Inverter Dead-time" (2022) |
| **G** | **抵抗・インバータ不確かさ下の磁束マップ同定** | モデル化の過程で R とインバータ非線形性を**相殺**（A/F の親戚） | "Identification of Flux Linkage Map of PMSM Under Uncertain Circuit Resistance and Inverter Nonlinearity" |
| **H** | **dq-θ／空間高調波マップ** | EMF を DFT して基本波＋高調波 EMF を取得。高精度 sim 向け（現状はオーバースペック気味） | Lee/Kwon (IEEE 2018, 2021)、"dq-θ Flux Map Model" |

補足: PoliTo の **SyR-e**（オープンソース）は B 系のフラックスマップ後処理ツールがあり、手順の実装参照に使えます。

---

## 4. このプロジェクトへの当てはめ（推奨）

C 案の磁束同定を作り直すなら、**A＋C＋E の組み合わせ**が筋が良いです:

1. **C（事前）**: d軸ランプ電流でインバータ誤差 $e(i)$ を LUT 化。`Vdt` を ψ 同定から切り離す（縮退の片側を消す）。
2. **A（本体）**: 20〜30 の代表 $(i_d,i_q)$ 点で ωe を 3〜5 段スイープ → $v$ vs ωe を二次回帰、傾きから $\psi_d,\psi_q$。ペアリング・対称性仮定を全廃。
3. **E（仕上げ）**: 得た疎な ψ サンプルに構造化飽和関数を当てて全面マップへ外挿（少データでも安定）。
4. **B（参照）**: Armando 2013 を手順書の原典に。鉄損・温度補正の作法をここから取る。

これは現状の `flux_identify.py`（単一 ωe・ペア平均・`Vdt`/`Rs` 半縮退）を、**速度軸で素直に解ける問題**に置き換える方向で、あなたが感じた限界の真因（identifiability 不足）にそのまま効きます。

次にやるとしたら、(a) **取得計画**（電流点×速度段の設計＋電圧上限チェック）を一枚に落とすか、(b) **A の回帰プロトタイプ**を既存データで（速度1段しかないので合成 or 2段でも）試して傾き分離の感度を確認するか。どちらから詰めますか？

---

**Sources:**
- [Armando et al., Experimental Identification of the Magnetic Model of Synchronous Machines (IEEE TIA 2013)](https://www.researchgate.net/publication/356277574_The_dq-theta_Flux_Map_Model_of_Synchronous_Machines)（dq-θ 版経由）
- [Identification of Flux Linkage Map of PMSM Under Uncertain Circuit Resistance and Inverter Nonlinearity](https://www.researchgate.net/publication/318126835_Identification_of_Flux_Linkage_Map_of_Permanent_Magnet_Synchronous_Machines_Under_Uncertain_Circuit_Resistance_and_Inverter_Nonlinearity)
- [Obtaining the Current-Flux Relations of the Saturated PMSM by Signal Injection (arXiv 1705.04605)](https://arxiv.org/pdf/1705.04605)
- [UKF-based Offline Estimation of PMSM Magnet Flux Linkage Considering Inverter Dead-time Voltage Error (IEEE 2022)](https://ieeexplore.ieee.org/document/9773526/)
- [Stand-still self identification of flux characteristics for SynRM using saturation approximating function + multiple linear regression](https://www.researchgate.net/publication/308813704_Stand-still_self_identification_of_flux_characteristics_for_SynRM_using_novel_saturation_approximating_function_and_multiple_linear_regression)
- [Experimental Identification of IPMSM Flux-Linkage Considering Spatial Harmonics (IEEE 2018)](https://ieeexplore.ieee.org/document/8558379/) / [Identification of IPMSM Flux-Linkage Map for High-Accuracy Simulation (IEEE 2021)](https://ieeexplore.ieee.org/document/9442885/)
- [Flux linkage maps identification of synchronous AC motors under controlled thermal conditions](https://www.researchgate.net/publication/319028696_Flux_linkage_maps_identification_of_synchronous_AC_motors_under_controlled_thermal_conditions)