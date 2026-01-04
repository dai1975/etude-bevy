https://taintedcoders.com/bevy/tutorials/pong-tutorial
https://docs.rs/bevy/latest/bevy/

# Rust プロジェクト作成
`rustup udpate` しとく。とりあえず stable で。2025-12-08 の rustc 1.92.0 のようだ。

`cargo new pong` から、 `cargo add bevy`
2025-12-30 現在 0.17.3. チュートリアルは 2025-09-30 で 0.17 のようだ。問題ないだろう。

とりあえず `cargo build && cargo run`. 
これで bevy 関連と一緒にビルドされ、実行される。
ok.

# boilerplate
## App new

```
use bevy::prelude::*;
fn main() {
  App::new().run();
}
```

prelde で良く使う名前がインポートされる、と。
でメインループは App オブジェクトを作って run を呼ぶようだ。

## plugin
Bevy はほとんどの機能はプラグイン化されているようだ。
bevy::prelude::DefaultPlugins に良く使うプラグインセットが入っているのでこれを登録する。
Diagnostics, DlssInit, FrameCount, Hierarchy, Input, PanicHandler, ScheduleRunner, TaskPool, Time, Transform
がチュートリアルで挙げられている。

あと、features で使えるようになるプラグイン群もあるとのこと。多くはデフォルトで feature 有効になっている。
Asset, Audio など

というわけで DefaultPlugins を追加。

```
App::new().add_plugins(DefaultPlugins).run();
```

## scheduling a system
system という仕組みをスケジューラに登録することで、ゲームループ内で実行できるようになるらしい。
システムは、ECS エンティティに対する操作ということかな? Rust の関数で書けるようだ。

```
fn hello_world() {
  println!("Hello, world");
}
fn main() {
  let mut app = App::new();
  app.add_plugins(...);
  app.add_system(Startup, hello_world);
  app.run();
}
```

あとでコードを示す時に分けられるよう、main 内は分割した。

add_system の第一引数は、呼び出されるタイミングを表す定数なのかな?
まぁ開始時に呼ばれるということは分かる。

cargo run すると、window が開いて、コンソールにログが表示されて、"Hello, world" も表示されてるな。

Startup 以外に、Update と FixedUpdate が良く使われるそうだ。
Update は毎ループ、FixedUpdate は `every X seconds` で、秒数?
毎フレーム処理が不要なのなら FixedUpdate を使っておいた方が良いとのこと。

API ドキュメント読むと、App#add_systems は

```
 pub fn add_systems<M>(
     &mut self,
     schedule: impl ScheduleLabel,
     systems: impl IntoScheduleConfigs<Box<dyn System<In = (), Out = ()>>, M>,
 ) -> &mut App
```

第一引数は ScheduleLabel trait のようだ。
第二引数は IntoScheduleConfigs ということで、意味付けは関数ではなくて config なの? in-out があるようでフィルタぽい定義だが。

ScheduleLabel は bevy::ecs::schedule::ScheduleLabel にあり、

```
pub trait ScheduleLabel:
    Send
    + Sync
    + Debug
    + DynEq
    + DynHash {
    // Required method
    fn dyn_clone(&self) -> Box<dyn ScheduleLabel>;

    // Provided method
    fn intern(&self) -> Interned<dyn ScheduleLabel>
       where Self: Sized { ... }
}
```

ふむ。dyn_clone など、実装曖昧な汎用型な感じね。
intern が特徴か。たぶん実行されるタイミングを示すんだろう。

IntoScheduleConfigs の方は DI として説明されいている

## Dependency injection / SystemParam
https://taintedcoders.com/bevy/building-bevy

IntoScheduleConfigs については触れられてないな。
add_systems の第二引数が関数であるとして、その関数の引数が SystemParam trait であるらしい。

[SystemParam の API]( https://docs.rs/bevy/latest/bevy/ecs/system/trait.SystemParam.html )は長いので省略するが、State と Item という型パラメータを持つ。
State は永続保管するところで、Item は SystemParam の作成時に取得する型。
ループのたびに、Item は再利用するが新たな StateParam を作るのかな。
で、それを関数に渡すと。
これでライフタイムのあれこれを回避してるのかしら。

でだ。具体的な Item の型が Bevy 実装時には固定されず、アプリ開発時や実行時に決まるので、これは DI だよねという話
DI という名前は大げさで構えてしまうが、要するにインターフェースを使うことで具体的な型を決め打ちしない設計手法ね。

たぶん `impl MutFn(T) for SystemParam<Item=T>` しているのだろう。





# Creating a boll

```
fn spawn(ball(mut commands: Commands) {
  commands.spawn_empty();
}
fn main() {
  ...
  app.add_systems(Startup, spawn_ball)
}
```

Comamands を取る引数を system 関数として与えている。
System 関数は上で見た DI の仕組みで引数は色々取れる、と。

spawn_empty() は Entity を生成する。
Bevy は内部に Entity 用の巨大配列を持っていて、Entity の実態はそのインデックスらしい。
参照ぽいがコピーとか自由にできるんだろう。lifetime も 'static なのかな?

entity は Bevy world 本体ではなく Commands に対して生成している。
つまり "entity を作るコマンド" を生成しているのかな。
world へ直接追加すると排他的な mut アクセスが必要になっちゃって効率悪いので、
Commands を挟み、そこでよしなにやってくれるのだろう。

# Creating a camera

```
fn spawn_camera(mut commands: Commands) {
    commands.spawn_empty().insert(Camera2d);
}
```

同様に Entity を作り、Camera2d component を insert している。
Bevy は OOP ではなく関数型なので、型は継承とかしない。
component という形で、Entity オブジェクトへ動的に mixin してるのかな。
insert までまとめた Camera2d 型を作りたくなるが、それは流儀に反するかしら?

実装的には、component 配列の entity id の index へ Component インスタンスを配置する、ことで実現するようだ。
これだと型にはできんな。
関数にはできる、というか既にあるみたい。

```
commands.spawn(Camera2d);
```

spawn は引数にタプルで複数の component を追加できるようだ。
```
commands.spawn((Camera2d, Transform::from_xyz(0., 0., 0.)));
```

メモリアクセスが気になる。
Entity 配列と Component(Camera2d) 配列がそれぞれあるようなこと書いてあるが、実体は単一配列で要素が分割されているんじゃないかな。
これだと Entity と Component 群のメモリ位置が近接するのでアクセスが早い。

Camera2d は裏でいくつかの必須コンポーネントも追加している。
https://docs.rs/bevy_camera/0.17.3/src/bevy_camera/components.rs.html#16

```
/// A 2D camera component. Enables the 2D render graph for a [`Camera`].
#[derive(Component, Default, Reflect, Clone)]
#[reflect(Component, Default, Clone)]
#[require(
    Camera,
    Projection::Orthographic(OrthographicProjection::default_2d()),
    Frustum = OrthographicProjection::default_2d().compute_frustum(&GlobalTransform::from(Transform::default())),
)]
pub struct Camera2d;
```

コンポーネントの実装はこうやる感じね。
ソフトウェア設計的には多重継承と似たようなものか。実装は違うけど。
複数 require された場合は一個だけ追加になるそうだ。
require は Bevy 0.15 から。2024-11-29 だから一年前か。古いドキュメント読む時注意。

最後にこの spawn_camera を登録。
add_systems にはタプルで複数の system を渡せるので、
```
app.add_systems(Startup, (spawn_ball, spawn_camera));
```
でよい。

# Drawing a ball
描画機能は RenderPlugins にある。これは DefaultPlugins に入っている。
RenerPlugins は描画に必要ないくつかの system を登録している。

とりあえず必要なのは 3つ:
 1. Transform. 位置を与える。
 2. Mesh2d. shape 担当。
 3. MeshMaterial2d. レンダラに shape の塗り方を指定する。
これらは RenderPlugins では分離されたコンポーネントとして取り扱われている。

## Transform
Transform コンポーネントは、3つの情報を取り扱う。
 1. Translation: 座標
 2. Rotation: where it is pointing. 回転のことだと思うが。
 3. Scale: 大きさ

Transform で、ボールの位置をそのまま扱うこともできるが、やらない方が良い。
第一に、このチュートリアルで作るゲームで Rotation, Scale は一定なので、定数とする。
第二に、 右や左はウィンドウのスケールに依存する(?)

ウィンドウサイズに依存しないゲーム内の論理位置を使うのが一案。
それをウィンドウ上の物理的な位置に変換することで、ディスプレイ設定に依存しないようにする。

ただそれよりも、自前で幾何学的な位置をコンポーネントとし、system で Transform に投影する方が良い。
```
#[derive(Component)]
struct Position(Vec2);
```

エンティティがボールであると区別できるように Ball component も定義しておく。
区別するだけなら何もない空構造体でよい。
```
#[derive(Component)]
struct Ball
```

Ball は Position を持つわけで、OOP のように継承やメンバとして Position を持たせるのはダメ。
```
#[derive(Ball)]
struct Ball {
  position: Position
}
```
これだと Position が「総称的(generically)」では無くなるそうだ...用語はピンと来ないが言わんとすることは分かる。
Bevy 的にも、これだと Position を検索した時に Ball は引っ掛からないらしい。

とはいえ、Ball 使うときは必ず Positoin も add してね、というのはバグを招く
```
commands.spawn((Positoin(Vec2::ZERO), Ball))
```

必ずこのコンポーネントも使ってね、というケースでは require を使う。
```
#[derive(Component, Default)]
#[require(Transform)]
struct Position(Vec2);

#[derive(Component)]
#[require(Position)]
struct Ball
```

Position は Default なのでデフォルト値で追加されるそうだが、require 対象は Default が必須かしら?
Default 付けないとコンパイルエラーになるな。

次に、Positoin の投影(porjection).
これも system 関数として実装するようだ。

```
fn main() {
  ...
  app.add_systems(FixedUpdate, project_positoins);
}

fn project_position(mut positionables: Query<(&mut Transform, &Position)>) {
  for (mut transform, position) in &mut positinables {
    transform.translation = position.0.extend(0.);
  }
}
```

query は置いといて、projection のアルゴリズムを見る。
とはいえ座標には何もせず、そのまま型変換しているだけ。
Vec2d から Vec3d にする必要はある。単純に extend で 3次元目(z座標)を 0 埋めしている。
2d->3d には Velocity(Vec3d) というのもあるみたい。

で、Query. これは Query<D,F> な総称型で、
 - D(QueryData): 返す component
 - F(QueryFilter): component を選択するフィルタ。オプション。
コードでは QueryData は `(&mut Transform, &Positon)` のタプル。Filter は無し。

なので、Transform と Position コンポーネントを備えたエンティティを要求している。
ゲームワールドからの全検索であるが、実際に for ループなどで query が実行されるまではコストは発生しない。

# Giving our ball a shape
形を表示するには二つの情報が必要:
 1. Mesh: オブジェクトのシェイプ(形)。透明。
 2. Material: シェイプを塗るテクスチャ。

シェイプの定義にはここでは Mesh2d を使う。2次元空間の頂点情報の集合体。
テクスチャには MeshMaterial2d を使い、Color::srgb の単色で描画させる。

Bevy では、アセットを登録すると、それぞれの型専用の「リソース」に保持される。
Mesh2d であれば Assets<Mesh2d>.
登録すると、このリソースへの Handle が得られる。

Assets<Mesh2d>, Assets<MeshMaterial2d> は AssetsPlugin(DefaultPlugins に含まれる)で提供されている。

system 関数の引数にリソースを追加すれば、そのリソースへのアクセス権が得られる。
リソースは Res<Assets<T>> や ResMut<Assets<T>>.
spawn 時に mesh のメモリ領域を確保して、それを ResMut に add してハンドルを得るという流れ。

```
conse BALL_MESH: Circle = Circle::new(100.);
const BALL_COLOR: Color = Color::srgb(1., 0., 0.);

fn spawn_ball(mut commands: Commands, mut meshes: ResMut<Assets<Mesh>>, mut materials: ResMut<Assets<ColorMaterial>>) {
  let mesh = meshes.add(BALL_SHAPE);
  let material = materials.add(BALL_COLOR);
  commands.spawn((Ball, Mesh2d(mesh), MeshMaterial2d(material)));
}
```

これで画面に赤い円が描画される。

ここで mesh なら mesh で、3つの型が出てきてるな。
 - `BALL_SHAPE: Circle = Circle::new(100.)`
   これは単純なシェイプデータ。まだゲームワールドには無いので抽象的なもの。
 - `mesh = meshad.add(BALL_SHAPE)`
   これは `Assets<Mesh>` リソースへのハンドル。
   BALL_SHAPE のメモリ情報はコピーされてると思う。
   ハンドル先は Circle のまま(かそのポインタ)で、データ表現としては抽象的なまま。
 - `Mesh2d(mesh)`
    これはコンポーネント。ここにきて Bevy システム上で取り扱うコンポーネントという型になっている。
    Mesh2d はタプル構造体で、中に抽象的なシェイプ型を取るのだろう。


# Moving our ball

まず ball の位置を変更する関数を実装してみる。
ボールサイズは 100. では大きいので 10. にしておく。

```
fn move_ball(mut ball: Query<&mut Position, With<Ball>>) {
  if let Ok(mut position) = ball.single_mut() {
    positoin.0.x += 1.0;
  }
}
```

Query のフィルタに `With<Ball>` を指定している。
これは、QueryData 指定のエンティティのうち、さらに Ball コンポーネントを備えたエンティティを選別する。

QueryData の方に Ball を指定しないのは、この関数はあくまで Position を変更するから、か?

Query からは single_mut でエンティティを一つだけ取っている。
このゲームはボールは一つだけだから、かな。
単一である保証があるなら、Query の代わりに Single も使える

```
fn move_ball(mut position: Single<&mut Position, With<Ball>>) {
  position.0.x += 1.0;
}
```
Single は内部で Query し、結果がゼロまたは2個以上あったら system 関数をスキップする。

続いてこれをスケジュール。

```
    app.add_systems(
        FixedUpdate,
        (move_ball.before(project_positions), project_positions),
    );
```

チュートリアルだと Update になってるけど、説明は FixedUpdate なのでこちらに。
Update でもあまり変わらん感じだが。後で違いをよく調べよう。
で、before を使い、move_ball は project_positions の前に実行されるように指定。

# Bouncing off a paddle

続いてパドル。
今までの知識でとりあえず描画まで。

```
#[derive(Component)]
#[require(Position)]
struct Paddle;

const PADDLE_SHAPE: Rectangle = Rectangle::new(20., 50.);
const PADDLE_COLOR: Color = Color::srgb(0., 1., 0.);

fn spawn_paddle(mut commands: Commands, mut shapes: ResMut<Assets<Mesh>>, mut materials: ResMut<Assets<ColorMaterial>>) {
    let shape = shapes.add(PADDLE_SHAPE);
    let material = materials.add(PADDLE_COLOR);
    commands.spawn((Paddle, Mesh2d(shape), MeshMaterial2d(material), Position(Vec2::new(250., 0.))));
}

    app.add_systems(Startup, (spawn_ball, spawn_paddles, spawn_camera));
```

パドルは後でもう一個追加するので spawn_paddles で。

実行すると、ボールは右へ動いてパドルへ当たり、そのまま突き抜けていく。
Bevy はデフォルトで衝突や物理は実装されないので。

# Handling collisions
物理は [Rapier]( https://taintedcoders.com/bevy/physics/rapier ) や [Avian]( https://taintedcoders.com/bevy/physics/xpbd ) といったライブラリを使うのが良いが、ここでは理解のため簡単なものを実装する。

## move の可変化
パドルにぶつかったらボールの移動を反転させるわけだが、そのためにまず移動の実装を可変にする。
OOD 的には移動量プロパティを持ったクラス、を ECS ではエンティティで表す。

```
const BALL_SPEED: f32 = 2.;

#[derive(Component, Default)]
struct Velocity(Vec2);

#[derive(Component)]
#[require(Position, Velocity = Velocity(Vec2::new(-BALL_SPEED, BALL_SPEED)))]
struct Ball;

fn move_ball(ball: Single<(&mut Position, &Velocity), With<Ball>>) {
  let (mut position, velocity) = ball.into();
  position.0 += velocity.0 * BALL_SPEED;
}
```

require にデフォルトも指定できるようだ。これなら Default を実装してなくてもいけるかもしれん(試してない)。

前は Query からは for や query_single でコンポーネントにしていた。
Single の場合は指定の QueryData 型で得られていたようで、何も変換はしていなかった。
が、今回は into してるな。よくわからんが。まぁ Single 型(trait) で得られているのだろうから、into 挟んどくのはむしろ分かりやすいか。

サンプルだと velocity 値と move 時で二度 BALL_SPEED を掛けている。
が、意図が良く分からんし、move 時は掛けなかった。

## collision
さて移動量を可変化したので、あとは衝突時に velocity を変更すればよい。
とりあえず画面隅との衝突から。

collision には bevy_math を使う。

```
use bevy::math::bouding::{ Aabb2d, BoudingVolume, IntersectsVolume };
```

aabb は axis-aligned bouding boxes. 軸平行直線との衝突検出。
BoundingVolume と IntersectsVolume は Aabb2d で使われている trait で use しとく必要があるそうだ。

次に衝突判定した後の衝突の種類を enum にしとく。
2d pong なので、上下左右どっちの衝突かがあればよい。

```
#[derive(Debug, PartialEq, Eq, Copy, Clone)]
enum Collision { Left, Right, Top, Bottom }
```

衝突判定関数を作る。

```
fn collide_with_side(ball: Aabb2d, wall: Aabb2d) -> Option<Collision> {
    if !ball.intersects(&wall) {
        return None;
    }
    let closest_point = wall.closest_point(ball.center());
    let offset = ball.center() - closest_point;

    let side = if offset.x.abs() > offset.y.abs() {
        if offset.x < 0. {
            Collision::Left
        } else {
            Collision::Right
        }
    } else {
        if offset.y > 0. {
            Collision::Top
        } else {
            Collision::Bottom
        }
    };
    Some(side)
}
```

衝突処理のアルゴリズムはゲーム特有のノウハウだな。
衝突自体は 2つの物体の bouding box が交差してるかを判定しているのだろう。
その後、ボール中心から至近の壁の位置を調べ、その点とボール位置の相対から衝突方向を確定する、と。
自分が素直に考えると、衝突瞬間の交差位置を計算しようとして困難が湧いてしまうが、このやり方でよいのね。
判定した瞬間の位置情報から計算するのがポイントか。

衝突判定に使う領域が必要。
shape から算出もできるが、毎回計算するのは重い。
ので、shape と別に計算後の値として保持する。
さらに、shape から算出するのも大変なので、明示的に与えてしまう。
で、この衝突領域の型は Aabb2d のようだが、Translate と Position の関係と同じく自前で Component を定義して Aabb2d に変換する。

```
#[derive(Component, Default)]
struct Collider(Rectangle);

#[derive(Component)]
#[require(
    Position,
    Velocity = Velocity(Vec2::new(BALL_SPEED, 0.)),
    Collider = Collider(Rectangle::new(BALL_SIZE, BALL_SIZE)),
)]
struct Ball;

#[derive(Component)]
#[require(Position, Collider = Collider(PADDLE_SHAPE))]
struct Paddle;
```

初速はパドル方向へにしておく。

`collider` は主に Unity で使われる用語のようだ。
発音は `kuh-lai-duh` で、コリジョン由来なのにコリダーでなくてコライダーになるのね。

続いて衝突処理のシステム関数を定義する。
Collider と Position(衝突判定に使う)を参照し、velocity の変更を伴うので、QueryData は `(&mut Velocity, &Collider)`

```
fn handle_collisions(
    ball: Single<(&mut Velocity, &Position, &Collider), With<Ball>>,
    other_things: Query<(&Position, &Collider), Without<Ball>>,
) {
    let (mut ball_velocity, ball_position, ball_collider) = ball.into_inner();
    let ball_aabb = Aabb2d::new(ball_position.0, ball_collider.0.half_size);

    for (other_position, other_collider) in &other_things {
        if let Some(collision) = collide_with_side(
            ball_aabb,
            Aabb2d::new(other_position.0, other_collider.0.half_size),
        ) {
            match collision {
                Collision::Left | Collision::Right => ball_velocity.0.x *= -1.,
                Collision::Top | Collision::Bottom => ball_velocity.0.y *= -1.,
            }
        }
    }
}
```

衝突判定対象は Paddle 指定じゃなくて Ball 以外全部。
後で壁を作るのかな。

system 関数として登録。

```
    app.add_systems(
        Update,
        (
            project_positions,
            move_ball.before(project_positions),
            handle_collisions.after(move_ball),
        ),
    );
```

衝突判定を位置更新の前後のどっちに入れるべきか、ぱっと分からんな。
チュートリアルでは特に理由書かず移動後に入れてるので、こういうもんなのかな。
どちらにせよ、衝突の瞬間に埋まって描画されてしまうのは変わらんか。
埋まらないようにするには、まず位置更新し、衝突判定。衝突してたら衝突分だけ位置を変える必要があるな。
衝突軸で反転させる...だけだと衝突してない部分が逆に埋め込まってしまう。むずいな。1フレームだけだしまぁいいのか?

# Spawning the other paddle
AI 用のパドルを出す。
二つのパドルを識別するためのマーカーコンポーネントを定義。

```
#[derive(Component)]
struct Player;
#[derive(Component)]
struct Ai;
```

どうでもいいけど AI という名称は誤解を招くよなぁ。
ゲームでは伝統的に使うし、1980年代くらいの意味で AI というのはそう外れてないが。
日本だと CPU と言ったりもするが、これはまた違う方向にズレてるし。
Artifial はいいのだが、Intelligence という程ではないよなぁ。
Human と Art とかの方が好みだな。Art もゲームの別の要素と混同あるか。
まぁ戻る。

さて、AI のパドルの位置はプレーヤーと正対に。
現在はプレーヤーの位置を決め打ちで x=250 にしている。
ボールは x=0 なので、AI は x=-250 にすりゃ良さそうだ。

チュートリアルではウィンドウサイズを取得し、そこから位置を決めているけど。
もともと画面サイズに依存しないために Position コンポーネント作って projection してたのでは?
というわけで、自前でそのように書く。

Position 座標系でのサイズは、1.0 が計算しやすそうだが感覚が分からんな。ball size=10 はいくつにすればいいとか。
今の Position=Transform な座標系で 0,0 が画面中心になってるぽいから、それは維持して。

まず spawn で二つのパドルを生成。`(Player, 512.0)` と `(Ai, -512.0)` をリストにしてループで spawn しようとしたが色々難しいな。
まず Player, Ai の型が違う。Component は dyn trait にできないみたい?
関数化して遅延評価にする手もありそうだが、型定義がめんどい。`Fn<dyn Component>` では無理ぽ。

component ではなく、component 付与した entity をリストに入れようともしたが、
```
let list = [ commands.spawn(Player), commands.spawn(Ai) ]
```
Player 時の commands の mut ref が生き残ってて `commands.spawn(Ai)` の時に取得できない。
vec.push で行分割しても同様。どうも spawn の return 値に保持しているかもしれないと判断しているようだ。
逆に、spawn した Entity を貰って `insert(Player)` などする関数を配列化すれば上手く通った。

```
fn spawn_paddles(
    mut commands: Commands,
    mut shapes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<ColorMaterial>>,
) {
    let shape = shapes.add(PADDLE_SHAPE);
    let material = materials.add(PADDLE_COLOR);

    let f = |c: &mut EntityCommands| {
        c.insert((
            Paddle,
            Mesh2d(shape.clone()),
            MeshMaterial2d(material.clone()),
        ));
    };
    f(&mut commands.spawn((Player, Position(Vec2::new(512.0, 0.)))));
    f(&mut commands.spawn((Ai, Position(Vec2::new(-512.0, 0.)))));
}
```

他のやり方でも書けそうだが、とりあえずこれで。

つづいて、projection.
2D の拡縮だけなので、window サイズ取って比計算するだけ。

```
fn project_positions(
    mut positionables: Query<(&mut Transform, &Position)>,
    window: Single<&Window>,
) {
    let wsize = window.resolution.size() / 2.;
    for (mut transform, position) in &mut positionables {
        transform.translation.x = position.0.x * wsize.x / 1024.0;
        transform.translation.y = position.0.y * wsize.y / 1024.0;
        transform.translation.z = 0.;
    }
}
```

動いた。
ちょっと衝突時にパドルにボールが埋め込むのが見れてしまうようだ。
SPEED = 2.0 相当がめり込むのは仕方ないけど、もっとめり込んでるように見える。
衝突判定計算は、全部 position 座標系になっているのでズレはなさそうである。

ウィンドウサイズを変えても描画位置は良い。
大きくすると衝突時の埋め込みの違和感が減っていく。
collider のサイズは position から porjection されるが、パドルやボールの mesh はウィンドウサイズ反映してないからか。
とりあえず最初の mesh 決める時にウィンドウサイズ考慮した projection にしてみて、埋め込まれないようになった。
ただ、リサイズした時に collider は変わるようになったけど、描画メッシュは変わらないので変になるな。これは今後の課題ということで。

# Creating the gutters
ガター。ボウリングのレーン脇の溝のあれね。
上下に横線を引く。

コードが増えて来たのでファイル分割もしとく。

``` gutter.rs
use super::collision::Collider;
use super::position::Position;
use bevy::prelude::*;

const GUTTER_COLOR: Color = Color::srgb(0., 0., 1.);
const GUTTER_HEIGHT: f32 = 20.;

#[derive(Component)]
#[require(Position, Collider)]
pub struct Gutter;

pub fn spawn_gutters(
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<ColorMaterial>>,
    window: Single<&Window>,
) {
    let material = materials.add(GUTTER_COLOR);
    let padding = 20.;

    let gutter_shape = Rectangle::new(window.resolution.width(), GUTTER_HEIGHT);
    let mesh = meshes.add(gutter_shape);

    let top_gutter_position = Vec2::new(0., window.resolution.height() / 2. - padding);
    commands.spawn((
        Gutter,
        Mesh2d(mesh.clone()),
        MeshMaterial2d(material.clone()),
        Position(top_gutter_position),
        Collider(gutter_shape),
    ));

    let bottom_gutter_position = Vec2::new(0., -window.resolution.height() / 2. + padding);
    commands.spawn((
        Gutter,
        Mesh2d(mesh.clone()),
        MeshMaterial2d(material.clone()),
        Position(bottom_gutter_position),
        Collider(gutter_shape),
    ));
}
```

ボールが壁に当たるよう、縦方向の速度を与えておく。

``` ball.rs
    Velocity = Velocity(Vec2::new(BALL_SPEED, BALL_SPEED * 2.)),
```

ボール衝突時の速度反転ロジックは、 Position, Colision 持ちに対して実装されているので、追加実装無しで壁に反射する。

``` ball.rs
pub fn handle_collisions(
    ball: Single<(&mut Velocity, &Position, &Collider), With<Ball>>,
    other_things: Query<(&Position, &Collider), Without<Ball>>,
) {
    ...;
    for (other_position, other_collider) in &other_things {
        if let Some(collision) = collide_with_side(
            ball_aabb,
            Aabb2d::new(other_position.0, other_collider.0.half_size),
        ) {
```

# 16. Moving our paddle
パドルを操作できるようにする。

ButtonInput リソースを扱えばいけるようだ。
プレーヤーパドルの操作なので paddle.rs にまとめるべきか、入力操作でまとめるべきか、悩むな。
入力は一か所にしといた方がいい気はするな。
モードで動作が違うとか、複雑になったときに、他の入力操作と俯瞰できた方が良さそうである。

移動は、paddle の position を直接書き換えてもできそうだが、velocity を経由させた方が柔軟そうで良いか。
チュートリアルもそうやってるし。
まずは paddle に velocity を持たせ、移動できるように。

``` paddle.rs
#[derive(Component)]
#[require(Position, Velocity, Collider = Collider(PADDLE_SHAPE))]
pub struct Paddle;

pub fn move_paddles(mut paddles: Query<(&mut Position, &Velocity), With<Paddle>>) {
    for (mut position, velocity) in &mut paddles {
        position.0 += velocity.0;
    }
}
```

ball とまとめて、Position, Velocity 持ちは全部移動するようにしてしまっても良さそうだが。
移動の制約が個別にあると大変そうだな。
そういう制約も抽象化し、component にして entity に持たせて、if で個別ロジック組むのかしら。
OOP 的には制約オブジェクトのメソッドにしたいところだが。
Rust でやると、enum にして、同一関数名の trait 持たせて実装するのかな。
でもこれだと OOP か? 関数型プログラミングでオーバーロードってどうやるんだっけな...まぁ後で。
とりあえず統合はしないでおく。チュートリアルでもやってないし。

続いて入力。
キーボードのリソースを system 関数にウォッチさせればいいみたい。

``` input.rs
const PADDLE_SPEED: f32 = 5.;

pub fn handle_player_input(
    keyboard_input: Res<ButtonInput<KeyCode>>,
    mut paddle_velocity: Single<&mut velocity::Velocity, With<paddle::Player>>,
) {
    if keyboard_input.pressed(KeyCode::ArrowUp) {
        paddle_velocity.0.y = PADDLE_SPEED;
    } else if keyboard_input.pressed(KeyCode::ArrowDown) {
        paddle_velocity.0.y = -PADDLE_SPEED;
    } else {
        paddle_velocity.0.y = 0.;
    }
}
```

入力はこんな感じになるのね。
コントローラーやキーマップ編集に対応させるなら、一旦抽象層を置けばよさそう。
必要な全入力を調べてしてマッピング反映し、そのあとで上の pressed チェックを抽象マップに対して行う、と。
複数ボタンの同時押しや、シーケンス入力も対応できるであろう。

それは今後ということで、次にこの mode_paddle と handle_player_input を登録する。
順序は、input は move の前だな。move は projection の前ならどこでもいいか。

```
    app.add_systems(
        FixedUpdate,
        (
            position::project_positions,
            ball::move_ball.before(position::project_positions),
            ball::handle_collisions.after(ball::move_ball),
            paddle::move_paddles.before(position::project_positions),
            input::handle_player_input.before(paddle::move_paddles),
        ),
    );
```

これで動く。
良い感じにボールも跳ね返せる。
ただ、パドルが壁を越えてしまうな。
速度変化時に位置判定してゼロにするのは、上の入力処理ロジックに位置も見ないといけないので複雑になる。
位置変化時に調整するのがよかろう。変化後にコリジョン、というか壁を越えていたら壁の中に戻せばいいかな。
チュートリアルでは、位置変更関数の中ではなく、専用にシステム関数を作っているな。確かにその方が良さそう。

```
pub fn constrain_paddle_position(
    mut paddles: Query<(&mut Position, &Collider), (With<Paddle>, Without<Gutter>)>,
    gutters: Query<(&Position, &Collider), (With<Gutter>, Without<Paddle>)>,
) {
    for (mut paddle_position, paddle_collider) in &mut paddles {
        for (gutter_position, gutter_collider) in &gutters {
            let paddle_aabb = Aabb2d::new(paddle_position.0, paddle_collider.0.half_size);
            let gutter_aabb = Aabb2d::new(gutter_position.0, gutter_collider.0.half_size);
            if let Some(collision) = collide_with_side(paddle_aabb, gutter_aabb) {
                match collision {
                    Collision::Top => {
                        paddle_position.0.y = gutter_position.0.y
                            + gutter_collider.0.half_size.y
                            + paddle_collider.0.half_size.y;
                    }
                    Collision::Bottom => {
                        paddle_position.0.y = gutter_position.0.y
                            - gutter_collider.0.half_size.y
                            - paddle_collider.0.half_size.y;
                    }
                    _ => {}
                }
            }
        }
    }
}
```

今更だが、座標系は上方向に y が増える。
rectangle の new は、中央点と、縦横の半分の長さなので half_size がちょいちょい出てくる。
Collision::Bottom(パドルの下にガター) だと、修正後のパドル位置をガターの上に置きたい。
その y 座標は、ガターの中央位置から上方向(プラス方向)にまずガターの半分の長さ。
そしてパドル位置指定は中央なわけで、そこからパドル半分だけ上方向の位置。

なのでこうなる。
大きさはシェイプ使った方が見た目に合うと思うが、コリジョンの方が衝突しない位置になるのでゲーム的には望ましいのだろう。

```
                        paddle_position.0.y = gutter_position.0.y
                            + gutter_collider.0.half_size.y
                            + paddle_collider.0.half_size.y;
```

これを move_paddle の後に置く。
move_paddle は projection の後という指定はそのままでよいだろう。move_ball とかと統一的な指定になるし。

```
            paddle::move_paddles.before(position::project_positions),
            input::handle_player_input.before(paddle::move_paddles),
            paddle::constrain_paddle_position.after(paddle::move_paddles),
```

よし。

# 17. Scoring
スコア。
component にするとして、どこのエンティティに持たせるべきか。
pong だと player, ai がそれぞれスコアあるから、Paddle かしら。
もしくはスコア保持用のエンティティを player, ai それぞれに作るか。
2要素を持つスコア構造体を component にして、それを持つエンティティ作るのもありか。
最後のやり方が好みだけど、さてチュートリアルは。
こういう、エンティティに持たせるのが合わないグローバルデータは Resource にしろだってさ。
Mesh 置き場とかのあれかね。

``` score.rs
#[derive(Resource)]
pub struct Score {
    player: u32,
    ai: u32,
}
```

app に追加すると。

```
    app.insert_resource(score::Score { player: 0, ai: 0 });
```
add_systems は複数形だが、リソースは単数形なのね。

表示...は次のセクションか。
スコア関連の動作から。
えーと、ボールが画面端に達したらスコア加算し、ボールを中央に戻すのかな。
ball positoin と score を受け取る system 関数を追加すればよさげ?
スコア更新とゲームリセットの処理は、たまたま今回はセットだが、ドメインは違うので分けた方がよいとのこと。
そして分割する場合、位置リセット後に判定が入らないように注意が必要になると。
分割しても依存関係はあるのね。一般化するよりかは、リセットは特殊なので特別扱いしてもよさそうだが。

チュートリアル続ける。
ここでまた新しい要素。system 間の連携に message と event が使えるそうだ。
今回はイベントかな。ゲーム終了のイベントを発火し、それをスコアリングとリセットのシステムが監視する?

えーと、イベントは二種類、entity に紐づく EntityEvent と、紐づかない大域の Event があるようだ。
チュートリアルでは、スコアを与えるエンティティイベント Scored を定義し、スコア加算側のエンティティと紐づける。
Paddle かしら。
Component ではなくてエンティティじゃないとダメだから、Paddle 持ちの Entity か。

``` score.rs
#[derive(EntityEvent)]
pub struct Scored {
    #[event_target]
    scorer: Entity,
}
```

システム関数。
当たり判定は aabb ではなくて単純に x 座標とウィンドウサイズで。

``` score.rs
pub fn detect_goal(
    ball: Single<(&Position, &Collider), With<Ball>>,
    player: Single<Entity, (With<Player>, Without<Ai>)>,
    ai: Single<Entity, (With<Ai>, Without<Player>)>,
    window: Single<&Window>,
    mut commands: Commands,
) {
    let (ball_position, ball_collider) = ball.into_inner();
    let half_window_size = window.resolution.size() / 2.;

    if ball_position.0.x - ball_collider.0.half_size.x > half_window_size.x {
        commands.trigger(Scored { scorer: *ai })
    } else if ball_position.0.x + ball_collider.0.half_size.x < -half_window_size.x {
        commands.trigger(Scored { scorer: *player })
    }
}
```

Single に * で Entity が得られるのね。
into_inner だと Singlet<T> の T への参照になるようだが。
Deref<Entity> で、&Component への変換が出来る型という感じか。

イベント発火は commands.trigger ね。了解。

次は、このイベントのハンドリングか。
集約するなということなので、リセット処理とスコアリング処理でそれぞれ分けて書くのがよいのかな。
スコアリングは score.rs だな。リセットは ball.rs かな?

``` score.rs
pub fn update_score(
    event: On<Scored>,
    mut score: ResMut<Score>,
    is_ai: Query<&Ai>,
    is_player: Query<&Player>,
) {
    if is_ai.get(event.scorer).is_ok() {
        score.ai += 1;
        info!("AI scored! {} - {}", score.player, score.ai);
    }
    if is_player.get(event.scorer).is_ok() {
        score.player += 1;
        info!("Player scored! {} - {}", score.player, score.ai);
    }
}
```

Query の get(Entity) で、Query 結果に Entity が含まれるか Result で返すのかな。

イベント監視は、On<Event> 引数で関心指定するだけか。
is_fired 的なチェックはこの関数呼ぶ前のところでやってくれると。

続いてリセット処理。

``` ball.rs
pub fn reset_ball(
    _ev: On<Scored>,
    ball: Single<(&mut position, &mut Velocity), With<Ball>>,
) {
    let (mut ball_position, mut ball_velocity) = ball.into_inner();
    ball_positoin.0 = Vec2::ZERO;
    ball_velocity.0 = Vec2::new(BALL_SPEED, BALL_SPEED * 2.);
}
```

位置リセットは spawn_ball と共通化したいところだがとりあえずこれで。

``` main.rs
    app.add_system(FixedUpdate, (..., score::detect_goal.after(ball::move_ball)));
    app.add_observer(ball::reset_ball);
    app.add_observer(score::update_score);
```

イベント監視は observer ね。複数一度に登録はできないのか。

# 18. Displaying our score
ではスコアの表示。
えーと、shape モノは Mesh2d, MeshMaterial2d コンポーネントを spawn 時に付けていた。
スコアはテキストなので、Mesh2d じゃなくてテキスト用のシェイプがあるのだろう。

スコア表示用に entity がある。
スコア自体は resource に記録されているが、表示システム関数での query 用にマーカーコンポーネントを作る。

``` score.rs
#[derive(Component)]
pub struct PlayerScore;

#[derive(Component)]
pub struct AiScore;
```

スコア表示はゲームオブジェクトではなく、UI 表示の仕組みに載せるみたい。
これはよくある GUI ライブラリみたく、ツリー構造でレイアウト書けるぽいな。
宣言型表記はまだなさそう。地道にツリーを手続き的に書いていく。

``` score.rs
pub fn spawn_scoreboard(mut commands: Commands) {
    let container = Node {
        width: percent(100.0),
        height: percent(100.0),
        justify_content: JustifyContent::Center,
        ..default()
    };

    let header = Node {
        width: px(200.),
        height: px(100.),
        ..default()
    };
    
    let player_score = (
        PlayerScore,
        Text::new("0"),
        TextFont::from_font_size(72.0),
        TextColor(Color::WHITE),
        TextLayout::new_with_justify(Justify::Center),
        Node {
            position_type: PositionType::Absolute,
            top: px(5.0),
            right: px(25.0),
            ..default()
        },
    );
    let ai_score = (
        AiScore,
        Text::new("0"),
        TextFont::from_font_size(72.0),
        TextColor(Color::WHITE),
        TextLayout::new_with_justify(Justify::Center),
        Node {
            position_type: PositionType::Absolute,
            top: px(5.0),
            left: px(25.0),
            ..default()
        },
    );

    commands.spawn((
        container,
        children![(header, children![player_score, ai_score])],
    ));
}
```

container, header の Node 構造体は分かりやすい。

player_score, ai_score は謎の表記だな。タプルの最初に component, あとは text ウィジェットの属性か?
これで Text component らしいが。
表示位置はチュートリアルだと AI が右になってる。どこかで間違ったか自分のコードは右側がプレーヤーなので右側にしておいた。

spawn 時に、children![] と指定してる。
表記は、`(親、children![])` かな。
これで親エンティティに子エンティティが結び付けられるのだろう。
transform が親従属になるとのこと。


最後にスコア更新表示。
これは FixedUpdate のシステム関数ね。

``` score.rs
pub fn update_scoreboard(
    mut player_score: Single<&mut Text, (With<PlayerScore>, Without<AiScore>)>,
    mut ai_score: Single<&mut Text, (With<AiScore>, Without<PlayerScore>)>,
    score: Res<Score>,
) {
    if score.is_changed() {
        player_score.0 = score.player.to_string();
        ai_score.0 = score.ai.to_string();
    }
}
```

さっきの player_score の謎タプルは Text component のようだ。
PlayerScore をタプルの最初に書いていたから、この component も追加されているのだろう。
何度も出てきてスルーしてたけど、player, ai は背反で実装してるから Without は要らない気もするが、こういうもんなのかね。

いつもの `let (mut player_score) = player_score.inner()` してないな。
Single は一要素だと into_inner() しないで .0 で取れるんだっけ。
色々アクセス方法あって覚えてないや。あとで整理しとこ。

score リソースの is_changed() 見てるけど、これは前回の update 呼び出しの間に更新されたかが保持されてるんかな?
もしくは score に最終更新時刻的なフィールドがあって、update_scoreboard が裏で前回チェック時刻を保存してて、それを比較してるのだろうか。
`Bevy's change detection feature` らしいが。後でドキュメントかソース確認しとこ。

で、app に登録。

```
    app.add_systems(
        Startup,
        (
            score::spawn_scoreboard,
    app.add_systems(
        FixedUpdate,
        (
            score::update_scoreboard.after(score::detect_goal),
```

# 19. Programming the AI player
Ai アルゴリズムは深そうだが、チュートリアルは単純だな。

ボールとパドルの高さ差分を計算して、その signum() とパドルスピードを掛けた値を velocity に入れる。
signum は -1,0,1 を返す関数か。単純に y 座標の大なり小なりの if でも良さそうである。
まぁ signum に代えて -1..1 のランダム数値にするとか変更しやすいか。

```
pub fn move_ai(
    ai: Single<(&mut Velocity, &Position), With<Ai>>,
    ball: Single<&Position, With<Ball>>,
) {
    let (mut velocity, position) = ai.into_inner();
    let a_to_b = ball.0 - position.0;
    velocity.0.y = a_to_b.y.signum() * PADDLE_SPEED;
}
```

ここでは `Without<Player>` してないのね。

app にも登録。

``` main.rs
    app.add_systems(
        FixedUpdate,
        (
            paddle::move_ai.before(paddle::move_paddles),
```
