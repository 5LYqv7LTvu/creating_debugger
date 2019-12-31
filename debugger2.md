# 2 デバッガの機能をつける

ここからはついにデバッガの機能を追加していきます。


## 親プロセスをデバッガに変身！
### ptraceを使う
#### ptraceについて
対象のプログラムをデバッグするためにptraceシステムコールを使いますこのシステムコールはよく使われているデバッガのgdbやstraceやradare2などにもつかわれているものです。
ptraceシステムコールは対象のプロセスを観察したり制御すること事ができる。
#### とりあえずptraceを使ってみる
とりあえず使ってみよう。
ということでまずはPTRACE_TRACEMEを使って「子プロセスの僕をトレースして！！！」と親プロセスにつたえる？ものを使ってみる。これを使うことによって、これ以降のexec()をを呼び出すたびにSIGTRAPが送信されるようだ、つまりデバッグ対象が実行される前に停止ししているために ***`Hello` は表示されない！***
またあと実行対象をコマンドライン引数から指定できるようにしてあるので実行の際は`$ ./tracer hello`のようにしてください。

```c:tracer3.c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/ptrace.h>

void run_target(const char* target){
    ptrace(PTRACE_TRACEME, 0, 0, 0);
    execl(target, target, NULL);
}

int main (int argc, char **argv)
{
    pid_t child_pid;
    
    if (argc < 1){
        printf("Usage: %s <target>\n",argv[0]);
    }

    child_pid = fork();

    if(child_pid == 0){
        printf("I`m tracee\n");
        run_target(argv[1]);
    }
    else if (child_pid > 0){
        printf("I`m tracer and child process id is %d \n", child_pid);
    }
    else {
        printf("error at forking target.\n");
    }    
}
```

実行すると次のようになった。

```
user@ubuntu:~/Desktop/tracer$ ./tracer hello
I`m tracer and child process id is 7286 
user@ubuntu:~/Desktop/tracer$ I`m tracee
```
実行対象をコマンドライン引数から指定できるようにしたために、Helloを引数として指定してある。
そしてHelloが表示されてないことから、execl()を実行した直後に、子プロセスは一時停止していることがわかります。

####止まっちゃったから動かす。
PTRACE_CONTを使ってさっき止めたトレーシーを再び動かしてみます。これは、親プロセスで実行し、一時停止したトレーシーのプロセスをリスタートさせるものです。
ここでは先程止まってしまっていた子プロセスを動かすために、デバッガ側でPTRACE_CONTを使いたいがそのまま使うと子プロセスのPTRACE_TRACEMEよりも前にPTRACE_CONTを実行してしまうために結局Hello.が表示されないままになってしまいます。そのためにデバッガ側で子プロセスが停止するのを待つ必要があります。
#### 何を使って待つか
waitpid()を使って待つことにする。waitpidは子プロセスが停止するまで待ち、そして子プロセスが何によって停止または終了したのかを教えてくれます。
以下のような感じになっています。あと `int status;`の宣言を忘れずに！

* WEFIXITED
    子プロセスが正常に終了した場合にtureを返す。
* WIFSIGNALED
    子プロセスがシグナルによって終了した場合にtrueを返す。
* WIFSTOPPED
    子プロセスがシグナルによって停止した場合にtrueを返す。

下記の実行結果よりここでは、”stopped by signal 5”のあとにHelloがあることからwaitpidによって親プロセスが子プロセスの停止を待機し、子プロセスのPTRACE_TRACEMEによって動作が停止されたあとに、親プロセスの動作が再開され、親プロセスのPTRACE_CONTによって子プロセスのが再開しHelloが表示された。

```c:tracer4.c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/ptrace.h>
#include <sys/wait.h>

void run_target(const char* target){
    ptrace(PTRACE_TRACEME, 0, 0, 0);
    execl(target, target, NULL);
}

int main (int argc, char **argv)
{
    pid_t child_pid;
    int status;
    
    if (argc < 1){
        printf("Usage: %s <target>\n",argv[0]);
    }

    child_pid = fork();

    if(child_pid == 0){
        printf("I`m tracee\n");
        run_target(argv[1]);
    }
    else if (child_pid > 0){
        printf("I`m tracer and child process id is %d \n", child_pid);
        waitpid(child_pid, &status, 0);
        if (WIFEXITED(status)) {
            printf("program exited normally\n");
            exit(0);
        } else if (WIFSIGNALED(status)) {
            printf("terminated by signal %d\n", WTERMSIG(status));
        } else if (WIFSTOPPED(status)) {
            printf("stopped by signal %d\n", WSTOPSIG(status));
        }

        ptrace(PTRACE_CONT, child_pid, NULL, NULL);
    }
    else {
        printf("error at forking target.\n");
    }    
}
```
で実行結果だ。
```
user@ubuntu:~/Desktop/tracer$ ./tracer hello
I`m tracer and child process id is 8360 
I`m tracee
stopped by signal 5
user@ubuntu:~/Desktop/tracer$ Hello
```

はい。
上記の実行結果でHello が$の後ろにあるのでデバッガ側の最後にもwaitpid() を入れるのが好ましいかもしれませんね。


## ブレークポイントを置く
やっとデバッガらしいことを始めていきます。ここではブレークポイントを置き任意の場所で動作を止めることを試みていきます。細かく言うとブレークの命令（0xCC つまり int 3つまりデバッグ例外ハンドラを呼び出す1バイト命令）を置くことによって動作を止めることをします。
またその次にはそのブレークポイントを取って動作を再開させることをします。細かく言うとブレークする際にもとの命令の内容（置き換えるアドレスの中身）を保存しておき、ブレークしたことが確認できたあとに、ブレークポイントをおいてしまったことによって実行されなかった命令を実行するために、rip（今どの目入れを実行しているかを示すレジスタ）の値を1戻しすことと（１バイト分戻し）保存しておいたもとの命令を0xCCから書き換えてもとに戻してあげることをします。また子プロセスが動く際には`waitpid`忘れないように注意してあげましょう。


### とりあえず命令を取ってきてみる。
破壊の前に、、、、
ブレークポイントの設置の前にもとの命令の内容を取ってくる必要があり、そうしないとブレークしてその後に再開したときに命令内容に矛盾が生じて正常にプログラムが動作しなくなってしまいます。
ここでやるのは、ripのアドレスにある命令を取ってくることだ。

```c:tracer5.c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/ptrace.h>
#include <sys/wait.h>
#include <stdlib.h>
#include <sys/user.h>

void run_target(const char* target){
    ptrace(PTRACE_TRACEME, 0, 0, 0);
    execl(target, target, NULL);
}

void run_debugger(pid_t child_pid){
    int status;
    unsigned original_text;
    struct user_regs_struct regs;
    
    printf("I`m tracer and child process id is %d \n", child_pid);
    waitpid(child_pid, &status, 0);
    if (WIFEXITED(status)) {
        printf("program exited normally\n");
        exit(0);
    } else if (WIFSIGNALED(status)) {
        printf("terminated by signal %d\n", WTERMSIG(status));
    } else if (WIFSTOPPED(status)) {
        printf("stopped by signal %d\n", WSTOPSIG(status));
    }
    ptrace(PTRACE_GETREGS, child_pid, 0, &regs);

    original_text = ptrace(PTRACE_PEEKTEXT, child_pid, regs.rip, 0);
    printf("rip: 0x%llx\n", regs.rip);
    printf("%x\n", original_text);
    

    ptrace(PTRACE_CONT, child_pid, NULL, NULL);

    waitpid(child_pid, &status, 0);
    if (WIFEXITED(status)) {
        printf("program exited normally\n");
        exit(0);
    } else if (WIFSIGNALED(status)) {
        printf("terminated by signal %d\n", WTERMSIG(status));
    } else if (WIFSTOPPED(status)) {
        printf("stopped by signal %d\n", WSTOPSIG(status));
    }
}

int main (int argc, char **argv)
{
    pid_t child_pid;
    int status;
    
    if (argc < 1){
        printf("Usage: %s <target>\n",argv[0]);
    }

    child_pid = fork();

    if(child_pid == 0){
        //子プロセス
        printf("I`m tracee\n");
        run_target(argv[1]);
    }
    else if (child_pid > 0){
        //親プロセス
        run_debugger(child_pid);
    }
    else {
        printf("error at forking target.\n");
    }    
}
```
注目してほしいのはptraceが使われているところの`PTRACE_GETREGS``PTRACE_PEEKTEXT`のところです。
PTRACE_GETREGSは子プロセスのすべてのレジスタの値を取得し、regs構造体に代入するということを行っています。これには`sys/user.h`が必要です。
PTRACE_PEEKTEXTはregs.ripが示すアドレスに入っているプログラムの命令文を取ってきて、original_textに代入するということを行っていて、その下の２つのprintfでripの値とそのアドレスにある命令文を表示しています。
実行結果は次の通りです。
```
user@ubuntu:~/tracer$ ./tracer hello
I`m tracer and child process id is 2361 
I`m tracee
stopped by signal 5
rip: 0x400a30
8949ed31
Hello
program exited normally
```
ripそしてripが示すアドレスの中身から4バイト分が示されていることがわかる。何度か実行を繰り返してみるとプロセスIDとripの値が変わるが、命令の内容は変わらないので正常にうごいているみたです。

### ブレークポイントの設置
##### どこで？
ブレークポイントを設置したいのですが、とりあえずmain関数の最初のアドレスにブレークポイントを設置したいと思います。
よってまず、ここではmain関数の始まりのアドレスを知る必要があります。
それはnmコマンドによって知ることができます。
> 余談ですが、これによってどこで上のときにhelloプログラムが停止したかがわかります。


```
user@ubuntu:~/tracer$ nm hello
```
 これをやるとhello内にあるシンボル情報つまり関数名に対応したアドレスを知ることができます。たくさん表示されているので、`grep`を使ってmainという文字列を含む行だけを抽出しましょう。
 
 ```
 user@ubuntu:~/tracer$ nm hello | grep main
000000000044d010 T _dl_get_dl_main_map
00000000006ba300 d _dl_main_map
00000000004160f0 T _IO_switch_to_main_get_area
0000000000470630 T _IO_switch_to_main_wget_area
00000000004a311c R _libc_intl_domainname
0000000000400de0 T __libc_start_main
0000000000400b4d T main
00000000006b9880 d main_arena
00000000006b90f8 D _nl_current_default_domain
0000000000492378 R _nl_default_default_domain
00000000006bc7e0 B _nl_domain_bindings
0000000000403840 T _nl_find_domain
000000000048fbf0 T _nl_finddomain_subfreeres
0000000000403ae0 T _nl_load_domain
00000000006bb858 b _nl_loaded_domains
000000000048fc40 T _nl_unload_domain
user@ubuntu:~/tracer$ 
 ```
 
 main関数の始まりのアドレスが400b4dということが分かりました！
 
 > 余談ですが、これによってどこで上のときにhelloプログラムが停止したかがわかります。tracer5の実行結果により0x400a30の位置でsignal5つまり原因がptraceによる停止が行われています。見てみましょう。
```
user@ubuntu:~/tracer$ nm hello | grep 400a30
0000000000400a30 T _start
user@ubuntu:~/tracer$ 
 
```
`_start`のいちで停止しているようです。_startとはプログラムのエントリーポイントです。詳しくはしらべてみてください。


##### ブレークポイントを置いてみる

```c:tracer6.c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/ptrace.h>
#include <sys/wait.h>
#include <stdlib.h>
#include <sys/user.h>

void run_target(const char* target){
    ptrace(PTRACE_TRACEME, 0, 0, 0);
    execl(target, target, NULL);
}

void run_debugger(pid_t child_pid){
    int status;
    unsigned long long original_text;
    unsigned long long break_text;
    struct user_regs_struct regs;
    
    printf("I`m tracer and child process id is %d \n", child_pid);
    waitpid(child_pid, &status, 0);
    if (WIFEXITED(status)) {
        printf("program exited normally\n");
        exit(0);
    } else if (WIFSIGNALED(status)) {
        printf("terminated by signal %d\n", WTERMSIG(status));
    } else if (WIFSTOPPED(status)) {
        printf("stopped by signal %d\n", WSTOPSIG(status));
    }

    original_text = ptrace(PTRACE_PEEKTEXT, child_pid, 0x400b4d, 0);
    // printf("rip: 0x%llx\n", regs.rip);
    // printf("%x\n", original_text);

     if(original_text == 0xFFFFFFFFFFFFFFFF){
        printf("faied.\n");
    }else{
    printf("%016llx\n", original_text);
    }
    
    ptrace(PTRACE_POKETEXT, child_pid, 0x400b4d, ((original_text & 0xFFFFFFFFFFFFFF00) | 0xCC));
    break_text = ptrace(PTRACE_PEEKTEXT, child_pid, 0x400b4d, 0);
    printf("%016llx\n",break_text);
    

    ptrace(PTRACE_CONT, child_pid, NULL, NULL);

    waitpid(child_pid, &status, 0);
    if (WIFEXITED(status)) {
        printf("program exited normally\n");
        exit(0);
    } else if (WIFSIGNALED(status)) {
        printf("terminated by signal %d\n", WTERMSIG(status));
    } else if (WIFSTOPPED(status)) {
        printf("stopped by signal %d\n", WSTOPSIG(status));
    }
}

int main (int argc, char **argv)
{
    pid_t child_pid;
    int status;
    
    if (argc < 1){
        printf("Usage: %s <target>\n",argv[0]);
    }

    child_pid = fork();

    if(child_pid == 0){
        //子プロセス
        printf("I`m tracee\n");
        run_target(argv[1]);
    }
    else if (child_pid > 0){
        //親プロセス
        run_debugger(child_pid);
    }
    else {
        printf("error at forking target.\n");
    }    
}
```
`PTRACE_POKETEXT`まではただ見やすくしただけだが、`PTRACE_PEEKTEXT`は正常に命令文を取ってこれないとき-1を返すのでそれをわかりやすくした。
重要なのは`PTRACE_POKETEXT`で、PTRACE_POKETEXTは命令文を書き換える事ができる命令です。ここで行っているのはmain関数の始まりのアドレスつまり`0x400b4d`にブレークポイントの命令、つまり0xCCつまりint 3を代入することによって、ブレークポイントの設置を試みています。ここではoriginal_textをマスクを使うことによって一番うしろの1バイトのみに0xccを入れています。
```
user@ubuntu:~/tracer$ ./tracer hello
I`m tracer and child process id is 3164 
I`m tracee
stopped by signal 5
rip: 0x400a30
ac3d8d48e5894855
ac3d8d48e58948cc
stopped by signal 5
rip: 0x400b4e
user@ubuntu:~/tracer$ Hello

```
プログラムがエントリーポイントで停止したところまではさっきまでと同様ですが、ここではアドレス`0x400b4d`にブレークポイントの命令`0xCC`を代入したことによってアドレスの中身が`ac3d8d48e5894855`から`ac3d8d48e58948cc`に変わり、動作を再開したことによってripがメイン関数の始まりのアドレスから一進んだところ(アドレスの中身で言う１バイト)にd+1 = e なっていることがわかります。
>0xCCを代入したところがアドレスの中身を表示したときだど最後になっていますが、これはリトルエンディアンによるもので実際のアドレスの中身は,
>`554889e5488d3dac`から`cc4889e5488d3dac`のようになっています。

うーんなんだか思っていたのと違いますというのも、main関数の最初のアドレスの中身にブレークポイントを設置しているのでHelloは表示されることを想定していません。もしかしたら停止した状態で小プロセスよりも先に親プロセスが停止したために、Linux側が自動的に小プロセスの動作を再開してくれているのかもしれません。でもそうだったとしても命令が書き換わっているために正常に動作しないはずですが、奇跡的にうまく行っているのかもしれません。そうだったとしてもここでの目標は0xcc を埋め込むことだったので見逃してあげましょう。


### ブレークポイントの再開

ということで、ブレークしたあとにまた子プロセスを再開しHelloを表示させたい。(さっきなっちゃったけど（笑）)正当なやり方でやって行くぜ
これを達成するためには、
* 命令をもとに戻してあげる。
* もう一度ブレークしたところの命令を実行させる、つまりripの値を一つ減らしてから実行あげる。
これらをすれば良い。
ここでもブレークポイントを設置したときのようにマスクを用いて、0xcc

```c:tracer7.c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/ptrace.h>
#include <sys/wait.h>
#include <stdlib.h>
#include <sys/user.h>

void run_target(const char* target){
    ptrace(PTRACE_TRACEME, 0, 0, 0);
    execl(target, target, NULL);
}

void run_debugger(pid_t child_pid){
    int status;
    unsigned long long original_text;
    unsigned long long break_text;
    struct user_regs_struct regs;
    
    printf("I`m tracer and child process id is %d \n", child_pid);
    waitpid(child_pid, &status, 0);
    if (WIFEXITED(status)) {
        printf("program exited normally\n");
        exit(0);
    } else if (WIFSIGNALED(status)) {
        printf("terminated by signal %d\n", WTERMSIG(status));
    } else if (WIFSTOPPED(status)) {
        printf("stopped by signal %d\n", WSTOPSIG(status));
    }
    ptrace(PTRACE_GETREGS, child_pid, 0, &regs);
    printf("rip: 0x%llx\n", regs.rip);


    original_text = ptrace(PTRACE_PEEKTEXT, child_pid, 0x400b4d, 0);
    // printf("rip: 0x%llx\n", regs.rip);
    // printf("%x\n", original_text);

     if(original_text == 0xFFFFFFFFFFFFFFFF){
        printf("faied.\n");
    }else{
    printf("%016llx\n", original_text);
    }
    
    ptrace(PTRACE_POKETEXT, child_pid, 0x400b4d, ((original_text & 0xFFFFFFFFFFFFFF00) | 0xCC));
    break_text = ptrace(PTRACE_PEEKTEXT, child_pid, 0x400b4d, 0);
    printf("%016llx\n",break_text);
    

    ptrace(PTRACE_CONT, child_pid, NULL, NULL);

    waitpid(child_pid, &status, 0);
    if (WIFEXITED(status)) {
        printf("program exited normally\n");
        exit(0);
    } else if (WIFSIGNALED(status)) {
        printf("terminated by signal %d\n", WTERMSIG(status));
    } else if (WIFSTOPPED(status)) {
        printf("stopped by signal %d\n", WSTOPSIG(status));
    }
    ptrace(PTRACE_GETREGS, child_pid, 0, &regs);
    printf("rip: 0x%llx\n", regs.rip);

    break_text = ptrace(PTRACE_PEEKTEXT, child_pid, 0x400b4d, 0);

    ptrace(PTRACE_POKETEXT,child_pid, 0x400b4d, ((break_text & 0xFFFFFFFFFFFFFF00) |(original_text & 0x00000000000000FF)));
    
    
    ptrace(PTRACE_GETREGS, child_pid, 0, &regs);
    regs.rip -= 1;
    ptrace(PTRACE_SETREGS, child_pid, 0, &regs);

    ptrace(PTRACE_CONT, child_pid, NULL, NULL);
    waitpid(child_pid, &status, 0);
    if (WIFEXITED(status)) {
        printf("program exited normally\n");
        exit(0);
    } else if (WIFSIGNALED(status)) {
        printf("terminated by signal %d\n", WTERMSIG(status));
    } else if (WIFSTOPPED(status)) {
        printf("stopped by signal %d\n", WSTOPSIG(status));
    }

}

int main (int argc, char **argv)
{
    pid_t child_pid;
    int status;
    
    if (argc < 1){
        printf("Usage: %s <target>\n",argv[0]);
    }

    child_pid = fork();

    if(child_pid == 0){
        //子プロセス
        printf("I`m tracee\n");
        run_target(argv[1]);
    }
    else if (child_pid > 0){
        //親プロセス
        run_debugger(child_pid);
    }
    else {
        printf("error at forking target.\n");
    }    
}
```
命令の内容を表示するところで本の命令の内容を変数original_text内に入れておいてあるのでそれをまた代入してあげればいい。が、PTRACE_POKETEXTのあとにPTRACE_PEEKTEXTを前のところでやってしまっているので。それをしないようにして、もとの命令をそのままにしておきたい
```
user@ubuntu:~/tracer$ gcc tracer7.c -o tracer
user@ubuntu:~/tracer$ ./tracer hello
I`m tracer and child process id is 3667 
I`m tracee
stopped by signal 5
rip: 0x400a30
ac3d8d48e5894855
ac3d8d48e58948cc
stopped by signal 5
rip: 0x400b4e
Hello
program exited normally
user@ubuntu:~/tracer$ 
```
しっかりとブレークポイントの次のアドレスで止まってくれて、かつ命令を戻したあとに動作を再開してもしっかりと動いてくれた。
