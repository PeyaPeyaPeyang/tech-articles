---
title: "JVM を読む | JVM の構造その３ - フレームについて"
emoji: "🖥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["java", "jvm", "jal"]
published: true
---

前回の続きです。前回はこちらから。

https://zenn.dev/peyang/articles/reading-jvm-chapter-02-5

このシリーズは，JVM の仕様書を読み解くためのガイドとして構成しています。
JVM の仕様書は非常に長大で難解な内容が多いため，各セクションごとに要点をまとめていきます。
また，JVM の内部構造や動作原理を知ることで，Java のパフォーマンスやセキュリティ，メモリ管理の仕組みを深く理解する試みです。

シリーズはこちらから。

https://zenn.dev/peyang/articles/reading-jvm-chapter-00

## 第二章 The Structure of the Java Virtual Machine

JVM の仕様書の第２章は「Java Virtual Machine の構造」です。
といいましてもこの章は全７章ある JVM の仕様書の中でも特に長く，また特に複雑な内容ですので，全８回に複数に分けて解説していきます。

ここでは Chapter 2.6 の内容（フレーム）を扱います。

## 2.6 フレーム（[› 2.6 Frames](https://docs.oracle.com/javase/specs/jvms/se24/html/jvms-2.html#jvms-2.6)）

フレームとは，データや状態の保存，動的リンク，メソッドの戻り値，および例外の処理を行うためのデータ構造です。

フレームは，JVMがメソッドを呼び出す際に作成され，メソッドの実行中に必要な情報を保持します。
フレームはスタック上に積まれ，メソッドの呼び出しと戻りに伴ってプッシュおよびポップされます。
メソッドの実行が正常に終了しても，（例外が発生して）異常終了しても，例外なくフレームはスタックから取り除かれて破棄されます。

例：
```
; main() 呼ばれた！   
+------------+
|  フレーム1  | ← main()
+============+

-=-=-=-=-=-=-=-=-=-=-

; doStuff() 呼ばれた！
+------------+
|  フレーム2  | ← doStuff()
+------------+
|  フレーム1  | ← main()
+============+

-=-=-=-=-=-=-=-=-=-=-

; doStuff() が終了！main() に戻る！
+------------+
|  フレーム1  | ← main()
+============+
```

### フレームの中身と制約

各フレームは，ローカル変数の配列，オペランド・スタック，現在のメソッドのランタイム定数テーブルへの参照を持ちます。
なお，或る JVM 実装によっては，デバッグ情報など追加の情報を持つ場合があります。  

なおローカル変数配列とオペランド・スタックの大きさは，コンパイル時に計算されてコードに埋め込まれます。 これにより，意図しないメモリのオーバーフローを防ぎます。

各スレッドでは，実行中のメソッドである１つのフレームのみがアクティブであり，名をば「Current Frame」と呼びます。
さらにそのアクティブなメソッドのことを「Current Method」，そのメソッドを持つクラスを「Current Class」と呼びます。
ローカル変数とオペランド・スタックに対する操作は，通常は Current Frame に対して行われます。

フレームは，その Current Method が別のメソッド呼び出したり，またはそのメソッドから抜けると Current ではなくなります。
また，或るメソッドが呼び出されるとき，新しい風レームが作成されて新しいメソッドに制御が移ると，それぞれ Current Frame や Current Method として扱われます。

メソッドから抜けると，（戻り値があるならば）前のフレームに戻り値が渡されて，さらにそのフレームが Current Frame になります。
なお抜ける前のフレームは，その時に破棄されます。

:::message alert
各スレッドの持つフレームは，他のスレッドからは参照できません。
:::

### フレームの状態遷移

```mermaid
sequenceDiagram
    participant Thread
    participant FrameStack as フレームスタック
    participant Frame1 as フレーム1
    participant Frame2 as フレーム2

    Note over Thread, FrameStack: スレッドにはフレームのスタックがある

    Thread->>Frame1: Current Frame がアクティブ
    Frame1->>Frame1: ローカル変数配列，オペランド・スタック，定数プール参照を持つ

    Note over Frame1: Current Method, Current Class もここでマークされる

    Thread->>FrameStack: メソッド呼び出し
    FrameStack->>Frame2: 新しいをフレーム作成 / Current Frame 交代
    Frame2->>Frame2: 新しいローカル変数配列とオペランド・スタック保持

    Frame2->>FrameStack: メソッド終了（戻り値あり）
    FrameStack->>Frame1: 戻り値を渡して，フレーム2を破棄
```

```mermaid
stateDiagram-v2
    [*] --> Idle : スレッド実行開始
    Idle --> FrameActive : メソッド呼び出し，新フレーム生成
    FrameActive --> FrameActive : 各命令の実行
    FrameActive --> Returning : メソッド終了
    Returning --> FrameActive : 戻り値を前のフレームに渡す
    Returning --> [*] : フレーム破棄

```

## 2.6.1 ローカル変数（[› 2.6.1 Local Variables](https://docs.oracle.com/javase/specs/jvms/se24/html/jvms-2.html#jvms-2.6.1)）

各フレームにはローカル変数の配列があり，これにはメソッドの引数やローカル変数を格納します。  

:::message
いつも Java を書いているときには，ローカル変数がどのように管理されているかを
意識することは少ないですが，JVM の仕様書には「ローカル変数は配列で管理される」と明記されています。
:::

ローカル変数の配列は，フレームの作成時にサイズが決定されます（そしてこれはコンパイル時に計算されて，コードに埋め込まれます）。
各配列の要素は， integer 型，float 型，参照型，または returnAddress 型のいずれかの型を持ちます。
もしくは，２つの連続した要素を組み合わせて long 型や double 型の値を表現します。

ローカル変数のアドレスはインデックスで指定され，最初の要素はインデックス 0 から始まります。

### ローカル変数配列における long/double 型の扱い

long 型または double 型の値は，ローカル変数配列では２つの連続した要素を使用して表現されます。
その場合は以下のようにビッグ・エンディアンで格納されます。

```
; [0x01234567_89ABCDEF] をローカル変数配列の slot 0 に格納
+------------+------------+
|   slot 0   |   slot 1   |
+------------+------------+
| 0x89ABCDEF | 0x01234567 |
+------------+------------+
```

:::message alert
このとき，２つ目のインデックス番号でこの long 型や double 型の値を参照することはできません。
この場合では`slot 0` のみが有効なインデックスであり，`slot 1` を用いてこの `long` 型や `double` 型の値を参照することはできません。
:::

なお， JVM はそのローカル変数が格納されている最初のスロットの数が偶数である必要はないとしています。
すなわち `slot 3` と `slot 4` に `double` 型の値を格納することも可能です。


### メソッド引数の受け渡し

JVM はメソッドの引数をローカル変数として扱い，呼び出されたメソッドのフレームに格納してからそのメソッドを実行します。
引数の数はメソッドの定義に基づいて決まり，ローカル変数配列の `0` から始まるインデックスに格納されます。
引数の型に応じて，ローカル変数配列の要素は適切な型で格納されます。

さらに，インスタンス・メソッドの呼び出しの場合は，ローカル変数 `0` に Java で言うところの `this` 引数（そのインスタンスの参照）が格納されます。
その後パラメータの値は，ローカル変数配列の `1` から始まるインデックスに格納されます。

```
; インスタンスメソッドの引数が 3 つある場合
public void exampleMethod(int a, String b, double c) {
    // ローカル変数配列の状態
    +------------+------------+------------+------------+
    |   slot 0   |   slot 1   |   slot 2   |   slot 3   |
    +============+============+============+============+
    | this (ref) | a (int)    | b (String) | c (double) |
    +------------+------------+------------+------------+
}

; 静的メソッドの引数が 2 つある場合
public static void exampleStaticMethod(int x, int y) {
    // ローカル変数配列の状態
    +------------+------------+
    |   slot 0   |   slot 1   |
    +============+============+
    | x (int)    | y (int)    |
    +------------+------------+
}

```

## 2.6.2 オペランド・スタック（[› 2.6.2 Operand Stack](https://docs.oracle.com/javase/specs/jvms/se24/html/jvms-2.html#jvms-2.6.2)）

各フレームは，オペランド・スタックという独自の LIFO（先入れ先出し）スタックを持ちます。
これはメソッドの実行中に，各命令が使用する一時的なデータを格納するためのものです。  
これもローカル変数配列と同様に，フレームの作成時にサイズが決定されます（そしてこれもコンパイル時に計算されて，コードに埋め込まれます）。
なお，オペランド・スタックは（ローカル変数とは違って）フレームの作成時は空の状態です。

JVM マシンは，ローカル変数やフィールド，定数や値をオペランド・スタックにプッシュする命令を提供します（`iload`, `fload`, `getfield` など）。
また一方でオペランド・スタックから値を取り出して，操作して結果をオペランド・スタックにプッシュする命令もあります（`iadd`, `fsub`, `invokevirtual` など）。
さらに，オペランド・スタックは，メソッドに渡すパラメータを準備したり，メソッドの結果を受け取るために使用されます。

### オペランド・スタックの操作

例えば `iadd` 命令は，２つの `int` 型の値を加算します。
この命令は加算する２つの `int` 値をオペランド・スタックの最上位から取り出します。
取り出す値は，前の何らかの命令（`iconst`, `iload` など）でオペランド・スタックの最上位にプッシュされた値です。
これらの値を加算して，その結果をオペランド・スタックの最上位にプッシュします。

#### `1 + 2 = 3` の例：

```java
int x = 1 + 1;
```

```
iconst_1   // オペランド・スタックに 1 をプッシュ
iconst_2   // オペランド・スタックに 2 をプッシュ
iadd       // オペランド・スタックから 1 と 2 を取り出して加算し，結果 3 をプッシュ
istore_0  // オペランド・スタックから 3 を取り出してローカル変数配列の slot 0 に格納
```

```mermaid
flowchart TB
    subgraph Before iadd
        A1[[...]]
        A2[1: Integer]
        A3[2: Integer]
    end

    subgraph After iadd
        B1[[...]]
        B2[3: Integer]
    end

    A3 --> B2
```

#### `System.out.println("Hello, World!")` の例：

```java
System.out.println("Hello, World!");
```

```java
// オペランド・スタックに PrintStream をプッシュ
getstatic java/lang/System->out:Ljava/io/PrintStream;

// オペランド・スタックに文字列をプッシュ
ldc "Hello, World!"

//  オペランド・スタックから PrintStream と文字列を取り出して println を呼び出す
invokevirtual java/io/PrintStream->println(Ljava/lang/String;)V
```

```mermaid
flowchart TD
    subgraph Step0["開始時のスタック"]
        A0[[...]]
    end

    Step0 --> Step1["getstatic java/lang/System->out"]
    subgraph Step1["getstatic 後のスタック"]
        B0[[...]]
        B1[[PrintStream]]
    end

    Step1 --> Step2["ldc"]
    subgraph Step2["ldc 後のスタック"]
        C0[[...]]
        C1[[PrintStream]]
        C2["Hello, World!"]
    end

    Step2 --> Step3["invokevirtual println"]
    subgraph Step3["invokevirtual 後のスタック"]
        D0[[...]]
    end
```

### オペランド・スタックの制約

オペランド・スタックの各要素には，`long` 型や `double` 型などを含む，JVM の任意の型の値を格納できます。

:::message alert
オペランド・スタックはその型に適した命令で操作する必要があります。
すなわち，オペランド・スタックにある２つの `float` 型の値を，`int` 型の命令を加算する `iadd` 命令で加算しようとすると，`VerifyError` が発生します。
:::

少数の命令は，型に関係なくオペランド・スタックの値を操作できます。（`swap` や `dup` など）
ただし，`long` 型や `double` 型の連続した値を破壊するような命令の使い方をすると `VerifyError` が発生します。

各命令には寄与するオペランド・スタックの要素数が決まっており，命令の実行前にオペランド・スタックに十分な要素があることを確認する必要があります。
例えば dup 命令は，オペランド・スタックの最上位の値を複製しますが，`long` 型や `double` 型の値は２つの連続した要素で表現されるため，そのままでは複製できません。
この場合には代わりに `dup2` 命令を使用して，これら２つの要素を同時に複製する必要があります。

## 2.6.3 動的リンク （[› 2.6.3 Dynamic Linking](https://docs.oracle.com/javase/specs/jvms/se24/html/jvms-2.html#jvms-2.6.3)）

各フレームには，メソッド・コードの実行に必要なランタイム定数プールへの参照が含まれます。
この定数プールは，メソッドの実行中に使用されるクラス，メソッド，フィールド，およびその他の定数を格納します。
動的リンクは，メソッドの実行中に必要なクラスやメソッドを解決するために使用されます。

動的リンクは，定数プールでシンボルとして指定されているクラスやメソッド，フィールドを動的に解決し，その参照を取得します。
未定義のシンボルは必要に応じてロードされ，またフィールドへのアクセスを実効上のフィールド参照（オフセット）に変換されます。

:::message
このように，メッセージと変数を遅延的に解決することで，JVM はクラスのロードやリンクを効率的に行っています。
またメソッドが使用する他のクラスに変更を加えた場合でも，動的リンクにより柔軟に対応できます。
:::

## 2.6.4 メソッド呼び出しの正常終了 （[› 2.6.4 Normal Method Invocation Completion](https://docs.oracle.com/javase/specs/jvms/se24/html/jvms-2.html#jvms-2.6.4)）

メソッド呼び出しが，（JVM から，もしくは `athrow` 命令によって）例外が発生せずに正常に完了した場合は，メソッド呼び出しが正常に完了したとみなされます。

そのときは，呼び出し元のメソッドに戻り値が返されることがあります。
さらに Current Frame は，呼び出し元の状態を復元するために使われ，呼び出し元の `pc` レジスタ（プログラムカウンタ）は，そのメソッド呼び出しを行った命令の次の命令に設定されます。
その後，呼び出し元のメソッドのフレームで，（戻り値があればば設定されて）通常の命令の実行が再開されます。

:::message alert
戻り値は呼び出したメソッドが `return` 軽命令(`ireturn`, `areturn` など) 命令を使用して返す場合に限られます。

なお，各 `return` 系命令は，そのメソッドが定義する戻り値の型と一致する必要があります。
例えば `int` 型の戻り値を持つメソッドで `freturn`(float 型を返す) 命令を使用すると，`VerifyError` が発生します。
:::

```mermaid
sequenceDiagram
    participant Caller as 呼び出し元メソッドフレーム
    participant JVM as JVM内部
    participant Callee as 呼び出し先メソッドフレーム

    Caller->>JVM: メソッド呼び出し要求
    JVM->>Callee: 新しいフレーム作成 & 実行開始
    Callee->>Callee: メソッド命令の実行
    Callee-->>JVM: 戻り値（あれば）返却
    JVM->>Caller: 戻り値を呼び出し元フレームのスタックに
    JVM->>Caller: pcレジスタを呼び出し命令の次の命令へ
    JVM->>Caller: 続けて呼び出し元メソッドの命令実行再開
```

## 2.6.5 メソッド呼び出しの異常終了 （[› 2.6.5 Abnormal Method Invocation Completion](https://docs.oracle.com/javase/specs/jvms/se24/html/jvms-2.html#jvms-2.6.5)）

メソッド呼び出しが異常終了した場合は，呼び出し元のフレームに戻ることなく例外処理が行われます。
これは JVM がエラーを投げたり，呼び出し先のメソッドが `athrow` 命令を使用して例外を投げた場合に発生します。

その例外が Current Method 内でキャッチできない場合は，メソッド呼び出しを即座に終了します。
その間は，呼び出し元に値を返すことなく，Current Frame を破棄して呼び出し元のフレームに制御を戻します。

### ぺやんぐ脚注

その後，呼び出し元のフレームは，例外を処理するための適切なハンドラを探します。
もし呼び出し元のフレームでも例外がキャッチできない場合は，さらに上位のフレームに制御が移ります。
最終的に，例外がキャッチされるかスレッドが終了するまでこのプロセスは続きます。

```mermaid
sequenceDiagram
    participant Caller as 呼び出し元フレーム
    participant JVM as JVM内部
    participant Callee as 呼び出し先フレーム

    Caller->>JVM: メソッド呼び出し要求
    JVM->>Callee: 新しいフレーム作成 & 実行開始
    Callee->>Callee: メソッド処理実行
    Callee-->>JVM: 例外発生（athrowなど）
    JVM->>Callee: Current Frame破棄
    JVM->>Caller: 呼び出し元フレームに制御戻す（戻り値なし）
    Caller->>JVM: 例外ハンドラ探索開始
    alt ハンドラあり
        JVM->>Caller: 例外処理へ遷移（ハンドラにジャンプ）
        Caller->>Caller: 例外処理実行
    else ハンドラなし
        JVM->>Caller: Current Frame破棄
        JVM->>上位フレーム: 制御移動（例外伝播）
        note right of JVM: これ繰り返し続く<br>最終的にキャッチされるかスレッド終了
    end
```

（脚注終わり）

## まとめ

いかがでしたか？
この記事では，JVM のフレームについて解説しました。
フレームはメソッドの実行に必要な情報を保持し，ローカル変数やオペランド・スタックを管理します。
また，フレームはメソッドの呼び出しと戻りを制御し，動的リンクを通じてクラスやメソッドの参照を解決します。

フレームの理解は，JVM の動作やメソッドの実行の仕組みを深く理解するために重要です。
次回は Chapter 2.7, 2.8 の内容を扱います（比較的軽い内容です）。
では，よいバイト・コードライフを！

#### 次回リンク

https://zenn.dev/peyang/articles/reading-jvm-chapter-02-7-8

#### 参考文献＆リンク集

+ Lindholm, T., Yellin, F., Bracha, G., & Smith, W. M. D. (2025). [*The Java® Virtual Machine Specification: Java SE 24 Edition*](https://docs.oracle.com/javase/specs/jvms/se24/html/).
+ Lindholm, T., & Yellin, F. (1999). *The Java™ Virtual Machine Specification* (2nd ed.). Addison-Wesley. ISBN 978-0-201-43294-7
+ Otavio, S. (2024). *Mastering the Java Virtual Machine*.  Packet Publishing. ISBN 978-1-835-46796-1
