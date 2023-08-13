
## 配置.gdbinit

gdb 启动时，会默认读取用户主目录和当前目录下的的配置文件，并执行里面的命令。这个文件通常为`.gdbinit`。

### 1.  保存历史命令`(set history save on)`
gdb 默认不保存历史命令，但可以使用如下命令设置成保存历史命令：
```
(gdb) set history save on
```
历史命令默认保存在当前目录下的`.gdb_history`文件中，可以通过如下命令来设置要保存的文件名和路径：
```
(gdb) set history filename fname
```
现在把这两个命令写到用户主目录下`.gdbinit`文件中：
```
set history filename ~/.gdb_history
set history save on
```
这样下次启动 gdb 时，就可以直接查找历史命令了。

### 2.  退出时不显示提示信息`(set confirm off)`
gdb 在退出时会提示：
```
A debugging session is active.
	Inferior 1 [process 3797] will be killed.
Quit anyway? (y or n)
```
如果不想显示这个信息，则可以在gdb中使用如下命令把提示信息关掉：
```
(gdb) set confirm off
```
可以把这个命令写到用户主目录下`.gdbinit`文件中。

### 3. 输出信息较多时不暂停打印`(set pagination off)`
当 gdb 输出信息较多时，gdb 会暂停输出，并会打印“ ---Type <return> to continue, or q <return> to quit--- ”这样的提示信息，如下面所示：
```
81 process 2639102 0xff04af84 in __lwp_park () from /usr/lib/libc.so.1
80 process 2573566 0xff04af84 in __lwp_park () from /usr/lib/libc.so.1
---Type <return> to continue, or q <return> to quit---Quit
```
可以使用如下命令开启全部打印模式：
```
(gdb) set pagination off 
或者
(gdb) set height 0
```
这样 gdb 就会全部输出，中间不会暂停。
同样，可以把该命令写到用户主目录下`.gdbinit`文件中。

### 4. 关闭 gdb 启动时提示信息`(alias gdb="gdb -q")`
gdb 在启动时会显示如下一些提示信息：
```
GNU gdb (Ubuntu 9.2-0ubuntu1~20.04.1) 9.2
Copyright (C) 2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
(gdb)
```
如果不想显示这个信息，则可以使用 -q 选项把提示信息关掉:
```
$ gdb -q
(gdb)
```
更好的方法是：在`~/.bashrc`中，为 gdb 设置一个别名：
```
alias gdb="gdb -q"
```



