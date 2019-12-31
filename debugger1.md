# デバッガを作る前に,,,


この本ではデバッガを自作することによって、低レイヤーの理解を深めようというものです。
## 必要な知識や環境
* 多少のC言語
* Linux環境
* レジスタとは
* gdbなどを使ったことがある
* 


## 環境
ここで学んでいく環境は
* Ubuntu
* c言語 gcc

これらを使っていきます。

ここからは環境が整ったことを前提に話を勧めていく。

## プロセスについて
デバッガを学ぶ上でどのようにプログラムが動いているかつまりどのようにプロセスが使われているかを理解する必要があります。
というわけで簡単にプロセスについて解説していきます。それ以外は自身でググってほしい。

#




# 目覚め 子プロセスの生成
というわけでプロセスについて学んできたが、ここからは実際にc言語のコードを書いてみることによってプロセスについて理解していこう。
デバッガを作る上で子プロセスを生成するまたは実行中のプロセスをあっタッチして制御にするということが必要不可欠です。はじめにここで作っていくデバッガは、まだ起動（実行中でない）していない実行ファイルを子プロセスとして起動して制御するものなので、下記で行う作業は重要なものです。
#### 子プロセスを作る

下記がコードだ。ここで行っているのは、`fork()`を使って子プロセスを生成を行っている。また`fork()`はプロセスの生成に成功したときに、子プロセスのプロセスIDを返す。そして生成された子プロセスでは、同じコードが実行されるが`fork()`で0を返す。

```c:tracer1.c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>

int main ()
{
    pid_t child_pid;
    
    child_pid = fork();

    if(child_pid == 0){
        printf("I`m tracee\n");
    }
    else if (child_pid > 0){
        printf("I`m tracer and child process id is %d \n", child_pid);
    }
    else {
        printf("error at forking target.\n");
    }    
}
```
で実行すると次のように出た。
```
user@ubuntu:~/Desktop/tracer$ gcc tracer1.c -o tracer
user@ubuntu:~/Desktop/tracer$ ./tracer
I`m tracer and child process id is 6448 
user@ubuntu:~/Desktop/tracer$ I`m tracee
```
今回は子プロセスのプロセスIDは6448だったようだ。
$ のあとに`Im tracee.`が表示されていて奇妙だがこれは親プロセスが終了したあとに、子プロセスの`printf()`が実行したのだろうと考えられる。

### 任意の実行ファイルを子プロセスとして実行する
#### 子プロセスとして、別なプログラムを実行してみる。
別なプログラムの例としてここでは、下の簡単なhelloプログラムをトレーサーと同じディレクトリ内に作ってそれを子プロセスとして動かすことを試みます。
しかしながらデバッガを使う際にプログラムが上手く動かないことがあるのでgcc の引数に`-static`を付ける必要があります。
```c:hello.c
#include <stdio.h>
int main ()
{
    printf("Hello\n");
    return 0;
}
```
下記では`child_pid`が0つまり子プロセスだった際に`run_target`関数を実行しその中の`execl()`を実行します。`execl()`はパスからプログラムを実行するものです。つまりここでは`hello`実行ファイルを実行します。

```c:tracer2.c
##include <stdio.h>
#include <sys/types.h>
#include <unistd.h>

void run_target(){
    execl("hello", "hello", NULL);
}

int main ()
{
    pid_t child_pid;
    
    child_pid = fork();

    if(child_pid == 0){
        printf("I`m tracee\n");
        run_target();
    }
    else if (child_pid > 0){
        printf("I`m tracer and child process id is %d \n", child_pid);
    }
    else {
        printf("error at forking target.\n");
    }    
}

```
で実行すると、
```
user@ubuntu:~/Desktop/tracer$ ./tracer
I`m tracer and child process id is 6601 
user@ubuntu:~/Desktop/tracer$ I`m tracee
Hello

```
Helloと出たので、ここではうまくいきました。
execlシステムコールはうまくいくと戻ってこないので戻ってきた場合はすべてエラーだったということになります。

これでやっと子プロセスとして実行ファイルを実行することができました。
