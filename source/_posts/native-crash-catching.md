---
title: Android native 崩溃信息捕获实践
date: 2019-04-06 09:35:52
categories: Android
tags: native-crash
description: 本篇是 bugly 一篇关于 native crash 捕获的文章的练习。由于他文章中已经给出了相关的大部分知识点，这里我就仅仅补充一些细节，并给出一个完整的 demo。
---

本篇是 bugly 一篇关于 native crash 捕获的文章的练习。由于他文章中已经给出了相关的大部分知识点，这里我就仅仅补充一些细节，并给出一个完整的 demo。建议大家先阅读 [Android 平台 Native 代码的崩溃捕获机制及实现
](https://mp.weixin.qq.com/s/g-WzYF3wWAljok1XjPoo7w?)，熟悉一下相关的知识。

相关代码可以在 [https://github.com/Jekton/NativeCrashCatching](https://github.com/Jekton/NativeCrashCatching)（目标平台是 Android 8） 找到。


### 设置 signal handler 的运行堆栈
```C++
static void SetUpStack() {
  stack_t stack{};
  stack.ss_sp = new(std::nothrow) char[SIGSTKSZ];

  if (!stack.ss_sp) {
    LOGW(kTag, "fail to alloc stack for crash catching");
    return;
  }
  stack.ss_size = SIGSTKSZ;
  stack.ss_flags = 0;
  if (stack.ss_sp) {
    if (sigaltstack(&stack, nullptr) != 0) {
      LOGERRNO(kTag, "fail to setup signal stack");
    }
  }
}
```
`SIGSTKSZ` 是一个 `signal.h` 预定义的常量，我们可以直接用它做目标的栈大小。`LOGERRNO, LOGD, LOGE` 等是我自己定义的打印 Android log 的宏。

### 设置信号处理函数
```C++
static std::map<int, struct sigaction> sOldHandlers;

static void SetUpSigHandler() {
  struct sigaction action{};
  action.sa_sigaction = SignalHandler;
  action.sa_flags = SA_SIGINFO | SA_ONSTACK;
  int signals[] = {
      SIGABRT, SIGBUS, SIGFPE, SIGILL, SIGSEGV, SIGPIPE
  };
  struct sigaction old_action;
  for (auto signo : signals) {
    if (sigaction(signo, &action, &old_action) == -1) {
      LOGERRNO(kTag, "fail to set signal handler for signo %d", signo);
    } else {
      if (old_action.sa_handler != SIG_DFL && old_action.sa_handler != SIG_IGN) {
        sOldHandlers[signo] = old_action;
      }
    }
  }
}
```
这里我们把旧的 signal handler 保存起来，执行完我们自己的函数后，再调用它们：
```C++
static void SignalHandler(int signo, siginfo_t* info, void* context) {
  DumpSignalInfo(info);
  DumpStacks(context);

  CallOldHandler(signo, info, context);
  exit(0);
}

static void CallOldHandler(int signo, siginfo_t* info, void* context) {
  auto it = sOldHandlers.find(signo);
  if (it != sOldHandlers.end()) {
    if (it->second.sa_flags & SA_SIGINFO) {
      it->second.sa_sigaction(signo, info, context);
    } else {
      it->second.sa_handler(signo);
    }
  }
}
```
`DumpSignalInfo` 用来打印 `siginfo_t`，`DumpStacks` 用来打印堆栈，很快我们就会看到他的实现。

### 打印 siginfo_t

打印 `siginfo_t` 没什么技术含量，就只是根据 `signo` 和 `si_code` 打印对应的消息。
```C++
static void DumpSignalInfo(siginfo_t* info) {
  switch (info->si_signo) {
  case SIGILL:
    LOGI(kTag, "signal SIGILL caught");
    switch (info->si_code) {
    case ILL_ILLOPC:
      LOGI(kTag, "illegal opcode");
      break;
    case ILL_ILLOPN:
      LOGI(kTag, "illegal operand");
      break;
    case ILL_ILLADR:
      LOGI(kTag, "illegal addressing mode");
      break;
    case ILL_ILLTRP:
      LOGI(kTag, "illegal trap");
      break;
    case ILL_PRVOPC:
      LOGI(kTag, "privileged opcode");
      break;
    case ILL_PRVREG:
      LOGI(kTag, "privileged register");
      break;
    case ILL_COPROC:
      LOGI(kTag, "coprocessor error");
      break;
    case ILL_BADSTK:
      LOGI(kTag, "internal stack error");
      break;
    default:
      LOGI(kTag, "code = %d", info->si_code);
      break;
    }
    break;
  case SIGFPE:
    LOGI(kTag, "signal SIGFPE caught");
    switch (info->si_code) {
    case FPE_INTDIV:
      LOGI(kTag, "integer divide by zero");
      break;
    case FPE_INTOVF:
      LOGI(kTag, "integer overflow");
      break;
    case FPE_FLTDIV:
      LOGI(kTag, "floating-point divide by zero");
      break;
    case FPE_FLTOVF:
      LOGI(kTag, "floating-point overflow");
      break;
    case FPE_FLTUND:
      LOGI(kTag, "floating-point underflow");
      break;
    case FPE_FLTRES:
      LOGI(kTag, "floating-point inexact result");
      break;
    case FPE_FLTINV:
      LOGI(kTag, "invalid floating-point operation");
      break;
    case FPE_FLTSUB:
      LOGI(kTag, "subscript out of range");
      break;
    default:
      LOGI(kTag, "code = %d", info->si_code);
      break;
    }
    break;
  case SIGSEGV:
    LOGI(kTag, "signal SIGSEGV caught");
    switch (info->si_code) {
    case SEGV_MAPERR:
      LOGI(kTag, "address not mapped to object");
      break;
    case SEGV_ACCERR:
      LOGI(kTag, "invalid permissions for mapped object");
      break;
    default:
      LOGI(kTag, "code = %d", info->si_code);
      break;
    }
    break;
  case SIGBUS:
    LOGI(kTag, "signal SIGBUS caught");
    switch (info->si_code) {
    case BUS_ADRALN:
      LOGI(kTag, "invalid address alignment");
      break;
    case BUS_ADRERR:
      LOGI(kTag, "nonexistent physical address");
      break;
    case BUS_OBJERR:
      LOGI(kTag, "object-specific hardware error");
      break;
    default:
      LOGI(kTag, "code = %d", info->si_code);
      break;
    }
    break;
  case SIGABRT:
    LOGI(kTag, "signal SIGABRT caught");
    break;
  case SIGPIPE:
    LOGI(kTag, "signal SIGPIPE caught");
    break;
  default:
    LOGI(kTag, "signo %d caught", info->si_signo);
    LOGI(kTag, "code = %d", info->si_code);
  }
  LOGI(kTag, "errno = %d", info->si_errno);
}
```

### 打印堆栈

最后是我们的重头戏 —— 打印堆栈。按 bugly 那文章的说法，直接在信号处理函数里调用 Java 函数经常会有问题（具体是什么问题，我也还没去看，理论上应该没关系才对），我们这里就先按他的建议，在后台起一个工作线程来打印堆栈。

我们在应用启动的时候，先启动一个后台线程：
```C++
static pid_t sTidToDump;    // guarded by sMutex
static void* sContext;
static std::mutex sMutex;
static std::condition_variable sCondition;
static void StackDumpingThread();

void InitCrashCaching() {
  LOGD(kTag, "InitCrashCaching");
  SetUpStack();
  SetUpSigHandler();
  std::thread{StackDumpingThread}.detach();
}
```
这里 `mutex` 和 `condition_variable` 用来给信号处理函数和这个工作线程通信，我们直接通过两个静态变量传递数据。

```C++
static void StackDumpingThread() {
  std::unique_lock<std::mutex> lock{sMutex};
  sCondition.wait(lock, [] { return sTidToDump > 0; });
  
  // dump stack

  sTidToDump = 0;
  // tell signal handler that we're done
  sCondition.notify_one();
}
```

现在我们可以继续看前面暂时放下的 `DumpStack` 函数了：
```C++
static void DumpStacks(void* context) {
  std::unique_lock<std::mutex> lock{sMutex};
  sTidToDump = gettid();    // 获取线程 id
  sContext = context;
  sCondition.notify_one();
  // 等待工作线程打印堆栈
  sCondition.wait(lock, []{ return sTidToDump == 0; });
}
```

以上就是 native 崩溃捕获的一个基本框架，下面我们看看如何获取堆栈。

bugly 文章给我们推荐的 libunwind，但这里我们使用另一个朋友推荐的 libbacktrace。libbacktrace 其实也是用 libunwind 实现的。为了绕开 Android N 以后的 classloader namespace 限制，我们用 [ndk_dlopen](https://github.com/Rprop/ndk_dlopen) 来加载 `libbacktrace.so`。

使用系统内置的 so 有一个好处，就是不用自己去编译共享库，并且 so 很可能根据不同的系统版本做了调整。坏处就是我们代码的兼容性会比较差（这里我给出的代码只能运行在 Android 8 上，如果是其他版本，读者需要自己根据系统源码做一些调整）。

libbacktrace 的源码在 `system/core` 下面，为了了解一个库的用法，一般是看看他相关的文档、头文件。很不幸的，libbacktrace 没有文档，查看源码目录，可以看到一个 `Backtrace.h`。这种跟库名一样的头文件名，一般就是库对外的接口。

另一个方向是，既然 libbacktrace 在 `system/core` 里，系统可能有某个地方使用了它，我们可以全局搜索 `system/core` 找一个使用了 libbacktrace 的地方，然后参考它的用法。这里我参考的是 `CallStack`：
```C++
// system/core/libutils/CallStack.cpp
void CallStack::update(int32_t ignoreDepth, pid_t tid) {
    mFrameLines.clear();

    std::unique_ptr<Backtrace> backtrace(Backtrace::Create(BACKTRACE_CURRENT_PROCESS, tid));
    if (!backtrace->Unwind(ignoreDepth)) {
        ALOGW("%s: Failed to unwind callstack.", __FUNCTION__);
    }
    for (size_t i = 0; i < backtrace->NumFrames(); i++) {
      mFrameLines.push_back(String8(backtrace->FormatFrameData(i).c_str()));
    }
}
```

可以看到，使用 libbacktrace 一共就 3 步：
1. 使用 `Backtrace::Create` 创建一个 `Backtrace` 实例
2. 调用 `Unwind` 函数 unwind 一下 stack
3. `FormatFrameData` 输出每个栈帧的文本信息（也可以自己根据 frame 自己打印）

下面我们先看看封装了 `libbacktrace` 的 `GetStackTrace` 接口，然后分小节来看这几个步骤。

#### GetStackTrace 接口

```C++
class GetTraceCallback {
 public:
  virtual void OnFrame(size_t frame_num, std::string frame) = 0;
  virtual void OnFail() = 0;
  virtual ~GetTraceCallback() {}
};

/*
 * @ctx can be nullptr or context from signal handler
 */
void GetStackTrace(pid_t tid, void* ctx, GetTraceCallback* callback);
```
`tid` 是需要打印对象的线程的 id，`ctx` 是信号处理函数的第三参数 `context`，`callback` 用于接收堆栈。之所以我们一个一个 frame 地传，是因为一次性打印堆栈过大，会被 Android 的 log 截断。

#### 创建 `Backtrace` 实例

为了使用 `ndk_dlopen`，我们先在 `InitCrashCaching` 的时候初始化它：
```C++
extern "C" JNIEXPORT void JNICALL
Java_com_example_nativecrashcatching_CrashCatching_initNative(
    JNIEnv* env, jclass clazz) {
  ndk_init(env);
  InitCrashCaching();
}
```

为了找到 `Backtrace::Create` 函数的在 so 里的符号，我们可以使用 `nm`：
```
$ aarch64-linux-android-nm -D libbacktrace.so | grep Create
00004e10 T _ZN12BacktraceMap6CreateEiRKNSt3__16vectorI15backtrace_map_tNS0_9allocatorIS2_EEEE
0000aab8 T _ZN12BacktraceMap6CreateEib
00008dec T _ZN9Backtrace6CreateEiiP12BacktraceMap
```
`aarch64-linux-android-nm` 可以在类似 `ndk-bundle/toolchains/aarch64-linux-android-4.9/prebuilt/darwin-x86_64/bin` 的路径里找到。

由于 so 里的符号都是 mangle 过的（C++ name mangling），我们可以先根据关键字 grep 出相关的符号，然后用 [https://demangler.com/](https://demangler.com/) demangle 出原来的符号名。`Backtrace::Create` 对应的符号是 `_ZN9Backtrace6CreateEiiP12BacktraceMap`。

知道函数对应的符号后，我们就可以用 `dlsym` 来找他了：
```C++
const char* kLibBacktrace = "libbacktrace.so";

static BacktraceStub* CreateBacktrace(pid_t tid) {
  auto deleter = [](void* handle) { ndk_dlclose(handle); };
  std::unique_ptr<void, decltype(deleter)> handle{ndk_dlopen(kLibBacktrace, RTLD_LAZY), deleter};
  if (!handle) {
    LOGERRNO(kTag, "CrateBacktrace, fail to dlopen %s", kLibBacktrace);
    return nullptr;
  }

  using BacktraceCreate = BacktraceStub* (*)(pid_t pid, pid_t tid, void* map);
  union { void* p; BacktraceCreate fn; } backtrace_create;
  backtrace_create.p = ndk_dlsym(handle.get(), "_ZN9Backtrace6CreateEiiP12BacktraceMap");
  if (!backtrace_create.p) {
    LOGE(kTag, "CrateBacktrace, fail to get symbol Backtrace::Create: %s", ndk_dlerror());
    return nullptr;
  }

  return backtrace_create.fn(BACKTRACE_CURRENT_PROCESS, tid, nullptr);
}
```
由于 C++ 不给我们把 `void*` 转成函数指针，这里只能曲线救国，用一个 `union` 来转换。`Backtrace::Create` 后会返回一个 `Backtrace` 指针。这里我们有两个选择，一是把整个 `Backtrace` 类的定义后拷贝过来，二是我们仿照他的定义，只加入我们需要的一小部分。我们选择的是后者。至于他的作用，我们很快就会看到。

#### 调用 `Unwind` 函数 unwind 一下 stack

查看原始的 `libbacktrace`，我们可以知道，`Unwind` 函数是一个虚函数。为了调用它，有两条路可以选择。
1. 拿到类的虚函数表，然后根据编译器的规则，算出 `Unwind` 的 offset
2. 定义一个跟 `Backtrace` 具有相同虚函数表的类，然后利用这个类来得到 `Unwind` 的 offset

从实现的角度，第二种方法虽然比较骚，但却比第一种简单很多。于是，我们定义了一个 `BacktraceStub`：
```C++
class BacktraceStub {
public:
  virtual ~BacktraceStub() {}
  virtual bool Unwind(size_t num_ignore_frames, void* context = NULL) = 0;
  virtual std::string GetFunctionName(uint64_t pc, uint64_t* offset,
                                      const backtrace_map_t* map = NULL) = 0;
  virtual void FillInMap(uint64_t pc, backtrace_map_t* map) = 0;
  virtual bool ReadWord(uint64_t ptr, word_t* out_value) = 0;
  virtual size_t Read(uint64_t addr, uint8_t* buffer, size_t bytes) = 0;
  virtual std::string FormatFrameData(size_t frame_num) = 0;

protected:
  virtual std::string GetFunctionNameRaw(uint64_t pc, uint64_t* offset) = 0;
  virtual bool VerifyReadWordArgs(uint64_t ptr, word_t* out_value) = 0;
};
```
除了我们需要用到的 `~BacktraceStub`、`Unwind` 和 `FormatFrameData`，其他函数其实可以随意定义。这部分相关的知识，读者可以参考《深度探索C++对象模型》。

有了这个 `BacktraceStub`，我们就可以调用 `Unwind` 了：
```C++
void GetStackTrace(pid_t tid, void* ctx, GetTraceCallback* callback) {
  std::unique_ptr<BacktraceStub> backtrace{CreateBacktrace(tid)};
  if (!backtrace) {
    callback->OnFail();
    return;
  }
  const auto ignoreDepth = 0;
  if (!backtrace->Unwind(ignoreDepth)) {
    LOGE(kTag, "GetStackTrace, fail to unwind stack");
    callback->OnFail();
    return;
  }

  ...
  ...
  ...
}
```
`ignoreDepth` 是忽略掉栈顶的 frame 数，我们传入 0 即可。

#### `FormatFrameData` 输出每个栈帧的文本信息

回忆一下前面的 `DumpStacks`：
```
static void DumpStacks(void* context) {
  std::unique_lock<std::mutex> lock{sMutex};
  sTidToDump = gettid();
  sContext = context;
  sCondition.notify_one();
  sCondition.wait(lock, []{ return sTidToDump == 0; });
}
```
由于我们信号处理函数里又多执行了一部分代码，最后拿到的堆栈会多出来几个。为了去掉这些，我们需要从信号处理函数的 `context` 里拿到异常发生时的 PC 值：
```C++
void GetStackTrace(pid_t tid, void* ctx, GetTraceCallback* callback) {
  ...
  ...

  auto context = reinterpret_cast<ucontext_t*>(ctx);
  // uc_mcontext.pc is the next instruction to be executed
  auto pc = static_cast<uint64_t>(context->uc_mcontext.pc) - 4;

  ...
  ...
}
```

为了拿到 libbacktrace 中栈帧的数据，我们需要再拷贝多一些类定义：
```C++
enum BacktraceUnwindError : uint32_t {
  BACKTRACE_UNWIND_NO_ERROR,
  // Something failed while trying to perform the setup to begin the unwind.
  BACKTRACE_UNWIND_ERROR_SETUP_FAILED,
  // There is no map information to use with the unwind.
  BACKTRACE_UNWIND_ERROR_MAP_MISSING,
  // An error occurred that indicates a programming error.
  BACKTRACE_UNWIND_ERROR_INTERNAL,
  // The thread to unwind has disappeared before the unwind can begin.
  BACKTRACE_UNWIND_ERROR_THREAD_DOESNT_EXIST,
  // The thread to unwind has not responded to a signal in a timely manner.
  BACKTRACE_UNWIND_ERROR_THREAD_TIMEOUT,
  // Attempt to do an unsupported operation.
  BACKTRACE_UNWIND_ERROR_UNSUPPORTED_OPERATION,
  // Attempt to do an offline unwind without a context.
  BACKTRACE_UNWIND_ERROR_NO_CONTEXT,
};

struct backtrace_map_t {
  uintptr_t start = 0;
  uintptr_t end = 0;
  uintptr_t offset = 0;
  uintptr_t load_base = 0;
  int flags = 0;
  std::string name;
};

struct backtrace_frame_data_t {
  size_t num;             // The current fame number.
  uintptr_t pc;           // The absolute pc.
  uintptr_t sp;           // The top of the stack.
  size_t stack_size;      // The size of the stack, zero indicate an unknown stack size.
  backtrace_map_t map;    // The map associated with the given pc.
  std::string func_name;  // The function name associated with this pc, NULL if not found.
  uintptr_t func_offset;  // pc relative to the start of the function, only valid if func_name is not NULL.
};

using word_t = unsigned long;

class BacktraceMap;

class BacktraceStub {
public:
  // virtual functions

  size_t NumFrames() const { return frames_.size(); }

  const backtrace_frame_data_t* GetFrame(size_t frame_num) {
    if (frame_num >= frames_.size()) {
      return nullptr;
    }
    return &frames_[frame_num];
  }

protected:
  pid_t pid_;
  pid_t tid_;

  BacktraceMap* map_;
  bool map_shared_;
  std::vector<backtrace_frame_data_t> frames_;
  // Skip frames in libbacktrace/libunwindstack when doing a local unwind.
  BacktraceUnwindError error_;
};
```
前面这些都是 libbacktrace 里拷贝出来的。由于我的测试机是 Android 8，所以使用 `system/core` 的 `oreo-release` 分支。读者需要根据自己手机系统的版本做一些调整。

下面是 `GetStackTrace` 的完整实现：
```
void GetStackTrace(pid_t tid, void* ctx, GetTraceCallback* callback) {
  std::unique_ptr<BacktraceStub> backtrace{CreateBacktrace(tid)};
  if (!backtrace) {
    callback->OnFail();
    return;
  }
  const auto ignoreDepth = 0;
  if (!backtrace->Unwind(ignoreDepth)) {
    LOGE(kTag, "GetStackTrace, fail to unwind stack");
    callback->OnFail();
    return;
  }

  auto context = reinterpret_cast<ucontext_t*>(ctx);
  // uc_mcontext.pc is the next instruction to be executed
  auto pc = static_cast<uint64_t>(context->uc_mcontext.pc) - 4;
  size_t j = 0;
  for (size_t i = 0, size = backtrace->NumFrames(); i < size; ++i) {
    auto frame = backtrace->GetFrame(i);
    // skip frames due to notification of dumping thread
    if (j == 0 && frame->pc != pc) continue;
    const_cast<backtrace_frame_data_t*>(frame)->num = j;
    auto frame_str = backtrace->FormatFrameData(i);
    ++j;
    callback->OnFrame(i, frame_str);
  }
}
```
我们忽略掉前面的几个栈帧后，为了让 `FormatFrameData` 输出的标号从 0 开始，我们还手动修改了 `frame->num`。下面是一个输出示例：
```
#00 pc 0000000000019870  /data/app/com.example.nativecrashcatching-cSDYvNiWs8hShuhH39mQUQ==/lib/arm64/libnative-lib.so (_Z3foov+16)
#01 pc 0000000000019894  /data/app/com.example.nativecrashcatching-cSDYvNiWs8hShuhH39mQUQ==/lib/arm64/libnative-lib.so (Java_com_example_nativecrashcatching_CrashCatching_dieNative+20)
#02 pc 00000000001fd700  /system/lib64/libart.so (offset 0x2f1000)
#03 pc 00000000001f4638  /system/lib64/libart.so (offset 0x2f1000)
#04 pc 00000000000d80b4  /system/lib64/libart.so (_ZN3art9ArtMethod6InvokeEPNS_6ThreadEPjjPNS_6JValueEPKc+260)
#05 pc 00000000002821dc  /system/lib64/libart.so (_ZN3art11interpreter34ArtInterpreterToCompiledCodeBridgeEPNS_6ThreadEPNS_9ArtMethodEPKNS_7DexFile8CodeItemEPNS_11ShadowFrameEPNS_6JValueE+352)
#06 pc 000000000027c8a4  /system/lib64/libart.so (_ZN3art11interpreter6DoCallILb0ELb0EEEbPNS_9ArtMethodEPNS_6ThreadERNS_11ShadowFrameEPKNS_11InstructionEtPNS_6JValueE+672)
#07 pc 00000000001dd130  /system/lib64/libart.so (offset 0x2f1000)
#08 pc 00000000001e5e94  /system/lib64/libart.so (offset 0x2f1000)
#09 pc 000000000025d620  /system/lib64/libart.so (_ZN3art11interpreterL7ExecuteEPNS_6ThreadEPKNS_7DexFile8CodeItemERNS_11ShadowFrameENS_6JValueEb+444)
#10 pc 0000000000263d20  /system/lib64/libart.so (_ZN3art11interpreter33ArtInterpreterToInterpreterBridgeEPNS_6ThreadEPKNS_7DexFile8CodeItemEPNS_11ShadowFrameEPNS_6JValueE+212)
#11 pc 000000000027c884  /system/lib64/libart.so (_ZN3art11interpreter6DoCallILb0ELb0EEEbPNS_9ArtMethodEPNS_6ThreadERNS_11ShadowFrameEPKNS_11InstructionEtPNS_6JValueE+640)
#12 pc 00000000001dd130  /system/lib64/libart.so (offset 0x2f1000)
#13 pc 00000000001e5e94  /system/lib64/libart.so (offset 0x2f1000)
#14 pc 000000000025d620  /system/lib64/libart.so (_ZN3art11interpreterL7ExecuteEPNS_6ThreadEPKNS_7DexFile8CodeItemERNS_11ShadowFrameENS_6JValueEb+444)
#15 pc 0000000000263d20  /system/lib64/libart.so (_ZN3art11interpreter33ArtInterpreterToInterpreterBridgeEPNS_6ThreadEPKNS_7DexFile8CodeItemEPNS_11ShadowFrameEPNS_6JValueE+212)
```

最后再提一个小知识点。前面打印出来的 pc 是相对地址，我们从信号处理还是里拿到的 `pc` 值是绝对地址，`frame->pc` 也是绝对地址。为了把信号处理函数中的 `pc` 值转成相对地址，可以使用 `dladdr`：
```
static uint64_t GetFaultPcRelative(ucontext_t* context) {
  void* pc = reinterpret_cast<void*>(context->uc_mcontext.pc);
  Dl_info dl_info;
  if (dladdr(pc, &dl_info)) {
    auto base = reinterpret_cast<uint64_t>(dl_info.dli_fbase);
    return reinterpret_cast<uint64_t>(pc) - base;
  } else {
    return 0;
  }
}
```
`pie-release` 的 libbacktrace 的 `backtrace_frame_data_t` 直接带了一个成员变量 `rel_pc`。低版本的代码读者可以从 libbacktrace 源码中找到将绝对地址转换为相对地址的代码。


