## 1. 安装依赖

首先需要安装两个依赖工具：
- [Linenoise](https://github.com/antirez/linenoise) 用于处理命令行输入
- [libelfin](https://github.com/TartanLlama/libelfin/tree/fbreg) 用于解析ELF调试信息

## 2. 创建 debugger 和 debugee
该部分实现的功能包括：
- 判断main函数传参，确保从命令行启动调试器时，同时指定可执行文件。
- 通过fork()将程序一分为二，在子进程则pid返回0，在父进程则返回子进程的pid。
- 采用传统的 fork/exec 模式，让子进程debugee接管被调试程序a.out。
- 父进程则执行debugger操作，监控用户输入命令，开启调试流程。

```
int main(int argc, char* argv[]) {
  if (argc < 2) {
    std::cerr << "Executable is not specified\n";
    return -1;
  }
  
  auto prog = argv[1];
  auto pid = fork();

  if (pid == 0) {
    // child process, debuggee
    // launch the debugee program with classic fork/exec pattern
    execute_debugee(prog);
  } else if (pid >= 1) {
    // parent process, debugger
    std::cout << "Started debugging process " << pid << '\n';
    debugger dbg{prog, pid}; // create class debugger to monit user input loop
    dbg.run();
  }
}
```

## 3. debugee 接管被调试程序 a.out
这里需要用到一个关键的系统调用 ptrace，ptrace提供了一种使父进程得以监视和控制其它进程的方式，它还能够改变子进程中的寄存器和内核映像，因而可以实现断点调试和系统调用的跟踪。使用ptrace，你可以在用户层拦截和修改系统调用(syscall)。像 gdb / lldb等调试器底层实现都是通过 ptrace，其函数原型为：

```
long ptrace(enum __ptrace_request request, pid_t pid, void *addr, void *data);
```

子进程允许被父进程跟踪 ，调用execl指定运行可执行程序：

```
void execute_debugee(const std::string &prog_name) {
  if (ptrace(PTRACE_TRACEME, 0, 0, 0) < 0) {
    std::cerr << "Error in ptrace\n";
    return;
  }
  execl(prog_name.c_str(), prog_name.c_str(), nullptr);
}
```

## 4. 命令解析辅助函数
is_prefix 函数实现了命令匹配功能，即通过输入前几个字母来匹配完整命令名，如 is_prefix(command, "continue")，可以通过输入命令 continue、con、c 来匹配 continue 命令。
split 函数以空格字符作为分隔符来获取命令行输入内容。

```
// split and is_prefix are a couple of small helper functions
std::vector<std::string> split(const std::string &s, char delimiter) {
  std::vector<std::string> out{};
  std::stringstream ss {s};
  std::string item;
 
  while (std::getline(ss, item, delimiter)) {
    out.push_back(item);
  }
 
  return out;
}

bool is_prefix(const std::string &s, const std::string &of) {
  if (s.size() > of.size())
    return false;
  return std::equal(s.begin(), s.end(), of.begin());
}
```

## 5. debugger类的成员函数
debugger类有3个成员函数：
- debugger::handle_command
- debugger::run
- debugger::continue_execution

1. debugger::handle_command 用来处理命令输入

```
void debugger::handle_command(const std::string &line) {
  auto args = split(line, ' ');
  auto command = args[0];
 
  if (is_prefix(command, "continue")) {
    continue_execution();
  } else {
    std::cerr << "Unknown command\n";
  }
}
```

2. debugger::run 程序运行，并记录历史命令。在 run 函数中，我们需要等待，直到子进程完成启动，然后一直从 linenoise 获取输入直到收到 EOF（CTRL+D）。当被跟踪的进程启动时，会发送一个 SIGTRAP 信号给它，这是一个跟踪或者断点中断。我们可以使用 waitpid 函数等待这个信号发送。

```
void debugger::run() {
  int wait_status;
  auto options = 0;
  waitpid(m_pid, &wait_status, options);
  char* line = nullptr;
 
  // get input until receive ctrl+d
  while ((line = linenoise("(xdb) ")) != nullptr) {
    handle_command(line);
    linenoiseHistoryAdd(line);
    // linenoiseFree(line);
  }
}
```

3. debugger::continue_execution 继续运行调试程序，通过 ptrace 中 PTRACE_CONT 枚举值参数实现。 continue_execution 函数会用 ptrace 告诉进程继续执行，然后用 waitpid 等待直到收到信号。

```
void debugger::continue_execution() {
  ptrace(PTRACE_CONT, m_pid, nullptr, nullptr);
  int wait_status;
  auto options = 0;
  waitpid(m_pid, &wait_status, options);
}
```

## 6. debugger类的头文件debugger.h

```
#ifndef DEBUGGER_H
#define DEBUGGER_H

#include <utility>
#include <string>
#include <linux/types.h>

namespace xdb {
  class debugger {
  public:
    debugger(std::string prog_name, pid_t pid)
	    : m_prog_name{std::move(prog_name)}, m_pid{pid} {}

    void run();

  private:
    void handle_command(const std::string &ine);
    void continue_execution();

    std::string m_prog_name;
    pid_t m_pid;
  };
}

#endif
```

## 扩展一：ptrace 介绍

通过 ptrace 可以控制另一个进程的执行状态，比如读取寄存器、读取内存、单步调试等，只需要传入一个枚举值和一些特定的参数即可调用，成功返回0，错误返回-1，errno被设置。

**枚举值 Request**

|Request参数枚举值|功能|
|---|---|
| PTRACE_TRACEME | 本进程被其父进程所跟踪。其父进程应该希望跟踪子进程。|
| PTRACE_PEEKTEXT, PTRACE_PEEKDATA | 从内存地址中读取一个字节，内存地址由addr给出。|
|PTRACE_PEEKUSR|从USER区域中读取一个字节，偏移量为addr。|
|PTRACE_POKETEXT, PTRACE_POKEDATA|往内存地址中写入一个字节。内存地址由addr给出。|
|PTRACE_POKEUSR|往USER区域中写入一个字节。偏移量为addr。|
|PTRACE_SYSCALL, PTRACE_CONT|重新运行。|
|PTRACE_KILL|杀掉子进程，使它退出。|
|PTRACE_SINGLESTEP|设置单步执行标志|
|PTRACE_ATTACH|跟踪指定pid 进程。|
|PTRACE_DETACH|结束跟踪|

Intel386特有：
|Request参数枚举值|功能|
|---|---|
|PTRACE_GETREGS|读取寄存器|
|PTRACE_SETREGS|设置寄存器|
|PTRACE_GETFPREGS|读取浮点寄存器|
|PTRACE_SETFPREGS|设置浮点寄存器 init进程不可以使用此函数|

**ptrace 功能详解**

1) PTRACE_TRACEME
形式：ptrace(PTRACE_TRACEME,0 ,0 ,0)
描述：本进程被其父进程所跟踪。其父进程应该希望跟踪子进程。

2) PTRACE_PEEKTEXT,PTRACE_PEEKDATA
形式：ptrace(PTRACE_PEEKTEXT, pid, addr, data)
描述：从内存地址中读取一个字节，pid表示被跟踪的子进程，内存地址由addr给出，data为用户变量地址用于返回读到的数据。在Linux（i386）中用户代码段与用户数据段重合所以读取代码段和数据段数据处理是一样的。

3) PTRACE_POKETEXT,PTRACE_POKEDATA
形式：ptrace(PTRACE_POKETEXT, pid, addr, data)
描述：往内存地址中写入一个字节。pid表示被跟踪的子进程，内存地址由addr给出，data为所要写入的数据。

4) TRACE_PEEKUSR
形式：ptrace(PTRACE_PEEKUSR, pid, addr, data)
描述：从USER区域中读取一个字节，pid表示被跟踪的子进程，USER区域地址由addr给出，data为用户变量地址用于返回读到的数据。USER结构为core文件的前面一部分，它描述了进程中止时的一些状态，如：寄存器值，代码、数据段大小，代码、数据段开始地址等。在Linux（i386）中通过PTRACE_PEEKUSER和PTRACE_POKEUSR可以访问USER结构的数据有寄存器和调试寄存器。

5) PTRACE_POKEUSR
形式：ptrace(PTRACE_POKEUSR, pid, addr, data)
描述：往USER区域中写入一个字节，pid表示被跟踪的子进程，USER区域地址由addr给出，data为需写入的数据。

6) PTRACE_CONT
形式：ptrace(PTRACE_CONT, pid, 0, signal)
描述：继续执行。pid表示被跟踪的子进程，signal为0则忽略引起调试进程中止的信号，若不为0则继续处理信号signal。

7) PTRACE_SYSCALL
形式：ptrace(PTRACE_SYS, pid, 0, signal)
描述：继续执行。pid表示被跟踪的子进程，signal为0则忽略引起调试进程中止的信号，若不为0则继续处理信号signal。与PTRACE_CONT不同的是进行系统调用跟踪。在被跟踪进程继续运行直到调用系统调用开始或结束时，被跟踪进程被中止，并通知父进程。

8) PTRACE_KILL
形式：ptrace(PTRACE_KILL,pid)
描述：杀掉子进程，使它退出。pid表示被跟踪的子进程。

9) PTRACE_SINGLESTEP
形式：ptrace(PTRACE_KILL, pid, 0, signle)
描述：设置单步执行标志，单步执行一条指令。pid表示被跟踪的子进程。signal为0则忽略引起调试进程中止的信号，若不为0则继续处理信号signal。当被跟踪进程单步执行完一个指令后，被跟踪进程被中止，并通知父进程。

10) PTRACE_ATTACH
形式：ptrace(PTRACE_ATTACH,pid)
描述：跟踪指定pid 进程。pid表示被跟踪进程。被跟踪进程将成为当前进程的子进程，并进入中止状态。

11) PTRACE_DETACH 形式：ptrace(PTRACE_DETACH,pid) 描述：结束跟踪。 pid表示被跟踪的子进程。结束跟踪后被跟踪进程将继续执行。

12) PTRACE_GETREGS
形式：ptrace(PTRACE_GETREGS, pid, 0, data)
描述：读取寄存器值，pid表示被跟踪的子进程，data为用户变量地址用于返回读到的数据。此功能将读取所有17个基本寄存器的值。

13) PTRACE_SETREGS
形式：ptrace(PTRACE_SETREGS, pid, 0, data)
描述：设置寄存器值，pid表示被跟踪的子进程，data为用户数据地址。此功能将设置所有17个基本寄存器的值。

14) PTRACE_GETFPREGS 形式：ptrace(PTRACE_GETFPREGS, pid, 0, data)
描述：读取浮点寄存器值，pid表示被跟踪的子进程，data为用户变量地址用于返回读到的数据。此功能将读取所有浮点协处理器387的所有寄存器的值。

15) PTRACE_SETFPREGS
形式：ptrace(PTRACE_SETREGS, pid, 0, data)
描述：设置浮点寄存器值，pid表示被跟踪的子进程，data为用户数据地址。此功能将设置所有浮点协处理器387的所有寄存器的值。

## 扩展二：waitpid 介绍

waitpid 会暂时停止目前进程的执行, 直到有信号来到或子进程结束，使用场景主要以下两种情况：

- 情况一，当用 fork 创建子进程的时候，子进程就有了新的生命周期，并将在其自己的地址空间内独立运行。如果想要知道某个创建的子进程何时结束，从而方便父进程做一些处理动作，这个时候就需要用到 waitpid 函数。
- 情况二，当用 ptrace 去 attach 一个进程，那个被 attach 的进程也可以算作那个 attach 它进程的子进程，这种情况下，想知道被调试的进程何时停止运行，也要用到 waitpid 函数。

以上两种情况下，都可以使用 Linux 中的 waitpid()函数做到。
waitpid 函数的定义：

```
#include <sys/types.h> 
#include <sys/wait.h>
pid_t waitpid(pid_t pid,int *status,int options);
```

如果在调用waitpid()函数时，当指定等待的子进程已经停止运行或结束了，则waitpid()会立即返回；但是如果子进程还没有停止运行或结束，则调用waitpid()函数的父进程则会被阻塞，暂停运行。
