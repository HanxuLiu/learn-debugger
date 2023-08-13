
## 显示可执行程序信息

**示例代码 bubble_sort.c**：

```
#include <stdio.h>

void BubbleSort(int *arr, int size) {
  int i, j, tmp;
  for (i = 0; i < size - 1; i++) {
    for (j = 0; j < size - i - 1; j++) {
      if (arr[j] > arr[j + 1])
      {
        tmp = arr[j];
        arr[j] = arr[j + 1];
        arr[j + 1] = tmp;
      }
    }
  }
}

int main(int argc, char* argv[]) {
  int a[5] = { 43, 56, 78, 23, 43 };
  printf("\n排序之前输出：\n");
  for (int i = 0; i < 5; i++)
    printf("%5d", a[i]);
  printf("\n");
  BubbleSort(a, 5);
  printf("\n排序之后输出：\n");
  for (int i = 0; i < 5; i++)
    printf("%5d", a[i]);
  printf("\n\n");
  return 0;
}
```

### 1. 显示源代码的几种方法
使用`list`（简写为l）命令来显示源代码：
####  1.1 显示源代码`(list)`
```
(gdb) l
3	void BubbleSort(int *arr, int size) {
4	  int i, j, tmp;
5	  for (i = 0; i < size - 1; i++) {
6	    for (j = 0; j < size - i - 1; j++) {
7	      if (arr[j] > arr[j + 1])
8	      {
9	        tmp = arr[j];
10	        arr[j] = arr[j + 1];
11	        arr[j + 1] = tmp;
12	      }
(gdb)
```
####  1.2 显示指定行号`(list num)`
```
(gdb) l 16
11	        arr[j + 1] = tmp;
12	      }
13	    }
14	  }
15	}
16	
17	int main(int argc, char* argv[]) {
18	  int a[5] = { 43, 56, 78, 23, 43 };
19	  printf("\n排序之前输出：\n");
20	  for (int i = 0; i < 5; i++)
(gdb)
```
####  1.3 显示指定函数`(list func)`
```
(gdb) l main
12	      }
13	    }
14	  }
15	}
16	
17	int main(int argc, char* argv[]) {
18	  int a[5] = { 43, 56, 78, 23, 43 };
19	  printf("\n排序之前输出：\n");
20	  for (int i = 0; i < 5; i++)
21	    printf("%5d", a[i]);
(gdb)
```
####  1.4 指定向前或向后显示`(list +/-)`
```
(gdb) l +
22	  printf("\n");
23	  BubbleSort(a, 5);
24	  printf("\n排序之后输出：\n");
25	  for (int i = 0; i < 5; i++)
26	    printf("%5d", a[i]);
27	  printf("\n\n");
28	  return 0;
29	}
(gdb) l -
12	      }
13	    }
14	  }
15	}
16	
17	int main(int argc, char* argv[]) {
18	  int a[5] = { 43, 56, 78, 23, 43 };
19	  printf("\n排序之前输出：\n");
20	  for (int i = 0; i < 5; i++)
21	    printf("%5d", a[i]);
(gdb)
```
####  1.5 显示指定范围`(list x,y)`
```
(gdb) l 9,11
9	        tmp = arr[j];
10	        arr[j] = arr[j + 1];
11	        arr[j + 1] = tmp;
(gdb)
```

### 2. 显示可执行程序中所有函数
####  2.1 显示所有函数`(info functions)`
使用`info functions`命令可以列出可执行文件的所有函数名称，gdb会列出可执行文件中的函数原型以及系统库中不带调试信息的函数：
```
(gdb) info functions 
All defined functions:

File bubble_sort.c:
3:	void BubbleSort(int *, int);
17:	int main(int, char **);

Non-debugging symbols:
0x0000000000001000  _init
0x0000000000001070  __cxa_finalize@plt
0x0000000000001080  putchar@plt
0x0000000000001090  puts@plt
0x00000000000010a0  __stack_chk_fail@plt
0x00000000000010b0  printf@plt
0x00000000000010c0  _start
0x00000000000010f0  deregister_tm_clones
0x0000000000001120  register_tm_clones
0x0000000000001160  __do_global_dtors_aux
0x00000000000011a0  frame_dummy
0x00000000000013a0  __libc_csu_init
0x0000000000001410  __libc_csu_fini
0x0000000000001418  _fini
(gdb)
```
####  2.2 显示筛选后的函数`(info functions regex)`
可以使用正则表达式`info functions regex`列出符合筛选要求的函数名称：
```
(gdb) info functions put*
All functions matching regular expression "put*":

Non-debugging symbols:
0x0000000000001080  putchar@plt
0x0000000000001090  puts@plt
(gdb) 
```

