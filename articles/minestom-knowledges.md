---
title: "【Minecraft】PaperMC ユーザのための Minestom 入門"
emoji: "📜"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["minecraft", "minestom", "gameserver", "papermc"]
publication_name: "kun_lab"
published: true
---

はじめまして。Peyang （ぺやんぐ）と申します。
普段は JavaJava しているのですが，息抜きに Minecraft についての記事も書いています。

前回の記事では，Minecraft サーバ・ライブラリの一つである [Minestom](https://minestom.net/) を紹介し，その特徴や利点について筆者の独断と偏見を交えつつ解説しました。
この記事では，私が実際に Minestom を使い始めるにあたって感じたことや，基本的な知見について解説します。
皆々様の Minecraft サーバ開発の一助となれば幸いです。

前回の記事はこちら：

https://zenn.dev/kun_lab/articles/minestom-getting-started

## Minestom あれこれ

Minestom についての詳細は，ぜひぜひ前回の記事をご覧いただきたいのですが，ここでは簡単に Minestom の特徴をおさらいします。

Minestom は，Java で書かれた軽量かつ高性能な Minecraft サーバ・ライブラリです。
あくまでも，Minecraft サーバを構築するためのライブラリであり，**完全なサーバ・ソフトウェアではありません**。

Minestom は，Minecraft プロトコルの実装やエンティティ管理，ワールド管理，イベントシステムなど，Minecraft サーバを構築するために必要な基本的な機能を提供します。
開発者は自分のニーズに合わせたカスタム Minecraft サーバを構築できます。

# PaperMC 開発者のための Minestom 入門

ふだん，我々 Minecraft サーバ開発者は，主に [PaperMC](https://papermc.io/) を用いてプラグインを開発していることでしょう。
或いは [SpigotMC](https://www.spigotmc.org/) や [Bukkit](https://bukkit.org/) などのソフトウェアを用いているかもしれません。

いずれにせよ，Minestom はこれらのソフトウェアとは全く異なる設計思想を持っているため，Minestom を使い始めるにあたっては戸惑うことも多いかと思います。
ここでは，PaperMC 開発者の視点から見た Minestom の基本的な使い方について解説します。

## 大抵のことは自分で実装する

Minestom を導入するにあたって最も重要なポイントは，**Minestom には何も無い**ということです。

みなさんが普段使っている PaperMC や SpigotMC には，チェストや Mobなどの Minecraft のゲームロジックが豊富に実装されています。
しかし，Minestom にはこれらの機能は一切実装されていません。

このことをどの程度覚悟できるかが，初動のハードルを大きく左右します。

以下に，Minestom の世界に無いものを明示しておきます。
それと同時に，以下は我々が必要に応じて実装しなければならない機能でもあります。

- **チェストやドアなどの全てのブロックの動作**
- **ポータルなどの特殊なブロック効果**
- **Mob の AI や行動**
- **プレイヤ・データ（インベントリ，経験値，ステータス効果など）の永続化**  
  なお，ランタイム中のデータ管理は Minestom がフル・サポートしています。
- **ワールド生成やワールド管理**  
  ワールドの読み込み・保存などの基本的な機能は Minestom が提供していますが，API を用いて自分で実装する必要があります。
- **アイテムの効果や動作**  
  これには，アイテムの使用によってワールドにもたらされる結果（例：火を起こす，Mob をスポーンさせるなど）も含まれます。
- **コンソールからのコマンド実行**  
  標準のサーバにように，コンソールからコマンドを実行する機能はありません。

などなど。

## インスタンスという概念

Minecraft サーバの文脈では，プレイヤが実際に遊ぶ世界のことを「ワールド」と言います。
もう少し古風な言い方をすると「レベル」などと言うのでしょう。

Minestom では，この「ワールド」に類似する概念を **インスタンス（`Instance`）** と呼んでいます。
インスタンスは，PaperMC/SpigotMC の `World` と同様に，ブロックやエンティティ，チャンクなどのゲーム要素を管理します。

さらに，各インスタンスは独立して動作します。
プレイヤが或るインスタンスに存在している場合には，他のインスタンスに存在するプレイヤは見えませんし，**Tab リストにも表示されません**。

使い所としては，ミニゲーム・サーバで複数のゲームセッションを同時に管理したい場合などが考えられます。

### `Instance`

先程述べた通り，Minestom にはワールド上のブロックを読み込む機能がありません。

従来の Minecraft では，**ワールドには必ずチャンクやブロックがある**ことが保証されています。
しかしながら，Minestom ではワールドという概念に囚われていない `Instance` という上位概念を導入しているため，**ワールドが存在しないインスタンス**を作成することも無論可能です。
例えば，空のインスタンスを作成し，そこにブロックを動的に配置することもできます。

### `InstanceContainer` と `AnvilLoader`

Minestom では，チャンクの無い `Instance` と区別して，独自のチャンクを持つ `InstanceContainer` クラスが提供されています。
`InstanceContainer` は，内部に `ChunkLoader` を持ち，プレイヤの移動に応じてチャンクを動的に読み込み・アンロードします。

`ChunkLoader` は，あらゆるチャンクの読み込み・保存を抽象化するインターフェースです。
ユーザは自分で `ChunkLoader` を実装することもできますし，既存の `ChunkLoader` 実装を利用することもできます。
例えば *Anvil 形式*（Minecraft の標準的なワールド保存形式）で保存されたワールドからチャンクを読み込みたい場合は，`AnvilLoader` クラスを使用します。

```java
MinecraftServer server = MinecraftServer.init();  
InstanceContainer instance = MinecraftServer.getInstanceManager().createInstanceContainer();

instance.setChunkLoader(new AnvilLoader("worlds/my-world"));
```

さらに，Anvil 形式よりも軽量で，かつパフォーマンスに優れた [Polar](https://github.com/hollow-cube/polar) というチャンク保存形式も存在します。
Polar 形式で保存されたワールドからチャンクを読み込みたい場合は，[Polar](https://github.com/hollow-cube/polar) ライブラリを導入したうえで，以下のようにします。

```java
MinecraftServer server = MinecraftServer.init();
InstanceContainer instance = MinecraftServer.getInstanceManager().createInstanceContainer();

instance.setChunkLoader(new PolarLoader("worlds/my-world.polar"));
```

このように， `ChunkLoader` を切り替えることで，様々なチャンク保存形式に対応できるのです。

## イミュータブルな構造

Minestom では，ゲーム内における多くのデータ構造がイミュータブル（不変）として設計されています。
例えば，`Block` クラスはイミュータブルであり，一度作成された `Block` オブジェクトの状態は変更できません。
代わりに，`Block` の状態を変更したい場合は，新しい `Block` オブジェクトを作成する必要があります。

イミュータブルな設計は，データの整合性を保って予期しない副作用を防ぐのに役立ちます。
しかしながら，頻繁に状態を変更する必要がある場合には，都度のオブジェクト生成がパフォーマンスに影響を与える可能性があります。

以下に，代表的なイミュータブル・クラスにおける実装上の考慮すべきポイントを挙げておきます。

### `Block` クラス

`Block` クラスはイミュータブルであり，ブロックの種類や状態を変更する場合は，新しい `Block` オブジェクトを生成する必要があります。
さらに，デフォルトのブロックは `Blocks` インタフェースで定義された定数として提供されており，通常ブロックを設置する際には，
これらの定数で示されるオブジェクトに対して，`withXXX` メソッドを使用して新しいブロックオブジェクトを生成します。

```java
// 石ブロックを設置
Block stoneBlock = Blocks.STONE;
world.setBlock(blockPosition, stoneBlock);

// 向きの異なるエンダー・チェストを設置
Block enderChestNorth = Blocks.ENDER_CHEST.withProperty("facing", "north");
world.setBlock(blockPosition, enderChestNorth);
```

### `ItemStack` クラス

`ItemStack` もイミュータブルであり，アイテムの種類や数量を変更する場合には，新しい `ItemStack` オブジェクトを生成する必要があります。

多数の状態を更新する場合，新たな `ItemStack` オブジェクトを繰り返し生成することは非効率的です。
そのため，`ItemStack#builder(Material)` で得られる `ItemStack.Builder` クラスを使用して，まとめて状態を設定した後に `build()` メソッドで最終的な `ItemStack` オブジェクトを生成することが推奨されます。

```java
// アイテムスタックを生成
ItemStack.of(Material.DIAMOND_BOOTS);

// アイテムスタックをビルダーで生成
ItemStack.builder(Material.DIAMOND_BOOTS)
         .set(DataComponents.UNBREAKABLE, Unit.INSTANCE)
         .build();

// すでにある ItemStack を基に新しい ItemStack をビルダーで生成
ItemStack originalStack = ...;
ItemStack modifiedStack = originalStack.builder()
                                       .amount(10)
                                       .build();

// すでにある ItemStack を基に新しい ItemStack を生成（ビルダーを使用しない場合）
ItemStack originalStack = ...;
ItemStack modifiedStack = originalStack.withAmount(10);
```

## 座標系クラスの使い分け

Minestom では，位置情報を表すために，`Point` インタフェースを実装した 以下の 3 種類の座標系クラスが提供されています。

1. `Pos` クラス(`x`, `y`, `z`, `yaw`, `pitch`)  
   `double` 型の 3次元座標と `float` 型の向き情報を保持します。
   主にエンティティの位置情報を表すために使用されます。
2. `BlockPos` クラス(`x`, `y`, `z`)  
   `int` 型の 3次元座標を保持します。
   主にブロックの位置情報を表すために使用されます。
3. `Vec` クラス(`x`, `y`, `z`)  
   `float` 型の 3次元座標を保持します。
   主に物理演算やベクトル計算に使用されます。

それぞれのクラスは `Point` に定義された `asBlockPos()`，`asPos()`，`asVec()` メソッドを実装しており，互いに変換できます。
例えば，`Pos` オブジェクトから `BlockPos` オブジェクトを取得するには，`asBlockPos()` メソッドを使用します。

:::message alert
それぞれの変換メソッドは，毎回新しいオブジェクトを生成するため，頻繁な変換はパフォーマンスに影響を与える可能性があります。
そのため，`Point` インタフェースを用いて引数を受け取ったり，フィールドに保持したりしつつ，本当に必要なときだけ変換を行うことが推奨されます。

特に，Minestom の API は `Point` インタフェースを多用しているため，開発者は `Point` を積極的に活用するべきです。
:::

## プレイヤのクラスを置換してしまう

PaperMC ユーザが頻繁に利用する `Player` クラスは，我々開発者の頭をよく悩ませます。
なぜなら，`Player` クラスは（パフォーマンスを保ったまま）追加のデータを持たせられる構造ではなく，**継承して拡張することもできない**からです。
そのため `WeakHashMap<Player, MyData>` や `HashMap<UUID, MyData>` などを用いて，`Player` オブジェクトに紐づくデータを管理する必要があります。

Minestom では，**`Player` クラスを自由に拡張して新たなフィールドやメソッドを追加**できます。
例えば，以下のように `CustomPlayer` クラスを定義して独自のデータを持たせられます：

```java
class CustomPlayer extends Player {
    private int myCustomData;

    public CustomPlayer(Connection connection, GameProfile gameProfile) {
        super(connection, gameProfile);
    }

    public int getMyCustomData() {
        return this.myCustomData;
    }

    public void setMyCustomData(int myCustomData) {
        this.myCustomData = myCustomData;
    }
}
```

もちろん，これだけでは `CustomPlayer` クラスは Minestom に認識されません。
ここからが PaperMC ユーザが最も驚くポイントです。

Minestom では，`PlayerProvider` 関数インタフェースを実装することで，プレイヤの**ログイン時に生成される `Player` オブジェクトをカスタマイズ**できます。
以下のようにして，`Player` クラスの代わりに `CustomPlayer` クラスのインスタンスを生成するように Minestom に指示できます：

```java
MinecraftServer server = MinecraftServer.init();
ConnectionManager connectionManager = this.server.getConnectionManager();
connectionManager.setPlayerProvider((connection, gameProfile) -> new CustomPlayer(connection, gameProfile));
```

## イベント・ノードの概念

Minestom のイベント・システムは，PaperMC/SpigotMC のイベント・システムとは大きく異なります。
従来のイベント・システムの問題点を解決するために，Minestom では**イベント・ノード**という概念を導入しています。

### 従来のイベント・システムの問題点

PaperMC/Spigot などのサーバ・ソフトウェアや Forge などの Modding フレームワークでは，イベント・リスナを登録する際には**サーバ全体で一つの大きなイベント・リスナ・リスト**に対してリスナを追加します。

例えば `EntitySpawnEvent` に対するリスナを登録すると，そのリスナーは**すべてのプラグインで共有される一つのリスト**に追加されます。

```java
Bukkit.getPluginManager().registerEvents(new PlayerJoinListener(), plugin);
Bukkit.getPluginManager().registerEvents(new PlayerQuitListener(), plugin);
// ...
```

これでは，イベントの呼び出し時には，すべてのリスナーを走査して該当するリスナを探す必要があります。
例え `PlayerEvent` を受け取るリスナしか登録されていない場合でも，**全てのリスナを順番にチェックしなければならない**のです。

このため，イベントリスナーの数が増えると，イベントの呼び出しにかかるコストが `O(n)` で増加してしまいます。

### Minestom のイベント・ノード

Minestom では，この問題を解決するために[イベント・ノード](https://minestom.net/docs/feature/events)という概念を導入しています。
イベント・ノードは，イベントリスナーを階層的に管理するための仕組みです。
各モジュールやコンポーネントは独自のイベント・ノードを持ち，そのノードに対してリスナーを登録します。

#### イベント・ノードの作成と登録

イベント・ノードは，`EventNode` クラスにある静的メソッドを用いて作成します。
作成したイベント・ノードは，サーバの**グローバル・イベント・ハンドラ**に登録する必要があります。

```java
EventNode<Event> rootEvents = EventNode.all("myserver-root");
MinecraftServer.getGlobalEventHandler().addChild(rootEvents);
```

#### イベント・ノードへのリスナ登録

イベント・ノードに対しては，`addListener` メソッドを用いてリスナを登録します。

```java
rootEvents.addListener(PlayerJoinEvent.class, event -> {
    Player player = event.getPlayer();
    player.sendMessage("Welcome to the server!");
});
```

#### イベント・ノードの階層化とフィルタリング

イベント・ノードは階層化という概念を持っており，親ノードと子ノードの関係を構築できます。
各ノードは子ノードとリスナの両方を持っており，親ノードから子ノードへとイベントが伝播します。
以下の例では，`game-events` ノードが `rootEvents` ノードの子ノードとして追加されます。
さらに， `game-events-session1` ノードが `game-events` ノードの子ノードとして追加されます。

```java
EventNode<Event> gameEvents = EventNode.all("game-events");
rootEvents.addChild(gameEvents);

EventNode<Event> gameEventsSession1 = EventNode.all("game-events-session1");
gameEvents.addChild(gameEventsSession1);
```

さらに，各イベント・ノードには「フィルタ」を設定でき，特定の条件に一致するイベントのみをそのノードで処理するように制限できます。
フィルタは階層的に適用されるため，イベントが発生した際には，まずルートノードから始まり，フィルタに一致するノードのみが順次処理されます。
これにより，イベントの呼び出しにかかるコストが大幅に削減され，`O(log n)` に近い効率でイベントを処理できます。

例えば，以下のような構造のプロジェクトを考えてみましょう。

```
my-server
├─ world-manager
│  ├─ world-1
│  └─ world-2
├─ player-manager
│  └─ player-Player1
└─ ...
```

このとき，各モジュールやコンポーネントごとにイベント・ノードを作成し，適切なフィルタを設定することで，イベントの処理を効率化できます。

```java
// すべてのイベントを処理するルートノード
EventNode<Event> rootEvents = EventNode.all("my-server-root");
MinecraftServer.getGlobalEventHandler().addChild(rootEvents);
```

```java
// ワールド管理モジュールのノード
EventNode<Event> worldManagerEvents = EventNode.all("world-manager");
rootEvents.addChild(worldManagerEvents);

worldManagerEvents.addListener(InstanceRegisterEvent.class, event -> {
    System.out.println("New instance registered!");
});

```

```java
// ワールド 1 のノード（world-1 インスタンスに関連するイベントのみ処理）
EventNode<Event> world1Events = EventNode.type("world-1", EventFilter.INSTANCE, world -> world == world1Instance);
worldManagerEvents.addChild(world1Events);

// このリスナは，world-1 インスタンスに関連するイベントのみを処理する。
world1Events.addListener(ChunkLoadEvent.class, event -> {
    System.out.println("Chunk loaded in world-1!");
});
```

```java
// プレイヤ管理モジュールのノード
EventNode<Event> playerManagerEvents = EventNode.all("player-manager");
rootEvents.addChild(playerManagerEvents);

// Player1 のノード（Player1 に関連するイベントのみ処理）
EventNode<Event> player1Events = EventNode.type("player-Player1", EventFilter.PLAYER, player -> player.getUsername().equals("Player1"));
playerManagerEvents.addChild(player1Events);

// このリスナは，Player1 に関連するイベントのみを処理する。
player1Events.addListener(PlayerChatEvent.class, event -> {
    System.out.println("Player1 sent a message: " + event.getMessage());
});
```

各モジュールやコンポーネントが独自のイベント・ノードを持つことで，イベントの処理が効率化され，コードの可読性も向上します。
積極的にイベント・ノードを活用しましょう。

## ブロックのハンドラ

Minestom では，ブロックの動作をカスタマイズするために **ブロック・ハンドラ（`BlockHandler`）** という概念が導入されています。
`BlockHandler` は，特定のブロックがプレイヤによって設置されたり，破壊されたり，使用されたりしたときに呼び出されるコールバックを定義するためのインターフェースです。
`BlockHandler` を使用することで，チェストやドアなどの特殊なブロックの動作を実装できます。

### 簡単な `BlockHandler` の実装と登録

`BlockHandler` を実装するには，`BlockHandler` インターフェースを実装したクラスを作成します。
このインタフェースでは `Key getKey()` メソッドのみが必須です。

例えば，チェストの動作を実装する場合は，以下のようになります。

```java
class ChestBlockHandler implements BlockHandler {
    private final Inventory chestInventory;

    @Override
    public void onInteract(Interaction interaction) {
        // チェストが使用されたときの処理
        Player player = interaction.getPlayer();
        player.openInventory(this.chestInventory);
    }

    @Override
    public Key getKey() {
        // ハンドルするブロックのキーを返す
        return Key.key("minecraft:chest");
    }
}
```

作成した `BlockHandler` を Minestom に登録するには，`BlockManager#registerHandler(Key, BlockHandler)` メソッドを使用します。

```java
MinecraftServer server = MinecraftServer.init();
BlockManager blockManager = server.getBlockManager();
blockManager.registerHandler(Key.key("minecraft:chest"), ChestBlockHandler::new);
```

### チックの対象となる `BlockHandler` の実装

かまどや醸造台などの一部のブロックは，定期的に処理（チック処理）を行う必要があります。
これらのブロックの動作を実装する場合は，`BlockHandler#tick()` メソッドをオーバーライドして，チック処理を実装します。

```java
class FurnaceBlockHandler implements BlockHandler {
    @Override
    public void tick(Instance instance, BlockPosition position, Block block) {
        // かまどのチック処理
        // 例：燃料の消費，アイテムの精錬など
    }
    
    @Override
    public void isTickable() {
        return true; // チック対象であることを示す
    }

    @Override
    public Key getKey() {
        return Key.key("minecraft:furnace");
    }
}
```

## スレッド・モデル

Minecraft サーバの多くは，メインスレッドと呼ばれる単一のスレッド上で全てのゲームロジックを実行します。
これにより，ゲームの状態が一貫性を保ち，複雑な同期問題を回避できます。

しかしながら，このスレッド・モデルは，パフォーマンスのボトルネックとなることがあり，様々な議論を呼んでいます。
実際に，PaperMC では，非同期タスクを用いて一部の処理をメインスレッドから切り離すことが推奨されています。

Minestom では，**完全に非同期なスレッド・モデル**を採用しています。
これは，ゲームロジックが常にあらゆるスレッドで実行される可能性があることを意味します。

この設計により，Minestom は高いパフォーマンスとスケーラビリティを実現していますが，開発者はスレッド・セーフなコードを書く必要があります。
例えば，あるエンティティの位置を更新するコードが，複数のスレッドから同時に実行される可能性があります。

### 知っておくべきこと

ここで，Minestom のスレッドと付き合う上で知っておくべき重要なポイントが２点あります。

１つは**デフォルトでは `1` つのスレッドで動作する**こと，もう一つは **`ThreadDispatcher` の概念**です。

Minestom はデフォルトでは単一のスレッドで動作します。 これは，PaperMC のメインスレッドと同様に，ゲームロジックが一貫性を保つことを意味します。
この設定はシステム・プロパティの `minestom.dispatcher-threads` の値を変更することで変更できます。

:::message
Minestom には，この他にも（ドキュメントにはない）様々なプロパティがあります。
代表的なものは サーバの TPS(`minestom.tps`)，１チックあたりの最大許容パケット量（`minestom.packets-per-tick`）などです。
詳細は [Minestom のソースコード](https://github.com/Minestom/Minestom/blob/master/src/main/java/net/minestom/server/ServerFlag.java) をご覧ください。
:::

さて，ここからは使用するスレッド数を `2` 以上に増やした場合について説明します。

Minestom では，**`ThreadDispatcher`** という概念が導入されています。
`ThreadDispatcher` は，処理を行う必要のある事物の集合が何らかの基準でグループ化され，それぞれのグループが特定のスレッドに割り当てられる仕組みです。

例えば，インスタンス（ワールド）には一つの `ThreadDispatcher<Chunk, Entity>` が割り当てられています。
この例では，`Entity` が `Chunk` に属しているため，特定のチャンクに存在する全てのエンティティの処理は，そのチャンクに割り当てられた同じスレッドで実行されます。
逆に言うと，**異なるチャンクに存在するエンティティの処理は，異なるスレッドで実行される**可能性があります。

この設計により，Minestom は高い並列性を実現しています。

## 便利ライブラリ集

Minestom のエコシステムには，Minestom を補完する様々な便利ライブラリが存在します。

ライブラリは，[Minestom Wiki の Libraries](https://minestom.net/libraries) や [Awesome Minestom](https://minestom.rocks/) などで紹介されています。
以下に，私が実際に利用してみて特に有用だと感じたライブラリをいくつか紹介します。

### MinestomPvP

[MinestomPvP](https://github.com/TogAr2/MinestomPvP) は，Minestom 用の PvP ミニゲーム・フレームワークです。
このライブラリは，基本的な PvP ロジックを提供しており，開発者は独自の PvP ゲームを迅速に構築できます。

以下のような機能が含まれています：
- ダメージの計算
- 投射物（矢，エンダー・パールなど）の処理
- 食べ物の効果
- 落下ダメージ
- TNT
- 武器のエンチャント効果

さらに，古い Minecraft バージョンの PvP メカニクスを再現するためのオプションも提供されています。
特に `1.8.9` の PvP メカニクスを再現する機能は，レトロな PvP サーバを構築したい場合に非常に有用で，これは実用に耐えうるクオリティです。

追加されるイベントは約 20 種類もあり，この中にはキャンセルが可能な `EntityKnockbackEvent` や `FinalAttackEvent` なども含まれています。

#### 導入方法

MinestomPvP を導入するには，Maven central リポジトリから依存関係を追加します。

```xml
<dependency>
    <groupId>io.github.togar2</groupId>
    <artifactId>MinestomPvP</artifactId>
    <version>2025.12.29-1.21.11</version>
</dependency>
```

```kotlin
implementation("io.github.togar2:MinestomPvP:2025.12.29-1.21.11")
```

#### 基本的な使い方

MinestomPvP を使用するには，`MinestomPvP#init()` メソッドを呼び出し，PvP システムを初期化します。
さらに `CombatFeatureSet` クラスから必要な機能を作成し，グローバル・イベント・ハンドラに登録します。

```java
MinestomPvP.init();

// 1.8.9 の PvP メカニクスを使用する場合
CombatFeatureSet legacy = CombatFeatureSet.legacyVanilla();
MinecraftServer.getGlobalEventHandler().addChild(legacy.createNode());

// 最新の PvP メカニクスを使用する場合
CombatFeatureSet modern = CombatFeatureSet.modernVanilla();
MinecraftServer.getGlobalEventHandler().addChild(modern.createNode());
```

#### 機能をカスタマイズする

MinestomPvP では，特定の機能を有効化・無効化したり，差し替えたりできます。
以下の例は，落下ダメージを無効化する方法です。

```java
CombatFeatureSet customSet = 
        CombatFeatureSet.getVanilla(CombatVersion.MODERN, DifficultyProvider.DEFAULT)
                        .remove(FeatureType.FALL)
                        .build();

MinecraftServer.getGlobalEventHandler().addChild(customSet.createNode());
```

#### 注意：プレイヤ・クラスをインジェクションしている場合

先程述べた方法で `Player` クラスを拡張している場合には注意が必要です。
MinestomPvP は 独自の `CombatPlayerImpl` を既に `Player` クラスとして登録しているため，`PlayerProvider` を用いて `Player` クラスを差し替えると，MinestomPvP の機能が正しく動作しなくなります。

この場合には，独自の `Player` クラスを `Player` ではなく `CombatPlayerImpl` を継承して実装する必要があります。

```diff java
- class CustomPlayer extends Player {
+ class CustomPlayer extends CombatPlayerImpl {
    // ...
}

MinecraftServer server = MinecraftServer.init();
server.getConnectionManager().setPlayerProvider(
        (connection, gameProfile) -> new CustomPlayer(connection, gameProfile)
);
```

#### リポジトリなど

- GitHub: [TogAr2/MinestomPvP](https://github.com/TogAr2/MinestomPvP)
- Maven Central: [io.github.togar2:MinestomPvP](https://central.sonatype.com/artifact/io.github.togar2/MinestomPvP)

### Blocks and Stuff

[Blocks and Stuff](https://github.com/everbuild-org/blocks-and-stuff) は，Minestom 用のブロックおよび流体の動作ライブラリです。
ブロックを正しい方向に設置したり，ドアやトラップドアなどの特殊なブロックの動作を実装したりするための便利な機能が提供されています。

さらに，ブロックを破壊した際のドロップ処理や，液体の流動処理などもサポートしています。

とりあえず導入しておけば間違いない，Minestom 開発者必携のライブラリです。

#### 導入方法

Blocks and Stuff を導入するには，[Asorda Repository](https://mvn.everbuild.org/#/public) を Maven リポジトリに追加し，依存関係を追加します。

```xml
<repositories>
    <repository>
        <id>asorda-repo</id>
        <url>https://mvn.everbuild.org/public/</url>
    </repository>
</repositories>

<dependency>
    <groupId>org.everbuild.blocksandstuff</groupId>
    <artifactId>blocksandstuff-blocks</artifactId>
    <version>1.9.0-SNAPSHOT</version>
</dependency>
<dependency>
  <groupId>org.everbuild.blocksandstuff</groupId>
  <artifactId>blocksandstuff-fluids</artifactId>
  <version>1.9.0-SNAPSHOT</version>
</dependency>
```

```kotlin
repositories {
    maven("https://mvn.everbuild.org/public/")
}

implementation("org.everbuild.blocksandstuff:blocksandstuff-blocks:1.9.0-SNAPSHOT")
implementation("org.everbuild.blocksandstuff:blocksandstuff-fluids:1.9.0-SNAPSHOT")
```

#### 基本的な使い方

Blocks and Stuff を使用するには，**MinecraftServer.init() の後に` 各モジュールの `init()` メソッドを呼び出して初期化します。

```java
MinecraftServer server = MinecraftServer.init();

BlockPlacementRuleRegistrations.registerDefault();
BlockBehaviorRuleRegistrations.registerDefault();
PlacedHandlerRegistration.registerDefault();
BlockPickup.enable();

MinestomFluids.enableFluids();
MinestomFluids.enableVanillaFluids();
```

#### リポジトリなど

- GitHub: [everbuild-org/blocks-and-stuff](https://github.com/everbuild-org/blocks-and-stuff)
- Asorda Repository: [org.everbuild.blocksandstuff](https://mvn.everbuild.org/#/public/org.everbuild.blocksandstuff/blocksandstuff-blocks)

### Spark

[Spark](https://spark.lucko.me/) は，Minecraft サーバのパフォーマンスを監視・分析するためのプロファイラです。
高機能かつ軽量なこともあり，GitHub では 1,200 を超えるスターが付けられ，業界標準のプロファイラとして広く認知されています。

Spark は，CPU 使用率，メモリ使用量，ガベージコレクションの統計情報など，様々なパフォーマンス指標を収集します。
さらに，Spark は，Minestom サーバの特定の部分で発生しているパフォーマンス・ボトルネックを特定するのに役立ちます。

公式による Minestom サポートはありませんが，[有志による Minestom 対応版](https://github.com/LooFifteen/spark) が存在します。

#### 導入方法

Spark を導入するには，３つの追加リポジトリを Maven リポジトリに追加し，依存関係を追加します。

```xml
<repositories>
    <repository>
        <id>lucko-repo</id>
        <url>https://repo.lucko.me/</url>
    </repository>
    <repository>
        <id>sonatype-snapshots</id>
        <url>https://oss.sonatype.org/content/repositories/snapshots/</url>
    </repository>
    <repository>
        <id>hypera-repo</id>
        <url>https://repo.hypera.dev/snapshots/</url>
    </repository>
</repositories>
<dependency>
    <groupId>me.lucko</groupId>
    <artifactId>spark-minestom</artifactId>
    <version>1.10-SNAPSHOT</version>
</dependency>
```

```kotlin
repositories {
    maven("https://repo.lucko.me/")
    maven("https://oss.sonatype.org/content/repositories/snapshots/")
    maven("https://repo.hypera.dev/snapshots/")
}

implementation("me.lucko:spark-minestom:1.10-SNAPSHOT")
```

#### 基本的な使い方

Spark を使用するには，`SparkMinestom#builder()` メソッドを呼び出してビルダーを取得し，`enable()` メソッドで Spark インスタンスを作成します。

```java
Path configurationPath = Path.of("spark");
SparkMinestom spark = SparkMinestom.builder(configurationPath)
        .command(true)  // コマンドを有効にする
        .permissionHandler((sender, permission) -> true) // 権限ハンドラを設定する
        .enable();
```

これにより，コマンドが登録されてサーバのパフォーマンス情報を収集できるようになります。

#### リポジトリなど

- GitHub: [LooFifteen/spark](https://github.com/LooFifteen/spark)
- GitHub（オリジナル）: [lucko/spark](https://github.com/lucko/spark)
- 公式サイト: [Spark - The Minecraft Profiler](https://spark.lucko.me/)

## まとめ

いかがでしたか？

Minestom は，様々なユニークな設計思想と機能を持つ Minecraft サーバ・フレームワークです。
何も無いところから自分だけの Minecraft サーバを構築したい開発者にとって，Minestom は非常に魅力的な選択肢となるでしょう。

Minestom の設計思想や機能を理解し，効果的に活用することで，より良い Minecraft サーバを構築できることを願っています。
この記事が，Minestom の理解を深める一助となれば幸いです。

次回の記事が公開されましたら，この記事の下にリンクが表示されますので，ぜひそちらもご覧ください。

では，よい Minecraft ライフを！
