## 三
使用libevent函数之前需要分配一个或者多个event_base结构体。每个event_base结构体持有一个事件集合，可以检测以确定哪个事件是激活的。
如果设置event_base使用锁，则可以安全地在多个线程中访问它。然而，其事件循环只能运行在一个线程中。如果需要用多个线程检测IO，则需要为每个线程使用一个event_base。
每个event_base都有一种用于检测哪种事件已经就绪的“方法”，或者说后端。可以识别的方法有：
v select
v poll
v epoll
v kqueue
v devpoll
v evport
v win32
用户可以用环境变量禁止某些特定的后端。比如说，要禁止kqueue后端，可以设置EVENT_NOKQUEUE环境变量。如果要用编程的方法禁止后端，请看下面关于event_config_avoid_method（）的说明。
1 建立默认的event_base
event_base_new（）函数分配并且返回一个新的具有默认设置的event_base。函数会检测环境变量，返回一个到event_base的指针。如果发生错误，则返回NULL。选择各种方法时，函数会选择OS支持的最快方法。
接口
这个函数当前仅在Windows上使用IOCP时有用，虽然将来可能在其他平台上有用。这个函数告诉event_config在生成多线程event_base的时候，应该试图使用给定数目的CPU。注意这仅仅是一个提示：event_base使用的CPU可能比你选择的要少。
示例

struct event_base *event_base_new(void);

大多数程序使用这个函数就够了。
event_base_new（）函数声明在<event2/event.h>中，首次出现在libevent 1.4.3版。
 
2 创建复杂的event_base
要对取得什么类型的event_base有更多的控制，就需要使用event_config。event_config是一个容纳event_base配置信息的不透明结构体。需要event_base时，将event_config传递给event_base_new_with_config（）。
2.1 接口

struct event_config *event_config_new(void);
struct event_base *event_base_new_with_config(const struct event_config *cfg);
void event_config_free(struct event_config *cfg);
要使用这些函数分配event_base，先调用event_config_new（）分配一个event_config。然后，对event_config调用其它函数，设置所需要的event_base特征。最后，调用event_base_new_with_config（）获取新的event_base。完成工作后，使用event_config_free（）释放event_config。
2.2 接口

int event_config_avoid_method(struct event_config *cfg, const char *method);

enum event_method_feature {
    EV_FEATURE_ET = 0x01,
    EV_FEATURE_O1 = 0x02,
    EV_FEATURE_FDS = 0x04,
};
int event_config_require_features(struct event_config *cfg,
                                  enum event_method_feature feature);

enum event_base_config_flag {
    EVENT_BASE_FLAG_NOLOCK = 0x01,
    EVENT_BASE_FLAG_IGNORE_ENV = 0x02,
    EVENT_BASE_FLAG_STARTUP_IOCP = 0x04,
    EVENT_BASE_FLAG_NO_CACHE_TIME = 0x08,
    EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST = 0x10,
    EVENT_BASE_FLAG_PRECISE_TIMER = 0x20
};
int event_config_set_flag(struct event_config *cfg,
    enum event_base_config_flag flag);
调用event_config_avoid_method（）可以通过名字让libevent避免使用特定的可用后端。调用event_config_require_feature（）让libevent不使用不能提供所有指定特征的后端。调用event_config_set_flag（）让libevent在创建event_base时设置一个或者多个将在下面介绍的运行时标志。
event_config_require_features（）可识别的特征值有：

v EV_FEATURE_ET：要求支持边沿触发的后端
v EV_FEATURE_O1：要求添加、删除单个事件，或者确定哪个事件激活的操作是O（1）复杂度的后端
v EV_FEATURE_FDS：要求支持任意文件描述符，而不仅仅是套接字的后端
event_config_set_flag（）可识别的选项值有：

v EVENT_BASE_FLAG_NOLOCK：不要为event_base分配锁。设置这个选项可以为event_base节省一点用于锁定和解锁的时间，但是让在多个线程中访问event_base成为不安全的。
v EVENT_BASE_FLAG_IGNORE_ENV：选择使用的后端时，不要检测EVENT_*环境变量。使用这个标志需要三思：这会让用户更难调试你的程序与libevent的交互。
v EVENT_BASE_FLAG_STARTUP_IOCP：仅用于Windows，让libevent在启动时就启用任何必需的IOCP分发逻辑，而不是按需启用。
v EVENT_BASE_FLAG_NO_CACHE_TIME：不是在事件循环每次准备执行超时回调时检测当前时间，而是在每次超时回调后进行检测。注意：这会消耗更多的CPU时间。
v EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST：告诉libevent，如果决定使用epoll后端，可以安全地使用更快的基于changelist的后端。epoll-changelist后端可以在后端的分发函数调用之间，同样的fd多次修改其状态的情况下，避免不必要的系统调用。但是如果传递任何使用dup（）或者其变体克隆的fd给libevent，epoll-changelist后端会触发一个内核bug，导致不正确的结果。在不使用epoll后端的情况下，这个标志是没有效果的。也可以通过设置EVENT_EPOLL_USE_CHANGELIST环境变量来打开epoll-changelist选项。
上述操作event_config的函数都在成功时返回0，失败时返回-1。
注意

设置event_config，请求OS不能提供的后端是很容易的。比如说，对于libevent 2.0.1-alpha，在Windows中是没有O（1）后端的；在Linux中也没有同时提供EV_FEATURE_FDS和EV_FEATURE_O1特征的后端。如果创建了libevent不能满足的配置，event_base_new_with_config（）会返回NULL。
2.3 接口

int event_config_set_num_cpus_hint(struct event_config *cfg, int cpus)
这个函数当前仅在Windows上使用IOCP时有用，虽然将来可能在其他平台上有用。这个函数告诉event_config在生成多线程event_base的时候，应该试图使用给定数目的CPU。注意这仅仅是一个提示：event_base使用的CPU可能比你选择的要少。
示例

struct event_config *cfg;
struct event_base *base;
int i;

/* My program wants to use edge-triggered events if at all possible.  So
   I'll try to get a base twice: Once insisting on edge-triggered IO, and
   once not. */
for (i=0; i<2; ++i) {
    cfg = event_config_new();

    /* I don't like select. */
    event_config_avoid_method(cfg, "select");

    if (i == 0)
        event_config_require_features(cfg, EV_FEATURE_ET);

    base = event_base_new_with_config(cfg);
    event_config_free(cfg);
    if (base)
        break;

    /* If we get here, event_base_new_with_config() returned NULL.  If
       this is the first time around the loop, we'll try again without
       setting EV_FEATURE_ET.  If this is the second time around the
       loop, we'll give up. */
}
这些函数和类型在<event2/event.h>中声明。
EVENT_BASE_FLAG_IGNORE_ENV标志首次出现在2.0.2-alpha版本。event_config_set_num_cpus_hint（）函数是2.0.7-rc版本新引入的。本节的其他内容首次出现在2.0.1-alpha版本。
3 检查event_base的后端方法
有时候需要检查event_base支持哪些特征，或者当前使用哪种方法。

3.1 接口

const char **event_get_supported_methods(void);
event_get_supported_methods（）函数返回一个指针，指向libevent支持的方法名字数组。这个数组的最后一个元素是NULL。
示例

int i;
const char **methods = event_get_supported_methods();
printf("Starting Libevent %s.  Available methods are:\n",
    event_get_version());
for (i=0; methods[i] != NULL; ++i) {
    printf("    %s\n", methods[i]);
}
注意

这个函数返回libevent被编译以支持的方法列表。然而libevent运行的时候，操作系统可能不能支持所有方法。比如说，可能OS X版本中的kqueue的bug太多，无法使用。
3.2 接口

const char *event_base_get_method(const struct event_base *base);
enum event_method_feature event_base_get_features(const struct event_base *base);
event_base_get_method（）返回event_base正在使用的方法。event_base_get_features（）返回event_base支持的特征的比特掩码。
示例

struct event_base *base;
enum event_method_feature f;

base = event_base_new();
if (!base) {
    puts("Couldn't get an event_base!");
} else {
    printf("Using Libevent with backend method %s.",
        event_base_get_method(base));
    f = event_base_get_features(base);
    if ((f & EV_FEATURE_ET))
        printf("  Edge-triggered events are supported.");
    if ((f & EV_FEATURE_O1))
        printf("  O(1) event notification is supported.");
    if ((f & EV_FEATURE_FDS))
        printf("  All FD types are supported.");
    puts("");
}
这个函数定义在<event2/event.h>中。event_base_get_method（）首次出现在1.4.3版本中，其他函数首次出现在2.0.1-alpha版本中。
4 释放event_base
使用完event_base之后，使用event_base_free（）进行释放。
接口

void event_base_free(struct event_base *base);
注意：这个函数不会释放当前与event_base关联的任何事件，或者关闭他们的套接字，或者释放任何指针。
event_base_free（）定义在<event2/event.h>中，首次由libevent 1.2实现。
5 设置event_base的优先级
libevent支持为事件设置多个优先级。然而，event_base默认只支持单个优先级。可以调用event_base_priority_init（）设置event_base的优先级数目。
接口

int event_base_priority_init(struct event_base *base, int n_priorities);

成功时这个函数返回0，失败时返回-1。base是要修改的event_base，n_priorities是要支持的优先级数目，这个数目至少是1。每个新的事件可用的优先级将从0（最高）到n_priorities-1（最低）。
常量EVENT_MAX_PRIORITIES表示n_priorities的上限。调用这个函数时为n_priorities给出更大的值是错误的。
注意

必须在任何事件激活之前调用这个函数，最好在创建event_base后立刻调用。
示例

关于示例，请看event_priority_set的文档。
默认情况下，与event_base相关联的事件将被初始化为具有优先级n_priorities / 2。event_base_priority_init（）函数定义在<event2/event.h>中，从libevent 1.0版就可用了。
6 在fork（）之后重新初始化event_base
不是所有事件后端都在调用fork（）之后可以正确工作。所以，如果在使用fork（）或者其他相关系统调用启动新进程之后，希望在新进程中继续使用event_base，就需要进行重新初始化。
接口

int event_reinit(struct event_base *base);
成功时这个函数返回0，失败时返回-1。
示例

struct event_base *base = event_base_new();

/*  add some events to the event_base  */

if (fork()) {
    /* In parent */
    continue_running_parent(base); /**/
} else {
    /* In child */
    event_reinit(base);
    continue_running_child(base); /**/
}
event_reinit（）定义在<event2/event.h>中，在libevent 1.4.3-alpha版中首次可用。
7 废弃的event_base函数
老版本的libevent严重依赖“当前”event_base的概念。“当前”event_base是一个由所有线程共享的全局设置。如果忘记指定要使用哪个event_base，则得到的是当前的。因为event_base不是线程安全的，这很容易导致错误。
老版本的libevent没有event_base_new（），而有：
接口

struct event_base *event_init(void);
这个函数的工作与event_base_new（）类似，它将分配的event_base设置成当前的。没有其他方法改变当前event_base。
本文描述的函数有一些用于操作当前event_base的变体，这些函数与新版本函数的行为类似，只是它们没有event_base参数。