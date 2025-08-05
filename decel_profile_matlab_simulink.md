
# MATLAB / Simulink で作る「逆バス型 (S‑curve) 減速プロファイル」オフラインガイド
最終更新: 2025‑08‑05

---

## 1. 目的
交通信号を検知した時点の **現在速度 `v₀`** と **停止線までの距離 `d`** だけを入力として、  
ジャーク (`j_max`)・減速度 (`a_max`) の快適性制約を守りながら **完全停止 (v → 0 m/s)** する “逆バス型 (S‑curve)” 減速プロファイルを生成し、Simulink 車両モデルへ組み込む手順をまとめます。

---

## 2. 必要パラメータ

| 記号 | 意味 | 典型値 (例) |
|------|------|------------|
| `v₀` | 検知時の車速 [m/s] | 実測 |
| `d`  | 停止線までの距離 [m] | センサ / HD マップ |
| `a_max` | 快適減速度上限 (正値) | 2–3 m/s² |
| `j_max` | 快適ジャーク上限 | 1 m/s³ |
| `T_s` | 制御周期 / サンプル時間 | 0.01–0.05 s |

---

## 3. プロファイル形状 ― 7 相 S‑curve

```
 jerk:        +j   0   -j   0   -j   0  +j
 acceleration: 0 → -a_max ……… -a_max ……… -a_max → 0
 velocity:      (連続的に低下、微分可能)
 position:      (C² 連続)
            |<-- t_j -->|<-- t_c -->|<-- t_j -->|
```

* **`t_j = a_max / j_max`** : ジャーク位相の時間  
* **`t_c`** : 定値減速度区間の時間  

\[
t_c = \frac{d - v₀ t_j - \tfrac12 a_{max} t_j^{2}}{v₀ - a_{max} t_j}
\]

`d` が短く `t_c < 0` になる場合は **非常制動** (a_max を引き上げ) へフォールバック。

---

## 4. MATLAB 実装例

```matlab
function [t,vel,acc,jerk] = decelProfile(v0,d,aMax,jMax,Ts)
% v0 [m/s], d [m], aMax>0, jMax>0, Ts [s]
tj = aMax/jMax;
tc = (d - v0*tj - 0.5*aMax*tj^2) / (v0 - aMax*tj);

if tc < 0                   % 距離不足 → 非常制動へ
    aMax = (v0^2)/(2*d);
    tj   = aMax/jMax;
    tc   = 0;
end

t_total = 2*tj + tc;
tvec  = 0:Ts:t_total;
jerk  = zeros(size(tvec));
acc   = zeros(size(tvec));
vel   = v0*ones(size(tvec));

for k = 2:numel(tvec)
    dt = tvec(k)-tvec(k-1);
    if tvec(k) < tj
        j = -jMax;
    elseif tvec(k) < tj+tc
        j = 0;
    elseif tvec(k) < 2*tj+tc
        j =  jMax;
    else
        j = 0;
    end
    jerk(k) = j;
    acc(k)  = acc(k-1) + j*dt;
    vel(k)  = max(0, vel(k-1) + acc(k-1)*dt + 0.5*j*dt^2);
end

t   = tvec';
vel = vel';
acc = acc';
jerk= jerk';
end
```

* この関数を **MATLAB Function ブロック** に入れ、`v0` と `d` を毎サイクル更新します。

---

## 5. Simulink への組込みパターン

| 方法 | ブロック / ツール | 特長 |
|------|------------------|------|
| **A. Trapezoidal Velocity Profile Trajectory** | Robotics System Toolbox | 3 相加減速だが実装が容易 |
| **B. `smoothTrajectory`** | MATLAB System ブロック (Driving Scenario Toolbox) | ジャーク制限付き自動生成 |
| **C. 独自 MATLAB Function** | §4 の関数 | Toolbox 追加不要、柔軟 |

---

## 6. 車両モデルとの接続

1. **Profiler** → `v_ref(t)` (速度指令)  
2. **Longitudinal Controller** (PI, MPC 等) → 車両モデルトルク入力  
3. **Vehicle Dynamics Blockset™** または独自動力学モデル

---

## 7. テストベンチ

| テスト | 合格基準 |
|--------|----------|
| 快適性 | `|a| ≤ a_max`, `|jerk| ≤ j_max` |
| 停止誤差 | 停止線中心 ±0.3 m |
| 応答性 | 検知距離 50–150 m で停止可否 |
| 計算量 | prof 関数実行時間 < `T_s` |

Test Manager でパラメータスイープを行うと効率的に評価できます。

---

## 8. 参考

* MathWorks File Exchange: Speed‑Acceleration‑Jerk Limited 7‑Phase Solution  
* Traffic Light Negotiation 公式サンプルモデル  
* Robotics System Toolbox ドキュメント – Trapezoidal Trajectory

---

### まとめ
* **S‑curve (逆バス型)** はジャークを滑らかにし乗り心地と応答性を両立。  
* 入力は **検知速度 `v₀`** と **残距離 `d`** のみ。  
* Simulink には純正ブロック/Toolbox/自作関数の 3 つの実装経路があるため、開発環境に合わせて選択してください。
