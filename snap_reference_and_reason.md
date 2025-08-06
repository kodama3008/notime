
# Snap（位置4階微分）に関する実測値・設計目安と公的閾値が未整備な理由

## 1. 実測・設計で報告されている代表的な snap 値

| 用途・文献 | 条件・備考 | snap の大きさ |
| --- | --- | --- |
| **ローラーコースター**（Dive Coaster *Valkyria*） | 加速度 −0.2 g → +4 g へ急変．センサー計測 | **+22 g s⁻² ≈ +216 m·s⁻⁴**<br>**−32 g s⁻² ≈ −314 m·s⁻⁴**[^1] |
| **高精度リニア位置決め装置**（半導体露光等） | 最小時間 50 ms で 5 cm 移動する 4 次基準軌道 | **最大 64 000 m·s⁻⁴**（設計値）[^2] |
| **鉄道（欧州 tilting train 制御案）** | 乗り心地限界：横 jerk ≤ 0.3 m·s⁻³ | **~0.3 m·s⁻⁴**（0 → 0.3 m·s⁻³ を 1 s 立ち上げと仮定）[^3] |
| **在来高速鉄道の線形設計指針** | Lateral Change of Acceleration（横 jerk）上限 0.4 m·s⁻³ | **~0.4 m·s⁻⁴**（同上）[^4] |
| **自動運転車向け快適性モデル（研究例）** | 5–7 次多項式で jerk を線形に制御 | **1–2 m·s⁻⁴**（論文中の事例／筆者換算）[^5] |

> **読み替えのポイント**  
> - **ローラーコースターの 200–300 m·s⁻⁴ は「スリル演出」用**であり，自動車・鉄道とは 2 桁以上違います。  
> - 乗客輸送を目的とする**鉄道・自動車では ⩽ 数 m·s⁻⁴ 程度**に抑えるのが一般的設計方針です。  
> - 産業用アクチュエータやドローンでは，構造強度とアクチュエータ帯域が許す限り **数千〜数万 m·s⁻⁴** を許容するケースもあります（快適性より工程時間が優先）。

---

## 2. 「公的閾値が未整備」の理由

1. **人体応答モデルが加速度・jerk 止まり**  
   国際規格 *ISO 2631‑1*（Whole‑Body Vibration）や鉄道 *EN 12299* は，加速度 r.m.s. で快適性を評価します。jerk についても勧告値（例 15 g·s⁻¹）はありますが，snap まで扱う標準は存在しません。  
   → **高次微分は快適性モデルに未実装** のため閾値を定義できない。
2. **計測ノイズと再現性の問題**  
   加速度を 2 回微分するとノイズが大きくなるため，フィルタ仕様やサンプリング周波数で値が変わります。  
   → **規格化できる再現性が確保しにくい**。
3. **jerk を連続関数にすれば snap は自動的に有限**  
   航空・自動車・鉄道の軌道設計では「jerk の連続性」を確保（例：5 次多項式 / クロソイド）。jerk を線形に立ち上げれば snap は定数になり十分小さく抑えられる。  
   → **snap を直接管理しなくても問題が起きにくい**。
4. **人間の知覚が snap まで要求しない**  
   乗員が不快と感じる主周波数帯（0.1–10 Hz）の支配項は加速度と jerk。snap まで抑えても体感差が統計的に明瞭でないと報告される。
5. **規制当局／産業界の優先度**  
   車両安全は衝突エネルギ・最大加速度が優先。快適性も jerk で十分管理できるとして，snap にコストを割く誘因が乏しい。

---

## 3. 実務的な目安値（自動車・AGV の例）

| 走行シーン | jerk 設計上限 | 推奨 snap 上限<br>(jerk の立ち上げ時間 Δt=0.5–1 s に対し) |
| --- | --- | --- |
| 高速道路レーンキープ / ACC | 0.5 m·s⁻³ | **0.5–1 m·s⁻⁴** |
| 都市部発進・停止 | 1.0 m·s⁻³ | **1–2 m·s⁻⁴** |
| 緊急フルブレーキ（安全優先） | jerk 制御不可（ABS 主導） | snap 評価対象外 |

> **Simulink 実装例**  
> `jerk_cmd = ramp(acc_cmd, T_ramp)` として **ramp の傾き 1/T_ramp が snap** に相当。  
> - `T_ramp=1 s` → snap = jerk_max / 1 s  
> - `Compare To Constant` ブロックで ±(1–2) m·s⁻⁴ を監視すると，多くの公道シナリオで快適性を外れません。

---

## 4. まとめ

- **公的な一律閾値は存在しない** が，旅客輸送システムでは  
  **snap ⩽ 数 m·s⁻⁴** 程度に設計するのが実務慣行。  
- 極端例（ローラーコースター）は ±200–300 m·s⁻⁴ でも「安全・許容範囲内」であり，産業用アクチュエータではさらに大きな値が用いられる。  
- 規格が未整備なのは，人体快適性モデルの限界・測定再現性・コスト対効果が主因。  
- **Simulink では jerk を連続関数に制約**（5 次ポリノミアルや `minsnap` ブロック）し，snap は監視用の参考指標とするのが現実的。

---

## 参考文献

[^1]: A. M. Pendrill & D. Eager, “Velocity, acceleration, jerk, snap and vibration: forces in our bodies during a roller‑coaster ride,” *Physics Education*, 55(5), 2020.  
[^2]: Li, X. *et al.*, “Fourth‑order reference trajectory generation for ultra‑precision positioning,” *IET Control Theory & Applications*, 14(12), 2020.  
[^3]: Fujimoto, D. *et al.*, “Strategies for less motion sickness on tilting trains,” Proc. *Computers in Railways*, 2010.  
[^4]: Vankov, L., “Analysis of hyperbolic transition curve geometry,” *Periodica Polytechnica Civil Engineering*, 59(4), 2015.  
[^5]: Sun, B. *et al.*, “Survey of Autonomous Vehicle Behaviors: Trajectory Planning, Motion Control and Decision‑Making,” *Sensors*, 24(3), 2024.
