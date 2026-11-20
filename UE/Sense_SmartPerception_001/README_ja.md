# Smart Perception Lite（Unreal Engine）- 敵の視覚と警戒度

敵や NPC に付ける「気づく」コンポーネントです（無料版）。敵に **Perception Sensor**（`ULogicDevPerceptionSensorComponent`）、プレイヤーに **Perception Target**（`ULogicDevPerceptionTargetComponent`）を付けるだけで、敵が見て（視野・距離・遮蔽物・体の複数の点）、警戒度のメーターを上げます（Unaware → Suspicious → Alert）。気づいたことは関数とイベント（Blueprint・C++）で AI に伝えます。敵を動かすこと（追跡・巡回・攻撃）はこのパッケージに含みません。あなたの AI（Blueprint、Behavior Tree、C++ など）がセンサーを読んで、何をするかを決めます。

---

## 特徴

- **体の一部が隠れた状態の視覚** - ターゲットごとに複数の点（既定は足・胸・頭）を調べます。低い壁の陰のプレイヤーは一部が見えていて、見えている量に応じてメーターが上がります
- **中心視野と周辺視野・距離** - 中心の角度の中では最も速く、視野の端や見える距離の端ほどゆっくりメーターが上がります
- **3段階の警戒度メーター** - Unaware → Suspicious → Alert。保持時間・下がり方・しきい値の余裕があり、しきい値の付近で状態がちらつきません
- **イベント** - 状態の変化、見つけた／見失った、を Blueprint のイベント（`BlueprintAssignable`）で。C++ では `AddDynamic` でつなげます
- **エディタのビューポートの表示** - 視野（視野角・中心視野・距離）をビューポートに線で表示します
- **敵の集団** - ほかの敵の体は既定では視界をさえぎらないので、集まった見張りどうしでプレイヤーを隠しません

---

## 動作環境

| 項目 | 内容 |
|------|------|
| Unreal Engine バージョン | UE 5.3 / 5.5 / 5.7 / 5.8 |
| 実装言語 | C++（プラグイン `LogicDevSmartPerception`。Runtime モジュール、Win64） |
| Blueprint 対応 | 設定は `BlueprintReadWrite`、関数は `BlueprintCallable`／`BlueprintPure`、イベントは `BlueprintAssignable`。Blueprint だけでも使えます |
| 入力システム | 依存なし。Enhanced Input などの入力用のプラグインは不要です（同梱デモのキーは、プレイヤーコントローラーからキーの状態を直接読み取ります） |
| コリジョン | 遮蔽物は、**ObstacleChannel**（既定は Visibility）をブロックするコンポーネントへのライントレースで調べます。プレイヤーにコリジョンは要りません（点は **Height** から作れます） |
| 単位 | 距離はすべて Unreal の単位（cm）です。角度は度、時間は秒です |

---

## 使い方

### セットアップ（コンポーネントを2つ付けるだけ）

1. 敵の Actor に **Perception Sensor** を追加します（Add Component で「Perception」と検索。カテゴリ LogicDevSupport）
2. センサーを**目の位置**に置き、前（+X）を見たい向きに向けます。**センサーのコンポーネントそのものが目です**。視野はセンサーの位置と前の向き（+X）から測ります。キャラクターの頭に付けるときは、センサーの Parent Socket に頭のボーンやソケットを選びます
3. プレイヤーの Actor に **Perception Target** を追加します。**CheckPoints** が空なら、Actor のコリジョンのあるコンポーネント（Character ならカプセル）から（無ければ **Height** から）足・胸・頭の3点を作ります
4. Play を押します。敵の状態は見えたものに合わせて変わります。**Get State** を読むか、イベントを使います

すべての設定は既定値のままで動きます。遮蔽物は **ObstacleChannel**（既定 Visibility）をブロックするコンポーネントです。センサー自身の Actor とターゲットの Actor（それぞれにアタッチされた Actor を含む）、オーバーラップだけのコンポーネントは遮りません。ほかのセンサーの体（Perception Sensor を持つ Actor と、それにアタッチされた Actor）は、**bOtherSensorsBlock** を ON にしない限り遮りません。

詳細パネルでは、設定は **Smart Perception** の下の Vision、Awareness、Gizmos にあります。詳細パネルの名前は、単語の間に空白が入り、先頭の `b` が取れた形で出ます（例: `bOtherSensorsBlock` → **Other Sensors Block**）。

### AI からセンサーを読む

Blueprint では、センサーのコンポーネントから **Get State**、**Get Awareness**、**Get Focus Target**、**Can See Focus** などのノード（カテゴリ LogicDevSupport | SmartPerception）を呼びます。イベントは、コンポーネントを選んだ詳細パネルの Events の **+**（**On Awareness Changed**、**On Suspicious**、**On Alert**、**On Calm**、**On Target Spotted**、**On Target Lost**）か、**Bind Event to ...** で受け取ります。

C++ では、モジュールの `Build.cs` の依存に `"LogicDevSmartPerception"` を足してから使います。

```cpp
#include "LogicDevPerceptionSensorComponent.h"
#include "LogicDevPerceptionTargetComponent.h"

ULogicDevPerceptionSensorComponent* Sensor = GetOwner()->FindComponentByClass<ULogicDevPerceptionSensorComponent>();

if (Sensor->GetState() == ELogicDevPerceptionAwareness::Alert && Sensor->CanSeeFocus())
{
    // Sensor->GetFocusTarget()->GetOwner() を追う
}
else if (Sensor->GetState() == ELogicDevPerceptionAwareness::Suspicious)
{
    // 止まって Sensor->GetFocusTarget() のほうを向く
}

// イベント。受け取る関数は UFUNCTION にしてください:
// void HandleAwarenessChanged(ELogicDevPerceptionAwareness Previous, ELogicDevPerceptionAwareness Current);
// void HandleSpotted(ULogicDevPerceptionTargetComponent* Target);
Sensor->OnAwarenessChanged.AddDynamic(this, &AMyAIController::HandleAwarenessChanged);
Sensor->OnTargetSpotted.AddDynamic(this, &AMyAIController::HandleSpotted);
```

**Get State** と **Get Awareness** は、すべてのターゲットのうち一番高いメーターです。**Get Focus Target** はそのメーターのターゲットです。

**ネット対戦**: センサーは、自分のいるマシンの中だけで動き、レプリケーションはしません。オンラインのゲームでは、センサーを1台（ふつうはサーバー）で動かし、結果（状態など）をゲームのコードでほかのプレイヤーに送ってください。

### エディタのビューポートの表示（Play していないとき）

センサーは、視野（視野角・中心視野・距離）をエディタのビューポートに線で表示します。**ShowGizmos** で表示のタイミングを選びます: `Selected`（既定。Actor を選んでいる間）、`Always`、`Off`。ビューポートの **Realtime** が ON のときに出ます。Play 中・パッケージしたゲームには出ません（Play 中の様子は、状態を読んで自分の表示に使ってください。同梱デモでは見張りの色が状態を表します）。

---

## 判定と計算

- **点が見える**: センサーから **ViewDistance** 以内、左右 **ViewAngle**・上下 **VerticalViewAngle** 以内（どちらもセンサーの前の向きから測る）、間に遮蔽物が無い
- **見えている**: 見える点の数 ≥ min(**RequiredVisiblePoints**, 点の数)。**bDetectable** が OFF のターゲットは見えません
- **メーターの上がり方**（毎秒）= RiseRate × Exposure × ターゲットの **Visibility**。Exposure = Σ（見える点の 角度の係数 × 距離の係数）÷ 点の数
  - 角度の係数: **CentralAngle** 以内は 1、ViewAngle で **PeripheralFactor**（間は直線）
  - 距離の係数: 0 cm で 1、ViewDistance で **FarFactor**（間は直線）
- **下がり方**: 最後に見てから **DecayDelay** 秒はそのまま、その後 **DecayRate**／秒
- **状態**: 上がるときはしきい値ちょうどで、下がるときは しきい値 − **CalmMargin** を下回ったとき（メーターが 0 になったら必ず Unaware）

---

## パラメータ一覧

### Perception Sensor - 視覚（Vision）

| パラメータ | 既定値 | 説明 |
|-----------|---------|-------------|
| `ViewDistance` | 1500（cm） | 見える距離（50〜20000） |
| `ViewAngle` | 60 | 左右の視野の半分（度。60 = 幅120°、180 = 全周） |
| `VerticalViewAngle` | 45 | 上下の視野の半分（度。センサーの前の向きから測る） |
| `CentralAngle` | 25 | 左右の中心視野（度）。この中ではメーターが最も速く上がる（ViewAngle より大きい値は ViewAngle になる） |
| `PeripheralFactor` | 0.3 | 視野の端でのメーターの速さ（中心視野に対する倍率） |
| `FarFactor` | 0.4 | ViewDistance でのメーターの速さ（目の前に対する倍率。間は直線） |
| `RequiredVisiblePoints` | 1 | 見えているとみなすのに必要な、見える点の数（1〜8） |
| `ObstacleChannel` | Visibility | 視線を遮るトレースチャンネル（センサーとターゲット自身の Actor、オーバーラップだけのコンポーネントは遮らない） |
| `bOtherSensorsBlock` | Off | ほかのセンサーの体（Perception Sensor を持つ Actor と、それにアタッチされた Actor）が視線を遮るか |

### Perception Sensor - 警戒度・表示（Awareness、Gizmos）

| パラメータ | 既定値 | 説明 |
|-----------|---------|-------------|
| `RiseRate` | 0.7 | 中心視野の目の前で全身が見えているときの、毎秒のメーターの上がり（1 = 1秒で Alert） |
| `DecayRate` | 0.2 | DecayDelay 秒見えなかった後の、毎秒のメーターの下がり |
| `DecayDelay` | 2 | 最後に見てから、メーターが下がり始めるまでの秒数 |
| `SuspiciousThreshold` | 0.3 | 状態が Suspicious になるメーターの値 |
| `AlertThreshold` | 1 | 状態が Alert になるメーターの値（SuspiciousThreshold より小さい値は SuspiciousThreshold になる） |
| `CalmMargin` | 0.1 | しきい値よりこれだけ下がったとき（または 0 になったとき）だけ、状態が下がる |
| `ShowGizmos` | Selected | エディタのビューポートに視野を描くタイミング（Off／Selected／Always） |

### Perception Sensor - イベント

詳細パネルの Events、または **Bind Event to ...**。C++ は `AddDynamic`。

| イベント | タイミング |
|-------|------|
| `OnAwarenessChanged` | 状態が変わった（前の状態、新しい状態） |
| `OnSuspicious`、`OnAlert`、`OnCalm` | 状態が Suspicious／Alert／Unaware になった |
| `OnTargetSpotted` | ターゲットが見えた |
| `OnTargetLost` | ターゲットが見えなくなった |

### Perception Sensor - 関数

| 関数 | 説明 |
|--------|-------------|
| `GetState()`、`GetAwareness()` | いまの状態とメーター（0..1）。すべてのターゲットのうち一番高いメーター |
| `GetFocusTarget()`、`GetFocusRecord(Out)`、`CanSeeFocus()` | メーターが一番高いターゲット、それについて知っていること（`FLogicDevPerceptionRecord`）、見えているか |
| `GetRecords()`、`GetRecord(Target, Out)`、`CanSee(Target)`、`GetTimeSinceSeen(Record)` | センサーが知っていること全部（ターゲットごとに1つの `FLogicDevPerceptionRecord`） |
| `ResetAwareness()` | すべてのメーターを 0 にする（状態は Unaware になる） |
| `GetViewFrame(Origin, Forward, Up)`、`GetViewAngles(...)` | 見る位置と向き、ある方向の左右・上下の角度 |
| `DistanceFactorAt(cm)`、`AngleFactorAt(Yaw)`、`TestPoint(...)`、`NextState(...)` | 速さの倍率、1点の視覚の判定、メーターの値から次の状態 |
| `Sense(DeltaTime)` | 手動で1回進める（コンポーネントのティックを止めて、自分の時間の刻みで呼ぶ） |
| static `GetAllSensors(WorldContext)` | そのワールドのすべてのセンサー |
| static `IsSensorBody(Actor)` | その Actor か、そのアタッチの親に Perception Sensor があるか |

### Perception Target（`ULogicDevPerceptionTargetComponent`）

| パラメータ | 既定値 | 説明 |
|-----------|---------|-------------|
| `CheckPoints` | （空） | センサーが調べる点: Actor のメッシュのソケット・ボーンの名前、または Actor のシーンコンポーネントの名前。空 = コリジョンのあるコンポーネント（無ければ Height）から足・胸・頭の3点 |
| `Height` | 180（cm） | コリジョンのあるコンポーネントが無いときの、自動の点のための体の高さ（Actor の位置を足元とする） |
| `Visibility` | 1 | 気づかれやすさ（0〜2。0.5 = メーターが半分の速さ、0 = 気づかれない） |
| `bDetectable` | On | Off = センサーはこのターゲットを無視する（隠れ場所の中など） |

関数: `GetCheckPoints(Out)`（いまの点のワールド位置）、static `GetAllTargets(WorldContext)`。

### そのほか

| 名前 | 説明 |
|------|-------------|
| `ULogicDevPerceptionSubsystem` | ワールドごとのセンサーとターゲットの一覧（`GetPerceptionSubsystem`） |
| `FLogicDevPerceptionRecord` | ターゲットごと: Target、Awareness、bIsSeen、VisibleFraction、Exposure、LastSeenTime、Points |
| `ELogicDevPerceptionAwareness` | Unaware、Suspicious、Alert |
| `ELogicDevPerceptionPointStatus` | Visible、OutOfRange、OutOfAngle、Blocked |

### Unity 版との違い

Unity 版の Smart Perception Lite と同じ機能・同じ既定値です。違いは次のとおりです。

| 項目 | Unity 版 | UE 版 |
|---|---|---|
| 目の位置 | `Eyes`（Transform。空なら自身） | センサーのコンポーネントそのもの（位置と +X の向き） |
| 距離の単位 | m（ViewDistance 15、Height 1.8） | cm（1500、180） |
| 遮蔽物 | `ObstacleLayers`（レイヤー）、Trigger は遮らない | `ObstacleChannel`（トレースチャンネル）、オーバーラップだけのコンポーネントは遮らない |
| 名前 | `PerceptionSensor`、`State`、`AwarenessChanged` など | `ULogicDevPerceptionSensorComponent`、`GetState()`、`OnAwarenessChanged` など。真偽の設定は `b` が付く（`bOtherSensorsBlock` など） |
| イベント | C# のイベントと UnityEvent | Blueprint のイベント（dynamic multicast delegate） |
| 表示 | シーンビューのギズモ（Play 中は各点への線と状態も） | エディタのビューポートの視野の線（Play していないとき） |

---

## 制限事項

- 各点は、センサーからのライントレース1本で調べます。遮蔽物の端のすぐ後ろの点は、まわりの体が少し見えていても、隠れているとみなします
- 視野は左右と上下の2つの角度で、丸い円すいではありません
- すべてのセンサーが毎フレーム視覚の判定をします（重さはセンサーの数 × ターゲットの数 × 点の数に応じて増えます）
- ネット対戦の同期（レプリケーション）は含みません

---

## デモシーンについて

同梱のデモレベル `L_SmartPerception_Demo`（プラグインの `Content/Demo/`。コンテンツブラウザでは、設定の **Show Plugin Content** を ON にすると、プラグイン `LogicDevSmartPerception` のフォルダの `Demo` に出ます）を開いて Play を押します。壁に囲まれた場所を3人の見張りが巡回します。あなたは下の小さな隠れ場所にいる青い円柱（`Player`）です。

- **操作**: **WASD／矢印キー** で移動、**Shift** を押している間は忍び足（遅い・見つかりにくい）、**R** でやり直し（見張りは全員落ち着き、プレイヤーは初めの位置に戻る）
- **見張り**: 巡回路を歩きます（向かっている巡回ポイントまで地面に細い線）。色が状態を表します（灰色 = Unaware、黄色 = Suspicious、赤 = Alert）。Suspicious か Alert のときは止まり、あなたが見えている間はあなたのほうを向きます。落ち着くと巡回に戻ります
- **物陰に隠れる**: 低い壁は足だけを隠します（見張りには体の一部が見えて、ゆっくり気づかれる）。高い壁は全身を隠します
- **視野**: 見張りの視野は、Play していないときにエディタのビューポートに描かれます（見張りの ShowGizmos を Always にしてあります）

> 注: 巡回・止まる・向きを変えることは、デモ専用のクラス（`ULogicDevPerceptionDemoPatrolComponent` など、名前に `Demo` が付くもの）が行います。センサーを使って AI が何をできるかの例で、製品の機能ではありません。**このクラスはまっすぐ進むだけで、経路探索をしません**。実際のゲームでは、Navigation Mesh（AI Move To など）で敵を動かしてください。

> 注: デモシーンはプリミティブの形だけでできています。マネキンを使ったイメージは Fab の商品ページで確認できます（このパッケージには含まれません）。

---

## 困ったとき

- **RTX 40 番台などで、同梱デモを開いた直後にエディタが落ちる（GPU のクラッシュ）とき**は、まず GPU のドライバを更新してください。直らなければ `Config/DefaultEngine.ini` の `[SystemSettings]` に `r.RDG.AsyncCompute=0` を足してください（UE の影と光の機能の、エンジンかドライバ側の不具合です。プラグインのコードは関係しません）。
- **Play in Editor で、Play を押した直後にキーが効かないとき**: 同梱デモでは、Play の始めにキーボードの入力先を自動でゲームの画面に向けています（デモのプレイヤーコントローラー `ALogicDevPerceptionDemoPlayerController`）。自分のレベルで効かないときは、一度ゲームの画面をクリックしてからキーを押すか、エディタの環境設定の **Level Editor > Play > Game Gets Mouse Control** を ON にしてください（UE の初期値は OFF で、マウスで Play を押した直後はキーがエディタ側に残ります）。
- **センサーが何も見ないとき**は、センサーの位置と前の向き（+X。ビューポートで Actor を選ぶと視野の線が出ます）、ターゲットの **bDetectable** と **Visibility**（0 だと気づかれません）、間にあるものが **ObstacleChannel** をブロックしていないかを確かめてください。
- **コンテンツブラウザにデモのレベルが出ないとき**は、コンテンツブラウザの設定で **Show Plugin Content** を ON にしてください。

---

## 上位版（Smart Perception Pro）でできること

Smart Perception Pro には Lite のすべて（同じプラグイン名・クラス・設定・イベントが同じ名前で）があり、さらに次ができます:

- **記憶** - 最後に見た位置と時刻を、見失ってから決めた時間だけ覚えます
- **聴覚** - 物音は1行（**Emit Noise**）。距離と遮蔽物で弱くなります
- **接触** - 敵にぶつかったターゲットには、どの向きからでもすぐ気づきます
- **仲間を呼ぶ声** - Alert になった敵が仲間を呼び、仲間は最後にプレイヤーが見えた場所へ見に来ます
- **距離による上がり方の曲線** - 距離ごとのメーターの上がり方を曲線で描けます（例:「2m 以内は即アラート」）
- **複数プレイヤー** - 敵が追う相手を選びます（近い順・優先度順・メーター順）。行ったり来たりしません
- **多数の敵** - 視覚の判定を複数のフレームに分けます
- **デバッグ表示と調整パネル** - 視野・メーター・物音・声をゲーム画面に表示し、Play 中にセンサーを調整できます（開発時のみ）
- **大きなデモ** - 追跡と捜索、足音と拍手、仲間を呼ぶ声、2人のプレイヤー、ミニマップの見本

https://www.fab.com/listings/ab734007-2ae6-4937-927d-8da8e1eb9e2f

**Lite から Pro に替えるとき**: エディタを閉じて、Pro を入れる前に Lite を削除してください（プロジェクトの `Plugins/LogicDevSmartPerception/` に置いた場合はそのフォルダを、エンジンに入れた場合はエンジンの側の Lite を削除）。2つの版は同じプラグイン名（`LogicDevSmartPerception`）・同じクラス名を使うため、両方があるとプラグインがぶつかり、上から重ねると Lite のファイルが残ることがあります。プラグインの中のファイル（デモのレベルなど）を編集している場合は、先にバックアップを取ってください。Lite で作ったレベルと Blueprint は、入れ替えた後も設定とイベントのつながりがそのまま残ります。

---

もしこのアセットが役に立ちましたら、ぜひ Fab の商品ページでレビューを書いて応援していただけると開発の励みになります！

https://www.fab.com/listings/394b12a0-78f8-470e-bcc5-0218e851a571
