---
title: "JVM を読む | JVM の構造その４ - オブジェクトの表現と浮動小数点数の計算について"
emoji: "🖥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["java", "jvm", "jal"]
published: true
---

前回の続きです。前回はこちらから。

https://zenn.dev/peyang/articles/reading-jvm-chapter-02-6

このシリーズは，JVM の仕様書を読み解くためのガイドとして構成しています。
JVM の仕様書は非常に長大で難解な内容が多いため，各セクションごとに要点をまとめていきます。
また，JVM の内部構造や動作原理を知ることで，Java のパフォーマンスやセキュリティ，メモリ管理の仕組みを深く理解する試みです。

シリーズはこちらから。

https://zenn.dev/peyang/articles/reading-jvm-chapter-00

## 第二章 The Structure of the Java Virtual Machine

JVM の仕様書の第２章は「Java Virtual Machine の構造」です。
といいましてもこの章は全７章ある JVM の仕様書の中でも特に長く，また特に複雑な内容ですので，全８回に分けて解説していきます。

ここでは Chapter 2.7, 2.8 の内容を扱います。
比較的軽い内容ですので，２つまとめて解説します。

## 2.7 オブジェクトの表現（[› 2.7 Representation of Objects](https://docs.oracle.com/javase/specs/jvms/se24/html/jvms-2.html#jvms-2.7)）

JVM は，オブジェクトを表現するための特定の内部構造について無頓着です。
すなわち オブジェクトのレイアウトやメモリ配置は JVM 実装に依存しており，この仕様では具体的に定義されていません。

## 2.8 浮動小数点数の計算（[› 2.8 Floating-Point Arithmetic](https://docs.oracle.com/javase/specs/jvms/se24/html/jvms-2.html#jvms-2.8)）

JVM は，IEEE 754 規格に準拠した浮動小数点数の計算をサポートしています。
この規格は，浮動小数点数の表現と演算に関する標準的な仕様であり，JVM はこの規格に従って浮動小数点数を表現して計算を行います。

:::message
Java SE 15 以前では，1985 年に策定された IEEE 754 仕様に基づいていましたが，
それ以降では 2019 年に改訂された IEEE 754 仕様に基づいています。
:::

算術演算と型変換のための命令の多くは，浮動小数点数をサポートしています。
（詳細は省略しますが，[Table 2-8-A. Correspondence with IEEE 754 operations](https://docs.oracle.com/javase/specs/jvms/se24/html/jvms-2.html#jvms-2.3.4:~:text=Table%C2%A02.8%2DA.%C2%A0Correspondence%20with%20IEEE%20754%20operations) に全ての命令が記載されています。）

ただし，JVMの浮動小数点数の計算は，IEEE 754 規格に完全には準拠していません。
以下の点に注意が必要です：
1. 剰余命令は，IEEE 754 の言う剰余演算には対応していません。（IEEE のものの丸め方が異なるため）
2. 否定命令は，IEEE 754 の言う否定演算には**正確には**対応していません（NaN の扱いが異なるため）。
3. 無効な演算やゼロ除算，オーバーフロー，アンダーフロー，さらに IEEE 754 の例外条件にある例外をスローしたりはしません。
4. IEEE 754 の Signaling NaN（SNaN）をサポートしていません。
5. IEEE 754 の丸め方をすべて網羅しておらず，またその指定もできません。
6. Binary32 と Binary64 拡張はサポートしていません。
7. いくつかの IEEE 754 の演算は，JVM の命令セットには存在しません。
  （その代わりに， `Math` クラスのメソッドを使用して計算できます。）

これらの点に注意しながら，JVM の浮動小数点数の計算を理解することが重要です。

### 浮動小数点数の丸め処理（Rounding Policy）

御存知の通り，浮動小数点演算は，実数演算の近似値を計算するための方法であり，精度や丸め誤差に関する問題が発生することがあります。
JVM はこれらの誤差を最小限に抑えるために *Rounding Policy*（丸め規則）を定義しています。

*Rounding Policy* は，浮動小数点数の計算において丸め誤差をどのように処理するかを定義します。
例えば実数の或る小さな範囲の値は，同じ１つの浮動小数点数に丸められます（例えば `1.49999999` ~ `1.50000001` は `1.5` に丸められます）。
また，（当たり前ですが）実数の `1.5` が浮動小数点数の `1.5` として存在できるなら， 実数 `1.5` は浮動小数点数 `1.5` に丸められます。

*Rounding Policy* は，次の２つの方法があります：

1. **Round to Nearest**（最も近い値に丸める）- JVM のデフォルトの丸め方  
  **正確な値に一番近い，表現可能な値に丸める**。
  ２つの値が同じ距離にある場合は LSB（Least Significant Bit）が `0` の方に丸める（偶数丸め）。
  例えば `1.5` は `2.0` に， `2.5` も `2.0` に丸められます。
2. **Round Toward Zero**（ゼロに方向に丸める）- 一部の命令で使用される丸め方  
  **値の絶対値が小さい方に丸める** -> 小数点以下をバッサリ切り捨てる。
  例えば `1.9` は `1` に， `-1.9` は `-1` に丸められます。
  
  適用される命令：整数への変換系（`d2i`，`d2l`，`f2i`，`f2l`）や剰余演算（`frem`，`drem`）のみ。
   
| 規則                  | 対象命令                 | 丸めの方向性        | IEEE 754 名称       |
|---------------------|----------------------|---------------|-------------------|
| *Round to Nearest*  | 通常の計算命令（+，-，×，÷など）   | 近いほう，同距離なら偶数に | `roundTiesToEven` |
| *Round Toward Zero* | 整数変換（f2i等）と剰余（frem等） | 小数点以下を切り捨て    | `roundTowardZero` |

### 昔話

浮動小数点数の計算は，JVM の変遷とともに変化してきました。

#### Java SE 1.0/1.1

+ `float` は IEEE 754 binary32（32ビット浮動小数点数）を使用する
+ `double` は IEEE 754 binary64（64ビット浮動小数点数）を使用する

さらに，丸め処理は必ずそのこの精度まで演算する必要がありました。
これは再現性は高いものの，一部の CPU(例えば Intel の x87) では，浮動小数点数の計算が遅くなる原因となり，取って代わられました。

#### Java SE 1.2 ~ JAVA SE 16

JVM 実装者に *Extended-Exponent Value Set*（拡張指数値セット）を使用することが許可されました。
これは，浮動小数点数の指数部を拡張して，より大きな値を表現できるようにするものです。
これにより，浮動小数点数の計算の精度が向上しました。

:::message
なお，メソッドのアクセス修飾子に *ACC_STRICT*(`strictfp`) が指定されている場合は，この拡張指数値セットは使用されずに，従来の厳格な IEEE 754 規格に従います。
:::

#### Java SE 17 以降

再び初期の頃の仕様に戻り，浮動小数点数の計算は厳格なものになりました。
というのも，現代の CPU は浮動小数点数の計算を高速に行えるため，パフォーマンスを引き合いにして精度を犠牲にする必要がなくなったからです。
そのため，先程述べた *ACC_STRICT*(`strictfp`) フラグを付けずとも厳格な演算が行われるため，
このフラグは（*major version* > 60 では）無視されます。

## まとめ

いかがでしたか？
この記事では，JVM の仕様書の第２章の一部である「オブジェクトの表現」と「浮動小数点数の計算」について解説しました。

オブジェクトの内部構造は JVM 実装に依存しており，どのようにメモリに配置するかについては完全に JVM 実装者の自由です。

浮動小数点数の計算は IEEE 754 規格に**概ね**準拠していますが，完全な準拠ではありません。
特に 剰余・否定・例外処理・SNaN・丸め制御については，いくつかの差異があります。
さらに JVM の歴史に伴って浮動小数点数の計算の仕様は変化してきたのです。

次回は Chapter 2.9, 2.10 の内容（特別なメソッドと例外制御）を扱います。

では，よいバイト・コードライフを！

#### 次回リンク

https://zenn.dev/peyang/articles/reading-jvm-chapter-02-9-10

#### 参考文献＆リンク集

+ Lindholm, T., Yellin, F., Bracha, G., & Smith, W. M. D. (2025). [*The Java® Virtual Machine Specification: Java SE 24 Edition*](https://docs.oracle.com/javase/specs/jvms/se24/html/).
+ Lindholm, T., & Yellin, F. (1999). *The Java™ Virtual Machine Specification* (2nd ed.). Addison-Wesley. ISBN 978-0-201-43294-7
+ Otavio, S. (2024). *Mastering the Java Virtual Machine*.  Packet Publishing. ISBN 978-1-835-46796-1
