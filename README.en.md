# `/dev/tty`

[中文](README.md)

Many popular terminal applications actually have one same bug:
```
   fzf 2> /dev/null
```
You can't see what `fzf` gives you to pick from any more. Press Ctrl-C and escape.
```
   vi > /dev/null
```
Unless you're some monstrosity that writes `alias vi=vscode` in your shell's config, what you get is likely this: your shell no longer responds to you. Ctrl-C doesn't help you either.

If you have a good intuition, you might be able to realize that you can `:wq` then press enter to quit. If you're some "I act before I think" shoujo manga heroine, or some "I act before I think" shounen manga hero, you're done. Your nonsense input have already forced your text editor into an unknown state, and you can't know what that state is, because its output is redirected. You can't get this shell back.

Some people might argue that this is not a bug. Indeed, people normally won't use these applications this way. Whether this counts as a bug depends on how you define a bug. The fact is, you use `vi` this way, your shell is sabotaged. I don't like this.

If you have some knowledge on terminals, you know what the problem is already. If you don't, let me explain this to you.

The programs that make fancy drawings on you terminal such as `vi` or `fzf` don't use magic to control you terminal. They send *terminal escape codes* to your terminal, in order to tell it what they want it to do. They can also receive some terminal escape codes from the terminal to get some info.

Let's write some C.
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
All code in this article could sabotage your terminal, so be prepared to reopen it.

Compile and execute. You terminal clears the text on it, and then starts to print "hello world" from row 1 column 1. You shell prints the prompt as usual on your terminal after this small program's done.

The format of terminal escape codes is not standardized. Well, it is ([ECMA-48](https://www.ecma-international.org/publications-and-standards/standards/ecma-48/)), but no implementation of terminals follows it. On Linux, one convenient way of learning those is `man console_codes`. This command tells you what codes the Linux console, one implementation of a terminal by Linux itself, supports. Most other terminals support most of these codes. Press Ctrl-Alt-F3 on Linux, what you see is a Linux console. If these codes are not enough for you, [here](https://invisible-island.net/xterm/ctlseqs/ctlseqs.html) are many other codes. Some codes are supported by some terminals. If you want to ensure that a functionality can be realized, the only way is to try it on a terminal that you want to support.

So, what happens if you run this small program redirected?
```shell
    $ ./a.out > /tmp/codes # Assuming the name of the executable is a.out.
```
Nothing happens to your terminal.
```shell
    $ cat /tmp/codes
```
Printing out this file does what executing the program did before.

Now we know what caused the problems we saw on `fzf` and `vi`. They control the terminals by sending the escape codes. `fzf` sends the codes to `stderr`. `vi` sends the codes to `stdout`. If `stderr` and `stdout` happen to be terminals, those applications can be used normally. The 2 streams can be redirected however. If they are redirected to files, the escape codes are written to those files. The terminal never receives those codes, and therefore won't draw following the programs' instructions.

One might argue that one can use `isatty` to decide whether the stream is a terminal, and just exit if it's not. This doesn't solve the problem completely. Some really evil people can even redirect the streams to other terminals (`man pts`). Some programs ask `stderr` what the terminal size is or where the cursor is, and gets the result from `stdin`. One can connect `stderr` and `stdin` to different terminals, and send fake responses to the program, and let the program make some edits at (200, 200) on a buffer representing a 20x20 terminal.

Is this problem solvable then? That's a problem that is not solved by those popular software. Does this mean there're no solutions? The answer is yes, theoretically. Practically? How do I know, I don't maintain those software.

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
Unix-like operating systems give a process a special file if it's connected to a terminal `/dev/tty`. This file *is* the terminal. Therefore, redirecting standard IO streams won't affect IO with this file.

There is no user-friendly documentation for `/dev/tty`. You get some documentation by `man 4 tty`.

Compile and run these several lines of code. If you simply run it, it's not very different from the "hello world" program we tried before. It's just that the output text is different. However, if you redirect the `stdout` of this program, you'll notice that what's sent to `/dev/tty` still gets to your terminal, and what's written to you redirection's target file, is just the line of text "hello stdout".

`/dev/tty` is not just an output handle. It can also be used for input.
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
Compile and run. Press enter after inserting one character. If you don't redirect `stdout`, this program directly tells you what that character is. If you redirected it, it write what you've inserted to the redirection's target.

Yes, even if you've never used `stdin`, you can still get user input.

This also means, by using `/dev/tty`, you also allow your user to redirect your program's `stdin`. You can write a program that is totally usable in a pipe. Imagine having a text editor that can be used in a pipe. It reads from `stdin`, lets the user edit it from the terminal, and then sends the result to `stdout`. That's interactive `sed`.

The program above needs you to press enter after inserting a character because, by default, when the terminal executes your program, there is an "input buffer". In this situation, the terminal sends what it gets to your program only after getting an enter press. `man stty` to learn more.

`/dev/tty` can also be used to change this:
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
The program above changes the mode of the terminal temporarily. Now we no longer need to press enter after giving a character. `man termios` to learn about the system calls used above.

Lots of programs use `stdout`, `stdin` or `stderr` with those system calls. `/dev/tty` is more terminal than those streams, and therefore of course can be used with the system calls.

Let's try piping as a conclusion.
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
