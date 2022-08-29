# NEMU-SDB完善

## sdb_mainloop详解

我们知道，在程序执行到sdb_mainloop函数中后:

```c
void sdb_mainloop() {
   //如果是is_batch_mode模式，则直接调用cmd_c函数
  if (is_batch_mode) {
    cmd_c(NULL);
    return;
  }
	//主循环
  for (char *str; (str = rl_gets()) != NULL; ) {//等待接收输入
      //解析输入
    char *str_end = str + strlen(str);

    /* extract the first token as the command */
    char *cmd = strtok(str, " ");
    if (cmd == NULL) { continue; }

    /* treat the remaining string as the arguments,
     * which may need further parsing
     */
    char *args = cmd + strlen(cmd) + 1;
    if (args >= str_end) {
      args = NULL;
    }

#ifdef CONFIG_DEVICE
    extern void sdl_clear_event_queue();
    sdl_clear_event_queue();
#endif
	//根据解析调用相应的函数
    int i;
    for (i = 0; i < NR_CMD; i ++) {
      if (strcmp(cmd, cmd_table[i].name) == 0) {
        if (cmd_table[i].handler(args) < 0) { return; }
        break;
      }
    }

    if (i == NR_CMD) { printf("Unknown command '%s'\n", cmd); }
  }
}
```

###  这个函数的主要逻辑是：

1.for循环等待接收到用户的输入，

2.接着解析用户的输入，

3.然后根据解析调用响应的处理函数来执行。

4.循环，直到满足循环停止的条件。



### 对于第一步：

我们陷入for循环中，等待用户输入，会调用rl_gets函数，其中又调用readline函数获得用户的输入：

```c
static char* rl_gets() {
  static char *line_read = NULL;

  if (line_read) {
    free(line_read);//先释放
    line_read = NULL;//再设置为NULL
  }

  line_read = readline("(nemu) ");//调用readline获得用户输入

  if (line_read && *line_read) {
    add_history(line_read);
  }

  return line_read;
}
```

### 对于第二步：

接着就是解析获得到的用户输入：(代码为for循环中的前面的一部分)

```c
for (char *str; (str = rl_gets()) != NULL; ) {
  char *str_end = str + strlen(str);

  /* extract the first token as the command */
  char *cmd = strtok(str, " ");
  if (cmd == NULL) { continue; }

  /* treat the remaining string as the arguments,
   * which may need further parsing
   */
  char *args = cmd + strlen(cmd) + 1;
  if (args >= str_end) {
    args = NULL;
  }
```

其中调用了strtok函数，此函式的使用细节比较多：

#### strtok函数原型：

```char *strtok(char *str, const char *delim);```

函数的作用是：将一个字符串分解成一个由零个或多个非空标记组成的序列。

**1.其实不是分解字符串，而是先将要分解的字符串传入str，根据指定的delim，将delim替换成"\0"(字符串结束标识符)**



2.在对strtok（）的**第一次**调用中，应该在str中指定要分析的字符串。在应该分析相同字符串的每个后续调用中，**str必须为NULL**。这是因为在strtok第一个参数为NULL的时候，该函数默认使用上一次未分割完的字符串的未分割的起始位置作为本次分割的起始位置，直到分割结束为止。



3.对strtok（）的每次调用都会返回一个指针，指向一个以null结尾的字符串，如果最后字符串为空了（没有符合的情况了）则返回NULL。

还有一些细节，看手册吧。



### 对于第三步：

直接strcmp比对，如果相同，则调用相应的函数就好。(代码为for循环中的后面的一部分)

```c
int i;
  for (i = 0; i < NR_CMD; i ++) {
    if (strcmp(cmd, cmd_table[i].name) == 0) {
      if (cmd_table[i].handler(args) < 0) { return; }
      break;
    }
  }

  if (i == NR_CMD) { printf("Unknown command '%s'\n", cmd); }
}
```



## 完成相应的SDB功能

有了上面的基本解析，加上上篇对TRM的代码的浅析。应该不难写出如下的需求：

### 1.单步执行

直接调用cpu_exec函数就好啦。上一篇中有提到过这个函数。

### 2.打印寄存器

上一篇中也介绍了寄存器是由一个结构体定义的：

```c
typedef struct {
  struct {
    rtlreg_t _32;
  } gpr[32];

  vaddr_t pc;
} riscv32_CPU_state;
```

但是不同指令集架构寄存器肯定是不同的，所以代码直接提供了isa_reg_display函数，抽象了逻辑。

教程中也有提到过：

> `init_isa()`的第二项任务是初始化寄存器, 这是通过`restart()`函数来实现的. 在CPU中, 寄存器是一个结构化特征较强的存储部件, 在C语言中我们就很自然地使用相应的结构体来描述CPU的寄存器结构. 不同ISA的寄存器结构也各不相同, 为此我们把寄存器结构体`CPU_state`的定义放在`nemu/src/isa/$ISA/include/isa-def.h`中, 并在`nemu/src/cpu/cpu-exec.c`中**定义一个全局变量`cpu`**. 初始化寄存器的一个重要工作就是设置`cpu.pc`的初值, 我们需要将它设置成刚才加载客户程序的内存位置, 这样就可以让CPU从我们约定的内存位置开始执行客户程序了. 对于mips32和riscv32, 它们的0号寄存器总是存放`0`, 因此我们也需要对其进行初始化

所以直接打印就好啦。



### 3.扫描内存

上一篇中也有介绍，我文章中没写，教程中有：

> 内存通过在`nemu/src/memory/paddr.c`中定义的大数组`pmem`来模拟. 在客户程序运行的过程中, 总是使用`vaddr_read()`和`vaddr_write()` (在`nemu/src/memory/vaddr.c`中定义)来访问模拟的内存. vaddr, paddr分别代表虚拟地址和物理地址. 这些概念在将来才会用到, 目前不必深究, 但从现在开始保持接口的一致性可以在将来避免一些不必要的麻烦.

目前阶段还不需要计算表达式，所以也是直接调用vaddr_read就好啦。

下面讲如何

#### 计算表达式：

1. 首先识别出表达式中的单元
2. 根据表达式的归纳定义进行递归求值

第一步识别出表达式中的单元，主要是make_token函数：

##### make_token

```c
typedef struct token {
  int type;     //存储token的类型
  char str[32]; //存储token的值,注意str长度有限,防止缓冲区溢出.
} Token;

//tokens 数组是用于按顺序存放已经被识别出的token信息
static Token tokens[32] __attribute__((used)) = {};
//nr_token是已经被识别出的token数量
static int nr_token __attribute__((used))  = 0;

static bool make_token(char *e) {
  int position = 0;
  int i;
  regmatch_t pmatch;

  nr_token = 0;

  while (e[position] != '\0') {
    /* Try all rules one by one. */
    for (i = 0; i < NR_REGEX; i ++) {
        // 把字符串逐个识别成token，存到pmatch
      if (regexec(&re[i], e + position, 1, &pmatch, 0) == 0 && pmatch.rm_so == 0) {
        char *substr_start = e + position;  // 把token对应的起始字符串地址存入substr_start
        int substr_len = pmatch.rm_eo;  // 把token长度存入substr_len

        Log("match rules[%d] = \"%s\" at position %d with len %d: %.*s",
            i, rules[i].regex, position, substr_len, substr_len, substr_start);

        position += substr_len;

        /* TODO: Now a new token is recognized with rules[i]. Add codes
         * to record the token in the array `tokens'. For certain types
         * of tokens, some extra actions should be performed.
         */

        switch (rules[i].token_type) {
            case TK_NOTYPE:
                break;
            case TK_EQ:
                break;
            case TK_NUM:
                tokens[nr_token].type = rules[i].token_type;
                if (substr_len>30)
                    substr_len=30;
                char temp=substr_start[substr_len];
                substr_start[substr_len]='\0';
                strcpy(tokens[nr_token].str,substr_start);
                substr_start[substr_len]=temp;
                nr_token++;
                break;
            default:
                tokens[nr_token++].type=rules[i].token_type;
        }
        break;
      }
    }

    if (i == NR_REGEX) {
      printf("no match at position %d\n%s\n%*.s^\n", position, e, position, "");
      return false;
    }
  }

  return true;
}
```

1.在使用make_token函数之前，要先定义好通过正则表达式匹配的规则(通过regcomp)，

2.然后就是调用regexec进行匹配，

3.最后就是将识别出的信息存在Token结构体中



1.首先，我们要自己将想要正则匹配的规则给实现了，即rules数组要写好。比如：

##### rules

```c
#include <regex.h>

enum {
  TK_NOTYPE = 256, TK_EQ, TK_NUM

  /* TODO: Add more token types */

};

#define NR_REGEX ARRLEN(rules)

static regex_t re[NR_REGEX] = {};

static struct rule {
  const char *regex;
  int token_type;
} rules[] = {

  /* TODO: Add more rules.
   * Pay attention to the precedence level of different rules.
   */

  {" +", TK_NOTYPE},    // spaces
  {"\\+", '+'},         // plus
  {"==", TK_EQ},        // equal
  {"[0-9]+", TK_NUM},
  {"\\-", '-'},
  {"\\*", '*'},
  {"\\/", '/'},
  {"\\(", '('},
  {"\\)",')'},
};
```

这里就是我们写了我们要识别的匹配规则。左边是正则表达式regex 用的是现成的regex库，右边是token_type 自己定义的枚举类型。

然后，框架代码会调用init_regex函数：main->init_monitor->init_sdb->init_regex

##### init_regex

```c
void init_regex() {
  int i;
  char error_msg[128];
  int ret;

  for (i = 0; i < NR_REGEX; i ++) {
      //int regcomp(regex_t *preg, const char *regx, int cflags);
      //将 regx 转为 regex_t 且将结果存在 preg 中 成功则返回 0
    ret = regcomp(&re[i], rules[i].regex, REG_EXTENDED);
    if (ret != 0) {
      regerror(ret, &re[i], error_msg, 128);
      panic("regex compilation failed: %s\n%s", error_msg, rules[i].regex);
    }
  }
}
```

其中会使用:

##### regcomp

```int regcomp(regex_t *preg, const char *regex, int cflags);```

regcomp用来编译文本regex生成匹配规则，执行结果保存在`regex_t`结构的`preg`参数中，执行成功时返回0。说白了就是将我们自己定义的结构体类型rules变为regex_t类型。



2.然后我们再回过头来看make_token函数，其中调用了regexec函数

##### regexec

```c
int regexec(const regex_t *preg, const char *string, size_t nmatch,
           regmatch_t pmatch[], int eflags);
```

`regexec`将匹配规则re[i]与目标文本e+position进行匹配，并将匹配结果保存在`regmatch_t`的结构数组`pmatch[]`中。

`regmatch_t`的结构如下：

```c
typedef struct {
    regoff_t rm_so;
    regoff_t rm_eo;
} regmatch_t;
```

`rm_so`,`rm_eo`分别记录了匹配成功字段的起始（第一个匹配字符）和结束点（结尾的下一个字符）的偏移量。



3.将识别出的信息存在tokens中，就很简单了，就是switch case那部分代码中要做的事情。

我们这里比如，会直接省略输入的空格符，所以：

```c
switch (rules[i].token_type) {
	case TK_EQ:
    	break;
		...
}
```

这些就按照需求来写就行了。



#### 递归求值

将表达式中的token识别出来后，我们就需要计算了。识别出的结果都存在了tokens数组中，所以递归求值的入参肯定就是数组啦。



token数组如下：

```
"4 +3*(2- 1)"
```

的token表达式为

```
+-----+-----+-----+-----+-----+-----+-----+-----+-----+
| NUM | '+' | NUM | '*' | '(' | NUM | '-' | NUM | ')' |
| "4" |     | "3" |     |     | "2" |     | "1" |     |
+-----+-----+-----+-----+-----+-----+-----+-----+-----+
```



递归求值的思路：(这部分是算法的内容啦，和计算机体系课没啥关系)

任何表达式，其实都可以看成是由一个个子表达式组成。所以我们呢自然就想到了分治法，自然就想到用递归啦。



那么，关键问题就是**将一个表达式按照什么样的规则进行子表达式的划分啦**。

具体的思路就看教程吧，写的很详细了，这里就不写啦。

