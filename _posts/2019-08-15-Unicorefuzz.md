---
layout: post
title:  Unicorefuzz:カーネル空間でのエミュレーションの可能性について
date:   2019-08-15 19:00:00 +0900
categories: [fuzz]
---

- [Unicorefuzz: On the Viability of Emulation for Kernelspace Fuzzing](https://www.usenix.org/system/files/woot19-paper_maier.pdf) を翻訳したものです

# アブストラクト

Fuzzingは、深刻な脆弱性を次々と発見している。ターゲットがクラッシュするまで実行するという単純な概念にもかかわらず。ファズ・テストの設定は複雑な問題を引き起こす可能性がある。これは特に、デバイスドライバーやカーネルコンポーネントなど、デスクトップオペレーティングシステムでユーザーランドプロセスの一部として実行できないコードに当てはまる。本論文では,カーネル空間における任意のパーサをカバーベースのフィードバックでファズするためのCPUエミュレーションの利用を検討した。筆者らは、オープンソースのUnicorefuzzを提案し、エミュレーション型ファジングアプローチの利点と落とし穴を説明した。このアプローチの実行可能性を、人工Linuxカーネルモジュール、Open vSwitchネットワーク仮想化コンポーネント、およびsyzkallerによって最初に発見されたバグに対して評価した。エミュレータベースのカーネルコードのファジングは、設定がそれほど複雑ではなく、ソースコードが利用できないオペレーティングシステムやデバイスをファズするためにも使用することができる。

# 1 はじめに

ファズ・テストは、さまざまなバグを自動的に発見する強力な方法です。企業は、Serebryany[36]が提案したOSS-Fuzzのようなファジングなパイプラインの構築を始めます。ユーザ空間ソフトウェアのファジングは数十年間[18]行われてきましたが、オペレーティングシステムとカーネルのファジングには、より困難な作業が伴います[34]。エミュレーター内の任意のカーネル・コンポーネントを直接フリーズさせることができれば、セキュリティー研究者はカーネルの任意の部分を詳細に調べることができ、セキュリティーの脆弱性に対する継続的な戦いにおいて大きな優位性を得ることができます。数年前には、Wi-Fiドライバーなどのカーネル・コードのファジングには複雑な設定が必要でした。それらは、フィードバックベースのファジング方法論に統合するのが難しい専用の不正アクセスポイント [6] のような現実のシナリオに似ていなければならなかった。どのような状態変更もシステム全体に影響を与えるため、カーネル空間コードのファジングは困難であり、コードはユーザランドから離れてしまい、クラッシュからの復旧は追加の抽象化レイヤなしではほとんど不可能です。カーネルのファジング、syzkallerの現在の技術では、仮想マシン内のカーネルをファジングし、システムコールを介してVM内部のユーザランドからコードをトリガし、ユーザランドにフィードバックを返すことによって、この問題を解決しています[17]。ただしこれは、すべてのテストのシステムコールが存在するか、または記述する必要があることを意味します。また、バイナリのみのカーネルモジュールや奇妙なオペレーティングシステムのセキュリティテストを困難にし、不可能にします。本論文では,問題領域に対する異なるアプローチを採用した。CPUエミュレータベースのファザーであるAFL‐Unicornに基づくカーネル空間におけるファズ・パーサへの新しい方法であるUnicorefuzzを提案した。Unicornエミュレータは広範囲のプロセッサアーキテクチャ[31]をサポートしており,任意のカーネルコードのファジングを,組込みアーキテクチャに対してさえも実現可能にしている。本論文では,任意のカーネル空間関数のファジングが可能であり,実行可能であることを示す。

## 1.1 基本用語

最初に、本論文で使用する基本用語を定義する。

**ファザー**

ファザーは,その挙動を観察しながら,ファジングターゲットへの入力を連続的に生成する。さらに分析するために、目的の動作をトリガする入力を収集します。

**ターゲット**

ファズターゲットは、ファザーを使用してテストされるアイテムです。使用する方法に応じて、解釈済みスクリプトからハードウェアデバイスまで、さまざまなものがあります。主な要件は、入力を受け入れ、結果として観察可能な動作を示すことです。

**カバレッジ**

無条件に実行されるステップまたは命令の連続ブロックは、基本ブロックと呼ばれ、ブランチは、基本ブロック間の条件付き転送である。入力に応じて、異なるパス(すなわち、基本ブロックのチェーン)がトリガされる場合があります。ターゲットに取り込まれるブランチの量は、カバレッジに影響します (数が多いほど、適用範囲は大きくなります)。正規化されたカバレージは、1回の実行で取得される可能性のあるすべてのブランチの割合を示します。

## 1.2 フィードバックのファジング

フィードバックファジングは、入力の効果に関する即時のフィードバックを受け取るような方法でファズターゲットを測定する。このフィードバックによると、将来のイテレーションでは、ファザーによってテストケースが変更されます。最も有名な例は、カバレッジガイド方式のぼかしです。これは、コードカバレッジ、または、密接に関連して、ファザーの遺伝的アルゴリズム[26]のためのフィードバックとして、トリガされる個別の基本ブロック遷移の数を使用する。ファザーは、より高いカバレッジと異なるコードパスをもたらすテストケースを好みます。目標はターゲットにおけるコードカバレッジの最大化である。遺伝的アルゴリズムの適合度関数としてコードカバレッジを採用することにより,ファザーはプログラムの入力文法のモデルを反復的に構築することができ,それはテストされたアプリケーションに与えられた入力を改善する。遺伝的アルゴリズムを通して試験入力を学習するために計装を使用する考えは,Stahmerにより1995年に既に提案されている[39]。2006年、Jared DeMottは再びこのアプローチを採用し、進化するファジングツール[1,112]を提案した。DrweryとOrmandyは同時に同様の方法[1,516]を提案し、特許を取得した。今日の著名なファザーであるAFLとlibFuzzerはどちらも、以前は実行されることが観測されていなかったコード・パスを引き起こす入力を好みます[27、43]。カバレッジ・ガイドによるファジングであっても、有効な入力ファイルを使用することが推奨されており、早い段階でより高いコードカバレッジが得られることが期待されます[37、43]。

## 1.3 貢献

Unicorefuzzを用いて,筆者らはカーネルファジングに対する新しいアプローチを提案した。パーサを部分エミュレーションによってファズする可能性について議論します。この手法は人工的なテストケースで良好な結果をもたらし,syzkallerによって以前に発見されたバグの再現に成功した。ケース・スタディとして、Open vSwitchのカーネル・コードにパーサーを追加しました。現在の設定は、静的にリンクされたカーネルモジュールやLinux上でロード可能なカーネルオブジェクトをファズするために使われていました。この調査で作成された実装はオープンソースになります。

# 2 関連研究

カーネル・コンポーネントのファジングは、ユーザー空間プログラムのファジングほどは普及していませんが、いくつかの注目すべきプロジェクトは、カーネル・サブシステム、ドライバー、APIのファジングに成功しています。デバイスドライバーを混乱させる出版物、例えば、Song et al.[38]によるPeriscopeなど、ハードウェアとOSの境界に焦点を当てた出版物については、これ以上詳細には触れない。代わりに、以下では注目すべき汎用カーネルファザーについて説明します。

## 2.1 トリニティ

syscallを純粋にランダムな値でファズすることは簡単に設定でき、いくつかのバグを見つけることができますが、ほとんどのテストは、パラメータが完全に間違っているために即座に失敗します(無効な引数)。無駄なテストの割合を減らすために、カーネルfuzzer Trinityはルールを使ってランダムより良い入力でシステムコールを実行します。ユーザは全てのsyscallに対して、引数の数や引数の種類を含む、いわゆるmadviseファイルを提供する必要があります。Trinityの開発者は2012年、システムコールコード、ネットワークスタック、仮想メモリ管理、デバイスドライバ[24]に150以上のバグを発見した。

## 2.2 DIFUZE

Corina et al.は、カーネル・ドライバー用のインターフェース認識型ファザーであるDIFUZEを提案している[9]。Trinityとは対照的に、DIFUZEはインターフェースの記述を必要とせず、ソースからカーネルを構築する際に推測されます。DIFUZEにはインストゥルメンテーションオプションはありません。

## 2.3 syzkaller

Trinityと同様に、syzkallerは無駄なテストの割合を減らすためにテンプレートに依存しますが、fuzzテストのコード・カバレッジを増やすためにカバレッジ・ベースのフィードバックfuzingを使います[17、42]。カバレッジ・データはタスクごとに追跡され、現在の計測情報[42]を出力する/sys/kernel/debug/kcovの特別なdebugfsエントリーを介してターゲットから抽出されます。このカーネルパッチは、すでにメインラインのLinuxカーネルに統合されています[23]。syzkaller氏は最初の数カ月だけで、当時最新のLinuxカーネルに150以上のバグを発見した。

## 2.4 カーネル内AFL

NossumとCasasnovasはAFLをカーネル空間に移植し、ファイルシステムの実装をテストしました。syzkallerと同じように、名前のないツールはGCCがサポートするカバレッジ・データを使用し、/dev/aflのユーザ空間からアクセス可能な共有メモリ領域を公開し、それをAFLのメモリにmmapします。このようにして、セクションは物理メモリーに1回だけ存在します。AFLは、ユーザー空間バイナリーをテストする場合と同じ方法でアクセスすることができます。KVM VMによる仮想化のオーバーヘッドが高かったため、KVM VMはUser-Mode Linuxに切り替えられ、60倍の高速化を実現しました。バグは、多くのファイルシステムドライバ[33]の実装ですぐに発見されました。

## 2.5 TriforceAFL

TriforceAFLは、AFLのQEMUモードを拡張することで仮想マシンをフリーズさせます。ホストはAFLとQEMUを実行し、ゲストはいわゆるドライバー (ターゲットを実行するユーザーランド・プログラム) を実行します。AFLは生成された入力をドライバに送り、QEMUはAFLにエッジ・トレースを送り返します[21]。ドライバは、QEMU内の専用のhypercallを介してTriforceAFL設定と通信します。forkシステム・コールは、呼び出しスレッドの状態を新しいプロセスに移動するだけで、スレッド化オブジェクト (ロックなど) を未定義の状態のままにしておく場合があります。QEMUは完全なシステム・エミュレーションに3つのスレッドを使用するため、TriforceAFLは必要なVM状態を明示的に保存してリストアします。TriforceAFLは、LinuxとOpenBSDで複数のバグを発見しました。Linuxで最も深刻なバグは、CVE-2016-4998(情報の開示やヒープ破壊につながる可能性のあるカーネルメモリの範囲外の読み取り)とCVE-2016-4997(正の場合の任意のカーネル整数の減少)です[21]。

図1:Unicorefuzzのアーキテクチャ

## 2.6 kAFL

fuzzer kAFLはインテルのプロセッサー・トレースを利用して、ハードウェア支援によるコードカバレッジを回復します。OSに依存しないカバレッジガイド付きのカーネルファジングをサポートし、Linux、macOS、Windowsドライバにいくつかのバグを発見しました。VMとkAFL間の通信チャネルを提供するために,QEMU(QEMU-PT)とKVM(KVM-PT)の修正版は新しいハイパーコールを提供する。VMが起動した後、エージェントはパニックハンドラのアドレス(またはWindowsのBugCheck)をQEMU-PTに送信し、QEMU-PTはそのアドレスのコードをHC_SUBMIT_PANEL hypercallに置き換えます。ターゲットがパニックすると、QEMU-PTはただちに通知されます。その後、エージェントは実際のユーザー・モード・エージェント(HC_GET_PROGRAM)をフェッチして起動し、CR3レジスタの値を送信し(HC_SUBMIT_CR3)、入力を要求する場所を宣言します(HC_SUBMIT_BUFFER)。これで、メインループを実行する準備ができました。PTデータはデコードされ、AFLのビットマップに変換され、kAFLに供給されて、テストの取ったブランチを評価する。どのベンチマークプラットフォームにおいても、kAFLはTriforceAFLの20倍以上の実行/秒を実現しています[34]。

# 3 Unicorefuzz

このセクションでは、Unicorefuzzの構成要素と実装自体について説明します。

エミュレータベースのインストゥルメンテーションカーネルコードのフィードバックファジングのために、UnicorefuzzはAFL-Unicorn上に構築されています。これはQEMUのフォークであるUnicornのパッチバージョンを使用します。1.2で説明したように、最新のファザーはフィードバックを使用します。通常、テストケース生成にはコードカバレッジや到達した基本ブロックが使用されます。QEMUなどの高度なエミュレーターは、基本ブロックごとに基本ブロックを動的に変換するので [3]、インストルメンテーションをエミュレーターまたは変換されたコードに簡単に追加できます。このようにターゲットをインスツルメンテーションすることで、ソースからコンパイルすることなく、フィードバックファジングによってテストできます。

UnicornはCPU命令をサポートされているバイナリ命令セットからホスト命令セットに動的に変換するCPUエミュレータです。特に、CPUプラットフォームのバイナリであるQEMUまたはUnicornの場合は、実行時に次の手順を実行することで動作します。

基本コード・ブロックをターゲット・プラットフォームの命令セットからホスト・プラットフォームの命令セットに変換します。

変換されたブロックをキャッシュします。

ソースプログラムカウンタからターゲットプログラムカウンタへのマッピングをアドレス参照テーブルに格納します。

変換されたブロックを実行します。

次の検出されたブロックについても同じ手順を繰り返します。

変換ブロックは、設計上の基本ブロックに似ています [3]。この変換ブロックと基本ブロックとの対応関係を利用し、また、各実行後にエミュレータに実行を戻すので、プログラムカウンタを新しい基本ブロックごとのフィードバックとして使用して、コンパイル時計装と同様の計装を実装することができる。AFLは一般的なフィードバックベースのファザーであり,QEMUモードでこの計測を提供する。AFL QEMUモードでは、パッチが適用されたQEMUでファズ・ターゲットがエミュレートされます。このQEMUでは、実行されたブランチがAFLに戻されます。速度を上げるために、forkserverは、ファズ・ランの間に状態を完全にロードされた初期状態にリセットします。新しい基本ブロックが変換された後、fork子の親もそれを変換し、結果をキャッシュして、コストのかかる変換が次のfork後に再び必要にならないようにします。制御は各ブロックの後にQEMUに戻り、共有メモリーを介してカバレッジ・マップをAFLに戻す関数afl_maybe_logを呼び出すことで拡張されます。

## 3.1 AFL-Unicorn

AFL-Unicornモードは、もともと前述のQEMU[31]から分岐した軽量CPUエミュレータであるUnicorn Engineを利用しています。AFL-QEMUでは、エミュレーターはすべての基本ブロックを拡張してインストルメンテーション情報をAFLに送信し、エミュレーターのセットアップ後にAFLのforkserverを起動します。速度を上げるために、基本ブロックはUnicornのAFL forkserverの親プロセスにキャッシュされます。AFL自体は単にハーネスを起動して入力を生成するだけです。Unicorefzz forkserverの内部動作を図に示します。3.3.図は左から右にタイムラインとして読むことができます。最初に、AFLは最初のカーネル・メモリー(中央で)をロードするUnicorefuzzハーネスを生成します。その後、ハーネスのカーネル部分は子プロセスをフォークし、子プロセスはそれぞれ出口に達するまで実行されます。子カーネル(底部)が動的に新しいメモリ領域を必要としたり、新しい基本ブロックを変換したりすると、親カーネルに報告します。ブロックはパイプを介して報告され、親によって変換されるため、次の実行のためにすでに変換され、メモリはディスクに格納されます。これにより、AFL-Unicornはすべてのバックエンドプロセッサアーキテクチャ[41]でサポートされるすべてのフロントエンドプロセッサアーキテクチャのバイナリのみのマシンコードをファズすることができます。

## 3.2 avator2

もう一つのUnicorefuzzとして、avatar2はオーケストレーションフレームワークであり、一連のデバッガーとバイナリ分析ターゲット[30]を抽象化します。後述するように、Unicorefuzzはプロセス・メモリーとfuzzターゲットのレジスターをダンプするためにavatar2を多用します。オーケストレーションフレームワークは、バイナリ分析フレームワーク、スマートデバイス、任意のGDBセッション、PANDAおよびQEMU VMに接続できます。

## 3.3 Unicorefuzz

ここでは、エミュレーションによってほとんどのターゲットをファズできるフレームワーク、Unicorefuzzについて説明します。

### 3.3.1 準備

AFL-Unicornは任意のバイナリをファズ・テストできるため、カーネル・コンポーネントもファズ・テストできます。ハードウェアやエミュレートされたデバイスとのやり取りはUnicornの一部ではありませんが、ほとんどすべてのコードをエミュレートできます。周辺のモデリングはUnicornでは容易な作業ではないので,筆者らはパーサーのエミュレーションに焦点を当てた。これらのパーサーは通常、信頼できない入力のバッファー上で動作し、周辺デバイスからの応答を必要としません。準備段階で、Unicorefuzzはすべてのレジスタをコピーし、AFLによって生成された入力を挿入し、エミュレーションを実行する必要があります。このため、Unicorefuzzはavatar2フレームワークを使用してgdbスタブと対話します。ユーザーは、ブレークポイントと、ハーネスでのAFL入力用のメモリ領域を指定する必要があります。実際のデバイスはUnicorefuzzを使用してファズすることができますが、ターゲット関数にブレークポイントを持つ仮想マシンを使用しました。ユーザーはラッパーでfuzzターゲットアドレスを指定し、それを起動して、仮想マシンに必要な機能を実行させる必要があります。ブレークポイントが発生すると、VMがフリーズし、必要なすべてのレジスタがダンプされ、動的マッピングメモリ要求を待ちます。

### 3.3.2 ダイナミックマッピング

ターゲットのメモリー全体をダンプし、それをハーネスにマッピングして関数をファズすることは可能ですが、その目的を達成できず、不必要なメモリー・オーバーヘッドが発生します。ターゲットVMをアクティブにし(停止した状態で)、fuzzerへの通信チャネルを作成することで、Unicorefuzzのプローブ・ラッパーは必要に応じて新しいメモリー・ページをロードします。1.UnicornがマップされていないメモリーにアクセスするたびにUC_HOOK_MEM_UNMAPPEDハンドラーを呼び出し、メモリー情報要求をUnicorefuzzのプローブ・ラッパー・コンポーネントに送信します。要求は正常に処理され(要求されたメモリが有効かどうか)、エミュレータにマッピングされるか、または拒否されます。その場合、Unicorefuzzは強制的にクラッシュします。成功すると、対応するメモリ領域もディスクにキャッシュされます。そうしないと、無効なメモリアクセスが見つかり、SIGSEGVによってプロセスが強制終了され、クラッシュがAFLに報告されます。

### 3.3.3 ファズテストの実行

キャッシュ機構のおかげで、VMとの相互作用はもう必要ありません-マッピングはこの時点でプレフォークです。Unicorefuzzのハーネスは、Unicorn Engineオブジェクトを作成し、必要なメモリー領域と以前に要求されたメモリー領域をすべてマップし、それに応じてレジスターの値を設定します。mem_write()またはreg_write()を呼び出して、入力をUnicornに渡す必要があります。ユーザーがfuzz入力用のアドレスを指定すると、AFL forkserverは1つの命令のエミュレーションを実行して起動します。エミュレーションがマッピングされていないメモリを再び参照すると、ハーネスは前回の実行でメモリ領域が要求されたかどうかをチェックします。プローブラッパーの出力ディレクトリにすでにダンプされたメモリーが含まれているか、拒否されている場合は、ただちに続行または中止できます。それ以外の場合は、ラッパーの入力ディレクトリ(ここで、ファイル名は要求されたアドレスを示します。)に空のファイルを作成し、ラッパーの出力ディレクトリをポーリングしてメモリー領域ダンプを探します。入力フォルダにファイルが存在する場合は、要求されたメモリを仮想マシンから読み取ろうとし、メモリが正常に読み取られた場合は出力フォルダにダンプし、それ以外の場合は拒否ファイルを作成します。

# 4 評価

エミュレータベースのカーネルファジングの実行可能性を評価するために,この概念を合成および実世界のターゲットの両方に対して試験した。

## 4.1 broken.ko

パフォーマンス分析とツールテストのために、私たちは意図的に壊れた様々なカーネルモジュールを作成しましたが、Unicorefuzzはほとんどすぐにバグを見つけることができます。次のモジュールを例にとります。

リスト1:カーネルモジュールのクラッシュ

```c
static ssize_t
write_callback ( struct file *file , const char __user * buf, size_t len , loff_t *offset ) {
    if(buf[0] =='A') {
        int a = 2;
        a -= 2;
        if (5/a > 0) {
            printk(KERN_INFO "this will never happen!\n");
        }
    }
    return len;
}
```

図2:Unicorefuzzが追加されたAFL-Unicorn forkserverの簡単な概要。

リスト1のカーネル・モジュールは、procfsエントリーに対する特定の入力を受け取るとクラッシュします。この関数のprocfs write_callbackは、(次の場合のみ)入力の最初のバイトが0x41('A')の場合は0で割ります。テストケースの設定は簡単ですが、まだ手作業が必要です。

1. write_callbackのアドレスと戻りアドレスを決定し、それをUnicorefuzzハーネスに追加します。
1. 入力バッファーアドレスを決定し、Unicorefuzzプローブラッパーに追加する
1. Fuzz

AFLコンポーネントはほとんど空の入力ディレクトリーで開始されましたが、誤検出やハングが発生することなく、入力のクラッシュを即座に検出することができました。コンパイラは0による除算を無効なオペコードに置き換えましたが、Unicorefuzzは無効な命令をキャッチし、それを適切にクラッシュとして報告します。これと他のテストケースは、カーネルファザーとしての実行可能性を証明します。procfsエントリは単なる例であることに注意してください。この概念はカーネル内の他の関数にも適用できます。

## 4.2 vSwitchを開く

ネットワーク入力を解析しなければならないソフトウェアのバグは、危険な影響を伴うセキュリティ問題であることが多い。以前の研究ではOpen vSwitchのユーザー空間コンポーネント[40]にバグが見つかっているため、Linuxソース・ツリーにある対応するカーネル空間には興味深いターゲットがあります。

### 4.2 .1 sk_buff

key_extract関数は、イーサネット・フレームからいわゆるフロー・キーを抽出します。パケットを含むsk_buffへのポインタと、フローキーが書き込まれるsw_flow_keyへのポインタを取得する。プローブラッパーはkey_extractでブレークするように適合されています。正しいパラメータをkey_extractに渡すには、手動による分析作業が必要です。sk_buffには、fuzzerに公開される可能性のある複数のフィールドがあります。図3は、sk_buff内の関連する部分の基本的な概要を示しています。データ構造体はLinuxによって前処理されるため、データ・バッファ以外の他の変数を使いこなすのは難しいかもしれません。ポインターを装った変数をファジングすると、間違いなく多くの誤検出が発生し、内部APIではさらに多くの誤検出が発生します。このため、最も有望な結果が得られると期待して、データ部分をぼかしました。それにもかかわらず、メタデータのファジングは興味深いテーマですが、真のバグと偽陽性を区別することはほとんど不可能でしょう。最終的には,データパケットデータのみをファザー化した。

図3:sk_bufとそのデータのレイアウト

### 4.2.2 結果

6500回以上のサイクルの後、AFLは62の新しいエッジを発見し、入力は正しいアドレスにコピーされました。key_extractにクラッシュやハングはありませんでした。可能性としては、outパラメータに不正なロジックがあったとしても(sw_flow_key [sw_flow_key])、コードの後半までバグは発生しません。より大きなスコープを選択し、スタック・トレースのさらに上の方から開始して、より大きなコードの塊 (スタック・トレースのいくつかの関数) を実行することは、将来テストされる予定です。

## 4.3 Syzbotバグの再発見

syzkaller[17]を開発したチームは、syzbotという、syzkallerで発見された何百ものバグをリストアップしたオープンなダッシュボードを提供している。この評価の一環として、次の場所で再現可能なsyzkallerバグを選択しました。

```
int ip_do_fragment(struct net *net , struct sock *sk , struct sk_buff *skb , int (*output)(struct net *, struct sock *, struct sk_buff*))
```

この関数は、関数の中でより深い部分で実際のBUG()をトリガーし、Open vSwitch用にすでにフリーズしたのと同じ内部ソケット・データ構造体であるsk_bufを使って呼び出しているため、完璧に見えました。正しいカーネル関数をセットアップした後、関数エントリのブレークポイントで再生スクリプトを実行すると、Unicorefuzzはファジングによってバグを発見します。今回は成功したが、その結果を価値あるものとは考えていない。このシナリオでは、syzkallerを実行して、この関数がクラッシュする状態にカーネルを移行していましたが、Unicorefuzzでは対応していません。これは、状態が関数呼び出しのたびにリセットされるためです。

リスト2:「ポート」カーネル・モジュール

```c
#include <stdlib.h>
#include <stdio.h>

static size_t
write_callback( void* file, char * buf, size_t len, void * offset) {
    if(buf[0]=='A') {
        int a = 2;
        a -= 2;
        if( 5/a > 0) {
            printf("nop\n");
        }
    }
}

void
main () {
    char input[1024];
    fgets(input , sizeof(input), stdin);
    write_callback(0, input, 0, 0);
}
```

図4:インテルCore i5-5257U(2.7 GHz)でより大きなカーネル関数をエミュレートする場合、約200人/秒

## 4.4 速度

Unicorefuzzの性能影響を評価するために,AFLのQEMUモードに対する性能を比較した。QEMUモードはユーザー空間バイナリーをファズすることしかできないため、セクション4.1でカーネル・モジュールからテストされた関数はユーザー空間バイナリーに移植されました。依存関係が最小限で関数の複雑さが低いため、移植は簡単でした。ソース・コード全体をリスト2に示します。テストはThinkPad T520(インテルCore i7-2670QM@2.20 GHz)で実行し、結果を図4.4に示します。ユニコーン・ベースのファジング設定では、QEMUモードのユーザー空間バイナリーと比較して、実行回数/秒が47%になります。順番に。はネイティブにコンパイルされたAFLよりもパフォーマンスにオーバーヘッドがあります。

図5:AFL、QEMUモードでのAFL、およびUnicorefuzzの速度比較

# 5 ディスカッション

ここでは、設計上の決定事項と、Unicorefuzzの構築中に直面した実際的な障害について説明します。

## 5.1 シード

プログラムの有効な入力を収集することは,初期入力生成のための簡単で効果的な方法を提供する。ツールの有効な使用、またはUnicorefuzzの場合の関数呼び出しには、問題のツールに関するドメイン固有の知識が含まれます。大規模なテストケースでは、より多くのデータを処理する必要があるため、実行に時間がかかります。同じコード・フローをカバーする多くの類似したシードは、新しい興味深いコーナー・ケースを引き起こすことなく、さらに時間がかかる実行につながります。空の入力やシングルバイトがfuzzターゲットのコードパスをトリガーすることはほとんどありません。代わりに、ある種の初期入力またはそれを生成する方法は、通常、ターゲットをファジングするときに適用されます。Unicorefuzzでは、ブレークポイントがトリガーされたときにパラメータのデータを収集できます。ただし、パラメータによっては内部データ構造がコール間で変更されている可能性があるため、この方法に注意せずに従っても成功しない場合があります。上位のシードは現在のデータ構造とパラメータを破壊してはいけません。

## 5.2 サニタイズ

Muenchらは、ファジングによって発見されたメモリ破損の結果を、さまざまなカテゴリに分類しており、その中で観察可能なクラッシュとハングは追跡しやすいものです。遅延クラッシュや誤動作を見つけるのは難しいかもしれませんが、アドレスのサニタイズのような観測チャネルが追加されなければ、影響のないクラスを追跡するのは不可能です。衛生化手法では、ある一定の仮定が常に正しいかどうかをチェックし、そのような場合には目標を達成できなかったり、停止することさえある。C/C++で書かれたバイナリアプリケーションの多くの脆弱性は、メモリ破壊のバグであり、例えば、範囲外の読み出しや範囲外の書き込みによって引き起こされる。しかし、メモリ破壊のバグを引き起こす入力のすべてが、必ずしもすぐに入力をクラッシュさせるわけではありません。メモリの破損自体と、その後のプログラムクラッシュの原因となる破損したメモリの使用は、かなり離れていることが多いため、クラッシュした入力が与えられても、根本的なメモリのバグを実際に検出するのは難しい場合があります。Kernel Address Sanitizer(カサン)で構築したカーネルに対して,Unicorefuzzを評価した。Address Sanitizerは、コンパイル時のインストルメンテーションを使用して、たとえば、メモリアクセスの範囲外や、使用後のバグ[35]を検出します。KASANはシャドウメモリを利用して、カーネルメモリの特定のバイトが安全にアクセスできるかどうかを評価します。アドレスサニタイズでは、シャドウメモリ[35]へのメモリのマッピング方法により、比較的低いパフォーマンスコストが発生します。x64上のLinuxカーネルはあまり知られていない機能を利用しているので、KASANは最初にUnicornを壊したので、テスト・ケースにはパッチを当てる必要がありました。しかし、修正のおかげで、カーネルとモジュールがKASANで構築されていれば、KASANを十分に活用することができます。

## 5.3 再生可能性

有効な入力によって有効なクラッシュが発生しても、前提条件によっては、入力がすぐに再生できない場合があります。ただし、Unicorefuzzの場合は、すべてのクラッシュが100%再生可能です。syzkallerとは対照的に、Unicorefuzzはステートレスです。ステートレスなファジングは、同じテスト状態からターゲットを繰り返し呼び出します。この場合、これは複合状態ですが、すべての実行に対して同じ適切に初期化された状態です。入力状態はディスクに保存され、すべての入力は通常のAFLフォルダー構造で保存されます。このシナリオではUnicornに外部入力ソースがないため、1つの入力を再生すると常に同じクラッシュが発生します。

## 5.4 入力の制約

モジュールの内部にあり、入力が特定の構造に従っていることを想定している関数をファジングすることは、困難な場合があります。このような関数の例としては、4.2 .1で説明したように、sk_bufを必要とするパーサーがあります。この関数はカーネル内に組み込まれているため、関数が呼び出される前に入力をチェックする上に他のレイヤーがあります。関数がフューザによって直接トリガされた場合、状態が既に呼び出し元の関数内で制約されているため、制約されていない入力はクラッシュを引き起こす可能性があります。したがって、誤検出の可能性が高い。必要以上に入力を制限すると、ファザーはバグを見逃します。これは、手作業を必要とする重要な問題です。しかし、共通構造のどのフィールドをあいまいにすべきか、またすべきでないかの定義は共有されるかもしれません。

## 5.5 GS_BASEの奇妙なケース

セグメントレジスタは、元のセグメントレジスタでは使用されません。x86。GSレジスタには、CPU64固有のメモリにアクセスするために使用されるGS_BASEレジスタがあります。FS_BASEについても同じことが言えます。特に、内部のブックキーピング、スタック・カナリア、カーネル・アドレス・サニタイザ(カザン)はこのレジスタを使用するので、確実にサポートする必要があります。FS_BASEとGS_BASEは、モデル固有のレジスタ(C000_0100hおよびC000_0101h) [1] にマップされます。モデル固有レジスタ(MSR)は、もともと実験的なCPUレジスタで、後継モデルへの組み込みが保証されていませんでしたが、GSおよびFSベースが一般的に使用されています。とはいえ、この記事を書いている時点では、Unicorn Engineにはこれらのレジスターと直接対話するためのAPIがありません。代わりに、Unicorefuzzの場合は、マップされていないスペースをマップし、wrmsr命令のシェルコードを使用してベース・レジスターを作成する必要があります。

図6:カーネルのファジング方法の比較

## 5.6 最初のエミュレートされた操作

AFL-Unicornは最初の命令の実行後にforkserverを起動し、入力をロードします[41]。最初の命令が入力の一部と相互作用する場合、その命令の結果は、AFLからの適切に修正された入力に基づくのではなく、ターゲットからの元の入力に基づく。5.5で説明したセグメントレジスタの回避策は、シェルコードを実行する必要があるため、最初の命令を実行すると、偶然にX86_64でこの問題が緩和されます。

## 5.7 スケジューリング

AFL-Unicornは1つのプロセッサー・コアのみをエミュレートするため、1つのコアでスケジューラーをエミュレートすることはできますが、アクティブな2次コアを完了する必要があるファジング機能は動作しません。さらに、ターゲットのブレークポイントが発生した瞬間にロックがビジー状態になっているロック保護リソースに関数がアクセスする必要がある場合、ほかのコアはロックを解放しないため、関数は戻ることができません。

## 5.8 レースコンディション

タイトハーネスでシングルプロセッサエミュレータをセットアップすると、システムに他の負荷がかからないため、競合状態は検出されません。ロックや並行データ構造の競合を引き起こすCPUは他にはありませんし、個々の同期カーネル関数をファズしても、ユーザ空間プログラムが干渉することはありません。これによって、競合状態に関連するバグが見つかる可能性は大幅に減りますが、クラッシュは完全に再現可能になります。

## 5.9 他のファザーとの比較

ファザー性能の評価は難しい(6.1項参照)。ただし、特定のフィーチャーを比較できます。図6で概説されているように、エミュレートされていないファザーはエミュレーターのパフォーマンスに影響を与えませんが、最近のインテルプロセッサーでのみ動作するkAFLを除外して、ソースコードが利用できないバイナリーをターゲットにする能力がありません。適切な計装によって実行速度を向上させることの有用性にもかかわらず,もしユーザ空間エージェントが所望の目標機能を迅速にトリガできるならば,エミュレートしないファザーの生のスループットがより速い結果をもたらすことを認識しなければならない。ターゲットがユーザ空間エージェントによって容易にトリガされない場合,停止した1つの仮想マシンが任意の数の分散ファズ・テストを実行するのに十分であるため,Unicorefuzzのターゲット手法が優れている。

# 6 今後の作業

本論文の過程で,ファジングの分野に沿った新しい研究を提示したが,改善の余地は多い。

## 6.1 統一評価基準

何年も前から、研究者たちは互いのあいまいなパフォーマンスを適切に評価する方法を模索してきたが、成果は限られていた。2009年、クラークはすでに「ファジングの効果を正確に測定することは不可能である。なぜなら、唯一の実用的な測定基準であるコードカバレッジは、ファジングの一つの「寸法 」を測定するだけだからである。すなわち、実行された(到達可能)コードの量である。;各コード領域でターゲットに供給される入力値の範囲は測定されません。また、ターゲットの監視が理想的とは言えないこともよくあります。」と述べている [8]。

Dolan-Gavittらは、性能評価[14]に使用できるバイナリ解析のためのテストセットであるLAVA-Mを提案しているが、人工的なベンチマークはオーバーフィッティングにつながる可能性がある。しかし,Muらは,現実世界のセキュリティ脆弱性の再現性は複雑であるが,それらの努力はsyzkaller[29]のようなOSレベルファザーを部分的に評価するために使用できることを示した。特に、Trent Bunsonは良いFuzizingを見分ける方法を提案している [5]。Kleesらは、多数のラン[25]を用いて、異なるファザーの科学的評価を行っている。カーネルのバグに対しても、同様のテスト・スイートが必要です。

## 6.2 埋め込みファジング

すべての組み込みプロセッサーには、メモリー・ページをエクスポートして値を登録できるデバッグ・ポートがあるため、組み込みオペレーティング・システムのフリーズにセットアップを適応させることは可能です。動的なメモリマッピング要求に応答するには1つのターゲットで十分なため、複数のローカルマシンまたはリモートマシンにファジングを配布することも簡単です。

## 6.3 静的書き換え

コンパイル時、または実行時に動的にインストゥルメンテーションを追加する代わりに、静的分析を使用して追加することもできます。明らかな利点は、エミュレートされたインスツルメンテーションとは対照的に、スループットが向上することです。バイナリを変更(ユーザーランド-)して追加し、コードカバレッジをレポートし、それらをファズするプロジェクト[32]。

## 6.4 エミュレーション・パフォーマンスの向上

ファジングのスピードは様々な要因に左右されます。最近の進歩では、パスの検出速度だけでなく、軽量なハードウェア機能[3,444]による計測の実行速度も重視されています。現在の研究では、QEMUのブロック・チェーンを再度有効にすることで、AFLのQEMUモードのパフォーマンスを改善できることが示されています。これは、AFLのインスツルメンテーションに干渉するため無効になっています。マージされたブロックは、含まれているすべてのブロックの後でエミュレータにジャンプして戻ることはないので、ダイレクトジャンプのトレースを有効に無効にしています。著者はインストルメンテーション・コードを翻訳されたコードに注入することで、安全にブロック・チェーンを有効にすることができます。適切なキャッシングと組み合わせることで、メインラインQEMUモード [4] の3~4倍の速度向上が得られます。このパッチをAFL-Unicornに移植することで、コンパイラー支援のインスツルメンテーションとのパフォーマンスギャップを大幅に減らすことができます。しかし、これを機能させるためには、Unicorn自体にさらにパッチを当てる必要があります。コードの複雑さを減らし、フックを単純化するために、Unicornからブロック・チェーンが削除されたのです。十分な時間とエネルギーがあれば、Falkが提案したベクトル化エミュレーションをAFL Unicorn[19]に移植することも可能かもしれません。

## 6.5 トリアージ

ファジングは潜在的なバグを大量に出力します。設定によっては、誤検出(特にパラメータを拘束していない場合)の場合もあれば、すべて同じバグの場合もあります。ファズ・テストでは、単一のアプリケーション[28]でさえ、多数のクラッシュが発生します。手動で監査するにはクラッシュの数が多すぎる可能性があります [7]。手動分析者からの圧力を維持するために、ファズ設定は通常、何らかの方法でターゲットをトリアージします。したがって、データの量は構造化されていません。

関数をあいまいにすると、複数の入力がクラッシュする可能性がありますが、これは同じバグに起因する可能性があります。

すべてのクラッシュがセキュリティ上の脅威を引き起こすバグを指しているわけではありません。しかし、多くの場合、セキュリティ研究者も開発者も、セキュリティの脆弱性を露呈するバグに注目したいと考えている。ファジング管路は衝突[28]の過酷さを評価する必要がある。

また、クラッシュの原因となった根本的なバグも見つけたいと思います。DeMottらはバグの原因となっているソースコードの部分を特定しています。バグの原因となるコード行が主に入力のクラッシュによって実行されるという前提を使用して、コード行ごとに疑わしいスコアを計算します[13]。

クラッシュの重複除外と分析は、クラッシュ時のプログラムのスタックトレースの類似性を調べることによって実行できます。そのような方法は既に成功裡に採用されている[10]。同様に、クラッシュの原因となったバグの悪用可能性に従ってクラッシュを分類するヒューリスティックもあります[20]。ヒューリスティックは、クラッシュ入力を使用してプログラムを実行し、クラッシュ状態を検査することによって機能します。たとえば、NULLポインタの参照解除のバグはクラッシュを引き起こす可能性がありますが、現代のオペレーティングシステムでは悪用するのが難しいか、まったくできません。しかし、EIPレジスタがユーザ定義の入力で制御できる場合、このバグは非常に重大です[37]。Huang et al.とThanassis et al.はどちらも、クラッシュから自動的にエクスプロイトを生成することを提案しています。完全に自動化された攻撃が可能であるとマークできる場合、そのバグは重大です[2、22]。その他のバグでは、クラッシュを適切に評価することは簡単ではありません。

# 7 結論

技術の現状を評価した後,カーネルコードをエミュレートする新しいファザーの実装を提供した。筆者らは,ターゲット機能がユーザ空間に公開されていない場合でも,エミュレータを用いたカーネル機能のファジングが可能であり,実行可能であり,設定が比較的容易であることを示した。この方法では、あらゆる構文解析関数は、ハードウェアと対話しない限り、カバレッジ主導のフィードバックであいまいにすることができます。ただし、メモリーアクセスフックをサポートするエミュレータを使用すると、実行速度に明らかな影響があります。それでもスループットはAFLのQEMUユーザーモードの約46%だ。受け入れられます。

カーネル・コードの任意の時点でファズ・テストを開始できるという本質的な利点は、たとえバイナリー・ブロブ上であっても、アーキテクチャー全体にわたってファズ・テストを開始できるという点では、私たちの知る限り、カーネル・ファジングに対する他の手法とは比べものになりません。パラメータ構造体や配列に対して有効な領域を選択するために必要な手動のオーバーヘッドには欠点がありますが、この点を改善したいと考えています。カーネルのファジングでは、遠くから見ただけでもかなりのバグが見つかっています。syscall API。しかし、テストは内部関数を直接ファズすることはできません。Unicornは多数のプロセッサアーキテクチャをエミュレートでき,そのカーネルは,そのプラットフォーム用のプローブラッパーを書くことができれば,記述された技術で容易にファズすることができる。Unicorefuzzは、マルチプロセッサーの調整を必要としないすべての処理を実行できます。私たちはUnicorefuzzの実装をオープンソース化し、この概念をさらに改善したいと考えています。

# 謝辞

著者は、Vincent Ulitzsch氏、Fabian Freyer氏、Marius Muench氏に貴重な情報を提供してくれたことに感謝したい。

# 参考文献

- [1] AMD: AMD64 Architecture Programmer’s Manual Volume 2: System Programming. Advanced Micro Devices

- [2] Avgerinos, T., Cha, S.K., Rebert, A., Schwartz, E.J., Woo, M., Brumley, D.: Automatic exploit generation. Commun. ACM 57(2), 74.84 (Feb 2014). https://doi.org/10.1145/2560217.2560219

- [3] Bellard, F.: Qemu, a fast and portable dynamic translator. In: USENIX Annual Technical Conference, FREENIX Track. vol. 41, p. 46 (2005)

- [4] Biondo, A.: Improving afl’s qemu mode performance. 0x41414141 in ?? () (Sep 2018), https://abiondo.me/2018/09/21/improvingafl-qemu-mode

- [5] Brunson, T.: How to Spot Good Fuzzing Research (Oct 2018), https://blog.trailofbits.com/2018/10/05/how-to-spot-good-fuzzing-research , - [Online; accessed 11. Nov. 2018]

- [6] Butti, L., Tinnes, J.: Discovering and exploiting 802.11 wireless driver vulnerabilities. Journal in Computer Virology 4(1), 25.37 (2008)

- [7] Chen, Y., Groce, A., Zhang, C., Wong, W.K., Fern, X., Eide, E., Regehr, J.: Taming compiler fuzzers. In: ACM SIGPLAN Notices. vol. 48, pp. 197.208. ACM (2013)

- [8] Clarke, T.: Fuzzing for software vulnerability discovery. Department of Mathematic, Royal Holloway, University of London, Tech. Rep. RHUL-MA-2009-4 (2009)

- [9] Corina, J., Machiry, A., Salls, C., Shoshitaishvili, Y., Hao, S., Kruegel, C., Vigna, G.: DIFUZE: interface aware fuzzing for kernel drivers. In: Thuraisingham, B.M., Evans, D., Malkin, T., Xu, D. (eds.) Proceedings of the 2017 ACM SIGSAC Conference on Computer and Communications Security, CCS 2017, Dallas, TX, USA, October 30 - November 03, 2017. pp. 2123.2138. ACM (2017). https://doi.org/10.1145/3133956.3134069

- [10] Dang, Y., Wu, R., Zhang, H., Zhang, D., Nobel, P.: Rebucket: A method for clustering duplicate crash reports based on call stack similarity. In: 2012 34th International Conference on Software Engineering (ICSE). pp. 1084.1093 (Jun 2012). https://doi.org/10.1109/ICSE.2012.6227111

- [11] DeMott, J.: The evolving art of fuzzing. DEF CON 14 (2006)

- [12] DeMott, J., Enbody, R., Punch, W.F.: Revolutionizing the field of grey-box attack surface testing with evolutionary fuzzing. BlackHat and Defcon (2007)

- [13] DeMott, J.D., Enbody, R.J., Punch, W.F.: Towards an automatic exploit pipeline. In: Internet Technology and Secured Transactions (ICITST), 2011 International Conference for. pp. 323.329. IEEE (2011)

- [14] Dolan-Gavitt, B., Hulin, P., Kirda, E., Leek, T., Mambretti, A., Robertson, W., Ulrich, F., Whelan, R.: LAVA: Large-scale automated vulnerability addition. In: 2016 IEEE Symposium on Security and Privacy (SP). pp. 110. 121 (May 2016). https://doi.org/10.1109/SP.2016.15

- [15] Drewry, W., Ormandy, T.: Flayer: Exposing application internals. WOOT 7, 1.9 (2007)

- [16] Drewry,W.A., Ormandy, T.: Software testing using taint analysis and execution path alteration (Feb 2013), uS Patent 8,381,192

- [17] Drysdale, D.: Coverage-guided kernel fuzzing with syzkaller (2016), https://lwn.net/Articles/677764/

- [18] Duran, J.W., Ntafos, S.: A report on random testing. In: Proceedings of the 5th international conference on Software engineering. pp. 179.183. IEEE Press (1981)

- [19] Falk, B.: Vectorized Emulation: Hardware accelerated taint tracking at 2 trillion instructions per second (Oct 2018), https://gamozolabs.github.io/fuzzing/2018/10/14/vectorized_emulation.html , - [Online; accessed 11. Nov. 2018]

- [20] Foote, J.: Exploitable (2018), https://github.com/jfoote/exploitable , - [Online; accessed 2018-09-09]

- [21] Hertz, J., Newsham, T.: Project triforce, https://raw.githubusercontent.com/nccgroup/TriforceAFL/master/slides/ToorCon16_TriforceAFL.pdf

- [22] Huang, S.K., Huang, M.H., Huang, P.Y., Lai, C.W., Lu, H.L., Leong,W.M.: Crax: Software crash analysis for automatic exploit generation by modeling attacks as symbolic continuations. In: 2012 IEEE Sixth International Conference on Software Security and Reliability. pp. 78. 87 (Jun 2012). https://doi.org/10.1109/SERE.2012.20

- [23] Kernel.org: kcov: code coverage for fuzzing, https://www.kernel.org/doc/html/v4.17/devtools/kcov.html

- [24] Kerrisk, M.: Lca: The trinity fuzz tester (2013), https://lwn.net/Articles/536173/

- [25] Klees, G., Ruef, A., Cooper, B., Wei, S., Hicks, M.: Evaluating fuzz testing. CoRR abs/1808.09700 (2018), http://arxiv.org/abs/1808.09700

- [26] Lemieux, C., Sen, K.: Fairfuzz: Targeting rare branches to rapidly increase greybox fuzz testing coverage. CoRR abs/1709.07101 (2017), http://arxiv.org/abs/1709.07101

- [27] LLVMProject: libFuzzer . a library for coverage-guided fuzz testing. (Sep 2018), https://llvm.org/docs/LibFuzzer.html, - [Online; accessed 15. Sep. 2018]

- [28] Miller, C.: Crash analysis with bitblaze. Blackhat (2010)

- [29] Mu, D., Cuevas, A., Yang, L., Hu, H., Xing, X., Mao, B., Wang, G.: Understanding the reproducibility of crowd-reported security vulnerabilities. In: Enck, W., Felt, A.P. (eds.) 27th USENIX Security Symposium, USENIX Security 2018, Baltimore, MD, USA, August 15-17, 2018. pp. 919.936. USENIX Association (2018), https://www.usenix.org/conference/usenixsecurity18/presentation/mu

- [30] Muench, M., Francillon, A., Balzarotti, D.: Avatar2: A multi-target orchestration platform. In: BAR 2018, Workshop on Binary Analysis Research, colocated with NDSS Symposium, 18 February 2018, San Diego, USA. San Diego, UNITED STATES (Feb 2018), http://www.eurecom.fr/publication/5437

- [31] Ngyuen, A.Q., Dang, H.V.: Unicorn: Next generation cpu emulator framework, http://www.unicornengine.org/BHUSA2015-unicorn.pdf

- [32] Nikolich, A.: Afl fuzzing blackbox binaries (2015), https://groups.google.com/forum/#!topic/aflusers/HlSQdbOTlpg

- [33] Nossum, V., Casasnovas, Q.: Filesystem fuzzing with american fuzzy lop (2016), https://events.static.linuxfound.org/sites/events/files/slides/AFL%20filesystem%20fuzzing;%20Vault%202016_0.pdf

- [34] Schumilo, S., Aschermann, C., Gawlik, R., Schinzel, S., Holz, T.: kafl: Hardware-assisted feedback fuzzing for OS kernels. In: 26th USENIX Security Symposium (USENIX Security 17). pp. 167.182. USENIX Association, Vancouver, BC (2017)

- [35] Serebryany, K., Bruening, D., Potapenko, A., Vyukov, D.: Addresssanitizer: A fast address sanity checker. In: USENIX Annual Technical Conference. pp. 309.318 (2012)

- [36] Serebryany, K.: Oss-fuzz-google’s continuous fuzzing service for open source software. In: USENIX Security Symposium (2017)

- [37] Shoshitaishvili, Y., Wang, R., Salls, C., Stephens, N., Polino, M., Dutcher, A., Grosen, J., Feng, S., Hauser, C., Kruegel, C., Vigna, G.: SoK: (State of) The Art of War: Offensive Techniques in Binary Analysis. In: IEEE Symposium on Security and Privacy (2016)

- [38] Song, D., Hetzelt, F., Das, D., Spensky, C., Na, Y., Volckaert, S., Vigna, G., Kruegel, C., Seifert, J., Franz, M.: Periscope: An effective probing and fuzzing framework for the hardware-os boundary. In: 26th Annual Network and Distributed System Security Symposium, NDSS 2019, San Diego, California, USA, February 24-27, 2019. The Internet Society (2019), https://www.ndss-symposium.org/ndsspaper/periscope-an-effective-probing-andfuzzing-framework-for-the-hardware-osboundary/

- [39] Sthamer, H.H.: The automatic generation of software test data using genetic algorithms. Ph.D. thesis, University of Glamorgan (1995)

- [40] Thimmaraju, K.: Ve-2018-1000155: Denial of service, improper authentication and authorization, and covert channel in the openflow 1.0+ handshake (2018), https://www.openwall.com/lists/oss-security/2018/05/09/4

- [41] Voss, N.: afl-unicorn: Fuzzing arbitrary binary code (October 2017), https://hackernoon.com/afl-unicorn-fuzzing-arbitrary-binary-code-563ca28936bf

- [42] Vyukov, D.: Add fuzzing coverage support (2015), https://gcc.gnu.org/viewcvs/gcc?view=revision&revision=231296

- [43] Zalewski, M.: Technical "whitepaper" for AFL-fuzz (2016)

- [44] Zhang, G., Zhou, X., Luo, Y., Wu, X., Min, E.: Ptfuzz: Guided fuzzing with processor trace feedback. IEEE Access 6, 37302.37313 (2018). https://doi.org/10.1109/ACCESS.2018.2851237