
# 自動運転シミュレーションにおける快適性指標の計測方法（Simulink 例）

## 1. 物理量の定義

| 物理量 | 数式（位置 \(x(t)\) の n 次微分） | 単位 | 役割 |
| --- | --- | --- | --- |
| 加速度 \(a(t)=\ddot x(t)\) | \(d^{2}x/dt^{2}\) | m·s⁻² | 乗員が直接感じる慣性力。大き過ぎると身体に負担。 |
| ジャーク \(j(t)=\dot a(t)\) | \(d^{3}x/dt^{3}\) | m·s⁻³ | 加速度の変化速度。滑らかさの指標。 |
| ステップ／スナップ \(s(t)=\dot j(t)\) | \(d^{4}x/dt^{4}\) | m·s⁻⁴ | ジャークの変化速度。最小 snap 軌道設計で抑制対象。[^3] |

## 2. 快適性の経験則（目安値）

| 状況 | 加速度の許容域 | ジャークの許容域 | 出典 |
| --- | --- | --- | --- |
| 横方向（レーンチェンジ・カーブ等） | 0.84 m·s⁻² | 0.42 m·s⁻³ | [^1] |
| 縦方向（発進・制動）—公共交通レベル | 0.93 – 1.47 m·s⁻² | 0.3 – 0.9 m·s⁻³ | [^2] |
| ACC 国際規格 (ISO 22179) 設計上限 | +4 / −5 m·s⁻² (低速側: +2 / −3.5 m·s⁻²) | −5 / −2.5 m·s⁻³ | [^1] |

> **メモ**  
> - 同じピーク加速度でも**短時間（高ジャーク）パルスの方が快適**という報告があるため、パルス幅とジャークのトレードオフが重要。[^2]  
> - ステップ（snap）の公的閾値は未整備だが、**minimum‑snap trajectory** の適用で jerk・加速度も滑らかになる。

## 3. Simulink での計測・監視フロー

### 3.1 典型ブロック構成

```mermaid
graph LR
    V[車速 v(t)] -->|Derivative| A[Acceleration a(t)]
    A -->|Derivative| J[Jerk j(t)]
    J -->|Derivative| S[Snap s(t)]
```

| ブロック | 主な用途 | 留意点 |
| --- | --- | --- |
| Derivative（Continuous / Discrete） | 連続微分・有限差分で各階微分を算出 | 微分はノイズを増幅。**Max step size** や **低域フィルタ**で数値安定化を図る。[^4] |
| Filtered Derivative | 微分＋一次低域フィルタを一体化 | ノイズ抑制と遅れのバランス設定。 |
| Minimum Snap Polynomial Trajectory | Waypoint を与えて位置～snap をまとめて生成 | R2022a 以降。snap を最適化できる。[^5] |

### 3.2 しきい値超過のリアルタイム検知

1. `Abs` ブロックで絶対値を取得  
2. `Compare To Constant` で上表の目安値を設定  
3. 超過時に Boolean フラグを `Logic` → `Trigger` ブロックへ接続し、ログ保存や緊急減速ロジックへ反映

### 3.3 推奨ワークフロー

1. **モデルベース設計**  
   - 軌道プランナで *minimum‑snap* を採用（snap コストを最小化）。  
   - プランナ出力を車両横運動モデルへ入力し、車両固有応答をシミュレーション。  
2. **ポストプロセス**  
   - `To Workspace` で a/j/s を出力 → MATLAB で rms・最大値を統計解析。  
3. **パラメータチューニング**  
   - 加速度または jerk が上限を超える箇所でプランニング・制御ゲインを自動調整（PID ゲインスケジューリングや MPC 約束制約）。

## 4. まとめ

- **加速度 ≤ 1 m·s⁻²、ジャーク ≤ 0.5 m·s⁻³** 程度が「多数乗客が不快に感じない」経験則。  
- snap を最小化する軌道生成（Minimum‑Snap Trajectory）が有効。  
- Simulink では Derivative 系ブロックで階微分を算出し、Compare To Constant で即時超過判定 → 制御へフィードバック可能。  

最終的には **HIL/SIL 試験とユーザ調査** による実機評価・パラメータ微調整が必須。

---

## 参考文献

[^1]: K. N. de Winkel *et al.* “Standards for passenger comfort in automated vehicles: Acceleration and jerk,” *Applied Ergonomics*, vol. 106, 2023.  
[^2]: I. Bae *et al.* “Personalized comfortable driving experience for autonomous vehicles,” arXiv:2001.03908, 2022.  
[^3]: “Fourth, fifth, and sixth derivatives of position,” Wikipedia, last updated 2025‑07‑xx.  
[^4]: MathWorks, “Derivative block — Simulink,” Documentation.  
[^5]: MathWorks, “Minimum Snap Polynomial Trajectory block,” Documentation.
