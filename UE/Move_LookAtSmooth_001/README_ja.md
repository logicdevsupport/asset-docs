# LogicDevLookAtSmooth (UE) - スムーズ注視コンポーネント

対象アクターをスムーズに注視し続ける LookAt コンポーネントです。全身回転版（`ULogicDevLookAtSmoothComponent`）と IK 首振り版（`ULogicDevLookAtSmoothIKComponent`）の2種類がセットで付属します。

---

## 特徴

- **2種類のLookAtが同梱** — 全身回転と頭部（IK）首振りを使い分けられる
- **C++ / Blueprint 両対応** — `UFUNCTION(BlueprintCallable)` によりBlueprintからも呼び出し可能
- **Details パネルでリアルタイム調整** — パラメータをPlay中に即座に確認・変更可能
- **3種類のメッシュ構成に自動対応（IK版）** — PoseableMeshComponent / SkeletalMeshComponent / 通常のSceneComponent階層のいずれでも動作
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
3. **HeadBoneName**（デフォルト `"head"`）に、回転させたいボーン名（PoseableMesh / SkeletalMesh の場合）または子 SceneComponent の名前を指定します
4. **IKWeight** で首振りの強さ（0〜1）を調整できます

> BeginPlay時にオーナーActorの構成を自動判別します：`UPoseableMeshComponent` があればボーン単位のフル制御、`USkeletalMeshComponent` のみの場合はベストエフォートでボーンを上書き、どちらもない場合は `HeadBoneName` と同名の `USceneComponent`（キューブ階層構成など）を回転させます。

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

### ULogicDevLookAtSmoothIKComponent（IK首振り版）

| パラメータ | 型 | デフォルト | 説明 |
|---|---|---|---|
| `Target` | `AActor*` | `nullptr` | 注視対象。nullptrのとき動作停止 |
| `HeadBoneName` | `FName` | `"head"` | 回転させるボーン名（PoseableMesh/SkeletalMesh）または子SceneComponent名 |
| `RotationSpeed` | `float` | `12.0`（範囲 0.0〜20.0） | 追従速度（RInterpTo係数） |
| `bClampAngleX` | `bool` | `true` | X軸角度制限の有効/無効 |
| `MinAngleX` | `float` | `-30.0`（範囲 -180〜0） | X軸最小角度（度） |
| `MaxAngleX` | `float` | `30.0`（範囲 0〜180） | X軸最大角度（度） |
| `bClampAngleY` | `bool` | `true` | Y軸角度制限の有効/無効 |
| `MinAngleY` | `float` | `-80.0`（範囲 -180〜0） | Y軸最小角度（度） |
| `MaxAngleY` | `float` | `150.0`（範囲 0〜180） | Y軸最大角度（度） |
| `IKWeight` | `float` | `1.0`（範囲 0〜1） | IK適用強度（0=無効、1=フル） |

---

## デモシーンについて

同梱のデモレベル `L_LookAtSmooth_Test`（`Content/LogicDevLookAtSmooth/Demo/`）で動作をすぐに確認できます。

- 左側：全身回転デモ（`Watcher_Body` が黄色い球を追う）
- 右側：IK首振りデモ（`IK_Character` が頭だけで赤い球を追う）

> **Note:** デモシーンはプリミティブ形状で構成されています。Mannequinを使用したイメージはストアページで確認できます（本パッケージには同梱されていません）。

---

## 上位版について

上位版（`Move_LookAtSmooth_002`）を開発予定です。お楽しみに！

---

もしこのアセットが役に立ちましたら、ぜひレビューを書いて応援していただけると開発の励みになります！
