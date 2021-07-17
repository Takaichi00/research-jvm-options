# research-jvm-options
JVM オプションを調査・まとめます。バージョンは Java 11 を想定。

# メモリ

## -Xms, -Xmx

JVM にはエルゴノミクスというプロセスがあり、マシンのスペックに応じてヒープサイズが自動決定される。 
デフォルトでは以下の設定となる。

* 初期ヒープ・サイズ: 物理メモリーの 1/64 (最大1GB)
* 最大ヒープ・サイズ: 物理メモリの 1/4 (最大1GB) 

※ OpenJDK 11 + CentOS 7 では 4GB かも (参考: https://qiita.com/hama777/items/3cfe63c050f0d85577a0)

また、-Xms と -Xmx の設定値を同じにしたほうがいい。なぜなら -Xms と -Xmx は同じ値にしたほうが、拡張/シュリンク時のコストが発生しないため。(ヒープサイズ調整のオーバーヘッドがなくなりパフォーマンスが上がる場合もある)

参考1: [【JUGナイトセミナー】検証では成功した Java のパッチが商用でコケた件](https://speakerdeck.com/takaichi00/jugnaitosemina-jian-zheng-dehacheng-gong-sita-java-falsepatutigashang-yong-dekoketajian?slide=10) / [2 エルゴノミクス](https://docs.oracle.com/javase/jp/8/docs/technotes/guides/vm/gctuning/ergonomics.html) 

参考2: https://yoskhdia.hatenablog.com/entry/2017/11/05/224428 / https://www.atmarkit.co.jp/ait/articles/0404/02/news079_2.html

## -XX:MaxNewSize / -XX:MaxNewSize /  -XX:NewRatio

設定しない。以下参考。

* 「New領域のリサイズが制限され，目標停止時間内にGC停止時間を抑えようとするG1GCのメリットを損なうため，G1GCでは指定しないことを推奨します。また，3～5のオプションを指定しなかった場合，New領域の初期サイズはJavaヒープの初期サイズ×0.2のサイズが確保され，Eden領域の初期サイズはSurvivorRatioのデフォルト値である8に従って割り当てられます。」(http://itdoc.hitachi.co.jp/manuals/link/cosmi_v0970/03Y0460D/EY040198.HTM)
* 「G1 GCを評価してチューニングするときは、常に次の推奨事項を考慮してください。 ... 若い世代のサイズ: -Xmnオプションやその他の関連オプション(-XX:NewRatioなど)を使用して、若い世代のサイズを明示的に設定しないでください。若い世代のサイズを固定すると、指定した一時停止時間目標がオーバーライドされます。」 (https://docs.oracle.com/javase/jp/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html)

## -XX:MetaspaceSize /  -XX:MaxMetaspaceSize

ロードされたclassなどの情報が格納される領域。Java 7 では ヒープの Permanent 領域となっていたが、Java 8 では Naitive Memory 扱いとなった。

デフォルトの MetaspaceSize は 12MBから約20MB ([参考](https://docs.oracle.com/javase/jp/11/gctuning/other-considerations.html#GUID-B29C9153-3530-4C15-9154-E74F44E3DAD9))。MaxMetaspaceSize は 約1.6 テラバイト (無尽蔵)。

* 確認コマンド: `$ java -XX:+PrintFlagsFinal -version -server | egrep ``" (Max)?MetaspaceSize"`   ([参考](https://www.atmarkit.co.jp/ait/articles/1410/15/news038_2.html)) 

上限値を指定しないことでなにかの拍子に Metaspace を無尽蔵に食いつぶすということも可能性としては出てきてしまうので上限値は設定したほうが良さそう

* 参考: [Java8のHotSpotVMからPermanent領域が消えた理由とその影響](http://equj65.net/tech/java8hotspot/))  
  * 「-XX:MetaspaceSizeと-XX:MaxMetaspaceSizeの値は同じ値を指定することを推奨します。」(http://itdoc.hitachi.co.jp/manuals/link/cosmi_v0970/03Y0460D/EY040198.HTM) ともあるので、明示的に同じ値を指定したほうがいいか? 

※ 一般的な REST API を提供するアプリケーションでは大体 100 ~ 300 MB を指定することが多かった

## -XX:CompressedClassSpaceSize

圧縮されたクラスを格納するサイズ。ネイティブメモリ扱い。ヒープが 32GB のデフォルトは 1GB。[デフォルトでは大きめに確保される](https://www.atmarkit.co.jp/ait/articles/1410/15/news038_2.html) ようなので、もう少し少なく指定しておくほうがいい (128 ~ 384MB)

「圧縮オブジェクトポインタ機能は，Javaアプリケーションによって作成されるJavaオブジェクトのサイズを圧縮することで，64bit環境で動作するJavaアプリケーションのメモリ使用効率を向上させる機能です。この機能を有効にすることによって，64bit版Application Server上で動作しているJavaアプリケーション実行中での，Javaヒープ領域とExplicitヒープ領域※の使用量を少なくできます。」: http://itdoc.hitachi.co.jp/manuals/link/cosmi_v0970/03Y0460D/OTHER054.HTM

「ただし，圧縮オブジェクトポインタ機能が有効な場合，Metaspace領域内にCompressed Class Spaceという領域が作成されます。この領域に，Javaヒープ内のオブジェクトから参照されるクラス情報が配置されます。そして，それ以外のメソッド情報などがCompressed Class Space以外のMetaspace領域に配置されます。」: http://itdoc.hitachi.co.jp/manuals/link/cosmi_v0970/03Y0460D/EY040144.HTM

参考: [Java 8時代の開発者のためのデバッグ／トラブル解決の基本・応用テクニック～](https://www.atmarkit.co.jp/ait/articles/1410/15/news038_2.html) / [7.18.3　-XX:CompressedClassSpaceSize](http://itdoc.hitachi.co.jp/manuals/link/has_v101000/0342020D/0782.HTM)

## -XX:+HeapDumpOnOutOfMemoryError / -XX:HeapDumpPath

OOM 発生時に HeapDump を出力する。指定しておいたほうが無難。

## ヒープのメモ

* [第 4 章 Java 実行システムのチューニング](https://docs.oracle.com/cd/E19528-01/820-1613/6nd986vco/index.html)

  * J2SE 5.0 では、HotSpot Java 仮想マシン (JVM) の 2 つの実装が提供されています。

    - クライアント VM は、起動時間を短縮し、メモリーの専有領域を縮小するように調整されています。`-client` JVM コマンド行オプションを使用して呼び出します。
    - サーバー VM は、プログラム実行速度が最大になるように設計されています。`-server` JVM コマンド行オプションを使用して呼び出します。

  * 世代別メモリーシステムの効率は、ほとんどのオブジェクトの生存期間が短いという観測に基づいています。

  * ヒープスペースは、古い世代と新しい世代に分けられます。新しい世代は、新しいオブジェクトの領域 (Eden) と、2 つの Survivor 領域で構成されます。JVM では、新しいオブジェクトは Eden 領域に割り当てられ、生存期間が長いオブジェクトは新しい世代から古い世代に移動されます。

  * 若い世代では、Eden と 2 つの下位領域 (Survivor 領域) を用いる高速コピーガベージコレクタが使用され、生存しているオブジェクトが一方の Survivor 領域からもう一方の Survivor 領域にコピーされます。若い世代の領域で複数回のコレクションを経ても生存しているオブジェクトは、**永続化**されます。つまり寿命の長い (tenured) 世代にコピーされます。

  * フル GC 中の中断時間が 4 秒を超えると、HADB でのセッションデータの持続に断続的に失敗する可能性があります。

  * ガベージコレクションのパフォーマンスは、主に**スループット**と**ポーズ**によって測定されます。スループットは、GC 以外のアクティビティーに費やされた合計時間の割合です。ポーズは、GC が原因でアプリケーションが応答していないように見える時間です。

    * そのほかにも、**フットプリント**と**即応性**の 2 つの考慮事項があります。フットプリントは JVM プロセスの作業サイズで、ページとキャッシュ行で測定されます。即応性は、オブジェクトがデッド状態になってからメモリーが利用可能になるまでの時間です。分散システムでは、これは重要な考慮事項です。
    * 各世代のサイズを決定するときは、これら 4 つのメトリック間のトレードオフを考慮します。たとえば、**若い世代を大きくすれば、スループットを最大にできるかもしれませんが、フットプリントと即応性は犠牲にされます。逆に、若い世代を小さくして 増分 GC を使用すれば、ポーズを最小にできるので即応性は増しますが、スループットは低下**します。

  * `-Xms` パラメータと `-Xmx` パラメータはそれぞれ、最小ヒープサイズと最大ヒープサイズを定義します。世代がいっぱいになると GC が行われるので、スループットは使用可能メモリーの量に反比例します。

  * ヒープサイズを固定する場合は、`-Xms` と `-Xmx` に同じ値を設定します。ヒープサイズが拡大または縮小されると、JVM では、事前に定義された `NewRatio` を維持するために、古い世代と新しい世代のサイズが再計算されます。

  * `NewSize` パラメータと `MaxNewSize` パラメータは、新しい世代の最小サイズと最大サイズを制御します。新しい世代のサイズを固定するには、これらのパラメータに同じ値を設定します。若い世代を大きくするほど、マイナーコレクションの発生回数は少なくなります。古い世代に対する若い世代のサイズの比率は、`NewRatio` で制御されます。たとえば、`-XX:NewRatio=3` と設定すると、古い世代と若い世代の比率は 1:3 になり、Eden 領域と Survivor 領域の合計サイズはヒープサイズの 4 分の 1 になります。

    デフォルトでは、Application Server は Java HotSpot Server JVM によって起動されます。Server JVM のデフォルトの `NewRatio` は 2 で、古い世代がヒープの 3 分の 2、新しい世代が 3 分の 1 を占めます。新しい世代を大きくするほど、生存期間の短いオブジェクトを数多く収容できるので、時間のかかるメジャーコレクションの回数が減ります。同時に、古い世代も、生存期間の長いオブジェクトを多数保持するのに十分な大きさがあります。

* [Java 8時代の開発者のためのデバッグ／トラブル解決の基本・応用テクニック～JJUG CCC 2014 Springまとめリポート（後編）](https://www.atmarkit.co.jp/ait/articles/1410/15/news038_2.html)
  * Permanent領域はJVMが確保するメモリのうち、クラス定義などを格納する領域でOutOfMemoryErrorの原因として多くのエンジニアを悩ませてきた。Permanent領域はデフォルトの最大サイズが64MB程度（プラットフォーム、JVMにより異なる）と小さく、アプリケーションの再デプロイ時やクラスがリークしている場合にOutOfMemoryErrorが発生する。
  * OutOfMemoryErrorの対処としておなじみの「-Xms」「-mx」オプションの指定ではPermanent領域の枯渇は解決しないため、対症療法的に定期的にJVMの再起動をして回避してしまうケースも見られる。対してMetaspaceは実質無制限になっているため「ちょっとアプリケーションの規模が大きい」程度ではOutOfMemoryErrorは発生しなくなった。
  * 再デプロイ時の一時的なPermanent領域不足の心配はなくなったが、クラスがリークしていればCompressedClassSpaceの最大サイズに達してやはりOutOfMemoryErrorが発生したり、プロセスサイズの肥大からOSやOutOfMemory Killerにプロセスを消されたりする可能性がある。
  * 最後に末永氏は、Metaspaceのチューニング方法として、デフォルトの「CompressedClassSpaceSize」は1GBと大きく確保されるため絞り気味に、不安を抱えたくなければ「CompressedClassPointers」（ヒープサイズが32GB以下の場合はデフォルトで有効）を明示的に無効にするテクニックを紹介した。
* [SerialGC使用時のJavaVMで使用するメモリ空間の構成とJavaVMオプション](http://itdoc.hitachi.co.jp/manuals/link/cosmi_v0970/03Y0460D/EY040144.HTM)
  * ヒープの内訳が図示されている

* [Java8のHotSpotVMからPermanent領域が消えた理由とその影響](http://equj65.net/tech/java8hotspot/)

  * ava8のHotSpotVMからは従来のPermanent領域が無くなった。

  * **近年はDIコンテナが発達してきており、DI対象となるクラス情報をサーバ起動時に一括してメモリ上に取り込むといった動作を行うコンテナも多い（JavaEEのCDIなど）**。こういった流れの中で、従来のクラス等のメタ情報をPermanent領域に配置するパターンでは、**Permanent領域のOutOfMemoryErrorが発生しやすくなる**といった状況となっていた。

  * Permanent領域がMetaspaceと呼ばれる領域に変更されている。また、これまでのPermanent領域はヒープ領域の一種として扱われていたが、MetaspaceはNativeメモリとして扱われている。

  * 従来のNativeメモリでは[JNI](http://ja.wikipedia.org/wiki/Java_Native_Interface)等を利用した場合のネイティブコードが配置されたり、JVM自体が利用する領域（Cヒープ）があった。

  * **MetaspaceにNativeメモリを利用することで、領域確保の上限を意識する必要がなくなる。**

  * 本番環境では基本的に従来のMaxPermanentSizeと同様に設定すべきだろう。
    特に、**独自のクラスローダを実装しており、且つこれらのクラスローダにメモリリークの疑いがある場合、パラメータ「-XX:MaxMetaspaceSize」を指定するべき**だ。

  * Metaspaceは魔法の領域ではない。従来発生していたメモリリークが解消される訳ではないのだ。クラスローディングのメモリリークが疑われる環境では、従来のPermanent領域の上限値が、ある種の”ストッパー”の役割を果たしてきた。
    Metaspaceではこのストッパーが存在しないため、無尽蔵にリークされた領域が広がっていく恐れがある。こういった環境では上限値（-XX:MaxMetaspaceSize）を明示的に設定すべきだろう。

  * 従来のPermanent領域はクラスやメソッドのメタ情報の他、staticな変数や定数等も領域内に格納していた。ところがMetaspaceではクラス、メソッドの情報のみ保持し、その他の情報はJavaヒープ上の保持に変更された。

  * MetaspaceがNativeメモリ上に存在していたとしてもフルガーベッジ時のコレクション対象となる。

  * ヒープの各領域の説明、GC の説明 (どのようにオブジェクトがメモリを移動するか) などが記載されている

  * なぜオブジェクトはSurvivor領域を行き来するのか？

    * メモリの断片化を避けるためである。
    * GCの役割はオブジェクトによって占有されたメモリ空間を開放することだが、これを1つの領域でのみ実施すると次第に領域が断片化し、メモリ空間が中途半端なスキマだらけになってしまう。これを避けるために、ScavegeGCが発生する都度、Survivor領域の間でオブジェクトを移動させ、オブジェクトを再配置することによって断片化を避けているのだ。

  * しかし、自作クラスローダを経由してクラスをロードし、このクラスローダのインスタンスがヒープから消え、且つロードされたクラスの情報も利用されていない場合に限り、クラス情報をGC対象とすることができる。これをクラスアンロードと呼び、この仕組みを利用して実行中のJVM上のクラス情報を動的に差し替えることが可能だ。

    

# GC

## -Xlog:gc:[option]

GC ログを出力する。GC ログを出力することによるパフォーマンス影響はそれほど無いという文献もあるが要検証。 ([参考](https://software.fujitsu.com/jp/manual/manualfiles/m110012/b1ws0944/02z200/b0944-a-07-03.html))

オプションの参考: https://www.slideshare.net/YujiKubota/unified-jvm-logging

## - XX:+UseG1GC

G1GC を利用するオプション。 java11 のデフォルト GC は G1GC だが、低スペックマシンでは SerialGC が勝手に選択されることもあるみたいなので、そのままのオプションを設定すればよい。 起動プションも java 8 と記述形式は変更なし。 参考: [低スペックマシンでJava 11を動かすと、デフォルトのGCはG1GCじゃなくてSerialGCになる](https://matsumana.info/blog/2018/12/09/java11-g1gc-default/)

## G1GC のメモ

* G1GC のチューニング: [10 ガベージファースト・ガベージ・コレクタのチューニング](https://docs.oracle.com/javase/jp/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html)

  * G1 GCには、満たそうとする一時停止時間目標(ソフト・リアルタイム目標)があります。若いコレクションでは、G1 GCはその若い世代(EdenのサイズとSurvivorのサイズ)を調整して、ソフト・リアルタイム目標を満たします。

  * G1 GCは、リージョンのEdenセットに追加されたリージョンからの割当て要求のほとんどを満たします。

  * 若いガベージ・コレクションでは、G1 GCは前回のガベージ・コレクションからEdenリージョンとSurvivorリージョンの両方を収集します。

  * 重要なデフォルト

    * G1 GCは適応型ガベージ・コレクタであり、デフォルト値を変更せずにそのまま使用して効果的に動作できます

  * **推奨事項**

    * **若い世代のサイズ:** `-Xmn`オプションやその他の関連オプション(`-XX:NewRatio`など)を使用して、若い世代のサイズを明示的に設定しないでください。若い世代のサイズを固定すると、指定した一時停止時間目標がオーバーライドされます。
      * `-Xmn` ... New領域初期サイズ
    * **一時停止時間目標**
      * G1 GCのスループット目標は、アプリケーション時間が90%、ガベージ・コレクション時間が10%です。
      * これをJava HotSpot VMのパラレル・コレクタと比較してみましょう。パラレル・コレクタのスループット目標は、アプリケーション時間が99%、ガベージ・コレクション時間が1%です。
      * したがって、G1 GCのスループットを評価するときは、一時停止時間目標を緩くしてください。あまり積極的な目標を設定すると、ガベージ・コレクションのオーバーヘッドが増加しても構わないという意味に解釈され、スループットに直接影響します。

  * **「to-space overflow」または「to-space exhausted」**メッセージがログに表示されている場合、G1 GCではSurvivorオブジェクトまたは昇格されたオブジェクト(あるいはその両方)用のメモリーが不足しています。Javaヒープはすでに最大値に達しているので拡張できません。

    * ```
      924.897: [GC pause (G1 Evacuation Pause) (mixed) (to-space exhausted), 0.1957310 secs]
      924.897: [GC pause (G1 Evacuation Pause) (mixed) (to-space overflow), 0.1957310 secs]
      ```

* [7.16　G1GCのチューニング](http://itdoc.hitachi.co.jp/manuals/link/cosmi_v0970/03Y0460D/EY040198.HTM) (日立のドキュメンテーション?)

  * 3～5のオプションを指定した場合，New領域のリサイズが制限され，目標停止時間内にGC停止時間を抑えようとするG1GCのメリットを損なうため，G1GCでは指定しないことを推奨します。また，3～5のオプションを指定しなかった場合，New領域の初期サイズはJavaヒープの初期サイズ×0.2のサイズが確保され，Eden領域の初期サイズはSurvivorRatioのデフォルト値である8に従って割り当てられます。

* [JVMアプリケーションを運用する際のメジャーどころチューニングポイントメモ](https://yoskhdia.hatenablog.com/entry/2017/11/05/224428)

* [ガベージコレクタの仕組みを理解する](https://www.atmarkit.co.jp/ait/articles/0404/02/news079_2.html)
  * -Xms、-Xmx（-Xms≦-Xmx）に異なる値を設定すると、ヒープサイズは起動時には-Xmsで指定された大きさですが、状況によってJVMがヒープが足りないと判断した場合は最大-Xmxで指定した大きさまで拡大します。これらの値を同じにするとヒープサイズ調整のオーバーヘッドがなくなりパフォーマンスが上がる場合もあります。
* [Unified JVM Logging](https://www.slideshare.net/YujiKubota/unified-jvm-logging)
  * Java 9 から変更された GC ログの形式について
  * GC ログの設定方法見方についても言及





# App CDS

## -XX:SharedArchiveFile=<.jsa file> -XX:+UseAppCDS

起動速度が早くなる事前に SharedArchiveFile を作成しておき、起動時に作成した SharedArchive を指定するということが必要参考: https://nowokay.hatenablog.com/entry/2019/10/19/002351 / https://yuya-hirooka.hatenablog.com/entry/2020/06/05/202606



# JFR

## -XX:StartFlightRecording=\ dumponexit=true,\ filename=[file path],\ duration=[time]

それぞれのオプションの意味dumponexit=true → JVM プロセスがシャットダウンした時にファイルにダンプするfilename → 出力するファイル名。ファイル名が指定されていない場合はプロセスの起動したディレクトリに自動的に生成された名前で出力されます。自動的に生成される名前は「プロセスID」「レコーディングID」「タイムスタンプ」を含んでいます。 例: hotspot-pid-47496-id-1-2018_01_25_19_10_41.jfrduration=time → JFRの記録時間を指定する。参考: https://koduki.github.io/docs/book-introduction-of-jfr/site/03/01-recording-jfr.html

* [AppCDSを使ってみる](https://yuya-hirooka.hatenablog.com/entry/2020/06/05/202606)

  * AppCDS(Application Class Data Sharing)は[CDS](http://d.hatena.ne.jp/keyword/CDS)(Class Data Sharing)の拡張であり、[Java](http://d.hatena.ne.jp/keyword/Java) 10以降で利用可能です。
  * この機能を使うことで、アプリケーションでロードされるクラスを共有[アーカイブ](http://d.hatena.ne.jp/keyword/%A5%A2%A1%BC%A5%AB%A5%A4%A5%D6)から読み込むようになり、その起動時間を削減することができます。
  * また、複数の[JVM](http://d.hatena.ne.jp/keyword/JVM)プロセス間でこの共有[アーカイブ](http://d.hatena.ne.jp/keyword/%A5%A2%A1%BC%A5%AB%A5%A4%A5%D6)からロードされるデータをシェアすることでメモリフットプリントを削減することが可能となるようです。
  * **[JVM](http://d.hatena.ne.jp/keyword/JVM)を起動する際には、[バイトコード](http://d.hatena.ne.jp/keyword/%A5%D0%A5%A4%A5%C8%A5%B3%A1%BC%A5%C9)のロード、検証、リンク、クラスとインターフェースの初期化などが実行されます**。ここで、コアなクラスやインターフェースに関しては[JVM](http://d.hatena.ne.jp/keyword/JVM)の更新が起きない限り、毎回同じプロセスが実行されることになります。
  * [CDS](http://d.hatena.ne.jp/keyword/CDS)ではそのプロセスの実行結果をファイルに共有[アーカイブ](http://d.hatena.ne.jp/keyword/%A5%A2%A1%BC%A5%AB%A5%A4%A5%D6)としてダンプし、[JVM](http://d.hatena.ne.jp/keyword/JVM)起動時にその共有[アーカイブ](http://d.hatena.ne.jp/keyword/%A5%A2%A1%BC%A5%AB%A5%A4%A5%D6)がメモリに[マッピング](http://d.hatena.ne.jp/keyword/%A5%DE%A5%C3%A5%D4%A5%F3%A5%B0)されることにより起動時間の短縮が可能となります。
  * また、共有[アーカイブ](http://d.hatena.ne.jp/keyword/%A5%A2%A1%BC%A5%AB%A5%A4%A5%D6)の名の通り、この[アーカイブ](http://d.hatena.ne.jp/keyword/%A5%A2%A1%BC%A5%AB%A5%A4%A5%D6)されたデータは複数の[JVM](http://d.hatena.ne.jp/keyword/JVM)のプロセスで共有されるようになるため、メモリフットプリントの削減にもつながります。**[Java](http://d.hatena.ne.jp/keyword/Java) 12以降ではこの[CDS](http://d.hatena.ne.jp/keyword/CDS)の機能はデフォルトで有効化されています**
* [Dynamic CDSよりJava10からある自力ダンプの方が起動が速い](https://nowokay.hatenablog.com/entry/2019/10/19/002351)
  * [Java 13のDynamic CDSで想像以上に起動速度が速くなった - きしだのHatena](https://nowokay.hatenablog.com/entry/2019/10/17/224720)
  * 昨日のエントリでDynamic [CDS](http://d.hatena.ne.jp/keyword/CDS)を使いましたが、自分でダンプしたクラスデータを使うほうが速くなりました。
  * [Java](http://d.hatena.ne.jp/keyword/Java) 11ではそれぞれに`-XX:+UseAppCDS`をつけておく必要があります。





# コンテナサポート

+ [JVMアプリケーションを運用する際のメジャーどころチューニングポイントメモ](https://yoskhdia.hatenablog.com/entry/2017/11/05/224428)
  + JDK 10 からコンテナサポートが強化される
  + `-XX:-UseContainerSupport`がデフォルトで有効となり、[JDK](http://d.hatena.ne.jp/keyword/JDK) 10からはDockerの設定[*6](https://yoskhdia.hatenablog.com/entry/2017/11/05/224428#f-3770e488)から値を取得するようになるようです。

# ヒープや Metaspace を調べる

参考:

- [jstat のリファレンス](https://docs.oracle.com/javase/jp/8/docs/technotes/tools/windows/jstat.html)
-  [Javaヒープ領域の使用量と容量をコマンドラインから取得する](https://nori3tsu.hatenablog.com/entry/2014/01/11/144927)

```
$ sudo -u uic jstat -gc <java pid>
$ sudo -u uic jstat -gccapacity <java pid>
```



| 平時 (性能試験実施していない状態)            |
| :------------------------------------------- |
| 使用している Metaspace: MU                   |
| Metaspace の容量: MC                         |
| 最大 Metaspace: MCMX                         |
| -Xms = NGCMN * OGCMN                         |
| -Xmx = NGCMX * OGCMX                         |
| Javaヒープ領域の容量 NGC + OGC               |
| Javaヒープ領域の使用量 = S0U + S1U + EC + OU |
| CCSMN: 圧縮されたクラス領域の最小容量(KB)。  |
| CCSMX: 圧縮されたクラス領域の最大容量(KB)。  |
| CCSC: 圧縮されたクラス領域の容量(KB)。       |



# 参考文献

* [JVMアプリケーションを運用する際のメジャーどころチューニングポイントメモ](https://yoskhdia.hatenablog.com/entry/2017/11/05/224428)
  * 	JVM の全体的なチューニング方法について記載されている。困ったら参考になりそう。
