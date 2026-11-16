# Smart Perception Pro - 敵の視覚・聴覚・警戒度

敵や NPC に付ける「気づく」部品です。敵に `PerceptionSensor`、プレイヤーに `PerceptionTarget` を付けるだけで、敵が見る（視野・距離・壁・体の複数の点）、物音を聞く、触れたことに気づく、警戒度のメーターを上げる（Unaware → Suspicious → Alert）、最後に見た・聞いた位置を覚える、見つけたら仲間を呼ぶ、プレイヤーが複数いるときに追う相手を選ぶ、ができます。気づいたことはプロパティとイベントで AI に伝えます。敵を動かすこと（追跡・巡回・攻撃）はこのパッケージに含みません。あなたの AI がセンサーを読んで、何をするかを決めます。

---

## 特徴

- **体の一部が隠れた状態の視覚** - ターゲットごとに複数の点（既定は足・胸・頭）を調べます。低い壁の陰のプレイヤーは一部が見えていて、見えている量に応じてメーターが上がります
- **中心視野と周辺視野・距離** - 中心の角度の中では最も速く、視野の端や見える距離の端ほどゆっくりメーターが上がります
- **距離による上がり方の曲線** - 距離ごとのメーターの上がり方を曲線で描けます（例:「2m 以内は即アラート」）。既定は OFF（直線を使います）
- **3段階の警戒度メーター** - Unaware → Suspicious → Alert。保持時間・下がり方・しきい値の余裕があり、しきい値の付近で状態がちらつきません
- **記憶** - 最後に見た位置と時刻を、見失ってから決めた時間だけ覚えます
- **聴覚** - 物音は1行: `PerceptionNoise.Emit(位置, 範囲, 出どころ)`。距離と壁で弱くなります。物音だけでは Alert になりません
- **接触** - すぐ近くまで来たターゲットには、どの向きからでもすぐ気づきます（後ろからぶつかる）
- **仲間を呼ぶ声（Alert Call）** - Alert になった敵が声を出し、近くの仲間は「呼んだ敵が最後にプレイヤーを見た場所」を聞いて見に来ます。声だけでは仲間は Alert にならないので、マップ全体に広がりません
- **複数プレイヤー（協力プレイ）** - 敵は、怪しいと思ったプレイヤーの中から追う相手を1人選びます: 近い順（既定）、優先度順、メーター順。見えている相手が音だけの相手より先、触られたらすぐその相手、保持時間で行ったり来たりしません
- **多数の敵** - 多くのセンサーの視覚の判定を複数のフレームに分けます（センサーごとの間隔と、1フレームの上限）。メーターは毎フレームなめらかに動きます。ほかの敵の体は既定では視界をさえぎらないので、集まった見張りどうしでプレイヤーを隠しません
- **イベント** - 状態の変化、見つけた／見失った／忘れた／触れた、物音を聞いた、仲間を呼んだ、追う相手が変わった、を C# のイベントと UnityEvent で
- **デバッグ表示と調整パネル（開発時のみ）** - 視野・視線・メーター・物音・声のゲーム画面への表示、Play 中に値を変えられるパネル、Play 中に変えた値を残す仕組み。エディタと Development Build にだけあります
- **ミニマップの見本** - デモに、公開されている API だけで作った小さなレーダー（プレイヤー、状態の色と視野の扇つきの敵、壁）があります。自分のミニマップを作るときの出発点に使えます

---

## 動作環境

| 項目 | 内容 |
|------|------|
| Unity バージョン | Unity 2022.3 LTS / Unity 6.3 LTS |
| レンダーパイプライン | Built-in Render Pipeline・URP・HDRP（確認済み: Unity 2022.3 URP／Built-in／HDRP、Unity 6.3 URP／Built-in／HDRP）。部品はレンダーパイプライン固有の API を使いません。同梱デモのマテリアルは、パッケージを入れたときに、使っているレンダーパイプラインの標準のシェーダーに切り替わります。HDRP では、デモの Play 中にデモのカメラが固定の露出と簡単な空を足します |
| Input System | 部品は入力の仕組みに依存しません。同梱デモとデバッグ用ツールのキーだけが New Input System（Input System パッケージ）を使います |
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
else if (sensor.State != PerceptionAwareness.Unaware && sensor.TryGetLastKnownPosition(out Vector3 pos))
{
    // pos へ行って見回す（最後に見た・触れた・聞いた位置）
}

sensor.AwarenessChanged += (previous, next) => Debug.Log(previous + " -> " + next);
sensor.TargetSpotted += target => Debug.Log("Spotted " + target.name);
```

同じイベントはインスペクターでもつなげます（**OnAwarenessChanged**、**OnSuspicious**、**OnAlert**、**OnCalm**、**OnTargetSpotted** など）。

`State` と `Awareness` は、すべてのターゲットのうち一番高いメーターです。`FocusTarget` はセンサーが追う1人のターゲットで（「複数プレイヤー」）、記憶・仲間を呼ぶ声・デバッグ表示はこのターゲットを使います。

### 物音（1行）

```csharp
// 足音・銃声・ドア: 出どころ（PerceptionTarget）を渡すと、見張りはそのターゲットのメーターを上げる
PerceptionNoise.Emit(transform.position, 8f, myTarget);

// 投げた石（出どころなし）: 見張りは知らせるだけ（OnNoiseHeard、LastNoise）
PerceptionNoise.Emit(stone.position, 10f);
```

`範囲` × そのセンサーの **HearingSensitivity** 以内の、有効なセンサーすべてが聞きます。強さは物音の場所で1、その距離で0（間は直線）で、物音と Eyes の間の壁1枚ごとに **WallDamping** を掛けます。最も強く聞こえた物音でメーターに **NoiseAwareness** を足し、弱いほど少なく足します。物音だけではメーターは **NoiseMaxAwareness**（既定 0.7: Suspicious にはなるが Alert にはならない）より上がりません。見張りは聞いた場所を覚えます（`TryGetLastKnownPosition`）。

物音のイベント（`OnNoiseHeard`／`NoiseHeard`）の中から `PerceptionNoise.Emit` を呼んでもかまいません（物音を伝え直すときなど）。

### 接触

敵の体（足元から Eyes までの線）から **TouchRadius**（既定 1m）より近いターゲットは、どの向きからでも、見えていなくても、メーターが少なくとも **TouchAwareness**（1 = すぐ Alert）になります。TouchRadius を 0 にすると OFF です。

### 距離による上がり方の曲線

既定では、メーターは Eyes の目の前で最も速く、**ViewDistance** で **FarFactor** の速さになります（間は直線）。**UseDistanceCurve** を ON にすると、代わりに **DistanceCurve** で倍率を自分で描けます（このとき FarFactor は灰色になり、使われません）:

- 横軸: 距離 ÷ ViewDistance（0 = Eyes の位置、1 = ViewDistance）
- 縦軸: 上がり方の倍率（1 = ふつう、0 = 上がらない、1 より大きい = 速い）
- 既定の曲線は FarFactor 0.4 と同じ直線なので、ON にしても、曲線を直すまでは何も変わりません

**例:「2m 以内は即アラート」**（ViewDistance 15、RiseRate 0.7）: 点 (0, 30)、(0.13, 30)、(0.14, 1)、(1, 0.4)。点を右クリックして **Both Tangents > Linear** を選び、直線でつなぎます。2m（0.13 × 15）以内ではメーターが30倍（毎秒 21）で上がり、60fps で約3フレームで Alert になります。それより遠くでは今までどおりです（テストでは 6m で Alert まで 1.75秒）。

### 仲間を呼ぶ声（Alert Call）

**AlertCall** を ON にすると、敵が Alert になったときに仲間を呼びます。声はふつうの物音（`PerceptionNoise.Emit`）で、出る場所は Eyes、音の持ち主は見つけたターゲットです:

- **AlertCallRange**（× 聞く側の HearingSensitivity。壁は WallDamping で弱くなる）以内の仲間は、そのターゲットのメーターを上げます（NoiseMaxAwareness まで）
- **声が伝えるのは、呼んだ敵が最後にターゲットを見た（触れた）位置です**（呼んだ敵の立っている位置ではありません。ターゲットの記憶が無いときは、呼んだ敵の Eyes の位置）。仲間はその位置を覚え、`TryGetLastKnownPosition` が返すので、AI はそこへ向かわせられます。声の届く範囲と壁は、今までどおり呼んだ敵の Eyes から測ります
- 自分の声には反応しません
- **AlertCallInterval**: Alert の間に声を繰り返す間隔（秒）。0 なら Alert になるたびに1回だけ
- **AlertCallOnlyWhileSeen**（既定 ON）: 声を繰り返すのはターゲットが見えている間だけです。見失った敵は呼び続けません。Alert になったときの最初の1回は、どちらでも出ます
- **声は連鎖しません**: 声で上がるメーターは NoiseMaxAwareness までなので、仲間は Suspicious にはなっても Alert にはならず、仲間がさらに呼ぶことはありません。仲間が Alert になるのは、自分で見るか触れたときだけです（NoiseMaxAwareness を AlertThreshold 以上にすると、声で Alert になり、さらに呼ぶようになります）
- イベント: **OnAlertCall**／`AlertCalled`（見つけたターゲット）。`LastAlertCallTime` と `LastAlertCallHeard`（直前の声を聞いたセンサーの数）

自分で伝える位置を決めて声を出すときは、`PerceptionNoise.Emit(位置, 範囲, 出どころ, 伝える位置)` を使います。聞いた側は `伝える位置` を覚え（`PerceptionHeardNoise.ReportedPosition`）、範囲と壁は `位置` から測ります。

### 複数プレイヤー（追う相手の選び方）

プレイヤーが複数いるとき（協力プレイ）、各センサーはそのうちの1人を追います: `FocusTarget`。選ぶのは、怪しいと思っている（メーターが **SuspiciousThreshold** 以上の）ターゲットだけです。誰もいなければ、メーターが一番高いターゲットです（このときも **FocusHoldTime** の間は替えないので、低いメーターの2人が抜きつ抜かれつしても行ったり来たりしません）。**FocusMode** で順番を選びます:

- **Nearest**（既定）: Eyes から近い順
- **Priority**: **Priority**（PerceptionTarget の設定）が高い順、同じなら近い順。例: 目標物を運んでいるプレイヤーの優先度を上げる
- **Awareness**: メーターが高い順、同じなら近い順（2人とも 100% のときは近い方）

3つに共通の決まり:

1. **見えている相手が先**: 見えている（触っている）ターゲットが、音や記憶だけのターゲットより先です。いまの相手が見えている間は、ほかの見えている相手にしか切り替えないので、後ろの足音で振り返りません。足音を立てた相手のメーターは上がり、位置も覚えます。いまの相手を見失ったら、そちらへ向き直ります
2. **触られたらすぐ**: センサーに触れたターゲットは、すぐに追う相手になります（保持時間・優先度より先）。触れている間はその相手のままです
3. **行ったり来たりしない**: 切り替えたら **FocusHoldTime** 秒（既定 2）はそのまま。その後も、はっきりよい相手にだけ切り替えます: **FocusSwitchMargin**（0.1）= 10% 以上近い、または Awareness ではメーターが 0.1 以上高い。優先度が高い相手、いまの相手を見失ったときの見えている相手、触った相手は、余裕なしで切り替わります

イベント: **OnFocusChanged**／`FocusChanged`（前の相手、新しい相手。どちらも null のことがあります）。`LastFocusChangeTime`。

**ネット対戦**: センサーは、自分のいる端末の中だけで動き、ネットワークのことは知りません。オンラインのゲームでは、センサーを1台（ふつうはサーバーかホスト）で動かし、結果（状態や追う相手）をゲームのコードでほかのプレイヤーに送ってください。同期は利用者のゲーム側で行います。

### 多数の敵（処理の分散）

センサーの視覚の判定（レイキャスト）は **UpdateInterval** 秒ごと（既定 0.1、0 = 毎フレーム）に行います。センサーは順番に判定し、`PerceptionScheduler.MaxSensorsPerFrame`（既定 32、0 = 上限なし）が1フレームに判定する数の上限です。判定と判定の間も、メーター・接触・物音は毎フレーム、直前の判定の結果で進みます。

見張りが集まると、前の見張りの体が後ろの見張りからプレイヤーを隠してしまいます。**OtherSensorsBlock** が OFF（既定）のときは、ほかのセンサー（PerceptionSensor を持つ物とその子）のコライダーは、視界をさえぎらず、物音も弱めません。敵どうしで隠し合うようにしたいときは ON にします。

### デバッグ表示・調整パネル・Play 中の値を残す

次のツールは `Debug` フォルダにあります。**エディタと Development Build にだけあり、リリースビルド（Development Build でないビルド）ではコードが無くなります**（シーンに置いた2つの部品は、Missing Script の警告が出ないよう空の部品として残ります）。センサーはこれらに依存しないので、`Debug` フォルダごと消してもかまいません。

**付け外し**: **Tools > LogicDevSupport > SmartPerception > Add Debug Tools to Scene** で、2つの部品を付けた GameObject（"Perception Debug Tools"）を1つ作ります。**Remove Debug Tools from Scene** で消します（どちらも Ctrl+Z で戻せます）。

- **Perception Debug View**: センサーごとに、視野（視野角・中心視野・距離）、各点への線（緑 = 見える、赤 = 壁まで）、頭上のメーター（目盛り = しきい値）、最後に見た位置、最後に聞いた位置、Suspicious／Alert でターゲットが見えていないときの最後に分かっている位置への点線（AI が見に行く先）、接触の半径、物音（輪と、聞いたセンサーへの線）、仲間を呼ぶ声（届く範囲の赤い輪）、左上の状態の表（1行 = 1センサー）。**G** で表示を切り替えます
- **Perception Tuning Panel**: 右下の小さなパネルで、Play 中に1つずつセンサーの主な設定を変えられます（`<` `>` でセンサーを選ぶ）。デバッグ表示の項目も切り替えられます。**T** で表示を切り替えます。タイトルをドラッグして動かせます
- **UI モード**: パネルが画面にある間は、クリックできるようにマウスカーソルを表示・解除します。自分のカメラのスクリプトは、1行で UI モード中のマウス操作を止められます: `if (PerceptionTuningPanel.IsUIMode) return;`（名前空間 `LogicDev.Perception`）。リリースビルドでは `IsUIMode` は常に false です
- **Play 中の値を残す（エディタのみ）**: **Record values**（パネル、または Play 中のセンサーのインスペクターの下）を押します。Play を終えると、シーンと違う設定が一覧で出ます。残すものにチェックして **Apply** を押します（Ctrl+Z で戻せます）

**シーンビューのギズモ（エディタ）**: 視野、接触の半径、視線、最後に見た位置。**ShowGizmos** で表示のタイミングを選びます: `Selected`（既定）、`Always`、`Off`。

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
| UseDistanceCurve | Off | FarFactor の代わりに DistanceCurve を使う |
| DistanceCurve | 直線 1 → 0.4 | 距離 ÷ ViewDistance ごとの倍率（「距離による上がり方の曲線」） |
| RequiredVisiblePoints | 1 | 見えているとみなすのに必要な、見える点の数 |
| ObstacleLayers | すべて | 視線を遮り、物音を弱くするレイヤー（センサーとターゲット自身のコライダー、Trigger は遮らない） |
| OtherSensorsBlock | Off | ほかのセンサーの体（PerceptionSensor を持つ物とその子）が視線を遮り、物音を弱くするか |

### PerceptionSensor - 警戒度と記憶

| パラメータ | 既定値 | 説明 |
|-----------|---------|-------------|
| RiseRate | 0.7 | 中心視野の目の前で全身が見えているときの、毎秒のメーターの上がり（1 = 1秒で Alert） |
| DecayRate | 0.2 | DecayDelay 秒何も気づかなかった後の、毎秒のメーターの下がり |
| DecayDelay | 2 | 最後に気づいて（見た・聞いた・触れた）から、メーターが下がり始めるまでの秒数 |
| SuspiciousThreshold | 0.3 | 状態が Suspicious になるメーターの値 |
| AlertThreshold | 1 | 状態が Alert になるメーターの値 |
| CalmMargin | 0.1 | しきい値よりこれだけ下がったとき（または 0 になったとき）だけ、状態が下がる |
| ForgetTime | 10 | 見失ってから、最後に見た・聞いた位置を覚えている秒数 |

### PerceptionSensor - 追う相手（複数プレイヤー）

| パラメータ | 既定値 | 説明 |
|-----------|---------|-------------|
| FocusMode | Nearest | 怪しいと思っているターゲットから FocusTarget を選ぶ順番: Nearest、Priority、Awareness |
| FocusHoldTime | 2 | 切り替えた後、追う相手を変えない秒数（触られたとき・いまの相手を見失ったときは待たない） |
| FocusSwitchMargin | 0.1 | 切り替えるのに必要な差: 0.1 = 10% 以上近い、または（Awareness）メーターが 0.1 以上高い |

### PerceptionSensor - 聴覚・仲間を呼ぶ声・接触・処理

| パラメータ | 既定値 | 説明 |
|-----------|---------|-------------|
| HearingSensitivity | 1 | 物音は範囲 × この値まで聞こえる（0 = 聞こえない） |
| WallDamping | 0.5 | 物音と Eyes の間の壁1枚ごとに残る強さ |
| NoiseAwareness | 0.5 | 最も強く聞こえた物音で足すメーターの値 |
| NoiseMaxAwareness | 0.7 | 物音（と仲間の声）だけではメーターはこの値より上がらない |
| AlertCall | Off | Alert になったときにほかのセンサーを呼ぶ |
| AlertCallRange | 12 | 声の届く範囲（ワールド単位） |
| AlertCallInterval | 0 | Alert の間に声を繰り返す間隔（秒。0 = Alert になるたびに1回） |
| AlertCallOnlyWhileSeen | On | 声を繰り返すのはターゲットが見えている間だけ |
| TouchRadius | 1 | 体からこれより近いターゲットにすぐ気づく（0 = OFF） |
| TouchAwareness | 1 | 接触で少なくともこの値になるメーター |
| UpdateInterval | 0.1 | 視覚の判定の間隔（秒。0 = 毎フレーム） |
| ShowGizmos | Selected | シーンビューのギズモを描くタイミング（Off／Selected／Always） |

### PerceptionSensor - イベント

| イベント（インスペクター／C#） | タイミング |
|-------|------|
| OnAwarenessChanged／AwarenessChanged | 状態が変わった（前の状態、新しい状態） |
| OnSuspicious、OnAlert、OnCalm | 状態が Suspicious／Alert／Unaware になった |
| OnTargetSpotted／TargetSpotted | ターゲットが見えた |
| OnTargetLost／TargetLost | ターゲットが見えなくなった |
| OnTargetForgotten／TargetForgotten | ターゲットを忘れた（最後に見た・触れたときから ForgetTime 後） |
| OnTargetTouched／TargetTouched | ターゲットが TouchRadius に入った |
| OnNoiseHeard／NoiseHeard | 物音を聞いた（位置、伝えられた位置、範囲、出どころ、強さ、壁の数、時刻） |
| OnAlertCall／AlertCalled | 仲間を呼んだ直後（見つけたターゲット） |
| OnFocusChanged／FocusChanged | FocusTarget が変わった（前の相手、新しい相手。どちらも null のことがある） |

### PerceptionSensor - プロパティとメソッド

| メンバー | 説明 |
|--------|-------------|
| State、Awareness | いまの状態とメーター（0..1）。すべてのターゲットのうち一番高いメーター |
| FocusTarget、Focus、CanSeeFocus | センサーが追うターゲット（「複数プレイヤー」）、それについて知っていること（PerceptionRecord）、見えているか |
| LastFocusChangeTime | FocusTarget が最後に変わった時刻 |
| Records、GetRecord(target)、CanSee(target) | センサーが知っていること全部（ターゲットごとに1つの PerceptionRecord） |
| HasMemory、LastSeenPosition、LastSeenTime | 追うターゲットの記憶 |
| TryGetLastKnownPosition(out pos) | 追うターゲットの最も新しい位置（最後に見た・触れた・聞いた位置） |
| ForgetTarget(target)、ResetAwareness() | 1つのターゲットを忘れる／すべてのメーターを 0 にして全部忘れる |
| HasHeardNoise、LastNoise、GetNoiseStrength(...)、HearNoise(...) | 最後に聞いた物音、物音がどれだけ強く聞こえるか、このセンサーに物音を聞かせる |
| LastAlertCallTime、LastAlertCallHeard | 最後に仲間を呼んだ時刻と、それを聞いたセンサーの数 |
| GetTouchDistance(target) | 体から最も近い点までの距離 |
| DistanceFactorAt(d)、AngleFactorAt(yaw)、TestPoint(...) | 速さの倍率と、1点の視覚の判定 |
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
| Priority | 0 | FocusMode が Priority のセンサーが使う。高いほど先に追われる |

### そのほかの型

| 型 | 説明 |
|------|-------------|
| PerceptionNoise.Emit(position, range, source) | 物音を出す。聞いたセンサーの数を返す。物音のたびに `Emitted` が呼ばれる |
| PerceptionNoise.Emit(position, range, source, reportedPosition) | 同じく物音を出し、聞いた側に覚えさせる位置を別に伝える（「仲間を呼ぶ声」） |
| PerceptionHeardNoise | 聞いた物音: Position、ReportedPosition、Range、Source、Strength、Obstacles、Time |
| PerceptionScheduler.MaxSensorsPerFrame | 1フレームの視覚の判定の上限（既定 32、0 = 上限なし） |
| PerceptionRecord | ターゲットごと: Awareness、IsSeen、VisibleFraction、Exposure、LastSeenTime、Points、HasMemory、LastSeenPosition、HasHeard、LastHeardPosition、LastHeardTime、IsTouching |
| PerceptionAwareness | Unaware、Suspicious、Alert |
| PerceptionFocusMode | Nearest、Priority、Awareness |

---

## 処理の重さ

Unity エディタ（Unity 2022.3 と 6.3、Intel Core i7-14700F）で、400体のセンサーが3点のターゲット1つを見る場合に測りました:

| 設定 | 1フレームの視覚の判定 | 1フレームの時間 |
|---------|------------------------|----------------|
| UpdateInterval 0（毎フレーム）、上限なし | 400 | 約 2.0 ms |
| UpdateInterval 0.1、MaxSensorsPerFrame 32（既定値） | 32 | 約 0.6 ms |

既定値でも、各センサーは毎秒約10回判定し、メーターは毎フレーム動きます。時間はセンサーの数 × ターゲットの数 × 点の数に応じて増えます。

---

## 制限事項

- 3D の物理だけが対象です。2D のゲーム（Collider2D、Physics2D）には対応していません
- 各点は Eyes からのレイ1本で調べます。壁の端のすぐ後ろの点は、まわりの体が少し見えていても、隠れているとみなします
- 視野は左右と上下の2つの角度で、丸い円すいではありません
- 物音は壁をまっすぐ通ります（壁1枚ごとに弱くなる）。角を回り込みません
- センサーが自分の更新の外で Alert になったとき（聞いた物音や声で）は、その声は次の更新で出します
- ネット対戦の同期は含みません。オンラインのゲームでは、センサーを1台で動かし、結果の同期はゲームのコードで行ってください（「複数プレイヤー」）

---

## デモシーン

`Demo_SmartPerception`（`Assets/LogicDevSupport/SmartPerception/Demo/`）を開いて Play を押します。壁に囲まれた場所を3人の見張りが巡回します。あなたは下の小さな隠れ場所にいる青いカプセルで、隣に2人目のプレイヤー（青緑）がいます。

- **操作**: **WASD／矢印キー** で移動（足音が聞こえる）、**Shift** を押している間は忍び足（音がしない・見つかりにくい）、**Space** で手をたたく（大きな音）、**Tab** で動かすプレイヤーを切り替え（もう1人はその場に止まる）、**R** でやり直し（見張りは全員あなたを忘れ、プレイヤーは初めの位置に戻る）、**M** でミニマップ、**G** でデバッグ表示、**T** で調整パネルの表示を切り替え
- **見張り**: 巡回路を歩きます（向かっている巡回ポイントまで地面に細い線）。色が状態を表します（灰色 = Unaware、黄色 = Suspicious、赤 = Alert）。Suspicious か Alert のときは止まり、最後に見た・聞いた方を向きます
- **追跡と捜索**: Alert の見張りは、追う相手のプレイヤーを追います。見失うと最後に見た場所へ行って見回し、巡回に戻ります
- **物音**: 歩くと足音がします。後ろで聞いた見張りは Suspicious になり、聞こえた場所へ行って見回し、巡回に戻ります。Suspicious にならない小さな音では、その場で音のほうを向くだけです
- **仲間を呼ぶ声**: 見張りは AlertCall が ON です（範囲 18、Alert であなたが見えている間は2秒ごと）。1人があなたを見つけると、近くの見張りがその声を聞き（デバッグ表示で赤い輪）、呼んだ見張りが最後にあなたを見た場所へ行って見回し、巡回に戻ります
- **2人のプレイヤー**: 1人を見張りの視野に入れてから、Tab で切り替えてもう1人を近づけたり、見張りにぶつけたりしてみてください。見張りは見えているうちの近い方を追い、触れたプレイヤーにはすぐ向き直ります
- **ミニマップ**: 右上のレーダー（M）に、壁、プレイヤー（矢印）、見張り（状態の色の点と視野の扇）が出ます。`PerceptionDemoMinimap` は公開されている API（`PerceptionSensor.All`、`State`、`GetViewFrame`、`ViewAngle`、`ViewDistance`、`PerceptionTarget.All`）だけで作ってあり、自分のミニマップの見本に使えます
- **物陰に隠れる**: 低い壁は足だけを隠します（見張りには体の一部が見えて、ゆっくり気づかれる）。高い壁は全身を隠します
- **接触**: 見張りに後ろからぶつかると、すぐ気づかれます

> 注: 巡回・追跡・物音や声の場所へ行くこと・見回し・ほかの見張りをよけることは、デモ専用のスクリプト（`Demo/Scripts/` の `PerceptionDemoPatrol`、`PerceptionDemoChase` など）が行います。センサーを使って AI が何をできるかの例で、製品には含まれません。消してもセンサーは動きます。**このスクリプトはまっすぐ進むだけで、経路探索をしません**。壁を回り込めない見張りは壁の手前で止まって見回し、巡回に戻ります。そのあと数秒（`ReturnTime`、6秒）は巡回路を歩き、その間は物音で出直さないので、壁の向こうで音が続いても壁の前で立ち止まったままにはなりません。実際のゲームでは、Unity の NavMesh などの経路探索で敵を動かし、センサーの位置（`TryGetLastKnownPosition` など）を行き先に使ってください。

> 注: HDRP では、デモの Play 中に、デモのカメラ（`PerceptionDemoExposure`）が固定の露出と簡単なグラデーションの空を足し、ほかのレンダーパイプラインと同じような見た目にします。Built-in Render Pipeline と URP では何もせず、あなたのシーンも変えません。

> 注: デモシーンはプリミティブの形だけでできています。Starter Assets を使ったイメージはアセットストアのページで確認できます（このパッケージには含まれません）。

---

## インポートの前に（重要）

プロジェクトに Smart Perception Lite（無料版）が入っている場合は、このパッケージをインポートする前に、そのフォルダ（`Assets/LogicDevSupport/SmartPerception/`）を消してください。2つの版は同じフォルダ・同じクラス名・同じファイルの識別子を使うため、Lite の上にインポートすると、フォルダの中に違う版のファイルが残ることがあります。Pro には Lite のすべてのクラス・設定・イベントが同じ名前であるので、入れ替えた後もシーンはそのまま動きます。

フォルダの中のファイル（デモシーンなど）を編集している場合は、消す前にバックアップを取ってください。

---

## 無料版への案内

まず無料版で試したい方は、Smart Perception Lite をどうぞ:

https://assetstore.unity.com/packages/slug/426870

---

もしこのアセットが役に立ちましたら、ぜひレビューを書いて応援していただけると開発の励みになります！

レビューはこちら（「Reviews」タブから投稿できます）:

https://assetstore.unity.com/packages/slug/426874
