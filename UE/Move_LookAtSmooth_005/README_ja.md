# LogicDevLookAtSmooth (UE) - スムーズ注視コンポーネント Complete Edition

対象アクターをスムーズに注視し続ける LookAt コンポーネントです。全身回転版（`ULogicDevLookAtSmoothComponent`）と IK 首振り版（`ULogicDevLookAtSmoothIKComponent`）の2種類がセットで付属します。複数の候補の中から注視する相手を自動で選んで切り替えるコンポーネント（`ULogicDevLookAtTargetSelectorComponent`）も付属します。本バージョン（Complete Edition、シリーズの最終版）では、首振りのなめらかなフェードイン・フェードアウト、「正面に一番近い相手」を選ぶモード、相手を選ぶときの視野角・距離・遮蔽物の考慮、エディタのビューポートの補助表示と、開発中だけ使えるゲーム画面のデバッグ表示を追加しました。

本パッケージはSmooth LookAt Componentシリーズの最終版です。これより上位のバージョンは出ません。今後の更新は、不具合の修正と対応バージョンの更新のみです。

---

## 特徴

- **2種類のLookAtが同梱** — 全身回転と頭部（IK）首振りを使い分けられる
- **複数ターゲットの自動切替** — 候補のリストから「一番近い相手」「優先度の高い相手（同じ優先度なら近い方）」**「正面に一番近い相手」（新機能）**を自動で選び、注視する相手を切り替えます。切り替わりすぎを防ぐ設定（保持時間・切替マージン）と、切り替わったことを受け取るイベント（Blueprint で受け取れます）が付いています。全身回転版・IK首振り版のどちらにも使えます
- **視野角・距離・遮蔽物の考慮（新機能）** — 視野の外にいる候補、距離の範囲外にいる候補、物の陰に隠れている候補を、選ばないようにできます。3つとも初期設定は OFF です
- **首振りのなめらかなフェード（新機能）** — `FadeOut()` / `FadeIn()` と **WeightFadeTime** で、IK の首振りを指定した時間でなめらかにやめたり、戻したりできます（0 なら従来どおり即時）
- **ビューポートの補助表示（新機能）** — エディタのビューポートに、角度制限の範囲、視野角・距離の範囲、候補（選べるか・選べないか）、選ばれている相手を表示します
- **ゲーム画面のデバッグ表示（新機能、開発用）** — 同じ情報と、オブジェクトごとの状態の表を Play 中のゲーム画面に表示します。DrawDebug 系の描画なので、Shipping・Test のビルドでは何も描かれません
- **予測追従（Predictive LookAt）対応** — ターゲットの速度を自動推定し、現在位置より少し先を狙うことで、動くターゲットを追う際の見た目の遅れを軽減します。デフォルトはOFFで、OFFのときはターゲットの現在位置を狙います
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
| 入力システム | 依存なし。Enhanced Input などの入力用のプラグインは不要です（同梱デモのキー M・V・F とデバッグ表示の G キーは、プレイヤーコントローラーからキーの状態を直接読み取ります） |

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

> **Note:** 出力される回転には、角度クランプ・RotationSpeed による補間・IKWeight がすでに反映されています。Animation Blueprint 側で値を加工する必要はありません。Target が無い間と `Pause()` 中は最後の値を保ち（頭は最後の向きのまま）、IKWeight が 0 のとき・`FadeOut()` の後は回転なし（アニメーションの姿勢）になります。

> **Note:** Animation Blueprint はコンポーネントの値を次のアニメーション更新で読むため、頭への反映は PoseableMesh の構成より 1〜2 フレーム遅れます。

> **Note:** 体の正面（左右 0° の基準）は、左右の鎖骨のボーン（`clavicle_l`／`clavicle_r`。無ければ `upperarm_l`／`upperarm_r`）の位置から求めます。これらの名前のボーンが無いスケルトンでは、メッシュのコンポーネントの向きを正面として扱います。

> **Note:** Animation Blueprint を使わずに C++／Blueprint から値を読む場合は、コンポーネントの `HeadLookAtDeltaRotation`（読み取り専用）または `GetHeadLookAtDeltaRotation()` を使ってください。

> 注: Details panelのTargetフィールドは、レベルに配置済みのアクターのみ選択できます。まだレベルに配置されていないBlueprintキャラクター等の場合は、ランタイムでSetTarget()を呼び出してください（例: BeginPlay内）。

### 首振りをなめらかにやめる・戻す（新機能）

会話の終わりやカットシーンの開始などで、IK の首振りを急に切らずに、なめらかにやめたり戻したりできます。

```cpp
ULogicDevLookAtSmoothIKComponent* IK = GetOwner()->FindComponentByClass<ULogicDevLookAtSmoothIKComponent>();
IK->FadeOut(0.5f);   // 0.5秒かけて頭を戻し、そのまま止める
IK->FadeIn(0.5f);    // 0.5秒かけて首振りに戻す
IK->FadeOut();       // 引数なし: WeightFadeTime の秒数を使う
```

- SkeletalMesh の構成ではアニメーションの姿勢へ、PoseableMesh と SceneComponent の構成ではコンポーネントが動き始めたときの頭の姿勢へ、なめらかに戻ります
- フェードアウト中も追従の計算は続いているので、`FadeIn()` は今の相手の方向へそのまま戻ります。フェードアウトの途中で `FadeIn()` を呼んだ場合（逆も同じ）は、その位置からなめらかに折り返します
- **WeightFadeTime**（初期値 0）は、**IKWeight** を変えたときにも使われ、新しい強さまでその時間をかけて変わります。0 のときは、従来どおりすべて1フレームで反映されます
- 状態は `IsFadedOut()`（`FadeOut()` から `FadeIn()` まで true）と `GetCurrentWeight()`（いま使っている強さ、0〜1）で分かります

フェードは IK 首振り版だけの機能です。全身回転版は Actor 全体を回すため、戻る先の「アニメーションの姿勢」がありません。止めるときは `Pause()` / `Resume()`（止めている間は向きを保ちます）を使うか、Target を替えてください。

### 複数ターゲットの自動切替を使う

1. `LogicDev LookAt Smooth` または `LogicDev LookAt Smooth IK` を付けた Actor に、`LogicDev LookAt Target Selector` コンポーネントを追加します。**LookAtComponent** / **LookAtIKComponent** が両方とも空の場合、同じ Actor 上のコンポーネントが自動で使われます（両方を持つ Actor では、両方に同じ相手を渡します）
2. **Candidates** に、注視の候補にする Actor を登録します。優先度順で使う場合は、各候補の **Priority**（Details パネルでは 0〜100、大きいほど優先）も設定します
3. **SelectionMode** で選び方を決めます。`Nearest` は一番近い候補を選びます（距離は、このコンポーネントを付けた Actor の位置から測ります）。`Priority` は Priority の値が一番大きい候補を選び、同じ値の候補が複数ある場合は、その中で一番近い候補を選びます。`Facing`（新機能）は、**ViewPoint** から見て、キャラクターの正面（LookAt コンポーネントの基準の向き＝角度制限を測る向き）との角度が一番小さい候補を選びます
4. 切り替わりすぎる場合は **MinHoldTime**（切り替えた後に相手を保持する秒数）と **SwitchMargin**（どれだけ良くなったら切り替えるかの割合。`Nearest`・`Priority` では距離、`Facing` では角度）を調整します。どちらも 0 で無効になります
5. （新機能）キャラクターから見えないはずの候補を選ばせたくない場合は、測る位置を **ViewPoint**（別の Actor。空ならこの Actor）と **ViewPointSocket**（その Actor のボーン・ソケット・SceneComponent の名前。キャラクターなら `head` など）で決め、**bUseViewAngle**（視野角: 基準の向きから左右・上下）、**bUseDistanceLimit**（測る位置から MinDistance〜MaxDistance、cm）、**bCheckLineOfSight**（測る位置から候補への線。**ObstacleChannel** のチャンネルをブロックするもので遮られる）のうち必要なものを ON にします。どれか1つでも満たさない候補は「いない」扱いになります。初期設定はすべて OFF なので、ON にしない限り動きは変わりません
6. 選んだ相手は、LookAt コンポーネントの **Target** に自動で設定されます（`SetTarget()` 経由）。LookAt コンポーネント側の Target は空のままで構いません（設定してあっても、選んだ相手で上書きされます）

> **Note:** Details パネルの Candidates には、レベルに配置済みの Actor しか指定できません。実行中にスポーンする Actor は `AddCandidate()` で追加してください。

候補は C++ / Blueprint からも追加・削除できます（ノードは **LogicDev | TargetSelector** カテゴリにあります）。

```cpp
// C++
ULogicDevLookAtTargetSelectorComponent* Selector = GetOwner()->FindComponentByClass<ULogicDevLookAtTargetSelectorComponent>();

// 候補を追加する（第2引数は優先度。省略時は 0）。登録済みの Actor を渡すと優先度だけが更新されます
Selector->AddCandidate(EnemyActor, 5);

// 候補の優先度を変更する
Selector->SetCandidatePriority(EnemyActor, 10);

// 候補から外す
Selector->RemoveCandidate(EnemyActor);

// 現在選ばれている相手（候補が1つもいないときは nullptr）
AActor* Current = Selector->GetCurrentTarget();

// 相手が切り替わったときに通知を受け取る（前の相手, 新しい相手）。
// HandleTargetChanged は UFUNCTION にしてください: void HandleTargetChanged(AActor* PreviousTarget, AActor* NewTarget);
Selector->OnTargetChanged.AddDynamic(this, &AMyActor::HandleTargetChanged);
```

Blueprint では、コンポーネントの Details パネル（Events）の **On Target Changed** の **+** ボタン、または **Bind Event to On Target Changed** で `OnTargetChanged` を受け取れます。

### 補助表示・デバッグ表示・Play 中の値を残す（新機能）

**エディタのビューポートの補助表示（Play していないとき）**: 3つのコンポーネントは、角度制限の範囲（水色）、視野角・距離の範囲（白）、各候補（緑＝選べる、灰＝視野・距離の範囲外、赤＝遮られている）、選ばれている相手（黄）を、エディタのビューポートに線で表示します。いつ出すかは、各コンポーネントの **ShowGizmos** で選べます: `Always`（既定。いつも）、`Selected`（Actor を選んでいる間だけ）、`Off`（出さない）。ビューポートの **Realtime** が ON のときに出ます。Play 中・パッケージしたゲームには出ません。

**ゲーム画面のデバッグ表示（`LogicDev LookAt Debug View`）**: Place Actors のウィンドウ（All Classes）からレベルに1つ置くだけで、レベル内のすべての LookAt コンポーネントと自動切替のコンポーネントについて、同じ情報を同じ色でゲーム画面に表示します（今向いている方向＝橙、先読み位置＝マゼンタも出ます）。LookAt コンポーネントはこのアクターに依存しないので、消しても動きは変わりません。

- 左上には、オブジェクトごとに1行の表を出します。列は Object・Mode・Target・Blocked（遮られた数）・Out of range（範囲外の数）・Prediction・Head fade で固定です。当てはまらない欄は空欄のままなので、列の位置は変わりません。行はアクターの名前の順です
- **bShowOverlay**（全体）、**bShowAngleLimits**（角度制限）、**bShowCandidates**（候補の線と視野・距離）、**bShowPrediction**（先読み位置）、**bShowText**（表）で、種類ごとに ON/OFF できます。**ToggleKey**（既定 **G キー**）で全体を出し入れします
- 物が多いときは、**MaxRows**（既定 8）で行数を抑え、あふれた分は最後の行に「+N more」と出します。**bOnScreenOnly**（既定 OFF）を ON にすると、カメラに映っていない物は表にも線にも出しません
- DrawDebug 系の描画なので、**Shipping・Test のビルドでは何も描かれません**（描くコードごとコンパイルされません）。外し忘れても製品版に表示は出ません

**調整は Details パネルで**: Play 中に値を変えるには、**F8** で操作をエディタに戻す（Eject）か **Simulate** で動かし、Outliner でアクターを選んで Details パネルで変更します。

**Play 中に変えた値を残す**: エンジン標準の **Keep Simulation Changes** を使います。F8 で Eject した状態または Simulate 中に、値を変えたアクターをビューポートか Outliner で右クリックし **Keep Simulation Changes** を選びます（**K キー**でも同じです）。Play を止めた後も、その Actor の値が残ります。その後レベルを保存してください。

### 予測追従（Predictive LookAt）を使う

1. Details パネルの **Enable Prediction**（**LogicDev | LookAt | Prediction** カテゴリ）をチェックします（デフォルトはOFF。OFFのままなら従来通りの挙動です）
2. **Prediction Time**（先読み秒数）・**Velocity Smooth Time**（速度推定の平滑化時定数）・**Max Prediction Distance**（先読みオフセットの上限、cm）を必要に応じて調整します。各項目の詳細は下記「パラメータ一覧」を参照してください
3. ターゲットの速度は自動推定されます。Target の Actor に物理シミュレーション中のコンポーネントがあればその物理速度（`GetPhysicsLinearVelocity()`）を、ない場合は Actor の位置の変化から速度を推定します。どちらの方式を使うかを指定する設定項目はありません（自動判定）
4. 次の場合、速度推定の内部状態は自動的にリセットされます（前の推定値が引き継がれることはありません）: ターゲットを切り替えたとき（`SetTarget()` の呼び出し・Target プロパティへの直接代入・自動切替のコンポーネントによる切り替えのどれでも）、Target が null になったとき（その後に同じ Actor を設定し直した場合も含む）、`Resume()` を呼んだとき、Enable Prediction を再びONにしたとき
5. 動きの速いターゲットほど、予測の効果を確認しやすくなります

### コードからの操作例

```cpp
// C++
ULogicDevLookAtSmoothComponent* LookAt = GetComponentByClass<ULogicDevLookAtSmoothComponent>();
LookAt->SetTarget(NewTargetActor);
LookAt->Pause();
LookAt->Resume();
LookAt->ResetClampState();

// 予測追従（内部状態の読み取りは検証・デバッグ用）
LookAt->bEnablePrediction = true;
const FVector Velocity = LookAt->EstimatedVelocity;   // cm/s
const FVector AimPoint = LookAt->PredictedPosition;   // 実際に狙っている位置
```

Blueprint からも、コンポーネントを取得して **Set Target** / **Pause** / **Resume** ノードを呼び出すだけで同様に操作できます。**Enable Prediction**・**Estimated Velocity**・**Predicted Position** も Blueprint のプロパティとして使えます。IK版（`ULogicDevLookAtSmoothIKComponent`）も同じAPIです。

> **Note:** 自動切替のコンポーネントがティックしている間は、`SetTarget()` で手動設定した相手は次の更新で上書きされます。相手を手動で決めたい間は、自動切替のコンポーネントのティックを止めてください（**Set Component Tick Enabled** を false。true に戻すと再開します）。

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
| `ShowGizmos` | `ELogicDevLookAtGizmoMode` | `Always` | （新機能）エディタのビューポートに角度制限の範囲・向いている方向・相手への線を出すとき: `Always` / `Selected`（選んでいる間）/ `Off`。Play していないときだけ |

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
| `IKWeight` | `float` | `1.0`（範囲 0〜1） | IK適用強度（0=無効、1=フル）。**WeightFadeTime** が 0 のとき、PoseableMesh・SceneComponent の構成で 0 にすると、従来どおり頭はその向きのまま止まります（戻すには `FadeOut()` を使うか、WeightFadeTime を設定してください） |
| `WeightFadeTime` | `float` | `0`（範囲 0〜5） | （新機能）`FadeIn()` / `FadeOut()` を秒数なしで呼んだときと、IKWeight を変えたときに、頭がなめらかに変わる秒数。0 なら従来どおり1フレームで反映されます |
| `ShowGizmos` | `ELogicDevLookAtGizmoMode` | `Always` | （新機能）エディタのビューポートに角度制限の範囲と相手への線を出すとき: `Always` / `Selected`（選んでいる間）/ `Off`。Play していないときだけ |
| `HeadLookAtDeltaRotation` | `FRotator`（読み取り専用） | – | SkeletalMesh の構成だけで使う出力。頭のボーンに足すワールド空間の回転（上記「SkeletalMeshComponent で使う」参照） |

> **Note（左右の角度制限の幅は180°以下に）:** `MinAngleY` から `MaxAngleY` までの幅（`MaxAngleY` − `MinAngleY`）は、180° 以下にしてください。向きは最短の回り方で補間されるため、幅が 180° を超えると、一方の端からもう一方の端へ向きを変える途中で、範囲の外（背中側）を通ります。幅が 180° を超える設定で `bClampAngleY` が有効な場合、実行時に Output Log へ Warning を1回表示します。

> **Note（角度制限の端とターゲットの切り替え）:** `bClampAngleY` が有効なとき、ターゲットが角度制限の範囲外にいる間は、最初に外れた側の端に向きを固定します。ターゲットが別のものに替わると（`SetTarget()` でも、Target プロパティへの直接代入でも）、この固定を解除し、新しいターゲットに近い側の端を選び直します（同じターゲットを設定し直した場合は解除しません。いったん null にしてから設定した場合は、同じターゲットでも解除します）。

> **Note（Unity版との初期値の違い）:** `RotationSpeed` の 12.0（Unity版は 5.0）は、補間方式の違いによる意図した差です。左右の角度制限の初期値 ±80（Unity版は ±60）は、公開済みのバージョンの動きを変えないよう、この値のままにしています。Unity版と同じ可動域にしたい場合は `MinAngleY`／`MaxAngleY` を ±60 に設定してください。

### ULogicDevLookAtTargetSelectorComponent（複数ターゲットの自動切替）

Details パネルの **LogicDev | TargetSelector** カテゴリ。

| パラメータ | 型 | デフォルト | 説明 |
|---|---|---|---|
| `LookAtComponent` | `ULogicDevLookAtSmoothComponent*` | `nullptr` | 相手を渡す全身回転版。LookAtIKComponent と両方とも空の場合、BeginPlay で同じ Actor 上のコンポーネントを自動で探します |
| `LookAtIKComponent` | `ULogicDevLookAtSmoothIKComponent*` | `nullptr` | 相手を渡す IK 首振り版（同上）。両方設定されている場合は、両方に同じ相手を渡します |
| `Candidates` | `TArray<FLogicDevLookAtCandidate>` | 空 | 候補のリスト。各要素は **Target**（`AActor*`）と **Priority**（`int32`、Details パネルでは 0〜100）を持ちます。空・破棄済み・非表示（**Actor Hidden In Game**）の候補は選ばれません |
| `SelectionMode` | `ELogicDevLookAtSelectionMode` | `Nearest` | `Nearest`: 一番近い候補。`Priority`: Priority が一番大きい候補（同じ値なら近い方）。`Facing`（新機能）: ViewPoint から見て、基準の向きとの角度が一番小さい候補。Priority の値は Priority モードでだけ使われます |
| `MinHoldTime` | `float` | `0.5`（範囲 0〜5） | 相手を切り替えた後、この秒数は、より良い候補が現れても切り替えません（0 で無効）。見ていた相手がいなくなったときと、SelectionMode を変更した直後は、この時間を待たずに選び直します |
| `SwitchMargin` | `float` | `0.1`（範囲 0〜0.9） | 同じ Priority の候補どうしで切り替えるのに必要な差の割合（0 で無効）。0.1 の場合、新しい候補までの距離が、今の相手までの距離より 10% 以上短いときだけ切り替えます（Facing モードでは角度で比べます。例: 今の相手が 20° のとき、18° 未満の候補にだけ切り替えます）。Priority モードで Priority が上の候補に対しては、この差を求めずに切り替えます（MinHoldTime は適用されます） |
| `ViewPoint` | `AActor*` | `nullptr` | （新機能）視野角・距離・遮蔽物の判定と、Facing の角度を測る Actor。空ならこの Actor |
| `ViewPointSocket` | `FName` | `None` | （新機能）ViewPoint（空ならこの Actor）のボーン・ソケット・SceneComponent の名前。指定すると Actor の位置の代わりにそこから測ります。キャラクターなら頭（`head` など）。None や見つからないときは Actor の位置 |
| `bUseViewAngle` | `bool` | `false` | （新機能）基準の向き（操作する LookAt コンポーネントの基準の向き＝角度制限を測る向き。無いときはこの Actor の正面）のまわりの視野の外にいる候補を選びません |
| `ViewAngleY` | `float` | `60`（範囲 0〜180） | （新機能）左右それぞれの視野角（度、片側） |
| `ViewAngleX` | `float` | `45`（範囲 0〜90） | （新機能）上下それぞれの視野角（度、片側） |
| `bUseDistanceLimit` | `bool` | `false` | （新機能）測る位置からの距離が MinDistance より近い、または MaxDistance より遠い候補を選びません |
| `MinDistance` | `float` | `0` | （新機能）cm |
| `MaxDistance` | `float` | `2000` | （新機能）cm |
| `bCheckLineOfSight` | `bool` | `false` | （新機能）物の陰に隠れている候補を選びません。測る位置から候補の Actor の位置へ線を1本飛ばして判定します。この Actor、操作する LookAt コンポーネントの Actor、ViewPoint、候補自身（それぞれにアタッチされた Actor を含む）では遮られません。オーバーラップだけのコリジョン（トリガー）でも遮られません |
| `ObstacleChannel` | `ECollisionChannel` | `Visibility` | （新機能）線のトレースチャンネル。このチャンネルをブロックするものが視線を遮ります |
| `ShowGizmos` | `ELogicDevLookAtGizmoMode` | `Always` | （新機能）エディタのビューポートに視野・距離の範囲、候補、選ばれている相手を出すとき: `Always` / `Selected`（選んでいる間）/ `Off`。Play していないときだけ |
| `OnTargetChanged` | デリゲート `(AActor* PreviousTarget, AActor* NewTarget)` | – | 選んだ相手が替わるたびに呼ばれます。どちらの引数も null のことがあります。前の相手が破棄済みの場合は null になります。`BlueprintAssignable` |

**関数**（`BlueprintCallable`。`GetCurrentTarget` は `BlueprintPure`）

| 関数 | 説明 |
|---|---|
| `void AddCandidate(AActor* NewCandidate, int32 Priority = 0)` | 候補を追加します。登録済みの Actor の場合は優先度だけを更新します。null を渡すと Warning を出して無視します |
| `bool RemoveCandidate(AActor* Candidate)` | 候補から外します。リストに無かった場合は false を返します |
| `bool SetCandidatePriority(AActor* Candidate, int32 Priority)` | 候補の優先度を変更します。リストに無い場合は Warning を出して false を返します |
| `AActor* GetCurrentTarget() const` | 現在選ばれている相手。選べる候補がいないときは null |
| `bool IsSelectable(AActor* Candidate) const` | （新機能）その相手を今選べるなら true（存在し、非表示でなく、ON になっている視野・距離・遮蔽物の条件を満たす）。候補のリストに入っている必要はありません。選択にも同じ判定を使っています |
| `ELogicDevLookAtCandidateStatus GetCandidateStatus(AActor* Candidate, FVector& OutBlockPoint) const` | （新機能）同じ判定の内訳: `Selectable`、`Unavailable`（null・破棄済み・非表示）、`OutOfDistance`、`OutOfViewAngle`、`Blocked`（OutBlockPoint＝線が障害物に当たった位置）。読み取りだけで、選択は変わりません |
| `void GetViewFrame(FVector& OutOrigin, FVector& OutForward, FVector& OutUp) const` | （新機能）視野・距離・遮蔽物と Facing の角度を測る、今の位置と向き |

`Candidates`・`SelectionMode`・`MinHoldTime`・`SwitchMargin` と視野・距離・遮蔽物の設定は `BlueprintReadWrite` なので、実行中に直接変更することもできます。

全身回転版・IK 首振り版の両方に、`ResetClampState()`（`BlueprintCallable`。下記「自動切替のコンポーネントの挙動」の「角度クランプの端」参照）と、読み取り用の `GetCurrentReferenceForward()` / `GetCurrentReferenceUp()`（角度制限を測る基準の向き）があります。IK 首振り版には、`FadeIn()` / `FadeOut()`、`IsFadedOut()`、`GetCurrentWeight()`、読み取り用の `GetCurrentHeadPosition()` / `GetCurrentAimDirection()`（新機能）もあります。

### 予測追従（Predictive LookAt）共通パラメータ（両コンポーネント共通）

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

## 自動切替のコンポーネントの挙動

| 状況 | 動作 |
|---|---|
| 見ていた相手が破棄された・候補から削除された・非表示（Actor Hidden In Game）になった | MinHoldTime を待たずに、次の更新で残りの候補から選び直します |
| （新機能）見ていた相手が視野・距離の範囲から出た、または隠れた（視野・距離・遮蔽物の判定が ON のとき） | 上と同じく「いない」扱いになり、MinHoldTime を待たずに選び直します。再び見えるようになったら、通常の候補として扱います（MinHoldTime・SwitchMargin が適用されます） |
| 選べる候補が1つもいない | LookAt コンポーネントの Target を null にします。LookAt コンポーネントは回転を止め、最後の向きを保ちます。`OnTargetChanged` は（前の相手, null）で呼ばれます。その後に候補が追加される・再び表示されると、すぐにその候補を選びます |
| MinHoldTime の保持中に、より良い候補が現れた・追加された | 保持時間が終わってから切り替えます |
| 候補の Priority を変更した | 次の更新から反映されます。保持中の場合は、保持時間が終わってから切り替えます |
| SelectionMode を変更した | 変更後の最初の更新に限り、MinHoldTime を待たずに選び直します（SwitchMargin は適用されます）。同じ値を設定し直しても変更とはみなしません |
| LookAt コンポーネントの `SetTarget()` を手動で呼んだ | 次の更新で、選ばれている相手に上書きされます。`OnTargetChanged` は呼ばれません |
| LookAt コンポーネントが `Pause()` 中 | 選び直しは続き、Target の更新と `OnTargetChanged` も通常どおり行われます。回転は止まったままで、`Resume()` 後に、その時点で選ばれている相手へ向きます |
| 自動切替のコンポーネントのティックを止めた（Set Component Tick Enabled を false） | 選び直しを止めます。LookAt コンポーネントの Target は最後に選んだ相手のままです。ティックを再開すると、止めたときの状態から選び直しを再開します |
| LookAt コンポーネントが見つからない・設定されていない | 相手の選択と `OnTargetChanged` の呼び出しだけを行います（BeginPlay で Output Log に Warning を出します） |
| 角度クランプの端（bClampAngleY 有効時） | LookAt コンポーネントは、相手が角度制限の範囲外にいる間、最初に外れた側の端に向きを固定します。相手が別のものに替わると（自動切替による切り替え、`SetTarget()`、Target プロパティへの直接代入のどれでも）、この固定を解除し、新しい相手に近い側の端を選び直します。同じ相手を設定し直した場合は解除しません（いったん null にしてから設定した場合は、同じ相手でも解除します）。相手を替えずに解除したい場合は `ResetClampState()` を呼んでください |
| 予測追従が有効 | 相手が切り替わるたびに速度推定がリセットされ、新しい相手の現在位置から推定し直します |

- 自動切替のコンポーネントは `TG_PrePhysics` のティックグループでティックし、相手を渡す LookAt コンポーネントのティックの前提（tick prerequisite）として登録されるため、必ず LookAt コンポーネントより先に判定します。切り替えは、その同じフレームの回転に反映されます
- `Nearest`・`Priority` の距離は 3 次元の直線距離で、このコンポーネントを付けた Actor の位置（`GetActorLocation()`）から測ります。IK 首振り版に使う場合も、頭の位置ではなく Actor の位置（キャラクターでは通常、カプセルやルートの位置）が基準です。Facing の角度と、視野角・距離・遮蔽物の判定は、**ViewPoint** と **ViewPointSocket** で決めた位置から測ります
- 遮蔽物の判定は、測る位置から候補の Actor の位置（ピボット）への線1本です。ピボットが隠れていれば、一部が見えていても「隠れている」扱いになります（逆も同じ）。候補自身にコリジョンは要りません
- 視野角は、左右と上下を別々に測る角度で判定します（円錐ではありません）
- 判定は毎回すべての候補に対して行います（フレームの間引きはしません）

**MinHoldTime と SwitchMargin の効果の例（当方の測定）**: ほぼ同じ距離で前後に揺れる2つの候補（10秒間に「近い方」が46回入れ替わる動き）を全身回転版に選ばせたとき、切り替えの回数は、両方 0 で 46 回、MinHoldTime 0.5 のみで 15 回、SwitchMargin 0.1 のみで 25 回、両方（デフォルト値）で 11 回でした。

---

## 仕様上の限界

**複数ターゲットの自動切替**

- 予測追従を有効にしている場合、相手が切り替わるたびに速度推定が 0 からやり直しになります。切り替えが頻繁だと先読みが効いている時間が短くなるため、MinHoldTime・SwitchMargin で切り替えの頻度を抑えてください
- 判定には、自動切替のコンポーネントがティックした時点（`TG_PrePhysics`）の候補の位置を使います。フレームの後半で動く候補（物理シミュレーションで動くもの、後のティックグループで動かすものなど）は、1フレーム前の位置で判定されます（近い方の入れ替わりの検出が1フレーム遅れます）

**予測追従**

- 予測追従は、ターゲットの**現在の速度**に基づく単純な線形の先読みです。ターゲットが加速・減速している場合や、旋回中（速度の向きが変化している場合）は、先読み位置に誤差が生じます
- 弾道計算や、発射物の速度を考慮した迎撃位置の計算は含まれていません
- 物理速度を使うかどうかは、ターゲットを設定した時点で判定します。ターゲットを設定した後で物理シミュレーションを開始した場合は、ターゲットを切り替えるまで位置の変化からの推定が続きます（推定値としては正しく機能します）
- Target の Actor に物理シミュレーション中のコンポーネントが複数ある場合は、最初に見つかったものの速度を使います

---

## デモシーンについて

同梱のデモレベル `L_LookAtSmooth_Test`（`Content/LogicDevLookAtSmooth/Demo/`）で動作をすぐに確認できます。

- 左側：全身回転デモ（傾く台に乗った `Watcher_Body` が、黄色い球 `A` と緑の球 `A2` のどちらかを追います。台が傾いても追従が続くこと＝乗り物・ドローンの傾きへの自動追従も確認できます）
- 右側：IK首振りデモ（`IK_Character` が頭だけで、赤い球 `B` と紫の球 `B2` のどちらかを追います。赤い球は左右に加えて上下にも動くため、頭の上下方向の追従も確認できます）
- 複数ターゲットの自動切替：開始時は「近い順」（`Nearest`）です。2つの球は左右に往復しており、近い方が入れ替わるたびに、キャラクターが見る相手を切り替えます
- モードの切り替え：Play 中に **M キー**を押すと、「近い順」（`Nearest`）→「優先度順」（`Priority`）→「正面に近い順」（`Facing`）の順に切り替わります。デモでは `A2`・`B2` の Priority を 2、`A`・`B` を 1 にしてあるため、優先度順にすると、遠くにいても `A2`・`B2` を見続けます（見えない間を除く）。キーを使わずに切り替える場合は、レベル内の `Demo_SelectionMode` アクターの **Mode** を Details パネルで変更してください（Play 中も変更できます）
- ラベルの見方：各キャラクターの上にラベルが表示されます。モードと相手のラベルは、現在のモードと、いま見ている相手の名前です（例: `Nearest: A2`、`Priority: B2`）。相手の名前は、その球と同じ系統の色で表示されます。予測のラベルは、予測追従の状態です（`Prediction: OFF` / `Prediction: ON`）
- 予測追従：左右どちらのデモも、4.5秒ごとに予測追従の OFF / ON が自動で切り替わります（開始時は OFF）。ON の間は、いま見ている相手に対して実際に狙っている先読み位置（`PredictedPosition`）を示す小さなマーカーが表示されます。マーカーを見やすくするため、デモのキャラクターは `PredictionTime = 0.4` にしています（コンポーネントのデフォルトは0.25）
- 視野・遮蔽物（新機能）：傾く台の上に壁が立っており、全身回転版の側は遮蔽物を判定するため、`A2` が壁の陰にいる間は選びません（ラベルに `(A2 hidden)` と出ます）。IK 首振り版の側は、頭から左右 ±70°・上下 ±45° の視野を持ち、その外にいる球は選びません（`(B2 out of view)`）。**V キー**で視野・遮蔽物の判定を OFF / ON できます
- 首振りのフェード（新機能）：`IK_Character` は、首振りを一定の周期でフェードアウト・フェードインします（ON 8秒 → フェードアウト 1秒 → OFF 2秒 → フェードイン 1秒）。ラベルに `Look: ON` / `Look: fading out` / `Look: OFF` / `Look: fading in` と出ます。**F キー**で手動でフェードアウト・フェードインできます（周期は止まります）
- デバッグ表示（新機能）：`Demo_Debug` アクター（`LogicDev LookAt Debug View`）が最初から表示しています。**G キー**で表示・非表示を切り替えます。上記「補助表示・デバッグ表示・Play 中の値を残す」を参照してください

> **Note:** デモでのモードの一括切り替え・ラベルの表示・予測の OFF / ON の自動切り替え・フェードの周期は、デモ専用のクラス（`LogicDevDemoSelectionModeSwitcherActor`・`LogicDevDemoSelectionLabelComponent`・`LogicDevDemoPredictionToggleComponent`・`LogicDevDemoFadeToggleComponent`）が行っています。ご自身のレベルでは、これらは不要です。自動切替のコンポーネントの **SelectionMode** と、LookAt コンポーネントの **Enable Prediction** を直接設定してください。

> **Note:** デモシーンはプリミティブ形状で構成されています。Mannequinを使用したイメージはFabの商品ページで確認できます（本パッケージには同梱されていません）。

---

## 導入前の注意（重要）

本パッケージは旧バージョン（`Move_LookAtSmooth_001`、`Move_LookAtSmooth_002`、`Move_LookAtSmooth_003`、`Move_LookAtSmooth_004`）と同じプラグイン名（`LogicDevLookAtSmooth`）・クラス名・namespaceを使用しています。**すでにこれらのいずれかをプロジェクトに導入している場合は、本パッケージを追加する前に必ずプロジェクトから削除してください**。複数のバージョンを同時に導入すると、プラグインの競合やクラス名の重複によるコンパイルエラーになります。

旧バージョンのプラグインフォルダ内のファイル（デモレベル等）を編集して使っている場合は、削除する前にバックアップを取ってください。

---

## 004からのアップグレードについて

`Move_LookAtSmooth_004` をお使いの方向けの注意点です。

- 上記「導入前の注意」の通り、004 も本パッケージを追加する前にプロジェクトから削除してください
- コンポーネントのクラス名と、既存の各プロパティ名は 004 から変更していません。レベルや Blueprint に設定した LookAt コンポーネントと自動切替のコンポーネントの設定はそのまま引き継がれます。保存してある SelectionMode（`Nearest`・`Priority`）はそのままの意味で使われます。新しい設定は初期値で始まります（視野・距離・遮蔽物の判定は OFF、WeightFadeTime は 0）
- 新しい設定を初期値のままにしておけば、動きは 004 と同じです（004 と本バージョンを同じ 4,222 の検査条件で UE 5.3 で動かし、記録した回転と選んだ相手が全フレームでビット単位まで一致することを確認しています）。
- 表示まわりで増えたものは `ShowGizmos`（既定は `Always`）だけです。エディタのビューポートに補助線が出るのを止めたい場合は `Selected` か `Off` にしてください

**003 以前からアップグレードする場合**は、次の点にもご注意ください: 全身回転版は左右だけを追います（パラメータ一覧の全身回転版の Note 参照）。IK版は Animation Blueprint を使って SkeletalMeshComponent に対応しました（「SkeletalMeshComponent（Animation Blueprint で動くキャラクター）で使う」参照）。別のターゲットに替えると角度クランプの端の固定を解除します（「自動切替のコンポーネントの挙動」の「角度クランプの端」参照）。

---

## 上位版について

本パッケージ（`Move_LookAtSmooth_005`）は、シリーズの最終版（Complete Edition、すべての機能を含む版）です。`Move_LookAtSmooth_004`（Advanced）までの機能に、首振りのフェード、「正面に一番近い相手」モード、視野角・距離・遮蔽物の考慮、ビューポートの補助表示、開発用のデバッグ表示を追加しました。

これより上位のバージョンは出ません。今後の更新は、不具合の修正と対応バージョンの更新のみです。

まず無料版で試したい方はこちら: https://www.fab.com/listings/b16ba60d-cb73-48ab-9e13-b23cd5e8bd80

---

もしこのアセットが役に立ちましたら、ぜひFabの商品ページでレビューを書いて応援していただけると開発の励みになります！
