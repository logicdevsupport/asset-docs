# LogicDevLookAtSmooth (UE) - スムーズ注視コンポーネント Plus

対象アクターをスムーズに注視し続ける LookAt コンポーネントです。全身回転版（`ULogicDevLookAtSmoothComponent`）と IK 首振り版（`ULogicDevLookAtSmoothIKComponent`）の2種類がセットで付属します。本バージョン（Plus）では、ターゲットの速度を自動推定して未来位置を先読みする「予測追従（Predictive LookAt）」機能を追加しました。

---

## 特徴

- **2種類のLookAtが同梱** — 全身回転と頭部（IK）首振りを使い分けられる
- **予測追従（Predictive LookAt）対応（新機能）** — ターゲットの速度を自動推定し、現在位置より少し先を狙うことで、動くターゲットを追う際の見た目の遅れを軽減します。デフォルトはOFFで、OFFのときは旧バージョン（002）と同じくターゲットの現在位置を狙います
- **C++ / Blueprint 両対応** — `UFUNCTION(BlueprintCallable)` によりBlueprintからも呼び出し可能
- **Details パネルでリアルタイム調整** — パラメータをPlay中に即座に確認・変更可能
- **対応構成（IK版）** — PoseableMeshComponent（指定した名前のボーンを回転）、SkeletalMeshComponent（Animation Blueprint にノードを1つ足して使います。「使い方」参照）、通常のSceneComponent階層（指定した名前の子コンポーネントを回転）で動作します
- **軸制限・角度クランプ対応** — X軸・Y軸ごとに最小/最大角度を設定可能
- **UpAxis対応（全身回転版）** — 乗り物・ドローンなど、自身が傾く対象の「上」方向に自動追従（詳細はパラメータ一覧参照）

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
3. 必要に応じて **RotationSpeed**、**bEnableYAxis**、角度クランプを調整します
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

### 予測追従（Predictive LookAt）を使う（新機能）

1. Details パネルの **Enable Prediction**（**LogicDev | LookAt | Prediction** カテゴリ）をチェックします（デフォルトはOFF。OFFのままなら従来通りの挙動です）
2. **Prediction Time**（先読み秒数）・**Velocity Smooth Time**（速度推定の平滑化時定数）・**Max Prediction Distance**（先読みオフセットの上限、cm）を必要に応じて調整します。各項目の詳細は下記「パラメータ一覧」を参照してください
3. ターゲットの速度は自動推定されます。Target の Actor に物理シミュレーション中のコンポーネントがあればその物理速度（`GetPhysicsLinearVelocity()`）を、ない場合は Actor の位置の変化から速度を推定します。どちらの方式を使うかを指定する設定項目はありません（自動判定）
4. 次の場合、速度推定の内部状態は自動的にリセットされます（前の推定値が引き継がれることはありません）: ターゲットを切り替えたとき（`SetTarget()` の呼び出し・Target プロパティへの直接代入のどちらでも）、Target が null になったとき（その後に同じ Actor を設定し直した場合も含む）、`Resume()` を呼んだとき、Enable Prediction を再びONにしたとき
5. 動きの速いターゲットほど、予測の効果を確認しやすくなります

### コードからの操作例

```cpp
// C++
ULogicDevLookAtSmoothComponent* LookAt = GetComponentByClass<ULogicDevLookAtSmoothComponent>();
LookAt->SetTarget(NewTargetActor);
LookAt->Pause();
LookAt->Resume();

// 予測追従（内部状態の読み取りは検証・デバッグ用）
LookAt->bEnablePrediction = true;
const FVector Velocity = LookAt->EstimatedVelocity;   // cm/s
const FVector AimPoint = LookAt->PredictedPosition;   // 実際に狙っている位置
```

Blueprint からも、コンポーネントを取得して **Set Target** / **Pause** / **Resume** ノードを呼び出すだけで同様に操作できます。**Enable Prediction**・**Estimated Velocity**・**Predicted Position** も Blueprint のプロパティとして使えます。IK版（`ULogicDevLookAtSmoothIKComponent`）も同じAPIです。

---

## パラメータ一覧

### ULogicDevLookAtSmoothComponent（全身回転版）

| パラメータ | 型 | デフォルト | 説明 |
|---|---|---|---|
| `Target` | `AActor*` | `nullptr` | 注視対象。nullptrのとき動作停止 |
| `RotationSpeed` | `float` | `5.0`（範囲 0.0〜20.0） | 追従速度（Slerp係数） |
| `bEnableYAxis` | `bool` | `true` | Y軸（左右）回転の有効/無効 |
| `bClampAngleY` | `bool` | `false` | Y軸角度制限の有効/無効 |
| `MinAngleY` | `float` | `-90.0`（範囲 -180〜0） | Y軸最小角度（度）。bClampAngleY=trueのとき有効 |
| `MaxAngleY` | `float` | `90.0`（範囲 0〜180） | Y軸最大角度（度）。bClampAngleY=trueのとき有効 |
| `ReferenceForward` | `FVector` | `(0,0,0)` | 角度クランプの基準（Yaw=0方向）となるワールド空間ベクトル。`(0,0,0)`のままならBeginPlay時にActorのForwardベクトルを自動使用。BeginPlay後にランタイムで値を変更する場合はSetReferenceForward()の呼び出しが必要 |
| `UpAxis` | `FVector` | `(0,0,1)` | ローカル空間での「上」方向。**Detailsパネルからは非表示**（スクリプト/Blueprint専用、上級者向け。下記Note参照） |

> **Note（全身回転版は左右だけを追います）:** 全身回転版は、左右（ヨー）だけを追います。上下には回転しません。以前のバージョンにあった `bEnableXAxis`（上下回転）と、上下の角度制限（`bClampAngleX`・`MinAngleX`・`MaxAngleX`）は使われなくなり、Details パネルにも表示されません（Blueprint からの読み書きは残してありますが、値は無視されます）。無料版（Move_LookAtSmooth_001）の上下回転の設定も引き継がれません。上下にも向かせたい場合は、IK首振り版（頭だけを上下左右に向けます）をお使いください。

> **Note（左右の角度制限の幅は180°以下に）:** `MinAngleY` から `MaxAngleY` までの幅（`MaxAngleY` − `MinAngleY`）は、180° 以下にしてください。向きは最短の回り方で補間されるため、幅が 180° を超えると、一方の端からもう一方の端へ向きを変える途中で、範囲の外（背中側）を通ります。幅が 180° を超える設定で `bClampAngleY` が有効な場合、実行時に Output Log へ Warning を1回表示します。

> **Note（角度制限の端とターゲットの切り替え）:** `bClampAngleY` が有効なとき、ターゲットが角度制限の範囲外にいる間は、最初に外れた側の端に向きを固定します。ターゲットが別のものに替わると（`SetTarget()` でも、Target プロパティへの直接代入でも）、この固定を解除し、新しいターゲットに近い側の端を選び直します（同じターゲットを設定し直した場合は解除しません。いったん null にしてから設定した場合は、同じターゲットでも解除します）。

> **Note（ほかの仕組みが回転を制御している Actor）:** 全身回転版は、毎フレーム Actor の回転を直接設定します。物理シミュレーション中の Actor や、Character Movement（Orient Rotation to Movement、Use Controller Rotation Yaw など）が回転を制御している Actor に付けると、回転を取り合うため、対象外です。そのような Actor では、回転を制御する側の設定を切るか、IK首振り版をお使いください。

> **Note（UpAxis）:** `UpAxis`は毎フレーム`Owner->GetActorRotation()`で再計算されワールド空間の上方向に変換されるため、乗り物やドローンなど自身が傾く対象にも自動追従します。通常は変更不要です。Detailsパネルからは非表示にしています（`BlueprintReadWrite`のみ、`EditAnywhere`なし）。Blueprint/C++から値を設定すること自体は可能です。

> **Note（ReferenceForwardとUnity版の違い）:** Unity版はY-up（水平面はXZ平面）のため、デモではこの値を`(0,0,-1)`のような水平ベクトルとして明示的に設定しています。UE版はZ-up（水平面はXY平面）のため、Unity版と同じ`(0,0,-1)`をそのまま入力すると真下方向になってしまい、水平方向としては無効な値（投影すると縮退）になります。そのため本デモでは`ReferenceForward`をデフォルトの`(0,0,0)`（オート）のままにし、Actor自体の配置回転が最初から正しい水平方向を向くようにしています。Actorの初期向きを変更する場合は、`ReferenceForward`に希望する水平方向（Yaw=0とする方向）を明示的に設定してください。

### ULogicDevLookAtSmoothIKComponent（IK首振り版）

> IK首振り版はUpAxis非対応です（全身回転版のみの機能）。

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

### 予測追従（Predictive LookAt）共通パラメータ（新機能、両コンポーネント共通）

全身回転版・IK首振り版のどちらでも、以下の同じパラメータが使用できます（Detailsパネルの **LogicDev | LookAt | Prediction** カテゴリ）。

| パラメータ | 型 | デフォルト | 説明 |
|---|---|---|---|
| `bEnablePrediction`（Enable Prediction） | `bool` | `false` | 予測追従の有効/無効。falseの場合、旧バージョンと同じく`Target->GetActorLocation()`をそのまま狙います |
| `PredictionTime` | `float` | `0.25`（最小0） | 先読み秒数。実際に狙う位置は「現在のターゲット位置 + 推定速度 × PredictionTime」になります。PredictionTime は `RotationSpeed` による追従の遅れを打ち消すための値なので、**RotationSpeed を大きく（追従を速く）した場合は PredictionTime を小さく、RotationSpeed を小さく（追従を遅く）した場合は PredictionTime を大きく**する方向で調整してください。下記 `PredictedPosition` で実際に狙っている位置を確認しながら調整するのがおすすめです。**IK首振り版は `RotationSpeed` の初期値が 12 と大きいので、PredictionTime は小さめ（0.08〜0.1 が目安）にしてください**。初期値の 0.25 のままだと、ターゲットより先を見ます（左右に往復するターゲットでの実測: 0.08〜0.1 でターゲットとのずれが 1° 前後以内、0.25 では 5〜10° 先） |
| `VelocitySmoothTime` | `float` | `0.1`（最小0） | 速度推定の平滑化にかかる時定数（秒）。値を大きくするほどノイズに強くなりますが、実際の速度変化への反応は遅くなります。0を指定すると平滑化を行いません |
| `MaxPredictionDistance` | `float` | `1000.0`（cm） | 先読みオフセット（推定速度 × PredictionTime）の上限距離（cm）。**この値には別の役割もあります**: ターゲットの位置が1フレームでこの距離を超えて移動した場合、瞬間移動（テレポート）と判断し、速度推定を一度リセットします（推定値0から再度収束します）。**この値を小さくしすぎると、実際には正当な高速移動をしているターゲットが誤ってテレポートと判定されることがあります**のでご注意ください。0以下を指定すると、先読みオフセットの上限とテレポート判定のどちらも無効になります |
| `EstimatedVelocity` | `FVector`（読み取り専用） | – | 現在推定されているターゲットの速度（cm/s）。検証・デバッグ用に Blueprint/C++ から読み取れます |
| `PredictedPosition` | `FVector`（読み取り専用） | – | 実際に狙っている位置（予測ONなら先読み位置、OFFならターゲットの現在位置。Target が null の間は `(0,0,0)`）。検証・デバッグ用に Blueprint/C++ から読み取れます |

---

## 仕様上の限界（新機能について）

- 予測追従は、ターゲットの**現在の速度**に基づく単純な線形の先読みです。ターゲットが加速・減速している場合や、旋回中（速度の向きが変化している場合）は、先読み位置に誤差が生じます
- 弾道計算や、発射物の速度を考慮した迎撃位置の計算は含まれていません
- 物理速度を使うかどうかは、ターゲットを設定した時点で判定します。ターゲットを設定した後で物理シミュレーションを開始した場合は、ターゲットを切り替えるまで位置の変化からの推定が続きます（推定値としては正しく機能します）
- Target の Actor に物理シミュレーション中のコンポーネントが複数ある場合は、最初に見つかったものの速度を使います

---

## デモシーンについて

同梱のデモレベル `L_LookAtSmooth_Test`（`Content/LogicDevLookAtSmooth/Demo/`）で動作をすぐに確認できます。

- 左側：全身回転デモ（`Watcher_Body` が黄色い球を追う。傾く台の上に乗せることで、乗り物・ドローンの傾きへの自動追従も確認できます）
- 右側：IK首振りデモ（`IK_Character` が頭だけで赤い球を追う。赤い球は左右に加えて上下にも動くため、頭の上下方向の追従も確認できます）
- 予測追従（新機能）：左右どちらのデモも、4.5秒ごとに予測追従の OFF / ON が自動で切り替わります（開始時は OFF）。各キャラクターの上に現在の状態（`Prediction: OFF` / `Prediction: ON`）が表示され、ON の間は、実際に狙っている先読み位置（`PredictedPosition`）を示す小さなマーカーが表示されます。同じキャラクター・同じターゲットのまま OFF と ON を見比べることで、予測追従による追従の遅れの違いを確認できます。マーカーを見やすくするため、デモのキャラクターは `PredictionTime = 0.8` にしています（コンポーネントのデフォルトは0.25）

> **Note:** デモでの OFF / ON の自動切り替えは、デモ専用のコンポーネント `LogicDevDemoPredictionToggleComponent` が行っています。ご自身のレベルで予測追従を使う場合、このコンポーネントは不要です。コンポーネントの **Enable Prediction** を直接設定してください。

> **Note:** デモシーンはプリミティブ形状で構成されています。Mannequinを使用したイメージはFabの商品ページで確認できます（本パッケージには同梱されていません）。

---

## 導入前の注意（重要）

本パッケージは旧バージョン（`Move_LookAtSmooth_001`、`Move_LookAtSmooth_002`）と同じプラグイン名（`LogicDevLookAtSmooth`）・クラス名・namespaceを使用しています。**すでに`Move_LookAtSmooth_001`または`Move_LookAtSmooth_002`をプロジェクトに導入している場合は、本パッケージを追加する前に必ずプロジェクトから削除してください**。複数のバージョンを同時に導入すると、プラグインの競合やクラス名の重複によるコンパイルエラーになります。

旧バージョンのプラグインフォルダ内のファイル（デモレベル等）を編集して使っている場合は、削除する前にバックアップを取ってください。

---

## 002からのアップグレードについて

`Move_LookAtSmooth_002` をお使いの方向けの注意点です。

- 予測追従機能はデフォルトでOFFのため、Enable Prediction を明示的にONにしない限り、002と同じくターゲットの現在位置を狙います
  （固定60fpsで同梱デモの269フレームを毎フレーム記録して002と比較し、IK首振り版は頭の回転が全フレームでビット単位まで一致することを確認済みです。全身回転版もUE 5.5/5.7ではビット単位で一致し、5.3/5.8で出た差も最大約3×10⁻⁷°の浮動小数点の丸め誤差のみでした）
- コンポーネントのクラス名と、既存の各プロパティ名（RotationSpeed、bClampAngleY 等）は002から変更していません。新しく追加した予測追従のプロパティは、上記のデフォルト値から始まります
- 上記「導入前の注意」の通り、002自体も003を追加する前にプロジェクトから削除してください

---

## 上位版について

本パッケージ（`Move_LookAtSmooth_003`）は、`Move_LookAtSmooth_001`（無料版）・`Move_LookAtSmooth_002`（Standard、UpAxis対応）に予測追従を追加した Plus 版です。

まず無料版で試したい方はこちら: https://www.fab.com/listings/b16ba60d-cb73-48ab-9e13-b23cd5e8bd80

さらに上位のバージョン（`Move_LookAtSmooth_004`、Advanced）がFabで公開されています: https://www.fab.com/listings/814c59ce-1615-4b4b-af79-0e0673490b56

---

もしこのアセットが役に立ちましたら、ぜひFabの商品ページでレビューを書いて応援していただけると開発の励みになります！
→ [レビューはこちら](https://www.fab.com/listings/28758cd4-7f9b-406c-8bba-2297a53039a6)
