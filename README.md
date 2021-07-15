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

参考: [Java 8時代の開発者のためのデバッグ／トラブル解決の基本・応用テクニック～](https://www.atmarkit.co.jp/ait/articles/1410/15/news038_2.html)

## -XX:+HeapDumpOnOutOfMemoryError / -XX:HeapDumpPath

OOM 発生時に HeapDump を出力する。指定しておいたほうが無難。



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





# その他

## -XX:SharedArchiveFile=<.jsa file> -XX:+UseAppCDS

起動速度が早くなる事前に SharedArchiveFile を作成しておき、起動時に作成した SharedArchive を指定するということが必要参考: https://nowokay.hatenablog.com/entry/2019/10/19/002351 / https://yuya-hirooka.hatenablog.com/entry/2020/06/05/202606

## -XX:StartFlightRecording=\ dumponexit=true,\ filename=[file path],\ duration=[time]

それぞれのオプションの意味dumponexit=true → JVM プロセスがシャットダウンした時にファイルにダンプするfilename → 出力するファイル名。ファイル名が指定されていない場合はプロセスの起動したディレクトリに自動的に生成された名前で出力されます。自動的に生成される名前は「プロセスID」「レコーディングID」「タイムスタンプ」を含んでいます。 例: hotspot-pid-47496-id-1-2018_01_25_19_10_41.jfrduration=time → JFRの記録時間を指定する。参考: https://koduki.github.io/docs/book-introduction-of-jfr/site/03/01-recording-jfr.html

## コンテナサポート

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
