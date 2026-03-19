

# 1. 关键词

## 1.1 #define和const的区别

`#define`是通过宏定义来进行定义，在代码预处理阶段会把使用define定义的语句进行文本替换。编译器永远看不到宏名，只看到替换后的文本。并且是**无类型**的，编译器不进行类型检查；而且作用域是全文件**从定义点开始到文件末尾**。

---

而`const`则是将修饰的**全局变量**写入到`.rodata`段，此段属性为只读属性因此无法对其进行更改。

如果是在函数内部使用`const`修饰的**局部变量**由于其是被写入到栈段，而栈段是可读可写的属性因此可以通过指针来更改其内容。

## 1.2 static

 在 C 语言中，`static` 主要用于控制变量或函数的**链接属性（作用域）** 和**存储期（生命周期）**。

### 1.2.1 修饰函数内变量

当 `static` 用于函数内部声明的局部变量时，它改变了变量的**存储期**。在程序开始时被创建并初始化**一次**，在程序的整个运行期间都存在，直到程序结束才被销毁。即使函数调用结束，它的值也会被保留。

``` c
/*FreeRTOS中heap.1中对堆起始地址的对齐初始化*/
void* pvPortMalloc( size_t xWantedSize ){
    /*....*/
    static uint8_t * pucAlignedHeap = NULL;
    if( pucAlignedHeap == NULL )
        {
            /* Ensure the heap starts on a correctly aligned boundary. */
            pucAlignedHeap = (uint8_t*) ( ( (portPOINTER_SIZE_TYPE) &ucHeap[portBYTE_ALIGNMENT] ) & 
                                         ( ~(portPOINTER_SIZE_TYPE) portBYTE_ALIGNMENT_MASK) );
        }
    /*...*/
}
```

### 1.2.2 修饰函数/全局变量

当 `static` 用于在文件作用域（即所有函数之外）声明的全局变量或函数时，它改变了它们的**链接属性**，从而限制了它们的**作用域**。只能在**定义它的当前源文件**中访问，对其他文件不可见。这实现了"文件作用域的封装"。

``` c
/*FreeRTOS中heap.2中定义起始链表节点与结束链表节点*/
static BlockLink_t xStart, xEnd;
static size_t xFreeBytesRemaining = configADJUSTED_HEAP_SIZE;
```

**如果外层函数未被 static 修饰，但是外层函数内调用了 static 修饰的函数，外层函数是否可被其他文件调用？**

# 2 .结构体

## 对齐与大小

先明确 3 个关键概念（结合比喻）：

1. **自身对齐值**：每个参数需要的对齐值（比如`int`的自身对齐值是 4 字节）；
2. **默认对齐值**：编译器规定的对齐值（32 位 GCC 默认 4 字节，64 位默认 8 字节，可通过`#pragma pack(n)`修改）；
3. **有效对齐值**：取 “自身对齐值” 和 “默认对齐值” 中**更小**的那个字节大小。

**规矩 1（成员起始地址规则）**：每个参数的起始地址必须是**自己的有效对齐值**的整数倍。

**规矩 2（结构体整体补齐规则）**：整个结构体的 “总房间数”，必须是**所有成员中最大的有效对齐值**的整数倍。

就像酒店要求：整个团队（结构体）住的房间总数，得是团队里 “最费房间” 那位客人的有效对齐值的整数倍，不够的话要补空房间（空房间不存数据，只为凑数）。

``` c
#include <stdio.h>

struct B {
    char a;   // 客人a：0号房（1间）
    short c;  // 客人c：有效对齐值2，起始号2（1号空），占2-3号（2间）
    int b;    // 客人b：有效对齐值4，起始号4，占4-7号（4间）
};

int main() {
    printf("struct B的大小：%zu\n", sizeof(struct B));
    return 0;
}
```

运行结果：`struct B的大小：8`。

---
下面是 FreeRTOS 中对**内存块链表结构体**（`BlockLink_t`） 的大小做**向上字节对齐**的经典实现，目的是让该结构体占用的内存大小严格等于`portBYTE_ALIGNMENT`（字节对齐边界）的整数倍，保证 FreeRTOS 堆管理中分配的内存块地址符合 CPU / 硬件的对齐要求，避免访问效率低或硬件异常。
``` c
static const uint16_t heapSTRUCT_SIZE = ( ( sizeof(BlockLink_t) + (portBYTE_ALIGNMENT - 1) ) & ~portBYTE_ALIGNMENT_MASK );
                                          // (助我破鼎) -1 是为了如果恰好是对齐的避免达到下一个边界

// portBYTE_ALIGNMENT       字节对齐的 “目标边界”（必须是 2 的幂次，如 4/8），代表内存地址 / 大小要对齐到多少字节
// portBYTE_ALIGNMENT_MASK  对齐掩码，等于portBYTE_ALIGNMENT - 1，作用是标记 “需要舍弃的低位字节”，取反后低位全是 0，只保留对齐后的整数倍 
```

## 位域

使用位域可以更加方便的使用一个变量类型中的每一个位。

``` c
#include <stdio.h>

struct Normal {
    unsigned int a;
    unsigned int b;
    unsigned int c;
};

struct BitField {
    unsigned int a:1;
    unsigned int b:1; 
    unsigned int c:1;
};

int main() {
    printf("普通结构体大小: %zu 字节\n", sizeof(struct Normal));   // 12字节
    printf("位域结构体大小: %zu 字节\n", sizeof(struct BitField));  // 4字节
    return 0;
}


struct BitField bf;
bf.streaming = 1;  // 打开
bf.error = 0;      // 关闭

if (bf.streaming) {
    printf("正在传输...\n");
}
```



# 3 .常量指针与指针常量

## 3.1 指针常量(保护指针)

指针常量（`int* const p`）：**指针本身是常量，指向的地址（指针的值）不可改**，地址上的内容可改。

``` c
char* const px;

void vQueueDelete( QueueHandle_t xQueue )
{
    Queue_t* const pxQueue = xQueue;  // 指针常量
    /*...*/
}
```
## 3.2 常量指针(保护数据)

常量指针（`const int *p` 或 `int const *p`）：**指向的内容**是常量内容不可改，指针本身的值可改（可以指向其他地址）。

``` c
const char *px;
```

# 4. 函数指针

- **本质**：**一个指针变量**，该指针**指向函数**（存储函数的入口地址，因此可以用来进行**软重启**）。

- **语法**：
  `返回类型 (*指针变量名)  (参数列表);`
  **关键**：`*` 必须用括号 `()` 括起来，表示 "这是一个指向函数的指针"。

- 作用：

  - 实现回调函数（如 GUI 事件处理）
  - 创建函数表（如状态机）
  - 提高代码灵活性（运行时动态绑定函数）

  ```c
  #include <stdio.h>
  
  // 定义普通函数
  int add(int a, int b) { return a + b; }
  int sub(int a, int b) { return a - b; }
  
  int main() {
      // 声明函数指针：指向返回int、接受两个int参数的函数
      int (*func_ptr) (int, int); 
      
      // 赋值：指向add函数（函数名即地址）
      func_ptr = add; 
      
      // 通过指针调用函数（两种等价写法）
      printf("Result: %d\n", (*func_ptr)(3, 4)); // 写法1：显式解引用
      printf("Result: %d\n", func_ptr(3, 4));     // 写法2：编译器自动解引用
      
      // 切换指向其他函数
      func_ptr = sub;
      printf("Result: %d\n", func_ptr(5, 2));     // 输出 3
  }
  ```

``` c
// 定义了一个函数指针变量 TaskFunction_t
void (* TaskFunction_t)( void * arg );
、
  
// 定义一个类型别名TaskFunction_t
typedef void (* TaskFunction_t)( void * arg );
// 声明3个同类型指针，写法和int a,b,c;一样简单
TaskFunction_t task1, task2, task3;
```



# 5. volatile与原子操作

假设你正在调试一个嵌入式系统，你有一个在主循环中运行的全局变量 `int sensor_value;`，同时它在一个定时器中断服务程序中被更新。你发现即使传感器的读数在物理上已经稳定，但主循环中打印出的这个值，偶尔会跳变成一个完全错误的值（比如一个很大的负数）。你已经检查了传感器和通信总线，都是正常的。

---

`volatile`关键字的主要作用是告诉编译器，这个变量可能会被程序之外的因素改变（例如，硬件、中断、其他线程等）。因此，编译器在优化代码时，**不会将该变量缓存到寄存器中，而是每次使用都从内存中重新读取，每次修改都立即写回内存**。这样确保了变量的可见性，即当变量被修改后，其他代码（如中断服务程序）能够看到最新的值。

`volatile`解决了"什么时候看到数据"的问题（可见性），但没有解决"如何完整地看到数据"的问题（原子性）。在嵌入式并发编程中，这两者都是必需的，但解决的是不同维度的问题。

这就是为什么一个有经验的嵌入式工程师听到"全局变量在中断中被修改"时，会立即想到两个必备措施：`volatile` + 原子性保护。

# 字符串相关

## `string`系列函数

C 语言没有原生的 “字符串类型”，`string.h`系列函数（`strcpy/strncpy/strlen/strcat` 等）操作的核心是：以`'\0'`（空字符，ASCII 值 0）为结束标志的**连续字符数组**（也称 C 风格字符串）。

关键点：

- **内存操作本质**：这些函数本质上是在内存块上操作，通过**指针访问**。
- **依赖空终止符**：几乎所有函数都**依赖'\0'来确定字符串结尾**。
- **无边界检查**：传统string函数不验证目标缓冲区大小。

---

`strcpy()`与`strncpy()`函数：

``` c
char *strcpy(char *dest, const char *src);
```
- 无长度限制：从`src`复制字符到`dest`，直到遇到'\0'。

- 不检查目标缓冲区大小。

- 如果`src`长度超过`dest`大小，必定缓冲区溢出。

``` c
char *strncpy(char *dest, const char *src, size_t n);
```

- **有长度限制**：最多复制n个字符。
- **行为复杂**：
  - 如果src长度≥n：复制前n个字符，**不会自动添加'\0'**。
  - 如果src长度<n：复制src所有字符+'\0'，剩余部分用'\0'填充。

因此`strncpy()`比`strcpy()`更加安全：

1. **长度限制**：可以**防止无限制复制**导致的缓冲区溢出。
2. **预防措施**：开发者在调用时需要显式指定最大复制长度，这促使思考缓冲区大小。

但是并不保证绝对安全：

1. 不保证目标字符串以'\0'结尾并且截断无提示：

``` c
char buf[8];
strncpy(buf, "12345678", 8); // 没有空间放'\0'！
// buf现在不是合法的C字符串，后续strlen等函数会越界访问

char buf [4];
strncpy(buf, "hello", 4);
buf[3] = '\0';  // 必须手动添加！很多开发者忘记这一步
// 字符串被静默截断，可能导致逻辑错误
```

2. 性能问题（在嵌入式系统中重要）:

``` c
char buf[1024];
strncpy(buf, "short", 1024);
// 填充999+个'\0'，在实时系统中可能造成不可接受的延迟
```

# 作用域

作用域通过`{}`来进行划分，如果嵌套作用域有相同命名的变量，则会有**覆盖**的效果：在内层作用域中无法访问外层作用域的同名变量。
