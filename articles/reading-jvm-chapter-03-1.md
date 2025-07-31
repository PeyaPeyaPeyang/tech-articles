---
title: "JVM を読む | JVM をハックする その２ - 前提知識構築編"
emoji: "🖥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["java", "jvm", "jal"]
published: true
---

前回の続きです。前回はこちらから。

https://zenn.dev/peyang/articles/reading-jvm-chapter-03

このシリーズは，JVM の仕様書を読み解くためのガイドとして構成しています。
JVM の仕様書は非常に長大で難解な内容が多いため，各セクションごとに要点をまとめていきます。
また，JVM の内部構造や動作原理を知ることで，Java のパフォーマンスやセキュリティ，メモリ管理の仕組みを深く理解する試みです。

シリーズはこちらから。

https://zenn.dev/peyang/articles/reading-jvm-chapter-00

## 第三章 Compiling for the Java Virtual Machine

JVM の仕様書の第３章は「Java Virtual Machine のためのコンパイル」です。
この章では，Java ソースコードを JVM が実行可能なバイトコードに変換するためのコンパイル方法について説明しています。

さて，前回の環境構築編では JAL（*Java Assembly Language*）を使って JVM の命令を手打ちするための環境を構築しました。
今回はその前提知識を一緒に構築していきましょう。

なお，この記事は 次の記事から始まる命令解説のハンド・ブックとしての役割も果たします。

## Chapter 3. Java Virtual Machine のためのコンパイル

JVM は， Java 言語用に設計されています。
実際に Oracle が提供する JDK には Java 言語のソースコードから JVM 用のバイト・コードに変換するコンパイラや， JVM 自体のランタイムが含まれています。

この章では Java 言語のソースコードを JVM が実行可能なバイトコードに変換するためのコンパイル方法について説明します。
あるコンパイラが JVM をどのように利用するかを学ぶことは，これからコンパイラを自作しようとする人にも，或いは JVM を理解しようとする人にも役立ちます。

なお，「コンパイル（*compile*）」という語は，JVM の命令を例えば特定の CPU アーキテクチャ用の機械語に変換することを指す場合もあります。
（例としては， JIT（*Just-In-Time*）コンパイラや， AOT（*Ahead-Of-Time*）コンパイラなどがあります。）

でもここではあくまでも， Java 言語のソースコードを JVM のバイトコードに変換することを指します。

## 3.1 記述の例（[› 3.1 Format of Examples](https://docs.oracle.com/javase/specs/jvms/se24/html/jvms-3.html#jvms-3.1)）

この章では，Java ソースコードの例と Oracle JDK Release 1.0.2 の `javac` コンパイラに通した結果生成される JVM バイト・コードの例を示します。
JVM バイト・コードは， Oracle が独自に開発している `javap` ユーティリティ・ツールによって生成される**非公式の** JVM アセンブラ言語…を踏襲している，私が作った JAL（*Java Assembly Language*）で表現しています。

JAL は Java 言語のソースコードを JVM バイト・コードに変換するためのアセンブラ言語であり，JVM の命令を人間が読みやすい形式で表現します。
JAL の命令は JVM の命令をすべて網羅しており，都度注釈がない限りは，全く同じ意味を持ちます。

また，JAL 原語で表記する命令と原文の `javap` ユーティリティ・ツールによって生成される命令が著しく異なる場合は，その旨を注釈で示すとともに `javap` による命令も併記します。 

### JAL 言語で示される例の形式

以下の形式は，アセンブリ言語を読んだことがある方にはおなじみの形式です。
各命令は次のような形式で表現されます。

```java
<命令> <オペランド1> [<オペランド2>, <オペランド3>, ...] [<コメント>]
```
（原文では `<命令>` の前に，その命令のバイト・コード・オフセット（*bytecode offset*）が付与されますが，特に使用する箇所がないので，都度注釈がない限りは省略します。）

ここで `<命令>`は JVM の命令のニーモニック（人が読みやすいように表現された命令の名前）を表します。
さらに `<オペランド 1>` や `<オペランド 2>` は，その命令に必要な引数（オペランド, operand）を表します。
末尾に `[<コメント>]` がある場合は，その命令に対する短い説明や注釈を表します。

#### 例

```java
bipush 100    // int 型の定数 100 をスタックにプッシュする命令
```

### JAL と javap の違い

JAL は JVM の命令を人間が読みやすい形式で表現するためのアセンブラ言語です。
一方，`javap` は JVM バイト・コードを表示するためのツールであり，JVM の命令をそのまま表示します。
両者は一長一短で，例えば前者は JVM の命令を人間が読みやすい形式で表現するため，命令のニーモニックが長くなることがあります。
一方後者は JVM の命令をそのまま表示するために正確性に非常に優れていますが，明らかに人間が読みやすい形式ではありません。

したがってこの記事では JAL を使用して JVM の命令を表現するとともに，以下に主たる違いを示します。

#### 1. バイト・コード・オフセットの仕様

バイト・コード・オフセットとは，その命令が JVM バイト・コードのどの位置にあるかを示す値です。
JAL ではバイト・コード・オフセットを省略しますが，`javap` では各命令の前にバイト・コード・オフセットが付与されます：
```java
8 bipush 100    // int 型の定数 100 をスタックにプッシュする命令
```
上記の例では `bipush` 命令の前に `8` というバイト・コード・オフセットが付与されています。
これは `bipush` 命令が JVM バイト・コードの 8 バイト目にあることを示しています。

バイト・コード・オフセットは，しばし `goto` や `if` 命令などの**ジャンプ先として使用**されます。
```java
3: bipush 100
5: if_cmpge 21
// ...
21: return
```

この例では，`if_cmpge` 命令のオペランドとして `21` が指定されています。
これは `if_cmpge` 命令が `return` 命令の位置にジャンプすることを示しています。

しかしこの記述法は，人間が読みやすいとは言えない上に，もし命令の順番が変わった場合にジャンプ先の位置を手動で更新する必要があるため，非常に面倒です。
そこで JAL ではこのようなジャンプ先の指定は，命令のオペランドとしてラベルを使用して表現します。
```java
  bipush 100
  if_cmpge returnLabel
returnLabel:
  return
```

この記事でも，ジャンプ先の指定はラベルを使用して表現します。

#### 2. 定数プールの使用

JVM バイト・コードでは，定数プール（*constant pool*）を使用して文字列やクラス名などの定数を管理します。
定数プールは JVM バイト・コードの一部であり，バイト・コードの先頭に定義されます。
`javap` では定数プールの内容を `#N` といった形で，インデックスで参照します。
```java
ldc #10    // constant pool #10 "Hello, World!"
```

これも同様に，定数プールのインデックスがズレてしまったときに大混乱が起こることは間違えないので，JAL では定数プールのインデックスを使用せずに，定数を直接指定します。
```java
ldc "Hello, World!"    // 文字列 "Hello, World!" をスタックにプッシュする命令
```

さらにメソッド呼び出しやフィールド参照などでも，定数プールのインデックスを使用せずに，クラス名やメソッド名を直接指定します。
`javap` では次のように表現されます：
```java
invokevirtual #2    // constant pool #2 java/io/PrintStream.println(Ljava/lang/String;)V
getstatic #1    // constant pool #1 java/lang/System.out:Ljava/io/PrintStream;
```

一方 JAL では次のように表現されます：
```java
// java/io/PrintStream クラスの println メソッドを呼び出す命令
invokevirtual java/io/PrintStream->println(Ljava/lang/String;)V
// java/lang/System クラスの out フィールドを取得する命令
getstatic java/lang/System->out:Ljava/io/PrintStream;
```

同じくこの記事でも，定数プールのインデックスを使用せずに，定数を直接指定します。
（なお原文では定数プールの中身は省略されているので，こちらのほうが情報量が多くなります。）

### これ以降の学び方

これ以降の記事では JAL を用いた例をたくさん示して，より具体的に JVM の命令を理解できるようにしています。
皆々様におかれましては，前回の記事で構築した JAL の環境を使って，JAL の命令を手打ちしてみてください。

前回登場した `Hello, World!` では，`main` 関数を定義して，`System.out.println` メソッドを呼び出していました。
これ以降の記事ではこの `main` 関数内に命令を記述することで，手軽に JVM の命令を実行できます。

もちろん実際に手を動かしてみなくても，JVM の命令を理解することはできますが，やはり手を動かすことでより深く理解できると思います。

### まとめ

いかがでしたか。

今回の記事は比較的短い内容でしたが、JVM の命令を理解するための基礎知識を構築できかたかと思います。
次回からは JVM の命令を一つ一つ詳しく解説していきますから，お楽しみにしていてください。

では，よいバイト・コードライフを！

### 次回リンク

https://zenn.dev/peyang/articles/reading-jvm-chapter-03-2

#### 参考文献＆リンク集

+ Lindholm, T., Yellin, F., Bracha, G., & Smith, W. M. D. (2025). [*The Java® Virtual Machine Specification: Java SE 24 Edition*](https://docs.oracle.com/javase/specs/jvms/se24/html/).
+ Lindholm, T., & Yellin, F. (1999). *The Java™ Virtual Machine Specification* (2nd ed.). Addison-Wesley. ISBN 978-0-201-43294-7
+ Otavio, S. (2024). *Mastering the Java Virtual Machine*. Packet Publishing. ISBN 978-1-835-46796-1
+ Godfrey, N., & Koichi , M. (2010). *デコンパイリング Java ― 逆解析技術とコードの難読化*  ISBN 978-4-87311-449-1
