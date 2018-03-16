# 概念

### UID

登陆终端的用户ID，使用echo $UID查看

### RUID
这个概念针对进程，指启动进程的真实用户ID

### EUID
这个概念针对进程，当进程有读写文件操作时，使用EUID进行权限判断，一般情况下二者是相等的，在使用了Set Uid后，二者可能会不同。


# 验证euid的改变
代码如下
```
#include <sys/types.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
 
int main(void)
{
    printf ("The real user ID is: %d\n", getuid());
    printf ("The effective user ID is :%d\n", geteuid());
    return (0);
}
```

在root用户下编译生成 a.out，
```
-rwxr-xr-x 1 root root 8657 3月   3 19:21 a.out
-rw-r--r-- 1 root root  170 3月   3 19:21 euid.c
```

在root下执行a.out
```
[root@gaochuan euid]# ./a.out
RUID 0
EUID 0
```
在gaochuan下执行a.out
```
[gaochuan@gaochuan euid]$ ./a.out
RUID 1004
EUID 1004
[gaochuan@gaochuan euid]$ echo $UID
1004
```
在root下使用chmod u+s 改变a.out的save uid标志，这个标志会让程序以它的owner的uid做为EUID而不是调用者自己的uid
```
[root@gaochuan euid]# chmod u+s a.out
[root@gaochuan euid]# ll
总用量 16
-rwsr-xr-x 1 root root 8657 3月   3 19:21 a.out
-rw-r--r-- 1 root root  170 3月   3 19:21 euid.c
```
可以发现a.out的属性变成了rws,多了s标志，切到gaochuan,再执行a.out
```
[root@gaochuan euid]# su gaochuan
[gaochuan@gaochuan euid]$ ./a.out
RUID 1004
EUID 0
```
发现RUID，即real userid 还是gaochuan的uid，但是euid已经变成root的uid了，即a.out的owner的uid了

### 验证EUID的保护作用

```
#include<sys/types.h>
#include<unistd.h>
#include<stdio.h>
#include<stdlib.h>
int main()
{
 FILE *fp;
 char ch;
 printf("RUID %d\n",getuid());
 printf("EUID %d\n",geteuid());
 fp = fopen("wo","rw");
 ch = fgetc(fp);
 while(ch != EOF)
 {
    putchar(ch);
    ch = fgetc(fp);
 }
 return (0);
}
```
编译生成a.out,在当前目录新建一个wo文件
```
-rwxr-x--x 1 root root 8812 3月   3 19:38 a.out
-rw-r--r-- 1 root root  298 3月   3 19:38 euid.c
-r--r--r-- 1 root root    4 3月   3 19:36 wo
```
在以上情况下，在root和gaochuan用户下都可以执行a.out并顺利读取到wo文件。

用gaochuan执行a.out时读取wo文件时，euid是gaochuan，对wo文件具有读权限，所以可以成功。

把wo文件的other的r去掉
```
[root@gaochuan euid]# chmod o-r wo
[root@gaochuan euid]# ll
总用量 20
-rwxr-x--x 1 root root 8812 3月   3 19:38 a.out
-rw-r--r-- 1 root root  298 3月   3 19:38 euid.c
-r--r----- 1 root root    4 3月   3 19:36 wo
```

这次root可以执行，但是gaochuan却无法执行了，因为gaochuan没有读属性了
```
[gaochuan@gaochuan euid]$ ./a.out
RUID 1004
EUID 1004
段错误
```

这种情况下要么改变wo文件的属性，要么对a.out使用set uid属性，`chmod u+s a.out`，这么设置之后，不管用户是谁，一律使用它的onwer属性做为euid，在本例中，即root，所以这样它就有读的权限了
```
[root@gaochuan euid]# chmod u+s a.out
[root@gaochuan euid]# su gaochuan
[gaochuan@gaochuan euid]$ ll
总用量 20
-rwsr-x--x 1 root root 8812 3月   3 19:38 a.out
-rw-r--r-- 1 root root  298 3月   3 19:38 euid.c
-r--r----- 1 root root    4 3月   3 19:36 wo
[gaochuan@gaochuan euid]$ ./a.out
RUID 1004
EUID 0
woa
```

# 说明

在使用其它用户的可执行程序时，该程序的所有权限都是你的权限，这个权限是会传递的。如果想打破这个传递就使用s标志位


