DIYでシミュレーションモデルを作るなら、読む価値とは少し順位が変わります。
**「モデル化しやすい」「結果が見える」「あなたのモータ制御テーマに接続しやすい」**という観点では、以下が特に面白いです。

---

## 最優先でDIYシミュレーション化するとよいもの

| 優先 | 文献テーマ                                                                                                        | DIY価値 | おすすめ度 | 理由                                                                                                                                  |
| -: | ------------------------------------------------------------------------------------------------------------ | ----: | ----: | ----------------------------------------------------------------------------------------------------------------------------------- |
|  1 | **High Power Factor IPMSM Drive System Using Electrolytic Capacitor-Less Inverter**                          | 非常に高い | ★★★★★ | 一番おすすめです。IPMSM、インバータ、DCリンク電圧リップル、入力力率、トルクリップルが全部つながります。モータ制御エンジニアとして、モデル化したときの学びが非常に大きいです。                                          |
|  2 | **Harmonic Current Suppression in Six-Phase PMSM Using Virtual Voltage Vector Modulation**                   | 非常に高い | ★★★★★ | あなたの open-end winding / dual inverter / zero-sequence / 6次・12次リップルの関心にかなり近いです。六相PMSMそのものを厳密にやらなくても、**高調波サブスペース分離モデル**を作るだけで価値があります。 |
|  3 | **Flexible Constraint Model Predictive Current Control of SMPMSM Drives with Switching Frequency Reduction** |    高い | ★★★★☆ | PI電流制御との比較対象として面白いです。有限制御集合MPCまたは連続値MPCを簡易実装し、電流追従、スイッチング回数、電圧制約を比較できます。                                                            |
|  4 | **Loss Evaluation of Magnetic Devices Used in Power Converters**                                             |    高い | ★★★★☆ | 鉄損・銅損・周波数依存損失のモデル化に向いています。あなたのアモルファスコア、鉄損増加、温度補償の検討に接続しやすいです。                                                                       |
|  5 | **Design and Control of Variable Field Permanent Magnet Motors**                                             |    高い | ★★★★☆ | 可変磁束PMモータの簡易モデルを作ると、MTPA/FW/MTPVと磁束可変の関係が見えます。ただし、モータ構造依存が強いので、最初は抽象モデルで十分です。                                                       |

---

## 1番おすすめ：電解コンデンサレスIPMSM駆動シミュレーション

対象文献：

**Survey and Analysis for High Power Factor IPMSM Drive System Using Electrolytic Capacitor-Less Inverter**

これはDIY価値がかなり高いです。

作るモデルは、最初はこの程度で十分です。

```text
単相/三相整流器
    ↓
小容量DCリンクコンデンサ
    ↓
三相インバータ
    ↓
IPMSM dqモデル
    ↓
FOC電流制御
```

見るべき現象はこれです。

```text
DCリンク電圧リップル
入力電流波形
入力力率
モータ電流リップル
トルクリップル
速度による悪化
電流制御帯域との干渉
```

特に面白いのは、**DCリンク電圧が一定ではない条件で、普通のFOCがどこまで耐えるか**です。

普通のPMSMシミュレーションでは、

```text
Vdc = 一定
```

としてしまいがちですが、電解コンデンサレスでは、

```text
Vdc = 大きく脈動する
```

ので、電圧飽和、電流歪み、トルクリップル、入力電流高調波が一気に出てきます。

これは実務感があります。

---

## 2番おすすめ：六相PMSM / 高調波サブスペースモデル

対象文献：

**Harmonic Current Suppression in Six-Phase PMSM Using Virtual Voltage Vector Modulation**

これは、あなたの最近のテーマにかなり合います。

ただし、最初から六相PMSMを完全再現する必要はありません。
おすすめは、**座標変換と高調波成分の分離**から始めることです。

作るモデルはこうです。

```text
six-phase current
    ↓
VSD変換 / 拡張Clarke変換
    ↓
基本波サブスペース αβ
    ↓
高調波サブスペース x-y
    ↓
zero-sequence成分
```

確認することはこれです。

```text
どの電流成分がトルクに寄与するか
どの電流成分が損失だけ増やすか
3次/5次/7次/9次などがどのサブスペースに出るか
PWM電圧ベクトルで高調波電流を抑えられるか
dqリップルにどう見えるか
```

open-end winding dual inverter の検討に接続するなら、さらに、

```text
ゼロ相電圧
ゼロ相電流
相電流高調波
dq 6次リップル
dq 12次リップル
平均トルク低下
```

まで見られます。

これはかなり価値があります。

---

## 3番おすすめ：MPC電流制御 vs PI電流制御

対象文献：

**Flexible Constraint Model Predictive Current Control of SMPMSM Drives with Switching Frequency Reduction**

これは制御屋として面白いです。

作るモデルは、最初はSMPMSMでよいです。

```text
SMPMSM dqモデル
    ↓
PI電流制御
    ↓
MPC電流制御
    ↓
比較
```

比較指標はこれです。

```text
電流追従誤差
トルクリップル
電圧飽和時の挙動
スイッチング回数
電流リップル
計算負荷
```

特にMPCでは評価関数をこう置けます。

```text
J = |id_ref - id_pred|^2
  + |iq_ref - iq_pred|^2
  + λsw * switching_count
  + λv * voltage_limit_penalty
```

これをやると、

```text
電流追従を良くしたい
でもスイッチング回数を減らしたい
電圧制約は破りたくない
```

というトレードオフが非常に見えやすくなります。

ただし、DIY初回としては少し重いです。
先に通常のdq電流制御モデルがある前提ならおすすめです。

---

## 4番おすすめ：鉄損・磁気デバイス損失モデル

対象文献：

**Loss Evaluation of Magnetic Devices Used in Power Converters**
**Improved Core Loss Model for Permanent Magnet Assisted Axial Flux High-Speed Switched Reluctance Motor**

これは、あなたの現在テーマである

```text
アモルファスコア
鉄損
温度ドリフト
トルク補償
P_meas = VdId + VqIq
```

に接続しやすいです。

最初に作るなら、詳細FEMではなく、簡易鉄損モデルでよいです。

```text
Pfe = kh * f * B^α
    + ke * f^2 * B^2
    + ka * f^1.5 * B^1.5
```

または簡略化して、

```text
Pfe = kh * f * B^2 + ke * f^2 * B^2
```

でも十分です。

モータ制御側に接続するなら、

```text
B ≒ ψ / A
f ≒ ωe / 2π
ψd = Ld id + φm
ψq = Lq iq
Pfe = f(ψd, ψq, ωe)
```

のように置くと、dq電流・回転数・磁束と鉄損がつながります。

見るべきものはこれです。

```text
idを弱めると鉄損がどう変わるか
高速域で鉄損がどう増えるか
MTPA点と最小損失点がずれるか
鉄損増加が入力電力にどう見えるか
P_meas - P_expected から異常検知できるか
```

これは、あなたの**電力FBトルク補償**の検討と相性が良いです。

---

## 5番おすすめ：可変界磁PMモータの簡易制御モデル

対象文献：

**Design and Control of Variable Field Permanent Magnet Motors**

これは題材としてはかなり面白いです。

ただし、機械構造の詳細まで入れると重いので、DIYではまず

```text
φm を可変パラメータにしたIPMSM
```

として扱うのがよいです。

通常のIPMSMは、

```text
vd = Rs id + dψd/dt - ωe ψq
vq = Rs iq + dψq/dt + ωe ψd

ψd = Ld id + φm
ψq = Lq iq
```

ですが、ここで

```text
φm = φm(u)
```

のように可変にします。

確認することはこれです。

```text
低速高トルクではφmを大きくする
高速弱め界磁ではφmを小さくする
同じトルクで必要電流がどう変わるか
電圧制約円に対して動作範囲がどう広がるか
MTPA/FW/MTPVがどう変わるか
```

これは、MTPA/FWを理解するには非常によい題材です。

---

## 私なら作る順番

あなたの今の関心を考えると、私はこの順番を推します。

```text
1. 電解コンデンサレスIPMSM駆動モデル
2. 六相PMSM / 高調波サブスペースモデル
3. 鉄損・磁気損失モデルをdqモータモデルに追加
4. MPC電流制御モデル
5. 可変界磁PMモータの簡易モデル
```

ただし、すでにPMSM/IPMSMのdqモデルを持っているなら、実装コストと効果のバランスではこうです。

```text
1. 鉄損モデル追加
2. DCリンクリップル追加
3. 高調波電流注入・dqリップル解析
4. MPC電流制御
5. 六相/VSDモデル
```

---

## 最初のDIYテーマとして一番よいもの

一つだけ選ぶなら、

**電解コンデンサレスIPMSM駆動シミュレーション**

です。

理由は、モデルが大きすぎず、結果が見やすく、実務的だからです。

最小構成はこれでよいです。

```text
IPMSM dqモデル
FOC電流制御
速度一定
Vdc = Vdc0 + ΔV sin(2ωgrid t)
インバータ電圧制限
```

最初から整流器まで作らなくても、

```text
Vdcが100Hzまたは120Hzで脈動する
```

と置くだけで十分面白いです。

見る結果は、

```text
id, iq
vd, vq
電圧飽和率
トルク
入力電力相当
トルクリップルFFT
```

です。

この簡易モデルでも、

```text
DCリンクリップルがあると電流制御がどこで破綻するか
高速域ほどなぜ厳しくなるか
弱め界磁時に電圧余裕がどれだけ減るか
トルクリップルがどの周波数に出るか
```

が見えます。

---

## あなたのテーマに一番接続しやすいもの

現在のあなたの関心に寄せるなら、最も価値があるのはこの組み合わせです。

```text
A. IPMSM dqモデル
B. DCリンクリップル
C. 鉄損モデル
D. P_meas = VdId + VqIq
E. トルク補償または異常検知
```

つまり、文献で言えば、

```text
High Power Factor IPMSM Drive System Using Electrolytic Capacitor-Less Inverter
+
Loss Evaluation of Magnetic Devices Used in Power Converters
```

の組み合わせです。

これはかなり良いDIYテーマです。

単なる「論文再現」ではなく、

```text
電圧リップル
電流制御
磁気損失
入力電力
トルク誤差
補償
```

を一つの小さなMILSにまとめられるからです。
この方向は、あなたの実務テーマにもかなり近いと思います。
