title: jvm初始化启动
author: jianghe
tags:
  - jvm
categories:
  - jvm
date: 2021-03-02 11:21:00
---
### Launcher简介
Launcher是一种用于启动jvm进程的启动器，关于Launcher的具体使用方式和标准选项配置，输入命令“java -help”，见下。

```java
Usage: java [-options] class [args...]
           (to execute a class)
   or  java [-options] -jar jarfile [args...]
           (to execute a jar file)
where options include:
    -d32	  use a 32-bit data model if available
    -d64	  use a 64-bit data model if available
    -server	  to select the "server" VM
                  The default VM is server,
                  because you are running on a server-class machine.


    -cp <class search path of directories and zip/jar files>
    -classpath <class search path of directories and zip/jar files>
                  A : separated list of directories, JAR archives,
                  and ZIP archives to search for class files.
    -D<name>=<value>
                  set a system property
    -verbose:[class|gc|jni]
                  enable verbose output
    -version      print product version and exit
    -version:<value>
                  Warning: this feature is deprecated and will be removed
                  in a future release.
                  require the specified version to run
    -showversion  print product version and continue
    -jre-restrict-search | -no-jre-restrict-search
                  Warning: this feature is deprecated and will be removed
                  in a future release.
                  include/exclude user private JREs in the version search
    -? -help      print this help message
    -X            print help on non-standard options
    -ea[:<packagename>...|:<classname>]
    -enableassertions[:<packagename>...|:<classname>]
                  enable assertions with specified granularity
    -da[:<packagename>...|:<classname>]
    -disableassertions[:<packagename>...|:<classname>]
                  disable assertions with specified granularity
    -esa | -enablesystemassertions
                  enable system assertions
    -dsa | -disablesystemassertions
                  disable system assertions
    -agentlib:<libname>[=<options>]
                  load native agent library <libname>, e.g. -agentlib:hprof
                  see also, -agentlib:jdwp=help and -agentlib:hprof=help
    -agentpath:<pathname>[=<options>]
                  load native agent library by full pathname
    -javaagent:<jarpath>[=<options>]
                  load Java programming language agent, see java.lang.instrument
    -splash:<imagepath>
                  show splash screen with specified image
```
> 这里一定要注意，Launcher不是虚拟机实现，从严格意义上讲，Launcher只是一个封装了虚拟机的执行外壳。由它负责装载jre环境和windows平台下的jvm.dll动态链接库（linux平台下则是装载libjvm.so）,它必须调用虚拟机中的函数来完毕运行java程序的功能。所以，你在Hotspot的源码找不到启动程序的main函数。
<font color='red'>openjdk7是有一个启动器的，在openjdk/hotspot/src/share/tools/layncher/java.c中main函数
openjdk8后，启动器在openjdk/jdk/src/share/bin/main.c中</font>


### HotSpot源码目录结构
```java
├─agent                            Serviceability Agent的客户端实现
├─make                             用来build出HotSpot的各种配置文件
├─src                              HotSpot VM的源代码
│  ├─cpu                            CPU相关代码（汇编器、模板解释器、ad文件、部分runtime函数在这里实现）
│  ├─os                             操作系相关代码
│  ├─os_cpu                         操作系统+CPU的组合相关的代码
│  └─share                          平台无关的共通代码
│      ├─tools                        工具
│      │  ├─hsdis                      反汇编插件
│      │  ├─IdealGraphVisualizer       将server编译器的中间代码可视化的工具
│      │  ├─launcher                   openjdk1.7之前存在launcher目录，这里存放启动程序“java”
│      │  ├─LogCompilation             将-XX:+LogCompilation输出的日志（hotspot.log）整理成更容易阅读的格式的工具
│      │  └─ProjectCreator             生成Visual Studio的project文件的工具
│      └─vm                           HotSpot VM的核心代码
│          ├─adlc                       平台描述文件（上面的cpu或os_cpu里的*.ad文件）的编译器
│          ├─asm                        汇编器接口
│          ├─c1                         client编译器（又称“C1”）
│          ├─ci                         动态编译器的公共服务/从动态编译器到VM的接口
│          ├─classfile                  类文件的处理（包括类加载和系统符号表等）
│          ├─code                       动态生成的代码的管理
│          ├─compiler                   从VM调用动态编译器的接口
│          ├─gc_implementation          GC的实现
│          │  ├─concurrentMarkSweep      Concurrent Mark Sweep GC的实现
│          │  ├─g1                       Garbage-First GC的实现（不使用老的分代式GC框架）
│          │  ├─parallelScavenge         ParallelScavenge GC的实现（server VM默认，不使用老的分代式GC框架）
│          │  ├─parNew                   ParNew GC的实现
│          │  └─shared                   GC的共通实现
│          ├─gc_interface               GC的接口
│          ├─interpreter                解释器，包括“模板解释器”（官方版在用）和“C++解释器”（官方版不在用）
│          ├─libadt                     一些抽象数据结构
│          ├─memory                     内存管理相关（老的分代式GC框架也在这里）
│          ├─oops                       HotSpot VM的对象系统的实现
│          ├─opto                       server编译器（又称“C2”或“Opto”）
│          ├─prims                      HotSpot VM的对外接口，包括部分标准库的native部分和JVMTI实现
│          ├─runtime                    运行时支持库（包括线程管理、编译器调度、锁、反射等）
│          ├─services                   主要是用来支持JMX之类的管理功能的接口
│          ├─shark                      基于LLVM的JIT编译器（官方版里没有使用）
│          └─utilities                  一些基本的工具类
└─test                             单元测试
```

### 使用Launcher启动jvm
调用栈

```java
openjdk/jdk/src/share/bin/main.c::WinMain/main
  openjdk/jdk/src/share/bin/java.c::JLI_Launch
    openjdk/jdk/src/solaris/bin/java_md_solinux.c::LoadJavaVM # Load JVM Library: libjvm.so
    openjdk/jdk/src/solaris/bin/java_md_solinux.c::JVMInit # Create JVM
      openjdk/jdk/src/share/bin/java.c::ContinueInNewThread
        openjdk/jdk/src/solaris/bin/java_md_solinux.c::ContinueInNewThread0(JavaMain, threadStackSize, (void*)&args);
          pthread_create(&tid, &attr, (void *(*)(void*))continuation, (void*)args)
            openjdk/jdk/src/share/bin/java.c::JavaMain
              openjdk/jdk/src/share/bin/java.c::InitializeJVM
                openjdk\hotspot\src\share\vm\prims\jni.cpp::JNI_CreateJavaVM
```
调用过程
1. main函数
启动函数

```c++
int WINAPI
WinMain(HINSTANCE inst, HINSTANCE previnst, LPSTR cmdline, int cmdshow)
{
    int margc;
    char** margv;
    const jboolean const_javaw = JNI_TRUE;

    __initenv = _environ;

#else /* JAVAW */
int
main(int argc, char **argv)
{
    int margc;
    char** margv;
    const jboolean const_javaw = JNI_FALSE;
#endif /* JAVAW */
#ifdef _WIN32
    {
        int i = 0;
        if (getenv(JLDEBUG_ENV_ENTRY) != NULL) {
            printf("Windows original main args:\n");
            for (i = 0 ; i < __argc ; i++) {
                printf("wwwd_args[%d] = %s\n", i, __argv[i]);
            }
        }
    }
    JLI_CmdToArgs(GetCommandLine());
    margc = JLI_GetStdArgc();
    // add one more to mark the end
    margv = (char **)JLI_MemAlloc((margc + 1) * (sizeof(char *)));
    {
        int i = 0;
        StdArg *stdargs = JLI_GetStdArgs();
        for (i = 0 ; i < margc ; i++) {
            margv[i] = stdargs[i].arg;
        }
        margv[i] = NULL;
    }
#else /* *NIXES */
    margc = argc;
    margv = argv;
#endif /* WIN32 */
	# 调用launcher
    return JLI_Launch(margc, margv,
                   sizeof(const_jargs) / sizeof(char *), const_jargs,
                   sizeof(const_appclasspath) / sizeof(char *), const_appclasspath,
                   FULL_VERSION,
                   DOT_VERSION,
                   (const_progname != NULL) ? const_progname : *margv,
                   (const_launcher != NULL) ? const_launcher : *margv,
                   (const_jargs != NULL) ? JNI_TRUE : JNI_FALSE,
                   const_cpwildcard, const_javaw, const_ergo_class);
}

```

2. JLI_Launch
创建运行环境

```c++

/*
 * Entry point.
 */
int
JLI_Launch(int argc, char ** argv,              /* main argc, argc */
        int jargc, const char** jargv,          /* java args */
        int appclassc, const char** appclassv,  /* app classpath */
        const char* fullversion,                /* full version defined */
        const char* dotversion,                 /* dot version defined */
        const char* pname,                      /* program name */
        const char* lname,                      /* launcher name */
        jboolean javaargs,                      /* JAVA_ARGS */
        jboolean cpwildcard,                    /* classpath wildcard*/
        jboolean javaw,                         /* windows-only javaw */
        jint ergo                               /* ergonomics class policy */
)
{
    int mode = LM_UNKNOWN;
    char *what = NULL;
    char *cpath = 0;
    char *main_class = NULL;
    int ret;
    InvocationFunctions ifn;
    jlong start, end;
    char jvmpath[MAXPATHLEN];
    char jrepath[MAXPATHLEN];
    char jvmcfg[MAXPATHLEN];

    _fVersion = fullversion;
    _dVersion = dotversion;
    _launcher_name = lname;
    _program_name = pname;
    _is_java_args = javaargs;
    _wc_enabled = cpwildcard;
    _ergo_policy = ergo;

    InitLauncher(javaw);
    DumpState();
    if (JLI_IsTraceLauncher()) {
        int i;
        printf("Command line args:\n");
        for (i = 0; i < argc ; i++) {
            printf("argv[%d] = %s\n", i, argv[i]);
        }
        AddOption("-Dsun.java.launcher.diag=true", NULL);
    }

    /*
     * Make sure the specified version of the JRE is running.
     *
     * There are three things to note about the SelectVersion() routine:
     *  1) If the version running isn't correct, this routine doesn't
     *     return (either the correct version has been exec'd or an error
     *     was issued).
     *  2) Argc and Argv in this scope are *not* altered by this routine.
     *     It is the responsibility of subsequent code to ignore the
     *     arguments handled by this routine.
     *  3) As a side-effect, the variable "main_class" is guaranteed to
     *     be set (if it should ever be set).  This isn't exactly the
     *     poster child for structured programming, but it is a small
     *     price to pay for not processing a jar file operand twice.
     *     (Note: This side effect has been disabled.  See comment on
     *     bugid 5030265 below.)
     */
    SelectVersion(argc, argv, &main_class);

    CreateExecutionEnvironment(&argc, &argv,
                               jrepath, sizeof(jrepath),
                               jvmpath, sizeof(jvmpath),
                               jvmcfg,  sizeof(jvmcfg));

    ifn.CreateJavaVM = 0;
    ifn.GetDefaultJavaVMInitArgs = 0;

    if (JLI_IsTraceLauncher()) {
        start = CounterGet();
    }
		
    # 加载虚拟机动态链接库，并初始化ifn的函数指针
    if (!LoadJavaVM(jvmpath, &ifn)) {
        return(6);
    }

    if (JLI_IsTraceLauncher()) {
        end   = CounterGet();
    }

    JLI_TraceLauncher("%ld micro seconds to LoadJavaVM\n",
             (long)(jint)Counter2Micros(end-start));

    ++argv;
    --argc;

    if (IsJavaArgs()) {
        /* Preprocess wrapper arguments */
        TranslateApplicationArgs(jargc, jargv, &argc, &argv);
        if (!AddApplicationOptions(appclassc, appclassv)) {
            return(1);
        }
    } else {
        /* Set default CLASSPATH */
        cpath = getenv("CLASSPATH");
        if (cpath == NULL) {
            cpath = ".";
        }
        SetClassPath(cpath);
    }

    /* Parse command line options; if the return value of
     * ParseArguments is false, the program should exit.
     */
    if (!ParseArguments(&argc, &argv, &mode, &what, &ret, jrepath))
    {
        return(ret);
    }

    /* Override class path if -jar flag was specified */
    if (mode == LM_JAR) {
        SetClassPath(what);     /* Override class path */
    }

    /* set the -Dsun.java.command pseudo property */
    SetJavaCommandLineProp(what, argc, argv);

    /* Set the -Dsun.java.launcher pseudo property */
    SetJavaLauncherProp();

    /* set the -Dsun.java.launcher.* platform properties */
    SetJavaLauncherPlatformProps();

    return JVMInit(&ifn, threadStackSize, argc, argv, mode, what, ret);
}
```

那对应的LoadJavaVM是什么?见下面代码
其中设置ifn函数的属性

```c++
jboolean
LoadJavaVM(const char *jvmpath, InvocationFunctions *ifn)
{
    void *libjvm;

    JLI_TraceLauncher("JVM path is %s\n", jvmpath);

    libjvm = dlopen(jvmpath, RTLD_NOW + RTLD_GLOBAL);
    if (libjvm == NULL) {
#if defined(__solaris__) && defined(__sparc) && !defined(_LP64) /* i.e. 32-bit sparc */
      FILE * fp;
      Elf32_Ehdr elf_head;
      int count;
      int location;

      fp = fopen(jvmpath, "r");
      if (fp == NULL) {
        JLI_ReportErrorMessage(DLL_ERROR2, jvmpath, dlerror());
        return JNI_FALSE;
      }

      /* read in elf header */
      count = fread((void*)(&elf_head), sizeof(Elf32_Ehdr), 1, fp);
      fclose(fp);
      if (count < 1) {
        JLI_ReportErrorMessage(DLL_ERROR2, jvmpath, dlerror());
        return JNI_FALSE;
      }

      /*
       * Check for running a server vm (compiled with -xarch=v8plus)
       * on a stock v8 processor.  In this case, the machine type in
       * the elf header would not be included the architecture list
       * provided by the isalist command, which is turn is gotten from
       * sysinfo.  This case cannot occur on 64-bit hardware and thus
       * does not have to be checked for in binaries with an LP64 data
       * model.
       */
      if (elf_head.e_machine == EM_SPARC32PLUS) {
        char buf[257];  /* recommended buffer size from sysinfo man
                           page */
        long length;
        char* location;

        length = sysinfo(SI_ISALIST, buf, 257);
        if (length > 0) {
            location = JLI_StrStr(buf, "sparcv8plus ");
          if (location == NULL) {
            JLI_ReportErrorMessage(JVM_ERROR3);
            return JNI_FALSE;
          }
        }
      }
#endif
        JLI_ReportErrorMessage(DLL_ERROR1, __LINE__);
        JLI_ReportErrorMessage(DLL_ERROR2, jvmpath, dlerror());
        return JNI_FALSE;
    }
# 设置ifn函数引用
    ifn->CreateJavaVM = (CreateJavaVM_t)
        dlsym(libjvm, "JNI_CreateJavaVM");
    if (ifn->CreateJavaVM == NULL) {
        JLI_ReportErrorMessage(DLL_ERROR2, jvmpath, dlerror());
        return JNI_FALSE;
    }

    ifn->GetDefaultJavaVMInitArgs = (GetDefaultJavaVMInitArgs_t)
        dlsym(libjvm, "JNI_GetDefaultJavaVMInitArgs");
    if (ifn->GetDefaultJavaVMInitArgs == NULL) {
        JLI_ReportErrorMessage(DLL_ERROR2, jvmpath, dlerror());
        return JNI_FALSE;
    }

    ifn->GetCreatedJavaVMs = (GetCreatedJavaVMs_t)
        dlsym(libjvm, "JNI_GetCreatedJavaVMs");
    if (ifn->GetCreatedJavaVMs == NULL) {
        JLI_ReportErrorMessage(DLL_ERROR2, jvmpath, dlerror());
        return JNI_FALSE;
    }

    return JNI_TRUE;
}
```

那对应JVMInit是什么？

```c++
int
JVMInit(InvocationFunctions* ifn, jlong threadStackSize,
        int argc, char **argv,
        int mode, char *what, int ret)
{
    ShowSplashScreen();
    return ContinueInNewThread(ifn, threadStackSize, argc, argv, mode, what, ret);
}
```

3. ContinueInNewThread函数

调用ContinueInNewThread0函数

```c++
int
ContinueInNewThread(InvocationFunctions* ifn, jlong threadStackSize,
                    int argc, char **argv,
                    int mode, char *what, int ret)
{

    /*
     * If user doesn't specify stack size, check if VM has a preference.
     * Note that HotSpot no longer supports JNI_VERSION_1_1 but it will
     * return its default stack size through the init args structure.
     */
    if (threadStackSize == 0) {
      struct JDK1_1InitArgs args1_1;
      memset((void*)&args1_1, 0, sizeof(args1_1));
      args1_1.version = JNI_VERSION_1_1;
      ifn->GetDefaultJavaVMInitArgs(&args1_1);  /* ignore return value */
      if (args1_1.javaStackSize > 0) {
         threadStackSize = args1_1.javaStackSize;
      }
    }

    { /* Create a new thread to create JVM and invoke main method */
      JavaMainArgs args;
      int rslt;

      args.argc = argc;
      args.argv = argv;
      args.mode = mode;
      args.what = what;
      args.ifn = *ifn;

      rslt = ContinueInNewThread0(JavaMain, threadStackSize, (void*)&args);
      /* If the caller has deemed there is an error we
       * simply return that, otherwise we return the value of
       * the callee
       */
      return (ret != 0) ? ret : rslt;
    }
}

```

4. ContinueInNewThread0函数

创建一个新的线程，然后运行JavaMain函数。启动新的线程，Launcher就会阻塞原有线程，其中int (JNICALL *continuation)(void *)这个参数就是ContinueInNewThread参数中的JavaMain函数

```c++
int
ContinueInNewThread0(int (JNICALL *continuation)(void *), jlong stack_size, void * args) {
    int rslt;
#ifdef __linux__
    pthread_t tid;
    pthread_attr_t attr;
    pthread_attr_init(&attr);
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_JOINABLE);

    if (stack_size > 0) {
      pthread_attr_setstacksize(&attr, stack_size);
    }

    if (pthread_create(&tid, &attr, (void *(*)(void*))continuation, (void*)args) == 0) {
      void * tmp;
      pthread_join(tid, &tmp);
      rslt = (int)tmp;
    } else {
     /*
      * Continue execution in current thread if for some reason (e.g. out of
      * memory/LWP)  a new thread can't be created. This will likely fail
      * later in continuation as JNI_CreateJavaVM needs to create quite a
      * few new threads, anyway, just give it a try..
      */
      rslt = continuation(args);
    }

    pthread_attr_destroy(&attr);
#else /* ! __linux__ */
    thread_t tid;
    long flags = 0;
    if (thr_create(NULL, stack_size, (void *(*)(void *))continuation, args, flags, &tid) == 0) {
      void * tmp;
      thr_join(tid, NULL, &tmp);
      rslt = (int)tmp;
    } else {
      /* See above. Continue in current thread if thr_create() failed */
      rslt = continuation(args);
    }
#endif /* __linux__ */
    return rslt;
}
```

5. JavaMain函数

InitializeJVM就是初始化java虚拟机，然后LoadMainClass加载主类

```c++
int JNICALL
JavaMain(void * _args)
{
    JavaMainArgs *args = (JavaMainArgs *)_args;
    int argc = args->argc;
    char **argv = args->argv;
    int mode = args->mode;
    char *what = args->what;
    InvocationFunctions ifn = args->ifn;

    JavaVM *vm = 0;
    JNIEnv *env = 0;
    jclass mainClass = NULL;
    jclass appClass = NULL; // actual application class being launched
    jmethodID mainID;
    jobjectArray mainArgs;
    int ret = 0;
    jlong start, end;

    RegisterThread();

    /* Initialize the virtual machine */
    start = CounterGet();
    if (!InitializeJVM(&vm, &env, &ifn)) {
        JLI_ReportErrorMessage(JVM_ERROR1);
        exit(1);
    }

    if (showSettings != NULL) {
        ShowSettings(env, showSettings);
        CHECK_EXCEPTION_LEAVE(1);
    }

    if (printVersion || showVersion) {
        PrintJavaVersion(env, showVersion);
        CHECK_EXCEPTION_LEAVE(0);
        if (printVersion) {
            LEAVE();
        }
    }

    /* If the user specified neither a class name nor a JAR file */
    if (printXUsage || printUsage || what == 0 || mode == LM_UNKNOWN) {
        PrintUsage(env, printXUsage);
        CHECK_EXCEPTION_LEAVE(1);
        LEAVE();
    }

    FreeKnownVMs();  /* after last possible PrintUsage() */

    if (JLI_IsTraceLauncher()) {
        end = CounterGet();
        JLI_TraceLauncher("%ld micro seconds to InitializeJVM\n",
               (long)(jint)Counter2Micros(end-start));
    }

    /* At this stage, argc/argv have the application's arguments */
    if (JLI_IsTraceLauncher()){
        int i;
        printf("%s is '%s'\n", launchModeNames[mode], what);
        printf("App's argc is %d\n", argc);
        for (i=0; i < argc; i++) {
            printf("    argv[%2d] = '%s'\n", i, argv[i]);
        }
    }

    ret = 1;

    /*
     * Get the application's main class.
     *
     * See bugid 5030265.  The Main-Class name has already been parsed
     * from the manifest, but not parsed properly for UTF-8 support.
     * Hence the code here ignores the value previously extracted and
     * uses the pre-existing code to reextract the value.  This is
     * possibly an end of release cycle expedient.  However, it has
     * also been discovered that passing some character sets through
     * the environment has "strange" behavior on some variants of
     * Windows.  Hence, maybe the manifest parsing code local to the
     * launcher should never be enhanced.
     *
     * Hence, future work should either:
     *     1)   Correct the local parsing code and verify that the
     *          Main-Class attribute gets properly passed through
     *          all environments,
     *     2)   Remove the vestages of maintaining main_class through
     *          the environment (and remove these comments).
     *
     * This method also correctly handles launching existing JavaFX
     * applications that may or may not have a Main-Class manifest entry.
     */
    mainClass = LoadMainClass(env, mode, what);
    CHECK_EXCEPTION_NULL_LEAVE(mainClass);
    /*
     * In some cases when launching an application that needs a helper, e.g., a
     * JavaFX application with no main method, the mainClass will not be the
     * applications own main class but rather a helper class. To keep things
     * consistent in the UI we need to track and report the application main class.
     */
    appClass = GetApplicationClass(env);
    NULL_CHECK_RETURN_VALUE(appClass, -1);
    /*
     * PostJVMInit uses the class name as the application name for GUI purposes,
     * for example, on OSX this sets the application name in the menu bar for
     * both SWT and JavaFX. So we'll pass the actual application class here
     * instead of mainClass as that may be a launcher or helper class instead
     * of the application class.
     */
    PostJVMInit(env, appClass, vm);
    /*
     * The LoadMainClass not only loads the main class, it will also ensure
     * that the main method's signature is correct, therefore further checking
     * is not required. The main method is invoked here so that extraneous java
     * stacks are not in the application stack trace.
     */
    mainID = (*env)->GetStaticMethodID(env, mainClass, "main",
                                       "([Ljava/lang/String;)V");
    CHECK_EXCEPTION_NULL_LEAVE(mainID);

    /* Build platform specific argument array */
    mainArgs = CreateApplicationArgs(env, argv, argc);
    CHECK_EXCEPTION_NULL_LEAVE(mainArgs);

    /* Invoke main method. */
    (*env)->CallStaticVoidMethod(env, mainClass, mainID, mainArgs);

    /*
     * The launcher's exit code (in the absence of calls to
     * System.exit) will be non-zero if main threw an exception.
     */
    ret = (*env)->ExceptionOccurred(env) == NULL ? 0 : 1;
    LEAVE();
}

```

CHECK_EXCEPTION_NULL_LEAVE() & CHECK_EXCEPTION_LEAVE() & LEAVE()
其中这些方法会调用下面代码
>逻辑是当java程序执行完，Launcher会调用DetachCurrentThread断开与主程序之间的连接。当成功断开与主程序之间的连接，Launcher就会等待程序中所有的非守护线程（non-daemon）全部执行结束，然后对jvm进行销毁，调用DestroyJavaVM

```c++
/*
 * Always detach the main thread so that it appears to have ended when
 * the application's main method exits.  This will invoke the
 * uncaught exception handler machinery if main threw an
 * exception.  An uncaught exception handler cannot change the
 * launcher's return code except by calling System.exit.
 *
 * Wait for all non-daemon threads to end, then destroy the VM.
 * This will actually create a trivial new Java waiter thread
 * named "DestroyJavaVM", but this will be seen as a different
 * thread from the one that executed main, even though they are
 * the same C thread.  This allows mainThread.join() and
 * mainThread.isAlive() to work as expected.
 */
#define LEAVE() \
    do { \
        if ((*vm)->DetachCurrentThread(vm) != JNI_OK) { \
            JLI_ReportErrorMessage(JVM_ERROR2); \
            ret = 1; \
        } \
        if (JNI_TRUE) { \
            (*vm)->DestroyJavaVM(vm); \
            return ret; \
        } \
    } while (JNI_FALSE)

#define CHECK_EXCEPTION_NULL_LEAVE(CENL_exception) \
    do { \
        if ((*env)->ExceptionOccurred(env)) { \
            JLI_ReportExceptionDescription(env); \
            LEAVE(); \
        } \
        if ((CENL_exception) == NULL) { \
            JLI_ReportErrorMessage(JNI_ERROR); \
            LEAVE(); \
        } \
    } while (JNI_FALSE)

#define CHECK_EXCEPTION_LEAVE(CEL_return_value) \
    do { \
        if ((*env)->ExceptionOccurred(env)) { \
            JLI_ReportExceptionDescription(env); \
            ret = (CEL_return_value); \
            LEAVE(); \
        } \
    } while (JNI_FALSE)
```

6. InitializeJVM函数

初始化java虚拟机
这里需要看下InvocationFunctions ifn函数的结构，然后就比较好理解了。之前对函数的赋值，然后创建虚拟机时调用。

```c++
typedef struct {
    CreateJavaVM_t CreateJavaVM;
    GetDefaultJavaVMInitArgs_t GetDefaultJavaVMInitArgs;
    GetCreatedJavaVMs_t GetCreatedJavaVMs;
} InvocationFunctions;
```
见InitializeJVM函数主代码
```c++
/*
 * Initializes the Java Virtual Machine. Also frees options array when
 * finished.
 */
static jboolean
InitializeJVM(JavaVM **pvm, JNIEnv **penv, InvocationFunctions *ifn)
{
    JavaVMInitArgs args;
    jint r;

    memset(&args, 0, sizeof(args));
    args.version  = JNI_VERSION_1_2;
    args.nOptions = numOptions;
    args.options  = options;
    args.ignoreUnrecognized = JNI_FALSE;

    if (JLI_IsTraceLauncher()) {
        int i = 0;
        printf("JavaVM args:\n    ");
        printf("version 0x%08lx, ", (long)args.version);
        printf("ignoreUnrecognized is %s, ",
               args.ignoreUnrecognized ? "JNI_TRUE" : "JNI_FALSE");
        printf("nOptions is %ld\n", (long)args.nOptions);
        for (i = 0; i < numOptions; i++)
            printf("    option[%2d] = '%s'\n",
                   i, args.options[i].optionString);
    }
	# 调用InvocationFunctions函数创建java虚拟机
    r = ifn->CreateJavaVM(pvm, (void **)penv, &args);
    JLI_MemFree(options);
    return r == JNI_OK;
}
```

7. LoadMainClass函数

获取java程序启动类和方法，然后调用CallStaticObjectMethod方法，该函数会调用到jni_invoke_static方法

```c++
/*
 * Loads a class and verifies that the main class is present and it is ok to
 * call it for more details refer to the java implementation.
 */
static jclass
LoadMainClass(JNIEnv *env, int mode, char *name)
{
    jmethodID mid;
    jstring str;
    jobject result;
    jlong start, end;
    jclass cls = GetLauncherHelperClass(env);
    NULL_CHECK0(cls);
    if (JLI_IsTraceLauncher()) {
        start = CounterGet();
    }
    NULL_CHECK0(mid = (*env)->GetStaticMethodID(env, cls,
                "checkAndLoadMain",
                "(ZILjava/lang/String;)Ljava/lang/Class;"));

    str = NewPlatformString(env, name);
    result = (*env)->CallStaticObjectMethod(env, cls, mid, USE_STDERR, mode, str);

    if (JLI_IsTraceLauncher()) {
        end   = CounterGet();
        printf("%ld micro seconds to load main class\n",
               (long)(jint)Counter2Micros(end-start));
        printf("----%s----\n", JLDEBUG_ENV_ENTRY);
    }

    return (jclass)result;
}
```

8. jni.cpp

这里会调用JavaCalls::call，会调用到java主类。

```c++
static void jni_invoke_static(JNIEnv *env, JavaValue* result, jobject receiver, JNICallType call_type, jmethodID method_id, JNI_ArgumentPusher *args, TRAPS) {
  methodHandle method(THREAD, Method::resolve_jmethod_id(method_id));

  // Create object to hold arguments for the JavaCall, and associate it with
  // the jni parser
  ResourceMark rm(THREAD);
  int number_of_parameters = method->size_of_parameters();
  JavaCallArguments java_args(number_of_parameters);
  args->set_java_argument_object(&java_args);

  assert(method->is_static(), "method should be static");

  // Fill out JavaCallArguments object
  args->iterate( Fingerprinter(method).fingerprint() );
  // Initialize result type
  result->set_type(args->get_ret_type());

  // Invoke the method. Result is returned as oop.
  JavaCalls::call(result, method, &java_args, CHECK);

  // Convert result
  if (result->get_type() == T_OBJECT || result->get_type() == T_ARRAY) {
    result->set_jobject(JNIHandles::make_local(env, (oop) result->get_jobject()));
  }
}

```
具体细节不用多扣，掌握流程和算法即可。