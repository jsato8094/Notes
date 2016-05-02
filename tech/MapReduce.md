# MapReduce: Simplified Data Processing on Large Clusters

Jeffrey Dean and Sanjay Ghemawat, OSDI 2004

## 概要

MapReduceは大規模データセットの処理及び生成のためのプログラミングモデルおよびその実装である．
大まかには，ユーザがmap/reduce関数（後述）を定義すれば，ユーザは並行処理や分散システムについてあまり考えることなく処理を記述できる．
主な利点は以下である．
- 簡単に並行処理がかける
- Fault-toleranceである
- スケーラブルである

## Programming Model

### Example

word countingの例

```
map(String key, String value):+1:
  // key: document name
  // value: document contents
  for each word w in value:
    EmitIntermediate(w, "1");
```

```
reduce(String key, Iterator values):
  // key: a word
  // values: a list of counts
  int result = 0;
  for each v in values:
    result += ParseInt(v);
  Emit(AsString(result));
```

`map`は単語を数え上げていく．
`reduce`は各単語についてカウントを足し合わせていく．

### Types

入力のkey/valueは出力のkey/valueの型とは異なる．
文字数カウントなら，入力：文書名/文書，出力：単語/カウント．

中間のkey/valueは，出力のkey/valueの型と同じ．

### More Examples

- __Distrubuted grep__: パターンマッチ検索．
- __Count of URL access frequency__: webのログからURL/カウントを中間生成，reduceで集計．
- __Reverse web-link graph__: ソース--map-->(target,source)--reduce-->(target,list(source))
- __Term-vector per host__: (URL,document)--map-->(hostname, term vector)--reduce-->(hostname, term vector)
- __Inverted index__: (document ID, document)--map-->(word, document ID)--reduce-->(word, list(document ID))
- __Distributed sort__: (record ID, record)--map-->(key, record)--reduce-->(key, record)(unchanged)

## Implementation

構成マシンやネットワークの説明．
1クラスタあたり数百〜数千のマシン．

### Execution Overview

![Kobito.utkIud.png](https://qiita-image-store.s3.amazonaws.com/0/123849/f0638263-864d-2131-bd32-9e91172dade4.png "Kobito.utkIud.png")

1. MapReduceライブラリが入力ファイルをM個に分ける．だいたい16〜64MB/個．クラスタにプログラムをコピー．
2. 1つだけmasterと呼ばれるプログラムのコピーが存在．残りがworkerで，M個のmap処理とR個のreduce処理をmasterから割り振られる．masterは待ち状態のworkerを選んで処理を割り振っていく．
3. map処理を割り当てられたworkerは，対応する分割データを読み込む．これからkey/valueのペアをパースし，ユーザが定義したMap関数に渡す．中間ペアをメモリに記憶していく．
4. 定期的に，バッファされている中間ペアをローカルディスクに書き出す．この際，partitioning functionによりR個の領域に分けて書き込まれていく（map処理workerがM個あり，それぞれのローカルにR個のファイルが生成される）．書き込まれたキーペアの場所はmasterに渡され，masterはこの情報をreduce担当のworkerに渡す．
5. reduce担当のworkerがmasterから中間ペアの場所を通知されると，remote procedure callなるものを用いてmap担当workerが持っている中間ペアを読み出す．すべての中間データを読み終えると，キーでソートして同じキーのペアをグループ化する．中間データのサイズがメモリに対して大きすぎる場合は，external sortなるものが使われる．
6. reduce担当workerはソートされた中間データを順に捜査して，各キーとそのキーに対応する値（複数のペア）をユーザが定義したReduce関数に渡す．Reduce関数の出力は最終出力ファイルに都度結合される（workerごと）．
7. すべてのmap処理とreduce処理が終わったら，masterはユーザのプログラムを起こす．

R個の出力ファイルは，通常結合されずにそのまま次の分散処理などに使われる．

### Master Data Structures

masterは中間ファイルのありかとか，workerの状態とか色々しってる．map処理が終わるごとに中間ファイル情報が更新され，masterはreduce処理worker（稼働中）に都度情報を渡す．

### Fault Tolerance

一度にたくさんのマシンを使うからfault tolerance大事だよ．

#### Worker Failure

masterはすべてのworkerに定期的にpingする． しばらく返事がなければそいつは死んだと判断する．死んだworkerが担当し完了したmap処理は一旦すべて待ち状態に戻され，他のworkerに再度スケジュールされる（map処理の結果は死んだworkerのローカルに記録されているため．reduce処理の結果はグローバルなファイルシステムに保存される）．処理継続状態であったmap処理およびreduce処理も再スケジュールされる．
一度完了していたmapが再処理されると，すべてのreduce担当workerに通知され，死んだworkerからまだデータを読みだしていなかったreduce担当workerは新しいmap担当workerから読み出すようになる．

#### Master Failure

定期的にチェックポイントを書き出しておき，死んだら最新のチェックポイントから新しいmasterのコピーを走らせる．ただ，masterが1つしかないためアボートするので，処理を再スタートしないといけない．

#### Semantics in the Presence of Failures

よくわからなかった．．．
ユーザがdeterministicなmap, reduce関数（operationと記述されている）を用意すれば，並列でない処理と全く同等の結果が保証される．関数の原子性の話で，それはわかる．
一方で，map and/or reduceがnon-deterministicな場合は，weakerだけどstill reasonable semanticsを提供するよ．．．というのは？？？

### Locality

ネットワークの帯域はシステムの中では比較的貧弱なリソース．map処理では入力データをコピーするが，あるworkerで失敗した場合，同じネットワーク内のworkerで再スケジュールすることで帯域を消費しないようにするなど工夫している．

### Task Granularity

MやRは，worker数に対してできるだけ大きくすると，負荷分散や再スケジュールが容易になる．
一方で，masterは _O(M+R)_ のスケジュール決定と， _O(M*R)_ のメモリを消費する．また，Rは最終的な出力ファイル数なので，多すぎると後で大変．
結局，Mは各入力ファイルが16〜64MBになるように，またRはreduce担当workerの数倍程度にしている．実際，M=200,000，R=5,000，worker数=2,000である．

### Backup Tasks

長いMapReduce処理では，なんらかの不具合で遅いworker("straggler")が最後めっちゃ時間を食うという問題がある．そこで，処理が終盤になったら，現在進行中の処理を，余っているworkerでも走らせ，同じ処理のどちらかが終われば完了とすることで，stragglerが処理を持ち続けて終わらない現象を回避できる．

## Refinements

map&reduce処理で基本ぜんぶいけるんだけど，それ以外の便利なものいくつかについて．

### Partitioning Function

Rに分割するためのもの．デフォルトはハッシュ（e.g. `hash(key) mod R`）．基本バランスのよい分割になる．
他には，出力キーがURLで，同じホストのURLを一つの出力ファイルにまとめたい時は，MapReduceライブラリの`hash(Hostname(urlkey)) mod R`を使うと，そのようにしてくれるらしい．

### Ordering Guarantees

中間ファイルの時点でincreasing key orderでソートしてくれていて，ソートされた出力ファイル生成が楽だし，それによって後々の検索などが楽．

### Combiner Function

文字カウントの例でいうと，map処理でめっちゃ出てくる`<the, 1>`のような中間キーをそのまま扱うと，`the`を割り当てられたreduce担当workerからほぼすべてのmap担当workerを参照しないといけなくなるので，ネットワークの帯域を消費する．
そこで，オプションでCombiner functionを使うと，map-->combinerで一旦`the`を一つの中間ファイルにまとめる-->reduceに送信，としてくれるので帯域を節約できる．

### Input and Output Types

幾つかのreader interfaceが定義されていて，例えばtextモードだと行ごととかでkey/valueをパースできる．自前で新しく実装もできる．出力もpre-definedなやつがあり，追加実装も簡単．

### Side-effects

結果の出力ファイルとは別に，map/reduce処理でなんらかの出力ファイル書き出しもできる．atomicityには気をつけること．ライブラリのサポートは特にない．

### Skipping Bad Records

幾つかのレコードに対してバグによってエラーが発生しちゃうけど，大した問題にならないなどで処理を進めたいときに，当該レコードをスキップして処理を進めることができる．
操作開始前に，MapReduceライプラリは引数の連番をグローバルに持っておく．設定しておいたシグナルを受け取ったら，レコードの番号をmasterに送る．同一のレコード番号に対して複数のエラーが出ていれば，そのレコードは飛ばすべきと判断して再スケジュールする．

### Local Execution

ローカルでのテスト用にわざわざ別のライブラリ用意してます．

### Status Information

masterは内部のHTTPサーバを走らせて，ステータスページを出力する．エラーとか出力とか時間とかどのworkerが死んだとか確認できます．

### Counters

諸々のイベントを（すべてのworker通して）カウントするための機能がある．例：

```
Counter* uppercase;
  uppercase = GetCounter("uppercase");
  map(String name, String contents):
    for each word w in contents:
      if (IsCapitalized(w)):
        uppercase->Increment();
      EmitIntermediate(w, "1");
```

## Performance

