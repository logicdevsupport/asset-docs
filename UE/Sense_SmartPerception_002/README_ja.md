# Smart Perception Pro（Unreal Engine）- 敵の視覚・聴覚・警戒度

敵や NPC に付ける「気づく」コンポーネントです。敵に **Perception Sensor**（`ULogicDevPerceptionSensorComponent`）、プレイヤーに **Perception Target**（`ULogicDevPerceptionTargetComponent`）を付けるだけで、敵が見る（視野・距離・遮蔽物・体の複数の点）、物音を聞く、触れたことに気づく、警戒度のメーターを上げる（Unaware → Suspicious → Alert）、最後に見た・聞いた位置を覚える、見つけたら仲間を呼ぶ、プレイヤーが複数いるときに追う相手を選ぶ、ができます。気づいたことは関数とイベント（Blueprint・C++）で AI に伝えます。敵を動かすこと（追跡・巡回・攻撃）はこのパッケージに含みません。あなたの AI（Blueprint、Behavior Tree、C++ など）がセンサーを読んで、何をするかを決めます。

---

## 特徴

- **体の一部が隠れた状態の視覚** - ターゲットごとに複数の点（既定は足・胸・頭）を調べます。低い壁の陰のプレイヤーは一部が見えていて、見えている量に応じてメーターが上がります
- **中心視野と周辺視野・距離** - 中心の角度の中では最も速く、視野の端や見える距離の端ほどゆっくりメーターが上がります
- **距離による上がり方の曲線** - 距離ごとのメーターの上がり方を曲線で描けます（例:「2m 以内は即アラート」）。既定は OFF（直線を使います）
- **3段階の警戒度メーター** - Unaware → Suspicious → Alert。保持時間・下がり方・しきい値の余裕があり、しきい値の付近で状態がちらつきません
- **記憶** - 最後に見た位置と時刻を、見失ってから決めた時間だけ覚えます
- **聴覚** - 物音は1行: `ULogicDevPerceptionNoise::EmitNoise(this, 位置, 範囲, 出どころ)`（Blueprint は **Emit Noise** ノード）。距離と遮蔽物で弱くなります。物音だけでは Alert になりません
- **接触** - すぐ近くまで来たターゲットには、どの向きからでもすぐ気づきます（後ろからぶつかる）
- **仲間を呼ぶ声（Alert Call）** - Alert になった敵が声を出し、近くの仲間は「呼んだ敵が最後にプレイヤーを見た場所」を聞いて見に来られます。声だけでは仲間は Alert にならないので、マップ全体に広がりません
- **複数プレイヤー（協力プレイ）** - 敵は、怪しいと思ったプレイヤーの中から追う相手を1人選びます: 近い順（既定）、優先度順、メーター順。見えている相手が音だけの相手より先、触られたらすぐその相手、保持時間で行ったり来たりしません
- **多数の敵** - 多くのセンサーの視覚の判定を複数のフレームに分けます（センサーごとの間隔と、1フレームの上限）。メーターは毎フレームなめらかに動きます。ほかの敵の体は既定では視界をさえぎらないので、集まった見張りどうしでプレイヤーを隠しません
- **イベント** - 状態の変化、見つけた／見失った／忘れた／触れた、物音を聞いた、仲間を呼んだ、追う相手が変わった、を Blueprint のイベント（`BlueprintAssignable`）で。C++ では `AddDynamic` でつなげます
- **デバッグ表示と調整パネル（開発用）** - 視野・視線・メーター・物音・声のゲーム画面への表示と、Play 中に値を変えられるパネル。Shipping と Test のビルドでは何も表示しません（描くコードごと無くなります）
- **ミニマップの見本** - デモに、公開されている関数だけで作った小さなレーダー（プレイヤー、状態の色と視野の扇つきの敵、壁）があります。自分のミニマップを作るときの出発点に使えます

---

## 動作環境

| 項目 | 内容 |
|------|------|
| Unreal Engine バージョン | UE 5.3 / 5.5 / 5.7 / 5.8 |
| 実装言語 | C++（プラグイン `LogicDevSmartPerception`。Runtime モジュール、Win64） |
| Blueprint 対応 | 設定は `BlueprintReadWrite`、関数は `BlueprintCallable`／`BlueprintPure`、イベントは `BlueprintAssignable`。Blueprint だけでも使えます |
| 入力システム | 依存なし。Enhanced Input などの入力用のプラグインは不要です（同梱デモのキーと、デバッグ表示の G・調整パネルの T は、プレイヤーコントローラーからキーの状態を直接読み取ります） |
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

詳細パネルでは、設定は **Smart Perception** の下の Vision、Awareness、Memory、Hearing、Alert Call、Touch、Focus、Performance、Gizmos にあります。詳細パネルの名前は、単語の間に空白が入り、先頭の `b` が取れた形で出ます（例: `bAlertCall` → **Alert Call**）。

### AI からセンサーを読む

Blueprint では、センサーのコンポーネントから **Get State**、**Get Awareness**、**Get Focus Target**、**Can See Focus**、**Try Get Last Known Position** などのノード（カテゴリ LogicDevSupport | SmartPerception）を呼びます。イベントは、コンポーネントを選んだ詳細パネルの Events の **+**（**On Awareness Changed**、**On Suspicious**、**On Alert**、**On Calm**、**On Target Spotted** など）か、**Bind Event to ...** で受け取ります。

C++ では、モジュールの `Build.cs` の依存に `"LogicDevSmartPerception"` を足してから使います。

```cpp
#include "LogicDevPerceptionSensorComponent.h"
#include "LogicDevPerceptionTargetComponent.h"

ULogicDevPerceptionSensorComponent* Sensor = GetOwner()->FindComponentByClass<ULogicDevPerceptionSensorComponent>();

FVector Pos;
if (Sensor->GetState() == ELogicDevPerceptionAwareness::Alert && Sensor->CanSeeFocus())
{
    // Sensor->GetFocusTarget()->GetOwner() を追う
}
else if (Sensor->GetState() != ELogicDevPerceptionAwareness::Unaware && Sensor->TryGetLastKnownPosition(Pos))
{
    // Pos へ行って見回す（最後に見た・触れた・聞いた位置）
}

// イベント。受け取る関数は UFUNCTION にしてください:
// void HandleAwarenessChanged(ELogicDevPerceptionAwareness Previous, ELogicDevPerceptionAwareness Current);
// void HandleSpotted(ULogicDevPerceptionTargetComponent* Target);
Sensor->OnAwarenessChanged.AddDynamic(this, &AMyAIController::HandleAwarenessChanged);
Sensor->OnTargetSpotted.AddDynamic(this, &AMyAIController::HandleSpotted);
```

**Get State** と **Get Awareness** は、すべてのターゲットのうち一番高いメーターです。**Get Focus Target** はセンサーが追う1人のターゲットで（「複数プレイヤー」）、記憶・仲間を呼ぶ声・デバッグ表示はこのターゲットを使います。

### 物音（1行）

```cpp
#include "LogicDevPerceptionNoise.h"

// 足音・銃声・ドア: 出どころ（Perception Target）を渡すと、見張りはそのターゲットのメーターを上げる
ULogicDevPerceptionNoise::EmitNoise(this, GetActorLocation(), 800.0f, MyTarget);

// 投げた石（出どころなし）: 見張りは知らせるだけ（On Noise Heard、Get Last Noise）
ULogicDevPerceptionNoise::EmitNoise(this, Stone->GetActorLocation(), 1000.0f, nullptr);
```

Blueprint では **Emit Noise** ノードです（戻り値は聞いたセンサーの数）。`範囲`（cm）× そのセンサーの **HearingSensitivity** 以内の、有効なセンサーすべてが聞きます。強さは物音の場所で1、その距離で0（間は直線）で、物音とセンサーの間の遮蔽物（ObstacleChannel をブロックするコンポーネント）1つごとに **WallDamping** を掛けます。聞いたときに、強さに比例して **NoiseAwareness** をメーターに足します（強さ1で NoiseAwareness そのまま）。物音だけではメーターは **NoiseMaxAwareness**（既定 0.7: Suspicious にはなるが Alert にはならない）より上がりません。見張りは聞いた場所を覚えます（**Try Get Last Known Position**）。強さが 0.01 未満の物音は聞こえません。

物音のイベント（**On Noise Heard**）の中から **Emit Noise** を呼んでもかまいません（物音を伝え直すときなど）。

### 接触

敵の体（敵の Actor の位置からセンサーまでの線分）から **TouchRadius**（既定 100 cm）より近いターゲット（チェックポイントのどれか）は、どの向きからでも、見えていなくても、メーターが少なくとも **TouchAwareness**（1 = すぐ Alert）になります。TouchRadius を 0 にすると OFF です。Actor の位置が体の中心（カプセルの中心など）の場合は、その位置からセンサーまでが「体」になります。

### 距離による上がり方の曲線

既定では、メーターはセンサーの目の前で最も速く、**ViewDistance** で **FarFactor** の速さになります（間は直線）。**bUseDistanceCurve** を ON にすると、代わりに **DistanceCurve** で倍率を自分で描けます（このとき FarFactor は灰色になり、使われません）:

- 横軸: 距離 ÷ ViewDistance（0 = センサーの位置、1 = ViewDistance）
- 縦軸: 上がり方の倍率（1 = ふつう、0 = 上がらない、1 より大きい = 速い。0 未満は 0 として扱います）
- 既定の曲線は FarFactor 0.4 と同じ直線（(0, 1) と (1, 0.4)）なので、ON にしても、曲線を直すまでは何も変わりません

**例:「2m 以内は即アラート」**（ViewDistance 1500、RiseRate 0.7）: 点 (0, 30)、(0.13, 30)、(0.14, 1)、(1, 0.4)。点を右クリックして補間を **Linear** にし、直線でつなぎます。195 cm（0.13 × 1500）以内ではメーターが30倍（毎秒 21）で上がり、60fps で約3フレームで Alert になります。それより遠くでは今までどおりです。

### 仲間を呼ぶ声（Alert Call）

**bAlertCall** を ON にすると、敵が Alert になったときに仲間を呼びます。声はふつうの物音で、出る場所はセンサーの位置、音の持ち主は見つけたターゲットです:

- **AlertCallRange**（cm。× 聞く側の HearingSensitivity。遮蔽物は WallDamping で弱くなる）以内の仲間は、そのターゲットのメーターを上げます（NoiseMaxAwareness まで）
- **声が伝えるのは、呼んだ敵が最後にターゲットを見た（触れた）位置です**（呼んだ敵の立っている位置ではありません。ターゲットの記憶が無いときは、呼んだ敵のセンサーの位置）。仲間はその位置を覚え、**Try Get Last Known Position** が返すので、AI はそこへ向かわせられます。声の届く範囲と遮蔽物は、呼んだ敵のセンサーの位置から測ります
- 自分の声には反応しません
- **AlertCallInterval**: Alert の間に声を繰り返す間隔（秒）。0 なら Alert になるたびに1回だけ
- **bAlertCallOnlyWhileSeen**（既定 ON）: 声を繰り返すのはターゲットが見えている間だけです。見失った敵は呼び続けません。Alert になったときの最初の1回は、どちらでも出ます
- **声は連鎖しません**: 声で上がるメーターは NoiseMaxAwareness までなので、仲間は Suspicious にはなっても Alert にはならず、仲間がさらに呼ぶことはありません。仲間が Alert になるのは、自分で見るか触れたときだけです（NoiseMaxAwareness を AlertThreshold 以上にすると、声で Alert になり、さらに呼ぶようになります）
- イベント: **On Alert Call**（見つけたターゲット）。**Get Last Alert Call Time** と **Get Last Alert Call Heard**（直前の声を聞いたセンサーの数）

自分で伝える位置を決めて声を出すときは、**Emit Noise Reported**（`ULogicDevPerceptionNoise::EmitNoiseReported(this, 位置, 範囲, 出どころ, 伝える位置)`）を使います。聞いた側は `伝える位置` を覚え（`FLogicDevPerceptionHeardNoise` の ReportedPosition）、範囲と遮蔽物は `位置` から測ります。

### 複数プレイヤー（追う相手の選び方）

プレイヤーが複数いるとき（協力プレイ）、各センサーはそのうちの1人を追います: **Get Focus Target**。選ぶのは、怪しいと思っている（メーターが **SuspiciousThreshold** 以上の）ターゲットだけです。誰もいなければ、メーターが一番高いターゲットです（このときも **FocusHoldTime** の間は替えないので、低いメーターの2人が抜きつ抜かれつしても行ったり来たりしません）。**FocusMode** で順番を選びます:

- **Nearest**（既定）: センサーから近い順（ターゲットの Actor の位置まで）
- **Priority**: **Priority**（Perception Target の設定）が高い順、同じなら近い順。例: 目標物を運んでいるプレイヤーの優先度を上げる
- **Awareness**: メーターが高い順、同じなら近い順（2人とも 100% のときは近い方）

3つに共通の決まり:

1. **見えている相手が先**: 見えている（触っている）ターゲットが、音や記憶だけのターゲットより先です。いまの相手が見えている間は、ほかの見えている相手にしか切り替えないので、後ろの足音で振り返りません。足音を立てた相手のメーターは上がり、位置も覚えます。いまの相手を見失ったら、そちらへ向き直ります
2. **触られたらすぐ**: センサーに触れたターゲットは、すぐに追う相手になります（保持時間・優先度より先）。触れている間はその相手のままです
3. **行ったり来たりしない**: 切り替えたら **FocusHoldTime** 秒（既定 2）はそのまま。その後も、はっきりよい相手にだけ切り替えます: **FocusSwitchMargin**（0.1）= 10% 以上近い、または Awareness ではメーターが 0.1 以上高い。優先度が高い相手、いまの相手を見失ったときの見えている相手、触った相手は、余裕なしで切り替わります

イベント: **On Focus Changed**（前の相手、新しい相手。どちらも空のことがあります）。**Get Last Focus Change Time**。

**ネット対戦**: センサーは、自分のいるマシンの中だけで動き、レプリケーションはしません。オンラインのゲームでは、センサーを1台（ふつうはサーバー）で動かし、結果（状態や追う相手）をゲームのコードでほかのプレイヤーに送ってください。同期は利用者のゲーム側で行います。

### 多数の敵（処理の分散）

センサーの視覚の判定（ライントレース）は **UpdateInterval** 秒ごと（既定 0.1、0 = 毎フレーム）に行います。センサーは順番に判定し、ワールドごとの上限（既定 32、0 = 上限なし）が1フレームに判定する数の上限です。上限は **Get Perception Subsystem** → **Set Max Sensors Per Frame** で変えられます（C++: `ULogicDevPerceptionSubsystem::Get(this)->SetMaxSensorsPerFrame(64)`）。判定と判定の間も、メーター・接触・物音は毎フレーム、直前の判定の結果で進みます。

当方の測定（UE 5.3）: センサー 400、UpdateInterval 0.1、上限 32 で、1フレームの視覚の判定は最大 32、各センサーは2秒間に 9〜10 回判定しました（上限から計算した 9.6 回と一致）。

見張りが集まると、前の見張りの体が後ろの見張りからプレイヤーを隠してしまいます。**bOtherSensorsBlock** が OFF（既定）のときは、ほかのセンサーの体（Perception Sensor を持つ Actor と、それにアタッチされた Actor）は、視界をさえぎらず、物音も弱めません。敵どうしで隠し合うようにしたいときは ON にします。

### デバッグ表示・調整パネル

次の2つは開発用のアクターです。Place Actors のウィンドウ（All Classes）で「Perception」と検索し、レベルに1つずつ置きます。**Shipping と Test のビルドでは何も表示しません（描くコードごと無くなります）**。センサーはこれらに依存しないので、置かなくても、消してもかまいません。同梱デモのレベルには最初から置いてあります。

- **Perception Debug View (Development only)**（`ALogicDevPerceptionDebugViewActor`）: センサーごとに、視野（視野角・中心視野（点線）・距離）、各点への線（緑 = 見える、赤 = 遮蔽物まで）、頭上のメーター（目盛り = しきい値）、最後に見た位置、最後に聞いた位置、Suspicious／Alert のときの最後に分かっている位置への地面の点線（AI が見に行く先）、接触の半径、物音（広がる輪と、聞いたセンサーへの線。橙 = 聞こえた、点線 = 遮蔽物で弱まった）、仲間を呼ぶ声（届く範囲の赤い輪）、左上の状態の表（1行 = 1センサー。列は Sensor・State・Meter・Target・Sight・Memory・Noise）。**G** で表示を切り替えます
- **Perception Tuning Panel (Development only)**（`ALogicDevPerceptionTuningPanelActor`）: 右下の小さなパネルで、Play 中に1つずつセンサーの主な設定（View Distance、View Angle、Vertical Angle、Central Angle、Peripheral Factor、Far Factor、Rise Rate、Decay Rate、Decay Delay、Suspicious Threshold、Alert Threshold、Forget Time、Hearing Sensitivity、Touch Radius）を変えられます（`<` `>` でセンサーを選ぶ）。デバッグ表示の項目も切り替えられます。**T** で表示を切り替えます
- **UI モード**: パネルが画面にある間は、クリックできるようにマウスカーソルを出します（入力は Game and UI。ゲームのキーとマウスのボタンもそのまま効きます）。T で隠すと、ゲームのマウスの設定に戻します。**bUIModeWhileShown** を OFF にすると、パネルはカーソルにも入力のモードにも触りません。自分のカメラのマウス操作は、`IsAnyPanelInUIMode`（静的。パネル1つなら `IsInUIMode`）が true の間は止めてください。イベント `OnUIModeEntered`／`OnUIModeExited` もあります。Shipping と Test のビルドでは、UI モードには決してなりません

使い方の詳細は「調整の仕方」の章を見てください。

**エディタのビューポートの表示（Play していないとき）**: センサーは、視野（視野角・中心視野・距離）と接触の半径を、エディタのビューポートに線で表示します。**ShowGizmos** で表示のタイミングを選びます: `Selected`（既定。Actor を選んでいる間）、`Always`、`Off`。ビューポートの **Realtime** が ON のときに出ます。Play 中・パッケージしたゲームには出ません。

---

## 判定と計算

- **点が見える**: センサーから **ViewDistance** 以内、左右 **ViewAngle**・上下 **VerticalViewAngle** 以内（どちらもセンサーの前の向きから測る）、間に遮蔽物が無い
- **見えている**: 見える点の数 ≥ min(**RequiredVisiblePoints**, 点の数)。**bDetectable** が OFF のターゲットは見えません
- **メーターの上がり方**（毎秒）= RiseRate × Exposure × ターゲットの **Visibility**。Exposure = Σ（見える点の 角度の係数 × 距離の係数）÷ 点の数
  - 角度の係数: **CentralAngle** 以内は 1、ViewAngle で **PeripheralFactor**（間は直線）
  - 距離の係数: 0 cm で 1、ViewDistance で **FarFactor**（間は直線）。または距離の曲線
- **下がり方**: 最後に気づいて（見た・聞いた・触れた）から **DecayDelay** 秒はそのまま、その後 **DecayRate**／秒
- **状態**: 上がるときはしきい値ちょうどで、下がるときは しきい値 − **CalmMargin** を下回ったとき（メーターが 0 になったら必ず Unaware）
- **記憶**: 見えている間（と触れている間）は位置を更新します。見えなくなってから **ForgetTime** 秒で忘れます（**On Target Forgotten**）。聞いた位置も同じ秒数だけ覚えます

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
| `bUseDistanceCurve` | Off | FarFactor の代わりに DistanceCurve を使う |
| `DistanceCurve` | 直線 1 → 0.4 | 距離 ÷ ViewDistance ごとの倍率（「距離による上がり方の曲線」） |
| `RequiredVisiblePoints` | 1 | 見えているとみなすのに必要な、見える点の数（1〜8） |
| `ObstacleChannel` | Visibility | 視線を遮り、物音を弱くするトレースチャンネル（センサーとターゲット自身の Actor、オーバーラップだけのコンポーネントは遮らない） |
| `bOtherSensorsBlock` | Off | ほかのセンサーの体（Perception Sensor を持つ Actor と、それにアタッチされた Actor）が視線を遮り、物音を弱くするか |

### Perception Sensor - 警戒度と記憶（Awareness、Memory）

| パラメータ | 既定値 | 説明 |
|-----------|---------|-------------|
| `RiseRate` | 0.7 | 中心視野の目の前で全身が見えているときの、毎秒のメーターの上がり（1 = 1秒で Alert） |
| `DecayRate` | 0.2 | DecayDelay 秒何も気づかなかった後の、毎秒のメーターの下がり |
| `DecayDelay` | 2 | 最後に気づいて（見た・聞いた・触れた）から、メーターが下がり始めるまでの秒数 |
| `SuspiciousThreshold` | 0.3 | 状態が Suspicious になるメーターの値 |
| `AlertThreshold` | 1 | 状態が Alert になるメーターの値（SuspiciousThreshold より小さい値は SuspiciousThreshold になる） |
| `CalmMargin` | 0.1 | しきい値よりこれだけ下がったとき（または 0 になったとき）だけ、状態が下がる |
| `ForgetTime` | 10 | 見失ってから、最後に見た・聞いた位置を覚えている秒数 |

### Perception Sensor - 追う相手（Focus）

| パラメータ | 既定値 | 説明 |
|-----------|---------|-------------|
| `FocusMode` | Nearest | 怪しいと思っているターゲットから追う相手を選ぶ順番: Nearest、Priority、Awareness |
| `FocusHoldTime` | 2 | 切り替えた後、追う相手を変えない秒数（触られたとき・いまの相手を見失ったときは待たない） |
| `FocusSwitchMargin` | 0.1 | 切り替えるのに必要な差: 0.1 = 10% 以上近い、または（Awareness）メーターが 0.1 以上高い |

### Perception Sensor - 聴覚・仲間を呼ぶ声・接触・処理・表示（Hearing、Alert Call、Touch、Performance、Gizmos）

| パラメータ | 既定値 | 説明 |
|-----------|---------|-------------|
| `HearingSensitivity` | 1 | 物音は範囲 × この値まで聞こえる（0 = 聞こえない） |
| `WallDamping` | 0.5 | 物音とセンサーの間の遮蔽物1つごとに残る強さ |
| `NoiseAwareness` | 0.5 | 強さ1で聞いた物音で足すメーターの値（弱いほど比例して少ない） |
| `NoiseMaxAwareness` | 0.7 | 物音（と仲間の声）だけではメーターはこの値より上がらない |
| `bAlertCall` | Off | Alert になったときにほかのセンサーを呼ぶ |
| `AlertCallRange` | 1200（cm） | 声の届く範囲 |
| `AlertCallInterval` | 0 | Alert の間に声を繰り返す間隔（秒。0 = Alert になるたびに1回） |
| `bAlertCallOnlyWhileSeen` | On | 声を繰り返すのはターゲットが見えている間だけ |
| `TouchRadius` | 100（cm） | 体からこれより近いターゲットにすぐ気づく（0 = OFF） |
| `TouchAwareness` | 1 | 接触で少なくともこの値になるメーター |
| `UpdateInterval` | 0.1 | 視覚の判定の間隔（秒。0 = 毎フレーム） |
| `ShowGizmos` | Selected | エディタのビューポートに視野と接触の半径を描くタイミング（Off／Selected／Always） |

### Perception Sensor - イベント

詳細パネルの Events、または **Bind Event to ...**。C++ は `AddDynamic`。

| イベント | タイミング |
|-------|------|
| `OnAwarenessChanged` | 状態が変わった（前の状態、新しい状態） |
| `OnSuspicious`、`OnAlert`、`OnCalm` | 状態が Suspicious／Alert／Unaware になった |
| `OnTargetSpotted` | ターゲットが見えた |
| `OnTargetLost` | ターゲットが見えなくなった |
| `OnTargetForgotten` | ターゲットを忘れた（最後に見た・触れたときから ForgetTime 後） |
| `OnTargetTouched` | ターゲットが TouchRadius に入った |
| `OnNoiseHeard` | 物音を聞いた（`FLogicDevPerceptionHeardNoise`: Position、ReportedPosition、Range、Source、Strength、Obstacles、Time） |
| `OnAlertCall` | 仲間を呼んだ直後（見つけたターゲット） |
| `OnFocusChanged` | 追う相手が変わった（前の相手、新しい相手。どちらも空のことがある） |

### Perception Sensor - 関数

| 関数 | 説明 |
|--------|-------------|
| `GetState()`、`GetAwareness()` | いまの状態とメーター（0..1）。すべてのターゲットのうち一番高いメーター |
| `GetFocusTarget()`、`GetFocusRecord(Out)`、`CanSeeFocus()` | センサーが追うターゲット（「複数プレイヤー」）、それについて知っていること（`FLogicDevPerceptionRecord`）、見えているか |
| `GetLastFocusChangeTime()` | 追う相手が最後に変わった時刻 |
| `GetRecords()`、`GetRecord(Target, Out)`、`CanSee(Target)`、`GetTimeSinceSeen(Record)` | センサーが知っていること全部（ターゲットごとに1つの `FLogicDevPerceptionRecord`） |
| `HasMemory()`、`GetLastSeenPosition()`、`GetLastSeenTime()` | 追うターゲットの記憶 |
| `TryGetLastKnownPosition(Out)` | 追うターゲットの最も新しい位置（最後に見た・触れた・聞いた位置） |
| `ForgetTarget(Target)`、`ResetAwareness()` | 1つのターゲットを忘れる（メーターは残る）／すべてのメーターを 0 にして全部忘れる |
| `HasHeardNoise()`、`GetLastNoise()`、`GetNoiseStrength(...)`、`HearNoise(...)` | 最後に聞いた物音、物音がどれだけ強く聞こえるか、このセンサーに物音を聞かせる |
| `GetLastAlertCallTime()`、`GetLastAlertCallHeard()` | 最後に仲間を呼んだ時刻と、それを聞いたセンサーの数 |
| `GetTouchDistance(Target)` | 体から最も近い点までの距離（cm） |
| `GetViewFrame(Origin, Forward, Up)`、`GetViewAngles(...)` | 見る位置と向き、ある方向の左右・上下の角度 |
| `DistanceFactorAt(cm)`、`AngleFactorAt(Yaw)`、`TestPoint(...)`、`NextState(...)` | 速さの倍率、1点の視覚の判定、メーターの値から次の状態 |
| `GetLastVisionTestTime()`、`GetVisionTestCount()` | 最後に視覚の判定をした時刻と、判定した回数 |
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
| `Priority` | 0 | FocusMode が Priority のセンサーが使う。高いほど先に追われる |

関数: `GetCheckPoints(Out)`（いまの点のワールド位置）、static `GetAllTargets(WorldContext)`。

### そのほか

| 名前 | 説明 |
|------|-------------|
| `ULogicDevPerceptionNoise::EmitNoise(WorldContext, Position, Range, Source)` | 物音を出す（Blueprint: **Emit Noise**）。聞いたセンサーの数を返す |
| `ULogicDevPerceptionNoise::EmitNoiseReported(WorldContext, Position, Range, Source, ReportedPosition)` | 同じく物音を出し、聞いた側に覚えさせる位置を別に伝える（「仲間を呼ぶ声」） |
| `ULogicDevPerceptionSubsystem` | ワールドごとのセンサーとターゲットの一覧。`GetPerceptionSubsystem`、`SetMaxSensorsPerFrame`／`GetMaxSensorsPerFrame`（既定 32、0 = 上限なし）、`GetSensorCount`、`GetTestsThisFrame` |
| `FLogicDevPerceptionRecord` | ターゲットごと: Target、Awareness、bIsSeen、VisibleFraction、Exposure、LastSeenTime、Points、bHasMemory、LastSeenPosition、MemoryTime、bHasHeard、LastHeardPosition、LastHeardTime、bIsTouching |
| `ELogicDevPerceptionAwareness` | Unaware、Suspicious、Alert |
| `ELogicDevPerceptionFocusMode` | Nearest、Priority、Awareness |
| `ELogicDevPerceptionPointStatus` | Visible、OutOfRange、OutOfAngle、Blocked |

### Unity 版との違い

Unity 版の Smart Perception Pro と同じ機能・同じ既定値です。違いは次のとおりです。

| 項目 | Unity 版 | UE 版 |
|---|---|---|
| 目の位置 | `Eyes`（Transform。空なら自身） | センサーのコンポーネントそのもの（位置と +X の向き） |
| 距離の単位 | m（ViewDistance 15、AlertCallRange 12、TouchRadius 1、Height 1.8） | cm（1500、1200、100、180） |
| 遮蔽物 | `ObstacleLayers`（レイヤー）、Trigger は遮らない | `ObstacleChannel`（トレースチャンネル）、オーバーラップだけのコンポーネントは遮らない |
| 名前 | `PerceptionSensor`、`State`、`AwarenessChanged` など | `ULogicDevPerceptionSensorComponent`、`GetState()`、`OnAwarenessChanged` など。真偽の設定は `b` が付く（`bAlertCall` など） |
| イベント | C# のイベントと UnityEvent | Blueprint のイベント（dynamic multicast delegate） |
| 1フレームの上限 | `PerceptionScheduler.MaxSensorsPerFrame` | `ULogicDevPerceptionSubsystem` の `SetMaxSensorsPerFrame` |
| Play 中の値を残す | パネルの Record values | エンジンの Keep Simulation Changes（「調整の仕方」） |
| デバッグ表示・パネルの付け方 | メニューの Add Debug Tools to Scene | 2つのアクターをレベルに置く |

---

## 調整の仕方（Play 中に値を変える）

センサーの値もデバッグ表示の設定も、Play 中に変えるとすぐ効きます。変え方は2つあります。

### 1. 詳細パネルで変える（エディタの Play in Editor）

1. Play を押して、ゲームを動かしたままにします。マウスがゲームの画面に取られているときは、**Shift+F1** でマウスを戻すか、**F8** で操作をエディタに戻します（Eject）。**Simulate** で動かしてもかまいません。
2. **アウトライナー**で変えたいものを選び、**詳細パネル**で値を変えます。
   - **見張りのセンサー**: 見張り（同梱デモでは `Guard_A` など）を選び、詳細パネルの上のコンポーネントの一覧で **Sensor** を選びます。**Smart Perception** の下（▶ で開く）に Vision（視野の距離・角度など）、Awareness（上がり方・下がり方・しきい値）、Memory、Hearing、Touch などがあります。変えた値はすぐ使われ、メーターは消えません。
   - **デバッグ表示**: アウトライナーで（フォルダの外にある） **Perception Debug View** を選びます。詳細パネルの **Debug View**: 表示する／しない、切り替えのキー（Toggle Key、既定は G）。**What to Draw**: 視野・線・メーター・記憶・物音・接触・状態の表をそれぞれ出す／出さない。**Which Sensors**: 調整パネルで選んだセンサーだけ・画面に映っているセンサーだけ。**Debug Look**: 物音を表示する秒数、表の行数、メーターの高さ。
   - **調整パネル**: アウトライナーで（フォルダの外にある） **Perception Tuning Panel** を選びます。詳細パネルの **Tuning Panel**: 始めは隠す（Start Hidden）、切り替えのキー（Toggle Key、既定は T）、パネルを出している間はマウスカーソルを出す（UI Mode While Shown）。
3. **Play を止めると、Play 中に変えた値は元に戻ります。** 残したいときは、F8 で Eject した状態か Simulate 中に、値を変えたアクターをビューポートかアウトライナーで右クリックし **Keep Simulation Changes** を選びます（**K キー**でも同じです）。その後、レベルを保存してください。

### 2. 画面上の調整パネル（Play in Editor でもゲームでも）

- **T** で右下のパネルを出す／隠す。出している間はマウスカーソルが出て、ボタンを押せます（キーボードでの移動などもそのまま使えます）。
- ボタン: **`<` `>`** 値を変えるセンサーを選ぶ、**`-` `+`** 値を1段階下げる／上げる、**`o`** 表示の項目を On／Off にする。
- **G** でデバッグ表示を出す／隠す。
- パネルで変えた値も、Play を止めると元に戻ります（残すときは上の Keep Simulation Changes。または詳細パネルに書き写してください）。
- **Play in Editor では、Play を押した直後はゲームの画面にキーが届かないことがあります。一度ゲームの画面をクリックしてから T を押してください。** ゲームの画面が小さいときは、パネルは収まる大きさまで小さくなります。

デバッグ表示と調整パネルは開発用です。Shipping と Test のビルドでは何も表示しません（コードが無くなります）。

---

## 制限事項

- 各点は、センサーからのライントレース1本で調べます。遮蔽物の端のすぐ後ろの点は、まわりの体が少し見えていても、隠れているとみなします
- 視野は左右と上下の2つの角度で、丸い円すいではありません
- 物音は遮蔽物をまっすぐ通ります（遮蔽物1つごとに弱くなる）。角を回り込みません。1つの壁が複数のコンポーネントでできているときは、それぞれを1つと数えます
- センサーが自分の更新の外で Alert になったとき（聞いた物音や声で）は、その声は次の更新で出します
- ネット対戦の同期（レプリケーション）は含みません。オンラインのゲームでは、センサーを1台で動かし、結果の同期はゲームのコードで行ってください（「複数プレイヤー」）
- デバッグ表示・調整パネルは開発用です。Shipping と Test のビルドでは表示されません

---

## デモシーンについて

同梱のデモレベル `L_SmartPerception_Demo`（プラグインの `Content/Demo/`。コンテンツブラウザでは、設定の **Show Plugin Content** を ON にすると、プラグイン `LogicDevSmartPerception` のフォルダの `Demo` に出ます）を開いて Play を押します。壁に囲まれた場所を3人の見張りが巡回します。あなたは下の小さな隠れ場所にいる青い円柱（`Player`）で、隣に2人目のプレイヤー（青緑、`Player_2`）がいます。

- **操作**: **WASD／矢印キー** で移動（足音が聞こえる）、**Shift** を押している間は忍び足（遅く、音がしない、見つかりにくい）、**Space** で手をたたく（大きな音）、**Tab** で動かすプレイヤーを切り替え（もう1人はその場に止まる）、**R** でやり直し（見張りは全員あなたを忘れ、プレイヤーは初めの位置に戻る）、**M** でミニマップ（始めは出ています）、**G** でデバッグ表示、**T** で調整パネル（始めは隠れています。隠れている間は右下に小さく「T: Tuning」と出ます）の表示を切り替え
- **見張り**: 巡回路を歩きます（向かっている巡回ポイントまで地面に細い線）。色が状態を表します（灰色 = Unaware、黄色 = Suspicious、赤 = Alert）。Suspicious か Alert のときは止まり、最後に見た・聞いた方を向きます
- **追跡と捜索**: Alert の見張りは、追う相手のプレイヤーを追います。見失うと最後に見た場所へ行って見回し、巡回に戻ります
- **物音**: 歩くと足音がします。後ろで聞いた見張りは Suspicious になり、聞こえた場所へ行って見回し、巡回に戻ります。Suspicious にならない小さな音では、その場で音のほうを向くだけです
- **仲間を呼ぶ声**: 見張りは bAlertCall が ON です（範囲 1800 cm、Alert であなたが見えている間は2秒ごと）。1人があなたを見つけると、近くの見張りがその声を聞き（デバッグ表示で赤い輪）、呼んだ見張りが最後にあなたを見た場所へ行って見回し、巡回に戻ります
- **2人のプレイヤー**: 1人を見張りの視野に入れてから、Tab で切り替えてもう1人を近づけたり、見張りにぶつけたりしてみてください。見張りは見えているうちの近い方を追い、触れたプレイヤーにはすぐ向き直ります
- **ミニマップ**: 右上のレーダー（M）に、操作しているプレイヤーを中心に、壁、プレイヤー（矢印）、見張り（状態の色の点と視野の扇）が出ます。センサーの部分は公開されている関数（`GetAllSensors`、`GetState`、`GetViewFrame`、`ViewAngle`、`ViewDistance`、`GetAllTargets`）だけで描いてあり、自分のミニマップの見本に使えます（`ALogicDevPerceptionDemoHUD`）
- **物陰に隠れる**: 低い壁は足だけを隠します（見張りには体の一部が見えて、ゆっくり気づかれる）。高い壁は全身を隠します
- **接触**: 見張りに後ろからぶつかると、すぐ気づかれます

> 注: 巡回・追跡・物音や声の場所へ行くこと・見回し・ほかの見張りをよけることは、デモ専用のクラス（`ULogicDevPerceptionDemoPatrolComponent`、`ULogicDevPerceptionDemoChaseComponent` など、名前に `Demo` が付くもの）が行います。センサーを使って AI が何をできるかの例で、製品の機能ではありません。**このクラスはまっすぐ進むだけで、経路探索をしません**。壁を回り込めない見張りは壁の手前で止まって見回し、巡回に戻ります。実際のゲームでは、Navigation Mesh（AI Move To など）で敵を動かし、センサーの位置（**Try Get Last Known Position** など）を行き先に使ってください。

> 注: デモシーンはプリミティブの形だけでできています。マネキンを使ったイメージは Fab の商品ページで確認できます（このパッケージには含まれません）。

---

## 困ったとき

- **RTX 40 番台などで、同梱デモを開いた直後にエディタが落ちる（GPU のクラッシュ）とき**は、まず GPU のドライバを更新してください。直らなければ `Config/DefaultEngine.ini` の `[SystemSettings]` に `r.RDG.AsyncCompute=0` を足してください（UE の影と光の機能の、エンジンかドライバ側の不具合です。プラグインのコードは関係しません）。
- **Play in Editor で、Play を押した直後に T・G などのキーが効かないとき**: 同梱デモでは、Play の始めにキーボードの入力先を自動でゲームの画面に向けています（デモのプレイヤーコントローラー `ALogicDevPerceptionDemoPlayerController`）。自分のレベルで効かないときは、一度ゲームの画面をクリックしてからキーを押すか、エディタの環境設定の **Level Editor > Play > Game Gets Mouse Control** を ON にしてください（UE の初期値は OFF で、マウスで Play を押した直後はキーがエディタ側に残ります）。
- **センサーが何も見ないとき**は、センサーの位置と前の向き（+X。ビューポートで Actor を選ぶと視野の線が出ます）、ターゲットの **bDetectable** と **Visibility**（0 だと気づかれません）、間にあるものが **ObstacleChannel** をブロックしていないかを確かめてください。
- **コンテンツブラウザにデモのレベルが出ないとき**は、コンテンツブラウザの設定で **Show Plugin Content** を ON にしてください。

---

## 無料版（Smart Perception Lite）からの入れ替え

無料版の Smart Perception Lite（視覚・警戒度・イベントだけ）もあります。Lite で作ったレベルと Blueprint は、Pro に替えても設定とイベントのつながりがそのまま残ります（同じプラグイン名・クラス名・設定の名前）。

https://www.fab.com/listings/394b12a0-78f8-470e-bcc5-0218e851a571

**すでに Lite を入れている場合は、Pro を入れる前に必ず Lite を削除してください**（エディタを閉じて、プロジェクトの `Plugins/LogicDevSmartPerception/` に置いた場合はそのフォルダを、エンジンに入れた場合はエンジンの側の Lite を削除）。2つの版は同じプラグイン名（`LogicDevSmartPerception`）・同じクラス名を使うため、両方があるとプラグインがぶつかり、上から重ねると Lite のファイルが残ることがあります。Lite のプラグインの中のファイル（デモのレベルなど）を編集している場合は、先にバックアップを取ってください。

---

もしこのアセットが役に立ちましたら、ぜひ Fab の商品ページでレビューを書いて応援していただけると開発の励みになります！

https://www.fab.com/listings/ab734007-2ae6-4937-927d-8da8e1eb9e2f
