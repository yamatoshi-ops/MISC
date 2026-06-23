# dq軸 MIMO 周波数応答(ボード線図)評価法 — 文献サーベイ

- **作成**: 2026-06-20
- **対象 feature**: `frequency-response`
- **目的**: no8(`sim-controller-no8`)を用いて、IPM / IM、開ループ / 閉ループのいずれでも成立する **dq軸 2入力 → 2出力(MIMO)** の周波数応答(ボード線図)評価法を、英語圏文献ベースで整理し、採用方針を確定する。

---

## 1. 問題設定

評価対象はすべて **dq 2×2 MIMO** に帰着する:

| 入力 | 出力 | 物理的意味 | ループ |
|---|---|---|---|
| 電流指令 $i_d^\*, i_q^\*$ | 電流 $i_d, i_q$ | 電流制御の**追従特性**(reference → output) | 閉ループ |
| 電圧指令 $v_d, v_q$ | 電流 $i_d, i_q$ | **plant / アドミタンス** $Y_{dq}$ | 開ループ |

伝達行列で書くと:

$$
\begin{bmatrix} y_d \\ y_q \end{bmatrix}
=
\begin{bmatrix} G_{dd}(s) & G_{dq}(s) \\ G_{qd}(s) & G_{qq}(s) \end{bmatrix}
\begin{bmatrix} u_d \\ u_q \end{bmatrix}
$$

- 対角項 $G_{dd}, G_{qq}$ = 直接応答
- 非対角項 $G_{dq}, G_{qd}$ = **dq軸間 cross-coupling**(dq変換と plant の動特性に起因)
- **IPM の突極性**($L_d \neq L_q$)、**IM の回転子動特性・非対称**は、この結合を周波数依存で強める

---

## 2. 評価アプローチの全体像(4系統)

英語圏では次の4系統が確立している。**C が「dq だが SISO 的に評価する」アプローチ**に対応する。

| 系統 | 描くもの | MIMO / SISO | 代表文献 |
|---|---|---|---|
| **A. 伝達行列(要素別ボード線図)** | $G_{dd},G_{dq},G_{qd},G_{qq}$ の4枚 | MIMO(素直) | Skogestad & Postlethwaite |
| **B. 特異値ボード線図(σ plot)** | 各周波数で2×2をSVD → $\bar\sigma,\underline\sigma$ の2本 | MIMO(方向非依存) | Skogestad、MATLAB `sigma` |
| **C. 複素ベクトル / 複素伝達関数** | $\underline x=x_d+jx_q$ の複素TF、**両側ボード線図** | **SISO的縮約** | Harnefors 2007 |
| **D. dq インピーダンス/アドミタンス測定** | 2×2 を加振で同定 | MIMO(実測) | Sun 2011、dq-impedance 法 |

---

## 3. 各アプローチ詳細

### A. 伝達行列を要素ごとにボード線図(最も素直)

2×2 の **4要素を個別にボード線図**化する。MATLAB / python-control の `bode` は MIMO を「SISO の配列」として各要素を独立に描画する。

- **長所**: 物理的に直感的。dq結合が非対角項としてそのまま見える。IPM の $L_d\neq L_q$、IM の結合が読める。
- **判定基準**(文献知見): 閉ループ伝達関数のボード線図が **0Hz に対して対称**になれば decoupling 良好。非対称さが結合残差の指標。
- **短所**: 4枚を同時に見ても MIMO 全体の帯域・安定余裕は一目で分からない(→ B で補う)。

### B. 特異値ボード線図(MIMO 帯域・ロバスト性を1枚で)

各周波数で $G(j\omega)$ を**特異値分解**し、最大特異値 $\bar\sigma(\omega)$・最小特異値 $\underline\sigma(\omega)$ をプロット(sigma plot)。

- $\bar\sigma$ = その周波数で**全入力方向にわたる最大ゲイン**。方向に依存しない MIMO の「帯域」「条件数」を表す。
- **SISO なら通常のボード線図と完全一致**。チャネル別ボード線図より、全体応答・安定余裕・条件数の把握に優れる。
- ロバスト解析(感度・相補感度のピーク)に直結。

### C. 複素ベクトル / 複素伝達関数 ★「dq を SISO として評価」する本命

dq を**複素数** $\underline x = x_d + j x_q$ で1変数化する(Harnefors 2007)。

- **対称系**(理想 decoupled・突極なし)なら、2×2 実行列が **1つの複素スカラー伝達関数** $G_+(s)$ に縮約 → 実質 SISO。
- ただしボード線図は **両側(正・負周波数で非対称)**。複素係数系は $+\omega$ と $-\omega$ で応答が異なるため(MATLAB でも複素係数モデルは正/負周波数の2ブランチを描く挙動)。
- **非対称系**(= **IPM の突極性、IM の回転子、不平衡**)では1本では足りず、共役項が現れる:

$$
\underline v(t) = G_+(p)\,\underline i(t) + G_-(p)\,\underline i^{*}(t)
$$

  $G_-$ が **mirror-frequency(負相)結合**。このとき複素 SISO は破綻し、$G_+, G_-$ の **2つの複素TF ⟺ 2×2 実行列** と等価になる。

> **MIMO vs SISO の結論**: 「dq を SISO 化」は複素ベクトル法で可能だが、**綺麗に効くのは理想的に decoupled された対称 plant に限られる**。本 feature が扱う **IPM(突極)/ IM(非対称)では $G_-\neq 0$**(mirror結合)が必ず出るため、純 SISO(複素TF 1本)では不足し、$G_+/G_-$ の両側ボード線図(= 2×2 と等価)が必要。

### D. dq インピーダンス / アドミタンス測定 ★「電圧指令 → 電流」がこれ

**電圧指令 $v_d,v_q$ → 電流 $i_d,i_q$(= 開ループ plant / アドミタンス)** は、系統連系分野の **dq インピーダンス測定**そのもの。

- **古典法**: d軸・q軸に**別々に2回加振**して 2×2 行列 $Y_{dq}$ を構成(step or 正弦掃引)。
- **最新法**(arXiv 2510.12338, 2025): 広帯域 RBS を**1回**同時注入 → $G_+,G_-$ を局所パラメトリック推定 → **両側周波数 $\pm\omega$ から 2×2 を復元**:

$$
Z_{dd} = \tfrac12\big[\,G_+(j\omega) + \overline{G_+(-j\omega)} + G_-(j\omega) + \overline{G_-(-j\omega)}\,\big]
$$

  ($Z_{qq}$ は $G_-$ 符号反転、$Z_{dq}/Z_{qd}$ は虚部結合)。**「両側周波数 ⟺ dq 2×2行列」の変換が明示**されており、**C と D は数学的に同じ構造**だと分かる。
- 注意点: **mirror frequency 効果**、**FFT 窓の開始点**の影響。

---

## 4. 加振信号の選択(同定の実務)

| 信号 | 特徴 | 適性 |
|---|---|---|
| **stepped sine(正弦掃引)** | 最も確実・高SNR、1点ずつ | シミュレータ向き(時間制約が緩い)。**第一候補** |
| **chirp** | 帯域 banded・振幅平坦で**滑らかなFRF** | 多共振系・広帯域を一度に |
| **multisine** | 1回で広帯域、位相設計で crest factor 低減 | 効率重視 |
| **PRBS / RBS / MLBS** | 疑似乱数、非周期でも可(RBS) | インピーダンス同定で多用 |
| **SSFR(standstill FR)** | IEEE Std 115 規格、VSI を波形発生器に流用可 | 同期機パラメータ同定 |

---

## 5. no8 への適用方針(**採用**)

ユーザーの4ケース(IPM / IM × 開 / 閉ループ)はすべて 2×2 MIMO。以下を本 feature の評価方針として採用する:

1. **基本は A(4要素ボード線図)** — cross-coupling が直接見え、IPM の $L_d\neq L_q$・IM の結合が物理的に読める。
2. **B(特異値プロット)を併用** — MIMO の帯域・安定余裕・条件数を1枚で要約。
3. **C / D(複素ベクトル両側・$G_+/G_-$)** — complex-vector PI など decoupling 制御の評価時に強力。**IPM / IM は突極・非対称ゆえ $G_-$(mirror結合)が必ず出る**前提で設計する。
4. **入力の使い分け** — 電流指令 → 閉ループ追従行列(reference → output)/ 電圧指令 → plant アドミタンス(D の枠組みを流用)。
5. **加振** — シミュレータなので **stepped sine** を第一候補。効率を上げるなら chirp / multisine。

### 次の設計課題(実装前に決める)

- no8 における**注入点と取得点**(`v_cmd ↔ i_true` 等、整合性は Phase 0 の校正方針と接続)
- 開 / 閉ループの**切り替え方法**(電流制御を活かす / バイパスする)
- 着手対象の**優先順位**(IPM / IM × 開 / 閉ループ)
- FFT / DFT 窓・周波数グリッド・mirror frequency の取り扱い

---

## 6. 主要 Source リスト

### 基礎理論
- **Harnefors, L.** (2007). *Modeling of Three-Phase Dynamic Systems Using Complex Transfer Functions and Transfer Matrices.* IEEE Trans. Industrial Electronics, **54(4):2239–2248**. — 複素ベクトル / 複素伝達関数の基礎(系統C)。<https://www.semanticscholar.org/paper/Modeling-of-Three-Phase-Dynamic-Systems-Using-and-Harnefors/25dc46ad49b4f9d695b1e6309f23d6d344ca4e11>
- **Skogestad, S. & Postlethwaite, I.** *Multivariable Feedback Control: Analysis and Design* (Wiley). — MIMO 周波数応答・特異値プロットの標準教科書(系統A・B)。

### MIMO ボード線図 / 特異値
- MATLAB — *Frequency Response of a MIMO System*(MIMO を SISO 配列として描画)。<https://www.mathworks.com/help/control/ug/frequency-response-of-a-mimo-system.html>
- MATLAB — *sigmaplot*(特異値プロット)。<https://www.mathworks.com/help/control/ref/sigmaplot.html>
- MIT 16.31 — *Feedback Control MIMO Systems / SVD*。<https://dspace.mit.edu/bitstream/handle/1721.1/45531/16-31Fall-2001/NR/rdonlyres/Aeronautics-and-Astronautics/16-31Feedback-Control-SystemsFall2001/99581B6D-F492-4D3A-963E-6CC5364E7678/0/1631_topic21.pdf>

### dq インピーダンス / アドミタンス測定(系統D)
- **Sun, J.** (2011). *Impedance-Based Stability Criterion for Grid-Connected Inverters.* IEEE Trans. Power Electronics, **26(11):3075–3078**. <https://ieeexplore.ieee.org/document/7812790/>
- *A dq-Domain Impedance Measurement Methodology for Three-Phase Converters in Distributed Energy Systems.* Energies (MDPI) 2018, **11(10):2732**. <https://www.mdpi.com/1996-1073/11/10/2732>
- *Ultrafast Grid Impedance Identification in dq-Asymmetric Three-Phase Power Systems.* arXiv:2510.12338 (2025) — 広帯域1回加振 + $G_+/G_-$ 局所パラメトリック + 両側周波数 → 2×2 復元。<https://arxiv.org/html/2510.12338>
- *DQ-Frame Impedance Measurement of Three-Phase Converters Using Time-Domain MIMO Parametric Identification.* <https://www.researchgate.net/publication/342823321>

### MIMO 安定判別 / decoupling
- *Control Design and Stability Analysis of Power Converters: The MIMO Generalized Bode Criterion.* <https://www.researchgate.net/publication/335846053>
- *DQ Current Control of Voltage Source Converters With a Decoupling Method...* IEEE Xplore. <https://ieeexplore.ieee.org/document/7812790/>

### 加振信号 / 同定
- 励起信号比較(PRBS / chirp / multisine / stepped sine)、SSFR(IEEE Std 115)。<https://ietresearch.onlinelibrary.wiley.com/doi/full/10.1049/iet-epa.2013.0195>

---

*補足: 本サーベイは web 調査(2026-06-20)に基づく。Harnefors 2007 原文の本文は未取得(出版社ページが 403)だが、その複素ベクトル定式化は arXiv:2510.12338 の $\underline v = G_+\underline i + G_-\underline i^{*}$ と整合する。一次資料の精読は実装設計フェーズで行う。*
