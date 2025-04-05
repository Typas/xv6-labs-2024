# Notes

## sleep
這邊只要用 `atoi` 處理 `argv[1]` 後，叫 `sleep` 處理就好。

不過我想再往下追一下流程。

```c
int sleep(int);
```

### syscall - sleep

這是在 `user/user.h` 裡面的宣告，問題來了，實作在哪？
lsp 找不到，但題目裡叫你去追 `user/usys.S` 與 `kernel/sysproc.c`。

```asm
.global sleep
sleep:
 li a7, SYS_sleep
 ecall
 ret
```

而這東西是由 `usys.pl` 所產生的，非常的制式。
至於解釋這串東西，我交給 ChatGPT，看完後再調整一下理解。

`.global sleep` 去定義了這函數可以被其他檔案找到，可以被 linker 去連結，相當於在 `usys.c` 裡面寫了一個 `int sleep() { syscall(SYS_sleep); }`。但這個東西為什麼可以被 linker 去對應到 `int sleep(int);` 是一個謎。
`a7` 則在 RISC-V 裡通常是給系統呼叫的保留位置，而 `SYS_sleep` 是一個要再找的常數。

`ecall` 去做一個 trap 給 OS 去叫 kernel 函數做事，叫哪個就是 `SYS_sleep` 這常數決定的。
`ret` 則是返回函數值。

在用了 `rg` 找完，在 `kernel/syscall.h` 裡找到了 `#define SYS_sleep  13` 這行定義。

再來則是在 `kernel/syscall.c` 去找到
```c
static uint64 (*syscalls[])(void) = {
/* 省略 */
[SYS_sleep]   sys_sleep,
/* 省略 */
};
```

這種十分變態的定義。這東西是一個函數指標陣列的定義，回傳型態為 `uint64`，而參數只有一個，且為 `void` 型態。`[SYS_sleep] sys_sleep,` 可以解成 `[13] sys_sleep`，是一個叫 designated initializers 的東西，直接指定索引值與其對應的函數。 ~~最簡單的 hashmap 就是陣列了，隱含了 hash function f(x) = x。~~ 

順便來看一下 `syscall` 是怎麼寫的。
```c
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    // Use num to lookup the system call function for num, call it,
    // and store its return value in p->trapframe->a0
    p->trapframe->a0 = syscalls[num]();
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

這邊用了 `myproc()` 取得了呼叫 trap 的程式，也就是上面那段奇妙的 RISC-V assembly，從 `a7` 拿了系統呼叫的編號，並在 `if` 表達式中確認了正確性，避免叫到 NULL 炸掉整個系統。在正確的情況下則是去把系統呼叫的回傳值放進 `a0` 裡面，到時 assembly 的 `ret` 會回傳這個值。

因為用了 designated initializers 這個玩法，所以可以保證就算跳號也不會被 `NELEM()` 這個巨集搞到，只是跳號會出現指去 NULL 的情況，所以需要再一個判斷排除。至於為什麼在 kernel 能用 `printf`，窩不知道。

### `myproc()`

`myproc()` 何許人也？在 `kernel/proc.c` 裡找到了它。
```c
// Return the current struct proc *, or zero if none.
struct proc*
myproc(void)
{
  push_off();
  struct cpu *c = mycpu();
  struct proc *p = c->proc;
  pop_off();
  return p;
}
```

好的，出現了更多莫名其妙的東西，`push_off()` 跟 `pop_off()` 看起來是成對的，而 `mycpu()` 在上面，等等看。這 `struct cpu` 裡面存有的結構之一就是 `struct proc *`，只要丟回去就好。

### `push_off` 與 `pop_off`

先看 `push_off()` 與 `pop_off()`，在 `kernel/spinlock.c`。

```c
// push_off/pop_off are like intr_off()/intr_on() except that they are matched:
// it takes two pop_off()s to undo two push_off()s.  Also, if interrupts
// are initially off, then push_off, pop_off leaves them off.

void
push_off(void)
{
  int old = intr_get();

  intr_off();
  if(mycpu()->noff == 0)
    mycpu()->intena = old;
  mycpu()->noff += 1;
}

void
pop_off(void)
{
  struct cpu *c = mycpu();
  if(intr_get())
    panic("pop_off - interruptible");
  if(c->noff < 1)
    panic("pop_off");
  c->noff -= 1;
  if(c->noff == 0 && c->intena)
    intr_on();
}
```

這兩個東西不意外的成對，如同 `malloc` 與 `free` 的關係。

要看懂這東西，我得先看懂 `intr_get`、`intr_off`、`mycpu`、`noff`、`intena`、`panic` 與 `intr_on` 各是什麼。

#### `intr_get()`

首先我們遇到了又一個不知道哪來的函數， `intr_get` ，再用 `rg` 找找，於是看到了 `kernel/riscv.h` 裡有這的定義。
```c
// are device interrupts enabled?
static inline int
intr_get()
{
  uint64 x = r_sstatus();
  return (x & SSTATUS_SIE) != 0;
}
```

這邊整體很簡單，去看 Supervisor Status Register，確定 Supervisor Interrupt Enable 這格有沒有起來。

##### `r_sstatus()`

好，又多了兩個沒看過的東西，繼續 `rg`，我就爛。
在同一個檔案裡看到了這東西。

```c
static inline uint64
r_sstatus()
{
  uint64 x;
  asm volatile("csrr %0, sstatus" : "=r" (x) );
  return x;
}
```

問問神奇的 ChatGPT，原來要再看一下原檔上面一點的程式碼，這邊有 `sstatus` 的各 bit 定義。
```c
// Supervisor Status Register, sstatus

#define SSTATUS_SPP (1L << 8)  // Previous mode, 1=Supervisor, 0=User
#define SSTATUS_SPIE (1L << 5) // Supervisor Previous Interrupt Enable
#define SSTATUS_UPIE (1L << 4) // User Previous Interrupt Enable
#define SSTATUS_SIE (1L << 1)  // Supervisor Interrupt Enable
#define SSTATUS_UIE (1L << 0)  // User Interrupt Enable
```

`asm volatile` 是為了保證編譯時不會被最佳化成快取之類的情況，保證取得的值是實際值。`csrr` 是 CSR Read，其中 CSR 為 Control and Status Register。`%0` 給編譯器自由發揮，能用的暫存器隨他選。`sstatus` 則是從 CSR 讀。後面的 `: "=r", (x)` 照我查到的資料，整個 RISC-V asm 語法上是
```c
asm volatile(
    "指令"
    : "輸出串列" // 選擇性
    : "輸入串列" // 選擇性
    : "可能用的暫存器" // 選擇性
)
```
所以後面那串是表示運算是怎麼輸出的。其中 `"=r"` 的 `=` 表示這個只寫不讀，而 `r` 表示任意的暫存器都可以。
所以這串如果照一般化的 C 程式碼來理解，會接近於 `x = csrr(sstatus);` 。
在下面還有一個 `w_sstatus`，是寫入的版本，所以 `r_sstatus` 的 `r` 是讀取。

#### `intr_off()`
這東西在 `kernel/riscv.h`，整體上是一個很簡單的 unset `SSTATUS_SIE` bit 後回寫到 CSR 裡面去。

```c
// disable device interrupts
static inline void
intr_off()
{
  w_sstatus(r_sstatus() & ~SSTATUS_SIE);
}

```

#### `intr_on()`
與上面的 `intr_off` 在隔壁，是 set `SSTATUS_SIE` bit 後回寫到 CSR 裡面去。
至於會不會有重複 set 的問題，那不關這函數的事。
```c
// enable device interrupts
static inline void
intr_on()
{
  w_sstatus(r_sstatus() | SSTATUS_SIE);
}
```

#### `panic()`
這個函數在 `kernel/printf.c`，也就是 `printf` 所在的地方。

```c
void
panic(char *s)
{
  pr.locking = 0;
  printf("panic: ");
  printf("%s\n", s);
  panicked = 1; // freeze uart output from other CPUs
  for(;;)
    ;
}
```

出現了很多不明所以的東西，像是 `pr.locking`、`panicked` 與最後的無限迴圈。
基本上就是釋放掉鎖，印出錯誤訊息後，宣布系統進 panic 狀態，並卡在無限迴圈中，等使用者去外力關機。

##### `pr`
這是一個萬惡的全域變數，放在同一個檔案裡。
```c
// lock to avoid interleaving concurrent printf's.
static struct {
  struct spinlock lock;
  int locking;
} pr;
```

一個專有的結構，包含了一個 spinlock 與一個 flag `locking`。

##### `panicked`
這是一個萬惡的全域變數，放在同一個檔案裡，去表示有沒有觸發無法處理的情況。

```c
volatile int panicked = 0;
```

### `mycpu()`
在 `kernel/proc.c`，在 `myproc` 的上面。

```c
// Return this CPU's cpu struct.
// Interrupts must be disabled.
struct cpu*
mycpu(void)
{
  int id = cpuid();
  struct cpu *c = &cpus[id];
  return c;
}
```

先是用 `cpuid` 取得了 ID，再去把取一個全域變數 `cpus` 陣列的位址回傳。

```c
struct cpu cpus[NCPU];
```

這邊定義很簡單，是一個全域的陣列。其中 `NCPU` 在 `kernel/param` 定義，是 CPU 數量的最大值。

#### `struct cpu`
這個結構在 `kernel/proc.h` 宣告，對應到的應該是 CPU 中的 thread。
```c
// Per-CPU state.
struct cpu {
  struct proc *proc;          // The process running on this cpu, or null.
  struct context context;     // swtch() here to enter scheduler().
  int noff;                   // Depth of push_off() nesting.
  int intena;                 // Were interrupts enabled before push_off()?
};
```

其中 `proc` 是一個指標，指向目前在跑的 process。`context` 則是在 context switching 會被換掉的東西。`noff` 記錄了 `push_off` 進了幾層。`intena` 則是記錄在 `push_off` 之前是否就有中斷了。

##### `struct context`
```c
// Saved registers for kernel context switches.
struct context {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};
```
這個相對簡單，就是把能存的暫存器都定義出來。

##### `struct proc`
一樣在 `kernel/proc.h`
```c
// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // wait_lock must be held when using this:
  struct proc *parent;         // Parent process

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```

裡面有一個 spinlock，與要有這個 lock 才能用的 `state`、`chan`、`killed`、`xstate` 與 `pid`，和要在 `kernel/proc.c` 裡的 spinlock `wait_lock` 有 hold 住才能用的 `parent`。剩下的都是屬於這個行程專有的東西，所以不需要 lock 才能存取。
細節不說，`trapframe` 的定義在同個檔案，與 RISC-V 的暫存器比較有關，`NOFILE` 被定義在 `kernel/param.h`，是單個行程能用的最大檔案數量。`pagetable_t` 實際上是一個 `uint64*`。

##### `enum procstate`
```c
enum procstate { UNUSED, USED, SLEEPING, RUNNABLE, RUNNING, ZOMBIE };
```
有很多的狀態，已經有人寫了[文章](https://ithelp.ithome.com.tw/articles/10306838)。

#### `cpuid()`

## pingpong

## primes

## find

## xargs

## challenges

### uptime

### regex

### shell modification

#### conditional `$`

#### support wait

#### support tab completion

#### keep history of passed shell commands
