# Smart Perception Lite - 敵の視覚と警戒度

敵や NPC に付ける「気づく」部品です（無料版）。敵に `PerceptionSensor`、プレイヤーに `PerceptionTarget` を付けるだけで、敵が見て（視野・距離・壁・体の複数の点）、警戒度のメーターを上げます（Unaware → Suspicious → Alert）。気づいたことはプロパティとイベントで AI に伝えます。敵を動かすこと（追跡・巡回・攻撃）はこのパッケージに含みません。あなたの AI がセンサーを読んで、何をするかを決めます。

---

## 特徴

- **体の一部が隠れた状態の視覚** - ターゲットごとに複数の点（既定は足・胸・頭）を調べます。低い壁の陰のプレイヤーは一部が見えていて、見えている量に応じてメーターが上がります
- **中心視野と周辺視野・距離** - 中心の角度の中では最も速く、視野の端や見える距離の端ほどゆっくりメーターが上がります
- **3段階の警戒度メーター** - Unaware → Suspicious → Alert。保持時間・下がり方・しきい値の余裕があり、しきい値の付近で状態がちらつきません
- **イベント** - 状態の変化、見つけた／見失った、を C# のイベントと UnityEvent で
- **シーンビューのギズモ** - 視野（視野角・中心視野・距離）、各点への線（緑 = 見える、赤 = 遮られた）、状態とメーター
- **敵の集団** - ほかの敵の体は既定では視界をさえぎらないので、集まった見張りどうしでプレイヤーを隠しません

---

## 動作環境

| 項目 | 内容 |
|------|------|
| Unity バージョン | Unity 2022.3 LTS / Unity 6.3 LTS |
| レンダーパイプライン | Built-in Render Pipeline・URP・HDRP（確認済み: Unity 2022.3 URP／Built-in／HDRP、Unity 6.3 URP／Built-in／HDRP）。部品はレンダーパイプライン固有の API を使いません。同梱デモのマテリアルは、パッケージを入れたときに、使っているレンダーパイプラインの標準のシェーダーに切り替わります。HDRP では、デモの Play 中にデモのカメラが固定の露出と簡単な空を足します |
| Input System | 部品は入力の仕組みに依存しません。同梱デモのキーだけが New Input System（Input System パッケージ）を使います |
| 物理 | 3D の物理だけが対象です。壁は 3D のコライダーへのレイキャストで調べます。2D の物理（Collider2D）には対応していません。プレイヤーにコライダーは要りません（点は Height から作れます） |

---

## 使い方

### セットアップ（部品を2つ付けるだけ）

1. 敵に **PerceptionSensor** を付けます（Add Component > LogicDevSupport > SmartPerception > Perception Sensor）
2. （おすすめ）頭の高さに、前を向いた空の子オブジェクトを作り、**Eyes** に入れます。視野はここから測ります（空のときは敵自身の Transform）
3. プレイヤーに **PerceptionTarget** を付けます（Add Component > LogicDevSupport > SmartPerception > Perception Target）。**CheckPoints** が空なら、プレイヤーのコライダーから（無ければ **Height** から）足・胸・頭の3点を作ります
4. Play を押します。敵の状態は見えたものに合わせて変わります。`State` を読むか、イベントを使います

すべての設定は既定値のままで動きます。壁は **ObstacleLayers**（既定はすべてのレイヤー）のコライダーです。ほかのセンサーの体（PerceptionSensor を持つ物とその子）は、**OtherSensorsBlock** を ON にしない限り壁になりません。

### AI からセンサーを読む

```csharp
using LogicDev.Perception;

PerceptionSensor sensor = GetComponent<PerceptionSensor>();

if (sensor.State == PerceptionAwareness.Alert && sensor.CanSeeFocus)
{
    // sensor.FocusTarget を追う
}
else if (sensor.State == PerceptionAwareness.Suspicious)
{
    // 止まって sensor.FocusTarget のほうを向く
}

sensor.AwarenessChanged += (previous, next) => Debug.Log(previous + " -> " + next);
sensor.TargetSpotted += target => Debug.Log("Spotted " + target.name);
```

同じイベントはインスペクターでもつなげます（**OnAwarenessChanged**、**OnSuspicious**、**OnAlert**、**OnCalm**、**OnTargetSpotted**、**OnTargetLost**）。

`State` と `Awareness` は、すべてのターゲットのうち一番高いメーターです。`FocusTarget` はそのメーターのターゲットです。

### メーターの上がり方

- 点が見える = **ViewDistance** 以内、左右 **ViewAngle**・上下 **VerticalViewAngle** 以内（上下は Eyes の前の向きから測る）で、間に **ObstacleLayers** のコライダーが無いとき
- ターゲットが見えている = 見える点が **RequiredVisiblePoints** 個以上のとき
- 見えている間、メーターは毎秒 **RiseRate** × 見え方 × ターゲットの **Visibility** だけ上がります。見え方は、点ごとの（角度の倍率 × 距離の倍率）の平均です。角度の倍率は **CentralAngle** の中で1、ViewAngle で **PeripheralFactor**、距離の倍率は Eyes の位置で1、ViewDistance で **FarFactor**（どちらも間は直線）。隠れた点は0として数えます
- 見えなくなると、メーターは **DecayDelay** 秒そのままで、その後は毎秒 **DecayRate** ずつ下がります
- 状態は **SuspiciousThreshold**／**AlertThreshold** ちょうどで上がり、しきい値より **CalmMargin** 下がったときだけ下がります（0 になったら必ず Unaware）

### シーンビューのギズモ

シーンビューで、センサーは視野（視野角・中心視野・距離）を描き、Play 中は各点への線（緑 = 見える、赤 = 壁まで）と、状態とメーターを描きます。**ShowGizmos** で表示のタイミングを選びます: `Selected`（既定）、`Always`、`Off`。

---

## パラメータ一覧

### PerceptionSensor - 視覚

| パラメータ | 既定値 | 説明 |
|-----------|---------|-------------|
| Eyes | （なし） | 見る位置と前の向き（空 = このオブジェクト） |
| ViewDistance | 15 | 見える距離（ワールド単位） |
| ViewAngle | 60 | 左右の視野の半分（度。60 = 幅120°、180 = 全周） |
| VerticalViewAngle | 45 | 上下の視野の半分（度。Eyes の前の向きから測る） |
| CentralAngle | 25 | 左右の中心視野（度）。この中ではメーターが最も速く上がる |
| PeripheralFactor | 0.3 | 視野の端でのメーターの速さ（中心視野に対する倍率） |
| FarFactor | 0.4 | ViewDistance でのメーターの速さ（目の前に対する倍率。間は直線） |
| RequiredVisiblePoints | 1 | 見えているとみなすのに必要な、見える点の数 |
| ObstacleLayers | すべて | 視線を遮るレイヤー（センサーとターゲット自身のコライダー、Trigger は遮らない） |
| OtherSensorsBlock | Off | ほかのセンサーの体（PerceptionSensor を持つ物とその子）が視線を遮るか |

### PerceptionSensor - 警戒度

| パラメータ | 既定値 | 説明 |
|-----------|---------|-------------|
| RiseRate | 0.7 | 中心視野の目の前で全身が見えているときの、毎秒のメーターの上がり（1 = 1秒で Alert） |
| DecayRate | 0.2 | ターゲットが DecayDelay 秒見えなかった後の、毎秒のメーターの下がり |
| DecayDelay | 2 | 最後に見てから、メーターが下がり始めるまでの秒数 |
| SuspiciousThreshold | 0.3 | 状態が Suspicious になるメーターの値 |
| AlertThreshold | 1 | 状態が Alert になるメーターの値 |
| CalmMargin | 0.1 | しきい値よりこれだけ下がったとき（または 0 になったとき）だけ、状態が下がる |
| ShowGizmos | Selected | シーンビューのギズモを描くタイミング（Off／Selected／Always） |

### PerceptionSensor - イベント

| イベント（インスペクター／C#） | タイミング |
|-------|------|
| OnAwarenessChanged／AwarenessChanged | 状態が変わった（前の状態、新しい状態） |
| OnSuspicious、OnAlert、OnCalm | 状態が Suspicious／Alert／Unaware になった |
| OnTargetSpotted／TargetSpotted | ターゲットが見えた |
| OnTargetLost／TargetLost | ターゲットが見えなくなった |

### PerceptionSensor - プロパティとメソッド

| メンバー | 説明 |
|--------|-------------|
| State、Awareness | いまの状態とメーター（0..1）。すべてのターゲットのうち一番高いメーター |
| FocusTarget、Focus、CanSeeFocus | メーターが一番高いターゲット、それについて知っていること（PerceptionRecord）、見えているか |
| Records、GetRecord(target)、CanSee(target) | センサーが知っていること全部（ターゲットごとに1つの PerceptionRecord） |
| ResetAwareness() | すべてのメーターを 0 にする（状態は Unaware になる） |
| DistanceFactorAt(d)、AngleFactorAt(yaw)、TestPoint(...) | 速さの倍率と、1点の視覚の判定 |
| GetViewFrame(...)、NextState(...) | 視野の原点と向き、メーターの値から決まる状態 |
| Sense(deltaTime) | 手動で1回進める（部品を無効にして、自分の時間の刻みで呼ぶ） |
| static All | 有効なすべてのセンサー |
| static IsSensorBody(transform) | その物か親に PerceptionSensor があるか |

### PerceptionTarget

| パラメータ | 既定値 | 説明 |
|-----------|---------|-------------|
| CheckPoints | （空） | センサーが調べる点。空 = コライダー（無ければ Height）から足・胸・頭の3点 |
| Height | 1.8 | コライダーが無いときの、自動の点のための体の高さ（ピボットを足元とする） |
| Visibility | 1 | 気づかれやすさ（0.5 = メーターが半分の速さ、0 = 気づかれない） |
| Detectable | On | Off = センサーはこのターゲットを無視する（隠れ場所の中など） |

### そのほかの型

| 型 | 説明 |
|------|-------------|
| PerceptionRecord | ターゲットごと: Awareness、IsSeen、VisibleFraction、Exposure、LastSeenTime、TimeSinceSeen、Points |
| PerceptionAwareness | Unaware、Suspicious、Alert |

---

## 制限事項

- 3D の物理だけが対象です。2D のゲーム（Collider2D、Physics2D）には対応していません
- 各点は Eyes からのレイ1本で調べます。壁の端のすぐ後ろの点は、まわりの体が少し見えていても、隠れているとみなします
- 視野は左右と上下の2つの角度で、丸い円すいではありません
- すべてのセンサーが毎フレーム視覚の判定をします（重さはセンサーの数 × ターゲットの数 × 点の数に応じて増えます）

---

## デモシーン

`Demo_SmartPerception`（`Assets/LogicDevSupport/SmartPerception/Demo/`）を開いて Play を押します。壁に囲まれた場所を3人の見張りが巡回します。あなたは下の小さな隠れ場所にいる青いカプセルです。

- **操作**: **WASD／矢印キー** で移動、**Shift** を押している間は忍び足（遅い・見つかりにくい）、**R** でやり直し（見張りは全員落ち着き、プレイヤーは初めの位置に戻る）
- **見張り**: 巡回路を歩きます（向かっている巡回ポイントまで地面に細い線）。色が状態を表します（灰色 = Unaware、黄色 = Suspicious、赤 = Alert）。Suspicious か Alert のときは止まり、あなたが見えている間はあなたのほうを向きます。落ち着くと巡回に戻ります
- **物陰に隠れる**: 低い壁は足だけを隠します（見張りには体の一部が見えて、ゆっくり気づかれる）。高い壁は全身を隠します
- **視野**: 見張りの視野と視線はシーンビューに描かれます（ShowGizmos を Always にしてあります）

> 注: 巡回・止まる・向きを変えることは、デモ専用のスクリプト（`Demo/Scripts/` の `PerceptionDemoPatrol` など）が行います。センサーを使って AI が何をできるかの例で、製品には含まれません。消してもセンサーは動きます。**このスクリプトはまっすぐ進むだけで、経路探索をしません**。実際のゲームでは、Unity の NavMesh などの経路探索で敵を動かしてください。

> 注: HDRP では、デモの Play 中に、デモのカメラ（`PerceptionDemoExposure`）が固定の露出と簡単なグラデーションの空を足し、ほかのレンダーパイプラインと同じような見た目にします。Built-in Render Pipeline と URP では何もせず、あなたのシーンも変えません。

> 注: デモシーンはプリミティブの形だけでできています。Starter Assets を使ったイメージはアセットストアのページで確認できます（このパッケージには含まれません）。

---

## 動画

- 導入（セットアップ）: https://youtu.be/JjSQFT_i1v8
- 紹介動画: https://youtu.be/vsd2rVXo6lU
- ショート: https://youtu.be/lvWqspnw7vM
- Smart Perception Pro: 紹介動画 https://youtu.be/5rk-AmZ5jss ／ ショート https://youtu.be/U7dnbUD4_wM

---

## 上位版（Smart Perception Pro）でできること

Smart Perception Pro には Lite のすべて（同じクラス・設定・イベントが同じ名前で）があり、さらに次ができます:

- **記憶** - 最後に見た位置と時刻を、見失ってから決めた時間だけ覚えます
- **聴覚** - 物音は1行（`PerceptionNoise.Emit`）。距離と壁で弱くなります
- **接触** - 敵にぶつかったターゲットには、どの向きからでもすぐ気づきます
- **仲間を呼ぶ声** - Alert になった敵が仲間を呼び、仲間は最後にプレイヤーが見えた場所へ見に来ます
- **距離による上がり方の曲線** - 距離ごとのメーターの上がり方を曲線で描けます（例:「2m 以内は即アラート」）
- **複数プレイヤー** - 敵が追う相手を選びます（近い順・優先度順・メーター順）。行ったり来たりしません
- **多数の敵** - 視覚の判定を複数のフレームに分けます
- **デバッグ表示と調整パネル** - 視野・メーター・物音・声をゲーム画面に表示し、Play 中にセンサーを調整できます（開発時のみ）
- **大きなデモ** - 追跡と捜索、足音と拍手、仲間を呼ぶ声、2人のプレイヤー、ミニマップの見本

https://assetstore.unity.com/packages/slug/426874

**Lite から Pro に替えるとき**: Pro をインポートする前に、Lite のフォルダ（`Assets/LogicDevSupport/SmartPerception/`）を消してください。2つの版は同じフォルダ・同じクラス名・同じファイルの識別子を使うため、Lite の上に Pro をインポートすると、フォルダの中に違う版のファイルが残ることがあります。フォルダの中のファイル（デモシーンなど）を編集している場合は、先にバックアップを取ってください。入れ替えた後もシーンはそのまま動きます。

---

もしこのアセットが役に立ちましたら、ぜひレビューを書いて応援していただけると開発の励みになります！

レビューはこちら（「Reviews」タブから投稿できます）:

https://assetstore.unity.com/packages/slug/426870
