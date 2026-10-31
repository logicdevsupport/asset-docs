# LogicDevLookAtSmooth (UE) - スムーズ注視コンポーネント

対象アクターをスムーズに注視し続ける LookAt コンポーネントです。全身回転版（`ULogicDevLookAtSmoothComponent`）と IK 首振り版（`ULogicDevLookAtSmoothIKComponent`）の2種類がセットで付属します。

---

## 特徴

- **2種類のLookAtが同梱** — 全身回転と頭部（IK）首振りを使い分けられる
- **C++ / Blueprint 両対応** — `UFUNCTION(BlueprintCallable)` によりBlueprintからも呼び出し可能
- **Details パネルでリアルタイム調整** — パラメータをPlay中に即座に確認・変更可能
- **対応構成（IK版）** — PoseableMeshComponent（指定した名前のボーンを回転）、SkeletalMeshComponent（Animation Blueprint にノードを1つ足して使います。「使い方」参照）、通常のSceneComponent階層（指定した名前の子コンポーネントを回転）で動作します
- **軸制限・角度クランプ対応** — X軸・Y軸ごとに最小/最大角度を設定可能

---

## 動作環境

| 項目 | 内容 |
|------|------|
| Unreal Engine バージョン | UE 5.3 / 5.5 / 5.7 / 5.8 |
| 実装言語 | C++ |
| Blueprint対応 | `UFUNCTION(BlueprintCallable)` 完備。Blueprintのみでも利用可能 |
| 入力システム | 依存なし |

---

## 使い方

### ULogicDevLookAtSmoothComponent（全身回転版）

1. 注視させたい Actor に `LogicDev LookAt Smooth` コンポーネントを追加します
2. Details パネルの **Target** に注視対象の Actor を設定します
3. 必要に応じて **RotationSpeed**、**bEnableXAxis**、**bEnableYAxis**、角度クランプを調整します
4. Play すると Target 方向へ Actor 全体がスムーズに回転します

### ULogicDevLookAtSmoothIKComponent（IK首振り版）

1. 頭部を振らせたいキャラクター Actor に `LogicDev LookAt Smooth IK` コンポーネントを追加します
2. Details パネルの **Target** に注視対象を設定します
3. **HeadBoneName**（デフォルト `"head"`）に、回転させたいボーン名（PoseableMesh・SkeletalMesh の場合）または子 SceneComponent の名前を指定します
4. **IKWeight** で首振りの強さ（0〜1）を調整できます

> BeginPlay時にオーナーActorの構成を自動判別します：`UPoseableMeshComponent` があれば `HeadBoneName` のボーンを直接回転させます。`UPoseableMeshComponent` が無く `USkeletalMeshComponent` がある場合は、頭のボーンに足す回転を計算して出力し、ボーンへの反映はキャラクターの Animation Blueprint が行います（下記「SkeletalMeshComponent で使う」）。どちらも無い場合は `HeadBoneName` と同名の `USceneComponent`（キューブ階層構成など）を回転させます。

### SkeletalMeshComponent（Animation Blueprint で動くキャラクター）で使う

UE では、Animation Blueprint が毎フレームすべてのボーンの姿勢を決めるため、コンポーネントから頭のボーンを直接書き換えることができません。そのため SkeletalMesh の構成では、IK首振り版が「頭のボーンに足す回転」（`HeadLookAtDeltaRotation`）を計算し、キャラクターの Animation Blueprint がそれを頭のボーンに足します。Animation Blueprint に足すのは、変数1つ、イベントグラフのノード3つ、AnimGraph のノード1つです。

1. キャラクターの Actor（SkeletalMeshComponent を持つ Actor。Character の Blueprint など）に `LogicDev LookAt Smooth IK` コンポーネントを追加し、**Target** と **HeadBoneName**（頭のボーン名。Mannequin は `head`）を設定します
2. そのキャラクターが使っている Animation Blueprint を開き、Rotator 型の変数を1つ追加します（例: `HeadLookAtDelta`）
3. イベントグラフで、**Event Blueprint Update Animation** の実行ピンを **Set HeadLookAtDelta** につなぎます。Set の入力には、**Get Owning Actor** → **Get Head Look At Delta Rotation From Actor**（カテゴリ LogicDev | LookAtIK）の戻り値をつなぎます
4. AnimGraph で、Output Pose の直前に **Transform (Modify) Bone** ノードを入れます（ローカル空間のポーズにつなぐと、前後に Local To Component・Component To Local が自動で入ります）。ノードを選び、Details パネルで次のとおり設定します
5. Animation Blueprint をコンパイルして保存します

| Transform (Modify) Bone の項目 | 設定値 |
|---|---|
| Bone to Modify | 頭のボーン（HeadBoneName と同じ。Mannequin は `head`） |
| Rotation の Rotation Mode | **Add to Existing** |
| Rotation の Rotation Space | **World Space** |
| Rotation ピン | 手順2の変数（`HeadLookAtDelta`）の Get をつなぐ |
| Translation Mode・Scale Mode | Ignore（既定のまま） |
| Alpha | 1.0（既定のまま） |

> **Note:** 出力される回転には、角度クランプ・RotationSpeed による補間・IKWeight がすでに反映されています。Animation Blueprint 側で値を加工する必要はありません。Target が無い間と `Pause()` 中は最後の値を保ち（頭は最後の向きのまま）、IKWeight が 0 のときは回転なし（アニメーションの姿勢）になります。

> **Note:** Animation Blueprint はコンポーネントの値を次のアニメーション更新で読むため、頭への反映は PoseableMesh の構成より 1〜2 フレーム遅れます。

> **Note:** 体の正面（左右 0° の基準）は、左右の鎖骨のボーン（`clavicle_l`／`clavicle_r`。無ければ `upperarm_l`／`upperarm_r`）の位置から求めます。これらの名前のボーンが無いスケルトンでは、メッシュのコンポーネントの向きを正面として扱います。

> **Note:** Animation Blueprint を使わずに C++／Blueprint から値を読む場合は、コンポーネントの `HeadLookAtDeltaRotation`（読み取り専用）または `GetHeadLookAtDeltaRotation()` を使ってください。

> 注: Details panelのTargetフィールドは、レベルに配置済みのアクターのみ選択できます。まだレベルに配置されていないBlueprintキャラクター等の場合は、ランタイムでSetTarget()を呼び出してください（例: BeginPlay内）。

### コードからの操作例

```cpp
// C++
ULogicDevLookAtSmoothComponent* LookAt = GetComponentByClass<ULogicDevLookAtSmoothComponent>();
LookAt->SetTarget(NewTargetActor);
LookAt->Pause();
LookAt->Resume();
```

Blueprint からも、コンポーネントを取得して **Set Target** / **Pause** / **Resume** ノードを呼び出すだけで同様に操作できます。IK版（`ULogicDevLookAtSmoothIKComponent`）も同じAPIです。

---

## パラメータ一覧

### ULogicDevLookAtSmoothComponent（全身回転版）

| パラメータ | 型 | デフォルト | 説明 |
|---|---|---|---|
| `Target` | `AActor*` | `nullptr` | 注視対象。nullptrのとき動作停止 |
| `RotationSpeed` | `float` | `5.0`（範囲 0.0〜20.0） | 追従速度（Slerp係数） |
| `bEnableXAxis` | `bool` | `true` | X軸（上下）回転の有効/無効 |
| `bEnableYAxis` | `bool` | `true` | Y軸（左右）回転の有効/無効 |
| `bClampAngleX` | `bool` | `false` | X軸角度制限の有効/無効 |
| `MinAngleX` | `float` | `-60.0`（範囲 -180〜0） | X軸最小角度（度）。bClampAngleX=trueのとき有効 |
| `MaxAngleX` | `float` | `60.0`（範囲 0〜180） | X軸最大角度（度）。bClampAngleX=trueのとき有効 |
| `bClampAngleY` | `bool` | `false` | Y軸角度制限の有効/無効 |
| `MinAngleY` | `float` | `-90.0`（範囲 -180〜0） | Y軸最小角度（度）。bClampAngleY=trueのとき有効 |
| `MaxAngleY` | `float` | `90.0`（範囲 0〜180） | Y軸最大角度（度）。bClampAngleY=trueのとき有効 |
| `ReferenceForward` | `FVector` | `(0,0,0)` | 角度クランプの基準（Yaw=0方向）となるワールド空間ベクトル。`(0,0,0)`のままならBeginPlay時にActorのForwardベクトルを自動使用。BeginPlay後にランタイムで値を変更する場合はSetReferenceForward()の呼び出しが必要 |

> **Note（左右の角度制限の幅は180°以下に）:** `MinAngleY` から `MaxAngleY` までの幅（`MaxAngleY` − `MinAngleY`）は、180° 以下にしてください。向きは最短の回り方で補間されるため、幅が 180° を超えると、一方の端からもう一方の端へ向きを変える途中で、範囲の外（背中側）を通ります。幅が 180° を超える設定で `bClampAngleY` が有効な場合、実行時に Output Log へ Warning を1回表示します。

> **Note（角度制限の端とターゲットの切り替え）:** `bClampAngleY` が有効なとき、ターゲットが角度制限の範囲外にいる間は、最初に外れた側の端に向きを固定します。ターゲットが別のものに替わると（`SetTarget()` でも、Target プロパティへの直接代入でも）、この固定を解除し、新しいターゲットに近い側の端を選び直します（同じターゲットを設定し直した場合は解除しません。いったん null にしてから設定した場合は、同じターゲットでも解除します）。

> **Note（ほかの仕組みが回転を制御している Actor）:** 全身回転版は、毎フレーム Actor の回転を直接設定します。物理シミュレーション中の Actor や、Character Movement（Orient Rotation to Movement、Use Controller Rotation Yaw など）が回転を制御している Actor に付けると、回転を取り合うため、対象外です。そのような Actor では、回転を制御する側の設定を切るか、IK首振り版をお使いください。

> **Note（Unity版との違い）：** Unity版はY-up（水平面はXZ平面）のため、デモではこの値を`(0,0,-1)`のような水平ベクトルとして明示的に設定しています。UE版はZ-up（水平面はXY平面）のため、Unity版と同じ`(0,0,-1)`をそのまま入力すると真下方向になってしまい、水平方向としては無効な値（投影すると縮退）になります。そのため本デモでは`ReferenceForward`をデフォルトの`(0,0,0)`（オート）のままにし、Actor自体の配置回転が最初から正しい水平方向を向くようにしています。Actorの初期向きを変更する場合は、`ReferenceForward`に希望する水平方向（Yaw=0とする方向）を明示的に設定してください。

### ULogicDevLookAtSmoothIKComponent（IK首振り版）

| パラメータ | 型 | デフォルト | 説明 |
|---|---|---|---|
| `Target` | `AActor*` | `nullptr` | 注視対象。nullptrのとき動作停止 |
| `HeadBoneName` | `FName` | `"head"` | 回転させるボーン名（PoseableMesh・SkeletalMesh）または子SceneComponent名 |
| `RotationSpeed` | `float` | `12.0`（範囲 0.0〜20.0） | 追従速度。値が大きいほど速く追います（PoseableMesh・SkeletalMesh の構成は指数的に近づく Slerp、SceneComponent の構成は `RInterpTo` の係数）。Unity版（5.0）と補間方式が異なるため、同程度の自然な追従感になるよう意図的に大きめの値にしています |
| `bClampAngleX` | `bool` | `true` | X軸角度制限の有効/無効 |
| `MinAngleX` | `float` | `-30.0`（範囲 -180〜0） | X軸最小角度（度） |
| `MaxAngleX` | `float` | `30.0`（範囲 0〜180） | X軸最大角度（度） |
| `bClampAngleY` | `bool` | `true` | Y軸角度制限の有効/無効 |
| `MinAngleY` | `float` | `-80.0`（範囲 -180〜0） | Y軸最小角度（度） |
| `MaxAngleY` | `float` | `80.0`（範囲 0〜180） | Y軸最大角度（度） |
| `IKWeight` | `float` | `1.0`（範囲 0〜1） | IK適用強度（0=無効、1=フル） |
| `HeadLookAtDeltaRotation` | `FRotator`（読み取り専用） | – | SkeletalMesh の構成だけで使う出力。頭のボーンに足すワールド空間の回転（上記「SkeletalMeshComponent で使う」参照） |

> **Note（左右の角度制限の幅は180°以下に）:** `MinAngleY` から `MaxAngleY` までの幅（`MaxAngleY` − `MinAngleY`）は、180° 以下にしてください。向きは最短の回り方で補間されるため、幅が 180° を超えると、一方の端からもう一方の端へ向きを変える途中で、範囲の外（背中側）を通ります。幅が 180° を超える設定で `bClampAngleY` が有効な場合、実行時に Output Log へ Warning を1回表示します。

> **Note（角度制限の端とターゲットの切り替え）:** `bClampAngleY` が有効なとき、ターゲットが角度制限の範囲外にいる間は、最初に外れた側の端に向きを固定します。ターゲットが別のものに替わると（`SetTarget()` でも、Target プロパティへの直接代入でも）、この固定を解除し、新しいターゲットに近い側の端を選び直します（同じターゲットを設定し直した場合は解除しません。いったん null にしてから設定した場合は、同じターゲットでも解除します）。

> **Note（Unity版との初期値の違い）:** `RotationSpeed` の 12.0（Unity版は 5.0）は、補間方式の違いによる意図した差です。左右の角度制限の初期値 ±80（Unity版は ±60）は、公開済みのバージョンの動きを変えないよう、この値のままにしています。Unity版と同じ可動域にしたい場合は `MinAngleY`／`MaxAngleY` を ±60 に設定してください。

---

## デモシーンについて

同梱のデモレベル `L_LookAtSmooth_Test`（`Content/LogicDevLookAtSmooth/Demo/`）で動作をすぐに確認できます。

- 左側：全身回転デモ（`Watcher_Body` が黄色い球を追う）
- 右側：IK首振りデモ（`IK_Character` が頭だけで赤い球を追う）

> **Note:** デモシーンはプリミティブ形状で構成されています。Mannequinを使用したイメージはストアページで確認できます（本パッケージには同梱されていません）。

---

## 上位版について

有料の上位版がFabで公開されています。

- `Move_LookAtSmooth_002`（Standard）: https://www.fab.com/listings/9f156df3-0e0a-4faf-a1c1-57476c214b86
- `Move_LookAtSmooth_003`（Plus）: https://www.fab.com/listings/28758cd4-7f9b-406c-8bba-2297a53039a6
- `Move_LookAtSmooth_004`（Advanced）: https://www.fab.com/listings/814c59ce-1615-4b4b-af79-0e0673490b56

---

もしこのアセットが役に立ちましたら、ぜひレビューを書いて応援していただけると開発の励みになります！

以下のストアページからレビューを投稿できます。

https://www.fab.com/listings/b16ba60d-cb73-48ab-9e13-b23cd5e8bd80
