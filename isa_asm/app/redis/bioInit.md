# BIOINIT
## 引用
redis/src/bio.c:bioInit 函数

## C 源码
```C
static pthread_t bio_threads[BIO_NUM_OPS];
static pthread_mutex_t bio_mutex[BIO_NUM_OPS];
static pthread_cond_t bio_newjob_cond[BIO_NUM_OPS];
static pthread_cond_t bio_step_cond[BIO_NUM_OPS];
static list *bio_jobs[BIO_NUM_OPS];
/* The following array is used to hold the number of pending jobs for every
 * OP type. This allows us to export the bioPendingJobsOfType() API that is
 * useful when the main thread wants to perform some operation that may involve
 * objects shared with the background thread. The main thread will just wait
 * that there are no longer jobs of this type to be executed before performing
 * the sensible operation. This data is also useful for reporting. */
static unsigned long long bio_pending[BIO_NUM_OPS];

/* This structure represents a background Job. It is only used locally to this
 * file as the API does not expose the internals at all. */
struct bio_job {
    time_t time; /* Time at which the job was created. */
    /* Job specific arguments.*/
    int fd; /* Fd for file based background jobs */
    lazy_free_fn *free_fn; /* Function that will free the provided arguments */
    void *free_args[]; /* List of arguments to be passed to the free function */
};

void *bioProcessBackgroundJobs(void *arg);

/* Make sure we have enough stack to perform all the things we do in the
 * main thread. */
#define REDIS_THREAD_STACK_SIZE (1024*1024*4)

/* Initialize the background system, spawning the thread. */
void bioInit(void) {
    pthread_attr_t a0.attr;
    pthread_t thread;
    size_t a1.stacksize;
    int j;

    /* Initialization of state vars and objects */
    for (j = 0; j < BIO_NUM_OPS; j++) {
        pthread_mutex_init(&bio_mutex[j],NULL);
        pthread_cond_init(&bio_newjob_cond[j],NULL);
        pthread_cond_init(&bio_step_cond[j],NULL);
        bio_jobs[j] = listCreate();
        bio_pending[j] = 0;
    }

    /* Set the stack size as by default it may be small in some system */
    pthread_attr_init(&a0.attr);
    pthread_attr_getstacksize(&a0.attr,&a1.stacksize);
    if (!a1.stacksize) a1.stacksize = 1; /* The world is full of Solaris Fixes */
    while (a1.stacksize < REDIS_THREAD_STACK_SIZE) a1.stacksize *= 2;
    pthread_attr_setstacksize(&a0.attr, a1.stacksize);

    /* Ready to spawn our threads. We use the single argument the thread
     * function accepts in order to pass the job ID the thread is
     * responsible of. */
    for (j = 0; j < BIO_NUM_OPS; j++) {
        void *arg = (void*)(unsigned long) j;
        if (pthread_create(&thread,&a0.attr,bioProcessBackgroundJobs,arg) != 0) {
            serverLog(LL_WARNING,"Fatal: Can't initialize Background Jobs.");
            exit(1);
        }
        bio_threads[j] = thread;
    }
}
```

## Mix-C ASM
```asm
proc bioInit:
    # 广播式赋值，j = 3，这里只设置一个寄存器
    bdcqix          j, 3

    # 加载立即数，立即数分成两段
    # imm 包含低 12bit
    # bdcqi 中 imm.bio_mutex 包含高 4bit
    # bio_mutex 在 C 源代码中是全局静态数组
    # 在 Mix-C ISA 中，全局变量是放在主线程栈上
    # 栈寄存器为 64bit 无符号数，其中高 32bit 为段 id，低 32bit 为偏移
    # 访问主栈时，可以消去段 id，让寄存器高 32bit 设置为 0
    # 在低 32bit 中，若第 31bit(0 based) 为 0，表示访问主栈，为 1 表示访问线程本地栈
    # 这里预估全局变量的总体积在 64K 以内
    imm
    bdcqi           bio_mutex, imm.bio_mutex

    imm
    bdcqi           bio_newjob_cond, imm.bio_newjob_cond

    imm
    bdcqi           bio_step_cond, imm.bio_step_cond

    imm
    bdcqi           bio_jobs, imm.bio_jobs

loop.begin.0
    # keep 是本 ISA 候选的指令
    # 具有 keep 和 keep.emit 两个版本
    # keep 将相当于惰性保存各个寄存器但不递增上下文指针，再调用 keep 时会覆盖旧值
    # keep.emit 是在 keep 的基础上改变缓冲区指针
    keep.emit       j, bio_mutex, bio_newjob_cond, bio_step_cond, bio_jobs

    # 设置参数 0
    bdcqq           a0.bio_mutex, bio_mutex

    # 设置参数 1
    xor             a1.null, a1.null

    # jal 中转移地址中第 0 位为 1 表示到全局函数列表中查找函数地址
    # 可以适用于动态库加载
    imm
    jal             pthread_mutex_init

    # rcv 是本 ISA 候选的指令
    # rcv 全称为 recover
    # 和 keep 互逆，但具有 rcv、 rcv.emit 和 rcv.nop 三个版本
    # rcv.nop 清除保存了但不需要再恢复的寄存器上下文
    rcv             bio_newjob_cond
    movqq           a0.bio_newjob_cond, bio_newjob_cond
    xor             a1.null, a1.null
    imm
    jal             pthread_cond_init

    rcv             bio_step_cond
    movqq           a0.bio_step_cond, bio_step_cond
    xor             a1.null, a1.null
    imm
    jal             pthread_cond_init

    rcv.emit        j, bio_mutex, bio_newjob_cond, bio_step_cond, bio_jobs

    imm
    jal             listCreate

    # 8 字节数据存储到内存
    stq             ret.listCreate, bio_jobs

    sub             j, 1

    # 类似 if (sta.zf == 0)
    # 如果条件满足，就执行下面的逻辑，否则跳转到 loop.end.0
    ifnz            loop.end.0

    # 48 需要 5bit 才能存储，其中 imm 提供低 12bit，add 中提供高 4bit
    # 其中 11bit 未用到
    imm
    add             bio_mutex, 48

    imm
    add             bio_newjob_cond, 40

    imm
    add             bio_step_cond, 40

    add             bio_jobs, sizeof(voidp)
    jmp             loop.begin.0
loop.end.0:
    # snew 是本 ISA 候选指令
    # 用于从栈上分配内存，本 ISA 中栈是向高地址生长的
    # 先将当前栈地址赋值给 rt 临时寄存器，然后将栈地址递增
    # 栈内存的分配单位为 8 字节，本次分配 7 * 8 字节内存
    snew            7

    # a0.attr = rt
    movqq           a0.attr, rt
    snew            8
    movqq           a1.stacksize, rt
    keep.emit       a0.attr, a1.stacksize

    imm
    jal             pthread_attr_init
    rcv             a0.attr, a1.stacksize

    imm
    jal             pthread_attr_getstacksize
    rcv.emit        a0.attr, a1.stacksize

    # 从栈内存中加载到寄存器
    ldq             a1.stacksize, [a1.stacksize]

    # 赋值立即数
    bdcqi           one, 1

    # 比较并设置最大最小值
    # 最小值放到 one，最大值放到 a1.stacksize
    miax            one, a1.stacksize

    # REDIS_THREAD_STACK_SIZE = 1024*1024*4
    imm
    imm
    bdcqi           REDIS_THREAD_STACK_SIZE, imm
loop.begin.1:
    # 仅用于寄存器间带转移比较指令
    # if (a1.stacksize < REDIS_THREAD_STACK_SIZE)
    ciflt           a1.stacksize, REDIS_THREAD_STACK_SIZE, loop.end.1

    # 左移 1bit
    shl             a1.stacksize, 1
    jmp             loop.begin.1
loop.end.1:
    rcv             a0.attr
    imm
    jal             pthread_attr_setstacksize

    rcv.emit        a0.attr, a1.stacksize
    xor             a3.j, a3.j

    snew            8
    movqq           a1.attr, a0.attr
    movqq           a0.thread, rt

    imm
    bdcqi           bio_threads, imm

    imm
    bdcqi           a2.bioProcessBackgroundJobs, bioProcessBackgroundJobs
loop.begin.2:
    cmp             a3.j, 3
    iflt            loop.end.2
    keep            a0.thread, a1.attr, a2.bioProcessBackgroundJobs, a3.j, bio_threads
    imm
    jal             pthread_create
    rcv.emit        a0.thread, a1.attr, a3.j, bio_threads
    cmp             ret.pthread_create, 0

    ifne            if.0
    movqqx          a0, 3
    imm
    bdcqi           a1, string
    imm
    jal             serverLog
    movqqx          a0, 1
    imm
    jal             exit
if.0:
    ldq             rt, [a0.thread]
    stq             rt, [bio_threads]
    add             j, 1
    add             bio_threads, 8
    jmp             loop.begin.2
loop.end.2:
    ret
```

## C 源码
```C
struct bio_job {
    time_t time; /* Time at which the job was created. */
    /* Job specific arguments.*/
    int fd; /* Fd for file based background jobs */
    lazy_free_fn *free_fn; /* Function that will free the provided arguments */
    void *free_args[]; /* List of arguments to be passed to the free function */
};

void bioSubmitJob(int type, struct bio_job *job) {
    job->time = time(NULL);
    pthread_mutex_lock(&bio_mutex[type]);
    listAddNodeTail(bio_jobs[type],job);
    bio_pending[type]++;
    pthread_cond_signal(&bio_newjob_cond[type]);
    pthread_mutex_unlock(&bio_mutex[type]);
}
```

## Mix-C ASM
```
proc bioSubmitJob:
    imm
    bdcqi           bio_mutex, imm

    imm
    bdcqi           bio_jobs, imm

    imm
    bdcqi           bio_pending, imm

    imm
    bdcqi           bio_newjob_cond, imm

    imm
    mul             rt, a0.type, sizeof(pthread_mutex_t)
    add             bio_mutex, rt

    shl             rt, a0.type, 3
    add             bio_jobs, rt
    add             bio_pending, rt

    imm
    mul             rt, a0.type, sizeof(pthread_cond_t)
    add             bio_newjob_cond, rt

    keep.emit       a1.job, bio_mutex, bio_mutex, bio_pending, bio_newjob_cond
    imm
    jal             time
    rcv             a1.job, bio_mutex
    stq             ret, a1.job:time

    movqq           a0.bio_mutex, bio_mutex
    imm
    jal             pthread_mutex_lock
    rcv             a1.job, bio_jobs

    movqq           a0.bio_jobs, bio_jobs
    imm
    bdcqi           bio_newjob_cond, imm
    ldq             bio_pending_v, [bio_pending]
    add             bio_pending_v, 1
    stq             bio_pending_v, [bio_pending]

    rcv             bio_new_job_cond
    movqq           a0.bio_new_job_cond, bio_new_job_cond
    imm
    jal             pthread_cond_signal

    rcv             bio_mutex
    movqq           a0.bio_mutex, bio_mutex
    imm
    jal             pthread_mutex_unlock
    ret

```

## C 源码
```C
typedef void lazy_free_fn(void *args[]);

void bioCreateLazyFreeJob(lazy_free_fn free_fn, int arg_count, ...) {
    va_list valist;
    /* Allocate memory for the job structure and all required
     * arguments */
    struct bio_job *job = zmalloc(sizeof(*job) + sizeof(void *) * (arg_count));
    job->free_fn = free_fn;

    va_start(valist, arg_count);
    for (int i = 0; i < arg_count; i++) {
        job->free_args[i] = va_arg(valist, void *);
    }
    va_end(valist);
    bioSubmitJob(BIO_LAZY_FREE, job);
}

```

## Mix-C ISA
```
proc bioCreateLazyFreeJob:
    # keep 指令占用的空间是独立于栈内存
    keep.emit       a0.free_fn, a1.arg_count
    shl             rt.arg_count, a1.arg_count, 3
    imm
    add             rt.bytes, rt.arg_count, sizeof(bio_job)
    movqq           a0.bytes, rt.bytes
    jal             zmalloc
    rcv.emit        a0.free_fn, a1.arg_count
    imm
    add             job.free_fn, ret.job, 16
    imm
    add             job.free_args, ret.job, 24
    stq             a0.free_fn, [job.free_fn]

    # 读取栈帧地址，栈是向高地址生长的
    rdstk           valist
loop.begin,0:
    sub             arg_count, 1
    ifge            loop.end.0
    ldq             rt, [vallist]
    stq             rt, [job.free_args]
    add             job.free_args, 8
    jmp             loop.begin,0
loop.end.0:
    movqqx          a0.BIO_LAZY_FREE, 2
    movqq           a1.job, ret.job
    jal             bioSubmitJob
    ret
```


