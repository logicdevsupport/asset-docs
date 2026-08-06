# LogicDevLookAtSmooth - スムーズ注視コンポーネント

対象オブジェクトをスムーズに注視し続ける LookAt コンポーネントです。全身回転版（`LogicDevLookAtSmooth`）と IK 首振り版（`LogicDevLookAtSmoothIK`）の2種類がセットで付属します。

---

## 特徴

- **2種類のLookAtが同梱** — 全身 Y 軸回転と Humanoid IK 首振りを使い分けられる
- **Inspector でリアルタイム調整** — パラメータをプレイ中に即座に確認・変更可能
- **Humanoid Animator に自動対応** — `OnAnimatorIK` を使用し、通常の Animator でも自然に機能
- **軸制限・角度クランプ対応** — X 軸・Y 軸ごとに最小/最大角度を設定可能
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
3. 必要に応じて **RotationSpeed**、**EnableXAxis**、**EnableYAxis**、角度クランプを調整します
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
| EnableXAxis | bool | true | X 軸（上下）回転の有効/無効 |
| EnableYAxis | bool | true | Y 軸（左右）回転の有効/無効 |
| ClampAngleX | bool | false | X 軸角度制限の有効/無効 |
| MinAngleX | float | -60 | X 軸最小角度（度）。ClampAngleX 有効時のみ |
| MaxAngleX | float | 60 | X 軸最大角度（度）。ClampAngleX 有効時のみ |
| ClampAngleY | bool | false | Y 軸角度制限の有効/無効 |
| MinAngleY | float | -90 | Y 軸最小角度（度）。ClampAngleY 有効時のみ |
| MaxAngleY | float | 90 | Y 軸最大角度（度）。ClampAngleY 有効時のみ |
| UpdateMode | enum | LateUpdate | Update / LateUpdate / FixedUpdate |

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

---

## デモシーンについて

同梱のデモシーン `Demo_LookAtSmooth`（`Assets/LogicDevSupport/Move_LookAtSmooth_001/Demo/`）で動作をすぐに確認できます。

- 左側：全身回転デモ（青いカプセルが黄色い球を追う）
- 右側：IK 首振りデモ（キャラクターが頭だけで赤い球を追う）

> **Note:** デモシーンはプリミティブ形状で構成されています。Starter Assets を使用したデモ動画はストアページで確認できます（本パッケージには同梱されていません）。

---

## 上位版について

上位版（`Move_LookAtSmooth_002`）を開発予定です。UpAxis 対応、イージングカーブ、複数ターゲット管理などの機能追加を予定しています。お楽しみに！

---

もしこのアセットが役に立ちましたら、ぜひレビューを書いて応援していただけると開発の励みになります！
