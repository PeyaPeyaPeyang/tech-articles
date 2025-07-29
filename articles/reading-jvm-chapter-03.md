---
title: "JVM を読む | コード・コンパイルその１ - 環境構築編"
emoji: "🖥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["java", "jvm", "jal"]
published: true
---

前回の続きです。前回はこちらから。

https://zenn.dev/peyang/articles/reading-jvm-chapter-02-12-13

このシリーズは，JVM の仕様書を読み解くためのガイドとして構成しています。
JVM の仕様書は非常に長大で難解な内容が多いため，各セクションごとに要点をまとめていきます。
また，JVM の内部構造や動作原理を知ることで，Java のパフォーマンスやセキュリティ，メモリ管理の仕組みを深く理解する試みです。

シリーズはこちらから。

https://zenn.dev/peyang/articles/reading-jvm-chapter-00

## 第三章 Compiling for the Java Virtual Machine

JVM の仕様書の第３章は「Java Virtual Machine のためのコンパイル」です。
この章では，Java ソースコードを JVM が実行可能なバイトコードに変換するためのコンパイル方法について説明しています。
今回の記事はその準備編ということで，まずは使用するツール（まだ秘密です！）の環境構築を行います。

### デコンパイリング Java

ここでやっと，（皆々様が覚えているかは分かりませんが）このシリーズの冒頭で述べた書籍である「デコンパイリング Java」の内容が活きてきます。
この書籍は Java のデコンパイル技術を解説するとともに，実際に手を動かしながら JVM クラス・ファイルの作成方法や，デコンパイル方法と難読化の手法を学べます（非常におすすめです）。

しかしながらこの本は 2010 年に出版されたものであり，当時の最新の Java バージョンは SE 1.6 でした。
こういうこともあって，この本の内容は現在の Java バージョン（例えば SE 17 や SE 21）では通用しない部分もあります。
そのため，この章では最新の Java バージョンに対応したコンパイル方法を（この本を参考にしながら）解説したいと思います。

### アセンブラを学ぶ

JVM の命令は実質的にはアセンブリ言語であり，Java ソースコードを JVM バイトコードに変換するためには（もちろんですが） Java 言語の知識と，多少のアセンブリ言語の知識が必要です。
アセンブリ言語は CPU の命令セットアーキテクチャ（ISA）に依存するため，まずは JVM の命令セットアーキテクチャを理解することが重要です。

### Jasmin - 超有名 Java 用アセンブラ言語

現在（狭い業界の）デファクト・スタンダートになっている Java 用アセンブラ言語があります。
*Jasmin* と名付けられたこのアセンブラ言語は, 1996 年に *NYU Research Lab* の *Jonathan Meyer*, *Daniel Reynaud* ら によって開発されました。
Jasmin は Java バイトコードを人間が読みやすい形式で記述できるように設計されており，Java バイトコードを生成するためのツールとして広く使用されています。

しかしながらこの Jasmin は 2010 年に開発が停止されており，2025 年現在の最新の Java バージョン（例えば SE 24）で使うには次の障壁があります。

1. Java の最新の機能に対応していない
   Jasmin は Java SE 1.5 までの機能にしか対応しておらず，Java SE 1.6 以降の新しい機能（例えば `invokedynamic` 命令）には対応していません。
   そのため，最新の Java バージョンでの実行には Jasmin は適していません。
2. *StackMapFrame* の生成がサポートされていない
   [JVM を読む | JVM の構造その１ - 型について]（https://zenn.dev/peyang/articles/reading-jvm-chapter-02-1-4#%E3%81%BA%E3%82%84%E3%82%93%E3%81%90%E6%B3%A8-2)で私が述べた通り，Java SE 1.6 で導入され，Java SE 1.7 で必須となった *StackMapFrame* の生成が Jasmin ではサポートされていません。
   *StackMapFrame* は，JVM がバイトコードを検証するために必要な情報を提供するものであり，もしこれがないと（必ず）*VerifyError* が発生して，JVM はまともに取り合ってくれません。
3. 構文がややわかりにくい
   例えばクラスの宣言は
   ```jasmin
   .class public HelloWorld
   .super java/lang/Object
   .source "HelloWorld.java"
   ; ...
   ```
   といったように，Java のクラス宣言とは大幅に異なる構文を使用します。
   これにより，Java のクラス宣言や構造化プログラミング言語に慣れた開発者にとっては，Jasmin のコードが読みづらく感じることがあります。
   
   また， Jasmin の命令は Java の構文に似ていますが，Java のキーワードや構文とは異なるため，Java 開発者にとっては大きな学習コストがかかります。

これらの理由から，Jasmin はもはや現代の遺物であり，最新の Java バージョンでの開発や実行には適していません。

### JAL - Java Assembly Language について

そこで登場するのが *JAL*(*Java Assembly Language*) です。
JAL は Jasmin の後継として私が設計した Java 用アセンブラ言語であり，以下のような特徴を持っています。
- **Java の構文に近い**  
   JAL は Java の構文に近い形式で記述できるため，Java 開発者にとっては学習コストが低くなっています。
   例えばクラスの宣言は
   ```jal
   class HelloWorld {
     public static main([Ljava/lang/String;)V {
       // ...
     }
   }
   ```
   といったように，Java のクラス宣言とほぼ同じ構文を使用します（メソッドの宣言は少々異なりますが…）。
- **StackMapFrame の自動生成**
   JAL は *StackMapFrame* の自動生成をサポートしており，Jasmin のように，コンパイルした後に ASM などのツールを使って StackMapFrame を生成する必要がありません。
   これにより，JAL で生成されたバイトコードは Java SE 1.6 以降のバージョンでも問題なく実行できます。
- **構文が直感的**
   JAL の構文は直感的であり，Java 開発者にとっては読みやすくなっています。
   
   例えば次のコードを考えます。
   ```jal
   istore 0 [->example]
   iload example
   ```
   ここでは `istore` 命令でローカル変数のスロット番号 0 に `example` という名前を付けて整数値を格納して，`iload` 命令でその値を読み込んでいます。
   このように JAL は，本来スロット番号を指定してローカル変数を操作しなければいけないところを，**変数名を指定して操作できるようにしています**。
- **[IntelliJ IDEA](https://www.jetbrains.com/idea/) との密な連携**
   JAL は IntelliJ IDEA のプラグインを提供しており， IDE 上で JAL コードを編集・実行できます。
   このプラグインは JAL の構文ハイライトやコード補完をカンペキにサポートしており，開発者は JAL コードを簡単に記述できます。
   
   さらに独自のスタック解析技術によって，デバッグ時にはスタックの状態を可視化しながらステップ実行（コードを一行ずつ実行）できます。

こういった特徴により，JAL は現代の Java 開発者にとって使いやすいアセンブラ言語となっています（そうなるように善処しています）。

このシリーズでは， JAL を使用して JVM のバイトコードを生成し，JVM の内部構造や動作原理を学んでいきます。

…
…
…

ええ，皆々様もう薄々感づかれているかも分かりませんが，つまりこのシリーズは，**JAL の宣伝記事**なのでした！
ちなみにこの記事のタグには [jal](https://zenn.dev/topics/jal) がついていますので，興味のある方はぜひご覧ください。
また，ドシドシ JAL の記事を投稿していただけると大変嬉しいです。


## JAL の環境構築

JAL を使用するためには，まずは JAL の環境を構築する必要があります。
JAL は次の形式でディストリビューションされています。

1. [`jalc` コンパイラ](https://github.com/PeyaPeyaPeyang/LangJAL)
   JAL コードを JVM バイトコードにコンパイルするためのコマンドラインツールです。
   JAL のソースコードをコンパイルして生成されます。
2. JAL Gradle プラグイン：[jal-gradle-plugin](https://plugins.gradle.org/plugin/tokyo.peya.langjal)
   JAL コードを Gradle プロジェクトで使用するためのプラグインです。
   JAL のコンパイルや実行を Gradle タスクとして定義できます。
3. JAL IntelliJ IDEA プラグイン：[Javasm](https://plugins.jetbrains.com/plugin/27944-javasm)
   JAL コードを IntelliJ IDEA 上で編集・実行するためのプラグインです。
   JAL の構文ハイライトやコード補完，デバッグ機能を提供します。

さて，本記事では Javasm を用いて解説を進めていきます。

:::message
なぜなら，１つ目の `jalc` コンパイラはインストールを手動で行う必要があり，やや煩雑です。

さらに２つ目の JAL Gradle プラグインは Gradle プロジェクトを作成する必要があるためです。
これは Gradle の前提知識が必要であり，また Gradle プロジェクトを作成するのはやや面倒です。
:::

### 1. IntelliJ IDEA をセットアップする

[これ](#2.-javasm-プラグインをインストールする)はこの手順をスキップするボタンです。
既に IntelliJ IDEA をインストールしている方は，この手順をスキップして次の手順に進められます。

[IntelliJ IDEA](https://www.jetbrains.com/idea/) はチェコの JetBrains 社が開発した Java 開発環境（IDE）です。
IDEA はいわば Java 開発の Swiss Army Knife であり，Java 開発に必要な機能をすべて備えています。
IDEA は次の２つのエディションがあり，どちらも JAL をサポートしています。
- **Community Edition**（無料）
  無料で使用できるオープンソース版です。Java 開発に必要な基本的な機能を備えています。
- **Ultimate Edition**（有料）
    有料版であり，Java 開発に加えて Web 開発やデータベース開発などの機能を備えています。
    無料トライアルもありますので，興味のある方は試してみてください。

ここでは，無料で使用できる *Community Edition* を使用します。
もちろん *Ultimate Edition* 版を既にお持ちの方，または *Ultimate Edition* 版を使用したい方はそちらを使用しても問題ありません。

#### ステップ１. IntelliJ IDEA をダウンロードする

以下のリンクから IntelliJ IDEA をダウンロードしてください。

https://www.jetbrains.com/ja-jp/idea/download

（Community 版は，少し下にスクロールしたところにあります。）
![downloading-intellij-idea](/images/reading-jvm-chapter-03/downloading-intellij-idea.png)

#### ステップ２. IntelliJ IDEA をインストールする

ダウンロードしたインストーラを実行して，IntelliJ IDEA をインストールします。

UAC（ユーザーアカウント制御）が表示されたら「はい」をクリックしてインストールを続行します。

![accepting-uac](/images/reading-jvm-chapter-03/accepting-uac.png)

あとは，画面の指示に従ってインストールを進めてください。

![installing-1](/images/reading-jvm-chapter-03/installing-idea-1.png)
![installing-2](/images/reading-jvm-chapter-03/installing-idea-2.png)
![installing-3](/images/reading-jvm-chapter-03/installing-idea-3.png)
![installing-4](/images/reading-jvm-chapter-03/installing-idea-4.png)

#### ステップ３. IntelliJ IDEA を起動して日本語化する

IntelliJ IDEA を起動すると，次のような画面が表示されます。
ここで，「Customize」ボタンをクリックして，設定をカスタマイズします。

![opening-customize](/images/reading-jvm-chapter-03/opening-customize.png)

その次に，言語設定をするために「English」を押下した後，「Japanese 日本語」を選択します。

![selecting-language](/images/reading-jvm-chapter-03/selecting-language.png)

そうすると「再起動が必要です」という旨のメッセージが表示されますので，「Restart」ボタンをクリックして再起動します。

![language-restart-prompt](/images/reading-jvm-chapter-03/language-restart-prompt.png)

#### ステップ４. フォントを変更する

立ち上がった IDEA は，大変醜いのでフォントを変更します。

画面左下にある歯車アイコンをクリックした後に，「設定」を選択します。
![opening-setings](/images/reading-jvm-chapter-03/opening-settings.png)

さらに「カスタムフォントの使用」から，お好みのフォントを選択したのち，「適用」ボタンをクリックします。

![selecting-font](/images/reading-jvm-chapter-03/selecting-font.png)

### 2. Javasm プラグインをインストールする

Javasm プラグインは IntelliJ IDEA 上で JAL コードを編集・実行するためのプラグインです。
Javasm プラグインは次の手順でインストールできます。

#### ステップ１. プラグインのインストール画面を開く

IntelliJ IDEA のメイン画面で，左下の歯車アイコンをクリックして「プラグイン」を選択します。
さらに検索ボックスに「Javasm」と入力して検索した後，表示された Javasm プラグインの「インストール」ボタンをクリックします。

![installing-plugin](/images/reading-jvm-chapter-03/installing-plugin.png)

もしくは…
[Javasm プラグインのページ](https://plugins.jetbrains.com/plugin/27944-javasm)にアクセスして「Install」ボタンをクリックしてもインストールできます。

https://plugins.jetbrains.com/plugin/27944-javasm

#### ステップ２. プラグインのインストールを確認する

プラグインを初回インストールする際には，サードパーティのプラグインを信頼してインストールするかどうかの確認が表示されます。
ところで「Javasm」は私が開発したプラグインですので，皆々様が私を信頼しているなら「同意する」ボタンをクリックしてください（もちろん，私を信頼していない方は「同意しない」ボタンをクリックしてくださってかまいません。なお，このプラグインはオープンソースであり，ソースコードは [GitHub](https://github.com/PeyaPeyaPeyang/Javasm) で公開されていますから，信頼してもよいのではないでしょうか。ここまで申し上げてもまだ私を信頼していないという方は，まず最初に適当なプラグインをインストールしてみてください。個人的には [Nyan Progress Bar](https://plugins.jetbrains.com/plugin/8575-nyan-progress-bar) がおすすめです。このとき同じプロンプトが表示されると思いますので，その作者を信頼するならば「同意する」ボタンをクリックしてください。その作者が信頼できないなら，どうか信頼できる作者を探して頂き，どうにかして「同意する」ボタンをクリックしてください。その後「Javasm」をインストールしてくだされば，このプロンプトは表示されませんので，私を信頼せずにとも Javasm をインストールできます。)

その後「OK」ボタンをクリックして，プラグインのインストールを完了します。

![plugin-installing-agreement](/images/reading-jvm-chapter-03/plugin-installing-agreement.png)

#### ステップ３. Javasm プラグインを有効化する

プラグインのインストールが完了したら，IntelliJ IDEA を再起動します。
そうすることで自動的に Javasm プラグインが有効化されます。


## JAL で Hello World を書いてみる

さて，JAL の環境が整いましたので，早速 JAL で Hello World を書いてみましょう。

### ステップ１. 新しい Java プロジェクトを作成する

IntelliJ IDEA を起動したら，メイン画面で「新規プロジェクト」を選択します。

![creating-new-project](/images/reading-jvm-chapter-03/creating-new-project.png)

その後，次のようにプロジェクトの設定を行います。

1. 任意の名前を入力します（ここでは「MyJALProject」とします）。
2. （変更したければ）プロジェクトの保存先を指定します。
3. （SDK がない，もしくは変更したい場合は）JDK を選択します。
   ここでは JDK 17 を選択します。
4. 「サンプルコードの追加」のチェックを外します。

![creating-java-project](/images/reading-jvm-chapter-03/creating-java-project.png)

### ステップ２. JAL ファイルを格納するディレクトリを作成する

プロジェクトが作成されたら，プロジェクトのルートディレクトリに `src/main/jal` というディレクトリを作成します。
そのためには，自動で作成されたディレクトリである「src」を右クリックして「新規作成」→「パッケージ」を選択します。

![creating-jal-package](/images/reading-jvm-chapter-03/creating-jal-package.png)

表示されたプロンプトには「`main.jal`」と入力したうえで，リターンキーを押下します。

![creating-jal-package-prompt](/images/reading-jvm-chapter-03/creating-jal-package-prompt.png)

### ステップ３. ソース・ルートの設定を変更する

次に，JAL ファイルを格納するディレクトリをソース・ルートとして設定します。
これにより， IDEA が JAL ファイルのルートがどこであるのかを認識できるようになります。

まずは，既存のマッピングを解除します。
「`src`」を右クリックして「ディレクトリをマーク」→「ソース ルート のマーク解除」を選択します。

![unmarking-as-src-root](/images/reading-jvm-chapter-03/unmarking-as-src-root.png)

そして新たに「`src/main/jal`」をソース・ルートとしてマークします。

「`jal`」を右クリックして「ディレクトリをマーク」→「ソース ルート」を選択します。

![marking-as-src-root](/images/reading-jvm-chapter-03/marking-as-src-root.png)

### ステップ４. JAL ファイルを作成する

さて，これで JAL ファイルを作成する準備が整いました。

では `HelloWorld` という名前の JAL ファイルを作成しましょう。
「`jal`」を右クリックして「新規作成」→「JAL Class」を選択します。
そして表示されたプロンプトに「`HelloWorld`」と入力してリターンキーを押下します。

![creating-new-jal-class](/images/reading-jvm-chapter-03/creating-new-jal-class.png)
![hello-world-prompt](/images/reading-jvm-chapter-03/hello-world-prompt.png)

そうすると，次のような JAL ファイルが作成されます。
このとき，拡張子 `.jal` が自動的に付与されますので，実際は `HelloWorld.jal` というファイル名で JAL ファイルが作成されます。

![created-jal-file](/images/reading-jvm-chapter-03/created-jal-file.png)

この画面において，行番号が２つ表示されていることにお気づきでしょうか。
左側の行番号は通常のファイルの行番号のように見えますが，右側の行番号は何でしょうか。

これは，JVM のバイト・コード・オフセットを示しています。
バイト・コード・オフセットは，JAL ファイルがコンパイルされたときに生成される JVM バイトコードの各命令のオフセットを示しています。
JAL ファイルをコンパイルすると，JVM バイトコードが生成されますが，そのバイトコードの各命令は連続したバイト列として表現されます。
Javasm はこのバイトコードの各命令のオフセットを表示するために，右側の行番号を使用しています。

### ステップ５. 習うより慣れろ

さて，JAL ファイルが作成できましたので，早速 Hello World を書いてみましょう。

以下のコードを `HelloWorld.jal` ファイルに入力してください。

```java
public class HelloWorld {
    public static main([Ljava/lang/String;)V {
    // Print "Hello, World!"
        getstatic java/lang/System->out:Ljava/io/PrintStream;
        ldc "Hello, World!"
        invokevirtual java/io/PrintStream->println(Ljava/lang/String;)V

    // Return from main
        return
    }
}
```

これは JAL で書かれた Hello World プログラムです。
このコードを IDE に入力すると，次のように構文ハイライトが適用されるとともに，緑色の三角ボタンがメソッド名の左側に表示されます。

![hello-world](/images/reading-jvm-chapter-03/hello-world.png)

Javasm は自動でエントリ・ポイントを検出してくれますので，この緑色の三角ボタンをクリックするだけで，JAL コードをコンパイルして実行できます。
試しにこの緑色の三角ボタンをクリックして，その先の「Run 'HelloWorld' のデバッグ」を押下してみてください。

![run-hello-world](/images/reading-jvm-chapter-03/run-hello-world.png)

そうすると，ビルド（ファイルのコンパイルが開始され，自動的にデバッグ画面が表示されます。
その後自動的にコンパイルされたバイト・コードが 実行され，コンソールに「Hello, World!」と表示されます。
![hello-world-result](/images/reading-jvm-chapter-03/hello-world-result.png)

Hello, World!

#### 裏で起きていること。

さて，ここまでで JAL の環境構築と Hello World の実行ができました。
ここで裏で何が起きているのかを少しだけ説明します。

まず，JAL ファイルを作成するとデフォルトで次のようなコードが生成されます。

```java
public class クラス名 (
    major_version=65,
    minor_version=0){
    public <init>()V {
        aload_0
        invokespecial java/lang/Object-><init>()V
        return
    }
}
```

このコードは，JAL におけるクラスの基本的な構造を示しています。
`public class クラス名` はクラスの宣言を示し，`major_version` と `minor_version` はクラスのバージョンを示しています。
`public <init>()V` はコンストラクタの宣言を示し，`invokespecial` 命令を使用してスーパークラスのコンストラクタを呼び出しています。

このとき `major_version` と `minor_version` は省略可能で，省略した場合はコンパイルしている環境の JDK のバージョンが自動的に設定されます。
さらにコンストラクタも省略可能で，省略した場合は自動的に `java/lang/Object` のコンストラクタが呼び出されます。
このように JAL は Java のクラスの基本的な構造を自動的に生成してくれます。

次に， `HelloWorld.jal` ファイルに入力したコードをみてましょう。

```java
public class HelloWorld {
    public static main([Ljava/lang/String;)V {
    // Print "Hello, World!"
        getstatic java/lang/System->out:Ljava/io/PrintStream;
        ldc "Hello, World!"
        invokevirtual java/io/PrintStream->println(Ljava/lang/String;)V

    // Return from main
        return
    }
}
```

このコードは，エントリ・ポイントである `main` メソッドを定義しています。
`public static main([Ljava/lang/String;)V` は `main` メソッドの宣言を示し，`[Ljava/lang/String;` は引数として文字列の配列を受け取ることを示しています。
`V` は戻り値の型を示し，ここでは戻り値がないことを示しています。
このような文字列は，[メソッド・ディスクリプタ](#補足メソッドディスクリプタについて)と呼ばれるもので，JVM がメソッドを識別するために使用されます。

次に `invokevirtual java/io/PrintStream->println(Ljava/lang/String;)V` は `java/io/PrintStream` クラスの `println` メソッドを呼び出す命令です。
この命令は先ほど取得した標準出力ストリームに対して文字列を出力するために使用されます。
引数として `Ljava/lang/String;` を指定していることから，`println` メソッドは文字列を引数として受け取ることがわかります。

最後に `return` 命令は `main` メソッドからの戻りを示します。

このようにして，JAL で書かれたコードは JVM バイトコードに変換されて，実行されるのです。

#### 補足：メソッド・ディスクリプタについて

`([Ljava/lang/String;)V` は，メソッド・ディスクリプタ(*method descriptor*)と呼ばれるものです。
これは名前と合わせてメソッドを一意に識別するためのもので，JVM ではこのようにしてメソッドを識別します。

メソッド・ディスクリプタは，タイプ・ディスクリプタ(*type descriptor*)を使用してメソッドの引数と戻り値の型を表現します。
タイプ・ディスクリプタは次のような形式で表現されます。
- `B` - `byte` 型を表します。
- `C` - `char` 型を表します。
- `D` - `double` 型を表します。
- `F` - `float` 型を表します。
- `I` - `int` 型を表します。
- `J` - `long` 型を表します。
- `S` - `short` 型を表します。
- `Z` - `boolean` 型を表します。
- `V` - 戻り値がないことを表します。
- `L` - `L<クラス名>;` の形式でクラス型を表します。
- `[` - 配列型を表します。例えば `[[I` は `int` 型の二次元配列を表します。

メソッド・ディスクリプタは，このタイプ・ディスクリプタを組み合わせて表現されます。
構文は次のようになります。
```
(<引数の型>[引数2の型...])<戻り値の型>
```
例：
- `([Ljava/lang/String;)V` は，引数として `java/lang/String` 型の配列を受け取り，戻り値がないことを示しています。
- `(I)Ljava/lang/String;` は，引数として `int` 型を受け取り，戻り値として `java/lang/String` 型を返すことを示しています。
- `(I)[Ljava/lang/String;` は，引数として `int` 型を受け取り，戻り値として `java/lang/String` 型の配列を返すことを示しています。
- `(Ljava/lang/String;I)Z` は，引数として `java/lang/String` 型と `int` 型を受け取り，戻り値として `boolean` 型を返すことを示しています。

次に，`getstatic java/lang/System->out:Ljava/io/PrintStream;` は `java/lang/System` クラスの `out` フィールド（型は `java/io/PrintStream`）を取得する命令です。
この命令は標準出力ストリームを取得するために使用されます。

さらに `ldc "Hello, World!"` は文字列リテラル `"Hello, World!"` を定数プールからロードする命令です。
JAL では実際の定数プールの存在を意識する必要はありません。その代わりにオペランドとして擬似的に
`"Hello, World!"` のように文字列リテラルを指定できます。

## まとめ

いかがでしたか！
この記事では，JVM の仕様書の第３章「Compiling for the Java Virtual Machine」の内容を理解するための準備として，JAL の環境構築を行いました。
JAL は Jasmin の後継として設計された Java 用アセンブラ言語であり，Java 開発者にとって使いやすいアセンブラ言語です。

次回の記事では，JAL を使用して JVM バイトコードを生成する方法について解説します。
JAL の基本的な構文や命令についても触れながら，実際に JAL コードを書いていきます。

では，よいバイト・コードライフを！

#### 参考文献＆リンク集

+ Lindholm, T., Yellin, F., Bracha, G., & Smith, W. M. D. (2025). [*The Java® Virtual Machine Specification: Java SE 24 Edition*](https://docs.oracle.com/javase/specs/jvms/se24/html/).
+ Lindholm, T., & Yellin, F. (1999). *The Java™ Virtual Machine Specification* (2nd ed.). Addison-Wesley. ISBN 978-0-201-43294-7
+ Otavio, S. (2024). *Mastering the Java Virtual Machine*. Packet Publishing. ISBN 978-1-835-46796-1
+ Godfrey, N., & Koichi , M. (2010). *デコンパイリング Java ― 逆解析技術とコードの難読化*  ISBN 978-4-87311-449-1
