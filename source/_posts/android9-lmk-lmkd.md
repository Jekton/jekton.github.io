---
title: Android P 源码分析 5 - Low memory killer 之 lmkd 守护进程
date: 2019-03-21 19:28:58
categories: Android
tags: [Android source]
description: lmkd 是在应用层实现的取代原有 lowmemorykiller 驱动的守护进程。通过监听 memory pressure 事件，lmkd 可以在内存 low、medium 和 critical 的时候得到通知，进而回收优先级比较低的进程
---

本来按顺序这一篇应该也还是 logd，但我刚开始写就碰到了 cgroup，一顿搜索又扯上了 lmk，没办法，只能先解决这拦路的石头，然后再继续 logd。

Android 早先的版本的 lmk 是以驱动的形式在内核中实现的，这种方式并不为主线内核所接受。后来有人给内核添加了 memory pressure event，这就为应用层实现 lmk 提供了可能性。通过监听 memory pressure 事件，应用可以在内存 low、medium 和 critical 的时候得到通知，从而回收一些优先级比较低的应用。

下面我们就一起来看看他的实现。

## 应用初始化

跟大多数守护进程一样，lmkd 也是由 init 进程启动的：
```
// system/core/lmkd/lmkd.rc
service lmkd /system/bin/lmkd
    class core
    group root readproc
    critical
    socket lmkd seqpacket 0660 system system
    writepid /dev/cpuset/system-background/tasks
```
这里创建的 socket lmkd 的 user/group 都是 system，而它的权限是 0660，所以只有 system 应用才能读写（一般是 activity manager）。

接下来的 writepid 跟 Linux 的 cgroups 相关，目前我也不太了解（流下了没技术的泪水），后面补上相关的知识后再来单独撸一篇（文章）。

应用启动后，开始执行 `main` 函数，`main` 函数主要做三件事：
1. 读取配置参数
2. 锁住内存页并设置进程调度器
3. 初始化 epoll 事件监听
4. 循环处理事件

为了让读者有整体感，这里我们先把一整个 `main` 函数放上了，然后再单独看各个部分的实现。
```C
// system/core/lmkd/lmkd.c
int main(int argc __unused, char **argv __unused) {
    struct sched_param param = {
            .sched_priority = 1,
    };

    /* By default disable low level vmpressure events */
    level_oomadj[VMPRESS_LEVEL_LOW] =
        property_get_int32("ro.lmk.low", OOM_SCORE_ADJ_MAX + 1);
    level_oomadj[VMPRESS_LEVEL_MEDIUM] =
        property_get_int32("ro.lmk.medium", 800);
    level_oomadj[VMPRESS_LEVEL_CRITICAL] =
        property_get_int32("ro.lmk.critical", 0);
    debug_process_killing = property_get_bool("ro.lmk.debug", false);

    /* By default disable upgrade/downgrade logic */
    enable_pressure_upgrade =
        property_get_bool("ro.lmk.critical_upgrade", false);
    upgrade_pressure =
        (int64_t)property_get_int32("ro.lmk.upgrade_pressure", 100);
    downgrade_pressure =
        (int64_t)property_get_int32("ro.lmk.downgrade_pressure", 100);
    kill_heaviest_task =
        property_get_bool("ro.lmk.kill_heaviest_task", false);
    low_ram_device = property_get_bool("ro.config.low_ram", false);
    kill_timeout_ms =
        (unsigned long)property_get_int32("ro.lmk.kill_timeout_ms", 0);
    use_minfree_levels =
        property_get_bool("ro.lmk.use_minfree_levels", false);

#ifdef LMKD_LOG_STATS
    statslog_init(&log_ctx, &enable_stats_log);
#endif

    // MCL_ONFAULT pins pages as they fault instead of loading
    // everything immediately all at once. (Which would be bad,
    // because as of this writing, we have a lot of mapped pages we
    // never use.) Old kernels will see MCL_ONFAULT and fail with
    // EINVAL; we ignore this failure.
    //
    // N.B. read the man page for mlockall. MCL_CURRENT | MCL_ONFAULT
    // pins ⊆ MCL_CURRENT, converging to just MCL_CURRENT as we fault
    // in pages.
    if (mlockall(MCL_CURRENT | MCL_FUTURE | MCL_ONFAULT) && errno != EINVAL)
        ALOGW("mlockall failed: errno=%d", errno);

    sched_setscheduler(0, SCHED_FIFO, &param);
    if (!init())
        mainloop();

#ifdef LMKD_LOG_STATS
    statslog_destroy(&log_ctx);
#endif

    ALOGI("exiting");
    return 0;
}
```

### 读取配置参数
```C
// system/core/lmkd/lmkd.c

/* memory pressure levels */
enum vmpressure_level {
    VMPRESS_LEVEL_LOW = 0,
    VMPRESS_LEVEL_MEDIUM,
    VMPRESS_LEVEL_CRITICAL,
    VMPRESS_LEVEL_COUNT
};

static int level_oomadj[VMPRESS_LEVEL_COUNT];
static int mpevfd[VMPRESS_LEVEL_COUNT] = { -1, -1, -1 };
static bool debug_process_killing;
static bool enable_pressure_upgrade;
static int64_t upgrade_pressure;
static int64_t downgrade_pressure;
static bool low_ram_device;
static bool kill_heaviest_task;
static unsigned long kill_timeout_ms;
static bool use_minfree_levels;

int main(int argc __unused, char **argv __unused) {

    /* By default disable low level vmpressure events */
    level_oomadj[VMPRESS_LEVEL_LOW] =
        property_get_int32("ro.lmk.low", OOM_SCORE_ADJ_MAX + 1);
    level_oomadj[VMPRESS_LEVEL_MEDIUM] =
        property_get_int32("ro.lmk.medium", 800);
    level_oomadj[VMPRESS_LEVEL_CRITICAL] =
        property_get_int32("ro.lmk.critical", 0);
    debug_process_killing = property_get_bool("ro.lmk.debug", false);

    /* By default disable upgrade/downgrade logic */
    enable_pressure_upgrade =
        property_get_bool("ro.lmk.critical_upgrade", false);
    upgrade_pressure =
        (int64_t)property_get_int32("ro.lmk.upgrade_pressure", 100);
    downgrade_pressure =
        (int64_t)property_get_int32("ro.lmk.downgrade_pressure", 100);
    kill_heaviest_task =
        property_get_bool("ro.lmk.kill_heaviest_task", false);
    low_ram_device = property_get_bool("ro.config.low_ram", false);
    kill_timeout_ms =
        (unsigned long)property_get_int32("ro.lmk.kill_timeout_ms", 0);
    use_minfree_levels =
        property_get_bool("ro.lmk.use_minfree_levels", false);

    ...
    ...
    ...
}
```

这段代码很简单，就是直接从系统属性里面读配置，然后放到静态变量里。关于这些属性的含义，读者可以参考 `system/core/lmkd/README.md`。

`enum vmpressure_level` 代表了内存压力等级，分别是我们前面提到的 low、medium 和 critical。

### 锁住内存页并设置进程调度器

```C
int main(int argc __unused, char **argv __unused) {
    // 读取配置参数

    // MCL_ONFAULT pins pages as they fault instead of loading
    // everything immediately all at once. (Which would be bad,
    // because as of this writing, we have a lot of mapped pages we
    // never use.) Old kernels will see MCL_ONFAULT and fail with
    // EINVAL; we ignore this failure.
    //
    // N.B. read the man page for mlockall. MCL_CURRENT | MCL_ONFAULT
    // pins ⊆ MCL_CURRENT, converging to just MCL_CURRENT as we fault
    // in pages.
    if (mlockall(MCL_CURRENT | MCL_FUTURE | MCL_ONFAULT) && errno != EINVAL)
        ALOGW("mlockall failed: errno=%d", errno);

    struct sched_param param = {
            .sched_priority = 1,
    };
    sched_setscheduler(0, SCHED_FIFO, &param);

    // 初始化 epoll 事件监听
    // 循环处理事件
}
```
`MCL_CURRENT` 把应用里当前已经在内存中的页锁着；`MCL_FUTURE` 把以后分配的内存区锁着；`MCL_ONFAULT` 表示不把那些当前不在内存里的页都加载到内存，而是当 page fault 的时候，再将加载进来的页面锁着。关于 `mlockall` 的更多信息，读者可以参考 man page。

接下来的 `sched_setscheduler` 将自己设置为实时进程，使用的调度器是 fifo。实时进程的优先级高于所有普通进程。对 fifo 调度器来说，在进程可运行（runnable）的时候，内核不会抢占它。


### 初始化 epoll 事件监听
```C
int main(int argc __unused, char **argv __unused) {
    // 读取配置参数
    // 锁住内存页并设置进程调度器

    if (!init())
        mainloop();

    return 0;
}
```
epoll 的初始化由 `init` 函数完成，`mainloop` 在一个主循环里处理事件，后者我们在下一小节讨论。
```C
static int init(void) {
    struct epoll_event epev;
    int i;
    int ret;

    page_k = sysconf(_SC_PAGESIZE);
    if (page_k == -1)
        page_k = PAGE_SIZE;
    page_k /= 1024;

    epollfd = epoll_create(MAX_EPOLL_EVENTS);
    if (epollfd == -1) {
        ALOGE("epoll_create failed (errno=%d)", errno);
        return -1;
    }

    // mark data connections as not connected
    for (int i = 0; i < MAX_DATA_CONN; i++) {
        data_sock[i].sock = -1;
    }

    ctrl_sock.sock = android_get_control_socket("lmkd");
    if (ctrl_sock.sock < 0) {
        ALOGE("get lmkd control socket failed");
        return -1;
    }

    ret = listen(ctrl_sock.sock, MAX_DATA_CONN);
    if (ret < 0) {
        ALOGE("lmkd control socket listen failed (errno=%d)", errno);
        return -1;
    }

    epev.events = EPOLLIN;
    ctrl_sock.handler_info.handler = ctrl_connect_handler;
    epev.data.ptr = (void *)&(ctrl_sock.handler_info);
    if (epoll_ctl(epollfd, EPOLL_CTL_ADD, ctrl_sock.sock, &epev) == -1) {
        ALOGE("epoll_ctl for lmkd control socket failed (errno=%d)", errno);
        return -1;
    }
    maxevents++;

    has_inkernel_module = !access(INKERNEL_MINFREE_PATH, W_OK);
    use_inkernel_interface = has_inkernel_module;

    if (use_inkernel_interface) {
        ALOGI("Using in-kernel low memory killer interface");
    } else {
        if (!init_mp_common(VMPRESS_LEVEL_LOW) ||
            !init_mp_common(VMPRESS_LEVEL_MEDIUM) ||
            !init_mp_common(VMPRESS_LEVEL_CRITICAL)) {
            ALOGE("Kernel does not support memory pressure events or in-kernel low memory killer");
            return -1;
        }
    }

    for (i = 0; i <= ADJTOSLOT(OOM_SCORE_ADJ_MAX); i++) {
        procadjslot_list[i].next = &procadjslot_list[i];
        procadjslot_list[i].prev = &procadjslot_list[i];
    }

    return 0;
}
```

这里初始化静态变量 `page_k`，`page_k` 表示一个内存页有多少 KB，默认的 `PAGE_SIZE` 为 4096。
```C
static long page_k;

/* data required to handle events */
struct event_handler_info {
    int data;
    void (*handler)(int data, uint32_t events);
};

/* data required to handle socket events */
struct sock_event_handler_info {
    int sock;
    struct event_handler_info handler_info;
};

/* max supported number of data connections */
#define MAX_DATA_CONN 2

/* socket event handler data */
static struct sock_event_handler_info ctrl_sock;
static struct sock_event_handler_info data_sock[MAX_DATA_CONN];
```

`ctrl_sock` 是由 init 进程帮我们创建的那个 socket lmkd，通过这个 socket，lmkd 进程监听这个 socket 并等待接受客户的连接。lmkd 最多支持两个客户（`MAX_DATA_CONN`），这两个客户的 socket fd 就放在 `data_sock`。

`struct event_handler_info` 定义了一个接口，`epoll` 返回时，主循环里就这个事件对应的 `handler`。调用时，第一个参数 `data` 是用户在创建 `struct event_handler_info` 对象时初始化的字段 `data`，第二个参数 `events` 是 `epoll` 返回的事件。

从上面的初始化代码可以看出，当 socket lmkd 有客户连接时，对应的回调是 `ctrl_connect_handler`。这里我们没有初始化 `ctrl_sock.handler_info.data`，是因为 `ctrl_connect_handler` 不使用这个额外的参数。

`INKERNEL_MINFREE_PATH` 宏是 `/sys/module/lowmemorykiller/parameters/minfree`，如果我们不能访问它，表示 lowmemorykiller 使用的是内核中的驱动（没错，目前有两个 lowmemorykiller，一个是我们现在在看的 lmkd，一个是在内核中实现的驱动。系统的编译的时候可以决定使用哪一个）。这里我们假设使用应用层 lmk。

`init_mp_common` 函数初始化 memory pressure 事件的监听：
```C++
#define MEMCG_SYSFS_PATH "/dev/memcg/"

static int mpevfd[VMPRESS_LEVEL_COUNT] = { -1, -1, -1 };

static bool init_mp_common(enum vmpressure_level level) {
    int mpfd;
    int evfd;
    int evctlfd;
    char buf[256];
    struct epoll_event epev;
    int ret;
    int level_idx = (int)level;
    const char *levelstr = level_name[level_idx];

    mpfd = open(MEMCG_SYSFS_PATH "memory.pressure_level", O_RDONLY | O_CLOEXEC);
    if (mpfd < 0) {
        ALOGI("No kernel memory.pressure_level support (errno=%d)", errno);
        goto err_open_mpfd;
    }

    evctlfd = open(MEMCG_SYSFS_PATH "cgroup.event_control", O_WRONLY | O_CLOEXEC);
    if (evctlfd < 0) {
        ALOGI("No kernel memory cgroup event control (errno=%d)", errno);
        goto err_open_evctlfd;
    }

    evfd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
    if (evfd < 0) {
        ALOGE("eventfd failed for level %s; errno=%d", levelstr, errno);
        goto err_eventfd;
    }

    ret = snprintf(buf, sizeof(buf), "%d %d %s", evfd, mpfd, levelstr);
    if (ret >= (ssize_t)sizeof(buf)) {
        ALOGE("cgroup.event_control line overflow for level %s", levelstr);
        goto err;
    }

    ret = TEMP_FAILURE_RETRY(write(evctlfd, buf, strlen(buf) + 1));
    if (ret == -1) {
        ALOGE("cgroup.event_control write failed for level %s; errno=%d",
              levelstr, errno);
        goto err;
    }

    epev.events = EPOLLIN;
    /* use data to store event level */
    vmpressure_hinfo[level_idx].data = level_idx;
    vmpressure_hinfo[level_idx].handler = mp_event_common;
    epev.data.ptr = (void *)&vmpressure_hinfo[level_idx];
    ret = epoll_ctl(epollfd, EPOLL_CTL_ADD, evfd, &epev);
    if (ret == -1) {
        ALOGE("epoll_ctl for level %s failed; errno=%d", levelstr, errno);
        goto err;
    }
    maxevents++;
    mpevfd[level] = evfd;
    close(evctlfd);
    return true;

err:
    close(evfd);
err_eventfd:
    close(evctlfd);
err_open_evctlfd:
    close(mpfd);
err_open_mpfd:
    return false;
}
```
`mpfd` 打开的时候利用 C/C++ 的字符串连接特性（相邻的字符串会自动由编译器连接到一起），他打开的实际路径是 `/dev/memcg/memory.pressure_level`。`evctlfd` 打开的路径类似。`eventfd` 是 LINUX 特有的一个系统调用，读者可以查看 man page 了解它的用法。

这里我们只需要知道，通过这么一顿骚操作之后，我们就能够通过描述符 `evfd` 得到 memory pressure 事件，就暂时把内核当做一个黑匣子来使用吧。

对于 memory pressure 事件，处理函数是 `mp_event_common`，传递给他的 `data` 是 `level_idx`（用于区分是发生了哪个级别的 mp 事件）。

最后要留意的是，`evfd` 被我们存放在了静态数组 `mpevfd` 里了，在后面将进程回收的时候（`mp_event_common` 函数）我们还会用到这个 `mpevfd`。

### 循环处理事件

事件循环由 `mainloop` 函数实现：
```C
static void mainloop(void) {
    struct event_handler_info* handler_info;
    struct epoll_event *evt;

    while (1) {
        struct epoll_event events[maxevents];
        int nevents;
        int i;

        nevents = epoll_wait(epollfd, events, maxevents, -1);

        if (nevents == -1) {
            if (errno == EINTR)
                continue;
            ALOGE("epoll_wait failed (errno=%d)", errno);
            continue;
        }

        /*
         * First pass to see if any data socket connections were dropped.
         * Dropped connection should be handled before any other events
         * to deallocate data connection and correctly handle cases when
         * connection gets dropped and reestablished in the same epoll cycle.
         * In such cases it's essential to handle connection closures first.
         */
        for (i = 0, evt = &events[0]; i < nevents; ++i, evt++) {
            if ((evt->events & EPOLLHUP) && evt->data.ptr) {
                ALOGI("lmkd data connection dropped");
                handler_info = (struct event_handler_info*)evt->data.ptr;
                ctrl_data_close(handler_info->data);
            }
        }

        /* Second pass to handle all other events */
        for (i = 0, evt = &events[0]; i < nevents; ++i, evt++) {
            if (evt->events & EPOLLERR)
                ALOGD("EPOLLERR on event #%d", i);
            if (evt->events & EPOLLHUP) {
                /* This case was handled in the first pass */
                continue;
            }
            if (evt->data.ptr) {
                handler_info = (struct event_handler_info*)evt->data.ptr;
                handler_info->handler(handler_info->data, evt->events);
            }
        }
    }
}
```

这段代码就很直观了，就是在一个循环里调用 `epoll_wait`，然后调用对应的处理函数。其中比较不一致的代码是当客户挂起（`EPOLLHUP`）的时候，这里我们直接把连接关掉：
```C
static void ctrl_data_close(int dsock_idx) {
    struct epoll_event epev;

    ALOGI("closing lmkd data connection");
    if (epoll_ctl(epollfd, EPOLL_CTL_DEL, data_sock[dsock_idx].sock, &epev) == -1) {
        // Log a warning and keep going
        ALOGW("epoll_ctl for data connection socket failed; errno=%d", errno);
    }
    maxevents--;

    close(data_sock[dsock_idx].sock);
    data_sock[dsock_idx].sock = -1;
}
```

## lmkd 客户端接口

进程 lmkd 的客户主要是 activity manager，它通过 socket `/dev/socket/lmkd` 跟 lmkd 进行通信。通过前面的代码我们已经知道，有客户连接时，调用的是 `ctrl_connect_handler`：
```C
static int get_free_dsock() {
    for (int i = 0; i < MAX_DATA_CONN; i++) {
        if (data_sock[i].sock < 0) {
            return i;
        }
    }
    return -1;
}

static void ctrl_connect_handler(int data __unused, uint32_t events __unused) {
    struct epoll_event epev;
    int free_dscock_idx = get_free_dsock();

    if (free_dscock_idx < 0) {
        /*
         * Number of data connections exceeded max supported. This should not
         * happen but if it does we drop all existing connections and accept
         * the new one. This prevents inactive connections from monopolizing
         * data socket and if we drop ActivityManager connection it will
         * immediately reconnect.
         */
        for (int i = 0; i < MAX_DATA_CONN; i++) {
            ctrl_data_close(i);
        }
        free_dscock_idx = 0;
    }

    data_sock[free_dscock_idx].sock = accept(ctrl_sock.sock, NULL, NULL);
    if (data_sock[free_dscock_idx].sock < 0) {
        ALOGE("lmkd control socket accept failed; errno=%d", errno);
        return;
    }

    ALOGI("lmkd data connection established");
    /* use data to store data connection idx */
    data_sock[free_dscock_idx].handler_info.data = free_dscock_idx;
    data_sock[free_dscock_idx].handler_info.handler = ctrl_data_handler;
    epev.events = EPOLLIN;
    epev.data.ptr = (void *)&(data_sock[free_dscock_idx].handler_info);
    if (epoll_ctl(epollfd, EPOLL_CTL_ADD, data_sock[free_dscock_idx].sock, &epev) == -1) {
        ALOGE("epoll_ctl for data connection socket failed; errno=%d", errno);
        ctrl_data_close(free_dscock_idx);
        return;
    }
    maxevents++;
}
```
这段代码很简单，就是在 `ctrl_sock` 上调用 `accept` 接受一个客户的连接，然后放到 `data_sock`（回想一下，`data_sock` 存放的是 lmkd 的客户，最多可以有两个）。

对客户连接来说，它的处理函数是 `ctrl_data_handler`，`handler_info.data` 对应 `data_sock` 数组的下标，这样在 `ctrl_data_handler` 执行时才知道是哪个客户端可读了。

这就是 `ctrl_sock` 所有的职责了——接受客户的连接。下面我们看看客户 socket 可读的时候会发生什么：
```C
static void ctrl_data_handler(int data, uint32_t events) {
    if (events & EPOLLIN) {
        ctrl_command_handler(data);
    }
}
```
客户端连接后，将通过 socket 给 lmkd 发送命令，这一部分是由 `ctrl_command_handler` 处理的。lmkd 接受三种命令：
```C
/*
 * Supported LMKD commands
 */
enum lmk_cmd {
    LMK_TARGET = 0,  /* Associate minfree with oom_adj_score */
    LMK_PROCPRIO,    /* Register a process and set its oom_adj_score */
    LMK_PROCREMOVE,  /* Unregister a process */
};
```
命令的格式分别为：
```
LMK_TARGET:
+---------+----------+----------------+----------+----------------+----
| lmk_cmd | minfree1 | oom_adj_score1 | minfree2 | oom_adj_score2 | ...
+---------+----------+----------------+----------+----------------+----

LMK_PROCPRIO:
+---------+-----+-----+---------+
| lmk_cmd | pid | uid | oom_adj |
+---------+-----+-----+---------+

LMK_PROCREMOVE:
+---------+-----+
| lmk_cmd | pid |
+---------+-----+
```
命令的每个字段都是 4 个字节（`int` 类型），并且使用**网络字节序**（我也表示很懵逼，命名是在本地通信，为什么不直接使用主机字节序）。

`LMK_TARGET` 是最长的一条协议，他最多可以接受 6 组参数（由 `MAX_TARGETS` 定义），所以一条控制命令的最大长度是 1 + 6 * 2 = 13 个 `int`，这个大小由 `CTRL_PACKET_MAX_SIZE` 定义：
```C
/*
 * Max number of targets in LMK_TARGET command.
 */
#define MAX_TARGETS 6

/*
 * Max packet length in bytes.
 * Longest packet is LMK_TARGET followed by MAX_TARGETS
 * of minfree and oom_adj_score values
 */
#define CTRL_PACKET_MAX_SIZE (sizeof(int) * (MAX_TARGETS * 2 + 1))

/* LMKD packet - first int is lmk_cmd followed by payload */
typedef int LMKD_CTRL_PACKET[CTRL_PACKET_MAX_SIZE / sizeof(int)];
```
了解了基本的数据类型和协议格式后，下面我们来看 `ctrl_command_hander` 的代码：
```C
static void ctrl_command_handler(int dsock_idx) {
    LMKD_CTRL_PACKET packet;
    int len;
    enum lmk_cmd cmd;
    int nargs;
    int targets;

    len = ctrl_data_read(dsock_idx, (char *)packet, CTRL_PACKET_MAX_SIZE);
    if (len <= 0)
        return;

    if (len < (int)sizeof(int)) {
        ALOGE("Wrong control socket read length len=%d", len);
        return;
    }

    cmd = lmkd_pack_get_cmd(packet);
    nargs = len / sizeof(int) - 1;
    if (nargs < 0)
        goto wronglen;

    switch(cmd) {
    case LMK_TARGET:
        targets = nargs / 2;
        if (nargs & 0x1 || targets > (int)ARRAY_SIZE(lowmem_adj))
            goto wronglen;
        cmd_target(targets, packet);
        break;
    case LMK_PROCPRIO:
        if (nargs != 3)
            goto wronglen;
        cmd_procprio(packet);
        break;
    case LMK_PROCREMOVE:
        if (nargs != 1)
            goto wronglen;
        cmd_procremove(packet);
        break;
    default:
        ALOGE("Received unknown command code %d", cmd);
        return;
    }

    return;

wronglen:
    ALOGE("Wrong control socket read length cmd=%d len=%d", cmd, len);
}

static int ctrl_data_read(int dsock_idx, char *buf, size_t bufsz) {
    int ret = 0;

    ret = TEMP_FAILURE_RETRY(read(data_sock[dsock_idx].sock, buf, bufsz));

    if (ret == -1) {
        ALOGE("control data socket read failed; errno=%d", errno);
    } else if (ret == 0) {
        ALOGE("Got EOF on control data socket");
        ret = -1;
    }

    return ret;
}
```

回忆一下，socket lmkd 的类型是 seqpacket，所以每次 `read` 都会恰好返回一条命令。调用 `ctrl_data_read` 读到一整条命名后，我们使用 `lmkd_pack_get_cmd` 函数得到这条命令的 `lmkd_cmd` 字段：
```C
/* Get LMKD packet command */
inline enum lmk_cmd lmkd_pack_get_cmd(LMKD_CTRL_PACKET pack) {
    return (enum lmk_cmd)ntohl(pack[0]);
}
```
前面我们说，协议数据使用网络字节序，所以这里用 `ntohl` 转换为主机字节序。三个命令的 cmd 字段都在第一个 int，所以这里直接返回 `pack[0]`。

接下来三个命令分别由 `cmd_target`，`cmd_procprio`，`cmd_procremove`处理。我们先看 `cmd_target`：
```C
static int lowmem_adj[MAX_TARGETS];
static int lowmem_minfree[MAX_TARGETS];
static int lowmem_targets_size;

/* LMK_TARGET packet payload */
struct lmk_target {
    int minfree;
    int oom_adj_score;
};

/*
 * For LMK_TARGET packet get target_idx-th payload.
 * Warning: no checks performed, caller should ensure valid parameters.
 */
inline void lmkd_pack_get_target(LMKD_CTRL_PACKET packet,
                                 int target_idx, struct lmk_target *target) {
    target->minfree = ntohl(packet[target_idx * 2 + 1]);
    target->oom_adj_score = ntohl(packet[target_idx * 2 + 2]);
}

static void cmd_target(int ntargets, LMKD_CTRL_PACKET packet) {
    int i;
    struct lmk_target target;

    if (ntargets > (int)ARRAY_SIZE(lowmem_adj))
        return;

    for (i = 0; i < ntargets; i++) {
        lmkd_pack_get_target(packet, i, &target);
        lowmem_minfree[i] = target.minfree;
        lowmem_adj[i] = target.oom_adj_score;
    }

    lowmem_targets_size = ntargets;

    if (has_inkernel_module) {
        char minfreestr[128];
        char killpriostr[128];

        minfreestr[0] = '\0';
        killpriostr[0] = '\0';

        for (i = 0; i < lowmem_targets_size; i++) {
            char val[40];

            if (i) {
                strlcat(minfreestr, ",", sizeof(minfreestr));
                strlcat(killpriostr, ",", sizeof(killpriostr));
            }

            snprintf(val, sizeof(val), "%d", use_inkernel_interface ? lowmem_minfree[i] : 0);
            strlcat(minfreestr, val, sizeof(minfreestr));
            snprintf(val, sizeof(val), "%d", use_inkernel_interface ? lowmem_adj[i] : 0);
            strlcat(killpriostr, val, sizeof(killpriostr));
        }

        writefilestring(INKERNEL_MINFREE_PATH, minfreestr);
        writefilestring(INKERNEL_ADJ_PATH, killpriostr);
    }
}
```
`cmd_target` 设置 `lowmem_minfree`， `lowmem_adj`，这两组参数用于控制 low memory 时候的行为。如果使用的是驱动 lmk，那就把参数写到 `INKERNEL_MINFREE_PATH` 和 `INKERNEL_ADJ_PATH`。

接下来是 `cmd_procprio`：
```C
/* LMK_PROCPRIO packet payload */
struct lmk_procprio {
    pid_t pid;
    uid_t uid;
    int oomadj;
};

/*
 * For LMK_PROCPRIO packet get its payload.
 * Warning: no checks performed, caller should ensure valid parameters.
 */
inline void lmkd_pack_get_procprio(LMKD_CTRL_PACKET packet,
                                   struct lmk_procprio *params) {
    params->pid = (pid_t)ntohl(packet[1]);
    params->uid = (uid_t)ntohl(packet[2]);
    params->oomadj = ntohl(packet[3]);
}

static void cmd_procprio(LMKD_CTRL_PACKET packet) {
    struct proc *procp;
    char path[80];
    char val[20];
    int soft_limit_mult;
    struct lmk_procprio params;

    lmkd_pack_get_procprio(packet, &params);

    if (params.oomadj < OOM_SCORE_ADJ_MIN ||
        params.oomadj > OOM_SCORE_ADJ_MAX) {
        ALOGE("Invalid PROCPRIO oomadj argument %d", params.oomadj);
        return;
    }

    snprintf(path, sizeof(path), "/proc/%d/oom_score_adj", params.pid);
    snprintf(val, sizeof(val), "%d", params.oomadj);
    writefilestring(path, val);

    if (use_inkernel_interface)
        return;

    if (low_ram_device) {
        if (params.oomadj >= 900) {
            soft_limit_mult = 0;
        } else if (params.oomadj >= 800) {
            soft_limit_mult = 0;
        } else if (params.oomadj >= 700) {
            soft_limit_mult = 0;
        } else if (params.oomadj >= 600) {
            // Launcher should be perceptible, don't kill it.
            params.oomadj = 200;
            soft_limit_mult = 1;
        } else if (params.oomadj >= 500) {
            soft_limit_mult = 0;
        } else if (params.oomadj >= 400) {
            soft_limit_mult = 0;
        } else if (params.oomadj >= 300) {
            soft_limit_mult = 1;
        } else if (params.oomadj >= 200) {
            soft_limit_mult = 2;
        } else if (params.oomadj >= 100) {
            soft_limit_mult = 10;
        } else if (params.oomadj >=   0) {
            soft_limit_mult = 20;
        } else {
            // Persistent processes will have a large
            // soft limit 512MB.
            soft_limit_mult = 64;
        }

        snprintf(path, sizeof(path),
             "/dev/memcg/apps/uid_%d/pid_%d/memory.soft_limit_in_bytes",
             params.uid, params.pid);
        snprintf(val, sizeof(val), "%d", soft_limit_mult * EIGHT_MEGA);
        writefilestring(path, val);
    }

    procp = pid_lookup(params.pid);
    if (!procp) {
            procp = malloc(sizeof(struct proc));
            if (!procp) {
                // Oh, the irony.  May need to rebuild our state.
                return;
            }

            procp->pid = params.pid;
            procp->uid = params.uid;
            procp->oomadj = params.oomadj;
            proc_insert(procp);
    } else {
        proc_unslot(procp);
        procp->oomadj = params.oomadj;
        proc_slot(procp);
    }
}
```
`cmd_procprio` 用于设置进程的 oomadj，在把 oomadj 写入 `"/proc/#pid/oom_score_adj"` 后，如果使用的是驱动 lmk，就可以直接返回了（驱动 lmk 会处理剩下的工作）。如果不是，接着往下执行。

如果机子是 `low_ram_device`（小内存机器），那么 lmkd 会根据应用的 oomadj 调整应用可用的内存大小。（震惊，原来还有这种骚操作。那在小内存机器里，应用处于后台时就更容易 OOM 了）

最后的一小段代码将这个应用的 oomadj 保存在一个哈希表里：
```C
struct proc {
    struct adjslot_list asl;
    int pid;
    uid_t uid;
    int oomadj;
    struct proc *pidhash_next;
};

#define PIDHASH_SZ 1024
static struct proc *pidhash[PIDHASH_SZ];
#define pid_hashfn(x) ((((x) >> 8) ^ (x)) & (PIDHASH_SZ - 1))

static struct proc *pid_lookup(int pid) {
    struct proc *procp;

    for (procp = pidhash[pid_hashfn(pid)]; procp && procp->pid != pid;
         procp = procp->pidhash_next)
            ;

    return procp;
}

static void proc_insert(struct proc *procp) {
    int hval = pid_hashfn(procp->pid);

    procp->pidhash_next = pidhash[hval];
    pidhash[hval] = procp;
    proc_slot(procp);
}
```
哈希表 `pidhash` 是以 pid 做 key，`proc_slot` 则是把 `struct proc` 插入到以 `oomadj` 为 key 的哈希表 `procadjslot_list` 里面：
```C
/* OOM score values used by both kernel and framework */
#define OOM_SCORE_ADJ_MIN       (-1000)
#define OOM_SCORE_ADJ_MAX       1000

#define ADJTOSLOT(adj) ((adj) + -OOM_SCORE_ADJ_MIN)
static struct adjslot_list procadjslot_list[ADJTOSLOT(OOM_SCORE_ADJ_MAX) + 1];

static void proc_slot(struct proc *procp) {
    int adjslot = ADJTOSLOT(procp->oomadj);

    adjslot_insert(&procadjslot_list[adjslot], &procp->asl);
}

static void adjslot_insert(struct adjslot_list *head, struct adjslot_list *new) {
    struct adjslot_list *next = head->next;
    new->prev = head;
    new->next = next;
    next->prev = new;
    head->next = new;
}
```
跟 pid 值不同，oomadj 是数值范围是 `[-1000, 1000]`，最多只有 2001 种，所以其实一个 oomadj 值就对应  `procadjslot_list` 的 slot。

最后是 `cmd_procremove`：
```C
/* LMK_PROCREMOVE packet payload */
struct lmk_procremove {
    pid_t pid;
};

/*
 * For LMK_PROCREMOVE packet get its payload.
 * Warning: no checks performed, caller should ensure valid parameters.
 */
inline void lmkd_pack_get_procremove(LMKD_CTRL_PACKET packet,
                                     struct lmk_procremove *params) {
    params->pid = (pid_t)ntohl(packet[1]);
}

static void cmd_procremove(LMKD_CTRL_PACKET packet) {
    struct lmk_procremove params;

    if (use_inkernel_interface)
        return;

    lmkd_pack_get_procremove(packet, &params);
    pid_remove(params.pid);
}
```
最后的 `pid_remove` 把这个 pid 对应的 `struct proc` 从 `pidhash` 和 `procadjslot_list` 里移除。
```C
static int pid_remove(int pid) {
    int hval = pid_hashfn(pid);
    struct proc *procp;
    struct proc *prevp;

    for (procp = pidhash[hval], prevp = NULL; procp && procp->pid != pid;
         procp = procp->pidhash_next)
            prevp = procp;

    if (!procp)
        return -1;

    if (!prevp)
        pidhash[hval] = procp->pidhash_next;
    else
        prevp->pidhash_next = procp->pidhash_next;

    proc_unslot(procp);
    free(procp);
    return 0;
}
```

## 回收进程

前面和 socket lmkd 相关的内容主要用于设置 lmk 参数和进程 oomadj。当系统的物理内存不足时，将会触发 mp 事件，这个时候 lmkd 就需要通过杀死一些进程来释放内存页了。

前面我们已经知道，mp 事件发生后，执行的是 `mp_event_common`。这个函数比较长，洋洋洒洒两百来行代码，我们一点一点慢慢看：
```C
static void mp_event_common(int data, uint32_t events __unused) {
    int ret;
    unsigned long long evcount;
    int64_t mem_usage, memsw_usage;
    int64_t mem_pressure;
    enum vmpressure_level lvl;
    union meminfo mi;
    union zoneinfo zi;
    static struct timeval last_report_tm;
    static unsigned long skip_count = 0;
    enum vmpressure_level level = (enum vmpressure_level)data;
    long other_free = 0, other_file = 0;
    int min_score_adj;
    int pages_to_free = 0;
    int minfree = 0;
    static struct reread_data mem_usage_file_data = {
        .filename = MEMCG_MEMORY_USAGE,
        .fd = -1,
    };
    static struct reread_data memsw_usage_file_data = {
        .filename = MEMCG_MEMORYSW_USAGE,
        .fd = -1,
    };

    ...
    ...
}
```
前面这一坨是函数用到的变量的定义。在 ANSI C 那个年代，局部变量都要在函数里先声明，但这对代码的可读性其实没有什么帮助（因为定义跟使用他的上下文脱节了）。这里我们先直接忽略它们，后面再在具体的上下文来看。

```C
static void mp_event_common(int data, uint32_t events __unused) {
    ...
    ...

    enum vmpressure_level lvl;
    enum vmpressure_level level = (enum vmpressure_level)data;
    /*
     * Check all event counters from low to critical
     * and upgrade to the highest priority one. By reading
     * eventfd we also reset the event counters.
     */
    for (lvl = VMPRESS_LEVEL_LOW; lvl < VMPRESS_LEVEL_COUNT; lvl++) {
        if (mpevfd[lvl] != -1 &&
            TEMP_FAILURE_RETRY(read(mpevfd[lvl],
                               &evcount, sizeof(evcount))) > 0 &&
            evcount > 0 && lvl > level) {
            level = lvl;
        }
    }

    ...
    ...
}
```

前面 `init_mp_common` 函数里面，我们把 `evfd` 存在了 `mpevfd` 数组里，为的就是这个时候能够通过读取它们的来判断是否有更高级别的 mp 事件。至此，变量 `level` 表示当前发生的最高 level 的 mp 事件。

```C
static void mp_event_common(int data, uint32_t events __unused) {
    ...
    ...

    static struct timeval last_report_tm;
    static unsigned long skip_count = 0;

    if (kill_timeout_ms) {
        struct timeval curr_tm;
        gettimeofday(&curr_tm, NULL);
        if (get_time_diff_ms(&last_report_tm, &curr_tm) < kill_timeout_ms) {
            skip_count++;
            return;
        }
    }

    if (skip_count > 0) {
        ALOGI("%lu memory pressure events were skipped after a kill!",
              skip_count);
        skip_count = 0;
    }

    ...
    ...
}
```
`kill_timeout_ms` 是我们在 `main` 函数里通过读系统属性设置的，表示上一次 kill 后，等多 `kill_timeout_ms` 再杀下一个。`last_report_tm` 在后面我们成功回收进程后会更新他的时间。这里要注意，`skip_count` 和 `last_report_tm` 都是 `static` 变量。

```C
union zoneinfo {
    struct {
        int64_t nr_free_pages;
        int64_t nr_file_pages;
        int64_t nr_shmem;
        int64_t nr_unevictable;
        int64_t workingset_refault;
        int64_t high;
        /* fields below are calculated rather than read from the file */
        int64_t totalreserve_pages;
    } field;
    int64_t arr[ZI_FIELD_COUNT];
};

union meminfo {
    struct {
        int64_t nr_free_pages;
        int64_t cached;
        int64_t swap_cached;
        int64_t buffers;
        int64_t shmem;
        int64_t unevictable;
        int64_t free_swap;
        int64_t dirty;
        /* fields below are calculated rather than read from the file */
        int64_t nr_file_pages;
    } field;
    int64_t arr[MI_FIELD_COUNT];
};

static void mp_event_common(int data, uint32_t events __unused) {
    ...
    ...

    union meminfo mi;
    union zoneinfo zi;
    if (meminfo_parse(&mi) < 0 || zoneinfo_parse(&zi) < 0) {
        ALOGE("Failed to get free memory!");
        return;
    }

    ...
    ...
}
```

`union meminfo mi` 和 `union zoneinfo zi` 表示系统当前的内存使用情况。`meminfo_parse` 和 `zoneinfo_parse` 分别读取 `/proc/meminfo` 和 `/proc/zoneinfo` 并将解析得到的数据填充到 `mi/zi`。（读者可以开个机器，然后 `cat /proc/meminfo` 看看具体的输出。关于他们解释，参考 `man 5 proc`）。

```C
static void mp_event_common(int data, uint32_t events __unused) {
    ...
    ...

    long other_free = 0, other_file = 0;
    int min_score_adj;

    if (use_minfree_levels) {
        int i;

        other_free = mi.field.nr_free_pages - zi.field.totalreserve_pages;
        if (mi.field.nr_file_pages > (mi.field.shmem + mi.field.unevictable + mi.field.swap_cached)) {
            other_file = (mi.field.nr_file_pages - mi.field.shmem -
                          mi.field.unevictable - mi.field.swap_cached);
        } else {
            other_file = 0;
        }

        min_score_adj = OOM_SCORE_ADJ_MAX + 1;
        for (i = 0; i < lowmem_targets_size; i++) {
            minfree = lowmem_minfree[i];
            if (other_free < minfree && other_file < minfree) {
                min_score_adj = lowmem_adj[i];
                break;
            }
        }

        if (min_score_adj == OOM_SCORE_ADJ_MAX + 1) {
            if (debug_process_killing) {
                ALOGI("Ignore %s memory pressure event "
                      "(free memory=%ldkB, cache=%ldkB, limit=%ldkB)",
                      level_name[level], other_free * page_k, other_file * page_k,
                      (long)lowmem_minfree[lowmem_targets_size - 1] * page_k);
            }
            return;
        }

        /* Free up enough pages to push over the highest minfree level */
        pages_to_free = lowmem_minfree[lowmem_targets_size - 1] -
            ((other_free < other_file) ? other_free : other_file);
        goto do_kill;
    }

    ...
    ...
}
```
`use_minfree_levels` 同样是从系统属性读取的配置，表示使用当我们准备杀死应用的时候，使用系统剩余的内存和文件缓存阈值作为判断依据。

`other_free` 表示系统可用的内存页的数目。

`nr_file_pages` 等于 `mi->field.cached`（文件在内存中的缓存）加上 `mi->field.swap_cached`（swap 出去又读进了内存的数据）加上 `mi->field.buffers`（硬盘的一个临时缓存），

`mi.shmem` 表示 tmpfs 使用的内存数，`unevictable` 表示那些不能 swap out 的内存。

最后 `other_file` 基本就等于除 tmpfs 和 unevictable 外的缓存在内存的文件所占用的 page 数。

有了 `other_free` 和 `other_file` 后，我们根据 `lowmem_minfree` 的值来确定 `min_score_adj`。`min_score_adj` 表示可以回收的最低的 oomadj 值（oomadj 越大，优先级越低，越容易被杀死），oomadj 小于 `min_score_adj` 的进程在这次回收过程中不会被杀死。

回想一下前面的 `cmd_target`，`lowmem_minfree` 和 `lowmem_adj` 值就是在他里面设置的。

`goto do_kill` 在函数比较靠后的地方，我们后面再看。


```C
struct {
    int64_t min_nr_free_pages; /* recorded but not used yet */
    int64_t max_nr_free_pages;
} low_pressure_mem = { -1, -1 };

static void mp_event_common(int data, uint32_t events __unused) {
    ...
    ...

    if (level == VMPRESS_LEVEL_LOW) {
        record_low_pressure_levels(&mi);
    }

    ...
    ...
}

void record_low_pressure_levels(union meminfo *mi) {
    if (low_pressure_mem.min_nr_free_pages == -1 ||
        low_pressure_mem.min_nr_free_pages > mi->field.nr_free_pages) {
        if (debug_process_killing) {
            ALOGI("Low pressure min memory update from %" PRId64 " to %" PRId64,
                low_pressure_mem.min_nr_free_pages, mi->field.nr_free_pages);
        }
        low_pressure_mem.min_nr_free_pages = mi->field.nr_free_pages;
    }
    /*
     * Free memory at low vmpressure events occasionally gets spikes,
     * possibly a stale low vmpressure event with memory already
     * freed up (no memory pressure should have been reported).
     * Ignore large jumps in max_nr_free_pages that would mess up our stats.
     */
    if (low_pressure_mem.max_nr_free_pages == -1 ||
        (low_pressure_mem.max_nr_free_pages < mi->field.nr_free_pages &&
         mi->field.nr_free_pages - low_pressure_mem.max_nr_free_pages <
         low_pressure_mem.max_nr_free_pages * 0.1)) {
        if (debug_process_killing) {
            ALOGI("Low pressure max memory update from %" PRId64 " to %" PRId64,
                low_pressure_mem.max_nr_free_pages, mi->field.nr_free_pages);
        }
        low_pressure_mem.max_nr_free_pages = mi->field.nr_free_pages;
    }
}
```
`low_pressure_mem.min_nr_free_pages` 记录的是目前遇到的最低可用内存页数，`low_pressure_mem.max_nr_free_pages` 记录的是目前遇到的最大可用的内存页数。就像代码中注释说的，有时候可用的内存数会突然暴涨，这里过滤掉了这种情况。

```C
#define MEMCG_MEMORY_USAGE "/dev/memcg/memory.usage_in_bytes"
#define MEMCG_MEMORYSW_USAGE "/dev/memcg/memory.memsw.usage_in_bytes"

static void mp_event_common(int data, uint32_t events __unused) {
    ...
    ...

    if (level_oomadj[level] > OOM_SCORE_ADJ_MAX) {
        /* Do not monitor this pressure level */
        return;
    }

    int64_t mem_usage, memsw_usage;
    static struct reread_data mem_usage_file_data = {
        .filename = MEMCG_MEMORY_USAGE,
        .fd = -1,
    };
    static struct reread_data memsw_usage_file_data = {
        .filename = MEMCG_MEMORYSW_USAGE,
        .fd = -1,
    };

    if ((mem_usage = get_memory_usage(&mem_usage_file_data)) < 0) {
        goto do_kill;
    }
    if ((memsw_usage = get_memory_usage(&memsw_usage_file_data)) < 0) {
        goto do_kill;
    }

    ...
    ...
}

static int64_t get_memory_usage(struct reread_data *file_data) {
    int ret;
    int64_t mem_usage;
    char buf[32];

    if (reread_file(file_data, buf, sizeof(buf)) < 0) {
        return -1;
    }

    if (!parse_int64(buf, &mem_usage)) {
        ALOGE("%s parse error", file_data->filename);
        return -1;
    }
    if (mem_usage == 0) {
        ALOGE("No memory!");
        return -1;
    }
    return mem_usage;
}
```
`get_memory_usage` 的实现很简单，就是读取 `reread_data.filename` 的内容并转换为 `int64`。这里还需要注意的是 `mem_usage_file_data` 和 `memsw_usage_file_data` 是静态变量。第一次打开文件后，会把文件描述符缓存在 `reread_data.fd` 里。

`mem_usage` 是所用的内存数，`memsw_usage` 是内存数加上 swap out 的内存数。接下来的代码根据这两个数据来计算内存压力（压力越大，swap 出去的内存就越多）：
```C
static void mp_event_common(int data, uint32_t events __unused) {
    ...
    ...

    // Calculate percent for swappinness.
    int64_t mem_pressure = (mem_usage * 100) / memsw_usage;

    if (enable_pressure_upgrade && level != VMPRESS_LEVEL_CRITICAL) {
        // We are swapping too much.
        if (mem_pressure < upgrade_pressure) {
            level = upgrade_level(level);
            if (debug_process_killing) {
                ALOGI("Event upgraded to %s", level_name[level]);
            }
        }
    }

    // If the pressure is larger than downgrade_pressure lmk will not
    // kill any process, since enough memory is available.
    if (mem_pressure > downgrade_pressure) {
        if (debug_process_killing) {
            ALOGI("Ignore %s memory pressure", level_name[level]);
        }
        return;
    } else if (level == VMPRESS_LEVEL_CRITICAL &&
               mem_pressure > upgrade_pressure) {
        if (debug_process_killing) {
            ALOGI("Downgrade critical memory pressure");
        }
        // Downgrade event, since enough memory available.
        level = downgrade_level(level);
    }

    ...
    ...
}
```
注意这里 `mem_pressure` 计算的是 `内存数 / (内存数 + swap)`，`mem_pressure` 越小，内存压力就越大。

`enable_pressure_upgrade`、`upgrade_pressure` 和 `downgrade_pressure` 的值是我们在 `main` 函数里根据系统属性设置的。

在内存压力比较大并且 `enable_pressure_upgrade` 打开的情况下，我们把内存压力向上提升一个等级（以期释放更多的内存）；在内存压力小于 `downgrade_pressure` 的时候，内存是充足的，没有必要通过杀死应用来回收内存；如果内存压力中等（upgrade_pressure < mem_pressure < downgrade_pressure）但是 level 却是 critical，就给他降一级。

lmkd 在给 mp level 升级的时候需要打开 enable_pressure_upgrade（默认关闭），而降级却总是可行的，说明 lmkd 尽力在不杀死应用的情况下满足系统的内存需求。

到目前为止，我们得到了三组跟内存压力相关的参数：
1. 在 `use_minfree_levels` 的情况下，`min_score_adj` 和 `pages_to_free`
2. 内存压力 `level`

接下来，我们开始真正的进程回收工作：
```C
static void mp_event_common(int data, uint32_t events __unused) {
    ...
    ...

do_kill:
    if (low_ram_device) {
        /* For Go devices kill only one task */
        if (find_and_kill_processes(level, level_oomadj[level], 0) == 0) {
            if (debug_process_killing) {
                ALOGI("Nothing to kill");
            }
        }
    } else {
        ...
        ...
    }
}
```
回收对象时分两大类，小内存设备和“大”内存设备。小内存设备一次就杀一个进程：
```C
static int find_and_kill_processes(enum vmpressure_level level,
                                   int min_score_adj, int pages_to_free) {
    int i;
    int killed_size;
    int pages_freed = 0;

    for (i = OOM_SCORE_ADJ_MAX; i >= min_score_adj; i--) {
        struct proc *procp;

        while (true) {
            procp = kill_heaviest_task ?
                proc_get_heaviest(i) : proc_adj_lru(i);

            if (!procp)
                break;

            killed_size = kill_one_process(procp, min_score_adj, level);
            if (killed_size >= 0) {
                pages_freed += killed_size;
                if (pages_freed >= pages_to_free) {
                    return pages_freed;
                }
            }
        }
    }

    return pages_freed;
}
```
这里我们从 oomadj 最大的应用开始回收，直到回收的内存页数达到 `pages_to_free`。对 `low_ram_device` 来说，`pages_to_free` 为 0，只有一个进程会被回收。

`kill_heaviest_task` 是从系统属性读的，默认为 `false`。打开的情况下，在相关 oomadj 的进程里，我们优先回收使用内存最多的那个：
```C
static struct proc *proc_get_heaviest(int oomadj) {
    struct adjslot_list *head = &procadjslot_list[ADJTOSLOT(oomadj)];
    struct adjslot_list *curr = head->next;
    struct proc *maxprocp = NULL;
    int maxsize = 0;
    while (curr != head) {
        int pid = ((struct proc *)curr)->pid;
        int tasksize = proc_get_size(pid);
        if (tasksize <= 0) {
            struct adjslot_list *next = curr->next;
            pid_remove(pid);
            curr = next;
        } else {
            if (tasksize > maxsize) {
                maxsize = tasksize;
                maxprocp = (struct proc *)curr;
            }
            curr = curr->next;
        }
    }
    return maxprocp;
}
```
如果是 `false`，调用的则是 `proc_adj_lru`：
```C
static struct adjslot_list *adjslot_tail(struct adjslot_list *head) {
    struct adjslot_list *asl = head->prev;

    return asl == head ? NULL : asl;
}

static struct proc *proc_adj_lru(int oomadj) {
    return (struct proc *)adjslot_tail(&procadjslot_list[ADJTOSLOT(oomadj)]);
}
```
这里我们取的是列表的尾端；而插入新元素时，我们总是把它放在头端。

`kill_one_process` 通过向应用发送信号 `SIGKILL` 来杀死对方：
```
/* Kill one process specified by procp.  Returns the size of the process killed */
static int kill_one_process(struct proc* procp, int min_score_adj,
                            enum vmpressure_level level) {
    int pid = procp->pid;
    uid_t uid = procp->uid;
    char *taskname;
    int tasksize;
    int r;

    taskname = proc_get_name(pid);
    if (!taskname) {
        pid_remove(pid);
        return -1;
    }

    tasksize = proc_get_size(pid);
    if (tasksize <= 0) {
        pid_remove(pid);
        return -1;
    }

    r = kill(pid, SIGKILL);
    ALOGI(
        "Killing '%s' (%d), uid %d, adj %d\n"
        "   to free %ldkB because system is under %s memory pressure oom_adj %d\n",
        taskname, pid, uid, procp->oomadj, tasksize * page_k,
        level_name[level], min_score_adj);
    pid_remove(pid);

    if (r) {
        ALOGE("kill(%d): errno=%d", pid, errno);
        return -1;
    } else {
        return tasksize;
    }

    return tasksize;
}
```

接下来我们看不是 `low_ram_device` 的情况：
```C
static void mp_event_common(int data, uint32_t events __unused) {
    ...
    ...

do_kill:
    if (low_ram_device) {
        ...
        ...
    } else {
        if (!use_minfree_levels) {
            /* If pressure level is less than critical and enough free swap then ignore */
            if (level < VMPRESS_LEVEL_CRITICAL &&
                mi.field.free_swap > low_pressure_mem.max_nr_free_pages) {
                if (debug_process_killing) {
                    ALOGI("Ignoring pressure since %" PRId64
                          " swap pages are available ",
                          mi.field.free_swap);
                }
                return;
            }
            /* Free up enough memory to downgrate the memory pressure to low level */
            if (mi.field.nr_free_pages < low_pressure_mem.max_nr_free_pages) {
                pages_to_free = low_pressure_mem.max_nr_free_pages -
                    mi.field.nr_free_pages;
            } else {
                if (debug_process_killing) {
                    ALOGI("Ignoring pressure since more memory is "
                        "available (%" PRId64 ") than watermark (%" PRId64 ")",
                        mi.field.nr_free_pages, low_pressure_mem.max_nr_free_pages);
                }
                return;
            }
            min_score_adj = level_oomadj[level];
        }

        ...
        ...
    }
}
```
前面我们总结过，在 `!use_minfree_levels` 的情况下，我们只有一个 mp `level`，还需要 `min_score_adj` 和 `pages_to_free` 才能开始回收进程。

`low_pressure_mem.max_nr_free_pages` 是前面我们在 `record_low_pressure_levels` 中记录的，`free_swap` 是系统 swap 分区空余的大小；如果内存压力不是 critical 并且 swap 分区还足够大，就不回收进程了（lmkd 也是不容易啊，只有在实在没有办法了才杀我们应用）。

此外，在空余内存页比我们遇到过的发生 mp 事件时系统剩余内存最多的那次还要多的时候（比以往最好的情况还要好），也不回收应用。即便真的需要回收内存，我们也只回收到系统（内存）状态跟以往最好的那次为止（`pages_to_free = max_nr_free_pages - nr_free_pages`）。

这里计算出 `pages_to_free` 和 `min_scrore_adj` 后，我们下面就该回收进程了：
```C
static void mp_event_common(int data, uint32_t events __unused) {
    ...
    ...

do_kill:
    if (low_ram_device) {
        ...
        ...
    } else {
        if (!use_minfree_levels) {
            // compute pages_to_free & min_score_adj
        }

        pages_freed = find_and_kill_processes(level, min_score_adj, pages_to_free);

        if (use_minfree_levels) {
            ALOGI("Killing because cache %ldkB is below "
                  "limit %ldkB for oom_adj %d\n"
                  "   Free memory is %ldkB %s reserved",
                  other_file * page_k, minfree * page_k, min_score_adj,
                  other_free * page_k, other_free >= 0 ? "above" : "below");
        }

        if (pages_freed < pages_to_free) {
            ALOGI("Unable to free enough memory (pages to free=%d, pages freed=%d)",
                  pages_to_free, pages_freed);
        } else {
            ALOGI("Reclaimed enough memory (pages to free=%d, pages freed=%d)",
                  pages_to_free, pages_freed);
            gettimeofday(&last_report_tm, NULL);
        }
    }
}
```

`find_and_kill_processes` 的实现在前面我们已经看了，他根据进程的 oomajd 值从大到小回收那些 oomadj 值比 `min_score_adj` 大的应用，并且只回收 `pages_to_free` 个内存页就停止。

最后把当前时间记录在 `last_report_tm`，他表示上次成功回收进程的时间（和 `kill_timeout_ms` 组合使用）。

到这里，我们就算是把 lmkd 的实现整个都读完了（长舒一口气）。

