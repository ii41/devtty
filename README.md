# `/dev/tty`

[English](README.en.md)

不少大人气终端应用程序其实都有一个bug：
```
   fzf 2> /dev/null
```
你看不见`fzf`给你的选项了。Ctrl-C逃跑吧。
```
   vi > /dev/null
```
除非你是什么在shell的配置里`alias vi=vscode`的怪物，基本上你遇到的情形是一样的：你的shell不理你了。用Ctrl-C也逃不出来。

要是你脑子快的话，会意识到这时候赶紧`:wq`再回车可以退出。如果你是什么思考之前先行动的少女漫画主人公，或者什么思考之前先行动的少年漫画主人公，那你完了。你的一堆莫名其妙的输入已经把文本编辑器逼进了一个不知如何的状态，但是它的输出被重定向了，所以你无法确认它的状态，也不知如何从这个状态里退出。这个shell救不回来了。

有人可能会说这不算bug, 毕竟正常来说没人会这么用这些程序。这就是个什么算bug的辩经问题。反正事实是，你这么用`vi`，你的shell就会被封印。我觉得这样不好。

要是你对终端有所了解的话，你已经知道问题是什么了。如果你不了解的话，我来稍微解释一下。

`vi`、`fzf`之类的，在你的终端上画出好看的画的程序，并不是通过魔法操纵你的终端的。它们向终端发送<u>终端转义码</u>（Terminal escape code）来告知终端自己想要它进行的操作。同时，它们也可以从终端接收终端转义码来获取一些信息。

来写点C吧。
```C
#include <stdio.h>
int main()
{
        fputs("\033[2J", stdout); // Clear the terminal.
        fputs("\033[1;1H", stdout); // Move the cursor to row 1, column 1.
        fputs("hello world\n", stdout);
        return 0;
}
```
这篇文章里所有的代码都可能导致你的终端没救，所以做好重开终端的准备。

编译、执行，你的终端会清空上面的文字，之后从第1行第1列开始打印“hello world”。你的shell执行完这个小程序之后又会如常在你的终端上打印出命令提示行。

终端转义码的具体格式是没有个标准的。或者说，有（[ECMA-48](https://www.ecma-international.org/publications-and-standards/standards/ecma-48/)），但是没有哪个终端的实现会遵守。在Linux上，有个比较方便的做法：`man console_codes`。这个命令会告诉你Linux console，也就是Linux自己对终端的实现支持的码。大部分别的终端也支持这些码中的大部分。在Linux上Ctrl-Alt-F3，你看见的就是Linux console。如果觉得这些不够，[这里](https://invisible-island.net/xterm/ctlseqs/ctlseqs.html)有很多别的码。有的码有的终端支持。反正你想要确认某个功能能不能实现，唯一的办法就是在想支持的终端上自己去试。

那么，重定向这个小程序，会发生什么呢？
```shell
    $ ./a.out > /tmp/codes # Assuming the name of the executable is a.out.
```
终端上什么都没有发生。
```shell
    $ cat /tmp/codes
```
打印这个文档却重现了之前执行程序的效果。

现在我们知道最开始的时候我们看到的`fzf`和`vi`的问题出在哪了。它们通过输出转义码控制终端。`fzf`把转义码发往`stderr`。`vi`把转义码发往`stdout`。`stderr`和`stdout`刚好就是终端的话，它们就能被正常使用。但是这两个输出流是可以被重定向的。它们被重定向到文件的话，转义码也会被写进文件，终端收不到这些转义码，自然也就不会遵循它们的指令画画。

你可能会想，可以用`isatty`来检查这些流是否是终端，如果不是，退出就好了。这也不能完全解决这个问题。有些真实恶人可以把这些流重定向到别的终端上（`man pts`）。有些程序，会向问`stderr`发转义码问一些终端有多大，光标在哪之类的问题，通过`stdin`获取结果。恶人就可以把`stderr`和`stdin`连接到不同的终端上，给程序的提问发送虚假的回复，然后让程序在一个表示20x20的终端的内存缓冲区上表示第200行第200列的位置做修改。

那这个问题能解决吗？像这种大人气软件都没能解决的问题，是不是无解的？答案是理论上能解决。工程上能不能，我又不是它们的开发者我怎么知道。
```C
#include <stdio.h>
int main()
{
        FILE *dev_tty = fopen("/dev/tty", "w");
        fputs("\033[2J", dev_tty); // Clear the terminal.
        fputs("\033[1;1H", dev_tty); // Move the cursor to row 1, column 1.
        fputs("hello terminal\n", dev_tty);
        fclose(dev_tty);
        fputs("hello stdout\n", stdout);
        return 0;
}
```
类Unix操作系统会给跟终端连接的程序分配一个特殊文件：`/dev/tty`。这个文件就是终端。因此，重定向标准输入输出流对这个文件上的输入输出没有影响。

`/dev/tty`没有什么详细易读的文档。`man 4 tty`多少可以看到一点。

编译执行这几行代码。单纯执行的话，跟之前的“hello world”相比，除了输出的文字有所不同之外，行为上没有什么区别。但是，若是重定向这个程序的`stdout`，会发现给`/dev/tty`的输出完全不受影响。而被写入成为重定向的目标的那个文件的，也就只是“hello stdout”这行文字而已。

`/dev/tty`不止可以用作输出目标，也可以用于读取输入。
```C
#include <stdio.h>

int main()
{
        FILE *dev_tty = fopen("/dev/tty", "r+");
        fputs("\033[2J", dev_tty); // Clear the terminal.
        fputs("\033[1;1H", dev_tty); // Move the cursor to row 1, column 1.
        fputs("hello terminal\nGive me a character > ", dev_tty);
        fflush(dev_tty);
        char c = fgetc(dev_tty);
        fclose(dev_tty);
        fprintf(stdout, "The character I got: %c\n", c);
        return 0;
}
```
编译、执行。键入一个字符之后按回车。没有重定向`stdout`的话，这段程序就会直接告诉你你键入的字符是什么。重定向了的话，就会把你键入的那个字符写进重定向的目标里。

没错，即使完全没有使用`stdin`，你依然可以读取用户输入。

这也就是说，利用`/dev/tty`，你还可以允许用户重定向你的程序的`stdin`。你完全可以写出能在管道中使用的终端图形程序。想象一下，有一个可以在管道中使用的文本编辑器，从`stdin`中获取文本，之后让用户通过终端编辑之，再把编辑过后的文本传给`stdout`。也就是交互版的`sed`。

上面的程序需要你在键入一个字符之后按回车是因为默认情况下，终端在执行你的程序时会有“输入缓冲”，这种情况下，终端接收到回车之后才会把得到的输入发给你的程序。`man stty`了解更多。

而`/dev/tty`也可以用来调节这一点：
```C
#include <stdio.h>
#include <termios.h>
#include <unistd.h>

int main()
{
        FILE *dev_tty = fopen("/dev/tty", "r+");
        struct termios original_termios;
        struct termios raw_termios;
        cfmakeraw(&raw_termios);
        tcgetattr(fileno(dev_tty), &original_termios);
        tcsetattr(fileno(dev_tty), TCSADRAIN, &raw_termios);
        fputs("\033[2J", dev_tty); // Clear the terminal.
        fputs("\033[1;1H", dev_tty); // Move the cursor to row 1, column 1.
        fputs("hello terminal\r\nGive me a character > ", dev_tty);
        fflush(dev_tty);
        char c = fgetc(dev_tty);
        fputs("\r\n", dev_tty);
        tcsetattr(fileno(dev_tty), TCSADRAIN, &original_termios);
        fclose(dev_tty);
        fprintf(stdout, "The character I got: %c\n", c);
        return 0;
}
```
上面的程序临时修改了终端的模式。这下我们输入一个字符之后不用再回车了。`man termios`了解上面那些特殊系统调用。

好多程序是对`stdout`，`stdin`，`stderr`使用这些系统调用来调整终端模式的。`/dev/tty`比这些输入输出流更加终端，因而这些系统调用自然也能对其使用。

最后试试管道吧：
```C
#include <stdio.h>
#include <termios.h>
#include <unistd.h>

int main()
{
        char c_stdin = fgetc(stdin);
        FILE *dev_tty = fopen("/dev/tty", "r+");
        struct termios original_termios;
        struct termios raw_termios;
        cfmakeraw(&raw_termios);
        tcgetattr(fileno(dev_tty), &original_termios);
        tcsetattr(fileno(dev_tty), TCSADRAIN, &raw_termios);
        fputs("\033[2J", dev_tty); // Clear the terminal.
        fputs("\033[1;1H", dev_tty); // Move the cursor to row 1, column 1.
        fputs("hello terminal\r\nGive me a character > ", dev_tty);
        fflush(dev_tty);
        char c_tty = fgetc(dev_tty);
        fputs("\r\n", dev_tty);
        fprintf(dev_tty, "The character I got from tty: %c\r\n", c_tty);
        tcsetattr(fileno(dev_tty), TCSADRAIN, &original_termios);
        fclose(dev_tty);
        fprintf(stdout, "The character I got stdin: %c\n", c_stdin);
        return 0;
}
```
```shell
    $ printf s | ./a.out | cat > /tmp/stdout
```
