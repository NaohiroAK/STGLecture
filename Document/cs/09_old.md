## 09. 当たり判定を実装してみよう

### 概要

前回までの内容で、複雑な動きをする敵がいくつか増えました。
しかし、依然として「自機が敵の弾に当たっても」「敵が自機の弾に当たっても」何も起こらず、これではどうもゲームらしくないですね。

そこで今回は、シューティングゲームにおいて最も重要な「当たり判定」の書き方を解説していきたいと思います。

### 「当たり判定」とは

「当たり判定」とは読んで字の如く、オブジェクト同士が当たっているかどうか判定するものです。当たり判定の実装方法には色々な手法が考えられますが、今回は簡単のために「円同士の当たり判定をピクセル単位で取る」という方法を扱います。

例えば下のような２つの円のオブジェクトを考え、２つの円の中心座標を考えてみましょう。２つの円の半径をa, bとおき、２つの円のx座標の差をd、y座標の差をeとおくと、中学校で習うような「三平方の定理」よりd^2+e^2<(a+b)^2ならば２つの円は「ぶつかっている」ということになりますね。

<図>



### 下準備をしよう

さて、今まで書いたコードについては、当たり判定を導入する前にいくつかの下準備をする必要があります。

#### 当たり判定を保証するインターフェース

まず、当たり判定を持つクラスを決めましょう。今回は、「プレイヤーと敵の弾」、「敵と自機の弾」が衝突してほしいので、 ```Player``` 、 ```Bullet``` 、 ```Enemy``` 、 ```EnemyBullet```　のそれぞれが当たり判定を持つクラスになるはずです。

そこで、C#が持つ[インターフェース](http://ufcpp.net/study/csharp/oo_interface.html)を用いて、クラスを拡張することにします。インターフェースはクラスの規約を決めるもので

「 ```Player``` 、 ```Bullet``` 、 ```Enemy``` 、 ```EnemyBullet```　のそれぞれが当たり判定を持つクラス」

であることを保証するのが、今回の「当たり判定をするインターフェース」の役割です。

では、当たり判定を持つクラスが実装すべきものを決めましょう。

* 位置座標
* 半径
* 相手と衝突したかどうか
* 相手と衝突したときの処理

この4つを実装するように、インターフェースで規約を決めましょう。
プロジェクトに新規追加でcsファイルを追加し、インターフェース ```ICollidable``` のコードを以下のように書きましょう。

```cs
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace STG
{
    public interface ICollidable
    {
    	// interface内に記述できるのは抽象プロパティ・メソッドのみで、その実装は行わない。
    	// この中では、プロパティ・メソッドは public abstract を省略して記述される。

        // Position を取得することができる
        ace.Vector2DF Position{get;}

        // Radius(当たり判定の半径) を取得することができる
        float Radius{get;}

        // 衝突しているかどうかを判定するメソッドを実装する
        bool IsCollide(ICollidable obj);

        // 衝突時の処理を行うメソッドを実装する
        void OnCollide(ICollidable obj);
    }
}
```

インターフェースで記述できるのは、あくまで抽象プロパティ・抽象メソッドであり、具体的な値を定めるなどの実装は行わないことに気を付けてください。これを実装するのは **```ICollidable```　インターフェースを持つクラスの義務**です。

* 位置座標は ```Position``` 
* 半径は ```Radius```
* 当たり判定は ```IsCollide```
* 当たった時の処理は ```OnCollide``` 

としました。このインターフェースをクラスに規則として加え、そのクラスでこれらの規則を改めて定義する必要があります。

では、 ```ICollidable``` インターフェースを各クラスに実装してみましょう。

```Enemy``` クラスを以下のように編集します。

```diff
-    public class Enemy : ace.TextureObject2D
+    public abstract class Enemy : ace.TextureObject2D, ICollidable
    {
        //毎フレーム1増加し続けるカウンタ変数（継承先のクラスで使いまわすため、protectedに設定する。）
        protected int count;

        //プレイヤーへの参照（継承先のクラスで使いまわすため、protectedに設定する。）
        protected Player player;

+        // 当たり判定の半径、値の取得(get)・代入(set)ができるように宣言
+        public float Radius {get; set;}

        //コンストラクタ(敵の初期位置を引数として受け取る。)
        public Enemy(ace.Vector2DF pos, Player player)
            : base()
        {
            // 敵のインスタンスの位置を設定する。
            Position = pos;

            //　画像を読み込み、敵のインスタンスに画像を設定する。
            Texture = ace.Engine.Graphics.CreateTexture2D("Resources/Enemy.png");

            // 敵のインスタンスに画像の中心位置を設定する。
            CenterPosition = new ace.Vector2DF(Texture.Size.X / 2.0f, Texture.Size.Y / 2.0f);

+            // 画像の半分の大きさを Radius とする
+            Radius = Texture.Size.X / 2.0f;

            // カウンタ変数を0に初期化する。
            count = 0;

            // Playerクラスへの参照を保持する。
            this.player = player;
        }

        ...(省略)...

+        public virtual bool IsCollide(ICollidable obj){
+            // Enemy の当たり判定
+        }

+        public virtual void OnCollide(ICollidable obj){
+            // Enemy の衝突時の処理
+        }

    }
```

他の3つのクラスについても同様の変更を行います。結果、以下のようになります。

```Player``` クラス

```diff
-public class Player : ace.TextureObject2D
+public class Player : ace.TextureObject2D, ICollidable
    {
+       public float Radius { get; set; }

        public Player()
        {
            // 画像を読み込み、プレイヤーのインスタンスに画像を設定する。
            Texture = ace.Engine.Graphics.CreateTexture2D("Resources/Player.png");

            // プレイヤーのインスタンスに画像の中心位置を設定する。
            CenterPosition = new ace.Vector2DF(Texture.Size.X / 2.0f, Texture.Size.Y / 2.0f);

            // プレイヤーのインスタンスの位置を設定する。
            Position = new ace.Vector2DF(320, 480);

+           // プレイヤーの Radius は小さめにしておく
+           Radius = Texture.Size.X / 8.0f;
        }

        protected override void OnUpdate()
        {
        	...(省略)...
        }

+        public virtual bool IsCollide(ICollidable obj){
+            // Player の当たり判定
+        }

+        public virtual void OnCollide(ICollidable obj){
+            // Player の衝突時の処理
+        }
    }
```

```Bullet``` クラス

```diff
-    class Bullet : ace.TextureObject2D
+    class Bullet : ace.TextureObject2D, ICollidable
    {
        public float Radius { get; set; }

        public Bullet(ace.Vector2DF position)
        {
            // 画像を読み込み、弾のインスタンスに画像を設定する。
            Texture = ace.Engine.Graphics.CreateTexture2D("Resources/PlayerBullet.png");

            // 弾のインスタンスに画像の中心位置を設定する。
            CenterPosition = new ace.Vector2DF(Texture.Size.X / 2.0f, Texture.Size.Y / 2.0f);

            // 弾のインスタンスの位置を設定する。
            Position = position;

+            // 画像の半分の大きさを Radius とする
+            Radius = Texture.Size.X / 2.0f;
        }

        protected override void OnUpdate()
        {
        	...(省略)...
        }

+        public virtual bool IsCollide(ICollidable obj){
+            // Bullet の当たり判定
+        }

+        public virtual void OnCollide(ICollidable obj){
+            // Bullet の衝突時の処理
+        }
    }
```

```EnemyBullet``` クラス

```diff
-    public class EnemyBullet : ace.TextureObject2D
+    public abstract class EnemyBullet : ace.TextureObject2D, ICollidable
    {
+        public float Radius { get; set; }

        // コンストラクタ(敵の初期位置を引数として受け取る。)
        public EnemyBullet(ace.Vector2DF pos)
            : base()
        {
            // 敵弾のインスタンスの位置を設定する。
            Position = pos;

            //　画像を読み込み、敵弾のインスタンスに画像を設定する。
            Texture = ace.Engine.Graphics.CreateTexture2D("Resources/EnemyBullet.png");

            // 敵弾のインスタンスに画像の中心位置を設定する。
            CenterPosition = new ace.Vector2DF(Texture.Size.X / 2.0f, Texture.Size.Y / 2.0f);

+            // 画像の半分の大きさを Radius とする
+            Radius = Texture.Size.X / 2.0f;
        }

        ...(省略)...

+        public virtual bool IsCollide(ICollidable obj)
+        {
+            // EnemyBullet の当たり判定
+        }

+        public virtual void OnCollide(ICollidable obj)
+        {
+            // EnemyBullet の衝突時の処理
+        }
    }
```



##### abstract とは
まず、突然現れた ```public abstract class``` についてですが、クラスに ```abstract``` 修飾子をつけることによって、このクラスが[抽象クラス](http://ufcpp.net/studY/csharp/oo_abstract.html)であることが示されます。

抽象クラスの特徴として、**直接インスタンス化してはいけない**という制限があります。

ゲームとして、敵キャラの要件は満たしているけれども、移動など細かいところは（それが雑魚敵でも）工夫が必要になる場合が多いです。そのため、8章では敵をクラス分けし、 ```Enemy``` クラスを全く動かない敵キャラとして、様々な動きをする「敵キャラのクラス」を別々に作りました。

この場合、 ```Enemy``` クラスのインスタンスそのものをゲームに登場させる必要はないはずです。ですから、ここではこの ```Enemy``` クラスを ```abstract class``` (抽象クラス) として取り扱います。これは ```EnemyBullet``` クラスにも同じことが言えますし、もし ```Bullet```　クラス(自機の弾のクラス) に種類のバリエーションが欲しいと思った場合には、 ```Bullet``` クラスを抽象クラスにする必要があるでしょう。

##### クラスにインターフェースを実装する

クラスにインターフェースを実装するためには、継承と同じような方法を用います。

	class クラス名 : インターフェース名
	{
	  クラスの定義
	}

クラスが継承できる基底クラス（親クラス）はC\#では1つだけですが、インターフェースは複数実装することができます。なぜなら、インターフェースはあくまで規約であり実装を伴わないので、継承元がコンフリクトすることがないからです。

基底クラスに加えて、インターフェースを実装する場合には、

	class クラス名 : 基底クラス名, インターフェース名

というように、カンマで区切ることでインターフェースを実装することができます。これは、複数のインターフェースを実装する場合も同様です。

今回の場合は

	class Enemy : ace.TextureObject2D, ICollidable

と読み替えることができます。

##### インターフェースに沿って実装をする

実装すべきものは4点ありました。

* 位置座標は ```Position``` 
* 半径は ```Radius```
* 当たり判定は ```IsCollide```
* 当たった時の処理は ```OnCollide``` 

このうち ```Position``` は、 ```ace.TextureObject2D``` クラスを継承した際、既に実装されています。
これについては、新たに実装する必要性はありません。

半径として ```Radius``` を実装する必要があります。インターフェースではフィールド（変数）を宣言できないので、[プロパティ](http://ufcpp.net/study/csharp/oo_property.html)として宣言する必要があった ```Radius``` ですが、ここでは ```get``` (値の取得)、 ```set``` (値の代入) ができるように以下のように宣言しています。

	public float Radius {get; set;}

単純にフィールド(変数)として宣言できないことに注意しましょう。

次に、```IsCollide``` と ```OnCollide``` です。

どちらのメソッドにも ```virtual``` という修飾子があります。これは、実装はしているが、子クラスがこのメソッドを ```override``` (上書き)することを認める記述です。基本的な動作は決まっているが、子クラスでもメソッドを工夫する可能性がある場合に使えるキーワードです。

このメソッドの内部は後ほど実装することにしましょう。


### 実装してみよう

では、実際に当たり判定を実装してみましょう。

#### IsCollideの実装

```Player``` 、 ```Bullet``` 、 ```Enemy``` 、 ```EnemyBullet``` 、それぞれにある ```IsCollide``` メソッドを編集します。

今回は、一貫して円による当たり判定を想定しています。これを実装したのが以下のコードです。このコードは4つのクラスで共通です。

```cs
        // ICollidable なオブジェクトを渡すと、衝突しているかどうかを判定し bool を返す
        public bool IsCollide(ICollidable obj)
        {
            // 二点間の距離 が お互いの半径の和 より小さい場合には　true
            if ((obj.Position - Position).Length < Radius + obj.Radius)
                return true;
            else
                return false;
        }
```

```(obj.Position - Position).Length``` では、位置座標のベクトルの差をとって、その大きさ（スカラー）をとることで、二点間の距離を得ています。

```Radius + obj.Radius``` では、お互いの半径の和をとってます。この半径は、当たり判定に使うために用意された円の半径です。

お互いの円が、その表面で接している状態というのは、「二点間の距離」と「お互いの半径の和」が等しい状態です。距離がこれ以上縮まってしまうと衝突します。そこで、距離が縮まってしまった場合には衝突したという bool (真偽)を送るようにします。

この ```IsCollide``` を実装できたら次は ```OnCollide``` です。

#### OnCollideの実装

```OnCollide``` は、 ```IsCollide``` がtrueとなった場合に起きる行動(衝突時の処理)
です。

わかりやすいように、衝突してしまったら「弾」も「機体」も消滅するようにしてみましょう。

```Enemy``` 、 ```EnemyBullet``` クラスに以下のような編集をします。

```cs
        // 衝突時の動作(子クラスでoverrideが可能)
        public virtual void OnCollide(ICollidable obj)
        {
            // ゲームからオブジェクトを消滅させる
            Dispose();
        }
```

```Player``` 、 ```Bullet``` クラスにも以下のような編集をしましょう。

```cs
        // 衝突時の動作
        public void OnCollide(ICollidable obj)
        {
            // ゲームからオブジェクトを消滅させる
            Dispose();
        }
```

これで、 ```OnCollide``` の実装は終わりです。あと少しです。実装された```IsCollide``` 、 ```OnCollide```を用いて当たり判定を反映させましょう。

#### プレイヤーと敵の弾、敵とプレイヤーの弾に当たり判定をつける

プレイヤーがダメージを受けるのは敵の弾、敵がダメージを受けるのはプレイヤーの弾なので相互に当たり判定の関係をつくりましょう。

まず、 ```Enemy``` クラスに、 ```CollideWith(ICollidable obj)``` メソッドを追加します。このメソッドは、```IsCollide``` 、 ```OnCollide```を用いて、当たり判定を実際に反映させるためのものです。

```cs
        // 自機の弾との当たり判定をコントロールするメソッド
        protected void CollideWith(ICollidable obj)
        {
            // 当たり判定の相手が見つかってない場合はメソッドを終了
            if (obj == null)
                return;

            // obj が Bulletである場合にのみelse内を動作させる
            if ((obj as Bullet) == null)
                return;
            else
            {
            	// obj が bullet であることを明示
                ICollidable bullet = obj;

                // bulletと衝突した場合には、衝突時処理を行う
                if (IsCollide(bullet))
                {
                    OnCollide(bullet);
                    bullet.OnCollide(this);
                }
            }
        }
```

ここで、新しく ```(obj as Bullet)``` のような表記がありましたが、これはこの ```obj``` が ```Bullet``` クラスに属しているかどうかを確かめています。

属しているならば、これは ```Bullet``` クラスとしての ```obj``` が返ってきますし、属してない場合は ```null``` が返ってきます。

次に、 ```Enemy``` クラスの ```Onupdate``` メソッドを編集して、以下のようにします。

```cs
        protected override void OnUpdate()
        {
            // 当たり判定をする
            foreach (var obj in Layer.Objects)
                CollideWith((obj as ICollidable));

            ++count;
        }
```

ここで、foreachとはfor文の拡張で、

	foreach (var obj in Layer.Objects)

というのは、 「 ```Layer.Objects``` の要素をとりだして、 ```obj``` と名前を付ける」 ことを```Layer.Objects``` の全要素について行ってくれます。

ところで、```Layer.Objects``` についてですが、レイヤーという考えを用いています。詳しくはこれより後の章で述べられますが、意味合いとしては、

「このオブジェクト(Enemyクラスのインスタンス)を登録した場所( ```Layer``` )に登録されているオブジェクト群( ```Objects``` )を取得する」

といった感じです。「ゲーム中に出てくるオブジェクトを全て取得する」と読み替えても、現段階では差し支えないです。

つまり、```Layer.Objects``` の要素の中には、```Player``` 、 ```Bullet``` 、 ```Enemy``` 、 ```EnemyBullet``` それぞれのクラスのインスタンスが含まれていることになります。これを用いて、当たり判定がとれるようになります。

そして、 ```obj as ICollidable``` を ```CollideWith``` メソッドに渡すことで 当たり判定を実装します。
```null``` をメソッドに渡してしまった場合は、それをメソッドが検知して終了する仕組みになっています。

これらによって、 ```Enemy``` クラスは以下のように編集されます。
```diff
public abstract class Enemy : ace.TextureObject2D, ICollidable
    {
        //毎フレーム1増加し続けるカウンタ変数（継承先のクラスで使いまわすため、protectedに設定する。）
        protected int count;

        //プレイヤーへの参照（継承先のクラスで使いまわすため、protectedに設定する。）
        protected Player player;

        // 
        public float Radius { get; set; }

        //コンストラクタ(敵の初期位置を引数として受け取る。)
        public Enemy(ace.Vector2DF pos, Player player)
            : base()
        {
        	...(省略)...
        }

        

        protected override void OnUpdate()
        {
+            // 当たり判定をする
+            foreach (var obj in Layer.Objects)
+                CollideWith((obj as ICollidable));

            ++count;
        }

        ...(省略)...

        public virtual bool IsCollide(ICollidable obj)
        {
            if ((obj.Position - Position).Length < Radius + obj.Radius)
                return true;
            else
                return false;
        }

        public virtual void OnCollide(ICollidable obj)
        {
            Dispose();
        }

+        // 自機の弾との当たり判定をコントロールするメソッド
+        protected void CollideWith(ICollidable obj)
+        {
+            // 当たり判定の相手が見つかってない場合はメソッドを終了
+            if (obj == null)
+                return;

+            // obj が Bulletである場合にのみelse内を動作させる
+            if ((obj as Bullet) == null)
+                return;
+            else
+            {
+            	// obj が bullet であることを明示
+                ICollidable bullet = obj;

+                // bulletと衝突した場合には、衝突時処理を行う
+                if (IsCollide(bullet))
+                {
+                    OnCollide(bullet);
+                    bullet.OnCollide(this);
+                }
+            }
+        }

    }
```

```Player``` クラスも同様に編集します。当たり判定をする相手が ```EnemyBullet``` クラスのインスタンスであることに気を付けてください。

```diff
    public class Player : ace.TextureObject2D, ICollidable
    {
        public float Radius { get; set; }

        public Player()
        {
        	...(省略)...
        }


        protected override void OnUpdate()
        {
+            foreach (var obj in Layer.Objects)
+                CollideWith(obj as ICollidable);

            ...(省略)...
        }

        public bool IsCollide(ICollidable obj)
        {
            if ((obj.Position - Position).Length < Radius + obj.Radius)
                return true;
            else
                return false;
        }

        public void OnCollide(ICollidable obj)
        {
            Dispose();
        }

+        // 敵の弾との当たり判定をコントロールするメソッド
+        protected void CollideWith(ICollidable obj)
+        {
+            // 当たり判定の相手が見つかってない場合はメソッドを終了
+            if (obj == null)
+                return;

+            // obj が EnemyBulletである場合にのみelse内を動作させる
+            if ((obj as EnemyBullet) == null)
+                return;
+            else
+            {
+                // obj が enemyBullet であることを明示
+                ICollidable enemyBullet = obj;

+                // bulletと衝突した場合には、衝突時処理を行う
+                if (IsCollide(enemyBullet))
+                {
+                    OnCollide(enemyBullet);
+                    enemyBullet.OnCollide(this);
+                }
+            }
+        }
    }
```

長い長いコーディングを経て、やっと当たり判定を実装することができました。実行してみると的敵の弾でプレイヤーが消滅したり、自機の弾で敵が消滅するのが確認できると思います。

```OnCollide``` を変えれば消滅以外にも他の事ができますし、 ```Radius``` を変えれば当たり判定を広げたり縮めたりすることができます。

### コラム：さらに踏み込んだ当たり判定を作るには

ここまでの方法で、「１つのフレーム内で、キャラクターと弾が当たっているかどうか」という当たり判定を取得することが出来ました。今回は自機の動作も敵の動作もゆっくりしているので、大方見た目通りの当たり判定が取れるようになっているかと思います。

ここで、次のような場合を考えてみましょう。

<図>

困ったことになりました。この２フレームの「間」では２つの物体は「当たっている」はずなのですが、両方のフレームでは計算上「当たっていない」ことになっているため、見かけ上「すり抜け」が起きてしまいます。
このような「すり抜け」を回避するためには、２つの物体の現在地と移動ベクトルを利用し、下図のように２つの物体の進路に交点があるかどうかを計算する必要があります。

<図>

具体的な計算方法については本章では長くなるため省略しますが、踏み込んだ当たり判定の方法としてこのような方法もあるということは覚えておくとよいでしょう。