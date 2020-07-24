##### 基于Android 9.0代码进行剖析，细解init进程启动的各个环节内容

涉及到的源码文件

```

system/core/init/init.cpp

system/core/init/init_first_stage.cpp

system/core/ueventd.cpp

system/core/init/selinux.cpp

system/core/init/sigchld_handler.cpp

system/core/init/property_service.cpp

```

本文主要的流程如下，init.rc解析及系统核心服务的启动请参考：[《Android启动流程之二init.rc解析》](https://www.jianshu.com/p/3f54d7cd80b9
) 和《Android启动流程之三 系统核心服务的启动》

![大致流程图](https://upload-images.jianshu.io/upload_images/2484366-0201d6060087da3a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



# 一、入口

init进程是Linux系统中用户空间的第一个进程，PID=1，内核启动后会启动init进程并调用main方法作为入口，执行init进程应该要履行的职责

- 挂载文件

- 解析系统配置的rc文件，执行rc文件中的配置项

- 处理子进程的终止信号

- 提供属性服务

- 启动zygote进程

### 1.1 init主入口，main函数

源码路径

```

system/core/init/init.cpp

```

哇，这个main函数有点长,200多行😯

```

int main(int argc, char** argv) {

    //如果文件名是ueventd，执行ueventd守护进程的主函数

    if (!strcmp(basename(argv[0]), "ueventd")) {

        return ueventd_main(argc, argv);

    }

     //如果文件名是ueventd，执行watchdogd守护进程的主函数

    if (!strcmp(basename(argv[0]), "watchdogd")) {

        return watchdogd_main(argc, argv);

    }

   //如果文件名是ueventd，执行subcontext守护进程的主函数

  if (argc > 1 && !strcmp(argv[1], "subcontext")) {

        InitKernelLogging(argv);

        const BuiltinFunctionMap function_map;

        return SubcontextMain(argc, argv, &function_map);

    }

    if (REBOOT_BOOTLOADER_ON_PANIC) {

        InstallRebootSignalHandlers();

    }

   //通过读取INIT_SECOND_STAGE来判断是否是系统启动的第一阶段

    bool is_first_stage = (getenv("INIT_SECOND_STAGE") == nullptr);

    if (is_first_stage) {

        //获取当前启动时间

        boot_clock::time_point start_time = boot_clock::now();

        // Clear the umask. 

       //清除umask的设置，保证新创建的文件访问权限不受umask的影响

        // umask是Linux中关于文件访问权限的操作命令，详情可参考： https://www.cnblogs.com/sench/p/8933638.html

        umask(0);

        clearenv();

        //配置环境变量

        setenv("PATH", _PATH_DEFPATH, 1);

        // Get the basic filesystem setup we need put together in the initramdisk

        // on / and then we'll let the rc file figure out the rest.

        //挂载tmpfs文件系统，关于tmpfs可参考：https://blog.csdn.net/java211/article/details/52295348/

        mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");

       //挂载pts伪终端，pts介绍可以参考：http://blog.chinaunix.net/uid-22746363-id-384008.html

        mkdir("/dev/pts", 0755);

        //挂载socket文件系统，Android核心socket存放在这里

        mkdir("/dev/socket", 0755);

        mount("devpts", "/dev/pts", "devpts", 0, NULL);

        #define MAKE_STR(x) __STRING(x)

        //挂载proc文件系统，proc是linux中虚拟文件系统，可以作为Linux的控制中心，通过修改一些文件来改变

       //内核运行状态，里面包括很多linux核心工具，例如cmdline等，可参考：https://www.cnblogs.com/zydev/p/8728992.html

        mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC));

        // Don't expose the raw commandline to unprivileged processes.

        chmod("/proc/cmdline", 0440);

        gid_t groups[] = { AID_READPROC };

        setgroups(arraysize(groups), groups);

        //sysfs文件系统是一个类 似于proc文件系统的特殊文件系统，用于将系统中的设备组织成层次结构，

        //并向用户模式程序提供详细的内核数据结构信息,可参考：https://blog.csdn.net/skyflying2012/article/details/11783847

        mount("sysfs", "/sys", "sysfs", 0, NULL);

        //挂载Selinux

        mount("selinuxfs", "/sys/fs/selinux", "selinuxfs", 0, NULL);

        //mknod可参考：https://blog.csdn.net/wsclinux/article/details/50907567

        mknod("/dev/kmsg", S_IFCHR | 0600, makedev(1, 11));

        if constexpr (WORLD_WRITABLE_KMSG) {

            mknod("/dev/kmsg_debug", S_IFCHR | 0622, makedev(1, 11));

        }

        mknod("/dev/random", S_IFCHR | 0666, makedev(1, 8));

        mknod("/dev/urandom", S_IFCHR | 0666, makedev(1, 9));

        // Mount staging areas for devices managed by vold

        // See storage config details at http://source.android.com/devices/storage/

        mount("tmpfs", "/mnt", "tmpfs", MS_NOEXEC | MS_NOSUID | MS_NODEV,

              "mode=0755,uid=0,gid=1000");

        // /mnt/vendor is used to mount vendor-specific partitions that can not be

        // part of the vendor partition, e.g. because they are mounted read-write.

        mkdir("/mnt/vendor", 0755);

        // Now that tmpfs is mounted on /dev and we have /dev/kmsg, we can actually

        // talk to the outside world...

        //init内核log，为什么要使用kernel的log系统，因为此时Android系统的log还没有启动，所以需要使用kernel的log系统。

        InitKernelLogging(argv);

        LOG(INFO) << "init first stage started!";

        //真正开始启动第一次初始化

        //在设备树中挂载fstab指定的分区,参考：init_first_stage.cpp

        if (!DoFirstStageMount()) {

            LOG(FATAL) << "Failed to mount required partitions early ...";

        }

        //设置AVB信息到recovery中，参考：init_first_stage.cpp中的：setenv("INIT_AVB_VERSION", avb_handle->avb_version().c_str(), 1);

        SetInitAvbVersionInRecovery();

        // Enable seccomp if global boot option was passed (otherwise it is enabled in zygote).

        global_seccomp();

        // Set up SELinux, loading the SELinux policy.

        //初始化SeLinux，参考 1.2节

        SelinuxSetupKernelLogging();

        SelinuxInitialize();

        // We're in the kernel domain, so re-exec init to transition to the init domain now

        // that the SELinux policy has been loaded.

        //使用加载好的Selinux安全策略，重新设置init文件的安全权限

        if (selinux_android_restorecon("/init", 0) == -1) {

            PLOG(FATAL) << "restorecon failed of /init failed";

        }

        //设置可以进入第二阶段初始化的标记

        setenv("INIT_SECOND_STAGE", "true", 1);

        //记录第一阶段花费的时间

        static constexpr uint32_t kNanosecondsPerMillisecond = 1e6;

        uint64_t start_ms = start_time.time_since_epoch().count() / kNanosecondsPerMillisecond;

        setenv("INIT_STARTED_AT", std::to_string(start_ms).c_str(), 1);

        char* path = argv[0];

        char* args[] = { path, nullptr };

        //重新启动init进入main函数执行第二阶段

        execv(path, args);

        // execv() only returns if an error happened, in which case we

        // panic and never fall through this conditional.

        PLOG(FATAL) << "execv(\"" << path << "\") failed";

    }

    //以上代码执行完成第一阶段的初始化，下面就会进入第二阶段的初始化

    // At this point we're in the second stage of init.

    InitKernelLogging(argv);

    LOG(INFO) << "init second stage started!";

    // Set up a session keyring that all processes will have access to. It

    // will hold things like FBE encryption keys. No process should override

    // its session keyring.

    keyctl_get_keyring_ID(KEY_SPEC_SESSION_KEYRING, 1);

    // Indicate that booting is in progress to background fw loaders, etc.

    // 在/dev目录下创建一个隐藏的.booting文件表示正在进行初始化，初始化完成后会把这个文件删除

    close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));

    //初始化Android系统属性，这个函数主要是创建一个共享区域来存储属性值,参考1.3节

    property_init();

    // If arguments are passed both on the command line and in DT,

    // properties set in DT always have priority over the command-line ones.

    //DT的优先级高于cmdline，因此需要先执行DT的初始化解析，两者详细请参考 1.4节

    process_kernel_dt();

    process_kernel_cmdline();

    // Propagate the kernel variables to internal variables

    // used by init as well as the current required properties.

    //如官方注释，这里会读取内核的属性，参考 1.5 节

    export_kernel_boot_props();

    // Make the time that init started available for bootstat to log.

    property_set("ro.boottime.init", getenv("INIT_STARTED_AT"));

    property_set("ro.boottime.init.selinux", getenv("INIT_SELINUX_TOOK"));

    // Set libavb version for Framework-only OTA match in Treble build.

    const char* avb_version = getenv("INIT_AVB_VERSION");

    if (avb_version) property_set("ro.boot.avb_version", avb_version);

    // Clean up our environment.

    unsetenv("INIT_SECOND_STAGE");

    unsetenv("INIT_STARTED_AT");

    unsetenv("INIT_SELINUX_TOOK");

    unsetenv("INIT_AVB_VERSION");

    // Selinux第二阶段的配置

    SelinuxSetupKernelLogging();

    SelabelInitialize();

    SelinuxRestoreContext();

    // 初始化epoll

    //epoll初始化为分为两个阶段，我们从函数命名就可以看得出来，

    // 第一阶段在init.cpp的main方法里面signal_handler_init函数调用之前，这里如果执行失败的话就直接退出了

    //第二阶段就在signal_handler_init初始化系统信号处理器时会执行

    epoll_fd = epoll_create1(EPOLL_CLOEXEC);

    if (epoll_fd == -1) {

        PLOG(FATAL) << "epoll_create1 failed";

    }

    //初始化子进程退出信号处理 代码：system/core/init/sigchld_handler.cpp，请参考 2.1 节

    sigchld_handler_init();

    if (!IsRebootCapable()) {

        // If init does not have the CAP_SYS_BOOT capability, it is running in a container.

        // In that case, receiving SIGTERM will cause the system to shut down.

        InstallSigtermHandler();

    }

    //加载系统xxx.prop:system/core/init/property_service.cpp

    property_load_boot_defaults();

    export_oem_lock_status();

    //start_property_service()函数创建了socket，然后监听，并且调用register_epoll_handler()函数把socket的fd放入epoll中

    start_property_service();

    set_usb_controller();

    const BuiltinFunctionMap function_map;

    Action::set_function_map(&function_map);

    subcontexts = InitializeSubcontexts();

    ActionManager& am = ActionManager::GetInstance();

    ServiceList& sm = ServiceList::GetInstance();

    //加载init.rc,参考 《Android启动流程之init.rc解析》

    LoadBootScripts(am, sm);

    // Turning this on and letting the INFO logging be discarded adds 0.2s to

    // Nexus 9 boot time, so it's disabled by default.

    if (false) DumpState();

    //以下代码把init.rc中的action添加到执行队列中

    //将early-init的Action添加到action_queue中并执行触发器

    am.QueueEventTrigger("early-init");

    // Queue an action that waits for coldboot done so we know ueventd has set up all of /dev...

    //等待冷插拔设备初始化完成

    am.QueueBuiltinAction(wait_for_coldboot_done_action, "wait_for_coldboot_done");

    // ... so that we can start queuing up actions that require stuff from /dev.

    am.QueueBuiltinAction(MixHwrngIntoLinuxRngAction, "MixHwrngIntoLinuxRng");

    am.QueueBuiltinAction(SetMmapRndBitsAction, "SetMmapRndBits");

    am.QueueBuiltinAction(SetKptrRestrictAction, "SetKptrRestrict");

    //设备组合键的初始化操作

    am.QueueBuiltinAction(keychord_init_action, "keychord_init");

    am.QueueBuiltinAction(console_init_action, "console_init");

    // Trigger all the boot actions to get us started.

    am.QueueEventTrigger("init");

    // Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random

    // wasn't ready immediately after wait_for_coldboot_done

    am.QueueBuiltinAction(MixHwrngIntoLinuxRngAction, "MixHwrngIntoLinuxRng");

    // Don't mount filesystems or start core system services in charger mode.

    std::string bootmode = GetProperty("ro.bootmode", "");

    //如果是充电模式，不要挂载文件系统和启动核心系统服务

    if (bootmode == "charger") {

        am.QueueEventTrigger("charger");

    } else {

        am.QueueEventTrigger("late-init");

    }

    // Run all property triggers based on current state of the properties.

    am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");

    //循环等待事件的发生 

    while (true) {

        // By default, sleep until something happens.

        int epoll_timeout_ms = -1;

        if (do_shutdown && !shutting_down) {

            do_shutdown = false;

            if (HandlePowerctlMessage(shutdown_command)) {

                shutting_down = true;

            }

        }

        if (!(waiting_for_prop || Service::is_exec_service_running())) {

            //依次执行action中有对应command的执行函数，参考1.6 节

            am.ExecuteOneCommand();

        }

        if (!(waiting_for_prop || Service::is_exec_service_running())) {

            if (!shutting_down) {

                 //如果有进程需要重启，执行重启操作,参考 1.7 节

                auto next_process_restart_time = RestartProcesses();

                // If there's a process that needs restarting, wake up in time for that.

                if (next_process_restart_time) {

                    epoll_timeout_ms = std::chrono::ceil<std::chrono::milliseconds>(

                                           *next_process_restart_time - boot_clock::now())

                                           .count();

                    if (epoll_timeout_ms < 0) epoll_timeout_ms = 0;

                }

            }

            // If there's more work to do, wake up again immediately.

            if (am.HasMoreCommands()) epoll_timeout_ms = 0;

        }

        epoll_event ev;

        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, epoll_timeout_ms));

        if (nr == -1) {

            PLOG(ERROR) << "epoll_wait failed";

        } else if (nr == 1) {

            ((void (*)()) ev.data.ptr)();

        }

    }

    return 0;

}

```

init初始化分为两个阶段，第一个阶段主要是

- 挂载各种系统核心文件系统

- 初始化内核log

- 加载系统核心属性

- 加载SeLinux安全策略

- 加载完成Selinux安全策略后重新进入init main函数执行第二阶段的初始化

### 1.2 初始化SeLinux

关于selinux的背景和详细信息我们不在这里过多描述，可以参考这里 https://blog.csdn.net/qq_19923217/article/details/81240027

或者是Google官方文档：https://source.android.com/security/selinux/images/SELinux_Treble.pdf

SelinuxSetupKernelLogging和SelinuxIntialize函数

- SelinuxSetupKernelLogging： 

```

system/core/init/selinux.cpp

// This function sets up SELinux logging to be written to kmsg, to match init's logging.

void SelinuxSetupKernelLogging() {

    selinux_callback cb;

    //selinux 打印log所需要用到的callback

    cb.func_log = selinux_klog_callback;

    selinux_set_callback(SELINUX_CB_LOG, cb);

}

```

SelinuxSetupKernelLogging主要是初始化和设置SeLinux需要打印log时用到的callback，log信息会被写入kmsg

- SelinuxIntialize 

```

void SelinuxInitialize() {

    //主要用于记录SeLinux初始化的耗时

    Timer t;

    LOG(INFO) << "Loading SELinux policy";

    //价值SeLinux策略

    if (!LoadPolicy()) {

        LOG(FATAL) << "Unable to load SELinux policy";

    }

    //Selinux工作模式包括：

    //1、permissive,这个模式下所有的操作都被允许，但是如果违反权限规则的话会被记录日志

    //2、enforcing，所有操作都会进行权限检查

    //以下代码主要是设置SeLinux工作模式，我们可以使用adb shell getenforce 来获取当前处于哪种模式

    bool kernel_enforcing = (security_getenforce() == 1);

    bool is_enforcing = IsEnforcing();

    if (kernel_enforcing != is_enforcing) {

        if (security_setenforce(is_enforcing)) {

            PLOG(FATAL) << "security_setenforce(%s) failed" << (is_enforcing ? "true" : "false");

        }

    }

    if (auto result = WriteFile("/sys/fs/selinux/checkreqprot", "0"); !result) {

        LOG(FATAL) << "Unable to write to /sys/fs/selinux/checkreqprot: " << result.error();

    }

    // init's first stage can't set properties, so pass the time to the second stage.

    //init的第一阶段无法写入属性，因此先记录当前耗时，然后传递给第二阶段

    setenv("INIT_SELINUX_TOOK", std::to_string(t.duration().count()).c_str(), 1);

}

```

接下来我们看一下LoadPolicy函数，这会真正的去价值SeLinux安全策略

LoadPolicy会把安全策略传递给kernel，这样kernel就有了安全策略的配置信息了,这里我们不做过多解释，SeLinux

加载流程不在本文范围内，后面可以通过其他专题来讨论

```

bool LoadPolicy() {

    return IsSplitPolicyDevice() ? LoadSplitPolicy() : LoadMonolithicPolicy();

}

```

### 1.3 property_init

读取系统属性配置到共享区域。

```

system/core/init/property_service.cpp

void property_init() {

    // 创建dev/_properties_存放系统属性

    mkdir("/dev/__properties__", S_IRWXU | S_IXGRP | S_IXOTH);

    //解析系统属性

    CreateSerializedPropertyInfo();

    if (__system_property_area_init()) {

        LOG(FATAL) << "Failed to initialize property area";

    }

    if (!property_info_area.LoadDefaultPath()) {

        LOG(FATAL) << "Failed to load serialized property info file";

    }

}

void CreateSerializedPropertyInfo() {

    auto property_infos = std::vector<PropertyInfoEntry>();

    if (access("/system/etc/selinux/plat_property_contexts", R_OK) != -1) {

        if (!LoadPropertyInfoFromFile("/system/etc/selinux/plat_property_contexts",

                                      &property_infos)) {

            return;

        }

        // Don't check for failure here, so we always have a sane list of properties.

        // E.g. In case of recovery, the vendor partition will not have mounted and we

        // still need the system / platform properties to function.

        if (!LoadPropertyInfoFromFile("/vendor/etc/selinux/vendor_property_contexts",

                                      &property_infos)) {

            // Fallback to nonplat_* if vendor_* doesn't exist.

            LoadPropertyInfoFromFile("/vendor/etc/selinux/nonplat_property_contexts",

                                     &property_infos);

        }

    } else {

        if (!LoadPropertyInfoFromFile("/plat_property_contexts", &property_infos)) {

            return;

        }

        if (!LoadPropertyInfoFromFile("/vendor_property_contexts", &property_infos)) {

            // Fallback to nonplat_* if vendor_* doesn't exist.

            LoadPropertyInfoFromFile("/nonplat_property_contexts", &property_infos);

        }

    }

    auto serialized_contexts = std::string();

    auto error = std::string();

    if (!BuildTrie(property_infos, "u:object_r:default_prop:s0", "string", &serialized_contexts,

                   &error)) {

        LOG(ERROR) << "Unable to serialize property contexts: " << error;

        return;

    }

    constexpr static const char kPropertyInfosPath[] = "/dev/__properties__/property_info";

    if (!WriteStringToFile(serialized_contexts, kPropertyInfosPath, 0444, 0, 0, false)) {

        PLOG(ERROR) << "Unable to write serialized property infos to file";

    }

    selinux_android_restorecon(kPropertyInfosPath, 0);

}

```

### 1.4 process dt 和cmdline

process_kernel_dt

该方法主要是打开dt目录下的素有文件并读取到系统属性中，文件定义在/proc/device-tree/firmware/android/目录下，

先判断compatible文件中的内容是否是android,firmware,如果不是就直接退出了。然后在循环读取目录下的文件，把文件

名作为属性名，文件内容作为属性值进行存储

```

system/core/init/init.cpp

static void process_kernel_dt() {

    //如果compatible的文件内容不是android,firmware，直接返回

    if (!is_android_dt_value_expected("compatible", "android,firmware")) {

        return;

    }

    //获取并打开dt文件目录，默认是定义在system/core/init/util.cpp文件中的

    //const std::string kDefaultAndroidDtDir("/proc/device-tree/firmware/android/")

    std::unique_ptr<DIR, int (*)(DIR*)> dir(opendir(get_android_dt_dir().c_str()), closedir);

    if (!dir) return;

    std::string dt_file;

    struct dirent *dp;

    //循环读取dt目录中的文件

    while ((dp = readdir(dir.get())) != NULL) {

        if (dp->d_type != DT_REG || !strcmp(dp->d_name, "compatible") || !strcmp(dp->d_name, "name")) {

            continue;

        }

        std::string file_name = get_android_dt_dir() + dp->d_name;

        android::base::ReadFileToString(file_name, &dt_file);

        std::replace(dt_file.begin(), dt_file.end(), ',', '.');

        //文件名作为属性名，文件内容作为属性值

        property_set("ro.boot."s + dp->d_name, dt_file);

    }

}

```

process_kernel_cmdline函数

该方法会执行两次import_kernel_cmdline函数，

第一遍主要是判断是否在qemu中，第一遍第一个参数传递false，表示不是qemu

第二遍才是真正的读取proc/cmdline文件中的内容并转换为内核属性，第二遍第一个参数传递true，表示已经是qemu，并

传递import_kernel_nv函数指针读取系统属性

```

static void process_kernel_cmdline() {

    // The first pass does the common stuff, and finds if we are in qemu.

    // The second pass is only necessary for qemu to export all kernel params

    // as properties.

    import_kernel_cmdline(false, import_kernel_nv);

    if (qemu[0]) import_kernel_cmdline(true, import_kernel_nv);

}

```

import_kernel_cmdline函数

```

system/core/init/util.cpp

void import_kernel_cmdline(bool in_qemu,

                           const std::function<void(const std::string&, const std::string&, bool)>& fn) {

    std::string cmdline;

    //读取proc/cmdline中的内容到cmdline字符串中

    android::base::ReadFileToString("/proc/cmdline", &cmdline);

    //循环读取属性，读取方式为，将字符串cmdline的内容按空格分隔成字符串数组，并循环遍历

    //这个数组，然后调用fn设置属性，这里fn就是在process_kernel_cmdline方法中传递进来的import_kernel_nv

    for (const auto& entry : android::base::Split(android::base::Trim(cmdline), " ")) {

        std::vector<std::string> pieces = android::base::Split(entry, "=");

        if (pieces.size() == 2) {

            fn(pieces[0], pieces[1], in_qemu);

        }

    }

}

```

import_kernel_nv函数

```

system/core/init/init.cpp

static void import_kernel_nv(const std::string& key, const std::string& value, bool for_emulator) {

    if (key.empty()) return;

    //import_kernel_cmdline第一次执行的时候传递过来的for_emulator为false，第二次为true

    //因此这里在第一遍调用的时候不会执行

    if (for_emulator) {

        // In the emulator, export any kernel option with the "ro.kernel." prefix.

        property_set("ro.kernel." + key, value);

        return;

    }

    if (key == "qemu") {

        strlcpy(qemu, value.c_str(), sizeof(qemu));

    } else if (android::base::StartsWith(key, "androidboot.")) {

        property_set("ro.boot." + key.substr(12), value);

    }

}

```

### 1.5 export_kernel_boot_props

```

static void export_kernel_boot_props() {

    struct {

        const char *src_prop;

        const char *dst_prop;

        const char *default_value;

    } prop_map[] = {

        { "ro.boot.serialno", "ro.serialno", "", },

        { "ro.boot.mode", "ro.bootmode", "unknown", },

        { "ro.boot.baseband", "ro.baseband", "unknown", },

        { "ro.boot.bootloader", "ro.bootloader", "unknown", },

        { "ro.boot.hardware", "ro.hardware", "unknown", },

        { "ro.boot.revision", "ro.revision", "0", },

    };

    // unknown作为默认值，如果没有写入指定的value的话就用unknown作为value

    for (size_t i = 0; i < arraysize(prop_map); i++) {

        std::string value = GetProperty(prop_map[i].src_prop, "");

        property_set(prop_map[i].dst_prop, (!value.empty()) ? value : prop_map[i].default_value);

    }

}

```

### 1.6  ExecuteOneCommand

```

void ActionManager::ExecuteOneCommand() {

    // Loop through the event queue until we have an action to execute

    while (current_executing_actions_.empty() && !event_queue_.empty()) {

        for (const auto& action : actions_) {

            if (std::visit([&action](const auto& event) { return action->CheckEvent(event); },

                           event_queue_.front())) {

                current_executing_actions_.emplace(action.get());

            }

        }

        event_queue_.pop();

    }

    //current_executing_actions_ 为空则退出

    if (current_executing_actions_.empty()) {

        return;

    }

    //获取current_executin_actions_ 头部

    auto action = current_executing_actions_.front();

    if (current_command_ == 0) {

        std::string trigger_name = action->BuildTriggersString();

        LOG(INFO) << "processing action (" << trigger_name << ") from (" << action->filename()

                  << ":" << action->line() << ")";

    }

    //system/core/init/action.cpp

    action->ExecuteOneCommand(current_command_);

    // If this was the last command in the current action, then remove

    // the action from the executing list.

    // If this action was oneshot, then also remove it from actions_.

   //如果这是最后一个action或者是一个oneshot的action，执行完成后直接移除队列

    ++current_command_;

    if (current_command_ == action->NumCommands()) {

        current_executing_actions_.pop();

        current_command_ = 0;

        if (action->oneshot()) {

            auto eraser = [&action](std::unique_ptr<Action>& a) { return a.get() == action; };

            actions_.erase(std::remove_if(actions_.begin(), actions_.end(), eraser));

        }

    }

}

```

```

void Action::ExecuteOneCommand(std::size_t command) const {

    // We need a copy here since some Command execution may result in

    // changing commands_ vector by importing .rc files through parser

    //为了避免因为解析.rc文件导致command_队列被改变，所以要先拷贝一份

    Command cmd = commands_[command];

    //执行command

    ExecuteCommand(cmd);

}

void Action::ExecuteCommand(const Command& command) const {

    android::base::Timer t;

   //调用Command的invokeFunc函数执行

    auto result = command.InvokeFunc(subcontext_);

    auto duration = t.duration();

    // There are many legacy paths in rootdir/init.rc that will virtually never exist on a new

    // device, such as '/sys/class/leds/jogball-backlight/brightness'. As of this writing, there

    // are 198 such failures on bullhead. Instead of spamming the log reporting them, we do not

    // report such failures unless we're running at the DEBUG log level.

    bool report_failure = !result.has_value();

    if (report_failure && android::base::GetMinimumLogSeverity() > android::base::DEBUG &&

        result.error_errno() == ENOENT) {

        report_failure = false;

    }

    // Any action longer than 50ms will be warned to user as slow operation

    if (report_failure || duration > 50ms ||

        android::base::GetMinimumLogSeverity() <= android::base::DEBUG) {

        std::string trigger_name = BuildTriggersString();

        std::string cmd_str = command.BuildCommandString();

        LOG(INFO) << "Command '" << cmd_str << "' action=" << trigger_name << " (" << filename_

                  << ":" << command.line() << ") took " << duration.count() << "ms and "

                  << (result ? "succeeded" : "failed: " + result.error_string());

    }

}

```

### 1.7 重启进程

```

static std::optional<boot_clock::time_point> RestartProcesses() {

    std::optional<boot_clock::time_point> next_process_restart_time;

    //遍历service列表进行重启

    for (const auto& s : ServiceList::GetInstance()) {

        //如果flag不是SVC_RESTARTING,取下一个

        if (!(s->flags() & SVC_RESTARTING)) continue;

        auto restart_time = s->time_started() + 5s;

        if (boot_clock::now() > restart_time) {

            //启动进程

            if (auto result = s->Start(); !result) {

                LOG(ERROR) << "Could not restart process '" << s->name() << "': " << result.error();

            }

        } else {

            if (!next_process_restart_time || restart_time < *next_process_restart_time) {

                next_process_restart_time = restart_time;

            }

        }

    }

    return next_process_restart_time;

}

```

# 二、信号处理

sigchld_handler_init主要是子进程死掉后，父进程中会接收到一个SIGCHLD信号，init进程接收到这个信号后会将已经死了的子进程移除，避免成为

僵尸进程占用系统程序表的空间。

### 2.1  sigchld_handler_init

```

system/core/init/sigchld_handler.cpp

void sigchld_handler_init() {

    // Create a signalling mechanism for SIGCHLD.

    int s[2];

    //创建用于接收和写入信号的两个socket

    if (socketpair(AF_UNIX, SOCK_STREAM | SOCK_NONBLOCK | SOCK_CLOEXEC, 0, s) == -1) {

        PLOG(FATAL) << "socketpair failed in sigchld_handler_init";

    }

    signal_write_fd = s[0];

    signal_read_fd = s[1];

    // Write to signal_write_fd if we catch SIGCHLD.

    //接收到SIGCHLD信号后就写入signal_write_fd

    struct sigaction act;

    memset(&act, 0, sizeof(act));

    //SIGCHLD_handler会被存放在sigaction数据结构里面，负责处理SIGCHLD信号

    act.sa_handler = SIGCHLD_handler;

    //用于标记只有在子进程死掉用才接收信息

    act.sa_flags = SA_NOCLDSTOP;

    //注册将需要监听的信号SIGCHLD及对应的处理器

    sigaction(SIGCHLD, &act, 0);

    //进入waitid处理子进程是否退出的情况

    ReapAnyOutstandingChildren();

    //注册回调用于监听信号，其中handle_signal是函数指针

    register_epoll_handler(signal_read_fd, handle_signal);

}

```

Linux系统中进程之间可以发送信息来实现进程间通讯，这里的这种机制可以叫做 信号，每个进程在处理信号时是需要被注册为处理者。

这里的两个函数中，SIGCHILD_handler负责写入信号，handle_signal负责读取信号。子进程在死掉后会产生SIGCHILD信号并写入signal_write_fd,信号通过

socket发送到父进程，父进程读取信号到signal_read_fd

```

//写入信号

static void SIGCHLD_handler(int) {

    //向signal_write_fd中写入1

    if (TEMP_FAILURE_RETRY(write(signal_write_fd, "1", 1)) == -1) {

        PLOG(ERROR) << "write(signal_write_fd) failed";

    }

}

//读取信号

static void handle_signal() {

    // Clear outstanding requests.

    char buf[32];

    read(signal_read_fd, buf, sizeof(buf));

    //这里只要是回收死掉的子进程

    ReapAnyOutstandingChildren();

}

```

### 2.2 ReapAnyOutstandingChildren函数

```

void ReapAnyOutstandingChildren() {

    while (ReapOneProcess()) {

    }

}

static bool ReapOneProcess() {

    siginfo_t siginfo = {};

    // This returns a zombie pid or informs us that there are no zombies left to be reaped.

    // It does NOT reap the pid; that is done below.

    if (TEMP_FAILURE_RETRY(waitid(P_ALL, 0, &siginfo, WEXITED | WNOHANG | WNOWAIT)) != 0) {

        PLOG(ERROR) << "waitid failed";

        return false;

    }

    auto pid = siginfo.si_pid;

    if (pid == 0) return false;

    // At this point we know we have a zombie pid, so we use this scopeguard to reap the pid

    // whenever the function returns from this point forward.

    // We do NOT want to reap the zombie earlier as in Service::Reap(), we kill(-pid, ...) and we

    // want the pid to remain valid throughout that (and potentially future) usages.

    auto reaper = make_scope_guard([pid] { TEMP_FAILURE_RETRY(waitpid(pid, nullptr, WNOHANG)); });

    std::string name;

    std::string wait_string;

    Service* service = nullptr;

    if (PropertyChildReap(pid)) {

        name = "Async property child";

    } else if (SubcontextChildReap(pid)) {

        name = "Subcontext";

    } else {

        //find service via pid

        service = ServiceList::GetInstance().FindService(pid, &Service::pid);

        if (service) {

            name = StringPrintf("Service '%s' (pid %d)", service->name().c_str(), pid);

            if (service->flags() & SVC_EXEC) {

                auto exec_duration = boot_clock::now() - service->time_started();

                auto exec_duration_ms =

                    std::chrono::duration_cast<std::chrono::milliseconds>(exec_duration).count();

                wait_string = StringPrintf(" waiting took %f seconds", exec_duration_ms / 1000.0f);

            }

        } else {

            name = StringPrintf("Untracked pid %d", pid);

        }

    }

    if (siginfo.si_code == CLD_EXITED) {

        LOG(INFO) << name << " exited with status " << siginfo.si_status << wait_string;

    } else {

        LOG(INFO) << name << " received signal " << siginfo.si_status << wait_string;

    }

    if (!service) return true;

    //回收子进程,参考system/core/init/service.cpp:: Reap

    service->Reap(siginfo);

    if (service->flags() & SVC_TEMPORARY) {

        ServiceList::GetInstance().RemoveService(*service);

    }

    return true;

}

```

service.cpp :: Reap

```

void Service::Reap(const siginfo_t& siginfo) {

    // flag为ONESHOT或者RESTART时直接kill掉进程组

    if (!(flags_ & SVC_ONESHOT) || (flags_ & SVC_RESTART)) {

        KillProcessGroup(SIGKILL);

    }

    // Remove any descriptor resources we may have created.

    std::for_each(descriptors_.begin(), descriptors_.end(),

                  std::bind(&DescriptorInfo::Clean, std::placeholders::_1));

    for (const auto& f : reap_callbacks_) {

        f(siginfo);

    }

    if (flags_ & SVC_EXEC) UnSetExec();

    if (flags_ & SVC_TEMPORARY) return;

    pid_ = 0;

    flags_ &= (~SVC_RUNNING);

    start_order_ = 0;

    // Oneshot processes go into the disabled state on exit,

    // except when manually restarted.

    if ((flags_ & SVC_ONESHOT) && !(flags_ & SVC_RESTART)) {

        flags_ |= SVC_DISABLED;

    }

    // Disabled and reset processes do not get restarted automatically.

    if (flags_ & (SVC_DISABLED | SVC_RESET)) {

        NotifyStateChange("stopped");

        return;

    }

    // If we crash > 4 times in 4 minutes, reboot into recovery.

    boot_clock::time_point now = boot_clock::now();

    if ((flags_ & SVC_CRITICAL) && !(flags_ & SVC_RESTART)) {

        if (now < time_crashed_ + 4min) {

            if (++crash_count_ > 4) {

                LOG(FATAL) << "critical process '" << name_ << "' exited 4 times in 4 minutes";

            }

        } else {

            time_crashed_ = now;

            crash_count_ = 1;

        }

    }

    flags_ &= (~SVC_RESTART);

    flags_ |= SVC_RESTARTING;

    // Execute all onrestart commands for this service.

    onrestart_.ExecuteAllCommands();

   //设置对应的service为restarting状态

    NotifyStateChange("restarting");

    return;

}

```

参考：

[https://www.jianshu.com/p/464c3d1203b1](https://www.jianshu.com/p/464c3d1203b1)

[http://gityuan.com/2016/02/05/android-init/](http://gityuan.com/2016/02/05/android-init/)
