# LogicDevLookAtSmooth - スムーズ注視コンポーネント

対象オブジェクトをスムーズに注視し続ける LookAt コンポーネントです。全身回転版（`LogicDevLookAtSmooth`）と IK 首振り版（`LogicDevLookAtSmoothIK`）の2種類がセットで付属します。

---

## 特徴

- **2種類のLookAtが同梱** — 全身 Y 軸回転と Humanoid IK 首振りを使い分けられる
- **乗り物・ドローンの傾きに自動追従** — 取り付けるだけで、物理演算やアニメーションでオブジェクト自身（親を含む）が傾いた場合でも、正しい「上」方向を保ったまま Target を追従します（設定不要）
- **Inspector でリアルタイム調整** — パラメータをプレイ中に即座に確認・変更可能
- **Humanoid Animator に自動対応** — `OnAnimatorIK` を使用し、通常の Animator でも自然に機能
- **角度クランプ対応** — 全身回転版は Y 軸（左右）、IK 首振り版は X・Y 軸ごとに最小/最大角度を設定可能
- **Update タイミング選択** — `Update` / `LateUpdate` / `FixedUpdate` から選択可能

---

## 動作環境

| 項目 | 内容 |
|------|------|
| Unity バージョン | Unity 2022.3 LTS / Unity 6.3 LTS |
| レンダリングパイプライン | URP (Universal Render Pipeline) |
| 入力システム | New Input System（依存なし） |

---

## 使い方

### LogicDevLookAtSmooth（全身回転版）

1. 注視させたい GameObject に `LogicDevLookAtSmooth` コンポーネントを追加します
2. Inspector の **Target** フィールドに注視対象の Transform を設定します
3. 必要に応じて **RotationSpeed**、**EnableYAxis**、角度クランプ を調整します（乗り物・ドローン等の傾きには自動追従するため、通常は追加設定不要です）
4. Play ボタンを押すとスムーズに Target を追従します

### LogicDevLookAtSmoothIK（IK首振り版）

1. Humanoid Animator を持つキャラクターの Root GameObject に `LogicDevLookAtSmoothIK` を追加します
2. Inspector の **Target** フィールドに注視対象を設定します
3. **非 Humanoid の場合** は **HeadBone** に回転させたいボーンの Transform を設定します
4. **IKWeight** で首振りの強さ（0〜1）を調整できます

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
| Target | Transform | null | 注視対象。null のとき動作停止 |
| RotationSpeed | float | 5.0 | 追従速度（Slerp 係数） |
| EnableYAxis | bool | true | Y 軸（左右）回転の有効/無効 |
| ClampAngleY | bool | false | Y 軸角度制限の有効/無効 |
| MinAngleY | float | -90 | Y 軸最小角度（度）。ClampAngleY 有効時のみ |
| MaxAngleY | float | 90 | Y 軸最大角度（度）。ClampAngleY 有効時のみ |
| UpAxis | Vector3 | (0, 1, 0) | ローカル空間での「上」方向。**Inspector には非表示**（スクリプトからのみ設定可能な上級者向け項目。下記 Note 参照） |
| ReferenceForward | Vector3 | (0, 0, 0) | Yaw=0の基準となるワールド方向。(0,0,0)（未設定）なら配置時のtransform.forwardを自動使用。非ゼロの値を明示的に設定するとその方向を基準にできます。実行時に動的に変更する場合はコードからSetReferenceForward()を呼び出してください(フィールドへの直接代入はAwake前またはInspector編集時のみ反映されます) |
| UpdateMode | enum | LateUpdate | Update / LateUpdate / FixedUpdate |

> **Note:** UpAxis は取り付けるだけで自動的にオブジェクト自身の傾きに追従するため、通常は変更不要です。見た目が回転して見える不具合を避けるため、Inspector からは意図的に非表示にしています。スクリプトから値を変更すること自体は可能ですが、デフォルト値 `(0, 1, 0)` 以外を手動設定すると、その軸を中心に Yaw 回転が生じ、見た目が回転しているように見える点にご注意ください。

### LogicDevLookAtSmoothIK（IK首振り版）

| パラメータ | 型 | デフォルト | 説明 |
|---|---|---|---|
| Target | Transform | null | 注視対象。null のとき動作停止 |
| HeadBone | Transform | null | 回転させる頭部 Transform（非 Humanoid 時に使用） |
| RotationSpeed | float | 5.0 | 追従速度（Slerp 係数） |
| ClampAngleX | bool | true | X 軸角度制限の有効/無効 |
| MinAngleX | float | -30 | X 軸最小角度（度） |
| MaxAngleX | float | 30 | X 軸最大角度（度） |
| ClampAngleY | bool | true | Y 軸角度制限の有効/無効 |
| MinAngleY | float | -60 | Y 軸最小角度（度） |
| MaxAngleY | float | 60 | Y 軸最大角度（度） |
| IKWeight | float | 1.0 | Humanoid Animator 使用時の IK 強度（0〜1） |
| UpdateMode | enum | LateUpdate | Update / LateUpdate / FixedUpdate |

> **Note:** Humanoidキャラクターに使用する場合、非常に広い角度範囲（180°に近いMin/Max設定）を指定しても、キャラクター自身の首・体幹の可動域（Muscle定義）により、設定値まで実際には追従しきれないことがあります。頭部・体幹のIKのみで、体の向きを変えずに真後ろを自然に見せることは、Unity標準のHumanoid LookAt IKの仕組み上、構造的に難しい点にご注意ください。通常の使用範囲（正面〜横程度）では問題なく機能します。

---

## デモシーンについて

同梱のデモシーン `Demo_LookAtSmooth`（`Assets/LogicDevSupport/Move_LookAtSmooth_002/Demo/`）で動作をすぐに確認できます。

- 左側：全身回転デモ（青いカプセルが黄色い球を追う。傾く台の上に乗せることで、乗り物・ドローンの傾きへの自動追従も確認できます）
- 右側：IK 首振りデモ（キャラクターが頭だけで赤い球を追う）

> **Note:** デモシーンはプリミティブ形状で構成されています。Starter Assets を使用したイメージはストアページで確認できます（本パッケージには同梱されていません）。

---

※ 本パッケージを導入する前に、旧バージョン（Move_LookAtSmooth_001）をプロジェクトから削除してからインポートしてください（同じクラス名を使用しているため、両方を同時に導入するとコンパイルエラーになります）。

---

## 上位版について

上位版（`Move_LookAtSmooth_003`）を開発予定です。予測追従（ターゲットの速度から先読みして追従）などの機能追加を予定しています。お楽しみに！

---

もしこのアセットが役に立ちましたら、ぜひレビューを書いて応援していただけると開発の励みになります！
