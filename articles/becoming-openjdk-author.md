---
title: "OpenJDK の Author になったのでいろいろ解説してみる"
emoji: "🖊️"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["java", "jvm", "openjdk"]
published: true
---

## はじめに

どうも，こんにちは。ぺやんぐ（Peyang）です。
私は普段 Java や JVM 関連の技術が大好きで，日々 JVM の仕組みや Java 言語で遊んでいます。 前回の更新から少し時間が空いてしまいましたが，これは Java で遊んでいたからです。
前回の記事はこちら：

https://zenn.dev/peyang/articles/making-jvm-knowledges

ところで，最近 OpenJDK の JDK プロジェクトにて Author になりましたので，その経験をもとに OpenJDK への貢献方法や，Java エコシステムの面白い話などを解説してみたいと思います。

![OpenJDK ｱｶｳﾝﾄ](/images/becoming-openjdk-author/openjdk-account.png)
（身バレ防止のためにのり弁にしています）

:::message
Java と Java SE について，これらは区別されるべきですが，この記事では便宜上「Java」という言葉で後者を指すことにします。
:::

## Java が面白い話

Java エコ・システムというのは大変面白いもので，そもそも Java の源流は Sun Microsystems 社にありますが，現在は Oracle 社が主導しています。
Java の主要な実装は [OpenJDK](https://openjdk.org/) というオープンソース・プロジェクトで開発されており，これは JDK の公式参照実装として，言語仕様の進化や新機能の導入において中心的な役割を果たしています。
この OpenJDK のコミュニティ主導の開発体制は，Java の発展やあらゆるプラットフォームへの適応を支える重要な基盤となっています。

### Java の貢献者たち

OpenJDK は開かれたコミュニティであり，Oracle 社をはじめとする多くの企業や個人が協力して行っています。
以下の図は，JDK 11 から JDK 25 までに修正された Issue 数を組織別に示したものです。

![OpenJDK の主要コントリビューター](/images/becoming-openjdk-author/issues-fixed-per-organisation.png)
引用： Issues fixed in JDK 11-JDK 25 per organization, Sharat Chander, 2025-09, [The Arrival of Java 25](https://blogs.oracle.com/java/the-arrival-of-java-25)

この画像から分かるように，Oracle 社（画像内赤色）が最も多くの貢献をしていますが，Red Hat 社や Microsoft 社，Amazon 社などの名だたる企業も多くの貢献をしています。
ここで注目すべきは，画像右上のオレンジ色の部分で，これは個人による貢献を示しています。
OpenJDK への貢献は，企業に所属する開発者だけでなく，個人の開発者も積極的に参加していることが分かります。

なるほど個人でも貢献ができることは分かりましたが，具体的な方法やノウハウについてはあまり知られていません。
そこで，この記事では OpenJDK への貢献方法と Author になるまでの流れを解説します。

## OpenJDK コミュニティ概要

OpenJDK に貢献をするには，まずコミュニティの概要を理解する必要があります。

OpenJDK は，いくつかの主要なプロジェクトに分かれています。
主なプロジェクトは以下の通りです：

- [**JDK**](https://openjdk.org/projects/jdk/): JDK 本体の開発を行うプロジェクトです。
- [**Loom**](https://openjdk.org/projects/loom/): 軽量スレッド（仮想スレッド）を導入するプロジェクトです。
- [**Panama**](https://openjdk.org/projects/panama/): Java とネイティブコード（C/C++）との相互運用性を向上させるプロジェクトです。
- [**Amber**](https://openjdk.org/projects/amber/): JEP(Java Enhancement Proposals) を通じて，Java 言語の新機能を模索するプロジェクトです。

さらに，[**Duke**](https://openjdk.org/projects/duke/) という Java のマスコットキャラクターを管理することを目的としたプロジェクトもあります。

### OpenJDK のロール

OpenJDK コミュニティには，各プロジェクト内で有効ないくつかのロール（役割）があります。

1. **Contributor**: コードやドキュメントの変更を提案する人です。後述する OCA（Oracle Contributor Agreement）に署名した人や組織に所属した人全員を指します。
2. **Author**: Contributor の中で，特定のプロジェクトに対して 2 つ以上のパッチが承認＆取り込まれた人を指します。OpenJDK への貢献を志す者が最初に目指すロールです。
3. **Committer**: Author の中で，さらに信頼され，プロジェクトのリポジトリに直接コミットできる権限を持つ人を指します。8 つ以上のパッチが承認＆取り込まれると申請できるようになります。
4. **Reviewer**: Committer の中で，さらに高い信頼を得た人を指します。提案された変更をレビューし，承認または拒否する権限を持ちます。申請には 32 個の高品質なパッチが必要です。

Author になることは（他のロールに比べると）比較的簡単です。２つ以上のパッチが承認されたあとに，そのプロジェクトのリーダーにメールで申請すると，機械的に承認されます。
一方で，Committer や Reviewer になるには，パッチの数だけではなくその品質や適正性も評価されるため，より高いハードルがあります。しかも，申請には既存の Committer や Reviewer からの推薦が必要となり，さらにメーリング・リストでの投票も行われます。

## OpenJDK への貢献方法

さて，本題の OpenJDK への貢献方法について解説します。なお，ここでは様々ある貢献方法の中でも，コードの変更を提案する方法に焦点を当てます。

### 1. OCA（Oracle Contributor Agreement）に署名する

何事よりも最初にするべきは，[OCA（Oracle Contributor Agreement）](https://oca.opensource.oracle.com/)への署名です。
OCA は，Oracle 社が OpenJDK への貢献を受け入れるための法的な同意書で，これに署名することで，以下のことに同意したことになります：
- 貢献したパッチ（コードやドキュメント）が，OpenJDK のライセンス（GPLv2 + Classpath Exception）に基づいて公開されること。
- 貢献したパッチが，Oracle 社および OpenJDK コミュニティによって使用・改変されること。
- 貢献したパッチを，Oracle 社が商用製品に組み込むこと。
- 貢献したパッチに関する知的財産権を，Oracle 社に譲渡すること。
（など）

[OCA 署名ページ](https://oca.opensource.oracle.com/)にすると，次の２つの選択肢が提示されます：
![OCA の署名ページ](/images/becoming-openjdk-author/oca.png)
- **Sign a Company Agreement**: 企業に所属しており，かつ所属企業が知的財産権を主張する場合に選択します。基本的には，企業が OCA に署名し，その後に個人が企業の一員として署名する形になります。
- **Sign a Individual Agreement**: 個人として署名する場合に選択します。氏名やメールアドレス，住所などの個人情報を入力する必要があります。

学生や個人開発者の場合は，通常は「*Sign a Individual Agreement*」を選択します。

OCA に署名すると，Oracle の担当者による確認が行われ，数日以内にステータスが「**Approved**」に更新されます。
![OCA 承認画面](/images/becoming-openjdk-author/oca-signed.png)

### 2. OpenJDK のメーリング・リストに参加する

OCA の承認を待つ間に，OpenJDK のメーリング・リストに参加します。
OpenJDK の開発や議論は，主にメーリング・リストを通じて行われているため，ここに参加することは非常に重要です。
各プロジェクトごとに専用のメーリング・リストが存在するため，自分が貢献したいプロジェクトのリストに参加します。

メーリング・リストの一覧は[こちら](https://mail.openjdk.org/mailman/listinfo)から。

JDK に貢献したい場合は，[announce](https://mail.openjdk.org/mailman/listinfo/announce)，[jdk-dev](https://mail.openjdk.org/mailman/listinfo/jdk-dev)，[core-libs-dev](https://mail.openjdk.org/mailman/listinfo/core-libs-dev) などに参加しておくと良いでしょう。 
表示されるフォームにメールアドレスを入力し，送信すると確認メールが届くので，そのメール内のリンクをクリックして参加を完了させます。

参加に完了すると１日に１００通程度のメールが届くようになります。なお，参加者に向けてメール送信したい場合には，各リストのメールアドレスに向けてメールを送信すれば大丈夫です。
返信をする場合には *Reply-All* を使って，リスト全体にも返信が届くようにすると良いでしょう（これにより，議論の履歴がメーリング・リストに残ります）。

:::message
メーリング・リストでは**本名を使うことが強く推奨**されています。匿名性の高いハンドルネームを使うと，信頼性が低く見られ，貢献が受け入れられにくくなる可能性があります。
:::

### 3. 環境を構築する

いよいよ OpenJDK のソースコードを取得し，開発環境を構築します。
私が普段使っているのは Windows 11 環境なので，ここでは Windows 環境での手順を解説します。 macOS や Linux 環境の場合でも，似たような手順で進められます。

#### MSYS2 のセットアップ

最初にするべきは，[MSYS2](https://www.msys2.org/)のインストールです。 MSYS2 は Windows 上で Unix ライクな環境を提供するツールで，OpenJDK のビルドに必要なツールチェインやライブラリが含まれています。公式サイトの *Installation* セクションに従ってインストールを行ってください。

必要なライブラリやパッケージもここでインストールします。 MSYS2 のターミナルを開き，以下のコマンドを実行してください：
```bash
pacman -S autoconf make mingw-w64-x86_64-toolchain tar unzip zip
```

#### JTReg のセットアップ

後々になって OpenJDK をテストするために，[JTReg](https://openjdk.org/jtreg/)もインストールしておきます。
JTReg は，OpenJDK のテストフレームワークで，Java テストケースの実行に使用されます。
バイナリの提供はないため，ソースコードからビルドする必要があります。

```bash
$ git clone https://github.com/openjdk/jtreg
$ cd jtreg
$ sh make/build.sh
# ↑ にコケた場合，JDK がインストールされているか確認し，明示的に指定する
$ sh make/build-all.sh --with-jdk=/c/path/to/your/jdk
```

ビルドに成功すると，`lib/jtreg.jar` が生成されます。後で使用するので，フル・パスを控えておいてください。
さらに，`bin/` ディレクトリにパスを通しておくと，後々便利です（`jtreg` コマンドが使えるようになります）。

#### OpenJDK プロジェクトのセットアップ

OpenJDK のソースコードは，[GitHub: openjdk/jdk](https://github.com/openjdk/jdk) リポジトリでホストされています。

まずはリポジトリをフォークし，ローカルにクローンします。さらに，環境を構築します。
```bash
$ git clone https://github.com/<YOUR_GITHUB_USERNAME>/jdk.git
$ cd jdk
$ bash configure --with-jtreg=<PATH_TO_YOUR_JTREG>/lib/jtreg.jar --boot-jdk=<PATH_TO_YOUR_BOOT_JDK>
```

なお，OpenJDK のビルドには既存の JDK が必要（[参考](https://ja.wikipedia.org/wiki/%E9%B6%8F%E3%81%8C%E5%85%88%E3%81%8B%E3%80%81%E5%8D%B5%E3%81%8C%E5%85%88%E3%81%8B))ですから，`--boot-jdk` オプションでそのパスを指定します。

#### 任意：IntelliJ IDEA と連携する

[IntelliJ IDEA](https://www.jetbrains.com/ja-jp/idea/) を使用して OpenJDK のコードを編集しようとすると，大量の謎のエラーと不整合に見舞われます。
IntelliJ IDEA は Java を用いて何かを開発することを期待しているわけで，Java 自体を開発することは想定していないためです。

JDK リポジトリには，IntelliJ IDEA 用のプロジェクト・ファイルを生成するスクリプトが含まれています。これを実行すると，IntelliJ IDEA で JDK のコードを快適に編集できるようになります。
```bash
$ ./bin/idea.sh
```

### 4. バグや改善点を見つける

OpenJDK にパッチを提出するためには，まずバグや改善点を見つける必要があります。
具体的には[*JDK Bug System*](https://bugs.openjdk.org/)にアクセスして，取り組めそうな Issue を探します。
Issue は，バグ修正や新機能の提案，ドキュメントの改善など，様々な種類があります。 自分の興味やスキルに合った Issue を選びましょう。

すべての**パッチは JBS に登録された Issue に紐づけられる必要がある**ため，まずはここで Issue を見つけることが重要です。

貢献可能な Issue は，以下のような条件でフィルタリングすると見つけられます：
- プロジェクトが「*JDK*」であること： JDK プロジェクトは最も活発で，貢献の機会が多いです（他のプロジェクトは，JDK 自体とはあまり関係ありません）。
- ステータスが「*Open*」であること： 修正や改善がまだ行われていない Issue を対象とします。
- 割り当てが「*Unassigned*」であること： 他の開発者が割り当てられている Issue には手を出すべきではありません。長い間更新がない場合には，その限りではありませんが，まずはメーリング・リストで確認するのが良いでしょう。

さらに，貢献初心者のために残してくれている Issue もあり，これらには「*starter*」ラベルが付与されています。
「*cleanup*」ラベルの付与された Issue も，コードの整理やリファクタリングを目的としており，初心者に適しています。

C++ に慣れておらず，Java コードのみを修正したい場合には，「*core-libs*」コンポーネントに属する Issue を探すと良いでしょう。
これらは，Java 標準ライブラリのコードに関連する Issue です。

### 5. パッチを作成する

貢献できそうな Issue を見つけたら，次にパッチを作成します。 Issue の説明をよく読み，問題の原因や改善点を理解します。

このあたりは一般的な OSS への貢献と同じです。コードを修正し，必要に応じてテストケースも追加します。
私が貢献を経てきた中で感じたのは，OpenJDK のコードベースは非常に大きく複雑であるため，最初は戸惑うかもしれません。
しかし，ドキュメントや既存のコードを参考にしながら，徐々に理解を深めていくことが重要です。

なお，変更は最小限に抑えるべきです。大規模な変更はレビューが難しくなり，承認されにくくなります。

#### 落とし穴はコミットにあり！

コミットの書き方には特に注意が必要です。
まず，名前とメール・アドレスは メーリング・リストで使っているものと一致させると良いでしょう（*SHOULD*）。
```sh
$ git config user.name "John Doe"
$ git config user.email "john.doe@example.com"
```
このようにすると，OpenJDK リポジトリでのみ設定が上書きされます。

また，コミット・メッセージには Issue 番号を含める必要があります（*MUST*）。
複数のコミットを行う場合には，最初のコミットにのみ Issue 番号を含めれば大丈夫です。
例えば，Issue JDK-123456789 に対する修正を行った場合，コミット・メッセージは以下のようになります： 
```
- 123456789: Fix NPE in path.to.SomeClass when input is null
- Added null check to prevent NPE
- Implemented unit test for null input case
```

コミット・メッセージに Issue 番号を含めないと，jdk リポジトリの Bot がうまく認識できず，リジェクトされてしまいます。

### 6. テストを実行する

変更を加えたら，必ずテストを実行して，変更が正しく動作することを確認します。
OpenJDK には膨大な数のテストケースが含まれており，これらを実行することで，変更が他の部分に悪影響を与えていないことを確認できます。
変更部分のみのテストでも構いませんが，パッチを出す前には `tier1` テスト（基本的なテストセット）を実行することが強く推奨されます。
```sh
$ make test-tier1
```

### 7. パッチを提出する

パッチが完成したら，次に提出します。まず，変更を自分の GitHub リポジトリにプッシュします。
```sh
$ git push origin <YOUR_BRANCH_NAME>
```

次に，OpenJDK リポジトリに対してプル・リクエスト（PR）を作成します。
PR のタイトルには，Issue 番号を含める必要があります（例： `123456789: Fix NPE in path.to.SomeClass`）。
PR の説明には，変更内容の詳細を記述します（それほど文量は必要ありませんが，レビュアーが理解しやすいようにすることが重要です）。

### 8. レビューと統合

PR を作成すると，5 分程で Bot によりコメントが編集されます。
![OpenJDK Bot のコメント](/images/becoming-openjdk-author/bot-editing.png)

このコメントには，PR のステータスや必要なアクションが記載されています。
もし，追加の情報や修正が必要な場合には，Bot が指摘してくれます。適宜対応しましょう。
すべてのチェックにパスし，レビューの準備ができたら `rfr` ラベルが自動的に付与されます。

![RFR の例](/images/becoming-openjdk-author/core-rfr.png)

１～２週間以内に，OpenJDK のコミッターやレビュアーが PR をレビューします。 レビューの結果，修正が必要な場合には，指摘された点を修正して PR を更新します。

何度かのレビューと修正を経て，最終的に PR が承認されると，`ready` ラベルが付与されます。
最終確認を行って問題がないことに責任を持てるようになったら，`/integrate` とコメントしましょう。マージの手続きが開始されます。

#### Author 以下の場合

あなたのロールが Author 以下の場合には，PR のマージには「*Sponsor*」が必要です。 
Sponsor とは，Committer や Reviewer の中で，あなたの PR を支援してくれる人を指します。

通常は，PR をレビューしてくれた人が Sponsor になってくれますが，もし誰も Sponsor になってくれない場合には，メーリング・リストで助けを求めることもできます。
Sponsor が `/sponsor` とコメントすると，マージが開始されます。

#### Committer 以上の場合

Committer 以上のロールを持っている場合には，あなた自身で PR をマージできます。
`/integrate` とコメントすると，すぐにマージが開始されます。

#### ドキっとすること

ここまで来ると，PR が無事マージされることを祈るばかりですが，なぜか PR は *Merged* ではなく *Closed* になります。
安心してください。これは OpenJDK の特殊な運用方法によるもので，PR が正常にマージされたことを意味しています。

Bot は *Merge-and-Squash* に相当する操作を行い，あなたの PR を JDK リポジトリに統合します。
このため， PR 自体は *Closed* ステータスになりますが，実際には変更が JDK リポジトリに取り込まれています。 その証拠に，PR には `integrated` ラベルが付与されます。

![integrated ラベル](/images/becoming-openjdk-author/integrated.png)

### 9. Author になる

２つ以上のパッチがマージされると，あなたは Author になる資格を得ます。

Author になるには，貢献したプロジェクトのリーダーにメールで申請します。メールには，あなたの名前，メールアドレス，GitHub ユーザー名，および貢献したパッチの一覧を含めます。
JDK プロジェクトの場合，リーダーは Mark Reinhold 氏です。メールアドレスは各自で調べてください（メーリング・リストで確認するのが良いでしょう）。

次のようなメールを送信します：
```
Subject: JDK Author request for: <YOUR_NAME>
Dear Mark Reinhold,

I would like to request Author status for the JDK project.
I have contributed the following patches:
- JDK-123456789: Fix NPE in path.to.SomeClass
- JDK-987654321: Improve performance of path.to.AnotherClass

My Email address: <YOUR_EMAIL>

Thank you for your consideration.
Best regards,
<YOUR_NAME>
```

概ね１週間以内に返信があり，OpenJDK コミュニティへの招待リンクが送られてきます。
ページ下部の「*accept*」を押下すると，OpenJDK のユーザ登録が完了します。

なお，[Census](https://openjdk.org/census) という OpenJDK コミュニティのメンバーシップ管理システムにも登録されます。 これには，２週間程度かかる場合があります。

### 10. GitHub と OpenJDK アカウントを連携する

Author になると，OpenJDK の GitHub リポジトリに対して，あなたの GitHub アカウントを紐づける必要があります。
これにより，あなたの貢献が正しく記録されるようになります。

[このリンク](https://bugs.openjdk.java.net/secure/CreateIssue.jspa?pid=11300&issuetype=3)から，次のような Issue を作成します：
```
Project: SKARA
Component: admin
Type: Task

Summary: Add GitHub user <YOUR_GITHUB_USERNAME>
```

Issue が作成されると，OpenJDK の管理者があなたの GitHub アカウントを紐づけてくれます。

## おわりに

いかがでしたか？
以上が，OpenJDK への貢献方法と Author になるまでの流れです。
OpenJDK への貢献は，Java エコシステムに対する貴重な経験となり，技術的なスキルの向上にもつながります。
ぜひ，この記事を参考にして，OpenJDK コミュニティへの参加を検討してみてください。

では，良いバイトコード・ライフを！

## 参考文献＆リンク集

- [The Arrival of Java 25](https://blogs.oracle.com/java/the-arrival-of-java-25), Sharat Chander, 2025-09
- [OpenJDK Developers' Guide](https://openjdk.org/guide/#becoming-an-author), OpenJDK Community
- [Contributing to OpenJDK](https://openjdk.org/contribute/), OpenJDK Community
- [Skara](https://wiki.openjdk.org/display/SKARA#Skara-AssociatingyourGitHubaccountandyourOpenJDKusername), OpenJDK Community
- [Bylaws](https://openjdk.org/bylaws), OpenJDK Community
- [JDK Project](https://openjdk.org/projects/jdk/), OpenJDK Community
- [Projects](https://openjdk.org/projects/jdk/), OpenJDK Community
