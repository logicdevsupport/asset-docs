# LogicDevLookAtSmooth - スムーズ注視コンポーネント

対象オブジェクトをスムーズに注視し続ける LookAt コンポーネントです。全身回転版（`LogicDevLookAtSmooth`）と IK 首振り版（`LogicDevLookAtSmoothIK`）の2種類がセットで付属します。

---

## 特徴

- **2種類のLookAtが同梱** — 全身 Y 軸回転と Humanoid IK 首振りを使い分けられる
- **Inspector でリアルタイム調整** — パラメータをプレイ中に即座に確認・変更可能
- **Humanoid Animator を自動判別** — Humanoid の場合は `OnAnimatorIK` で頭を向けます（Animator Controller の IK Pass を有効にする必要があります。下記「使い方」参照）。非 Humanoid の場合は指定したボーンを直接回転させます
- **軸制限・角度クランプ対応** — X 軸・Y 軸ごとに最小/最大角度を設定可能
- **Update タイミング選択** — `Update` / `LateUpdate` / `FixedUpdate` から選択可能

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
3. 必要に応じて **RotationSpeed**、**EnableXAxis**、**EnableYAxis**、角度クランプを調整します
4. Play ボタンを押すとスムーズに Target を追従します

### LogicDevLookAtSmoothIK（IK首振り版）

**Humanoid の場合**

1. Humanoid の Animator が付いている GameObject（通常はキャラクターのルート）に `LogicDevLookAtSmoothIK` を追加します。**Animator と同じ GameObject に付けてください**。Unity は `OnAnimatorIK` を Animator と同じ GameObject 上のスクリプトにしか呼び出さないため、親や子の GameObject に付けると首は動きません（この場合、開始時に Console へ Warning が1回表示されます）
2. **Animator Controller のレイヤー設定で「IK Pass」を有効にします**（Animator ウィンドウ > Layers > 対象レイヤーの歯車アイコン > IK Pass にチェック）。Unity は IK Pass が有効なレイヤーに対してだけ `OnAnimatorIK` を呼び出すため、無効のままだと頭は動きません（本コンポーネントは IK Pass の設定を検査しないため、警告は表示されません）
3. Inspector の **Target** フィールドに注視対象を設定します
4. **RotationSpeed**（追従の速さ）と **IKWeight**（首振りの強さ、0〜1）を必要に応じて調整します

> **Note:** Animator の Culling Mode が「Always Animate」以外の場合、キャラクターが画面に映っていない間は Unity が Animator の IK 処理を呼び出さないため、その間は首の向きが更新されません。

**非 Humanoid の場合**

1. キャラクターの GameObject に `LogicDevLookAtSmoothIK` を追加します
2. **HeadBone** に回転させたいボーン（頭の Transform）を設定します
3. Inspector の **Target** フィールドに注視対象を設定します

### コードからの操作例

```csharp
// ターゲットをコードから変更する
GetComponent<LogicDevLookAtSmooth>().SetTarget(newTarget);

// 注視を一時停止
GetComponent<LogicDevLookAtSmooth>().Pause();

// 注視を再開
GetComponent<LogicDevLookAtSmooth>().Resume();
```

IK 版も同じ API を使用できます。

---

## パラメータ一覧

### LogicDevLookAtSmooth（全身回転版）

| パラメータ | 型 | デフォルト | 説明 |
|---|---|---|---|
| Target | Transform | null | 注視対象。null のときは回転を止め、そのときの向きを保ちます |
| RotationSpeed | float | 5.0 | 追従速度（Slerp 係数） |
| EnableXAxis | bool | true | X 軸（上下）回転の有効/無効 |
| EnableYAxis | bool | true | Y 軸（左右）回転の有効/無効 |
| ClampAngleX | bool | false | X 軸角度制限の有効/無効 |
| MinAngleX | float | -60 | X 軸最小角度（度）。ClampAngleX 有効時のみ |
| MaxAngleX | float | 60 | X 軸最大角度（度）。ClampAngleX 有効時のみ |
| ClampAngleY | bool | false | Y 軸角度制限の有効/無効 |
| MinAngleY | float | -90 | Y 軸最小角度（度）。ClampAngleY 有効時のみ |
| MaxAngleY | float | 90 | Y 軸最大角度（度）。ClampAngleY 有効時のみ |
| UpdateMode | enum | LateUpdate | Update / LateUpdate / FixedUpdate |

> **Note:** MinAngleY から MaxAngleY までの幅（MaxAngleY − MinAngleY）は、180° 以下にしてください。向きは最短の回り方で補間されるため、幅が 180° を超えると、一方の端からもう一方の端へ向きを変える途中で、範囲の外（背中側）を通ります。幅が 180° を超える設定で ClampAngleY が有効な場合、実行時に Console へ Warning を1回表示します。

> **Note:** 全身回転版は、毎フレーム Transform の回転を直接設定します。Rigidbody（非 Kinematic）や CharacterController、ほかのスクリプトが同じオブジェクトの回転を制御している場合は、回転を取り合うため、対象外です。そのようなオブジェクトでは、回転を制御する側を止めるか、見た目用の子オブジェクトに全身回転版を付けるか、IK 首振り版をお使いください。

> **Note:** ClampAngleY が有効なとき、相手が角度制限の範囲外にいる間は、最初に外れた側の端に向きを固定します。相手が別のものに替わると（`SetTarget()` でも、Target フィールドへの直接代入でも）、この固定を解除し、新しい相手に近い側の端を選び直します（同じ相手を設定し直した場合は解除しません。いったん null にしてから設定した場合は、同じ相手でも解除します）。

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

> **Note:** MinAngleY から MaxAngleY までの幅（MaxAngleY − MinAngleY）は、180° 以下にしてください（Humanoid・非 Humanoid 共通）。向きは最短の回り方で補間されるため、幅が 180° を超えると、一方の端からもう一方の端へ向きを変える途中で、範囲の外（真後ろ側）を通ります。Humanoid ではこのとき体ごと大きくねじれて見えることがあります。幅が 180° を超える設定で ClampAngleY が有効な場合、実行時に Console へ Warning を1回表示します。

> **Note:** Humanoid では、ClampAngleY を無効にした場合も同じことが起きます（範囲の制限が無いため、相手を反対側の相手へ切り替えたときなどに、狙う向きが真後ろを通り、頭と体が大きくねじれて見えることがあります）。Humanoid では ClampAngleY を有効にし、幅を 180° 以下にして使うことをおすすめします。Humanoid で ClampAngleY が無効な場合、実行時に Console へ Warning を1回表示します。

> **Note:** ClampAngleY が有効なとき、相手が角度制限の範囲外にいる間は、最初に外れた側の端に向きを固定します。相手が別のものに替わると（`SetTarget()` でも、Target フィールドへの直接代入でも）、この固定を解除し、新しい相手に近い側の端を選び直します（同じ相手を設定し直した場合は解除しません。いったん null にしてから設定した場合は、同じ相手でも解除します）。

> **Note:** Humanoid の場合、`Pause()` 中と Target が null の間は、首は最後の向きのまま止まります（アニメーションの姿勢には戻りません）。アニメーションの姿勢に戻したい場合は、IKWeight を 0 にするか、コンポーネントを無効（enabled = false）にしてください。どちらの場合も、首は1フレームでアニメーションの姿勢に戻ります（滑らかに戻したい場合は、IKWeight をスクリプトから少しずつ変えてください）。再度有効にすると、キャラクターの正面の向きから相手へ RotationSpeed で向き直ります。

---

## デモシーンについて

同梱のデモシーン `Demo_LookAtSmooth`（`Assets/LogicDevSupport/Move_LookAtSmooth_001/Demo/`）で動作をすぐに確認できます。

- 左側：全身回転デモ（青いカプセルが黄色い球を追う）
- 右側：IK 首振りデモ（キャラクターが頭だけで赤い球を追う）

> **Note:** デモシーンはプリミティブ形状で構成されています。Starter Assets を使用したイメージはストアページで確認できます（本パッケージには同梱されていません）。

---

## 上位版について

有料の上位版がUnity Asset Storeで公開されています。

- Smooth LookAt Component Plus（傾く物の上での追従と、予測追従を追加）: https://assetstore.unity.com/packages/slug/404560
- Smooth LookAt Component Complete Edition（シリーズの最終版。Plus の全機能に、複数の相手の自動切り替え・視野の制限・首振りのフェード・ギズモ・開発用の道具を追加）: https://assetstore.unity.com/packages/slug/410220

動画: https://www.youtube.com/@LogicDevSupport

※ 上位版を導入する場合は、導入前にプロジェクトからこの無料版（`Move_LookAtSmooth_001`）を削除してください（同じクラス名を使用しているため、両方を同時に導入するとコンパイルエラーになります）。

---

もしこのアセットが役に立ちましたら、ぜひレビューを書いて応援していただけると開発の励みになります！

以下のストアページの「Reviews」タブから投稿できます。

https://assetstore.unity.com/packages/tools/utilities/smooth-lookat-component-389610
