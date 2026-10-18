# LogicDevLookAtSmooth - スムーズ注視コンポーネント Advanced

対象オブジェクトをスムーズに注視し続ける LookAt コンポーネントです。全身回転版（`LogicDevLookAtSmooth`）と IK 首振り版（`LogicDevLookAtSmoothIK`）の2種類がセットで付属します。本バージョン（Advanced）では、複数の候補の中から注視する相手を自動で選んで切り替えるコンポーネント（`LogicDevLookAtTargetSelector`）を追加しました。

---

## 特徴

- **2種類のLookAtが同梱** - 全身 Y 軸回転と IK 首振りを使い分けられる
- **複数ターゲットの自動切替に対応（新機能）** - 候補のリストから「一番近い相手」または「優先度の高い相手（同じ優先度なら近い方）」を自動で選び、注視する相手を切り替えます。切り替わりすぎを防ぐ設定（保持時間・切替マージン）と、切り替わったことを受け取るイベントが付いています
- **予測追従（Predictive LookAt）に対応** - ターゲットの速度を自動推定し、未来位置を先読みして追従します。無効時（デフォルト）は、Target.position をそのまま狙います
- **乗り物・ドローンの傾きに自動追従** - 取り付けるだけで、物理演算やアニメーションでオブジェクト自身（親を含む）が傾いた場合でも、正しい「上」方向を保ったまま Target を追従します（設定不要）
- **Inspector でリアルタイム調整** - パラメータをプレイ中に即座に確認・変更可能
- **Humanoid Animator を自動判別** - Humanoid の場合は `OnAnimatorIK` で頭を向けます（Animator Controller の IK Pass を有効にする必要があります。下記「使い方」参照）。非 Humanoid の場合は指定したボーンを直接回転させます
- **角度クランプ対応** - 全身回転版は Y 軸（左右）、IK 首振り版は X・Y 軸ごとに最小/最大角度を設定可能
- **Update タイミング選択** - `Update` / `LateUpdate` / `FixedUpdate` から選択可能

---

## 動作環境

| 項目 | 内容 |
|------|------|
| Unity バージョン | Unity 2022.3 LTS / Unity 6.3 LTS |
| レンダリングパイプライン | URP (Universal Render Pipeline)。同梱デモシーンのマテリアルが URP 用です。コンポーネント本体のスクリプトは、レンダーパイプライン固有の API を使用していません |
| 入力システム | コンポーネント本体は入力システムに依存しません。同梱デモの M キーによるモード切り替えだけが New Input System を使用します（未導入の場合、キー切り替えだけが無効になります） |

---

## 使い方

### LogicDevLookAtSmooth（全身回転版）

1. 注視させたい GameObject に `LogicDevLookAtSmooth` コンポーネントを追加します
2. Inspector の **Target** フィールドに注視対象の Transform を設定します
3. 必要に応じて **RotationSpeed**、**EnableYAxis**、角度クランプ を調整します（乗り物・ドローン等の傾きには自動追従するため、通常は追加設定不要です）
4. Play ボタンを押すとスムーズに Target を追従します

### LogicDevLookAtSmoothIK（IK首振り版）

**Humanoid の場合**

1. Humanoid の Animator が付いている GameObject（通常はキャラクターのルート）に `LogicDevLookAtSmoothIK` を追加します。**Animator と同じ GameObject に付けてください**。Unity は `OnAnimatorIK` を Animator と同じ GameObject 上のスクリプトにしか呼び出さないため、親や子の GameObject に付けると首は動きません（この場合、開始時に Console へ Warning が1回表示されます）
2. **Animator Controller のレイヤー設定で「IK Pass」を有効にします**（Animator ウィンドウ > Layers > 対象レイヤーの歯車アイコン > IK Pass にチェック）。Unity は IK Pass が有効なレイヤーに対してだけ `OnAnimatorIK` を呼び出すため、無効のままだと頭は動きません（本コンポーネントは IK Pass の設定を検査しないため、警告は表示されません）
3. Inspector の **Target** フィールドに注視対象を設定します
4. **RotationSpeed**（追従の速さ）と **IKWeight**（首振りの強さ、0〜1）を必要に応じて調整します

> Note: Animator の Culling Mode が「Always Animate」以外の場合、キャラクターが画面に映っていない間は Unity が Animator の IK 処理を呼び出さないため、その間は首の向きが更新されません。

**非 Humanoid の場合**

1. キャラクターの GameObject に `LogicDevLookAtSmoothIK` を追加します
2. **HeadBone** に回転させたいボーン（頭の Transform）を設定します
3. Inspector の **Target** フィールドに注視対象を設定します

### 複数ターゲットの自動切替を使う（新機能）

1. `LogicDevLookAtSmooth` または `LogicDevLookAtSmoothIK` を付けた GameObject に、`LogicDevLookAtTargetSelector` を追加します（Add Component > LogicDevSupport > Move > LookAt Target Selector）。**LookAtComponent** / **LookAtIKComponent** が両方とも空の場合、同じ GameObject 上のコンポーネントが自動で使われます
2. **Candidates** に、注視の候補にする Transform を登録します。優先度順で使う場合は、各候補の **Priority**（0〜100、大きいほど優先）も設定します
3. **SelectionMode** で選び方を決めます。`Nearest` は一番近い候補を選びます（距離は `LogicDevLookAtTargetSelector` を付けた GameObject の位置から測ります）。`Priority` は Priority の値が一番大きい候補を選び、同じ値の候補が複数ある場合は、その中で一番近い候補を選びます
4. 切り替わりすぎる場合は **MinHoldTime**（切り替えた後に相手を保持する秒数）と **SwitchMargin**（どれだけ近くなったら切り替えるかの割合）を調整します。どちらも 0 で無効になります
5. 選んだ相手は、LookAt コンポーネントの **Target** に自動で設定されます。LookAt コンポーネント側の Target フィールドは空のままで構いません（設定してあっても、`LogicDevLookAtTargetSelector` が選んだ相手で上書きされます）

候補はスクリプトからも追加・削除できます。

```csharp
var selector = GetComponent<LogicDevLookAtTargetSelector>();

// 候補を追加する（第2引数は優先度。省略時は 0）。登録済みの Transform を渡すと優先度だけが更新されます
selector.AddCandidate(enemy.transform, 5);

// 候補の優先度を変更する
selector.SetCandidatePriority(enemy.transform, 10);

// 候補から外す
selector.RemoveCandidate(enemy.transform);

// 現在選ばれている相手（候補が1つもいないときは null）
Transform current = selector.CurrentTarget;

// 相手が切り替わったときに通知を受け取る（前の相手, 新しい相手）
selector.OnTargetChanged.AddListener((previous, next) =>
{
    Debug.Log("Target changed: " + (next != null ? next.name : "none"));
});
```

`OnTargetChanged` は Inspector からも登録できます。

### 予測追従（Predictive LookAt）を使う

1. Inspector の **Enable Prediction** をチェックします（デフォルトはOFF）
2. **Prediction Time**（先読み秒数）・**Velocity Smooth Time**（速度推定の平滑化時定数）・**Max Prediction Distance**（先読みオフセットの上限）を必要に応じて調整します。各項目の詳細は下記「パラメータ一覧」を参照してください
3. ターゲットの速度は自動推定されます。内部的には、Target に Rigidbody（Kinematic ではないもの）が付いていればその速度を、付いていない場合は位置の差分から速度を推定します。どちらの方式を使うかを指定する設定項目はありません（自動判定）
4. `SetTarget()` の呼び出し、Target フィールドへの直接代入、`LogicDevLookAtTargetSelector` による切り替えのいずれでも、ターゲットが替わると速度推定の内部状態は自動的にリセットされます（旧ターゲットの速度推定値が新しいターゲットに引き継がれることはありません）。切り替えた直後は先読みなし（新しいターゲットの現在位置）から始まり、その後あらためて速度を推定します
5. 動きの速いターゲットほど、予測の効果を確認しやすくなります

### コードからの操作例

```csharp
// ターゲットをコードから変更する
GetComponent<LogicDevLookAtSmooth>().SetTarget(newTarget);

// 注視を一時停止
GetComponent<LogicDevLookAtSmooth>().Pause();

// 注視を再開
GetComponent<LogicDevLookAtSmooth>().Resume();

// 予測追従の内部状態を読み取る（検証・デバッグ用、読み取り専用）
Vector3 velocity = GetComponent<LogicDevLookAtSmooth>().EstimatedVelocity;
Vector3 aimPoint = GetComponent<LogicDevLookAtSmooth>().PredictedPosition;
```

IK 版も同じ API を使用できます。

> Note: `LogicDevLookAtTargetSelector` が有効な間は、`SetTarget()` で手動設定した相手は次の更新で上書きされます。相手を手動で決めたい間は、`LogicDevLookAtTargetSelector` を無効（enabled = false）にしてください。

---

## パラメータ一覧

### LogicDevLookAtSmooth（全身回転版）

| パラメータ | 型 | デフォルト | 説明 |
|---|---|---|---|
| Target | Transform | null | 注視対象。null のときは回転を止め、そのときの向きを保ちます |
| RotationSpeed | float | 5.0 | 追従速度（Slerp 係数） |
| EnableYAxis | bool | true | Y 軸（左右）回転の有効/無効 |
| ClampAngleY | bool | false | Y 軸角度制限の有効/無効 |
| MinAngleY | float | -90 | Y 軸最小角度（度）。ClampAngleY 有効時のみ |
| MaxAngleY | float | 90 | Y 軸最大角度（度）。ClampAngleY 有効時のみ |
| UpAxis | Vector3 | (0, 1, 0) | ローカル空間での「上」方向。**Inspector には非表示**（スクリプトからのみ設定可能な上級者向け項目。下記 Note 参照） |
| ReferenceForward | Vector3 | (0, 0, 0) | Yaw=0の基準となるワールド方向。(0,0,0)（未設定）なら配置時のtransform.forwardを自動使用。非ゼロの値を明示的に設定するとその方向を基準にできます。実行時に動的に変更する場合はコードからSetReferenceForward()を呼び出してください（フィールドへの直接代入はAwake前またはInspector編集時のみ反映されます） |
| UpdateMode | enum | LateUpdate | Update / LateUpdate / FixedUpdate |

> Note: 全身回転版は、左右（Y 軸）だけを追います。上下には回転しません。無料版（Move_LookAtSmooth_001）にある上下回転（EnableXAxis）の設定は引き継がれません。001 で上下回転を ON にして保存したシーン・プレハブでも、本バージョンでは左右だけを追います。

> Note: MinAngleY から MaxAngleY までの幅（MaxAngleY − MinAngleY）は、180° 以下にしてください。向きは最短の回り方で補間されるため、幅が 180° を超えると、一方の端からもう一方の端へ向きを変える途中で、範囲の外（背中側）を通ります。幅が 180° を超える設定で ClampAngleY が有効な場合、実行時に Console へ Warning を1回表示します。

> Note: 全身回転版は、毎フレーム Transform の回転を直接設定します。Rigidbody（非 Kinematic）や CharacterController、ほかのスクリプトが同じオブジェクトの回転を制御している場合は、回転を取り合うため、対象外です。そのようなオブジェクトでは、回転を制御する側を止めるか、見た目用の子オブジェクトに全身回転版を付けるか、IK 首振り版をお使いください。

> Note: UpAxis は取り付けるだけで自動的にオブジェクト自身の傾きに追従するため、通常は変更不要です。見た目が回転して見える不具合を避けるため、Inspector からは意図的に非表示にしています。スクリプトから値を変更すること自体は可能ですが、デフォルト値 `(0, 1, 0)` 以外を手動設定すると、その軸を中心に Yaw 回転が生じ、見た目が回転しているように見える点にご注意ください。

### LogicDevLookAtSmoothIK（IK首振り版）

| パラメータ | 型 | デフォルト | 説明 |
|---|---|---|---|
| Target | Transform | null | 注視対象。null のときは首の回転を止め、最後の向きを保ちます（Humanoid・非 Humanoid 共通） |
| HeadBone | Transform | null | 回転させる頭部 Transform（非 Humanoid 時に使用） |
| RotationSpeed | float | 5.0 | 追従速度（Slerp 係数）。Humanoid・非 Humanoid のどちらでも有効です |
| ClampAngleX | bool | true | X 軸角度制限の有効/無効 |
| MinAngleX | float | -30 | X 軸最小角度（度） |
| MaxAngleX | float | 30 | X 軸最大角度（度） |
| ClampAngleY | bool | true | Y 軸角度制限の有効/無効 |
| MinAngleY | float | -60 | Y 軸最小角度（度） |
| MaxAngleY | float | 60 | Y 軸最大角度（度） |
| IKWeight | float | 1.0 | Humanoid Animator 使用時の IK 強度（0〜1）。非 Humanoid では使われません。Unity の `Animator.SetLookAtWeight` にそのまま渡されます。0 でアニメーションの姿勢になります。値の変更は補間されず、1フレームで反映されます（当方の測定例: 相手が正面から約50°の位置にいるとき、首の向きは 1.0 で約48°、0.5 で約32°、0 で 0°） |
| UpdateMode | enum | LateUpdate | Update / LateUpdate / FixedUpdate（非 Humanoid 時に使用。Humanoid の場合は Animator の IK 処理のタイミングで動作します） |

> Note: Humanoid の場合、`Pause()` 中と Target が null の間は、首は最後の向きのまま止まります（アニメーションの姿勢には戻りません）。アニメーションの姿勢に戻したい場合は、IKWeight を 0 にするか、コンポーネントを無効（enabled = false）にしてください。どちらの場合も、首は1フレームでアニメーションの姿勢に戻ります（滑らかに戻したい場合は、IKWeight をスクリプトから少しずつ変えてください）。再度有効にすると、キャラクターの正面の向きから相手へ RotationSpeed で向き直ります。

> Note: MinAngleY から MaxAngleY までの幅（MaxAngleY − MinAngleY）は、180° 以下にしてください（Humanoid・非 Humanoid 共通）。向きは最短の回り方で補間されるため、幅が 180° を超えると、一方の端からもう一方の端へ向きを変える途中で、範囲の外（真後ろ側）を通ります。Humanoid ではこのとき体ごと大きくねじれて見えることがあります。幅が 180° を超える設定で ClampAngleY が有効な場合、実行時に Console へ Warning を1回表示します。

> Note: Humanoid では、ClampAngleY を無効にした場合も同じことが起きます（範囲の制限が無いため、相手を反対側の相手へ切り替えたときなどに、狙う向きが真後ろを通り、頭と体が大きくねじれて見えることがあります）。Humanoid では ClampAngleY を有効にし、幅を 180° 以下にして使うことをおすすめします。Humanoid で ClampAngleY が無効な場合、実行時に Console へ Warning を1回表示します。

> Note: Humanoidキャラクターに使用する場合、非常に広い角度範囲（180°に近いMin/Max設定）を指定しても、キャラクター自身の首・体幹の可動域（Muscle定義）により、設定値まで実際には追従しきれないことがあります。頭部・体幹のIKのみで、体の向きを変えずに真後ろを自然に見せることは、Unity標準のHumanoid LookAt IKの仕組み上、構造的に難しい点にご注意ください。

### LogicDevLookAtTargetSelector（複数ターゲットの自動切替、新機能）

| パラメータ | 型 | デフォルト | 説明 |
|---|---|---|---|
| LookAtComponent | LogicDevLookAtSmooth | null | 相手を渡す全身回転版。LookAtIKComponent と両方とも空の場合、同じ GameObject 上のコンポーネントを自動で探します |
| LookAtIKComponent | LogicDevLookAtSmoothIK | null | 相手を渡す IK 首振り版（同上）。両方設定されている場合は、両方に同じ相手を渡します |
| Candidates | List | 空 | 候補のリスト。各要素は **Target**（Transform）と **Priority**（int、Inspector では 0〜100）を持ちます。空・破棄済み・非アクティブの候補は選ばれません |
| SelectionMode | enum | Nearest | `Nearest`: 一番近い候補。`Priority`: Priority が一番大きい候補（同じ値なら近い方）。Nearest では Priority の値は使われません |
| MinHoldTime | float | 0.5 | 相手を切り替えた後、この秒数は、より良い候補が現れても切り替えません（0〜5、0 で無効）。見ていた相手がいなくなったときと、SelectionMode を変更した直後は、この時間を待たずに選び直します |
| SwitchMargin | float | 0.1 | 同じ Priority の候補どうしで切り替えるのに必要な差の割合（0〜0.9、0 で無効）。0.1 の場合、新しい候補までの距離が、今の相手までの距離より 10% 以上短いときだけ切り替えます。Priority モードで Priority が上の候補に対しては、この差を求めずに切り替えます（MinHoldTime は適用されます） |
| OnTargetChanged | `UnityEvent<Transform, Transform>` | - | 選んだ相手が替わるたびに呼ばれます。引数は（前の相手, 新しい相手）で、どちらも null のことがあります。前の相手が破棄済みのオブジェクトの場合もあります |

**スクリプト API**

| メンバー | 説明 |
|---|---|
| `void AddCandidate(Transform target, int priority = 0)` | 候補を追加します。登録済みの Transform の場合は優先度だけを更新します。null を渡すと Warning を出して無視します |
| `bool RemoveCandidate(Transform target)` | 候補から外します。リストに無かった場合は false を返します |
| `bool SetCandidatePriority(Transform target, int priority)` | 候補の優先度を変更します。リストに無い場合は Warning を出して false を返します |
| `Transform CurrentTarget { get; }` | 現在選ばれている相手（読み取り専用）。選べる候補がいないときは null |

`Candidates`・`SelectionMode`・`MinHoldTime`・`SwitchMargin` は public フィールドなので、実行中にスクリプトから直接変更することもできます。

また、全身回転版・IK 首振り版の両方に `void ResetClampState()` を追加しました（下記「挙動」の「角度クランプの端」参照）。

### 予測追従（Predictive LookAt）共通パラメータ（両コンポーネント共通）

全身回転版・IK首振り版のどちらでも、以下の同じパラメータが使用できます。

| パラメータ | 型 | デフォルト | 説明 |
|---|---|---|---|
| Enable Prediction | bool | false | 予測追従の有効/無効。falseの場合、狙う位置は常にTarget.positionがそのまま使われます（ターゲット参照の変化を検知する軽量な内部処理は、無効時も動作します） |
| Prediction Time | float | 0.25 | 先読み秒数。実際に狙う位置は「現在のターゲット位置 + 推定速度 × Prediction Time」になります。**確認済みの目安**: RotationSpeed=5、Velocity Smooth Time=0.1 の設定で、全身回転版（円運動するターゲット）では、Prediction Time が 0.25 付近のとき追従の遅れがほぼ消えることを確認しています（相手の方向との差は 1° 未満）。Humanoid の IK 首振り版（左右に往復するターゲット、2種類の動き）では、予測 OFF で平均 7〜10° あった遅れが、Prediction Time 0.25 で平均 0.1〜2.4°（相手より少し先）になりました。他の RotationSpeed の値については未検証です。**RotationSpeed を変更した場合は、下記 Predicted Position を使って実際に狙っている位置を確認しながら、Prediction Time を再調整してください**。調整の方向の目安: Prediction Time は RotationSpeed による追従の遅れを打ち消すための値なので、**RotationSpeed を大きく（追従を速く）した場合は Prediction Time を小さく、RotationSpeed を小さく（追従を遅く）した場合は Prediction Time を大きく**する方向で調整してください |
| Velocity Smooth Time | float | 0.1 | 速度推定の平滑化にかかる時定数（秒）。値を大きくするほどノイズに強くなりますが、実際の速度変化への反応は遅くなります。0以下を指定すると平滑化を行いません |
| Max Prediction Distance | float | 10 | 先読みオフセット（推定速度 × Prediction Time）の上限距離（ワールド単位）。**この値には別の役割もあります**: ターゲットの位置が1ステップでこの距離を超えて移動した場合、瞬間移動（テレポート）と判断し、速度推定を一度リセットします（推定値0から再度収束します）。**この値を小さくしすぎると、実際には正当な高速移動をしているターゲットが誤ってテレポートと判定されることがあります**のでご注意ください。0以下を指定すると、先読みオフセットの上限とテレポート判定のどちらも無効になります |
| Estimated Velocity | Vector3（読み取り専用） | - | 現在推定されているターゲットの速度。検証・デバッグ用に公開されています |
| Predicted Position | Vector3（読み取り専用） | - | 実際に狙っている先読み位置。検証・デバッグ用に公開されています |

---

## LogicDevLookAtTargetSelector の挙動

| 状況 | 動作 |
|---|---|
| 見ていた相手が破棄された・候補から削除された・非アクティブになった | MinHoldTime を待たずに、残りの候補から選び直します。`Destroy()` の場合は、破棄が反映される次のフレームで選び直します |
| 選べる候補が1つもいない | LookAt コンポーネントの Target を null にします。LookAt コンポーネントは回転を止め、最後の向きを保ちます。`OnTargetChanged` は（前の相手, null）で呼ばれます。その後に候補が追加されると、すぐにその候補を選びます |
| MinHoldTime の保持中に、より良い候補が現れた・追加された | 保持時間が終わってから切り替えます |
| 候補の Priority を変更した | 次の判定から反映されます。保持中の場合は、保持時間が終わってから切り替えます |
| SelectionMode を変更した | 変更後の最初の判定に限り、MinHoldTime を待たずに選び直します（SwitchMargin は適用されます） |
| LookAt コンポーネントの `SetTarget()` を手動で呼んだ | 次の判定で、選ばれている相手に上書きされます。`OnTargetChanged` は呼ばれません |
| LookAt コンポーネントが `Pause()` 中 | 選び直しは続き、Target の更新と `OnTargetChanged` も通常どおり行われます。回転は止まったままで、`Resume()` 後に、その時点で選ばれている相手へ向きます |
| `LogicDevLookAtTargetSelector` を無効にした | 選び直しを止めます。LookAt コンポーネントの Target は最後に選んだ相手のままです。再度有効にすると、選び直しを再開します |
| LookAt コンポーネントが見つからない・設定されていない | 相手の選択と `OnTargetChanged` の呼び出しだけを行います（起動時に Warning を出します） |
| 角度クランプの端（ClampAngleY 有効時） | LookAt コンポーネントは、相手が角度制限の範囲外にいる間、最初に外れた側の端に向きを固定します。相手が別のものに替わると（`LogicDevLookAtTargetSelector` による切り替え、`SetTarget()`、Target フィールドへの直接代入のどれでも）、この固定を解除し、新しい相手に近い側の端を選び直します。同じ相手を設定し直した場合は解除しません（いったん null にしてから設定した場合は、同じ相手でも解除します）。相手を替えずに解除したい場合は `ResetClampState()` を呼んでください |
| 予測追従が有効 | 相手が切り替わるたびに速度推定がリセットされ、新しい相手の現在位置から推定し直します |

- 判定は、LookAt コンポーネントが実際に動作するタイミング（その UpdateMode。Humanoid の IK 首振り版の場合は、Animator の Update Mode が Normal なら Update、Animate Physics（Unity 6 では Fixed）なら FixedUpdate）で、LookAt コンポーネントより先に行われます。切り替えは、その同じフレームの回転に反映されます
- 距離は 3 次元の直線距離で、`LogicDevLookAtTargetSelector` を付けた GameObject の位置（transform.position）から測ります。IK 首振り版に使う場合も、頭の位置ではなく GameObject の位置が基準です

**MinHoldTime と SwitchMargin の効果の例（当方の測定）**: ほぼ同じ距離で前後に揺れる2つの候補（10秒間に「近い方」が46回入れ替わる動き）を全身回転版に選ばせたとき、切り替えの回数は、両方 0 で 46 回、MinHoldTime 0.5 のみで 15 回、SwitchMargin 0.1 のみで 25 回、両方（デフォルト値）で 11 回でした。

---

## 仕様上の限界

**複数ターゲットの自動切替**

- 選び方は `Nearest`（距離）と `Priority`（優先度）の2種類です。視野角・距離の上限・遮蔽物（壁の向こうにいるか）は考慮しません。条件に合わない候補を除外したい場合は、`RemoveCandidate()` で候補から外すか、その GameObject を非アクティブにしてください
- 予測追従を有効にしている場合、相手が切り替わるたびに速度推定が 0 からやり直しになります。切り替えが頻繁だと先読みが効いている時間が短くなるため、MinHoldTime・SwitchMargin で切り替えの頻度を抑えてください
- Humanoid の IK 首振り版（Animator の Update Mode が Normal）に使う場合、判定は Update の早い段階（実行順 -100）で行われます。候補を通常の実行順の Update で動かしていると、判定には1フレーム前の位置が使われます（入れ替わりの検出が1フレーム遅れます）

**予測追従**

- 予測追従は、ターゲットの**現在の速度**に基づく単純な線形の先読みです。ターゲットが加速・減速している場合や、旋回中（速度の向きが変化している場合）は、先読み位置に誤差が生じます
- 弾道計算や、発射物の速度を考慮した迎撃位置の計算は含まれていません
- Rigidbody を持たず、FixedUpdate で位置を更新しているターゲットの場合、描画フレームと物理更新のタイミングのずれにより、Estimated Velocity（推定速度）が揺れることがあります（当方の測定では、真値に対して最大で約10%）。対策として、(1) ターゲットに非Kinematic な Rigidbody を付与し、物理演算（または velocity への直接代入）で動かす（Rigidbody の速度を直接参照するようになり、この揺れを避けられます）、(2) Velocity Smooth Time を大きめに設定する（平滑化を強める代わりに、速度変化への反応は遅くなります）、のいずれかが有効です

---

## デモシーンについて

同梱のデモシーン `Demo_LookAtSmooth`（`Assets/LogicDevSupport/Move_LookAtSmooth_004/Demo/`）で動作をすぐに確認できます。

- 左側：全身回転デモ（傾く台に乗った青いカプセルが、黄色い球 `A` と緑の球 `A2` のどちらかを追います。台が傾いても追従が続くことも確認できます）
- 右側：IK 首振りデモ（キャラクターが頭だけで、赤い球 `B` と紫の球 `B2` のどちらかを追います。赤い球は左右に加えて上下にも動くため、頭の上下方向の追従も確認できます）
- 複数ターゲットの自動切替（新機能）：開始時は「近い順」（`Nearest`）です。2つの球は左右に往復しており、近い方がおよそ2〜3.5秒ごとに入れ替わるたびに、キャラクターが見る相手を切り替えます
- モードの切り替え：Play 中に **M キー**を押すと、「近い順」（`Nearest`）と「優先度順」（`Priority`）が切り替わります（New Input System が有効な場合）。デモでは `A2`・`B2` の Priority を 2、`A`・`B` を 1 にしてあるため、優先度順にすると、遠くにいても `A2`・`B2` を見続けます。キーを使わずに切り替える場合は、シーン内の `Demo_SelectionMode` オブジェクトの **Mode** を Inspector で変更してください（Play 中も変更できます）
- ラベルの見方：各キャラクターの上に2行のラベルが表示されます。上の行は、現在のモードと、いま見ている相手の名前です（例: `Nearest: A2`、`Priority: B2`）。相手の名前は、その球と同じ系統の色で表示されます。下の行は、予測追従の状態です（`Prediction: OFF` / `Prediction: ON`）
- 予測追従：左右どちらのデモも、4.5秒ごとに予測追従の OFF / ON が自動で切り替わります（開始時は OFF）。ON の間は、いま見ている相手に対して実際に狙っている先読み位置（Predicted Position）を示す小さなマゼンタ色のマーカーが表示されます

> Note: デモでのモードの一括切り替え・ラベルの表示・予測の OFF / ON の自動切り替えは、デモ専用のスクリプト（`Demo/Scripts/` の `DemoSelectionModeSwitcher`・`DemoSelectionLabel`・`DemoPredictionToggle`）が行っています。ご自身のシーンでは、これらのスクリプトは不要です。`LogicDevLookAtTargetSelector` の **SelectionMode** と、LookAt コンポーネントの **Enable Prediction** を直接設定してください。

> Note: デモシーンはプリミティブ形状で構成されています。Starter Assets を使用したイメージはストアページで確認できます（本パッケージには同梱されていません）。

---

## 導入前の注意（重要）

本パッケージを導入する前に、旧バージョン（Move_LookAtSmooth_001、Move_LookAtSmooth_002、Move_LookAtSmooth_003）をすべてプロジェクトから削除してください。削除せずにインポートすると、共有されているスクリプトやシーンの中身が上書きされ、フォルダ名と実際の中身が一致しなくなることがあります。また、旧バージョンによってはコンパイルエラーになります。

旧バージョンのフォルダ内のファイル（デモシーン等）を編集して使っている場合は、削除する前にバックアップを取ってください。

---

## 003からのアップグレードについて

Move_LookAtSmooth_003 をお使いの方向けの注意点です。

- 上記「導入前の注意」の通り、003 をプロジェクトから削除してから本パッケージをインポートしてください
- 003 のフォルダを削除してから本パッケージをインポートすることで、既存のシーン・プレハブに設定済みの各パラメータ（RotationSpeed、角度クランプ、予測追従の設定等）はそのまま引き継がれます（Unity 2022.3 で、全身回転版・IK 首振り版・Prefab に既定値と異なる値を設定して検証済み。Missing Script も発生しません）
- `LogicDevLookAtTargetSelector` を追加しない限り、全身回転版と、非 Humanoid の IK 首振り版の動きは 003 と変わりません（同梱デモの動きを 003 と比較し、全フレームで回転が一致することを確認しています）。ただし次の1点だけ変わります: ClampAngleY が有効なとき、相手を別のものに替えると（`SetTarget()` でも、Target フィールドへの直接代入でも）、角度クランプの端の固定が解除されます（003 までは、相手が範囲内に入るまで元の端に固定されたままでした。下記「挙動」の「角度クランプの端」参照）

**Humanoid の IK 首振り版は、003 から次の3点で動きが変わります。**

- **RotationSpeed で補間するようになりました**。003 までは、Humanoid の場合 RotationSpeed が使われず、首は毎フレームちょうど相手の方向を向いていました（相手を切り替えると1フレームで向きが変わっていました）。本バージョンでは、非 Humanoid と同じく RotationSpeed に従って滑らかに向きます。デフォルト値 5 の場合、相手が切り替わってから向きの9割まで回るのに約0.3〜0.5秒かかります。首の追従を速くしたい場合は RotationSpeed を大きくしてください（最大 20）
- **Target が null の間と `Pause()` 中は、最後の向きのまま止まるようになりました**。003 までは、1フレームでアニメーションの姿勢に戻っていました。アニメーションの姿勢に戻したい場合は、IKWeight を 0 にするか、コンポーネントを無効にしてください
- **予測追従を有効にしている場合**、003 までは補間による遅れが無かったため、先読みのぶんだけ相手より先を向いていました。本バージョンでは、先読みが RotationSpeed による遅れを打ち消す形になります（当方の測定では、RotationSpeed=5・Prediction Time=0.25 で、相手の方向との差は平均 0.1〜2.4°。相手より少し先を向きます）。003 で Humanoid 用に Prediction Time を調整していた場合は、再調整してください

**無料版（Move_LookAtSmooth_001）から入れ替える場合**

- 全身回転版は、本バージョンでは左右（Y 軸）だけを追います。001 の上下回転（EnableXAxis）と、上下の角度制限（ClampAngleX・MinAngleX・MaxAngleX）の設定は引き継がれません。001 で上下回転を ON にして保存したシーン・プレハブでも、上下には回転しません（Inspector にもこれらの項目は表示されません）
- 上下にも向かせたい場合は、IK 首振り版（頭だけを上下左右に向けます）をお使いください

---

## 上位版について

上位版（`Move_LookAtSmooth_005`）を開発予定です。お楽しみに！

無料版（`Move_LookAtSmooth_001`）は以下からご利用いただけます。

https://assetstore.unity.com/packages/slug/389610

---

もしこのアセットが役に立ちましたら、ぜひレビューを書いて応援していただけると開発の励みになります！

以下のストアページの「Reviews」タブから投稿できます。

https://assetstore.unity.com/packages/slug/417124
