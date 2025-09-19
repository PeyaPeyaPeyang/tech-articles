---
title: "JVM を読む | JVM の構造その８ - 閑話：JVM が満たすべき要件と原則について"
emoji: "🖥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["java", "jvm", "jal"]
published: true
---

前回の続きです。前回はこちらから。

https://zenn.dev/peyang/articles/reading-jvm-chapter-02-11-5-10

このシリーズは，JVM の仕様書を読み解くためのガイドとして構成しています。
JVM の仕様書は非常に長大で難解な内容が多いため，各セクションごとに要点をまとめていきます。
また，JVM の内部構造や動作原理を知ることで，Java のパフォーマンスやセキュリティ，メモリ管理の仕組みを深く理解する試みです。

シリーズはこちらから。

https://zenn.dev/peyang/articles/reading-jvm-chapter-00

## 第二章 The Structure of the Java Virtual Machine

JVM の仕様書の第２章は「Java Virtual Machine の構造」です。
といいましてもこの章は全７章ある JVM の仕様書の中でも特に長く，また特に複雑な内容ですので，全８回に分けて解説していきます。

さて，直近では 2 回に分けて JVM の仕様書の Chapter 2.11 の内容を扱いました。
- [JVM を読む | JVM の構造その６ - 命令セット概論と型の関係について](https://zenn.dev/peyang/articles/reading-jvm-chapter-02-11-1-4)
- [JVM を読む | JVM の構造その７ - オブジェクトとフローの制御について](https://zenn.dev/peyang/articles/reading-jvm-chapter-02-11-5-10)

Chapter 2.11 は JVM の命令セットに関する内容であり，JVM の動作の基礎を成す重要な部分で，かつ最も量が多いセクションです。
というわけで JVM の峠は前回で超えてしまいました（良かったですね）。
（最大の山場は Chapter 4 なのですが… それはまた後の話です。）

さて，今回は Chapter 2.11 の残りの部分を扱います。

### 2.12 クラス・ライブラリ（[› 2.12 Class Libraries](https://docs.oracle.com/javase/specs/jvms/se24/html/jvms-2.html#jvms-2.12)）

JVM は Java SE プラットフォームのクラス・ライブラリの実装に十分な機能を提供する必要があります。
事実これらのライブラリの一部のクラスには， JVM による特別なサポートなしでは実装できないものもあります。

以下に特別なサポートが必要なクラスの例を挙げます：

+ リフレクションに関するもの
  - `java.lang.reflect.*`
  - `java.lang.Class`
  などなど
+ クラスとインタフェースの読み込み，或いは定義に関するもの
  - `java.lang.ClassLoader`
  などなど
+ クラスとインタフェースのリンク，或いは初期化に関するもの
  - `java.lang.Class`
  - `java.lang.ClassLoader`
  - `java.lang.invoke.*`
+ モジュール・システムに関するもの
  - `java.lang.Module`
  - `java.lang.ModuleLayer`
  - `java.lang.module.*`
+ スレッドに関するもの
  - `java.lang.Thread`
  - `java.lang.ThreadGroup`
  - `java.lang.ThreadLocal`
  - `java.lang.StackWalker`
+ 弱参照など，参照に関するもの
  - `java.lang.ref.*`
  
なお，このリストは網羅的なものではなく，JVM の実装者が特別なサポートを必要とするクラスを特定するための参考として提供されています。

:::message
もし，JVM を自作するような苦行に挑戦する場合には，これらの実装が一番大きなハードルとなります。
特に，リフレクションやクラス・ローダ，スレッドに関する部分は非常に複雑で，JVM の設計と実装にネイティブ・レベルで関与します。
:::

#### 特別なサポートの例

例えばリフレクションにおける `java.lang.reflect.Method` クラスは，メソッドの動的な呼び出しを可能にするために， JVM 内部に実行時に動的に呼び出す機構が必要です。
さらにクラスの読み込みや定義に関する `java.lang.ClassLoader` クラスは，クラスのロードやリンクを行うためにバイト・コードからクラスを作成できる機能が必要です。

或いはスレッドに関するクラスでは，例えば `java.lang.Thread` オブジェクトを実際の OS のスレッドにマッピングする機構が必要です。

などなど，まだまだたくさんのクラスが JVM の特別なサポートを必要とします。
（あくまでも，これは網羅的なリストではないので，JVM を実装する際には Java SE プラットフォームの仕様書を参照してください。）

### 2.13 外部設計と内部実装の分離（[› 2.13 Public Design, Private Implementation](https://docs.oracle.com/javase/specs/jvms/se24/html/jvms-2.html#jvms-2.13)）

これまでの説明では， JVM の外部設計について述べてきました。
JVM は厳格に規定されたクラス・ファイル形式と命令セットを持ち，Java 言語の仕様に従って動作します。
これを各 JVM の実装社が守ることで，特定のオペレーティング・システムやハードウェアに依存せずにプログラムを実行できるのです。

JVM を実装するには，外部設計と内部実装の境界を理解することが非常に重要です。
JVM は**クラス・ファイルを読み込んで，その中の JVM の機械語を正確に実行しなければなりません**。

それにはこのドキュメント（原文）を聖書のように読み，**文字通りに実装することが求められます**。
一方で或いは，**実装者が制約内で変更したり，最適化して実装することも可能**であり，むしろ推奨されています。

クラス・ファイルを読み込んで，機械語を正確に実行できるかぎりは，どのような実装をしても構いません。
例えば…

+ コードの読み込み中，或いは実行中に JVM の機械語を別の仮想マシンの機械語に変換して実行する
+ コードの読み込み中，或いは実行中に JVM の機械語をネイティブコードに変換して実行する（これは一般に JIT(*Just-In-Time*) コンパイルと呼ばれます）

:::message
もちろん例外はどこにだってあります。これも例外ではありません（例外がどこにでもあるように）。
デバッガやプロファイラ，あるいは JIT（*Just-In-Time*）コンパイラなどのツールは，JVM の内部実装に依存することがあります。
:::

#### 原文のユーモア

この節の原文に大変素晴らしい表現がありました！
一部を引用します：

> The implementor may prefer to think of them as a means to securely communicate fragments of programs between hosts each implementing the Java SE Platform, rather than as a blueprint to be followed exactly.
> （[The Java® Virtual Machine Specification Java SE 24 Edition: 2.13 Public Design, Private Implementation](https://docs.oracle.com/javase/specs/jvms/se24/html/jvms-2.html#jvms-2.13)）

これをそのまま日本語に訳すと以下のようになります：
> 実装者は，これを厳密に守るべき設計図（*Blue print*）ではなく，Java SE プラットフォームを実装する各ホスト間でプログラムの断片を安全に通信する手段と考えることを好むかもしれません。

この表現は JVM の外部設計と内部実装の分離を強調しつつ，実装者に自由度を与えることを示唆しています。
さらに JVM しようという一見厳格な文章の中に突如として現れる詩的なメタファーであり，仕様書の読み物として面白い部分でもあります。
単に「厳格に従え」という規則ではなく，「コードの断片を安全に通信する手段」という表現には，分散システムにおけるコード移送や相互運用性の本質を捉えた哲学的な深さがあります。

これを単なる「設計図（*blueprint*）」ではなく「コードの断片を安全に通信する手段（*communicate fragments of programs*）」と転換して捉えることは，
設計の柔軟さと JVM の思想を如実に言語化して表現しています。

この設計思想は，実は UNIX の哲学「小さなプログラムを組み合わせて大きなシステムを作る（*Small is beautiful.*）」という考え方に通じるものがあります。
UNIX の世界では，それぞれのツールは小さく，単一責任を持ち，さらに標準入出力を通して疎結合的に連携します。
その思想は JVM においても，バイト・コードを通してプログラムの断片を安全に通信し，実行するという形で受け継がれています。

さらに UNIX の哲学では「移植性を効率よりも優先する（*Choose portability over efficiency.*）」という考え方があります。
これは JVM の根幹にある「*Write Once, Run Anywhere*」（一度書けばどこでも動く）という特性と一致します。
「ネイティブ・コードで最適化すればさらに早くなる」という批判に対して，JVM は敢えて「バイト・コードという共通形式を定義して，どの環境でも同じ意味で動かす」という選択をしました。
その結果として， Java プログラムは異なるプラットフォーム間での互換性を保ちながら広く普及しました。
さらには *Kotlin* や *Scala* などの他の言語も JVM 上で動作するようになり，Java エコシステム全体が形成されました。

UNIX もまた移植性を重視し，例えばアセンブラで高速なコードを書くことよりも，C 言語で抽象的に書いたたコードを様々なプラットフォームで動作させることを優先しました。

このように，JVM や UNIX の設計思想は単なる技術的な要件を超えて，プログラミングの哲学や文化にまで影響を与えているのです。

### まとめ

いかがでしたか？
JVM の仕様書の第２章の残りの部分を解説しました。
JVM のクラス・ライブラリの実装に必要な特別なサポートや，公開仕様と実装の自由度のバランスについて説明しました。

次回からは第三章の内容に入ります。
第三章では，実践的なバイト・コードの書き方について解説していきます（実際に手を動かしてみてもいいかも分かりません）。

では，よいバイト・コードライフを！

#### 次回リンク

https://zenn.dev/peyang/articles/reading-jvm-chapter-03

#### 参考文献＆リンク集

+ Lindholm, T., Yellin, F., Bracha, G., & Smith, W. M. D. (2025). [*The Java® Virtual Machine Specification: Java SE 24 Edition*](https://docs.oracle.com/javase/specs/jvms/se24/html/).
+ Lindholm, T., & Yellin, F. (1999). *The Java™ Virtual Machine Specification* (2nd ed.). Addison-Wesley. ISBN 978-0-201-43294-7
+ Otavio, S. (2024). *Mastering the Java Virtual Machine*.  Packet Publishing. ISBN 978-1-835-46796-1
+ Mike, G., & Katsura, Y. (2001). *UNIX という考え方: その設計思想と哲学* オーム社. ISBN 978-4-274-06406-7
