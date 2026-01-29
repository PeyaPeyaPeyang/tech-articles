---
title: "【Minecraft】Minestom でのテストは驚くほど簡単"
emoji: "✔️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["minecraft", "minestom", "gameserver", "testing"]
publication_name: "kun_lab"
published: true
---

はじめまして。Peyang （ぺやんぐ）と申します。
普段は JavaJava しているのですが，息抜きに Minecraft についての記事も書いています。

前回の記事では，Minecraft サーバ・ライブラリの一つである [Minestom](https://minestom.net/) を紹介し，その特徴や利点について筆者の独断と偏見を交えつつ解説しました。
この記事では，Minestom を利用した開発でのテスト手法について解説します。
皆々様の Minecraft サーバ開発の一助となれば幸いです。

前回の記事はこちら：

https://zenn.dev/kun_lab/articles/minestom-knowledges

## Minestom あれこれ

Minestom についての詳細は，ぜひぜひ前回の記事をご覧いただきたいのですが，ここでは簡単に Minestom の特徴をおさらいします。

Minestom は，Java で書かれた軽量かつ高性能な Minecraft サーバ・ライブラリです。
あくまでも，Minecraft サーバを構築するためのライブラリであり，**完全なサーバ・ソフトウェアではありません**。

Minestom は，Minecraft プロトコルの実装やエンティティ管理，ワールド管理，イベントシステムなど，Minecraft サーバを構築するために必要な基本的な機能を提供します。
開発者は自分のニーズに合わせたカスタム Minecraft サーバを構築できます。

## テストの重要性

プロダクションを支えるコードの品質を保証するためには，テストが欠かせません。
Minestom を利用した開発においても，テストは重要な役割を果たします。


### 既存の Minecraft サーバ・ソフトウェアでのテスト

ここで，通常 PaperMC などの既存の Minecraft サーバ・ソフトウェアを利用して開発した場合のテスト手法について考えてみましょう。

まず，通常環境でのユニット・テストは非常に困難です。
[Mockito](https://site.mockito.org/) などのモックライブラリを利用しても，Minecraft サーバ・ソフトウェアの膨大かつ複雑な内部構造を完全に模倣することは難しいのです。
特に，エンティティの挙動やワールドの状態など，動的な要素は全くと言ってよいほどテストできません。

例の [Scenamatica](https://scenamatica.kunlab.org/) は E2E テストを支援する強力なツールですが，ユニット・テストには対応していません。

## Minestom でのテスト

Minestom を利用した開発では，驚くほど簡単にユニット・テストを実装できます。
しかも軽量で高速に実行できるため，開発サイクルを大幅に改善できます。

これには [JUnit 5](https://junit.org/junit5/)と [Minestom/testing](https://minestom.net/) の組み合わせを使用します。

では，これらを用いて Minestom ベースの Minecraft サーバでユニット・テストを実装する方法を見ていきましょう。

### テスト環境のセット・アップ

まず，[JUnit 5](https://junit.org/junit5/) と [Minestom](https://minestom.net/) の依存関係をプロジェクトに追加します。
Maven を使用している場合，`pom.xml` に以下の依存関係を追加します。

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-engine</artifactId>
    <version>6.0.2</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>net.minestom</groupId>
    <artifactId>testing</artifactId>
    <version>2026.01.08-1.21.11</version>
    <scope>test</scope>
</dependency>
```

Gradle を使用している場合，`build.gradle.kts` に以下の依存関係を追加します。

```kotlin
testImplementation(platform("org.junit:junit-bom:6.0.2"))
testImplementation("org.junit.jupiter:junit-jupiter")
testImplementation("net.minestom:testing:2026.01.08-1.21.11")
```

:::message
それぞれの最新バージョンは，以下のリンクから確認できます（記載しているバージョンは記事執筆時点のものです）。
- [JUnit 5 - Maven Central](https://search.maven.org/artifact/org.junit.jupiter/junit-jupiter-engine)
- [Mockito - Maven Central](https://search.maven.org/artifact/org.mockito/mockito-core)
- [Testing - Maven Central](https://search.maven.org/artifact/net.minestom/testing)

なお，`testing` は Minestom のバージョンと同じにする必要があります。
:::

### Minestom テスト・フレーム・ワークについて

Minestom では，標準でテスト用フレーム・ワークが提供されています。
このフレーム・ワークは JUnit とシームレスに統合されており，サーバの起動や停止を意識することなくテストを実行できます。
さらに，仮想的なプレイヤやワールドを簡単に作成できるため，複雑のテストも容易に実装できます。


### 基本的な使い方

Minestom のテスト・フレーム・ワークを利用するには，テスト・クラスに `@EnvTest` アノテーションを付与します。
さらに，Minestom を使用するテスト・メソッドには，`Env` オブジェクトを引数として受け取るようにします。

```java
import net.minestom.testing.EnvTest;
import org.junit.jupiter.api.Test;

@EnvTest
class MyMinestomTest {
    @Test
    void testSomething(Env env) {
        // テストコード
    }
}
```

`Env` オブジェクトは，テスト環境を操作するための様々なメソッドを提供します。
以下に，いくつかの基本的なメソッドを紹介します。

| メソッド                                                     | 説明                                           |
|----------------------------------------------------------|----------------------------------------------|
| `Instance createFlatInstance()`                          | スーパー・フラットなワールドを作成します。                        |
| `Instance createEmptyInstance()`                         | 空のワールドを作成します。                                |
| `Player createPlayer(Instance, Pos)`                     | 指定したワールドと位置に仮想的なプレイヤを作成します。                  |
| `ServerProcess process()`                                | テスト用のサーバ・プロセスを取得します。                         |
| `FlexibleListener<E extends event> listen(Class<E>)`     | 指定したイベントのリスナを登録します。                          |
| `void tick()`                                            | サーバのティックを 1 回進めます。                           |
| `boolean tickWhile(BooleanSupplier, @Nullable Duration)` | 指定した条件が満たされるまでサーバのティックを進めます。タイムアウトを別途指定できます。 |

### 例：プレイヤをスポーンさせる

テスト用のプレイヤをスポーンさせてみましょう。

詳しい説明は後にして，コード例を以下に示します：
```java

import net.minestom.testing.EnvTest;
import net.minestom.server.instance.Instance;
import net.minestom.server.coordinate.Pos;
import net.minestom.server.entity.player.Player;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

@EnvTest
class PlayerSpawnTest {
    @Test
    void testPlayerSpawn(Env env) {
        // スーパー・フラットなワールドを作成
        Instance instance = env.createFlatInstance();
        // (0, 10, 0) にプレイヤをスポーンさせる
        Player player = env.createPlayer(instance, new Pos(0, 10, 0));
        // プレイヤが正しい位置にスポーンしていることを確認
        assertEquals(new Pos(0, 10, 0), player.getPosition());
    }
}
```

驚くほど簡単に，Minestom ベースの Minecraft サーバでユニット・テストを実装できました。

ここで，少し詳しい解説をしておきましょう。

まず，このフレーム・ワークは `Env` を受け取るメソッドを検知したら，自動的に Minestom サーバを起動します。
そのため，テスト・メソッド内では Minestom サーバが起動していることを前提にコードを書けます。

`Instance Env#createFlatInstance()` メソッドは，高さ 0～40 に石ブロックが敷き詰められたスーパー・フラットなワールドを作成します。
なお，別途 `ChunkLoader` を指定することで，カスタムなワールドを作成することも可能です。

`Player Env#createPlayer(Instance, Pos)` メソッドは，指定したワールドと位置に仮想的なプレイヤを作成します。
このプレイヤは，Minestom サーバに接続されているわけではなく，内部で作成・管理されているだけです。
そのため，OS のリソースを消費せずに，軽量かつ高速にプレイヤを作成できます。

:::message
`createPlayer` を用いてプレイヤを作成した場合，ログイン処理が省略されます。
関連イベント自体は発火するものの，サーバ管理するプレイヤの状態はすぐに `PLAY` 状態になります。
通常行われる `HANDSHAKE` -> `STATUS` -> `LOGIN` -> `PLAY` の各状態遷移は発生しませんので，注意してください。
:::

### 例：サーバのティックを進める

テスト環境の Minestom サーバは標準ではティックが進みません。
一見不便に思える仕様ですが，任意のタイミングで好きなだけティックを進められるため，テストの制御性が向上します。

では，サーバのティックを進める例を見てみましょう。

```java
import net.minestom.testing.EnvTest;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;
import net.minestom.server.instance.Instance;
import net.minestom.server.coordinate.Pos;
import net.minestom.server.entity.player.Player;
import net.minestom.server.entity.EntityType;
import net.minestom.server.event.player.PlayerMoveEvent;
import java.time.Duration;
import java.util.concurrent.atomic.AtomicBoolean;

@EnvTest
class ServerTickTest {
    @Test
    void testServerTick(Env env) {
        // 空のワールドを作成
        Instance instance = env.createEmptyInstance();
        // (0, 10, 0) にプレイヤをスポーンさせる
        Player player = env.createPlayer(instance, new Pos(0, 10, 0));
        player.setGravity(true);
        
        // プレイヤが地面に到達するまでサーバのティックを進める
        // そのために，フラグを用いて地面到達を検知します。
        AtomicBoolean onGround = new AtomicBoolean(false);
        player.eventNode().addListener(PlayerMoveEvent.class, e -> {
            if (e.getPlayer().isOnGround()) {
                onGround.set(true);
            }
            
            // 到達した瞬間の処理をここに書けます。
            if (e.getPlayer().isOnGround()) {
                System.out.println("Player has reached the ground!");
            }
        });
        env.tickWhile(() -> !onGround.get(), Duration.ofSeconds(5));
        // プレイヤが地面に到達していることを確認
        assertTrue(player.isOnGround());
        
        // 重力に到達した"瞬間"に何らかの処理を行えます。
    }
}
```

この例では，空のワールドにプレイヤをスポーンさせ，重力を適用しています。
その後，`Env#tickWhile` メソッドを用いて，プレイヤが地面に到達するまでサーバのティックを進めています。
`Env#tickWhile` メソッドは，指定した条件が満たされるまでサーバのティックを進めます。

このように，自由なタイミングでティックを進められるため，物理演算や時間経過に依存する処理のテストが容易になります。
（通常は Listener を追加するまでにティックが進んでしまい，イベントが発火しないといったことが発生するかもしれませんが，このフレーム・ワークではその心配は無用です。）

## まとめ

いかがでしたか？

今回は，Minestom を利用した Minecraft サーバ開発におけるテスト手法について解説しました。
Minestom のテスト・フレーム・ワークを利用することで，軽量かつ高速にユニット・テストを実装できることがお分かりいただけたかと思います。
Minestom を利用した開発において，テストは品質保証の重要な要素です。
ぜひ，Minestom のテスト・フレーム・ワークを活用して，高品質な Minecraft サーバを構築してください。

なお，今回は非常に軽めな記事となりました。私が執筆した若干20本の記事はすべて重めでしたから，ちょうど良いバランスかと思います。
今後はさらに踏み込んだ内容の記事や，「知らないと損！？ Minecraft/Spigot/PaperMC に存在するヘンな仕様一覧」などの記事も執筆予定です。

それでは，良き Minecraft ライフを！
