# LogicDevLookAtSmooth - スムーズ注視コンポーネント Plus

対象オブジェクトをスムーズに注視し続ける LookAt コンポーネントです。全身回転版（`LogicDevLookAtSmooth`）と IK 首振り版（`LogicDevLookAtSmoothIK`）の2種類がセットで付属します。本バージョン（Plus）では、ターゲットの速度を自動推定して未来位置を先読みする「予測追従（Predictive LookAt）」機能を追加しました。

---

## 特徴

- **2種類のLookAtが同梱** - 全身 Y 軸回転と Humanoid IK 首振りを使い分けられる
- **予測追従（Predictive LookAt）に対応（新機能）** - ターゲットの速度を自動推定し、未来位置を先読みして追従します。有効化するだけで、動くターゲットに対する追従の遅れを軽減できます。無効時（デフォルト）は、狙う位置に常にTarget.positionがそのまま使われ、最終的な追従計算（Yaw/Pitchの算出）は予測機能のない旧バージョンと同じになります（ターゲット参照の切り替えを検知する軽量な内部処理自体は、無効時も動作します）
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
| 入力システム | New Input System（依存なし） |

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

### 予測追従（Predictive LookAt）を使う（新機能）

1. Inspector の **Enable Prediction** をチェックします（デフォルトはOFF。OFFのままなら従来通りの挙動です）
2. **Prediction Time**（先読み秒数）・**Velocity Smooth Time**（速度推定の平滑化時定数）・**Max Prediction Distance**（先読みオフセットの上限）を必要に応じて調整します。各項目の詳細は下記「パラメータ一覧」を参照してください
3. ターゲットの速度は自動推定されます。内部的には、Target に Rigidbody（Kinematic ではないもの）が付いていればその速度を、付いていない場合は位置の差分から速度を推定します。どちらの方式を使うかを指定する設定項目はありません（自動判定）
4. `SetTarget()` の呼び出しや、Target フィールドへの直接代入でターゲットを切り替えた場合、速度推定の内部状態は自動的にリセットされます（旧ターゲットの速度推定値が新しいターゲットに引き継がれることはありません）
5. 動きの速いターゲットほど、予測の効果を確認しやすくなります
6. **相手の先を狙う（偏差撃ちのような）使い方:** **RotationSpeed** を上げて向きが狙う点に追いつくようにし、**Prediction Time** を先読みしたい秒数（例: 弾が相手に届くまでの時間）にします。確認済みの目安: RotationSpeed=20（スライダーの上限）・Prediction Time=0.25 で、約3m前方を左右に往復するターゲットに対し、向きと Predicted Position の差は、全身回転版（平らな床）で平均約3°（最大約5°）、Humanoid の IK首振り版（ターゲットは上下にも移動）で平均約2.5°（最大約3.5°）でした。追従の遅れを打ち消すだけの使い方は、下記「パラメータ一覧」の Prediction Time を参照してください。先読みは、ターゲットの現在の速度による直線的な予測です（下記「仕様上の限界」参照）

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

> Note: 全身回転版は、左右（Y 軸）だけを追います。上下には回転しません。無料版（Move_LookAtSmooth_001）にある上下回転（EnableXAxis）の設定は引き継がれません。001 で上下回転を ON にして保存したシーン・プレハブでも、本バージョンでは左右だけを追います。上下にも向かせたい場合は、IK 首振り版（頭だけを上下左右に向けます）をお使いください。

> Note: MinAngleY から MaxAngleY までの幅（MaxAngleY − MinAngleY）は、180° 以下にしてください。向きは最短の回り方で補間されるため、幅が 180° を超えると、一方の端からもう一方の端へ向きを変える途中で、範囲の外（背中側）を通ります。幅が 180° を超える設定で ClampAngleY が有効な場合、実行時に Console へ Warning を1回表示します。

> Note: 全身回転版は、毎フレーム Transform の回転を直接設定します。Rigidbody（非 Kinematic）や CharacterController、ほかのスクリプトが同じオブジェクトの回転を制御している場合は、回転を取り合うため、対象外です。そのようなオブジェクトでは、回転を制御する側を止めるか、見た目用の子オブジェクトに全身回転版を付けるか、IK 首振り版をお使いください。

> Note: ClampAngleY が有効なとき、相手が角度制限の範囲外にいる間は、最初に外れた側の端に向きを固定します。相手が別のものに替わると（`SetTarget()` でも、Target フィールドへの直接代入でも）、この固定を解除し、新しい相手に近い側の端を選び直します（同じ相手を設定し直した場合は解除しません。いったん null にしてから設定した場合は、同じ相手でも解除します）。

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

> Note: MinAngleY から MaxAngleY までの幅（MaxAngleY − MinAngleY）は、180° 以下にしてください（Humanoid・非 Humanoid 共通）。向きは最短の回り方で補間されるため、幅が 180° を超えると、一方の端からもう一方の端へ向きを変える途中で、範囲の外（真後ろ側）を通ります。Humanoid ではこのとき体ごと大きくねじれて見えることがあります。幅が 180° を超える設定で ClampAngleY が有効な場合、実行時に Console へ Warning を1回表示します。

> Note: Humanoid では、ClampAngleY を無効にした場合も同じことが起きます（範囲の制限が無いため、相手を反対側の相手へ切り替えたときなどに、狙う向きが真後ろを通り、頭と体が大きくねじれて見えることがあります）。Humanoid では ClampAngleY を有効にし、幅を 180° 以下にして使うことをおすすめします。Humanoid で ClampAngleY が無効な場合、実行時に Console へ Warning を1回表示します。

> Note: ClampAngleY が有効なとき、相手が角度制限の範囲外にいる間は、最初に外れた側の端に向きを固定します。相手が別のものに替わると（`SetTarget()` でも、Target フィールドへの直接代入でも）、この固定を解除し、新しい相手に近い側の端を選び直します（同じ相手を設定し直した場合は解除しません。いったん null にしてから設定した場合は、同じ相手でも解除します）。

> Note: Humanoid の場合、`Pause()` 中と Target が null の間は、首は最後の向きのまま止まります（アニメーションの姿勢には戻りません）。アニメーションの姿勢に戻したい場合は、IKWeight を 0 にするか、コンポーネントを無効（enabled = false）にしてください。どちらの場合も、首は1フレームでアニメーションの姿勢に戻ります（滑らかに戻したい場合は、IKWeight をスクリプトから少しずつ変えてください）。再度有効にすると、キャラクターの正面の向きから相手へ RotationSpeed で向き直ります。

> Note: Humanoidキャラクターに使用する場合、非常に広い角度範囲（180°に近いMin/Max設定）を指定しても、キャラクター自身の首・体幹の可動域（Muscle定義）により、設定値まで実際には追従しきれないことがあります。頭部・体幹のIKのみで、体の向きを変えずに真後ろを自然に見せることは、Unity標準のHumanoid LookAt IKの仕組み上、構造的に難しい点にご注意ください。通常の使用範囲（正面〜横程度）では問題なく機能します。

### 予測追従（Predictive LookAt）共通パラメータ（新機能、両コンポーネント共通）

全身回転版・IK首振り版のどちらでも、以下の同じパラメータが使用できます。

| パラメータ | 型 | デフォルト | 説明 |
|---|---|---|---|
| Enable Prediction | bool | false | 予測追従の有効/無効。falseの場合、狙う位置は常にTarget.positionがそのまま使われ、最終的な追従計算は予測機能のない旧バージョンと同じになります（ターゲット参照の変化を検知する軽量な内部処理は、無効時も動作します） |
| Prediction Time | float | 0.25 | 先読み秒数。実際に狙う位置は「現在のターゲット位置 + 推定速度 × Prediction Time」になります。**確認済みの目安**: RotationSpeed=5、Velocity Smooth Time=0.1 の設定で、全身回転版（円運動するターゲット）では、Prediction Time が 0.25 付近のとき追従の遅れがほぼ消えることを確認しています（相手の方向との差は 1° 未満）。Humanoid の IK 首振り版（左右に往復するターゲット、2種類の動き）では、予測 OFF で平均 7〜10° あった遅れが、Prediction Time 0.25 で平均 0.1〜2.4°（相手より少し先）になりました。他の RotationSpeed の値については未検証です。**RotationSpeed を変更した場合は、下記 Predicted Position を使って実際に狙っている位置を確認しながら、Prediction Time を再調整してください**。調整の方向の目安: Prediction Time は RotationSpeed による追従の遅れを打ち消すための値なので、**RotationSpeed を大きく（追従を速く）した場合は Prediction Time を小さく、RotationSpeed を小さく（追従を遅く）した場合は Prediction Time を大きく**する方向で調整してください |
| Velocity Smooth Time | float | 0.1 | 速度推定の平滑化にかかる時定数（秒）。値を大きくするほどノイズに強くなりますが、実際の速度変化への反応は遅くなります。0以下を指定すると平滑化を行いません |
| Max Prediction Distance | float | 10 | 先読みオフセット（推定速度 × Prediction Time）の上限距離（ワールド単位）。**この値には別の役割もあります**: ターゲットの位置が1ステップでこの距離を超えて移動した場合、瞬間移動（テレポート）と判断し、速度推定を一度リセットします（推定値0から再度収束します）。**この値を小さくしすぎると、実際には正当な高速移動をしているターゲットが誤ってテレポートと判定されることがあります**のでご注意ください。0以下を指定すると、先読みオフセットの上限とテレポート判定のどちらも無効になります |
| Estimated Velocity | Vector3（読み取り専用） | - | 現在推定されているターゲットの速度。検証・デバッグ用に公開されています |
| Predicted Position | Vector3（読み取り専用） | - | 実際に狙っている先読み位置。検証・デバッグ用に公開されています |

---

## 仕様上の限界（新機能について）

- 予測追従は、ターゲットの**現在の速度**に基づく単純な線形の先読みです。ターゲットが加速・減速している場合や、旋回中（速度の向きが変化している場合）は、先読み位置に誤差が生じます
- 弾道計算や、発射物の速度を考慮した迎撃位置の計算は含まれていません
- 全身回転版（`LogicDevLookAtSmooth`）・IK首振り版（`LogicDevLookAtSmoothIK`）のどちらでも、上記の予測追従パラメータは全く同じ形で使用できます
- Rigidbody を持たず、FixedUpdate で位置を更新しているターゲットの場合、描画フレームと物理更新のタイミングのずれにより、Estimated Velocity（推定速度）が揺れることがあります（当方の測定では、真値に対して最大で約10%）。対策として、(1) ターゲットに非Kinematic な Rigidbody を付与し、物理演算（または velocity への直接代入）で動かす（Rigidbody の速度を直接参照するようになり、この揺れを避けられます）、(2) Velocity Smooth Time を大きめに設定する（平滑化を強める代わりに、速度変化への反応は遅くなります）、のいずれかが有効です

---

## デモシーンについて

同梱のデモシーン `Demo_LookAtSmooth`（`Assets/LogicDevSupport/Move_LookAtSmooth_002/Demo/`）で動作をすぐに確認できます。

- 左側：全身回転デモ（青いカプセルが黄色い球を追う。傾く台の上に乗せることで、乗り物・ドローンの傾きへの自動追従も確認できます）
- 右側：IK 首振りデモ（キャラクターが頭だけで赤い球を追う。赤い球は左右に加えて上下にも動くため、頭の上下方向の追従も確認できます）
- 予測追従（新機能）：左右どちらのデモも、4.5秒ごとに予測追従の OFF / ON が自動で切り替わります（開始時は OFF）。各キャラクターの上に現在の状態（`Prediction: OFF` / `Prediction: ON`）が表示され、ON の間は、実際に狙っている先読み位置（Predicted Position）を示す小さなマゼンタ色のマーカーが表示されます。同じキャラクター・同じターゲットのまま OFF と ON を見比べることで、予測追従による追従の遅れの違いを確認できます

> Note: デモでの OFF / ON の自動切り替えは、デモ専用のスクリプト `DemoPredictionToggle`（`Demo/Scripts/`）が行っています。ご自身のシーンで予測追従を使う場合、このスクリプトは不要です。コンポーネントの **Enable Prediction** を直接設定してください。

> Note: デモシーンはプリミティブ形状で構成されています。Starter Assets を使用したイメージはストアページで確認できます（本パッケージには同梱されていません）。

---

## 導入前の注意（重要）

無料版（`Move_LookAtSmooth_001`）がプロジェクトに入っている場合は、本パッケージをインポートする前に削除してください。同じクラス名を使用しているため、両方を同時に導入するとコンパイルエラー（クラスの重複定義）になります。

**本パッケージの更新について**: 本パッケージの以前のリリース（Smooth LookAt Component Standard として販売していたもの。フォルダ `Move_LookAtSmooth_002`）をお持ちの場合は、何も削除する必要はありません。Package Manager（My Assets）から更新してください。同じ `Move_LookAtSmooth_002` フォルダの中のファイルが置き換わり、シーン・プレハブのコンポーネントは設定値ごとそのまま動きます。

パッケージのフォルダ内のファイル（デモシーン等）を編集して使っている場合は、更新・削除の前にバックアップを取ってください。

---

## Smooth LookAt Component Standard からの更新について

本パッケージは、以前は Smooth LookAt Component Standard として販売していました。現在は Smooth LookAt Component Plus です。Standard の全機能に、予測追従（Predictive LookAt）を加えたものです。

- 予測追従はデフォルトで OFF のため、Enable Prediction を明示的に ON にしない限り、これまでと同じように動作します
- 既存のシーン・プレハブに設定済みの各パラメータ（RotationSpeed、ClampAngleY、UpAxis 等）は、更新してもそのまま引き継がれます。新しい予測追従の設定は、ドキュメント記載の初期値で始まります

---

## 上位版について

シリーズの最終版で、最も機能の多い **Smooth LookAt Component Complete Edition** が Unity Asset Store で公開されています。本バージョンの全機能に加えて、複数の候補の中から注視する相手（一番近い相手・優先度の高い相手・正面に一番近い相手）を自動で選んで切り替える機能、視野の外や壁の向こうの相手を除く機能、首振りのフェードイン・フェードアウト、シーンビューの補助表示（ギズモ）と開発用の道具（ゲーム画面のデバッグ表示・実行中の調整パネル）が入っています。

https://assetstore.unity.com/packages/slug/410220

無料版（`Move_LookAtSmooth_001`）は以下からご利用いただけます。

https://assetstore.unity.com/packages/slug/389610

動画: 紹介動画 https://youtu.be/eowEC9rYavM ／ ショート https://youtu.be/1GorMFFqxgQ ／ チャンネル https://www.youtube.com/@LogicDevSupport

---

もしこのアセットが役に立ちましたら、ぜひレビューを書いて応援していただけると開発の励みになります！

以下のストアページの「Reviews」タブから投稿できます。

https://assetstore.unity.com/packages/slug/404560
