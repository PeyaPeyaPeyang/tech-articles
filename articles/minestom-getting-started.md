---
title: "【Minecraft】Minestom のすゝめ"
emoji: "⛏️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["minecraft", "minestom", "gameserver", "papermc"]
publication_name: "kun_lab"
published: true
---

はじめまして。Peyang （ぺやんぐ）と申します。
普段は JavaJava しているのですが，息抜きに Minecraft についての記事も書いています。

最近になってですが，所属している組織「KUN Lab」の Publication ができました。
この記事は，そちらの初回記事として執筆しています。

## 端書

この記事では，Minecraft サーバ・ライブラリの一つである [Minestom](https://minestom.net/) を紹介し，その特徴や利点，使い心地などについて
筆者の独断と偏見を交えつつ解説する試みを行います。
皆々様の Minecraft サーバ開発の一助となれば幸いです。

## Minestom とは？導入するメリットとは？調べてみました！

[Minestom](https://minestom.net/) は，Java で書かれた軽量かつ高性能な Minecraft サーバ・ライブラリです。
あくまでも，Minecraft サーバを構築するためのライブラリであり，**完全なサーバ・ソフトウェアではありません**。

Minestom は，Minecraft プロトコルの実装やエンティティ管理，ワールド管理，イベントシステムなど，Minecraft サーバを構築するために必要な基本的な機能を提供します。
これにより，開発者は自分のニーズに合わせたカスタム Minecraft サーバを構築できます。

![Minestom](/images/minestom-getting-started/minestom.png)
↑ Minestom Wiki(https://minestom.net/) より引用

### Minestom vs バニラ実装

Minestom の最大な特徴は，**既存の Minecraft のゲームロジックを一切実装していない**点にあります。
これは，例えばチェストを右クリックしても何も起きませんし，プレイヤが再ログインした際には全てのインベントリが空っぽの状態で復元されることを意味します。

バニラの Minecraft に慣れた我々にとっては，全く意味のわからない，なぜ中途半端な状態で提供するのか理解に苦しむ仕様でしょうか。
しかしながら，Minestom の設計思想は，**開発者に完全な自由度を提供すること**にあります。

### Minestom の利点

**開発者に完全な自由度を提供する**とはどういうことでしょうか。
これは，文字通り Minestom を使用することで，**Minecraft サーバの全ての側面を自分で実装できる**ことを意味します。

ゲームに**不要な機能が一切ない**，純粋な Minecraft サーバの基盤を提供することで，開発者は自分のニーズに合わせたカスタム Minecraft サーバを構築できるのです。

### 「不要な機能が一切ない」とは

例えば，純粋なテトリスを Minecraft サーバ上で実装することを考えましょう。
プレイヤはワールド内を移動し，何らかの操作方法（コマンドやアイテムの使用など）でテトリスの操作を行います。

Minecraft のバニラ実装では，このようなゲームを**余計なパフォーマンスの低下**や**余計なコードの肥大化**の無しに実装することは非常に困難です。

知っての通り，純粋なゲーム「テトリス」には一切の Mob やアイテム，インベントリ，クラフト，建築などの要素は不要です。
テトリスにはブタは居ませんし，村人が勝手に村を作って住まうことも，少なくとも私の知る限りではありません。

なるほど，では バニラ Minecraft にあるこのような機能を止めてしまえば良いのでしょうか。いいえ，そうは問屋は卸しません。  
バニラ Minecraft は，これらの機能を止めるための設定オプションを提供していないからです。

これを停止するには，NMS(`net.minecraft.server` パッケージ配下にある非公開 API)を ~~黒魔術~~リフレクションなどで操作し，強引に無効化する必要があります。
それを行ってしまうと，例えばバージョンが変わった際に動かなくなったり，予期せぬ副作用が発生したりするリスクがあります。
いずれにせよ，その機能に到達する `tick()` は毎度呼び出されるわけですから，これは完全なソリューションとは言えません。

:::message alert
ここでは「バニラ Minecraft の機能」を便宜上「不要な機能」と呼んでいますが，これは**ミニ・ゲーム開発者が実装したくない，或いは必要としない機能**を指しています。
もちろん，純粋な Minecraft は楽しいですし，それをプレイするのも素晴らしい体験です。
しかしながら，特定のゲーム・モードやミニ・ゲームを実装する際には，バニラ Minecraft の全ての機能が必要とは限りません。
:::

### Minestom の出番

こんなニーズに Minestom はしっかりと応えます。

Minestom では，前述のような機能は一切ありません。 ブタは動きませんし，建造物の生成もありません。
Minestom は，**Minecraft サーバの基盤として必要な最低限の機能のみを提供**します。

既存の Minecraft サーバ・ソフトウェアの拡張・開発は，このような機能を削ぐことから始まる「引き算の開発」でしたが，Minestom では最初から不要な機能が無いため，「足し算の開発」に集中できます。
無論，メリットばかりではなく，**全ての機能を自分で実装する必要がある**というデメリットもあります。

例えば，インベントリを DB に保存したい場合は，自分でインベントリ管理機能を実装し，DB に保存するコードを書く必要があることでしょう。
これは大変な作業ですが，**標準の プレイヤ・データの保存や読み込みをせき止める必要がない**分，DB 保存の実装に集中できます。

Minecraft のロジックは詳細のわからないブラック・ボックスです。
そこで，最初から何も無い Minestom を使用することで，**自分のニーズに合わせたカスタム Minecraft サーバを構築できる**のです。
Minestom を導入する理由もここにあるでしょう。

### Minestom のデメリット

もちろん，Minestom にはデメリットも存在します。
以下主なデメリットを挙げてみました。

1. **Minecraft に期待される機能が一切無い**  
   → 場合によっては，Minecraft の或る機能を自分で実装する必要があります。
2. **Spigot/Bukkit および PaperMC との互換性が一切無い**  
   → **既存のプラグインは一切流用できません**。移植も容易いことではありません。
   **導入を検討する際には，この点を十分に考慮する必要があります**。
3. **ドキュメントやコミュニティがまだ成熟していない**
   → 最近は成熟しつつありますが，まだまだ情報が少ないです。手探り手探りで進める必要があります。
   
特に，２点目の「既存のプラグインは一切流用できない」という点は，Minestom を導入する際に最も重要な考慮事項となります。
既存のプラグインを流用できないということは，例えば，権限管理やワールド管理，経済システムなどの基本的な機能も自分で実装する必要があることを意味します。

:::message
権限管理については，[LuckPerms の Minestom バック・ポート版](https://github.com/LooFifteen/LuckPerms)
があるので，そちらを利用することも検討できます。
:::

## Minestom ことはじめ

ここまででは Minestom の概要や特徴，利点，欠点などについて述べてきました。
ここからは，Minestom を実際に導入してみようとおもいます。


### Minestom のよくある誤解

さて，Minestom を入れようと思った時，我々が（特に私が）最初に行うとすることは
「公式サイトから `.jar` をダウンロードし，それを適当なディレクトリに置いて，`java -jar minestom.jar` で起動する」
ことではないでしょうか。

残念ながら，Minestom は起動可能な `JAR` を提供していません。
Minestom はあくまでもライブラリであり，**自分でメイン・クラスを実装して起動可能な JAR を作成する必要があります**。

### 導入してみよう

Minestom は [Maven Central Repository](https://central.sonatype.com/artifact/net.minestom/minestom) に公開されているため，
以下のような文言を Maven や Gradle のビルド・スクリプトに追加することで，簡単に導入できます。

```xml
<dependency>
    <groupId>net.minestom</groupId>
    <artifactId>minestom</artifactId>
    <version>2026.01.08-1.21.11</version>
</dependency>
```

```kotlin
dependencies {
    implementation("net.minestom:minestom:2026.01.08-1.21.11")
}
```

:::message alert
なお，ここで紹介しているバージョンは， 2026 年 1 月時点での最新バージョンです。
これには別途 [Java SE 25](https://www.oracle.com/java/technologies/javase/jdk25-archive-downloads.html) 以降が必要ですから，
事前にインストールしておいてください。
:::

### 起動してみよう

これを追加した後，以下のようなメイン・クラスを実装します。

```java
import net.minestom.server.MinecraftServer;

class Main {
    public static void main(String[] args) {
        // Minestom の初期化
        MinecraftServer server = MinecraftServer.init();
        
        // 0.0.0:25565 でサーバを起動
        server.start("0.0.0.0", 25565);
        
        // MinecraftServer#start はブロッキングしないので，ここに到達する
        System.out.println("Minestom server started!");
    }
}
```

このようにすると，大体 100ms ほどで Minestom サーバが完全に起動します。
この時点でのメモリ消費量は，およそ **50MB** 程度です。

では，早速ログインしてみましょう！

![ERROR DURING LOGIN](/images/minestom-getting-started/error-during-login.png)

やや！謎のエラーが発生しましたね。
コンソールを見てみましょう。

```
java.lang.NullPointerException: You need to specify a spawning instance in the AsyncPlayerConfigurationEvent
	at net.minestom.server.utils.validate.Check.notNull(Check.java:21)
	at net.minestom.server.network.ConnectionManager.doConfiguration(ConnectionManager.java:246)
	at net.minestom.server.listener.preplay.LoginListener.lambda$executeConfig$0(LoginListener.java:250)
	at java.base/java.lang.VirtualThread.run(VirtualThread.java:456)
```

なるほど，`AsyncPlayerConfigurationEvent` でスポーンするインスタンスが指定されていないため，`NullPointerException` が発生しているようです。
Minestom では，プレイヤがログインした際にスポーンするインスタンスを自分で指定する必要があります。

### インスタンスを作成してみよう

インスタンスとは，Minestom が管理するサーバの最小単位のことです。
各インスタンスには１つのワールド，１つのプレイヤ・リストが紐づけられます。
同じインスタンスに所属するプレイヤのみがTab メニューに表示されます（デフォルト）。

プレイヤは，ログイン時に指定されたインスタンス（に紐づけられたワールド）にスポーンします。
そのため，事前にインスタンスを作成し，プレイヤがスポーンするインスタンスを指定する必要があります。

以下の例は，簡単なインスタンス，特にワールドには一面石ブロックが広がるだけのインスタンス，を作成する最小のコード例です。

```java
import net.minestom.server.MinecraftServer;
import net.minestom.server.instance.Instance;
import net.minestom.server.event.player.AsyncPlayerConfigurationEvent;
import net.minestom.server.coordinate.Pos;
import net.minestom.server.instance.block.Block;

class Main {
    public static void main(String[] args) {
        MinecraftServer server = MinecraftServer.init();

        // 新しいインスタンスを作成（MinecraftServer#getInstanceManager() から取得可能）
        Instance instance = MinecraftServer.getInstanceManager().createInstanceContainer();

        // チャンク・生成器を設定（ここでは一面石ブロックにする）
        instance.setGenerator(unit -> unit.modifier().fillHeight(0, 1, Block.STONE));

        // プレイヤがログインした際にスポーンするインスタンスを指定
       MinecraftServer.getGlobalEventHandler().addListener(
           AsyncPlayerConfigurationEvent.class,
           e -> {
              e.setSpawningInstance(instance);
              e.getPlayer().setRespawnPoint(new Pos(0, 2, 0));
           }
       );

        
        server.start("0.0.0.0", 25565);
    }
}
```

このコードを実行し，改めてログインしてみましょう。

![welcome to the stone world](/images/minestom-getting-started/welcome-to-the-stone-world.png)
↑影の関係で真っ黒に見えていますが，これは確かに石ブロックです……

おお，無事にログインできましたね！
このように，Minestom では自分でインスタンスを作成し，プレイヤがスポーンするインスタンスを指定する必要があります。

:::message
これらのことは，[Minestom Wiki](https://minestom.net/docs/setup/your-first-server) により詳しく解説されていますので，ぜひそちらもご覧ください。
:::

### 付録：実行可能 JAR の作成方法

Minestom はあくまでもライブラリであり，実行可能 JAR を提供していません。
そのため，自分で実行可能 JAR を作成する必要があります。
ここでは，Gradle を使用して実行可能 JAR を作成する方法を紹介します。

Gradle で作成した jar ファイルには，（何もしない限りは）ライブラリは同梱されません。
そこで，`Shadow` プラグインを使用して，依存関係を全て含んだ実行可能 JAR を作成します。

以下のように `build.gradle.kts` を編集します。

```kotlin
plugins {
    id("java")
    id("com.gradleup.shadow") version "9.3.1"
}

java {
    toolchain {
        // Minestom 2026.01.08-1.21.11 は Java 25 が必要です
       languageVersion.set(JavaLanguageVersion.of(25))
    }
}


// 以下は，適宜変更してください
group = "com.example"
version = "1.0.0"

repositories {
    mavenCentral()
}

dependencies {
    implementation("net.minestom:minestom:2026.01.08-1.21.11")
}

tasks {
    jar {
        manifest {
            attributes {
                "Main-Class" to "com.example.Main" // ← ここを自分のメイン・クラスに変更してください
            }
        }
    }

    build {
        dependsOn(shadowJar)
    }
   
    shadowJar {
        archiveFileName.set("my-minestom-server.jar") // ← ここを好きなファイル名に変更してください
        mergeServiceFiles()
    }
}
```

### まとめ

いかがでしたか？

Minestom は，Minecraft サーバを構築するための強力なライブラリであり，開発者に完全な自由度を提供します。
今までは「引き算の開発」だった Minecraft サーバ開発が，「足し算の開発」に変わる，そんな可能性を秘めています。

さて，今回の記事では Minestom を導入したところで終わりましたが，私が行ったカスタマイズやテクニック，ナレッジを今後の記事で紹介していきたいと考えています。
この記事が，皆々様の Minecraft サーバ開発の一助となれば幸いです。

次回の記事が公開されましたら，この記事の下にリンクが表示されますので，ぜひそちらもご覧ください。

では，よい Minecraft ライフを！

### 次回リンク

https://zenn.dev/kun_lab/articles/minestom-knowledges
