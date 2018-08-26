PrimalSpec
============

原始的な形式仕様記述言語 (Haskell eDSL)

特徴
-------

1. 状態遷移モデルを直接エンコードせず、CSPライクな記法によりコンパクトかつ直感的な記述が可能です
    * VDMのstのような陰関数の記述も行えます。これによって非決定性も表現出来ます

1. イベント列を指定することで記述した内容についての振る舞いが確認出来ます
    * 太古のソルバ(人間)を使って陰関数や非決定性にも対応可能
    * 仕様に対するユースケース・テストケースの記述としても使えます

1. 内部遷移(τ)と並行合成はサポートしていません
    * 割り込み演算子をサポートしています。(TODO)
    * 内部選択・隠蔽・リネームもサポートしていません。ただし、非決定性について記述出来ない訳ではありません。

1. イベント発生前後の(グローバルな)状態変化を表現出来ます
    * 主に割り込み時のコンテキスト間の情報共有の為に用意しています。

1. モデル検査機ではない
    * 網羅的な検査機能は提供していません
    * デッドロック・ライブロック・詳細化関係についての機能も提供していません

作った理由
------------

* 無料でビジネスにも使えるCSPライクなツールが見つからなかったのです・・・(勉強用なら無料でFDR4が使える)


PrimalSpecでの記述方法
-------------------------------

状態遷移モデルは規模が大きくなると、記述がそもそも大変だったり読み辛かったりします。
この問題を解決する１つの手段としてCSPがあります。これは並行プロセスを記述する為に作られたものですが、状態遷移の記述にも適しています。

PrimalSpecはCSPから使いこなすのが難しい並行性と内部遷移に関する演算子を取り除いたものになっています。

### イベント

イベントは状態遷移のトリガです。PrimalSpecでは主にイベントを使って仕様を記述します。

* 仕様を記述する場合、イベントとはシステム利用者とシステムを構成するマシンの間で共有される現象をモデル化したモノになります。
    * 具体的にはセンサ、通信ポート、ディスプレイ、モーター、時間経過等についての振る舞いです。
* イベントはアトミックに発生します。つまり、あるイベントが発生している間に異なるイベントが発生するといった記述は出来ません。
* イベントにはペイロードをのせることが出来ます。
    * 例えば、```Fill 3``` で自動販売機にジュースを３本補充するというイベントを表すことが出来ます
* 終了を表す特別なイベント(✓)がありますが、これは直接ユーザーモデル上には現れません。

### プロセス式

PrimalSpecでは、プロセス式とはイベントがどういう順番で発生しうるかを表現している式だと思って下さい(トレース意味論)。

プロセス式はイベント、原始プロセス、合成演算子によって構築されます。

* 原始プロセス
    * ```Stop``` : いかなるイベントも発生しないプロセス式。異常終了やデッドロックを表すのに使う。
    * ```Skip``` : いかなるイベントも発生しないプロセス式。正常終了を表すのに使う。逐次合成や割り込みの振る舞いがStopとは異なる。
* 合成演算子
    * ```ev --> P``` : プリフィクス演算子
    * ```ev ?-> \x -> P``` : 受信演算子
    * ```ev &-> P``` : ガード
    * ```P1 |=| P2``` : 外部選択演算子
    * ```P1 >>> P2``` : 逐次合成演算子
    * ```P1 <|> P2``` : 割り込み演算子
    * ```P1 |<| P2``` : 復帰式割り込み演算子(通常コンテキスト)
    * ```P1 |>| P2``` : 復帰式割り込み演算子(割り込みコンテキスト)


### 遷移と規則の記述方法

プロセス式P が イベントaを受理した後プロセスQとして振る舞うという命題は以下のように表現します。


       a
    P ===> Q

✓を除いたイベント全体の集合を```A```で表現します。
✓を含めた P が受理可能なイベントの集合を ```α(P)``` で表現します。
各命題に対して仮説部（線の上側）が全て成立すると結論部(線の下側)の各命題がそれぞれ成立すると読んでください。
以下の場合、(P かつ Q) ならば (R かつ S) です。
論理学とは異なり、(P かつ Q) ならば (R または S) ではない点に注意して下さい。


        P
        Q
    ----------
        R
        S


### SkipとStop

    ------------------
           ✓
     Skip ===>  Stop


### Prefix演算子

任意のa ∈ A に対して、```a --> P``` は a を受け取ったらプロセスPとして振る舞うプロセスを意味します。

    ------------------
                a
     (a --> P) ===>  P


式の外で指定されたイベントのペイロードを使いたい場合は、受信演算子を使います。つまり、P の中でxはペイロードの中身を指します。

    -----------------------------
                        a.y
     (a --> \x -> P(x)) ===>  P(y)


### ガード演算子

    (b &-> P) = if b then P else Stop

### 外部選択演算子

```P |=| Q``` は P またはQとして振る舞います。P,Q のどちらのイベントが先に起こった方が選択されます。

任意のa ∈ (A U ✓ ) に対して、

       a
    P ===> P'

    a ∉ α(Q)

    --------------------
               a
    (P |=| Q) ===>  P'

               a
    (Q |=| P) ===>  Q'


### 逐次合成演算子

```P >>> Q``` は P が終わったら次にQとして振る舞うプロセスです。


       ✓
    P ===> P'

       ✓
    Q ===> Q'

    -------------------------------------
               ✓
    (P >>> Q) ===>  STOP


任意のa ∈ A に対して、

       a
    P ===> P'

    ✓  ∉ α(P) || a ∉ α(Q)

    -------------------------------------
               a
    (P >>> Q) ===>  (P' >>> Q)


任意のa ∈ A に対して、

       ✓
    P ===> P'

    a ∉ α(P)

       a
    Q ===> Q'

    -------------------------------------
               a
    (P >>> Q) ===>  Q'

### 割り込み演算子(TODO)

```P <|> Q``` は P 中にQが割り込まれる可能性のあるプロセスです。

       ✓
    P ===> P'

    -------------------------------------
               ✓
    (P <|> Q) ===>  P'


任意の a ∈ A に対して、

       a
    P ===> P'

    a ∉ α(Q)

    -------------------------------------
               a
    (P <|> Q) ===>  (P' <|> Q)


任意の a ∈ (A U ✓ ) に対して、

       a
    Q ===> Q'

    a ∉ α(P)

    -------------------------------------
               a
    (P <|> Q) ===>  Q'


### 復帰型割り込み演算子(TODO)

```P |<| Q``` は P 中にQが割り込まれる可能性のあるプロセスです。ただし、割り込みが完了すると再び元の状態に戻ります。
```P |>| Q``` は P 中にQが割り込まれている状態のプロセスです。


       ✓
    P ===> P'

    -------------------------------------
               ✓
    (P |<| Q) ===>  P'


任意の a ∈ A に対して、

       a
    P ===> P'

    a ∉ α(Q)

    -------------------------------------
               a
    (P |<| Q) ===>  (P' |<| Q)


任意の a ∈ A に対して、

       a
    Q ===> Q'

    a ∉ α(P)

    -------------------------------------
               a
    (P |<| Q) ===>  ((P |<| Q) |>| Q')


任意の a ∈ A に対して、

       a
    P ===> P'

       ✓
    Q ===> Q'

    -------------------------------------
               a
    (P |<| Q) ===>  P'


任意の a ∈ A に対して、

       a
    Q ===> Q'

    a ∉ α(P)

    -------------------------------------
               a
    (P |>| Q) ===>  (P |>| Q')



### その他のルール


```
    P    |=|  Q        = Q |=| P
    Stop |=|  P        = P
    Stop >>>  P        = Stop
    P    <|>  Stop     = P
    Stop |<|  P        = P
    P    |<|  Stop     = P
    P    |>|  Stop     = Stop
    P    <||> Q        = Q <||> P
    P    <||> P        = P
    Stop <||> P        = Stop
```


### 状態変化

プロセス式はHaskellのeDSLで記述されます。

プロセス変数というものは存在しませんが、
Haskellの関数を使ってプロセスを抽象化することが出来ます。
関数なので引数を使えば、インデックス付けられたプロセス式の表現も可能になります。

```
heater :: Int -> Process
heater n = Up --> heater (n+1)
```

上の方法でローカルな状態変化を表現することが出来ます。


### SuchThat

SuchThatによって陰関数や非決定性を表現出来ます。

```haskell
s_t_ :: String -> (a -> Bool) -> a
```

以下のように使えます。

```haskell
f :: a -> b
f a = s_t_ "message" \ret -> p a ret
    where
    p :: a -> b -> Bool
    p = xxxxx
```

ただし、動作させる場合は自分で答えをコンソール等から入力しなければなりません。
答えが間違っていたらエラーになります。


記述例
-----------

では、実際に PrimalSpecで自動販売機の仕様を記述していみましょう。

demo/Main.hs

をご覧下さい。


参考文献
---------

* "Concurrent and Real Time Systems: the CSP approach" (S.Schneider) の前半



