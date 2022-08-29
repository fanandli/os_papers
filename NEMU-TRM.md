# NEMU--TRM实现浅析

## 代码结构

主要有4个模块 monitor cpu memory device。

### 看一下代码结构树：

```
nemu
├── configs                    # 预先提供的一些配置文件
├── include                    # 存放全局使用的头文件
│   ├── common.h               # 公用的头文件
│   ├── config                 # 配置系统生成的头文件, 用于维护配置选项更新的时间戳
│   ├── cpu
│   │   ├── cpu.h
│   │   ├── decode.h           # 译码相关
│   │   ├── difftest.h
│   │   ├── exec.h             # 执行相关
│   │   └── ifetch.h           # 取指相关
│   ├── debug.h                # 一些方便调试用的宏
│   ├── device                 # 设备相关
│   ├── difftest-def.h
│   ├── generated
│   │   └── autoconf.h         # 配置系统生成的头文件, 用于根据配置信息定义相关的宏
│   ├── isa.h                  # ISA相关
│   ├── macro.h                # 一些方便的宏定义
│   ├── memory                 # 访问内存相关
│   ├── rtl
│   │   ├── pseudo.h           # RTL伪指令
│   │   └── rtl.h              # RTL指令相关定义
│   └── utils.h
├── Kconfig                    # 配置信息管理的规则
├── Makefile                   # Makefile构建脚本
├── README.md
├── resource                   # 一些辅助资源
├── scripts                    # Makefile构建脚本
├── src                        # 源文件
│   ├── cpu
│   │   └── cpu-exec.c         # 指令执行的主循环
│   ├── device                 # 设备相关
│   ├── engine
│   │   └── interpreter        # 解释器的实现
│   ├── filelist.mk
│   ├── isa                    # ISA相关的实现
│   │   ├── mips32
│   │   ├── riscv32
│   │   ├── riscv64
│   │   └── x86
│   ├── memory                 # 内存访问的实现
│   ├── monitor
│   │   ├── monitor.c
│   │   └── sdb                # 简易调试器
│   │       ├── expr.c         # 表达式求值的实现
│   │       ├── sdb.c          # 简易调试器的命令处理
│   │       └── watchpoint.c   # 监视点的实现
│   ├── nemu-main.c            # 你知道的...
│   └── utils                  # 一些公共的功能
│       ├── log.c              # 日志文件相关
│       ├── rand.c
│       ├── state.c
│       └── timer.c
└── tools                      # 一些工具
```

主要要知道 include 是包含了所有的头文件 src是包含了所有的源代码。

你也可以确实可以看到src子文件夹有cpu device memory monitor。



### **那么我们现在就来看看源码吧！！！**

#### 入口函数nemu-main.c

```c
#include <common.h>

void init_monitor(int, char *[]);
void am_init_monitor();
void engine_start();
int is_exit_status_bad();

int main(int argc, char *argv[]) {
  /* Initialize the monitor. */
#ifdef CONFIG_TARGET_AM
  am_init_monitor();
#else
  init_monitor(argc, argv);
#endif

  /* Start engine. */
  engine_start();

  return is_exit_status_bad();
}
```

main函数代码看起来挺少的哈：

1.如果定义了CONFIG_TARGET_AM就函数调用am_init_monitor()，如果没有就调用init_monitor(argc, argv)

2.然后再调用engint_start().

3.最后回调函数is_exit_status_bad()



为啥是这个逻辑呢？

因为NEMU是一个用来执行客户程序的程序, 但客户程序一开始并不存在于客户计算机中. 我们需要将客户程序读入到客户计算机中, 这件事是monitor来负责的. 于是NEMU在开始运行的时候, 首先会调用`init_monitor()`函数(在`nemu/src/monitor/monitor.c`中定义) 来进行一些和monitor相关的初始化工作.



说明：

1.CONFIG_TARGET_AM还不知道是啥意思。只知道是：

kconfig会根据配置选项的结果在 `nemu/include/generated/autoconf.h`中定义一些形如`CONFIG_xxx`的宏, 我们可以在C代码中通过条件编译的功能对这些宏进行测试, 来判断是否编译某些代码。 

 猜测意思是如果满足某些条件编译了，那么OCNFIG_TAEGET_AMj就有定义 ，此时就需要调用am_init_monitor.否则，就运行init_monitor。

#### 2.我们来看一下init_monitor函数

```c
void init_monitor(int argc, char *argv[]) {
  /* Perform some global initialization. */

  /* Parse arguments. */
  parse_args(argc, argv);

  /* Set random seed. */
  init_rand();

  /* Open the log file. */
  init_log(log_file);

  /* Initialize memory. */
  init_mem();

  /* Initialize devices. */
  IFDEF(CONFIG_DEVICE, init_device());

  /* Perform ISA dependent initialization. */
  init_isa();

  /* Load the image to memory. This will overwrite the built-in image. */
  long img_size = load_img();

  /* Initialize differential testing. */
  init_difftest(diff_so_file, img_size, difftest_port);

  /* Initialize the simple debugger. */
  init_sdb();

  IFDEF(CONFIG_ITRACE, init_disasm(
    MUXDEF(CONFIG_ISA_x86,     "i686",
    MUXDEF(CONFIG_ISA_mips32,  "mipsel",
    MUXDEF(CONFIG_ISA_riscv32, "riscv32",
    MUXDEF(CONFIG_ISA_riscv64, "riscv64", "bad")))) "-pc-linux-gnu"
  ));

  /* Display welcome message. */
  welcome();
}
```

结构还是挺好阅读的哈：

1.首先是调用parse_args 解析参数，然后调用init_rand设置随机数种子，接着调用init_log初始化log文件；接着调用init_men初始化memory。

2.然后是一个宏。意思是：如果定义了CONFIG_DEVICE 才会调用init_device初始化device。（IFDEF宏定义在nemu/include/macro.h中）

**3.接着是调用init_isa 做一些有关于isa的初始化工作。**

4.后面还有一些代码：将镜像加载到内存中，初始化differential testing，初始化sdb，根据情况init_disasm （应该是初始化指令集架构吧）

5.最后调用welcome函数，显示欢迎信息，表示成功运行。

（我们这里主要看一下1，3两部分。第4点的代码后面会涉及，暂时不看。）



##### 看一下parse_args

```c
static int parse_args(int argc, char *argv[]) {
  const struct option table[] = {
    {"batch"    , no_argument      , NULL, 'b'},
    {"log"      , required_argument, NULL, 'l'},
    {"diff"     , required_argument, NULL, 'd'},
    {"port"     , required_argument, NULL, 'p'},
    {"help"     , no_argument      , NULL, 'h'},
    {0          , 0                , NULL,  0 },
  };
  int o;
  while ( (o = getopt_long(argc, argv, "-bhl:d:p:", table, NULL)) != -1) {
    switch (o) {
      case 'b': sdb_set_batch_mode(); break;
      case 'p': sscanf(optarg, "%d", &difftest_port); break;
      case 'l': log_file = optarg; break;
      case 'd': diff_so_file = optarg; break;
      case 1: img_file = optarg; return optind - 1;
      default:
        printf("Usage: %s [OPTION...] IMAGE [args]\n\n", argv[0]);
        printf("\t-b,--batch              run with batch mode\n");
        printf("\t-l,--log=FILE           output log to FILE\n");
        printf("\t-d,--diff=REF_SO        run DiffTest with reference REF_SO\n");
        printf("\t-p,--port=PORT          run DiffTest with port PORT\n");
        printf("\n");
        exit(0);
    }
  }
  return 0;
}
```

首先我们早就知道了这函数是用来解析参数的，那么参数是从哪来的呢?

> 从int main(int argc, char *argv[])这里传参到init_monitor(argc, argv)，然后再传参到parse_args(int argc,  char *argv[])的。那么main是从哪里接收的呢？在命令行参数中传的~举例：我在命令行中输入`./main a b c `那么此时argc就是4。argv为["./main", "a", "b", "c"]。那稍微大一点的程序，肯定是要些makefile的，不会向我们刚才说的只是输一条命令行就行了的。所以，**这个时候传参到底传的是什么就应该要去看makefile是怎么写的了(啊，我是凭感觉的，还是要学一下makefile的用法呀，不然看不懂)**。



```c
int getopt_long(int argc, char * const argv[],
                  const char *optstring,
                  const struct option *longopts, int *longindex);
```

> **getopt作用：**getopt用于分析参数。一般情况下，我们自己写一个option类型的数组，设置长短选项。然后再用swithc 匹配函数，如上面的代码逻辑。 
>
> 前两个参数就是main的参数。
>
> 后面是个选项字符串：上文中的`-bhl:d:a:`就是命令行中对应的`-b -h -l -d -a`，冒号表示这个选项后面必须带有参数，可以和选项连在一起写，也可以用空格隔开。两个冒号表示参数可选，就是可以带有参数也可以不带。
>
> longopts是指向option类型的指针。option的申明如下：
>
> struct option {
>                const char *name;
>                int         has_arg;
>                int        *flag;
>                int         val;
>            };
>
> name:长选项的名字
>
> has_arg:0,不需要参数；1，需要参数；2，参数是可选的
>
> flag: 指定如何返回长选项的结果。如果flag是null,那么getopt_long返回val(可以将val设置为等效的短选项字符)，否则getopt_long返回0.
>
> val:要返回的值。**结构体option数组的最后一个元素必须用零填充**



> optarg指向额外参数
>
> **return value：**如果选项成功找到，返回选项字母。当所有命令行被解析，则getopt返回`-1`。如果存在未知的选项或缺失选项，getopt会返回`?`。



##### 看一下init_rand

```c
void init_rand() {
  srand(MUXDEF(CONFIG_TARGET_AM, 0, time(0)));
}
```

就是调用一下srand 里面用的是MUXDEF的宏，不是很重要的细节，后面再补充吧。



##### 看一下init_log

```c
void init_log(const char *log_file) {
  log_fp = stdout;
  if (log_file != NULL) {
    FILE *fp = fopen(log_file, "w");
    Assert(fp, "Can not open '%s'", log_file);
    log_fp = fp;
  }
  Log("Log is written to %s", log_file ? log_file : "stdout");
}
```

就是一开始让log_fp成为stdout 然后fopen打开log_file 如果成功就log_fp = fp不成功就assert。



##### 看一下init_mem

```c
void init_mem() {
#if   defined(CONFIG_TARGET_AM)
  pmem = malloc(CONFIG_MSIZE);
  assert(pmem);
#endif
#ifdef CONFIG_MEM_RANDOM
  uint32_t *p = (uint32_t *)pmem;
  int i;
  for (i = 0; i < (int) (CONFIG_MSIZE / sizeof(p[0])); i ++) {
    p[i] = rand();
  }
#endif
  Log("physical memory area [" FMT_PADDR ", " FMT_PADDR "]",
      (paddr_t)CONFIG_MBASE, (paddr_t)CONFIG_MBASE + CONFIG_MSIZE);
}
```

代码的逻辑也比较直接。如果定义了CONFIG_TARGET_AM则用malloc分配内存，assert防止分配出错，如果定义了CONFIG_MEN_RANDOM 则用for循环调用rand为每个内存赋值随机数。



##### 主要看一下init_isa

```c
void init_isa() {
  /* Load built-in image. */
  memcpy(guest_to_host(RESET_VECTOR), img, sizeof(img));

  /* Initialize this virtual computer system. */
  restart();
}
```

```c
uint8_t* guest_to_host(paddr_t paddr) { return pmem + paddr - CONFIG_MBASE; }
```

此函数的主要功能是做一些ISA相关的初始化工作：

###### 1.将内置的客户程序读入到内存中

​	1.1客户程序与ISA相关，所以放在nemu/src/isa/$ISA/init.c 如：

```c
static const uint32_t img [] = {
  0x800002b7,  // lui t0,0x80000
  0x0002a023,  // sw  zero,0(t0)
  0x0002a503,  // lw  a0,0(t0)
  0x0000006b,  // nemu_trap
};
```

​	1.2内存是一段连续的存储空间，nemu/src/memory/paddr.c

```c
static uint8_t pmem[CONFIG_MSIZE] PG_ALIGN = {};
```

​	1.3读入到内存哪里？nemu简化了，约定总是直接放在nemu/include/memory/paddr.h的RESET_VECTOR中

guest_to_host：

> x86的物理内存是从0开始编址的, 但对于一些ISA来说却不是这样, 例如mips32和riscv32的物理地址均从`0x80000000`开始. 因此对于mips32和riscv32, 其`CONFIG_MBASE`将会被定义成`0x80000000`. 将来CPU访问内存时, 我们会将CPU将要访问的内存地址映射到`pmem`中的相应偏移位置, 这是通过`nemu/src/memory/paddr.c`中的`guest_to_host()`函数实现的. 例如如果mips32的CPU打算访问内存地址`0x80000000`, 我们会让它最终访问`pmem[0]`, 从而可以正确访问客户程序的第一条指令. 这种机制有一个专门的名字, 叫地址映射, 在后续的PA中我们还会再遇到它.



###### 2.初始化寄存器

代码中的restart函数就是初始化寄存器。

```c
static void restart() {
  /* Set the initial program counter. */
  cpu.pc = RESET_VECTOR;

  /* The zero register is always 0. */
  cpu.gpr[0]._32 = 0;
}
```

一共做了两件事，

一是将寄存器中的pc赋值为RESET_VECTOR，这样我们就可以让cpu从我们约定的内存位置开始执行，刚才我们也已经将内置的客户程序加载到了那里，所以，我们就可以运行内置客户程序了。

二是对于mips32和riscv32, 它们的0号寄存器总是存放`0`, 因此我们也需要对其进行初始化。这是一个规定。



一些说明：

寄存器我们使用结构体定义的

```c
typedef struct {
  struct {
    rtlreg_t _32;
  } gpr[32];

  vaddr_t pc;
} riscv32_CPU_state;
```

```c
CPU_state cpu = {};
```

寄存器结构体`CPU_state`的定义放在`nemu/src/isa/$ISA/include/isa-def.h`中, 并在`nemu/src/cpu/cpu-exec.c`中定义一个全局变量`cpu`. 



###### init_isa函数执行后，内存布局如下：

```
pmem:

CONFIG_MBASE      RESET_VECTOR
      |                 |
      v                 v
      -----------------------------------------------
      |                 |                  |
      |                 |    guest prog    |
      |                 |                  |
      -----------------------------------------------
                        ^
                        |
                       pc
```



##### 简单看一下load_img

```c
static long load_img() {
  if (img_file == NULL) {
    Log("No image is given. Use the default build-in image.");
    return 4096; // built-in image size
  }

  FILE *fp = fopen(img_file, "rb");
  Assert(fp, "Can not open '%s'", img_file);

  fseek(fp, 0, SEEK_END);
  long size = ftell(fp);

  Log("The image is %s, size = %ld", img_file, size);

  fseek(fp, 0, SEEK_SET);
  int ret = fread(guest_to_host(RESET_VECTOR), size, 1, fp);
  assert(ret == 1);

  fclose(fp);
  return size;
}
```

这个函数会将一个有意义的客户程序从[镜像文件](https://en.wikipedia.org/wiki/Disk_image)读入到内存, 覆盖刚才的内置客户程序. 这个镜像文件是运行NEMU的一个可选参数, 在运行NEMU的命令中指定. 如果运行NEMU的时候没有给出这个参数, NEMU将会运行内置客户程序.



init_monitor的剩下的操作后面再写。



#### 3.engine_start

init_monitor运行完之后，说明monitor的初始化工作已经完成。就会运行engine_start函数。

```c
void engine_start() {
#ifdef CONFIG_TARGET_AM
  cpu_exec(-1);
#else
  /* Receive commands from user. */
  sdb_mainloop();
#endif
}
```

其中调用sdb_mainloop函数，进入简易调试器的主循环。

```c
void sdb_mainloop() {
  if (is_batch_mode) {
    cmd_c(NULL);
    return;
  }

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

#ifdef CONFIG_DEVICE
    extern void sdl_clear_event_queue();
    sdl_clear_event_queue();
#endif

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

这段代码的主要逻辑是通过rl_gets函数循环获取用户输入（同时输出"(nemu)"字符串）赋值给变量str 

然后再通过字符串处理函数strtok分离字符串赋值给cmd指针。

然后（看这段代码的最后一个for循环）将cmd（用户键入）和cmd_tablefor循环对比，如果找到了，就调用cmd_table[i].handler（一个函数，看cmd_table就知道 name与handler一一映射）

> 比如我们在代码执行到此循环时，屏幕会输出(nemu)并等待用户输入，如果此时我们输入c。因为
>
> ```c
> static struct {
>   const char *name;
>   const char *description;
>   int (*handler) (char *);
> } cmd_table [] = {
>   	{ "help", "Display informations about all supported commands", cmd_help },
>   	{ "c", "Continue the execution of the program", cmd_c },
>   	{ "q", "Exit NEMU", cmd_q },
>   	{"si", "单步执行 N 条指令后, 暂停执行. N 默认为1.", cmd_si },
>   	{"info", "r-打印寄存器状态, w-查看监视点信息", cmd_info},
> 
>   	{"x", "扫描内存", cmd_x},
> 
>   	/* TODO: Add more commands */
> 
> };
> ```
>
> 所以会匹配到cmd_c函数
>
> ```c
> static int cmd_c(char *args) {
>   	cpu_exec(-1);
>   	return 0;
> }
> ```
>
> cmd_c则会调用cpu_exec函数。

##### cmd_exec

此函数模拟了cpu的工作方式:不断执行指令. 

具体地, 代码将在一个for循环中不断调用`fetch_decode_exec_updatepc()`函数, 这个函数的功能就是我们在上一小节中介绍的内容: 让CPU执行当前PC指向的一条指令, 然后更新PC.

```c
void cpu_exec(uint64_t n) {
  g_print_step = (n < MAX_INSTR_TO_PRINT);
  switch (nemu_state.state) {
    case NEMU_END: case NEMU_ABORT:
      printf("Program execution has ended. To restart the program, exit NEMU and run again.\n");
      return;
    default: nemu_state.state = NEMU_RUNNING;
  }

  uint64_t timer_start = get_time();

  Decode s;
  for (;n > 0; n --) {
    fetch_decode_exec_updatepc(&s);
    g_nr_guest_instr ++;
    trace_and_difftest(&s, cpu.pc);
    if (nemu_state.state != NEMU_RUNNING) break;
    IFDEF(CONFIG_DEVICE, device_update());
  }

  uint64_t timer_end = get_time();
  g_timer += timer_end - timer_start;

  switch (nemu_state.state) {
    case NEMU_RUNNING: nemu_state.state = NEMU_STOP; break;

    case NEMU_END: case NEMU_ABORT:
      Log("nemu: %s at pc = " FMT_WORD,
          (nemu_state.state == NEMU_ABORT ? ASNI_FMT("ABORT", ASNI_FG_RED) :
           (nemu_state.halt_ret == 0 ? ASNI_FMT("HIT GOOD TRAP", ASNI_FG_GREEN) :
            ASNI_FMT("HIT BAD TRAP", ASNI_FG_RED))),
          nemu_state.halt_pc);
      // fall through
    case NEMU_QUIT: statistic();
  }
}
```

此代码的具体逻辑后面再介绍。反正此函数的功能就是从内存中：**读取指令，执行，更新pc ，读取指令，执行，更新pc......**如此循环。（遇到某些情况则停止循环）

由于刚才我们运行NEMU的时候并未给出客户程序的镜像文件, 此时NEMU将会运行上文提到的内置客户程序. 

> static const uint32_t img [] = {
>   0x800002b7,  // lui t0,0x80000
>   0x0002a023,  // sw  zero,0(t0)
>   0x0002a503,  // lw  a0,0(t0)
>   0x0000006b,  // nemu_trap
> };

NEMU将不断执行指令, 直到满足某些条件才停止。

键入c后，如果此函数执行结束，那么接着就会重新回到sdb_mainloop中，循环等待用户输入其他的指令，然后查表，调用相应的函数，执行相关逻辑。

可以看一下我的执行结果：

我先make run，然后键入c 然后又键入c 然后键入help。



可以看到make run 之后，我们就陷入了sdb_mainloop函数的循环中，等待用户输入，

然后键入c执行cmd_c，此时cpu便会读取内存，执行，更新pc.这套流程。

因为只有一个内置应用程序，所以很快就执行完了，且触发了**nemu_trap** （nemu_trap不存在于ISA手册中，是我们NEMU自己为了结束指令的执行做的。我们的内置应用程序中的最后一个刚好调用了这个nemu_trap）停止了cpu的工作流程。 

 再次键入c之后，因为内存中只有一个内置应用程序，刚才运行完了后更新了pc，现在pc所指向的地方已经没有应用程序了，所以就提示了Program execution has ended. To restart the program, exit NEMU and run again.

所以回到sdb_mainloop中继续等待用户输入。

然后我又输入了help，显示了相应的内容。

>ubuntu@VM-12-8-ubuntu:~/ics2021/nemu$ make run
>+ LD /home/ubuntu/ics2021/nemu/build/riscv32-nemu-interpreter
>/home/ubuntu/ics2021/nemu/build/riscv32-nemu-interpreter --log=/home/ubuntu/ics2021/nemu/build/nemu-log.txt  
>[src/utils/log.c:13 init_log] Log is written to /home/ubuntu/ics2021/nemu/build/nemu-log.txt
>[src/memory/paddr.c:36 init_mem] physical memory area [0x80000000, 0x88000000]
>[src/monitor/monitor.c:36 load_img] No image is given. Use the default build-in image.
>[src/monitor/monitor.c:13 welcome] Trace: ON
>[src/monitor/monitor.c:14 welcome] If trace is enabled, a log file will be generated to record the trace. This may lead to a large log file. If it is not necessary, you can disable it in menuconfig
>[src/monitor/monitor.c:17 welcome] Build time: 16:16:59, Mar 11 2022
>Welcome to riscv32-NEMU!
>For help, type "help"
>[src/monitor/monitor.c:20 welcome] Exercise: Please remove me in the source code and compile NEMU again.
>(nemu) c
>[src/cpu/cpu-exec.c:115 cpu_exec] nemu: HIT GOOD TRAP at pc = 0x8000000c
>[src/cpu/cpu-exec.c:48 statistic] host time spent = 54 us
>[src/cpu/cpu-exec.c:49 statistic] total guest instructions = 4
>[src/cpu/cpu-exec.c:50 statistic] simulation frequency = 74,074 instr/s
>(nemu) c
>Program execution has ended. To restart the program, exit NEMU and run again.
>(nemu) help
>help - Display informations about all supported commands
>c - Continue the execution of the program
>q - Exit NEMU
>si - 单步执行 N 条指令后, 暂停执行. N 默认为1.
>info - r-打印寄存器状态, w-查看监视点信息
>x - 扫描内存
>(nemu) 

现在看这些流程，是不是有点感觉了呢？





## 一些额外的说明

### 配置系统和项目构建

#### 配置系统kconfig

**(主要看下黑体字就行)**

NEMU中的配置系统源代码位于`nemu/tools/kconfig`（**所以不用看这里的代码，和项目无关**）



在NEMU项目中, "配置描述文件"的文件名都为`Kconfig`, 如`nemu/Kconfig`.（**我们要做的就是自己写nemu/kconfig**）

你键入`make menuconfig`的时候, 背后其实发生了如下事件:

- 检查`nemu/tools/kconfig/build/mconf`程序是否存在, 若不存在, 则编译并生成`mconf`

- 检查`nemu/tools/kconfig/build/conf`程序是否存在, 若不存在, 则编译并生成`conf`

  - 运行命令`mconf nemu/Kconfig`, 此时`mconf`将会解析`nemu/Kconfig`中的描述, 以菜单树的形式展示各种配置选项, 供开发者进行选择

- 退出菜单时, **`mconf`会把开发者选择的结果记录到`nemu/.config`文件中**

- 运行命令

  ```
  conf --syncconfig nemu/Kconfig
  ```

  此时 conf 将会解析 nemu/Kconfig 中的描述, 并读取选择结果 nemu/.config,

   结合两者来生成如下文件:

  - 可以被包含到C代码中的宏定义(`nemu/include/generated/autoconf.h`), 这些宏的名称都是形如`CONFIG_xxx`的形式
  - 可以被包含到Makefile中的变量定义(`nemu/include/config/auto.conf`)
  - 可以被包含到Makefile中的, 和"配置描述文件"相关的依赖规则(`nemu/include/config/auto.conf.cmd`), 为了阅读代码, 我们可以不必关心它
  - 通过时间戳来维护配置选项变化的目录树`nemu/include/config/`, 它会配合另一个工具`nemu/tools/fixdep`来使用, 用于在更新配置选项后节省不必要的文件编译, 为了阅读代码, 我们可以不必关心它

**所以, 目前我们只需要关心配置系统生成的如下文件:**

- `nemu/include/generated/autoconf.h`, 阅读C代码时使用
- `nemu/include/config/auto.conf`, 阅读Makefile时使用



#### 项目构建makefile

**(同样主要看下黑体字)**

NEMU的Makefile会稍微复杂一些, 它具备如下功能:

##### 与配置系统进行关联

**通过包含`nemu/include/config/auto.conf`, 与kconfig生成的变量进行关联. 因此在通过menuconfig更新配置选项后, Makefile的行为可能也会有所变化.**

##### 文件列表(filelist)

**通过文件列表(filelist)决定最终参与编译的源文件. 在`nemu/src`及其子目录下存在一些名为`filelist.mk`的文件,** 它们会根据menuconfig的配置对如下4个变量进行维护:

- `SRCS-y` - 参与编译的源文件的候选集合
- `SRCS-BLACKLIST-y` - 不参与编译的源文件的黑名单集合
- `DIRS-y` - 参与编译的目录集合, 该目录下的所有文件都会被加入到`SRCS-y`中
- `DIRS-BLACKLIST-y` - 不参与编译的目录集合, 该目录下的所有文件都会被加入到`SRCS-BLACKLIST-y`中

Makefile会包含项目中的所有`filelist.mk`文件, 对上述4个变量的追加定义进行汇总, 最终会过滤出在`SRCS-y`中但不在`SRCS-BLACKLIST-y`中的源文件, 来作为最终参与编译的源文件的集合.



上述4个变量还可以与menuconfig的配置结果中的布尔选项进行关联, 例如`DIRS-BLACKLIST-$(CONFIG_TARGET_AM) += src/monitor/sdb`, 当我们在menuconfig中选择了`TARGET_AM`相关的布尔选项时, kconfig最终会在`nemu/include/config/auto.conf`中生成形如`CONFIG_TARGET_AM=y`的代码, 对变量进行展开后将会得到`DIRS-BLACKLIST-y += src/monitor/sdb`; 当我们在menuconfig中未选择`TARGET_AM`相关的布尔选项时, kconfig将会生成形如`CONFIG_TARGET_AM=n`的代码, 或者未对`CONFIG_TARGET_AM`进行定义, 此时将会得到`DIRS-BLACKLIST-n += src/monitor/sdb`, 或者`DIRS-BLACKLIST- += src/monitor/sdb`, 这两种情况都不会影响`DIRS-BLACKLIST-y`的值, 从而实现了如下效果:

```
在menuconfig中选中TARGET_AM时, nemu/src/monitor/sdb目录下的所有文件都不会参与编译.
```

（**看不懂。。。以后再说**）

贴一个文章咯，makefile的使用，写的不错。

[makefile介绍 — 跟我一起写Makefile 1.0 文档 (seisman.github.io)](https://seisman.github.io/how-to-write-makefile/introduction.html)

##### 编译和链接

（**后面再回头看**）

Makefile的编译规则在`nemu/scripts/build.mk`中定义:

```makefile
$(OBJ_DIR)/%.o: %.c
  @echo + CC $<
  @mkdir -p $(dir $@)
  @$(CC) $(CFLAGS) -c -o $@ $<
  $(call call_fixdep, $(@:.o=.d), $@)
```

其中关于`$@`和`$<`等符号的含义, 可以RTFM进行了解. `call_fixdep`的调用用于生成更合理的依赖关系, 目前我们主要关注编译的命令, 因此可以先忽略`call_fixdep`.

我们可以先查看`make`过程中都运行了哪些命令, 然后反过来理解`$(CFLAGS)`等变量的值. 为此, 我们可以键入`make -nB`, 它会让`make`程序以"只输出命令但不执行"的方式强制构建目标. 运行后, 你可以看到很多形如

```
gcc -O2 -MMD -Wall -Werror -I/home/user/ics2021/nemu/include
-I/home/user/ics2021/nemu/src/engine/interpreter -I/home/use
r/ics2021/nemu/src/isa/riscv32/include -O2    -D__GUEST_ISA__
=riscv32  -c -o /home/user/ics2021/nemu/build/obj-riscv32-nem
u-interpreter/src/utils/timer.o src/utils/timer.c
```

的输出, 这样你就很容易理解上述Makefile变量的值了:

```
$(CC) -> gcc
$@ -> /home/user/ics2021/nemu/build/obj-riscv32-nemu-interpreter/src/utils/timer.o
$< -> src/utils/timer.c
$(CFLAGS) -> 剩下的内容
```

于是你就可以根据上述输出结果和Makefile反推`$(CFLAGS)`的值是如何形成的. 因为编译每个文件的命令都很类似, 当你理解了一个源文件的编译之后, 你就能类推到其它源文件的编译过程了. 同理, 你也可以按照上述方法理解最后的链接命令.