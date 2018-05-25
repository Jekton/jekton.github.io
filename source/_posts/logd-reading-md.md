---
title: Android log 机制 - 读取 logd 中的 log 数据
date: 2018-05-25 13:56:56
categories: Android
tags: [Android source, logd]
---

当客户端想要读取 log 数据的时候，可以使用 socket 连接至 `/dev/socket/logdr`。对应的连接由 `LogReader` 处理。这里的 reader 是从用户的角度来看的。如果站在 logd 的位置，实际上是把 log 数据写入 socket。为了避免混淆，后文统称“写回数据”。


## LogReader 初始化

在 [logd 总览](/2018/05/11/logd-overview)一篇中，我们知道，`LogReader` 是在 `main` 函数里启动的：
```C++
// system/core/logd/main.cpp
int main(int argc, char* argv[]) {
    // ...

    // LogReader listens on /dev/socket/logdr. When a client
    // connects, log entries in the LogBuffer are written to the client.

    LogReader* reader = new LogReader(logBuf);
    if (reader->startListener()) {
        exit(1);
    }

    // ...
}
```
下面我们看看它的构造函数：
```C++
// system/core/logd/LogReader.h
class LogReader : public SocketListener {
    LogBuffer& mLogbuf;
    // ...
};

// system/core/logd/LogReader.cpp
LogReader::LogReader(LogBuffer* logbuf)
    : SocketListener(getLogSocket(), true), mLogbuf(*logbuf) {
}

// system/core/logd/LogReader.cpp
int LogReader::getLogSocket() {
    static const char socketName[] = "logdr";
    int sock = android_get_control_socket(socketName);

    if (sock < 0) {
        sock = socket_local_server(
            socketName, ANDROID_SOCKET_NAMESPACE_RESERVED, SOCK_SEQPACKET);
    }

    return sock;
}
```
和 `LogListener` 一样，`LogReader` 也继承了 `SocketListener`，关于 `SocketListener`，不熟悉的读者，可以看[这篇](/2018/05/16/logd-writing-part1)。

## 读取客户请求参数

当有客户端连接的时候，`SocketListener` 会回调子类的 `onDataAvailable` 函数。在这个函数中，`LogReader` 主要做 3 件事：
1. 设置线程名
2. 读取客户端传过来的参数
3. 生成一个 `FlushCommand` 用于向客户端写回 log 数据。

这里我们先看前两个工作：
```C++
// system/core/logd/LogReader.cpp
bool LogReader::onDataAvailable(SocketClient* cli) {
    static bool name_set;
    if (!name_set) {
        prctl(PR_SET_NAME, "logd.reader");
        name_set = true;
    }

    char buffer[255];

    int len = read(cli->getSocket(), buffer, sizeof(buffer) - 1);
    if (len <= 0) {
        doSocketDelete(cli);
        return false;
    }
    buffer[len] = '\0';

    unsigned long tail = 0;
    static const char _tail[] = " tail=";
    char* cp = strstr(buffer, _tail);
    if (cp) {
        tail = atol(cp + sizeof(_tail) - 1);
    }

    log_time start(log_time::EPOCH);
    static const char _start[] = " start=";
    cp = strstr(buffer, _start);
    if (cp) {
        // Parse errors will result in current time
        start.strptime(cp + sizeof(_start) - 1, "%s.%q");
    }

    uint64_t timeout = 0;
    static const char _timeout[] = " timeout=";
    cp = strstr(buffer, _timeout);
    if (cp) {
        timeout = atol(cp + sizeof(_timeout) - 1) * NS_PER_SEC +
                  log_time(CLOCK_REALTIME).nsec();
    }

    unsigned int logMask = -1;
    static const char _logIds[] = " lids=";
    cp = strstr(buffer, _logIds);
    if (cp) {
        logMask = 0;
        cp += sizeof(_logIds) - 1;
        while (*cp && *cp != '\0') {
            int val = 0;
            while (isdigit(*cp)) {
                val = val * 10 + *cp - '0';
                ++cp;
            }
            logMask |= 1 << val;
            if (*cp != ',') {
                break;
            }
            ++cp;
        }
    }

    pid_t pid = 0;
    static const char _pid[] = " pid=";
    cp = strstr(buffer, _pid);
    if (cp) {
        pid = atol(cp + sizeof(_pid) - 1);
    }

    bool nonBlock = false;
    if (!fastcmp<strncmp>(buffer, "dumpAndClose", 12)) {
        // Allow writer to get some cycles, and wait for pending notifications
        sched_yield();
        LogTimeEntry::lock();
        LogTimeEntry::unlock();
        sched_yield();
        nonBlock = true;
    }

    // ...
}
```
可以看到，通信用的是文本协议，主要设置 `tail, start, timeout, logMask, pid, 和 nonblock`。如果客户没有传递对应的参数，会使用默认值。

其中，`tail` 表示读取 log 的最新 `tail` 条数据；`start` 是 log 的起始时间；`timeout` 比较诡异，表示读取 log 前，先睡 `timeout` 这么一个时长。

在实际读取 log 前，会先执行下面一个优化措施：
```C++
// system/core/logd/LogReader.cpp
bool LogReader::onDataAvailable(SocketClient* cli) {
    // 参数读取部分

    log_time sequence = start;
    //
    // This somewhat expensive data validation operation is required
    // for non-blocking, with timeout.  The incoming timestamp must be
    // in range of the list, if not, return immediately.  This is
    // used to prevent us from from getting stuck in timeout processing
    // with an invalid time.
    //
    // Find if time is really present in the logs, monotonic or real, implicit
    // conversion from monotonic or real as necessary to perform the check.
    // Exit in the check loop ASAP as you find a transition from older to
    // newer, but use the last entry found to ensure overlap.
    //
    if (nonBlock && (sequence != log_time::EPOCH) && timeout) {
        class LogFindStart {  // A lambda by another name
           private:
            const pid_t mPid;
            const unsigned mLogMask;
            bool mStartTimeSet;
            log_time mStart;
            // 注意，mSequence 是一个引用
            log_time& mSequence;
            log_time mLast;
            bool mIsMonotonic;

           public:
            LogFindStart(pid_t pid, unsigned logMask, log_time& sequence,
                         bool isMonotonic)
                : mPid(pid),
                  mLogMask(logMask),
                  mStartTimeSet(false),
                  mStart(sequence),
                  mSequence(sequence),
                  mLast(sequence),
                  mIsMonotonic(isMonotonic) {
            }

            static int callback(const LogBufferElement* element, void* obj) {
                // ojb 就是我们传递给 LogBuffer::flushTo 的第 7 个参数
                LogFindStart* me = reinterpret_cast<LogFindStart*>(obj);
                if ((!me->mPid || (me->mPid == element->getPid())) &&
                    (me->mLogMask & (1 << element->getLogId()))) {
                    log_time real = element->getRealTime();
                    if (me->mStart == real) {
                        me->mSequence = real;
                        me->mStartTimeSet = true;
                        return -1;
                    } else if (!me->mIsMonotonic || android::isMonotonic(real)) {
                        if (me->mStart < real) {
                            me->mSequence = me->mLast;
                            me->mStartTimeSet = true;
                            return -1;
                        }
                        me->mLast = real;
                    } else {
                        me->mLast = real;
                    }
                }
                return false;
            }

            bool found() {
                return mStartTimeSet;
            }

        } logFindStart(pid, logMask, sequence,
                       logbuf().isMonotonic() && android::isMonotonic(start));

        logbuf().flushTo(cli, sequence, nullptr, FlushCommand::hasReadLogs(cli),
                         FlushCommand::hasSecurityLogs(cli),
                         logFindStart.callback, &logFindStart);

        if (!logFindStart.found()) {
            doSocketDelete(cli);
            return false;
        }
    }

    // ...
}
```
当 `nonBlock && (sequence != log_time::EPOCH) && timeout` 成立时，这里会先扫描一遍 log 队列，确保有符合条件的 log 项，如果没有，直接关闭 socket 并返回。

关于 `LogBuffer::flushTo` 函数，我们后面再仔细看它的实现。

他的第 6 个参数是一个函数指针，作为过滤器使用：
1. 如果返回 `true`，则写入对应的 log 项
2. 返回 `false`，跳过 log 项
3. 返回其他值，则结束迭代

接下来，生成一个 `FlushCommand` 实例，开始真正的读取 log 工作。
```C++
// system/core/logd/LogReader.cpp
bool LogReader::onDataAvailable(SocketClient* cli) {
    // ...

    FlushCommand command(*this, nonBlock, tail, logMask, pid, sequence, timeout);

    // Set acceptable upper limit to wait for slow reader processing b/27242723
    struct timeval t = { LOGD_SNDTIMEO, 0 };
    setsockopt(cli->getSocket(), SOL_SOCKET, SO_SNDTIMEO, (const char*)&t,
               sizeof(t));

    command.runSocketCommand(cli);
    return true;
}

```


## 启动读 log 线程

`FlushCommand` 继承了 `SocketClientCommand`，`SocketClientCommand` 是 `SocketListener` 框架的一部分：
```C++
// system/core/logd/FlushCommand.h
class FlushCommand : public SocketClientCommand {
    // ...

public:
    virtual void runSocketCommand(SocketClient* client);
}
```
之所以如此，是为了在写入 log 的时候，利用 `SocketListener` 的“广播”功能。在 [log的写入](/2018/05/17/logd-writing-part2/)一篇中我们说过，当写入 log 数据后，由于此时可能有客户端在等待读取数据，所以需要唤醒他们：
```C++
// system/core/logd/LogReader.cpp

// When we are notified a new log entry is available, inform
// all of our listening sockets.
void LogReader::notifyNewLog() {
    FlushCommand command(*this);
    runOnEachSocket(&command);
}
```
`runOnEachSocket` 会对所有的 socket 执行 `command->runSocketCommand(client)`。

就像本节的标题预示的那样，`FlushCommand` 的实际工作其实是启动读取 log 线程：
```C++
// system/core/logd/FlushCommand.cpp

// runSocketCommand is called once for every open client on the
// log reader socket. Here we manage and associated the reader
// client tracking and log region locks LastLogTimes list of
// LogTimeEntrys, and spawn a transitory per-client thread to
// work at filing data to the  socket.
//
// global LogTimeEntry::lock() is used to protect access,
// reference counts are used to ensure that individual
// LogTimeEntry lifetime is managed when not protected.
void FlushCommand::runSocketCommand(SocketClient* client) {
    LogTimeEntry* entry = NULL;
    // 每个读者都对应 times 列表里的一个元素
    LastLogTimes& times = mReader.logbuf().mTimes;

    LogTimeEntry::lock();
    LastLogTimes::iterator it = times.begin();
    while (it != times.end()) {
        entry = (*it);
        // 遍历列表，找到自己
        if (entry->mClient == client) {
            if (entry->mTimeout.tv_sec || entry->mTimeout.tv_nsec) {
                // 等 timeout 结束后就会醒来，所以直接 return
                if (mReader.logbuf().isMonotonic()) {
                    LogTimeEntry::unlock();
                    return;
                }
                // If the user changes the time in a gross manner that
                // invalidates the timeout, fall through and trigger.
                log_time now(CLOCK_REALTIME);
                // 如果时间被修改，可能会导致 timeout 无效，用当前时间判断 timeout 是否还有效
                // 这里其实是一个bug，mEnd 是创建 entry 时的时间，而后面在使用 mTimeout 的时候，直
                // 接把它传递给了 pthread_cond_timedwait，也就是说， mTimeout 也是一个绝对时间
                if (((entry->mEnd + entry->mTimeout) > now) &&
                    (now > entry->mEnd)) {
                    LogTimeEntry::unlock();
                    return;
                }
            }
            // 唤醒读 log 线程
            entry->triggerReader_Locked();
            if (entry->runningReader_Locked()) {
                // 线程已经在运行，直接返回
                LogTimeEntry::unlock();
                return;
            }
            // 只有在创建是 entry 后，创建线程失败才会执行到这些，后面重新尝试启动线程
            entry->incRef_Locked();
            break;
        }
        it++;
    }

    if (it == times.end()) {
        // Create LogTimeEntry in notifyNewLog() ?
        if (mTail == (unsigned long)-1) {
            LogTimeEntry::unlock();
            return;
        }
        entry = new LogTimeEntry(mReader, client, mNonBlock, mTail, mLogMask,
                                 mPid, mStart, mTimeout);
        times.push_front(entry);
    }

    client->incRef();

    // release client and entry reference counts once done
    entry->startReader_Locked();
    LogTimeEntry::unlock();
}
```

`entry->startReader_Locked()` 会启动读 log 的线程，从 `LogBuffer` 读取 log 后写回客户端：
```C++
// system/core/logd/LogTimes.cpp
void LogTimeEntry::startReader_Locked(void) {
    pthread_attr_t attr;

    threadRunning = true;

    if (!pthread_attr_init(&attr)) {
        if (!pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED)) {
            if (!pthread_create(&mThread, &attr, LogTimeEntry::threadStart,
                                this)) {
                pthread_attr_destroy(&attr);
                return;
            }
        }
        pthread_attr_destroy(&attr);
    }
    threadRunning = false;
    if (mClient) {
        mClient->decRef();
    }
    decRef_Locked();
}
```
这里的逻辑比较简单，读者自己看一看就好。

## log 读取线程

现在，我们来看看上一节所启动的线程到底做了些什么工作：
```C++
// system/core/logd/LogTimes.cpp
void* LogTimeEntry::threadStart(void* obj) {
    prctl(PR_SET_NAME, "logd.reader.per");

    LogTimeEntry* me = reinterpret_cast<LogTimeEntry*>(obj);

    // 定义一个清理函数
    pthread_cleanup_push(threadStop, obj);

    SocketClient* client = me->mClient;
    if (!client) {
        me->error();
        return nullptr;
    }

    LogBuffer& logbuf = me->mReader.logbuf();

    bool privileged = FlushCommand::hasReadLogs(client);
    bool security = FlushCommand::hasSecurityLogs(client);

    me->leadingDropped = true;

    lock();

    log_time start = me->mStart;

    while (me->threadRunning && !me->isError_Locked()) {
        // 如果带 timeout，先睡 timeout 时长，再读 log
        if (me->mTimeout.tv_sec || me->mTimeout.tv_nsec) {
            // LogBuffer::prune 可能会唤醒它，此时返回值不等于 ETIMEDOUT，下个循环还将继续
            // 等待 timeout 时长
            if (pthread_cond_timedwait(&me->threadTriggeredCondition,
                                       &timesLock, &me->mTimeout) == ETIMEDOUT) {
                // 清零后，就不再执行这个操作了
                me->mTimeout.tv_sec = 0;
                me->mTimeout.tv_nsec = 0;
            }
            if (!me->threadRunning || me->isError_Locked()) {
                break;
            }
        }

        unlock();

        // mTail 表示读取最新的 mTail 条 log，为了达到这个目的，需要先遍历一遍
        if (me->mTail) {
            logbuf.flushTo(client, start, nullptr, privileged, security,
                           FilterFirstPass, me);
            me->leadingDropped = true;
        }
        // 实际的读取 log 操作
        start = logbuf.flushTo(client, start, me->mLastTid, privileged,
                               security, FilterSecondPass, me);

        lock();

        // 如果向客户端写回 log 数据失败，将返回 LogBufferElement::FLUSH_ERROR
        if (start == LogBufferElement::FLUSH_ERROR) {
            me->error_Locked();
            break;
        }

        me->mStart = start + log_time(0, 1);

        // mNonBlock 的情况下，读完当前的 log 后，就直接返回
        if (me->mNonBlock || !me->threadRunning || me->isError_Locked()) {
            break;
        }

        // skip 表示要跳过的 log 条数，我们刚读了所有的 log，skip 已经没有意义了
        me->cleanSkip_Locked();

        if (!me->mTimeout.tv_sec && !me->mTimeout.tv_nsec) {
            pthread_cond_wait(&me->threadTriggeredCondition, &timesLock);
        }
    }

    unlock();

    // 执行清理函数 threadStop 
    pthread_cleanup_pop(true);

    return nullptr;
}
```

我们假定 `mTail != 0`，这个时候会执行两次遍历操作。第一次遍历的回调函数如下：
```C++
// system/core/logd/LogTimes.cpp

// A first pass to count the number of elements
int LogTimeEntry::FilterFirstPass(const LogBufferElement* element, void* obj) {
    LogTimeEntry* me = reinterpret_cast<LogTimeEntry*>(obj);

    LogTimeEntry::lock();

    if (me->leadingDropped) {
        if (element->getDropped()) {
            LogTimeEntry::unlock();
            return false;
        }
        me->leadingDropped = false;
    }

    if (me->mCount == 0) {
        me->mStart = element->getRealTime();
    }

    if ((!me->mPid || (me->mPid == element->getPid())) &&
        (me->isWatching(element->getLogId()))) {
        ++me->mCount;
    }

    LogTimeEntry::unlock();

    return false;
}
```
就跟函数的注释说的，这里就是计算所有复合条件的 log 的数目。

下面我们看第二个：
```C++
// system/core/logd/LogTimes.cpp
// A second pass to send the selected elements
int LogTimeEntry::FilterSecondPass(const LogBufferElement* element, void* obj) {
    LogTimeEntry* me = reinterpret_cast<LogTimeEntry*>(obj);

    LogTimeEntry::lock();

    me->mStart = element->getRealTime();

    // skip 是在 LogBuffer::prune 里设置的，快速跳过对应的 log 项后，prune 就能够把他们
    // 释放掉了
    if (me->skipAhead[element->getLogId()]) {
        me->skipAhead[element->getLogId()]--;
        goto skip;
    }

    // leading drop 其实是空的 log 项
    if (me->leadingDropped) {
        if (element->getDropped()) {
            goto skip;
        }
        me->leadingDropped = false;
    }

    // Truncate to close race between first and second pass
    // 总共只有 mCount 条 log，mIndex >= mCount 表示已经没有更多的 log 了
    if (me->mNonBlock && me->mTail && (me->mIndex >= me->mCount)) {
        goto stop;
    }

    if (!me->isWatching(element->getLogId())) {
        goto skip;
    }

    if (me->mPid && (me->mPid != element->getPid())) {
        goto skip;
    }

    if (me->isError_Locked()) {
        goto stop;
    }

    if (!me->mTail) {
        goto ok;
    }

    ++me->mIndex;
    // 我们只需要读取 mTail 条 log，所以忽略前面的 mCount - mTail 条 log
    if ((me->mCount > me->mTail) && (me->mIndex <= (me->mCount - me->mTail))) {
        goto skip;
    }

    if (!me->mNonBlock) {
        me->mTail = 0;
    }

ok:
    if (!me->skipAhead[element->getLogId()]) {
        LogTimeEntry::unlock();
        return true;
    }
// FALLTHRU

skip:
    LogTimeEntry::unlock();
    return false;

stop:
    LogTimeEntry::unlock();
    return -1;
}
```

线程清理函数比较简单，就不看了。


## 读取 log

现在，是时候看看那个神秘的 `LogBuffer::flushTo` 了。

```C++
// system/core/logd/LogBuffer.cpp
log_time LogBuffer::flushTo(SocketClient* reader, const log_time& start,
                            pid_t* lastTid, bool privileged, bool security,
                            int (*filter)(const LogBufferElement* element,
                                          void* arg),
                            void* arg) {
    LogBufferElementCollection::iterator it;
    uid_t uid = reader->getUid();

    pthread_mutex_lock(&mLogElementsLock);

    // 根据 start 参数找到开始迭代的地方
    if (start == log_time::EPOCH) {
        // client wants to start from the beginning
        it = mLogElements.begin();
    } else {
        // 3 second limit to continue search for out-of-order entries.
        log_time min = start - pruneMargin;

        // Cap to 300 iterations we look back for out-of-order entries.
        size_t count = 300;

        // Client wants to start from some specified time. Chances are
        // we are better off starting from the end of the time sorted list.
        LogBufferElementCollection::iterator last;
        for (last = it = mLogElements.end(); it != mLogElements.begin();
             /* do nothing */) {
            --it;
            LogBufferElement* element = *it;
            if (element->getRealTime() > start) {
                last = it;
            } else if (!--count || (element->getRealTime() < min)) {
                break;
            }
        }
        it = last;
    }

    log_time max = start;

    LogBufferElement* lastElement = nullptr;  // iterator corruption paranoia
    static const size_t maxSkip = 4194304;    // maximum entries to skip
    size_t skip = maxSkip;
    for (; it != mLogElements.end(); ++it) {
        LogBufferElement* element = *it;

        if (!--skip) {
            android::prdebug("reader.per: too many elements skipped");
            break;
        }
        if (element == lastElement) {
            android::prdebug("reader.per: identical elements");
            break;
        }
        lastElement = element;

        if (!privileged && (element->getUid() != uid)) {
            continue;
        }

        if (!security && (element->getLogId() == LOG_ID_SECURITY)) {
            continue;
        }

        if (element->getRealTime() <= start) {
            continue;
        }

        // filter 就是我们前面多次传递进来的函数
        // 1. 返回 true 表示写回该 log 项
        // 2. false 表示忽略
        // 3. 其他值则结束迭代
        // NB: calling out to another object with mLogElementsLock held (safe)
        if (filter) {
            int ret = (*filter)(element, arg);
            if (ret == false) {
                continue;
            }
            if (ret != true) {
                break;
            }
        }

        bool sameTid = false;
        if (lastTid) {
            sameTid = lastTid[element->getLogId()] == element->getTid();
            // Dropped (chatty) immediately following a valid log from the
            // same source in the same log buffer indicates we have a
            // multiple identical squash.  chatty that differs source
            // is due to spam filter.  chatty to chatty of different
            // source is also due to spam filter.
            lastTid[element->getLogId()] =
                (element->getDropped() && !sameTid) ? 0 : element->getTid();
        }

        pthread_mutex_unlock(&mLogElementsLock);

        // 这里把单个 log 项的数据写回客户端
        // range locking in LastLogTimes looks after us
        max = element->flushTo(reader, this, privileged, sameTid);

        if (max == element->FLUSH_ERROR) {
            return max;
        }

        skip = maxSkip;
        pthread_mutex_lock(&mLogElementsLock);
    }
    pthread_mutex_unlock(&mLogElementsLock);

    return max;
}
```

最后我们看看 `LogBufferElement::flushTo` 函数：
```C++
// system/core/logd/LogBufferElement.cpp
log_time LogBufferElement::flushTo(SocketClient* reader, LogBuffer* parent,
                                   bool privileged, bool lastSame) {
    struct logger_entry_v4 entry;

    memset(&entry, 0, sizeof(struct logger_entry_v4));

    entry.hdr_size = privileged ? sizeof(struct logger_entry_v4)
                                : sizeof(struct logger_entry_v3);
    entry.lid = mLogId;
    entry.pid = mPid;
    entry.tid = mTid;
    entry.uid = mUid;
    entry.sec = mRealTime.tv_sec;
    entry.nsec = mRealTime.tv_nsec;

    struct iovec iovec[2];
    iovec[0].iov_base = &entry;
    iovec[0].iov_len = entry.hdr_size;

    char* buffer = NULL;

    if (!mMsg) {
        entry.len = populateDroppedMessage(buffer, parent, lastSame);
        if (!entry.len) return mRealTime;
        iovec[1].iov_base = buffer;
    } else {
        entry.len = mMsgLen;
        iovec[1].iov_base = mMsg;
    }
    iovec[1].iov_len = entry.len;

    log_time retval = reader->sendDatav(iovec, 1 + (entry.len != 0))
                          ? FLUSH_ERROR
                          : mRealTime;

    if (buffer) free(buffer);

    return retval;
}
```
由于我们需要将多个缓冲的数据写回客户端，这里使用是是 `writev`。实际的数据写入在 `SocketClient` 中实现：
```C++
// system/core/libsysutils/src/SocketClient.cpp
int SocketClient::sendDatav(struct iovec *iov, int iovcnt) {
    pthread_mutex_lock(&mWriteMutex);
    int rc = sendDataLockedv(iov, iovcnt);
    pthread_mutex_unlock(&mWriteMutex);

    return rc;
}

int SocketClient::sendDataLockedv(struct iovec *iov, int iovcnt) {

    if (mSocket < 0) {
        errno = EHOSTUNREACH;
        return -1;
    }

    if (iovcnt <= 0) {
        return 0;
    }

    int ret = 0;
    int e = 0; // SLOGW and sigaction are not inert regarding errno
    int current = 0;

    // 当向一个写入端已经被关闭的 socket 写入数据的时候，内核会发送 SIGPIPE，
    // 默认的行为是结束进程。这里我们要忽略它
    struct sigaction new_action, old_action;
    memset(&new_action, 0, sizeof(new_action));
    new_action.sa_handler = SIG_IGN;
    sigaction(SIGPIPE, &new_action, &old_action);

    for (;;) {
        ssize_t rc = TEMP_FAILURE_RETRY(
            writev(mSocket, iov + current, iovcnt - current));

        if (rc > 0) {
            size_t written = rc;
            // 可能只是部分写入了数据。逐个检查那些已经写入成功的 iov
            while ((current < iovcnt) && (written >= iov[current].iov_len)) {
                written -= iov[current].iov_len;
                current++;
            }
            if (current == iovcnt) {
                break;
            }
            // 这个 iov 有部分数据已经写入
            iov[current].iov_base = (char *)iov[current].iov_base + written;
            iov[current].iov_len -= written;
            continue;
        }

        if (rc == 0) {
            e = EIO;
            SLOGW("0 length write :(");
        } else {
            e = errno;
            SLOGW("write error (%s)", strerror(e));
        }
        ret = -1;
        break;
    }

    sigaction(SIGPIPE, &old_action, &new_action);

    if (e != 0) {
        errno = e;
    }
    return ret;
}
```

现在，恭喜你，log 数据的读取到这里就结束了。

<br><br>
