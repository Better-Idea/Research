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
    pthread_attr_t attr;
    pthread_t thread;
    size_t stacksize;
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
    pthread_attr_init(&attr);
    pthread_attr_getstacksize(&attr,&stacksize);
    if (!stacksize) stacksize = 1; /* The world is full of Solaris Fixes */
    while (stacksize < REDIS_THREAD_STACK_SIZE) stacksize *= 2;
    pthread_attr_setstacksize(&attr, stacksize);

    /* Ready to spawn our threads. We use the single argument the thread
     * function accepts in order to pass the job ID the thread is
     * responsible of. */
    for (j = 0; j < BIO_NUM_OPS; j++) {
        void *arg = (void*)(unsigned long) j;
        if (pthread_create(&thread,&attr,bioProcessBackgroundJobs,arg) != 0) {
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
    # 栈寄存器位 64bit 无符号数，其中高 32bit 为段 id，低 32bit 为偏移
    # 访问主栈时，可以小消去段 id，让寄存器高 32bit 设置为 0
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

loop.begin
    # keep 是本 ISA 候选的指令
    # 具有 keep 和 keep.emit 两个版本
    # keep 将相当于惰性保存各个寄存器但不递增缓冲区指针，再调用 keep 时会覆盖旧值
    # keep.emit 是再 keep 的基础上改变缓冲区指针
    keep.emit       j, bio_mutex, bio_newjob_cond, bio_step_cond, bio_jobs

    # 设置参数 0
    bdcqq           a0.bio_mutex, bio_mutex

    # 设置参数 1
    xor             a1.null, a1.null

    # 调用
    imm
    imm
    jal             pthread_mutex_init

    # rcv 是本 ISA 候选的指令
    # rcv 全称为 recover
    # 和 keep 互逆，也具有 rcv 和 rcv.emit 两个版本
    rcv             bio_newjob_cond
    movqq           a0.bio_newjob_cond, bio_newjob_cond
    xor             a1.null, a1.null
    imm
    imm
    jal             pthread_cond_init

    rcv             bio_step_cond
    movqq           a0.bio_step_cond, bio_step_cond
    xor             a1.null, a1.null
    imm
    imm
    jal             pthread_cond_init

    rcv.emit        j, bio_mutex, bio_newjob_cond, bio_step_cond, bio_jobs

    # 相关性较强的函数会被放到一个 64K 区域
    imm
    jal             listCreate

    # 8 字节数据存储到内存
    stq             ret.listCreate, bio_jobs

    sub             j, 1
    ifnz            loop.end

    imm
    add             bio_mutex, 48

    imm
    add             bio_newjob_cond, 40

    imm
    add             bio_step_cond, 40

    add             bio_jobs, sizeof(voidp)
    jmp             loop.begin
loop.end:
    snew.fetch      56
    movqq           attr, rt
    snew.fetch      8
    movqq           stacksize, rt
    keep            attr, stacksize
    imm
    jal             pthread_attr_init
    rcv             attr, stacksize
    imm
    jal             pthread_attr_getstacksize
    rcv             attr, stacksize
    ldq             stacksize, [stacksize]
    bdcqi           one, 1
    miax            one, stacksize

    imm
    imm
    bdcqi           REDIS_THREAD_STACK_SIZE, imm
loop.begin.2:
    ciflt           stacksize, REDIS_THREAD_STACK_SIZE, loop.end.2
    shl             stacksize, 1
    jmp             loop.begin.2
loop.end.2:
    rcv             attr
    imm
    jal             pthread_attr_setstacksize

    rcv.emit.c      attr
    xor             a3.j, a3.j

    snew.fetch      8
    movqq           a0.thread, rt
    movqq           a1.attr, attr

    imm
    bdcqi           bio_threads, imm
loop.begin.3:
    cmp             a3.j, 3
    iflt            loop.end.3
    keep            a0.thread, a1.attr, a3.j, bio_threads
    imm
    jal             pthread_create
    rcv.emit        a0.thread, a1.attr, a3.j, bio_threads
    cmp             ret.pthread_create, 0

    ifne            if.0
    movqqx          a0, 3
    imm
    bdcqi           a1, imm
    imm
    jal             serverLog
    movqqx          a0, 1
    imm
    jal             exit
if.0:
    ldq             rt, [a0.thread]
    stq             rt, [p_bio_threads]
    add             j, 1
    add             p_bio_threads, 8
    jmp             loop.begin.3
loop.end.3:
    ret
```