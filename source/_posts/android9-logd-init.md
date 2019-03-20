---
title: Android P 源码分析 4 - logd 的初始化
date: 2019-03-20 19:45:44
categories: Android
tags: [Android source]
---

为了跟老罗的书保持一个比较一致的步伐，这一篇开始我们来看 logd 的实现。当然，这个 logd 不是老罗书里讲的 log 驱动，而是在应用层实现的一个守护进程。

<!-- more -->

在进入正题之前先说明一下，logd 虽然是用 C++ 写的，但由于比较接近系统，需要读者对系统编程有一定的了解。不熟悉的读者可以通过《Linux系统编程》快速入个门，《UNIX环境高级程序设计》则是关于这一主题最好的书籍。

## logd 的启动

通过查看 logd 源码目录，我们可以看到这样一个文件：
```
// system/core/logd/logd.rc
service logd /system/bin/logd
    socket logd stream 0666 logd logd
    socket logdr seqpacket 0666 logd logd
    socket logdw dgram+passcred 0222 logd logd
    file /proc/kmsg r
    file /dev/kmsg w
    user logd
    group logd system package_info readproc
    writepid /dev/cpuset/system-background/tasks

service logd-reinit /system/bin/logd --reinit
    oneshot
    disabled
    user logd
    group logd
    writepid /dev/cpuset/system-background/tasks

on fs
    write /dev/event-log-tags "# content owned by logd
"
    chown logd logd /dev/event-log-tags
    chmod 0644 /dev/event-log-tags
```

init 进程是在 post-fs 阶段启动 logd 的
```
// system/core/rootdir/init.rc
on post-fs
    # Load properties from
    #     /system/build.prop,
    #     /odm/build.prop,
    #     /vendor/build.prop and
    #     /factory/factory.prop
    load_system_props
    # start essential services
    start logd
    start servicemanager
    start hwservicemanager
    start vndservicemanage
```

从这里我们可以得出几个信息：
1. logd 是经由 init 进程启动的
2. init 进程为 logd 创建了 3 个（UNIX 域）socket，分别是 `/dev/socket/logd, /dev/socket/logdr, /dev/socket/logdw`
3. init 进程为 logd 打开了两个文件 `/proc/kmsg, /dev/kmsg`
4. 把 logd 的 uid 设置为 logd，gid 设置为 logd、system、package_info 和 readproc
5. 把 logd 进程的 pid 写到文件 /dev/cpuset/system-background/tasks

关于 socket 的相关知识，读者可以参考《UNIX 网络编程，卷1》。

logd-reinit 用来触发 logd 的重新初始化，同样执行的是 logd 程序，只是多了一个参数 `--init`。后面我们讲 logd 的控制命令时再详细说。

至于 init 进程如何解析 init.rc，以后有机会写 init 进程相关文章的时候再讨论。

## logd 的初始化

init 进程启动 logd 后，接下来执行的自然是 logd 的 `main` 函数。这个函数有点长，这里先把代码放上来，后面再一点点慢慢看。
```C
// system/core/logd/main.cpp

// Foreground waits for exit of the main persistent threads
// that are started here. The threads are created to manage
// UNIX domain client sockets for writing, reading and
// controlling the user space logger, and for any additional
// logging plugins like auditd and restart control. Additional
// transitory per-client threads are created for each reader.
int main(int argc, char* argv[]) {
    // logd is written under the assumption that the timezone is UTC.
    // If TZ is not set, persist.sys.timezone is looked up in some time utility
    // libc functions, including mktime. It confuses the logd time handling,
    // so here explicitly set TZ to UTC, which overrides the property.
    setenv("TZ", "UTC", 1);
    // issue reinit command. KISS argument parsing.
    if ((argc > 1) && argv[1] && !strcmp(argv[1], "--reinit")) {
        return issueReinit();
    }

    static const char dev_kmsg[] = "/dev/kmsg";
    fdDmesg = android_get_control_file(dev_kmsg);
    if (fdDmesg < 0) {
        fdDmesg = TEMP_FAILURE_RETRY(open(dev_kmsg, O_WRONLY | O_CLOEXEC));
    }

    int fdPmesg = -1;
    bool klogd = __android_logger_property_get_bool(
        "ro.logd.kernel",
        BOOL_DEFAULT_TRUE | BOOL_DEFAULT_FLAG_ENG | BOOL_DEFAULT_FLAG_SVELTE);
    if (klogd) {
        static const char proc_kmsg[] = "/proc/kmsg";
        fdPmesg = android_get_control_file(proc_kmsg);
        if (fdPmesg < 0) {
            fdPmesg = TEMP_FAILURE_RETRY(
                open(proc_kmsg, O_RDONLY | O_NDELAY | O_CLOEXEC));
        }
        if (fdPmesg < 0) android::prdebug("Failed to open %s\n", proc_kmsg);
    }

    // Reinit Thread
    sem_init(&reinit, 0, 0);
    sem_init(&uidName, 0, 0);
    sem_init(&sem_name, 0, 1);
    pthread_attr_t attr;
    if (!pthread_attr_init(&attr)) {
        struct sched_param param;

        memset(&param, 0, sizeof(param));
        pthread_attr_setschedparam(&attr, &param);
        pthread_attr_setschedpolicy(&attr, SCHED_BATCH);
        if (!pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED)) {
            pthread_t thread;
            reinit_running = true;
            if (pthread_create(&thread, &attr, reinit_thread_start, nullptr)) {
                reinit_running = false;
            }
        }
        pthread_attr_destroy(&attr);
    }

    bool auditd =
        __android_logger_property_get_bool("ro.logd.auditd", BOOL_DEFAULT_TRUE);
    if (drop_privs(klogd, auditd) != 0) {
        return -1;
    }

    // Serves the purpose of managing the last logs times read on a
    // socket connection, and as a reader lock on a range of log
    // entries.

    LastLogTimes* times = new LastLogTimes();

    // LogBuffer is the object which is responsible for holding all
    // log entries.

    logBuf = new LogBuffer(times);

    signal(SIGHUP, reinit_signal_handler);

    if (__android_logger_property_get_bool(
            "logd.statistics", BOOL_DEFAULT_TRUE | BOOL_DEFAULT_FLAG_PERSIST |
                                   BOOL_DEFAULT_FLAG_ENG |
                                   BOOL_DEFAULT_FLAG_SVELTE)) {
        logBuf->enableStatistics();
    }

    // LogReader listens on /dev/socket/logdr. When a client
    // connects, log entries in the LogBuffer are written to the client.

    LogReader* reader = new LogReader(logBuf);
    if (reader->startListener()) {
        exit(1);
    }

    // LogListener listens on /dev/socket/logdw for client
    // initiated log messages. New log entries are added to LogBuffer
    // and LogReader is notified to send updates to connected clients.

    LogListener* swl = new LogListener(logBuf, reader);
    // Backlog and /proc/sys/net/unix/max_dgram_qlen set to large value
    if (swl->startListener(600)) {
        exit(1);
    }

    // Command listener listens on /dev/socket/logd for incoming logd
    // administrative commands.

    CommandListener* cl = new CommandListener(logBuf, reader, swl);
    if (cl->startListener()) {
        exit(1);
    }

    // LogAudit listens on NETLINK_AUDIT socket for selinux
    // initiated log messages. New log entries are added to LogBuffer
    // and LogReader is notified to send updates to connected clients.

    LogAudit* al = nullptr;
    if (auditd) {
        al = new LogAudit(logBuf, reader,
                          __android_logger_property_get_bool(
                              "ro.logd.auditd.dmesg", BOOL_DEFAULT_TRUE)
                              ? fdDmesg
                              : -1);
    }

    LogKlog* kl = nullptr;
    if (klogd) {
        kl = new LogKlog(logBuf, reader, fdDmesg, fdPmesg, al != nullptr);
    }

    readDmesg(al, kl);

    // failure is an option ... messages are in dmesg (required by standard)

    if (kl && kl->startListener()) {
        delete kl;
    }

    if (al && al->startListener()) {
        delete al;
    }

    TEMP_FAILURE_RETRY(pause());

    exit(0);
}
```

### 打开 /dev/kmsg

前面我们看 init.rc 的时候已经知道，init 进程会为我们打开设备文件 `/dev/kmsg`，所以这里我们只要找到他对应的文件描述符就可以了。
```C
static int fdDmesg = -1;

int main(int argc, char* argv[]) {
    // ...

    static const char dev_kmsg[] = "/dev/kmsg";
    fdDmesg = android_get_control_file(dev_kmsg);
    if (fdDmesg < 0) {
        fdDmesg = TEMP_FAILURE_RETRY(open(dev_kmsg, O_WRONLY | O_CLOEXEC));
    }

    // ...
}
```
子进程想要使用父进程为其打开的文件，一般情况下有这么几种方法：
1. 约定好对应的描述符是多少（比方说，使用 shell 对输入输出进行重定向，就是在 0 1 2 上打开文件）
2. 通过命令行参数告诉子进程（如，`--kmsg 1`）
3. 通过环境变量。这个是 init 进程采用的方法

下面我们就来看看 `android_get_control_file` 是如何实现的：
```C
// system/core/libcutils/android_get_control_file.h
#define ANDROID_FILE_ENV_PREFIX "ANDROID_FILE_"

// system/core/libcutils/android_get_control_file.cpp
int android_get_control_file(const char* path) {
    int fd = __android_get_control_from_env(ANDROID_FILE_ENV_PREFIX, path);

#if defined(__linux__)
    // Find file path from /proc and make sure it is correct
    char *proc = NULL;
    if (asprintf(&proc, "/proc/self/fd/%d", fd) < 0) return -1;
    if (!proc) return -1;

    size_t len = strlen(path);
    // readlink() does not guarantee a nul byte, len+2 so we catch truncation.
    char *buf = static_cast<char *>(calloc(1, len + 2));
    if (!buf) {
        free(proc);
        return -1;
    }
    ssize_t ret = TEMP_FAILURE_RETRY(readlink(proc, buf, len + 1));
    free(proc);
    int cmp = (len != static_cast<size_t>(ret)) || strcmp(buf, path);
    free(buf);
    if (ret < 0) return -1;
    if (cmp != 0) return -1;
    // It is what we think it is
#endif

    return fd;
}

// bionic/libc/include/unistd.h
/* Used to retry syscalls that can return EINTR. */
#define TEMP_FAILURE_RETRY(exp) ({         \
    __typeof__(exp) _rc;                   \
    do {                                   \
        _rc = (exp);                       \
    } while (_rc == -1 && errno == EINTR); \
    _rc; })
```

`__android_get_control_from_env` 拿到这个 `fd` 后，如果运行的系统是 Linux，就执行后面的一些检查。具体来说就是读符号链接 `/proc/self/fd/#fd_num` 的内容，如果这个内容跟 `path` 相等，就认为这个描述符确实是我们所需要的。

`TEMP_FAILURE_RETRY` 在系统的源码里出现的频率很高，主要用来处理系统调动被信号中断的情况。`__typeof__` 是编译器提供的运算符，类似于 C++ 的 `decltype`。

下面我们看看 `__android_get_control_from_env`：
```C++
// system/core/libcutils/android_get_control_file.cpp
LIBCUTILS_HIDDEN int __android_get_control_from_env(const char* prefix,
                                                    const char* name) {
    if (!prefix || !name) return -1;

    char *key = NULL;
    if (asprintf(&key, "%s%s", prefix, name) < 0) return -1;
    if (!key) return -1;

    char *cp = key;
    while (*cp) {
        if (!isalnum(*cp)) *cp = '_';
        ++cp;
    }

    const char* val = getenv(key);
    free(key);
    if (!val) return -1;

    errno = 0;
    long fd = strtol(val, NULL, 10);
    if (errno) return -1;

    // validity checking
    if ((fd < 0) || (fd > INT_MAX)) return -1;

    // Since we are inheriting an fd, it could legitimately exceed _SC_OPEN_MAX

    // Still open?
#if defined(F_GETFD) // Lowest overhead
    if (TEMP_FAILURE_RETRY(fcntl(fd, F_GETFD)) < 0) return -1;
#elif defined(F_GETFL) // Alternate lowest overhead
    if (TEMP_FAILURE_RETRY(fcntl(fd, F_GETFL)) < 0) return -1;
#else // Hail Mary pass
    struct stat s;
    if (TEMP_FAILURE_RETRY(fstat(fd, &s)) < 0) return -1;
#endif

    return static_cast<int>(fd);
}
```
前面我们传入的的 `ANDROID_FILE_` 和 `/dev/kmsg`，这里我们把他们拼接起来得到 `ANDROID_FILE_/dev/kmsg`。随后的循环把不是字母、数字的字符换成 `_`，最后这个 key 是 `ANDROID_FILE__dev_kmsg`。

我们拿这个 key 去 `getenv`，如果存在这个环境变量，就调用 `strtol` 将其转换为 `long`。所谓的文件描述符，其实仅仅是一个数字。这里将 `val` 转换为 `long`，我们也就拿到了文件对应的 fd。

拿到这个 fd 后，还要验证一下它是不是还打开着。这里使用的方法是用 `fcntl` 去获取一下 fd flag。如果成功，文件自然是打开着的。

获取 fd flag 一般只需要访问文件表，所以是最快的；获取 file flag 要通过文件表去拿 file 对象，这个慢一点；而  file stat 则需要再通过 file 对象拿到 inode 节点的数据，这个是最慢的。

我们直接通过环境变量取得描述符，这并不能保证它就是我们所期望的文件（比方说，可以先关掉这个 fd，然后再打开任意一个文件，新打开的文件 fd 的数值将会和我们刚刚关闭的那个一样），所以在 `android_get_control_file` 里还要用 `/proc/self/fd/##` 验证多一次。

`/dev/kmsg` 设备文件是用来读写内核 log 的，有兴趣的读者可以参考文档 [dev-kmsg](https://www.kernel.org/doc/Documentation/ABI/testing/dev-kmsg)。logd 本身提供的就是 log 机制，但在自己还没启动完成或者出错的时候，如果需要写 log，就只能写到内核的 log 去了。

### 打开 /proc/kmsg

```C++
int main(int argc, char* argv[]) {

    // open /dev/kmsg

    int fdPmesg = -1;
    bool klogd = __android_logger_property_get_bool(
        "ro.logd.kernel",
        BOOL_DEFAULT_TRUE | BOOL_DEFAULT_FLAG_ENG | BOOL_DEFAULT_FLAG_SVELTE);
    if (klogd) {
        static const char proc_kmsg[] = "/proc/kmsg";
        fdPmesg = android_get_control_file(proc_kmsg);
        if (fdPmesg < 0) {
            fdPmesg = TEMP_FAILURE_RETRY(
                open(proc_kmsg, O_RDONLY | O_NDELAY | O_CLOEXEC));
        }
        if (fdPmesg < 0) android::prdebug("Failed to open %s\n", proc_kmsg);
    }

    // ...
}
```
打开 `/proc/kmsg` 是为了读内核的日志，但这个是可选的，这里我们通过读系统属性来判断是否需要读内核的日志。

### 启动 reinit 线程
```C++
int main(int argc, char* argv[]) {

    // open /dev/kmsg
    // open /proc/kmsg

    // Reinit Thread
    sem_init(&reinit, 0, 0);
    sem_init(&uidName, 0, 0);
    sem_init(&sem_name, 0, 1);
    pthread_attr_t attr;
    if (!pthread_attr_init(&attr)) {
        struct sched_param param;

        memset(&param, 0, sizeof(param));
        pthread_attr_setschedparam(&attr, &param);
        pthread_attr_setschedpolicy(&attr, SCHED_BATCH);
        if (!pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED)) {
            pthread_t thread;
            reinit_running = true;
            if (pthread_create(&thread, &attr, reinit_thread_start, nullptr)) {
                reinit_running = false;
            }
        }
        pthread_attr_destroy(&attr);
    }

    // ...
}
```

reinit 线程主要处理最开始时前面我们提到了 reinit 命令。另外，logd 还使用这个线程做 uid 转 name 的工作。关于他的实现，后面我们讲 logd 的管理接口时再看。

### 设置运行时优先级、权限
```C++
int main(int argc, char* argv[]) {

    // open /dev/kmsg
    // open /proc/kmsg
    // 启动 Reinit Thread

    bool auditd =
        __android_logger_property_get_bool("ro.logd.auditd", BOOL_DEFAULT_TRUE);
    if (drop_privs(klogd, auditd) != 0) {
        return -1;
    }

    // ...
}
```

这一部分代码跟平台相关性比较大，普通的应用开发一般不会使用到这些。这部分我们这里先略过，后面用单独的一篇文章来讲。

### 启动各个 log 监听器

```C++
int main(int argc, char* argv[]) {

    // open /dev/kmsg
    // open /proc/kmsg
    // 启动 Reinit Thread
    // 设置运行时优先级、权限

    // Serves the purpose of managing the last logs times read on a
    // socket connection, and as a reader lock on a range of log
    // entries.

    LastLogTimes* times = new LastLogTimes();

    // LogBuffer is the object which is responsible for holding all
    // log entries.

    logBuf = new LogBuffer(times);

    signal(SIGHUP, reinit_signal_handler);

    if (__android_logger_property_get_bool(
            "logd.statistics", BOOL_DEFAULT_TRUE | BOOL_DEFAULT_FLAG_PERSIST |
                                   BOOL_DEFAULT_FLAG_ENG |
                                   BOOL_DEFAULT_FLAG_SVELTE)) {
        logBuf->enableStatistics();
    }

    // LogReader listens on /dev/socket/logdr. When a client
    // connects, log entries in the LogBuffer are written to the client.

    LogReader* reader = new LogReader(logBuf);
    if (reader->startListener()) {
        exit(1);
    }

    // LogListener listens on /dev/socket/logdw for client
    // initiated log messages. New log entries are added to LogBuffer
    // and LogReader is notified to send updates to connected clients.

    LogListener* swl = new LogListener(logBuf, reader);
    // Backlog and /proc/sys/net/unix/max_dgram_qlen set to large value
    if (swl->startListener(600)) {
        exit(1);
    }

    // Command listener listens on /dev/socket/logd for incoming logd
    // administrative commands.

    CommandListener* cl = new CommandListener(logBuf, reader, swl);
    if (cl->startListener()) {
        exit(1);
    }

    // LogAudit listens on NETLINK_AUDIT socket for selinux
    // initiated log messages. New log entries are added to LogBuffer
    // and LogReader is notified to send updates to connected clients.

    LogAudit* al = nullptr;
    if (auditd) {
        al = new LogAudit(logBuf, reader,
                          __android_logger_property_get_bool(
                              "ro.logd.auditd.dmesg", BOOL_DEFAULT_TRUE)
                              ? fdDmesg
                              : -1);
    }

    LogKlog* kl = nullptr;
    if (klogd) {
        kl = new LogKlog(logBuf, reader, fdDmesg, fdPmesg, al != nullptr);
    }

    readDmesg(al, kl);

    // failure is an option ... messages are in dmesg (required by standard)

    if (kl && kl->startListener()) {
        delete kl;
    }

    if (al && al->startListener()) {
        delete al;
    }

    TEMP_FAILURE_RETRY(pause());

    exit(0);
}
```

后面的这些代码实现上算是非常直观的，各个类的作用也都通过注释写得很清楚。`LogAudit` 读的是 selinux 的 log，`LogKlog` 读的是内核的 log，`readDmsg` 用 `klogctl` 把内核的 log 读出来以后，又把数据通过 `LogAudit` 和 `LogKlog` 写到由 logd 管理的 `LogBuffer` 里面。这两个我都不太熟悉，后面我们先就直接忽略他了。哪天补上了相关知识点，有机会再来写多两篇。

到目前为止，我们算是了解了 logd 的骨架，后面我们再分 4 篇文章，分别写 Linux 的权限控制、logd 命令控制、读 log 写 log。
