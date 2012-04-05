# はじめに

> 「そろそろ頃合い」セイウチは言った
> 「あれやこれやの積もる話」
>
> ルイス・キャロル
> 「不思議の国のアリス」

今回は盛りだくさんです！ まずは身近な例を使って，OpenFlow の動作モデルを説明します。これが理解できれば，OpenFlow の基本概念はバッチリです。次に，「トラフィック集計付きスイッチ」 を実現するコントローラを実際に作ります。これは OpenFlow の重要な処理をすべて含んでいるので，応用するだけでさまざまなタイプのコントローラが作れるようになります。最後に，作成したコントローラを Trema の仮想ネットワーク上で実行します。すばらしいことに，Trema を使えば開発から動作テストまでを開発マシン 1 台だけで完結できます！

では前置きはこのぐらいにして，まずは OpenFlow でスイッチを制御するしくみを理解しましょう。


# OpenFlow の動作モデル

OpenFlow の動作を現実世界にたとえると，製品の電話サポートサービスに似ています。


## 電話サポートの業務手順

友太郎 （ゆうたろう） 君は，エアコンが故障したので修理に出そうと考えました （図1）。電話サポートに問い合わせると，サポート係の葵 （あおい） さんはエアコンの症状を聞き，手元のマニュアルに対処方法が載っている場合にはこれをすぐに教えてくれます。問題は，マニュアルに対処法が載っていない場合です。このようなときは少し時間がかかりますが，上司の宮坂主任にどうしたらよいか聞きます。そして，宮坂主任からの回答が得られたら，葵さんは友太郎君に折り返し電話をします。また，次からの同じ問い合わせにはすばやく答えられるようにするため，葵さんは教わった対処法を手元のマニュアルに追加しておきます。

簡単ですね？ 信じられないかもしれませんが，あなたはすでに OpenFlow の 95% を理解したも同然なのです。

![電話サポートの業務手順](https://github.com/trema/Programming-Trema/raw/master/images/2_001.png)

図1　電話サポートの業務手順

## OpenFlowに置き換えると……

OpenFlow では，お客さんがパケットを発生させるホスト，電話サポート係がスイッチ，上司がコントローラ，マニュアルがスイッチのフローテーブル （後述） に対応します （図2）。

![OpenFlowの動作モデル](https://github.com/trema/Programming-Trema/raw/master/images/2_002.png)

図2　OpenFlowの動作モデル

スイッチはホストからのパケットを受信すると，最初はその処理方法がわかりません。そこで，上司にあたるコントローラに問い合わせます。この問い合わせを `packet_in` メッセージと呼びます。コントローラはこれを受け取ると，同様のパケットが届いた場合にスイッチでどう処理すべきか （パケットを転送する，書き換えるなど） を決めます。これをアクションと呼びます。そして 「スイッチで処理すべきパケットの特徴」＋「アクション」 の組 （フローと呼びます） をスイッチのマニュアルに追加します。この命令を `flow_mod` メッセージと呼び，スイッチのマニュアルをフローテーブルと呼びます。処理すべきパケットの特徴とアクションをフローテーブルに書いておくことで，以後，これに当てはまるパケットはスイッチ側だけですばやく処理できます。忘れてはならないのが，`packet_in` メッセージで上がってきた最初のパケットです。これはコントローラに上がってきて処理待ちの状態になっているので，`packet_out` メッセージで適切な宛先に転送してあげます。

電話サポートとの大きな違いは，フローテーブルに書かれたフローには期限があり，これを過ぎると消えてしまうということです。これは，「マニュアルに書かれた内容は徐々に古くなるので，古くなった項目は消す必要がある」 と考えるとわかりやすいかもしれません。フローが消えるタイミングでコントローラには `flow_removed` メッセージが送信されます。これには，あるフローに従ってパケットがどれだけ転送されたか -- 電話サポートの例で言うと，マニュアルのある項目が何回参照されたか -- つまり，トラフィックの集計情報が記録されています。

それではしくみの話はこのぐらいにして，早速実践に移りましょう。もし途中でわからなくなったら，この節の頭から読み直してください。


# 「トラフィック集計スイッチ」 コントローラの概要

トラフィック集計スイッチは，パっと見は普通の L2 スイッチとして動作します。しかし，裏では各ホストが送信したトラフィックをカウントしており，定期的に集計情報を表示してくれます。これを使えば，ネットワークを無駄に使いすぎているホストを簡単に特定できます。

## 設計と実装

「L2 スイッチ機能」と「トラフィックの集計機能」のためにはどんな部品が必要でしょうか？ まずは，スイッチに指示を出す上司にあたるコントローラクラスが必要です。これを `TrafficMonitor` クラスと名付けましょう。また，パケットを宛先のスイッチポートへ届けるための `FDB` クラス （注1），あとはトラフィックを集計するための `Counter` クラスの 3 つが最低限必要です。

注1） FDB とは Forwarding DataBase の略で，スイッチの一般的な機能です。詳しくは続く実装で説明します。

### FDBクラス

`FDB` クラス （リスト1） は，ホストの MAC アドレスとホストが接続しているスイッチポートの対応を学習するデータベースです。このデータベースを参照することで，`packet_in` メッセージで入ってきたパケットの宛先 MAC アドレスからパケット送信先のスイッチポートを決定できます。

```ruby
class FDB
  def initialize
    @db = {} # <- 連想配列（MACアドレス→スイッチポート番号）
  end

  def lookup mac # <- MACアドレスからスイッチポート番号を引く
    @db[ mac ]
  end

  def learn mac, port_number # <- MACアドレス＋スイッチポートを学習
    @db[ mac ] = port_number
  end
end
```

リスト1　MACアドレス→スイッチポートのデータベースFDBクラス（fdb.rb）

### Counter クラス

`Counter` クラス （リスト2） は，ホスト （MAC アドレスで区別します） ごとの送信パケット数およびバイト数をカウントします。また，カウントした集計情報を表示するためのヘルパメソッドを提供します。

```ruby
class Counter
  def initialize
    @db = {} # <- ホストごとの集計情報を記録する連想配列
  end

  def add mac, packet_count, byte_count # <- ホスト （MAC アドレス = mac） の送信パケット数、バイト数を追加
    @db[ mac ] ||= { :packet_count => 0, :byte_count => 0 }
    @db[ mac ][ :packet_count ] += packet_count
    @db[ mac ][ :byte_count ] += byte_count
  end

  def each_pair &block # <- 集計情報の表示用
    @db.each_pair &block
  end
end
```

リスト2　トラフィックを記録し集計する `Counter` クラス （`counter.rb`）

### TrafficMonitor クラス

`TrafficMonitor` クラスはコントローラの本体です （リスト3）。メインの処理はリスト 3 1-3 の 3 つになります。

① `packet_in` メッセージが到着したとき，パケットを宛先のスイッチポートに転送し，フローテーブルを更新する部分
② `flow_removed` メッセージが到着したとき，トラフィック集計情報を更新する部分
③ タイマーで 10 秒ごとにトラフィックの集計情報を表示する部分


```ruby
require "counter"
require "fdb"

class TrafficMonitor < Controller
  periodic_timer_event :show_counter, 10 # (3)

  def start
    @counter = Counter.new # <- Counter オブジェクト
    @fdb = FDB.new # <- FDB オブジェクト
  end

  def packet_in datapath_id, message # (1)
    macsa = message.macsa # <- パケットを送信したホストの MAC アドレス
    macda = message.macda # <- パケットの宛先ホストの MAC アドレス

    @fdb.learn macsa, message.in_port
    @counter.add macsa, 1, message.total_len
    out_port = @fdb.lookup( macda )
    if out_port
      packet_out datapath_id, message, out_port
      flow_mod datapath_id, macsa, macda, out_port
    else
      flood datapath_id, message
    end
  end

  def flow_removed datapath_id, message # (2)
    @counter.add message.match.dl_src,message.packet_count, message.byte_count
  end

  private # <- 以下、プライベートメソッド

  def show_counter # <- カウンタを表示
    puts Time.now
    @counter.each_pair do | mac, counter |
      puts "#{ mac } #{ counter[ :packet_count ] } packets (#{ counter[ :byte_count ] } bytes)"
    end
  end

  def flow_mod datapath_id, macsa, macda, out_port # <- macsa から macda へのパケットを out_port へ転送する flow_mod を打つ
    send_flow_mod_add(
      datapath_id,
      :hard_timeout => 10, # <- flow_mod の有効期限は10秒
      :match => Match.new( :dl_src => macsa, :dl_dst => macda ),
      :actions => Trema::ActionOutput.new( out_port )
    )
  end

  def packet_out datapath_id, message, out_port # <- packet_in したメッセージを out_port へ転送
    send_packet_out(
      datapath_id,
      :packet_in => message,
      :actions => Trema::ActionOutput.new( out_port )
    )
  end

  def flood datapath_id, message # <- packet_inしたメッセージをin_port以外の全スイッチポートへ転送
    packet_out datapath_id, message, OFPP_FLOOD
  end
end
```

リスト3　本体 `TrafficMonitor` クラス （`traffic-monitor.rb`）

それでは，とくに重要な (1) の処理を詳しく見ていきましょう。なお，リスト 3 中で使われているメソッドの引数など API の詳細については，「Trema Ruby API ドキュメント」 を参照してください。

以下の説明では図 3 に示すホスト 2 台 + スイッチ 1 台からなるネットワーク構成を使います。host1 から host2 にパケットを送信したときの動作シーケンスは図 4 のようになります。

![TrafficMonitor を動作させるネットワーク構成の例](https://github.com/trema/Programming-Trema/raw/master/images/2_003.png)

図3　`TrafficMonitor` を動作させるネットワーク構成の例


![host1からhost2宛にパケットを送信したときの動作シーケンス](https://github.com/trema/Programming-Trema/raw/master/images/2_004.png)

図4　`host1` から `host2` 宛にパケットを送信したときの動作シーケンス

1. `host1` から `host2` を宛先としてパケットを送信すると，まずはスイッチにパケットが届く
2. スイッチのフローテーブルは最初はまっさらで，どう処理すればよいかわからない状態なので，コントローラである `TrafficMonitor` に `packet_in` メッセージを送る
3. `TrafficMonitor` の `packet_in` メッセージハンドラでは，`packet_in` メッセージの `in_port` （`host1` のつながるスイッチポート） と `host1` の MAC アドレスを FDB に記録する
4. また，`Counter` に記録された `host1` の送信トラフィックを 1 パケット分増やす
5. `packet_in` メッセージの宛先 MAC アドレスから転送先のスイッチポート番号を FDB に問い合わせる。この時点では `host2` のスイッチポートは学習していないので，結果は 「不明」
6. そこで，パケットを `in_port` 以外のすべてのスイッチポートに出力する `packet_out` メッセージ （FLOOD と呼ばれる） をスイッチに送り，`host2` が受信してくれることを期待する
7. スイッチは，パケットを `in_port` 以外のすべてのポートに出す

これで，最終的に `host2` がパケットを受信できます。逆に，この状態で `host1` を宛先として `host2` からパケットを送信したときの動作シーケンスは次のとおりになります （図5）。4 までの動作は図 4 と同じですが，5 からの動作が次のように異なります。

![host1 から host2 宛にパケットを送信したときの動作シーケンス](https://github.com/trema/Programming-Trema/raw/master/images/2_005.png)

図5　host1 から host2 宛にパケットを送信したときの動作シーケンス

1. `host1` から `host2` を宛先としてパケットを送信すると，まずはスイッチにパケットが届く
2. スイッチのフローテーブルは最初はまっさらで，どう処理すればよいかわからない状態なので，コントローラである `TrafficMonitor` に `packet_in` メッセージを送る
3. `TrafficMonitor` の `packet_in` メッセージハンドラでは，`packet_in` メッセージの `in_port` （`host1` のつながるスイッチポート） と `host1` の MAC アドレスを FDB に記録する
4. また，`Counter` に記録された `host1` の送信トラフィックを 1 パケット分増やす
5. `packet_in` メッセージの宛先 MAC アドレスから，転送先のスイッチポート番号を FDB に問い合わせる。これは，先ほど `host1` から `host2` にパケットを送った時点で FDB に学習させているので，送信先はスイッチポート 1 番ということがわかる
6. そこで，`TrafficMonitor` はパケットをスイッチポート 1 番へ出力する `packet_out` メッセージをスイッチに送る。スイッチはこれを受け取ると，パケットをスイッチポート 1 番に出し，最終的に `host1` がパケットを受信する
7. 「送信元 = 00:00:00:00:00:02，送信先 = 00:00:00:00:00:01 となるパケットはスイッチポート 1 番に転送せよ」 という `flow_mod` メッセージをスイッチに送信する

最後の 7 によって，以降の `host2` から `host1` へのパケットはすべてスイッチ側だけで処理されるようになります。


# 実行してみよう

それでは，早速実行してみましょう （注2）。リスト 4 の内容の仮想ネットワーク設定を `traffic-monitor.conf` として保存し，次のように実行してください。

    % ./trema run ./traffic-monitor.rb -c ./traffic-monitor.conf

リスト 4　仮想スイッチ `0xabc` に仮想ホスト `host1`，`host2` を接続する設定

```ruby
vswitch { # <- 仮想スイッチ 0xabc を定義
  datapath_id 0xabc
}

vhost ("host1") { # <- 仮想ホスト host1 を定義
  ip "192.168.0.1"
  mac "00:00:00:00:00:01"
}

vhost ("host2") { # <- 仮想ホスト host2 を定義
  ip "192.168.0.2"
  mac "00:00:00:00:00:02"
}

link "0xabc", "host1" # <- ホスト host1、host2 をスイッチ 0xabc に接続
link "0xabc", "host2"
```

実行すると，図 3 に示した仮想ネットワークが構成され，`TrafficMonitor` コントローラが起動します。

それでは，実際にトラフィックを発生させて集計されるか見てみましょう。Trema の `send_packets` コマンドを使うと，仮想ホスト間で簡単にパケットを送受信できます。別ターミナルを開き，次のコマンドを入力してください。

    % ./trema send_packets --source host1 --dest host2 --n_pkts 10 --pps 10← host1からhost2宛にパケットを10個送る
    
    % ./trema send_packets --source host2 --dest host1 --n_pkts 10 --pps 10← host2からhost1宛にパケットを10個送る

`trema run` を実行した元のターミナルに次のような出力が出ていれば成功です（注3）。

    ...
    00:00:00:00:00:01 10 packets (640 bytes)
    ↑host1からパケットが10個送信された
    
    00:00:00:00:00:02 10 packets (640 bytes)
    ↑host2からパケットが10個送信された
    ...

注2） Trema のセットアップが済んでいない人は，前回もしくは Trema のドキュメントを参考にセットアップしておいてください。なお，Trema は頻繁に更新されていますので，すでにインストールしている人も最新版にアップデートすることをお勧めします。

注3） その他のトラフィック情報も出るかもしれませんが，これは Linux カーネルが送っている IPv6 のパケットなので，`host1`， `host2` とは関係ありません。


# まとめ

今回は 「トラフィック集計機能付きスイッチ」 を実現するコントローラを書きました。学んだことは次の2つです。

* 電話サポートの例を使って OpenFlow の動作モデルを学びました。パケットの転送はスイッチ上のフローテーブルによって行われ，`flow_mod` メッセージによって書き換えることができます。また，フローテーブルに登録されていないパケットによって `packet_in` メッセージがコントローラに届きます。
* 仮想ネットワークを使ったコントローラの動作テスト方法を学びました。仮想スイッチと仮想ホストを起動してつなぎ，`send_packets` コマンドを使って仮想ホスト間でパケットを送受信することで，コントローラの簡単な動作テストができます。

次回は Trema を使ったテストファースト開発を紹介します。Rails や Sinatra を使った Web アジャイル開発ではお馴染みのテストファースト開発ですが，もちろん Trema でもサポートしています。とくに OpenFlow コントローラのように複雑な動作シーケンスを持つソフトウェアの実装には，テストファーストによるインクリメンタルな開発が有効です。
