## （二）设置libevent

libevent有一些被整个进程共享的、影响整个库的全局设置。必须在调用libevent库的任何其他部分之前修改这些设置，否则，libevent会进入不一致的状态。

### 1 Libevent中的日志消息

libevent可以记录内部错误和警告。如果编译进日志支持，还会记录调试信息。默认配置下这些信息被写到stderr。通过提供定制的日志函数可以覆盖默认行为。

接口
```c
#define EVENT_LOG_DEBUG 0
#define EVENT_LOG_MSG   1
#define EVENT_LOG_WARN  2
#define EVENT_LOG_ERR   3
    
/* Deprecated; see note at the end of this section */
#define _EVENT_LOG_DEBUG EVENT_LOG_DEBUG
#define _EVENT_LOG_MSG   EVENT_LOG_MSG
#define _EVENT_LOG_WARN  EVENT_LOG_WARN
#define _EVENT_LOG_ERR   EVENT_LOG_ERR
    
typedef void (*event_log_cb)(int severity, const char *msg);
    
void event_set_log_callback(event_log_cb cb);
```

要覆盖libevent的日志行为，编写匹配event_log_cb签名的定制函数，将其作为参数传递给event_set_log_callback（）。随后libevent在日志信息的时候，将会把信息传递给你提供的函数。再次调用event_set_log_callback（），传递参数NULL，就可以恢复默认行为。

示例
```c
#include <event2/event.h>
#include <stdio.h>

static void discard_cb(int severity, const char *msg)
{
    /* This callback does nothing. */
}

static FILE *logfile = NULL;
static void write_to_file_cb(int severity, const char *msg)
{
    const char *s;
    if (!logfile)
        return;
    switch (severity) {
        case _EVENT_LOG_DEBUG: s = "debug"; break;
        case _EVENT_LOG_MSG:   s = "msg";   break;
        case _EVENT_LOG_WARN:  s = "warn";  break;
        case _EVENT_LOG_ERR:   s = "error"; break;
        default:               s = "?";     break; /* never reached */
    }
    fprintf(logfile, "[%s] %s\n", s, msg);
}

/* Turn off all logging from Libevent. */
void suppress_logging(void)
{
    event_set_log_callback(discard_cb);
}

/* Redirect all Libevent log messages to the C stdio file 'f'. */
void set_logfile(FILE *f)
{
    logfile = f;
    event_set_log_callback(write_to_file_cb);
}
```
注意

在用户提供的event_log_cb回调函数中调用libevent函数是不安全的。比如说，如果试图编写一个使用bufferevent将警告信息发送给某个套接字的日志回调函数，可能会遇到奇怪而难以诊断的bug。未来版本libevent的某些函数可能会移除这个限制。

这个函数在<event2/event.h>中声明，在libevent 1.0c版本中首次出现。

### 2 处理致命错误

libevent在检测到不可恢复的内部错误时的默认行为是调用exit（）或者abort（），退出正在运行的进程。这类错误通常意味着某处有bug：要么在你的代码中，要么在libevent中。

如果希望更优雅地处理致命错误，可以为libevent提供在退出时应该调用的函数，覆盖默认行为。

接口
```c
typedef void (*event_fatal_cb)(int err);
void event_set_fatal_callback(event_fatal_cb cb);
```

要使用这些函数，首先定义libevent在遇到致命错误时应该调用的函数，将其传递给event_set_fatal_callback（）。随后libevent在遇到致命错误时将调用你提供的函数。
你的函数不应该将控制返回到libevent：这样做可能导致不确定的行为。为了避免崩溃，libevent还是会退出。你的函数被不应该调用其它libevent函数。

这些函数声明在<event2/event.h>中，在libevent 2.0.3-alpha版本中首次出现。

### 3 内存管理

默认情况下，libevent使用C库的内存管理函数在堆上分配内存。通过提供malloc、realloc和free的替代函数，可以让libevent使用其他的内存管理器。希望libevent使用一个更高效的分配器时；或者希望libevent使用一个工具分配器，以便检查内存泄漏时，可能需要这样做。

接口
```c
void event_set_mem_functions(void *(*malloc_fn)(size_t sz),
                             void *(*realloc_fn)(void *ptr, size_t sz),
                             void (*free_fn)(void *ptr));
```

这里有个替换libevent分配器函数的示例，它可以计算已经分配的字节数。实际应用中可能需要添加锁，以避免运行在多个线程中时发生错误。

示例
```c
#include <event2/event.h>
#include <sys/types.h>
#include <stdlib.h>

/* This union's purpose is to be as big as the largest of all the
 * types it contains. */
union alignment {
    size_t sz;
    void *ptr;
    double dbl;
};
/* We need to make sure that everything we return is on the right
   alignment to hold anything, including a double. */
#define ALIGNMENT sizeof(union alignment)

/* We need to do this cast-to-char* trick on our pointers to adjust
   them; doing arithmetic on a void* is not standard. */
#define OUTPTR(ptr) (((char*)ptr)+ALIGNMENT)
#define INPTR(ptr) (((char*)ptr)-ALIGNMENT)

static size_t total_allocated = 0;
static void *replacement_malloc(size_t sz)
{
    void *chunk = malloc(sz + ALIGNMENT);
    if (!chunk) return chunk;
    total_allocated += sz;
    *(size_t*)chunk = sz;
    return OUTPTR(chunk);
}
static void *replacement_realloc(void *ptr, size_t sz)
{
    size_t old_size = 0;
    if (ptr) {
        ptr = INPTR(ptr);
        old_size = *(size_t*)ptr;
    }
    ptr = realloc(ptr, sz + ALIGNMENT);
    if (!ptr)
        return NULL;
    *(size_t*)ptr = sz;
    total_allocated = total_allocated - old_size + sz;
    return OUTPTR(ptr);
}
static void replacement_free(void *ptr)
{
    ptr = INPTR(ptr);
    total_allocated -= *(size_t*)ptr;
    free(ptr);
}
void start_counting_bytes(void)
{
    event_set_mem_functions(replacement_malloc,
                            replacement_realloc,
                            replacement_free);
}
```
注意

* 替换内存管理函数影响libevent随后的所有分配、调整大小和释放内存操作。所以，必须保证在调用任何其他libevent函数之前进行替换。否则，libevent可能用你的free函数释放用C库的malloc分配的内存。
* 你的malloc和realloc函数返回的内存块应该具有和C库返回的内存块一样的地址对齐。
* 你的realloc函数应该正确处理realloc(NULL,sz)（也就是当作malloc(sz)处理）
* 你的realloc函数应该正确处理realloc(ptr,0)（也就是当作free(ptr)处理）
* 你的free函数不必处理free(NULL)
* 你的malloc函数不必处理malloc(0)
* 如果在多个线程中使用libevent，替代的内存管理函数需要是线程安全的。
* libevent将使用这些函数分配返回给你的内存。所以，如果要释放由libevent函数分配和返回的内存，而你已经替换malloc和realloc函数，那么应该使用替代的free函数。
 
event_set_mem_functions函数声明在<event2/event.h>中，在libevent 2.0.1-alpha版本中首次出现。

可以在禁止event_set_mem_functions函数的配置下编译libevent。这时候使用event_set_mem_functions将不会编译或者链接。在2.0.2-alpha及以后版本中，可以通过检查是否定义了EVENT_SET_MEM_FUNCTIONS_IMPLEMENTED宏来确定event_set_mem_functions函数是否存在。

### 4 锁和线程

编写多线程程序的时候，在多个线程中同时访问同样的数据并不总是安全的。
libevent的结构体在多线程下通常有三种工作方式：

* 某些结构体内在地是单线程的：同时在多个线程中使用它们总是不安全的。
* 某些结构体具有可选的锁：可以告知libevent是否需要在多个线程中使用每个对象。
* 某些结构体总是锁定的：如果libevent在支持锁的配置下运行，在多个线程中使用它们总是安全的。
* 
为获取锁，在调用分配需要在多个线程间共享的结构体的libevent函数之前，必须告知libevent使用哪个锁函数。

如果使用pthreads库，或者使用Windows本地线程代码，那么你是幸运的：已经有设置libevent使用正确的pthreads或者Windows函数的预定义函数。

接口
```c
#ifdef WIN32
int evthread_use_windows_threads(void);
#define EVTHREAD_USE_WINDOWS_THREADS_IMPLEMENTED
#endif
#ifdef _EVENT_HAVE_PTHREADS
int evthread_use_pthreads(void);
#define EVTHREAD_USE_PTHREADS_IMPLEMENTED
#endif
```
这些函数在成功时都返回0，失败时返回-1。
如果使用不同的线程库，则需要一些额外的工作，必须使用你的线程库来定义函数去实现：
* 锁
* 锁定
* 解锁
* 分配锁
* 析构锁
* 条件变量
* 创建条件变量
* 析构条件变量
* 等待条件变量
* 触发/广播某条件变量
* 线程
* 线程ID检测
* 
使用evthread_set_lock_callbacks和evthread_set_id_callback接口告知libevent这些函数。

接口
```c
#define EVTHREAD_WRITE  0x04
#define EVTHREAD_READ   0x08
#define EVTHREAD_TRY    0x10

#define EVTHREAD_LOCKTYPE_RECURSIVE 1
#define EVTHREAD_LOCKTYPE_READWRITE 2

#define EVTHREAD_LOCK_API_VERSION 1

struct evthread_lock_callbacks {
       int lock_api_version;
       unsigned supported_locktypes;
       void *(*alloc)(unsigned locktype);
       void (*free)(void *lock, unsigned locktype);
       int (*lock)(unsigned mode, void *lock);
       int (*unlock)(unsigned mode, void *lock);
};

int evthread_set_lock_callbacks(const struct evthread_lock_callbacks *);

void evthread_set_id_callback(unsigned long (*id_fn)(void));

struct evthread_condition_callbacks {
        int condition_api_version;
        void *(*alloc_condition)(unsigned condtype);
        void (*free_condition)(void *cond);
        int (*signal_condition)(void *cond, int broadcast);
        int (*wait_condition)(void *cond, void *lock,
            const struct timeval *timeout);
};

int evthread_set_condition_callbacks(
        const struct evthread_condition_callbacks *);
```

     
evthread_lock_callbacks结构体描述的锁回调函数及其能力。对于上述版本，lock_api_version字段必须设置为EVTHREAD_LOCK_API_VERSION。必须设置supported_locktypes字段为EVTHREAD_LOCKTYPE_*常量的组合以描述支持的锁类型（在2.0.4-alpha版本中，EVTHREAD_LOCK_RECURSIVE是必须的，EVTHREAD_LOCK_READWRITE则没有使用）。alloc函数必须返回指定类型的新锁；free函数必须释放指定类型锁持有的所有资源；lock函数必须试图以指定模式请求锁定，如果成功则返回0，失败则返回非零；unlock函数必须试图解锁，成功则返回0，否则返回非零。

可识别的锁类型有：

* 0：通常的，不必递归的锁。
* EVTHREAD_LOCKTYPE_RECURSIVE：不会阻塞已经持有它的线程的锁。一旦持有它的线程进行原来锁定次数的解锁，其他线程立刻就可以请求它了。
* EVTHREAD_LOCKTYPE_READWRITE：可以让多个线程同时因为读而持有它，但是任何时刻只有一个线程因为写而持有它。写操作排斥所有读操作。

可识别的锁模式有：

* EVTHREAD_READ：仅用于读写锁：为读操作请求或者释放锁
* EVTHREAD_WRITE：仅用于读写锁：为写操作请求或者释放锁
* EVTHREAD_TRY：仅用于锁定：仅在可以立刻锁定的时候才请求锁定

	
id_fn参数必须是一个函数，它返回一个无符号长整数，标识调用此函数的线程。对于相同线程，这个函数应该总是返回同样的值；而对于同时调用该函数的不同线程，必须返回不同的值。
evthread_condition_callbacks结构体描述了与条件变量相关的回调函数。对于上述版本，condition_api_version字段必须设置为EVTHREAD_CONDITION_API_VERSION。alloc_condition函数必须返回到新条件变量的指针。它接受0作为其参数。free_condition函数必须释放条件变量持有的存储器和资源。wait_condition函数要求三个参数：一个由alloc_condition分配的条件变量，一个由你提供的evthread_lock_callbacks.alloc函数分配的锁，以及一个可选的超时值。调用本函数时，必须已经持有参数指定的锁；本函数应该释放指定的锁，等待条件变量成为授信状态，或者直到指定的超时时间已经流逝（可选）。wait_condition应该在错误时返回-1，条件变量授信时返回0，超时时返回1。返回之前，函数应该确定其再次持有锁。最后，signal_condition函数应该唤醒等待该条件变量的某个线程（broadcast参数为false时），或者唤醒等待条件变量的所有线程（broadcast参数为true时）。只有在持有与条件变量相关的锁的时候，才能够进行这些操作。
关于条件变量的更多信息，请查看pthreads的pthread_cond_*函数文档，或者Windows的CONDITION_VARIABLE（Windows Vista新引入的）函数文档。

示例

关于使用这些函数的示例，请查看Libevent源代码发布版本中的evthread_pthread.c和evthread_win32.c文件。
这些函数在<event2/thread.h>中声明，其中大多数在2.0.4-alpha版本中首次出现。2.0.1-alpha到2.0.3-alpha使用较老版本的锁函数。event_use_pthreads函数要求程序链接到event_pthreads库。
条件变量函数是2.0.7-rc版本新引入的，用于解决某些棘手的死锁问题。
可以创建禁止锁支持的libevent。这时候已创建的使用上述线程相关函数的程序将不能运行。

### 5 调试锁的使用

为帮助调试锁的使用，libevent有一个可选的“锁调试”特征。这个特征包装了锁调用，以便捕获典型的锁错误，包括：

* 解锁并没有持有的锁
* 重新锁定一个非递归锁

如果发生这些错误中的某一个，libevent将给出断言失败并且退出。

接口
```c
void evthread_enable_lock_debugging(void);
#define evthread_enable_lock_debuging() evthread_enable_lock_debugging()
```
注意

必须在创建或者使用任何锁之前调用这个函数。为安全起见，请在设置完线程函数后立即调用这个函数。
这个函数是在2.0.4-alpha版本新引入的。

### 6 调试事件的使用

libevent可以检测使用事件时的一些常见错误并且进行报告。这些错误包括：

* 将未初始化的event结构体当作已经初始化的
* 试图重新初始化未决的event结构体

跟踪哪些事件已经初始化需要使用额外的内存和处理器时间，所以只应该在真正调试程序的时候才启用调试模式。

接口
```c
void event_enable_debug_mode(void);
```

必须在创建任何event_base之前调用这个函数。
 

如果在调试模式下使用大量由event_assign（而不是event_new）创建的事件，程序可能会耗尽内存，这是因为没有方式可以告知libevent由event_assign创建的事件不会再被使用了（可以调用event_free告知由event_new创建的事件已经无效了）。如果想在调试时避免耗尽内存，可以显式告知libevent这些事件不再被当作已分配的了：

接口
```c
void event_debug_unassign(struct event *ev);
```

没有启用调试的时候调用event_debug_unassign没有效果。
这些调试函数在libevent 2.0.4-alpha版本中加入。

### 7 检测libevent的版本

新版本的libevent会添加特征，移除bug。有时候需要检测libevent的版本，以便：

* 检测已安装的libevent版本是否可用于创建你的程序
* 为调试显示libevent的版本
* 检测libevent的版本，以便向用户警告bug，或者提示要做的工作

接口
```c
#define LIBEVENT_VERSION_NUMBER 0x02000300
#define LIBEVENT_VERSION "2.0.3-alpha"
const char *event_get_version(void);
ev_uint32_t event_get_version_number(void);
```

宏返回编译时的libevent版本；函数返回运行时的libevent版本。注意：如果动态链接到libevent，这两个版本可能不同。

可以获取两种格式的libevent版本：用于显示给用户的字符串版本，或者用于数值比较的4字节整数版本。整数格式使用高字节表示主版本，低字节表示副版本，第三字节表示修正版本，最低字节表示发布状态：0表示发布，非零表示某特定发布版本的后续开发序列。

所以，libevent 2.0.1-alpha发布版本的版本号是[02 00 01 00]，或者说0x02000100。2.0.1-alpha和2.0.2-alpha之间的开发版本可能是[02 00 01 08]，或者说0x02000108。