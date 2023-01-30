# 00 イントロ
- オペレーティングシステム: Windows, MacOS, Linux, BSD, iOS, Androidなど
  - アプリケーションを動かすためのソフトウェア (基本ソフトウェア)
  - 存在理由:
    - 抽象化: プログラミングをより簡単にする
    - 効率性: 簡単なプログラムで高速に動作するようにする
    - リソースの保護・管理: リソース(CPU、メモリなど)の独占を防ぎ、公平に割り当てる
  - OSがないとどうなるかを考えることで、その意味を理解することが良い


OSがない場合:
- ユーザのプログラムがCPU(プロセッサ)上で直接動く
- 以下のようなことが非常に困難になる
  1. CPU(計算のための資源)を公平に分配すること
  2. メモリ(記憶のための資源)を安全に分配すること
  3. 外部ストレージを安全に分配すること
  4. 入出力

OSがないと. . .
- CPU (プロセッサ) 上に直接ユーザのプログラムが動く
- CPU (計算のための資源) を公平に分け合うことが困難
- メモリ (記憶のための資源) を安全に分け合うことが困難
- 外部ストレージを安全に分け合うことが困難
- 入出力が複雑

## OSの機能
- CPU を分け合う: **プロセス, スレッド**
- メモリを分け合う: **プロセス (アドレス空間), 仮想記憶**
- ストレージを分け合う: **ファイルシステム, システムコール**
- 入出力: **ファイルシステム, プロセス間通信**

### CPUの特権モード・ユーザモード
- CPU の動作モードに (少なくとも)2 種類ある
- ユーザモード
- 特権モード (スーパバイザモード)

両者の主な違い
1. 一部の命令が特権モードでしか実行できない (特権命令)
2. 一部のメモリ領域に「ユーザモードでアクセス不可」という属性をつけられる

OSのデータやプログラムがOS以外のプログラムには読み書き不能な仕組み
- OSが管理する領域を「ユーザモードでアクセス不可」
- OS以外はユーザモードで動作

下手に設計すれば, 結局誰でも特権モードで好きな命令を実行可能になる危険 → トラップ命令

### トラップ命令
- 以下の2つを行う
  1. ユーザモードから特権モードへ移行
  2. ある定められた番地へジャンプ
- x86の場合
  - int 0x80h命令
  - syscall命令
- ある定められた番地は「**割り込みベクタ**」と呼ばれるメモリ上の配列に登録されており、OSが起動時に設定する
- ユーザプログラムからOSへの「入り口」 = **システムコール**

### システムコール
- OS がユーザに対して提供している (根源的な) 機能
  - open, write, read, close, fork, exec, wait, exit, socket, send, recv, etc
  - 本当にシステムコールが呼び出されている瞬間は, トラップ命令で OS 内の命令に突入する瞬間

- OS内には多数の機能が存在するが、ユーザプログラムから「入り口」は1つだけ存在する
- 唯一の入り口から分岐してシステムコールがすべての機能ごとに実行されている
- ユーザプログラムが正規の入り口(システムコール)を通らずに特権モードに移行することはできない
- OS内部のプログラム(特権モードで実行される)をしっかり書けば、OSを保護することが可能


# 01 プロセス

- プログラム: 実行すべき命令が書かれているもの。Firefox, シェル (bash), ls, a.out などが含まれる。実体はファイルとして存在する。
- プロセス: プログラムが実行されているもの。メニューからアプリを起動したり、コマンドプロンプトからコマンドを実行するたびに作られる。

例:
- プログラム ≈ マニュアル
- プロセス ≈ マニュアルに従って働いている人

- プロセスの役割
  - CPU を分け合うための抽象化
  - ユーザがプロセスを作ることができる
  - 各プロセスは全力で動かせば良い (他のプロセスに CPU を譲る必要はない)
  - OS がプロセスに CPU を与えたり奪ったりすることができる
  - メモリを分け合うための抽象化 (アドレス空間)
  - 他のプロセスのメモリにアクセスすることはできない, 壊すこともできない

```shell
$ ps auxww
$ ps auxww | grep ssh # ssh 走ってるか?
$ ps auxww | grep tau # ユーザtau のプロセス
$ pgrep -f ssh # \simeq ps + grep
$ pgrep -f tau
```

- プロセスID (PID)
  - 存在する全てのプロセスに付けられた一意な識別子
  - Linux では通常、4194304 までの整数
  - 外から強制終了する場合などに必要


- manコマンド（略）

## Unix: プロセス関連のシステムコール
- `fork`: プロセスを作る (コピーする)
- `execve`: 現在のプロセスで指定のプログラムを実行する
  - 変種: exec{v,l}p?e? (引数の渡し方, 微妙な意味の違い)
  - 総称して exec と呼ぶ (exec という名前の関数はない)
- `exit`: 現在のプロセスを終了する
- `waitpid`: 子プロセスの終了待ち + 処理
  - 変種: wait, wait3, wait4

### 処理の流れ
- 子プロセスの生成
  1. `fork` 実行によりプロセスが複製される
  2. 親プロセスと子プロセスの両方が `fork` の続きを実行する

- 子プロセスの終了
  1. 子プロセスが終了する (`exit` を呼ぶ, main 関数が return するなど)

- 処理
  1. 親プロセスが処理をするまで (`wait`, `waitpid` などを呼ぶ)
  2. 子プロセスは「ゾンビ (プロセス番号だけが存在する) 状態」
  3. 親プロセスが処理を終えるとすべてがなくなる

### `fork`
- 呼び出したプロセスを複製
- fork() の続きが 2 プロセス (親と子) で実行される
- 親と子で返り値だけが違う
  - 親: 子プロセスのプロセス番号
  - 子: 0

[Man page of FORK](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/fork.2.html)
```C
#include <sys/types.h>
#include <unistd.h>

pid_t pid = fork();
if (pid == -1) {
失敗 (子プロセスは作られていない)
} else if (pid == 0) { /* child */
子プロセス
} else { /* child */
親プロセス
}
```

### `exit`
- exit を呼んだプロセスを, 指定した終了ステータス(exit status:0~255 の整数) で終了させる
- 当然ながら exit 呼び出し以降は実行されない
- main 関数が終了した場合も同じ効果 (従って main 関数の最後にわざわざ呼ばないのが普通)
  - main の返り値 (return value) が exit status
- exit status は親プロセスが取得可能

### `waitpid`
- 基本は子プロセスの終了待ちと処理(ゾンビ状態を解消、プロセス番号の回収)
- 子プロセスの終了待ち方法についての指定が可能 (特定のプロセス、全て、など)
- 終了を待つか待たないかも指定可能

[Man page of WAIT](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/wait.2.html)
```C
#include <sys/types.h>
#include <sys/wait.h>

int ws; // wstatus の意味
pid_t pid = -1; /* -1 : どの子プロセスでも... */
int options = 0; /* 0 : 終了するまで待つ */
pid_t cid = waitpid(pid, &ws, options);
if (cid == -1) {
失敗;
} else {
... ws に, 子プロセスcid に何が起きたかの情報 ...
}
```

## ゾンビ (defunct)
  - プロセス C がゾンビ (defunct) の意味は、C が終了しているが、その親が (waitpid などで)C の終了を確認していない状態
  - 本質的には、C のプロセス番号を再利用できない状態
  - waitpid が「C が終了した」と親に知らせるまでは、C のプロセス番号を他のプロセスに再利用すると、プロセス番号からプロセスを一意に特定できなくなるため
  - waitpid ≈ お葬式; 子プロセスに「成仏」「輪廻転生」してもらう

- Q. 子が終了する前に親が waitpid 等を呼んだら？
  - A. 子が終了するまで return しない（wait の意味通り）

- Q. 子が終了する前に親が終了できる？
  - A. できる

- Q. その場合, 誰がその子の葬式をする (その子はゾンビ状態のまま)？
  - A. 先祖のプロセス(init)が子の親となる

[fork_wait.c](https://github.com/maronuu/operating_system_exercise/blob/main/process/fork_wait.c)
```C:fork_wait.c
#include <err.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

int main() {
  pid_t pid = fork();
  if (pid == -1) {
    err(1, "fork");
  } else if (pid == 0) {
    // child
    for (int i = 0; i < 5; ++i) {
      printf("This is child (%d): %d times\n", getpid(), i);
      fflush(stdout);
      usleep(100 * 1000);
    }
    return 123;  // my status
  } else {
    int ws;
    printf("parent: wait for child (pid=%d) to finish\n", pid);
    pid_t cid = waitpid(pid, &ws, 0);
    if (WIFEXITED(ws)) {
      printf("exited, status=%d\n", WEXITSTATUS(ws));
      fflush(stdout);
    } else if (WIFSIGNALED(ws)) {
      printf("killed by signal %d\n", WTERMSIG(ws));
      fflush(stdout);
    }
  }
  return 0;
}
```

## exec - 子プロセスの生成
1. fork ～ プロセスが複製される. 親と子が両方,fork の続きを実行
2. 子プロセスが exec を実行

- 現プロセスで, 指定したプログラムを実行する
- 注意事項:
  - exec の呼び出した後の部分(上記のもの以降)は実行されない
  - 呼び出したプロセスは現在の情報をすべて忘れて、指定されたプログラムを実行するだけの新しいプロセスとして生まれ変わる
  - **exec は子プロセスを作らない**

execの変種くんたち：exec{v,l}p?e?
- execv, execve, execvp, execvpe, execl, execle, execlp
- execve だけがシステムコール, 残りはそれの亜種
- v と l : 引数の渡し方 (v : 配列; l : 引数のリスト)
- p : 環境変数 PATH を参照してコマンドを検索する
- e : 子プロセスの環境変数を指定する (ない場合は親を引き継ぐ)

[Man page of EXEC](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/exec.3.html)
```C
#include <unistd.h>

execv("ls", argv); // NG
execv("/bin/ls", argv); // OK
execvp("ls", argv); // OK
```

[fork_exec.c](https://github.com/maronuu/operating_system_exercise/blob/main/process/fork_exec.c)
```C:fork_exec.c
#include <err.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

extern char **environ;

int main() {
  pid_t pid = fork();
  if (pid == -1) {
    err(1, "fork");
  } else if (pid == 0) {
    char *const argv[] = {"ls", "-l", 0};
    execvp(argv[0], argv);
    err(1, "execv");
  } else {
    int ws;
    pid_t cid = waitpid(pid, &ws, 0);
    if (WIFEXITED(ws)) {
      printf("exited, status=%d\n", WEXITSTATUS(ws));
      fflush(stdout);
    } else if (WIFSIGNALED(ws)) {
      printf("killed by signal %d\n", WTERMSIG(ws));
      fflush(stdout);
    }
  }
  return 0;
}
```



# 02 スレッド
- スレッドはCPUを分け合うための抽象化
- スレッドがなければCPUは割り当てられず計算ができない
- CPUコアをN個使用したい場合はN個以上のスレッドが必要
- 1つのプロセスには複数のスレッドが存在することがあり、プロセスを作ると必ず1つのスレッドができる
  - C言語ではmain関数を実行するスレッドがある

**プロセス = アドレス空間 (箱) + 1 つ以上のスレッド **
- CPU ⊃ 物理コア ⊃ 仮想コア
  - スレッドに「CPU を割り当てる」と言うが実際に割り当てているのは, 1 つの仮想コア

スレッドを観察するコマンド
```shell
# Unix CUI
$ ps auxmww
$ top -H
# Linux
/proc/pid/task/tid
```

Unix: スレッド関連のAPI
- POSIX スレッド (または単に Pthreads) (Portable Operating System Interface (Unix系 OS 共通の API 仕様) )
- pthread_create : スレッドを作る
- pthread_exit : 現スレッドを終了
- pthread_join : スレッドの終了を待つ

スレッドの生成～終了～処理
1. pthread_create() により、子スレッドが生成される。
2. 子スレッドが終了する (pthread_exit() を呼ぶ、または子スレッド生成時に指定した関数が終了)。
3. どれかのスレッドが pthread_join() を呼ぶ

#### pthread_create()
- `pthread create(tid, attr, f, arg)`
  - f(arg) を実行するスレッド (子スレッド) を作る
  - pthread create 呼び出し以降と, f(arg) が並行して実行される
  - f は void* を受け取り void* を返す関数
  - スレッドの開始関数と呼ぶ
  - 諸々の属性を attr で指定
  - pthread_create の return 後, 子スレッドの ID が*tid に書き込まれる

#### pthread_exit()
- `pthread exit(p)`
  - pthread exit を呼んだプロセスを, 指定した終了ステータス p で終了させる
  - p : ポインタ (void * )
  - スレッドの開始関数が終了した場合も同じ効果
  - 開始関数の返り値 (return value) が終了ステータス
  - p は `pthread_join` で取得可能

#### pthread_join()
- `pthread join(tid, q)`
  - 子スレッド tid の終了を待つ
  - tid の終了ステータスが* q に返される
  - waitpid のスレッド版だが細かい違い
    - `pthread join(tid, q)` を呼ぶのは, tid の親スレッドである必要はない (同じプロセス中のどのスレッドでもよい)

### プロセス vs. スレッド
- 同じプロセス内のスレッドは「アドレス空間」を共有する
  - ≈ プログラム内のデータを共有する
  - ≈ あるスレッドが書き込んだ値は他のスレッドも自動的に観測する

- 複数のプロセスはそれぞれ独立した「アドレス空間」を持つ
  - ≈ プログラム内のデータは共有されない
  - ≈ あるプロセスが書き込んだ値が他のプロセスに観測されることはない
    ![clipboard.png](assets/img.png)

[Man page of PTHREADS](https://linuxjm.osdn.jp/html/LDP_man-pages/man7/pthreads.7.html)

[measure_pthread.c](https://github.com/maronuu/operating_system_exercise/blob/main/thread/measure_pthread.c)
```C:measure_pthread.c

#include <err.h>
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

long cur_time() {
  struct timespec ts[1];
  clock_gettime(CLOCK_REALTIME, ts);
  return ts->tv_sec * 1000000000L + ts->tv_nsec;
}

void* do_nothing(void* arg) {
    pthread_t tid = pthread_self();
    printf("thread[%lu]: do_nothing called\n", tid);
    return arg;
}

int main(int argc, char** argv) {
  int n = (argc > 1 ? atoi(argv[1]) : 5);
  long t0 = cur_time();

  /* ここにプログラムを書く */
  pthread_t threads[n];
  void *ret;
  for (int i = 0; i < n; ++i) {
    if (pthread_create(&threads[i], NULL, do_nothing, NULL)) {
      err(1, "pthread_create");
    }
    if (pthread_join(threads[i], ret)) {
      err(1, "pthread_join");
    }
  }

  long t1 = cur_time();
  long dt = t1 - t0;
  printf("%ld nsec to pthrea_create-and-join %d threads (%ld nsec/thread)\n",
         dt, n, dt / n);
  return 0;
}
```


# 03 スケジューリング
- OS の重要な仕事 : スレッドに CPU(仮想コア) を割り当てる
  - 基本 : かわりばんこ (round robin)
- 文字通り存在するすべてのスレッド (> 1,000個は当たり前) に均等時間ずつではない

スレッドの状態：以下の 3 つの状態を区別している
- 実行中 (running)
- 実行可能 (runnable, active)
- 中断中 (blocked, suspended)

![clipboard.png](assets/img_1.png)

- スレッドが中断する時の例:
1. OS が, 現在実行中のスレッドがこれ以上実行不能と判断する時
2. read, recv などブロッキング I/O で読むデータがない
3. waitpid, pthread join などで, 子プロセス・スレッドが終了していない
4. pthread mutex lock, pthread cond wait など同期 API で同期が成立していない
5. sleep, usleep, nanosleep など休眠 API
6. ページフォルトで I/O が発生
- OS は, 現在実行中のスレッドから, 別のスレッドに切り替える (**コンテクストスイッチ**)

ほとんどのスレッドは中断している
- ps auxww, top コマンドの STAT 欄
  - R 実行中または実行可
  - S 中断中 (割り込み可能)
  - D 中断中 (割り込み不可能)
  - その他 . . .
- S と D の違いはあまり気にしなくて良い. D はそれほど
  見かけない
- Unix の load average (負荷平均) = R か D 状態にあるスレッド数 ≈ R 状態にあるスレッド数 (の最近何分かの平均)

## タスクキュー、ランキュー（FIFOとは限らない）
- 実行中・実行可状態のスレッドを維持する
- OS はスレッドを切り替える際, ランキュー中から, **ある基準**に従って次のスレッドを選んで実行する

### ランキューの動き
- ランキューには実行可・実行中のスレッドが入っている
- 実行中のスレッドが中断 → ランキューから外れる
- ランキューから次のスレッドが選ばれて実行される
  - 中断中のスレッドが再開 → ランキューに挿入される
  - タイマ割り込み → 実行中のスレッドが十分な時間を消費していたら, **preemption**
- ランキューから次のスレッドが選ばれて実行される

### Preemption (横取り)
- スレッドが自発的に OS に制御を渡さない (なにもシステムコールを呼ばずに走り続けている) 状態でも, 強制的に制御を奪うこと (≡ preemption)
- preemption を行うスケジューラ ≡ preemptive なスケジューラ
- 今日の OS のスケジューラは事実上すべてが preemptive
- preemption を実現する仕組み: **タイマ割り込み**

### タイマ割り込み
仮想コアは以下を繰り返す
```C
while (1) {
  PC の指すアドレスから命令を読む
  命令を実行する (一部のレジスタが書き換わる, PC 含め)
  if (割り込みが来た) {
    PC := 割り込みベクタ [割り込み番号];
  }
}
PCはプログラムカウンタレジスタのこと
```

## スレッド選択
- OS はスレッドを切り替える際, ランキュー中から, **ある基準**に従って次のスレッドを選んで実行する
- **ある基準**の例
  - **Round Robin** (純粋なかわりばんこ)
    - ランキューが純粋な FIFO
    - 中断から回復したらキューの末尾に入る
    - 実行中のスレッドの time quantum が expire したら末尾に入る
    - 次のスレッドを選ぶ際はキューの先頭が選ばれる
    - 問題
      - 公平性: 「中断していた ⇐⇒ CPU を使っていなかった」スレッドも「preempty された ⇐⇒ ずっと CPUを使っていた」スレッドも同じ扱い
      - 対話的なプログラムの応答性: 実行可のスレッドが多い(load average が高い) と, それに比例して, 「応答時間 = 中断状態から再開してから実行されるまでの時間」が長くなる
  - **Linux Completely Fair Scheduler (CFS)**
    - Linux 2.6.23 以降のデフォルトスケジューラ
    - 各スレッドが「**消費した合計 CPU 時間 (→ vruntime)**」を管理
    - スレッド切り替え (実行中スレッドが中断した, 中断中スレッドが復帰した, タイマ割り込みがおきた, など) 時に, 実行可能スレッド中で vruntime が最小のスレッドを次に実行する
      - 注: ランキューを, vruntime の小さい順にスレッドが並ぶ優先度キューとすれば実現可能
    - 長い目で見て, 各スレッドへの CPU 割当時間を均等にすることができる → 公平
    - しばらく中断していた (CPU を使っていなかった) スレッドが再開した時, その間実行されていたスレッドよりも vruntime が小さいことが期待される→ 対話的スレッドの応答性が高い

### vruntime
1. 生成時は親の vruntime を継承. 親スレッド A が子スレッド B を生成

$$
B.vruntime = A.vruntime
$$

2. タイマ割り込み, スレッド中断時に, 実行中だったスレッド T の vruntime を加算.

$$
T.vruntime += T が今回消費した CPU 時間
$$

3. スレッド T が中断から復帰する時は, T の vruntime が他の実行可・中スレッドよりも極端に小さくならないことを保証

$$
T.vruntime = \max(T.vruntime, \min_{t:実行可}
t.vruntime−20 ms)
$$


# 並行処理と同期

## 共有メモリと競合状態
### 例
同一プロセス内のスレッドはメモリを共有している。

競合する例: 2 child threads

片方のg += 100 の間にもう一方のスレッドによりgが書き換えられていなければ成功する。
なぜ？: CPUが一度に行えるメモリに対する操作はread, writeどちらかだけである。
```c
int g = 0; // global

void *f(void *arg) {
    g += 100;
    return 0;
}

int main() {
    int err;
    g = 200;
    /* スレッドを作る */
    pthread_t child_thread_id[2];
    for (int i = 0; i < 2; i++)
        pthread_create(&child_thread_id[i], 0, f, 0);
    /* 終了待ち */
    for (int i = 0; i < 2; i++) {
        void * ret = 0;
        pthread_join(child_thread_id[i], &ret);
    }
    printf("g = %d\n", g);
    return 0;
}
```

### 用語の定義
定義: 競合状態とは以下のような状態
- 複数スレッドが
    - (a) 同じ場所を
    - (b) 並行してアクセスしていて
- うち少なくとも１つは書き込みである

定義: 際どい領域 Critical Section
- コード上で競合状態が発生している領域

### 競合状態の分類
- 不可分性(Atomicity)の崩れ
    - 一度にできない一連の操作の途中に他のUpdate処理が挟まるために、意図しない動作になる
- 順序・依存関係(Dependency)の崩れ
    - 複数スレッド間で、Read/Writeの順序保証をする必要がある処理なのに、その保証ができない場合にダメになる。

## 同期 (Synchronization)
### 排他制御 `mutex`
1人しか入れない個室トイレ
- `lock`: トイレが空いてれば入って鍵をかける。空いてなければ空くまで待つ
- `unlock`: 鍵を空けてトイレを空ける。
Atomicに行いたい処理をlock/unlockで挟む

#### API
API: [man page](https://linuxjm.osdn.jp/html/LDP_man-pages/man7/pthreads.7.html)
```c
#include <pthread.h>

pthread_mutex_t m; /* 排他制御オブジェクト */
pthread_mutex_init(&m, attr);
pthread_mutex_destroy(&m);
pthread_mutex_lock(&m); /* lock */
pthread_mutex_try_lock(&m);
pthread_mutex_unlock(&m); /* unlock */
```

![](./assets/concurrent_mutex_api.png)

#### 例: スレッドセーフなIncrement Counter
```c
typedef struct {
  long n;
  pthread_mutex_t mutex;
} counter_t;

void counter_init(counter_t* c) {
  c->n = 0;
  pthread_mutex_init(&c->mutex, NULL);
  return;
}

long counter_inc(counter_t* c) {
  long ret;
  pthread_mutex_lock(&c->mutex);
  ret = c->n;
  (c->n)++;
  pthread_mutex_unlock(&c->mutex);
  return ret;
}

long counter_get(counter_t* c) {
  long ret;
  pthread_mutex_lock(&c->mutex);
  ret = c->n;
  pthread_mutex_unlock(&c->mutex);
  return ret;
}
```

### バリア同期 `barrier`
```c
#include <pthread.h>
pthread_barrier_t b; /* バリアオブジェクト */
pthread_barrier_init(&b, attr, count);
/* count=参加するスレッド数 */
pthread_barrier_destroy(&b);
pthread_barrier_wait(&b);
/* 同期点に到達; 他のスレッドを待つ */
```

![](./assets/barrier_api.png)


### 条件変数 `cond`
ある条件が整うまで待つ、待っているスレッドを叩き起こす汎用機構。布団。

```c
#include <pthread.h>
pthread_cond_t c;
pthread_mutex_t m;

pthread_cond_init(&c, attr);
pthread_cond_destroy(&c);
pthread_cond_wait(&c, &m); /* 寝る */
pthread_cond_broadcast(&c); /* 全員起こす */
pthread_cond_signal(&c); /* 誰か一人起こす */
```

#### `pthread_cond_wait(&c, &m)`の動作
- `pthread_cond_wait`を呼び出した時点でスレッドはmをLockしている。
    - mをunlockする
    - cの上で寝る (中断・ブロック)
を**不可分に**行う。
    - returnする際にはまたmをLockしていることが保証
![](./assets/condvar.png)

#### 例: 飽和付きカウンタ (os07)
```c
/* 注: このプログラムはOMP_NUM_THREADSを使わずにコマンドラインで受け取った引数でスレッド数を決めている(#pragma omp parallel num_threads(...)) */

#include <assert.h>
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <omp.h>

/* 飽和カウンタ */
typedef struct {
  long x;
  long capacity;
  pthread_mutex_t m[1];
  pthread_cond_t c[1];
  pthread_cond_t d[1];
} scounter_t;

/* 初期化(値を0にする) */
void scounter_init(scounter_t * s, long capacity) {
  s->x = 0;
  s->capacity = capacity;
  if (pthread_mutex_init(s->m, 0)) {
    die("pthread_mutex_init");
  }
  if (pthread_cond_init(s->c, 0)) {
    die("pthread_cond_init");
  }
  if (pthread_cond_init(s->d, 0)) {
    die("pthread_cond_init");
  }
}

/* +1 ただしcapacityに達していたら待つ */
long scounter_inc(scounter_t * s) {
  pthread_mutex_lock(s->m);
  long x = s->x;
  // capacityに達していたらwait
  while (x >= s->capacity) {
    assert(x == s->capacity);
    pthread_cond_wait(s->c, s->m);
    x = s->x;
  }
  // 飽和が解消されたので
  s->x = x + 1; // increment
  // この操作によってEmptyが解消されたら、下限condに寝てるThreadを起こす
  if (x <= 0) {
    assert(x == 0);
    pthread_cond_broadcast(s->d);
  }
  pthread_mutex_unlock(s->m);
  assert(x < s->capacity);
  return x;
}

/* -1 */
long scounter_dec(scounter_t * s) {
  pthread_mutex_lock(s->m);
  long x = s->x;
  // emptyだったらwait
  while (x <= 0) {
    assert(x == 0);
    pthread_cond_wait(s->d, s->m);
    x = s->x;
  }
  // emptyが解消されたので
  s->x = x - 1; // decrement
  // この操作によって飽和が解消されたら、上限condで寝てるThreadを起こす
  if (x >= s->capacity) {
    assert(x == s->capacity);
    pthread_cond_broadcast(s->c);
  }
  pthread_mutex_unlock(s->m);
  return x;
}

/* 現在の値を返す */
long scounter_get(scounter_t * s) {
  return s->x;
}
```

#### 条件変数使い方テンプレ
ある条件Cが成り立つまで待って、Aをする、という場合
```c
pthread_mutex_lock(&m);
while (1) {
    C = ...; /* 条件評価 */
    if (C) break;
    pthread_cond_wait(&c, &m);
}
/* C が成り立っているのでここで何かをする */
A
/* 寝ている誰かを起こせそうなら起こす */
...
pthread_mutex_unlock(&m);
```

#### pthread_cond_waitがmutexも引数にとる理由
- 条件判定時、そのThreadはmutexをlockしているはず。
- 自分が処理をブロックしてwaitする際は、他のThreadにMutexを開けわたさないといけない。
- mのunlockと自分の休眠を不可分に行う必要がある(Lost wake up問題への対処)

## 不可分更新命令
1変数に対するRead/Writeを不可分に行ういくつかの命令がある
- adhocな命令
    - test&set p (0だったら1にする)
    ```c
    if (*p == 0) {
        *p = 1; return 1;
    } else {
        return 0;
    }
    ```
    - fetch&add p, x
    ```c
    *p = *p + x;
    ```
    - swap p, r
    ```c
    x = *p;
    *p = r;
    r = x;
    ```
- 汎用命令
    - compare&swap: 自分が読んだ値が書きかわっていないことを確かめながら不可分に書き込む
    ```c
    x = *p;
    if (x == r) {
        *p = s;
        s = x;
    }
    ```
    - `bool __sync_bool_compare_and_swap(type *p, type r, type s)`
        - swapが起きたかどうかを返す
    - `type __sync_val_compare_and_swap(type *p, type r, type s)`
        - `*p`に入っていた値を返す

### CASテンプレ
`*p = f(*p)`という更新を不可分に
```c
while (1) {
    x = *p;
    y = f(x);
    if (__sync_bool_compare_and_swap(p, x, y)) break;
}
```

## 同期の実装
- ナイーブな発想
```c
// NG例
int lock(mutex_t *m) {
    while (1) {
        if (m->locked == 0) { // 評価(競合)
            m->locked = 1; // 更新(競合)
            break;
        } else {
            BLOCK
        }
    }
}
```

- 不可分更新命令を使う & `futex`を使う
```c
// OK
int lock(mutex_t * m) {
    while (!test_and_set(&m->locked)) {
        /* m->locked == 1 だったらブロック */
        futex(&m->locked, FUTEX WAIT, 1, 0, 0, 0);
    }
}
int unlock(mutex_t * m) {
    m->locked = 0;
    futex(&m->locked, FUTEX_WAKE, 1, 0, 0, 0);
}
```

### `futex`
`if(u==v) then block`を不可分に実行
```c
futex(&u, FUTEX_WAIT, v, 0, 0, 0);
```
参照: [man futex](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/futex.2.html)

### 休眠待機　vs 繁忙待機
- 休眠待機 (blocking wait)
    - `futex`, `cond_wait`などで待つ
    - OSに「CPUを割り当てなくて良い」とわかるように待つ
- 繁忙待機
    - `while(!condition) { 何もしない }`
    - OSに「CPUを割り当てなくて良い」ことがわからないように待つ(実行可能であり続ける)
例はスライド04 p60-65あたり参照

![](./assets/busy_wait.png)

- 基本は使わないが、全てのスレッドが同時に実行されている想定のときは高速化できるかも。多数のスレッドが中断・復帰するときには有効。

### スピンロック
mutexの繁忙待機バージョン
`pthread_spinlock_*`

### futex自体の実装(advanced)
スライド参照。。

### デッドロック
同期のための待機状態が循環し、どのスレッド・プロセスも永遠にブロックしたままになる状態
Example
- 二つ以上の排他制御
![](./assets/deadlock1.png)
- 送受信バッファ
![](./assets/deadlock2.png)

回避方法
- 問題の根源は「誰かを待たせながら誰かを待つ」こと。これがダメ。
- mutexを一つだけにする (giant lock)
- 1つのスレッドは2つのMutexを同時にLockしない。
- 全てのmutexに順序をつけ、全てのスレッドはその全順序の順でしかLockをしない
- 不可分更新をするのに排他制御を使わない
    - 不可分更新
    - トランザクショナルメモリ


# 05 メモリ管理
## 論理アドレス空間

**プロセスごとに**論理アドレス（↔物理アドレス）空間を作る

論理アドレスの範囲は物理メモリの量によらず、OSによって決まり、論理アドレスと物理アドレスの変換はCPUが行う。**対応する物理アドレスが存在するとは限らない**。

## メモリ管理ユニット(MMU)
CPU内のハードウェアでメモリアクセスに介在する。論理アドレスに対して、
1. アドレス権の検査
2. 対応する物理アドレスの有無の検査
3. 存在していればアクセス
を行う。

MMUによって
1. プロセス間でメモリの分離
2. カーネル(OS)の保護
3. 物理メモリ量を超えたメモリ割り当て
4. *要求時ページング*
   - 物理メモリを確保せずに論理アドレスを割り当てる（高速）

が行える。

## アドレス交換のやり方
論理アドレス → 物理アドレス/アクセス許可への写像を作る
すべてのアドレスに対応する情報をもたせるのは非現実的なので、ページングを行う。

$$ 2^{48}\times 8 \times Process = 2PB \times Process $$

最下位Abit(大体12bit or 13bit)**以外共通**の2^A個のアドレスからなる領域をページと言う。ページのサイズが大きいほど、ページテーブルのサイズが小さくなるので、メモリの節約になる。ページのサイズが小さいほど、ページテーブルのサイズが大きくなるので、ページテーブルのアクセスが遅くなる。

アクセス許可情報や、物理アドレス（オフセット以外）は、ページごとに一つもたせる。
これで必要なものは、以下の写像になる。

論理ページ → アドレス許可/物理ページ番号

この論理ページto物理ページ番号をページテーブルという形で実現する。ただし、論理ページ番号をただの配列とすると2^36個の要素をもつ配列が必要になるので、ページテーブルは多段ページテーブルで実現される。例えば

2^9 -> 次の2^9 -> 次の 2^9 -> 次の2^9

というふうに4段にする。最悪の場合、512GB必要なのは変わらないが、ほとんどのプロセスは論理アドレス空間のごく一部しか使わないので、下位の表は殆どの場合不要になり、多段ページテーブルの場合、効率が良い。（不要なページテーブルは作成しないので）

### ページテーブルエントリ
ページテーブルのエントリは、以下のようになる。
- P : 1ならページが存在する
  - W : 1なら書き込み可能
  - U : 1ならユーザープロセスからアクセス可能
  - A : 1ならアクセスされた
  - D : 1なら書き込みされた
  - PFN : 物理ページ番号

![](assets/2023-01-30-14-59-07.png)

### TLB(Translation Lookaside Buffer)
毎回論理アドレスの変換のために4回メモリアクセスはだるい
→TLB（CPU内のキャッシュ）内に一部の写像を保存しておく（1024個程度 36bit x2 x 1024 = 9KiB程度？）

## UNIXのメモリ割当API
- `brk(l)`
  - データセグメントの終わりのアドレスをlにする。`(x < l)`のアドレスxが利用可能になる(割り当てられる)
- `sbrk(sz)`
  - プロセスのデータセグメントの境界をszだけ伸ばす
  - `sbrk(0)`で現在のデータセグメントの終わりのアドレスを返す

- `mmap, mremap, munmap` 
  - めっちゃ大事なやつら(後述)

- `mprotect(a, sz, prot)`
  - メモリ領域aからszバイトのメモリ領域のアクセス権をprotに変更する

メモリを割り当てることによって、**論理アドレスの範囲がアクセス可能になる**。実際にはアドレス空間表に割り当て済みであることを記述するだけで、**アクセス発生時にメモリが割り当てられる**。

## ページアウト/ページイン
物理メモリが足りなくなったときに、ページアウトを行う。ページアウトは、ページテーブルの物理ページを不在とし、次にアクセスする際にページフォルトが起こるようにしておく。ページアウトされたページは、ページインされるまで、物理メモリには存在しない。スライド05の図を見る。

## 資源使用量やメモリの割当状況を知るためのAPI
- `getrusage`
  - プロセスの資源使用量を返す
- `setrlimit`
  - プロセスの資源使用量の上限を設定する
- `prlimit`
  - プロセスの資源使用量の上限を取得する
- `mincore`
  - メモリ領域のページの存在状況を返す
  - `mincore(addr, len, vec)`で、addrからlenバイトのメモリ領域のページの存在状況をvecに書き込む

## cgrouops
- `cgroup`は、Linuxカーネルの機能で、プロセスのグループを作成し、グループ内のプロセスに対して、資源使用量の上限を設定することができる。

`/sys/fs/cgroup/hoge/`に、`cpu.cfs_quota_us`と`cpu.cfs_period_us`を書き込むことで、CPU使用量の上限を設定できる。`cpu.cfs_quota_us`には、CPU使用量の上限をマイクロ秒単位で書き込む。`cpu.cfs_period_us`には、CPU使用量の上限を計算する周期をマイクロ秒単位で書き込む。`cpu.cfs_quota_us`には、`cpu.cfs_period_us`の周期で、`cpu.cfs_quota_us`マイクロ秒までCPUを使用できる。`cpu.cfs_quota_us`には、`cpu.cfs_period_us`の周期で、`cpu.cfs_quota_us`マイクロ秒までCPUを使用できる。`memory.high`でメモリの使用量の上限を設定できる。

## ページ置換アルゴリズム
物理メモリが足りなくなったときに、どのページをページアウトするかを決めるアルゴリズム。
ページ置換が頻繁に起こる状態をスラッシングという。

オフライン問題（将来のアクセス系列をすべて知っている）に対する最適アルゴリズムは、 $a_i \notin R_i$のときに、常駐ページのうち、次にアクセスされるまでの時間が最も長いページを置換するというもの。 

オンライン問題に対しては、どんなアルゴリズムにしようと最悪のケース（ページフォルト率1）は、避けられない。仕方ないので、**最近使われたものはまたすぐ使われる**という予想をする。（LRU, FIFO, LRUの近似, エイジング）

## LRU
LRUの根拠として、空間局所性及び時間局所性があげられる。未来のアクセスが過去のアクセスの反転であるとして最適アルゴリズムを適用しているのと同じである。

LRUはこんな配列を用意して、新しいのが出たら後ろに追加し、古いのを追い出す。既存のなら、後ろに移動して、前を詰めるという動作をする。

![](assets/2023-01-30-19-17-17.png)

ページへのアクセスのたびにデータ構造を変更する必要があり、特別なハードウェアの仕組みが必要。（メモリアクセスのたびに介入）→NRUでは、最近に使われたものをざっくりと把握する。

## FIFO
先入れ先出し方式。LRUと異なるのはすでにページとして存在するものへのアクセスの場合、物理メモリ上のページを書き換えないことである。問題点は、ページインしてから使われたものとそうでないもの（しばらく放置されているもの）を区別できないところ。

## セカンドチャンス
物理メモリ上にあるページにアクセス時、MMUがそのページの参照ビットを1にする(1のときはそのまま)。ページアウトが必要なとき、リストの先頭の参照ビットが0ならば、そのページをアウトする。参照ビットが1ならば、参照ビットを0にして、リストの末尾に移動する（セカンドチャンス）。結果的に、参照ビットが0のページでもっとも古いものがアウトされる。

![](assets/2023-01-30-19-27-14.png)
![](assets/2023-01-30-19-27-27.png)

### セカンドチャンスの限界
P(ページテーブルのサイズ)回のページフォルトより短い粒度や、それ以前のアクセス履歴は用いていない。

## クロックアルゴリズム
動作はセカンドチャンスと同じで、二重リンクリストの代わりに循環バッファを使う

## エイジング
物理メモリ上の各ページに、一定間隔の区間ごとのアクセスあり・なしをN世代分記録したカウンタ値Hpを持たせる。

![](assets/2023-01-30-19-33-23.png)

1世代 = 一定の実時間かページ置換回数で区切る

世代が変わるとき、Hpをdecayさせ、直近のアクセスあるなしを反映

![](assets/2023-01-30-19-35-35.png)

ページ置換時には、カウンタ値Hpが最も小さいページを置換する。

## 実際の実装における考慮
ページアウトは物理メモリが枯渇する前にページフォルトと非同期に始める。ページアウト対象を選ぶ際は、最近のページのアクセスの有無だけでなく、二次記憶への書き込みが必要かどうかも考慮する必要がある。
- 初めてページアウトされるとき
- ページインから変更されているとき

は書き込みが必要であり、IOが発生する。

ページアウトは1ページ毎ではなく、連続した数十ページをまとめて行ったほうがIOの効率が良い。

# ページング制御API
`posix_madvise(addr, len, advice)`、`madvise(addr, len, advice)`でページング制御を行える。
後者はLinux特有で、機能が多い。
![](assets/2023-01-30-19-44-33.png)

# ファイルシステムの役割
二次記憶装置を簡便な共通のインタフェースで読めるようにする

# API
`open(path, flags, mode)`
- path: ファイルのパス
- flags: ファイルを開く方法
  - `O_RDONLY`: 読み込み専用
  - `O_WRONLY`: 書き込み専用
  - `O_RDWR`: 読み書き両方
  - `O_APPEND`: ファイルの末尾に追記
  - `O_CREAT`: ファイルが存在しない場合は作成
  - `O_EXCL`: ファイルが存在する場合はエラー
  - `O_TRUNC`: ファイルを空にする

などなど
ファイルディスクリプタにはファイルオフセット（次の読み書きが行われる場所）の情報が含まれている。

`read(fd, buf, sz)`
- fd: ファイルディスクリプタ
- buf: 読み込んだデータを格納するバッファ
- sz: 読み込むバイト数

実際に読み込まれたバイト数を返し、オフセットをその分だけ進ませる。

`write(fd, buf, sz)`
- fd: ファイルディスクリプタ
- buf: 書き込むデータが格納されたバッファ
- sz: 書き込むバイト数

場合によってはファイルが伸長する。実際に書き込まれたバイト数を返し、ファイルオフセットがその分だけ進む。

`lseek(fd, offset, whence)`
- fd: ファイルディスクリプタ
- offset: オフセット
- whence: オフセットの基準
  - `SEEK_SET`: ファイルの先頭
  - `SEEK_CUR`: 現在のオフセット
  - `SEEK_END`: ファイルの末尾

ファイルのオフセットを引数でしたものに変更する。

`posix_fallocate(fd, offset, len)`
`fallocate(fd, mode, offset, len)`
- fd: ファイルディスクリプタ
- mode: ファイルの拡張方法
  - `FALLOC_FL_KEEP_SIZE`: ファイルサイズを変更しない
  - `FALLOC_FL_PUNCH_HOLE`: ファイルの中身を空にする
- offset: ファイルの先頭からのオフセット
- len: ファイルの拡張する長さ

ファイルの[offset, offset+len]バイトの範囲を必要ならばファイルを伸長し、その範囲を0で埋める。

`ftruncate(fd, len)`
- fd: ファイルディスクリプタ
- len: ファイルのサイズ

ファイルの大きさをlenバイトに短縮する

`close(fd)`
- fd: ファイルディスクリプタ
 
openしたファイルを閉じる

`unlink(path)`
- path: ファイルのパス

ファイルを消す

# `mmap`
`void*a = mmap(addr, len, prot, flags, fd, offset)`
- addr: メモリのアドレス
- len: メモリの長さ
- prot: メモリの保護
  - `PROT_READ`: 読み込み可能
  - `PROT_WRITE`: 書き込み可能
  - `PROT_EXEC`: 実行可能
- flags: メモリのマッピング方法
  - `MAP_SHARED`: メモリ領域が複数のプロセスで共有される
    - 書き込みがファイルへ反映され、プロセス間で共有される。
  - `MAP_PRIVATE`: マッピングがコピーオンライトになる。
    - 書き込みがファイルへ反映されず、プロセス間でも共有されない。
- fd: ファイルディスクリプタ
- offset: ファイルの先頭からのオフセット

基本はfdに対応するバイトのoffsetからlenバイトをaddrにマッピングするという動作をする。fdが-1のときはただメモリを割り当てる。a[i]が、ファイルの(offset + i)バイト目を指すようになる。
すなわち、メモリが新たに割り当てられることにもなる。

以下の役割を持つ。
1. ファイルをメモリ上にあるかのように読み書きする
2. メモリを割り当てる（sbrk）
3. 割り当てるアドレスを指定する
4. 書き込み保護を設定する

## ファイル読み出し

```c
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main() {
  int fd = open("test.txt", O_RDONLY);
  struct stat st;
  fstat(fd, &st);
  char *p = mmap(NULL, st.st_size, PROT_READ, MAP_SHARED, fd, 0);
  for (long i = 0; i < st.st_size; i++) {
    printf("%c", p[i]);
  }
  munmap(p, st.st_size);
  close(fd);
}
```

## ファイル書き出し

```c
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main() {
  int fd = open("test.txt", O_RDWR|O_TRUNC|O_CREAT, 0777);
  posix_fallocate(fd, 0, 100000000); //サイズを100MBにする
  struct stat st;
  fstat(fd, &st);
  char *p = mmap(NULL, st.st_size, PROT_WRITE, MAP_SHARED, fd, 0);
  for (long i = 0; i < st.st_size; i++) {
    p[i] = i % 128;
  }
  munmap(p, st.st_size);
  close(fd);
}
```

## メモリ割り当て

```c
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main() {
  char *p = mmap(NULL, 100000000, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
  munmap(p, 100000000);
}
```

1. 保護属性を指定出来る
2. 個別に開放出来る
3. 割り当てるアドレスを第一引数で指定出来る

といった利点がある。

## 実装概要
1. [a, a+sz) を論理的に割り当てる
2. ファイルとの対応関係も記録する
3. ページテーブル上では、ページは不在としておく。
4. ページフォルト発生時にアドレス空間記述表を見て、ファイルに対応した領域であればファイルの対応する領域を読み込む

## 共有マッピングとプライベートマッピング
違いは物理メモリをどう使うか。

### `MAP_SHARED`
- 同じ場所に対しては同じ物理アドレスを用いる（実はキャッシュをそのまま使う）

### `MAP_PRIVATE`
- 書き込みが実際に起こるまでは、物理メモリを共有し、書き込みが発生したら異なる物理ページを利用する。（コピーオンライト）

※readは各プロセスが異なる物理アドレスを用いる

## 効果的な場面
1. ファイルのごく一部を飛び飛びにアクセスする
   1. lseek+readは面倒、readですべて読み込むのは無駄なのでmmapでファイル全体をマップする
2. 多くのプロセスが同じファイルをアクセスする
   1. 同一領域に対する物理ページは共有される（MAP_SHARED）
   2. 書き込まれていない領域に対する物理ページは共有(MAP_PRIVATE)
   3. readは各プロセスが異なる物理アドレスを用いる

3. プログラムの読み込み
   1. 上にあげた性質をもつため。

# コピーオンライト

変更するまでは物理メモリを共有する方式のこと。物理ページを共有しつつ、MMUの機能で書き込み不可の属性をつけておく。書き込み時に保護例外を発生させ、OSが対応する物理メモリをコピーし、書き込まれたページ→新しい物理ページにマッピングを変更する。



# キャッシュ
一度読んだファイルをメモリ上に保持し、一定期間保持する。

1. read要求された部分がキャッシュにない
   1. カーネル内にキャッシュのための領域を割り当てる
   2. 二次記憶からキャッシュ上に読み込む
   3. プロセスのページにもコピー
2. キャッシュ上にある
   1. キャッシュからプロセスのページにコピーする

# 先読み
プロセスが要求していない部分を先に読んでおくこと。実際にはファイルの読み出しが連続した部分で何回か行われた時点で発動する。


# ファイルディスクリプタと疑似ファイル

## Unixの特徴
### 1. 出力先の変更
Terminalに出力するつもりでプログラムを書けば、それがそのままファイルを出力するプログラムになる

```c
int main() { printf("hello\n"); }
```

```bash
$ ./hello
hello
```
出力先をredirect
```bash
$ ./hello > out.txt
```

### 2. 入力先の変更
同様に入力も。
```bash
$ cat content.txt
hello hiroshi
$ ./say_something < content.txt
hello hiroshi
```

### 3. パイプでプロセス間通信
ex
```bash
ps auxww | grep firefox
```

プロセス外部とのやりとりはすべてファイルディスクリプタを経由して行われる。

everything is file
- プロセスごとにアドレス空間は分離されている
- fdに `read/write`を発行して通信しよう


## リダイレクト・パイプの仕組み
### 子プロセスにfdを継承する
```c
int fd = open(...);
pid_t pid = fork();
if (pid == 0) {
    // child
    read(fd, buf, sz); // OK
}
```
childの中でexecしても有効なので、execした先でもfdの値さえ分かれば openした内容を使える。

### 標準入出力
- fd=0: 標準入力
- fd=1: 標準出力
- fd=2: 標準エラー出力

### ファイルAPI(高水準)
生のfdに対応する高水準APIがC言語では提供されている
```c
FILE *fp = fopen(filename, mode);
fread(buf, size, n, fp);
fwrite(buf, size, n, fp);
```
日本語記事: https://programming-place.net/ppp/contents/c/040.html#rw_open

### `dup2` システムコール
```c
int err = dup2(oldfd, newfd);
```
ファイルディスクリプタ`oldfd`を`newfd`でも使えるようにする(複製)。

#### 入力リダイレクト`cmd < filename`の実現方法
```c
const int fd = open(filename, O_RDONLY);
pid_t pid = fork();
if (pid) {
    // parent
    close(fd); // 親には不要
} else {
    // child: fd -> 0へ付け替えて、0(stdin)で読めるように
    if (fd != 0) {
        dup2(fd, 0);
        close(fd);
    }
    execvp(cmd, ...);
}
```
#### 出力リダイレクト`cmd > filename`の実現方法
```c
const int fd = creat(filename); // 新しいfdを生成
pid_t pid = fork();
if (pid) {
    // parent
    close(fd); // 親には不要
} else {
    // child: fd -> 1へと付け替えて、1(stdout)に吐けるように
    if (fd != 1) {
        close(fd);    // ここの順序は先にcloseしないとバグるのかは不明
        dup2(fd, 1);
    }
    execvp(cmd, ...);
}
```

### `pipe`システムコール
```c
int rw[2]; // fdの配列
int err = pipe(rw);
```
`rw[0]`, `rw[1]`にそれぞれ読み出し・書き出し用のfdが生成される。
`rw[1]`に書いたデータが`rw[0]`から呼び出せる。
これをforkによるfdの継承を使って、親子プロセス間通信を実現。

#### pipe (親 -> 子)
親がwに書き出したものを、子がstdio(0)から読めれば良い
```c
int rw[2]; pipe(rw);
int r = rw[0], w = rw[1];
pid_t pid = fork();
if (pid) { // parent
    close(r); // 親はreadは不要
    // WRITE to `w` here
    close(w);
} else { // child
    close(w); // 子はwrite不要
    dup2(r, 0);
    close(r);
    execvp(...);    // stdio(0)から読むコマンド
}
```
#### pipe (子 -> 親)
子がstdout(1)に書き出したものを、親がrから読めれば良い
```c
int rw[2]; pipe(rw);
int r = rw[0], w = rw[1];
pid_t pid = fork();
if (pid) { // parent
    close(w); // 親はwrite不要
    // READ from `r` here
    close(r);
} else { // child
    close(r); // 子はread不要
    dup2(w, 1);
    close(w);
    execvp(...);    // stdout(1)へ書き出すコマンド
}
```
例: `popen`ライブラリ関数

## 疑似ファイル
2次記憶上のデータに限らず、openしてread/writeできるものは全てファイルとする。

- 名前付きパイプ(FIFO)
  - `int err = mkfifo(pathname, mode);`
- `/proc`ファイルシステム
  - プロセスやOSの内部状態に関するファイル
- `cgroups`ファイルシステム(詳細は05memory.pdf)
  - プロセスの集合に割り当てる資源を制御する機能
  - `sudo mount -t cgroup2 none dir`
- `tmpfs`
  - 実体がメモリ上にある(揮発)ファイルシステム
- デバイスファイル
  - I/O装置(camera, microphone, ...)もファイルとして扱う
  - 単純な例:
    - `/dev/null`, `/dev/zero`, `/dev/urandom`