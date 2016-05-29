---
layout:     post
title:      "Android系统启动流程"
subtitle:   "Android源码解析"
date:       2016-05-29
author:     "SaladJack"
header-img: "img/android-system-launch.jpg"
tags:
    - Android源码
---

# Android系统启动流程
* 当系统引导程序启动Linux内核，内核会加载各种数据结构，和驱动程序，加载完毕之后，Android系统开始启动并加载第一个用户级别的进程：init（system/core/init/Init.c）

* 查看Init.c代码，看main函数

		int main(int argc, char **argv)
		{
	   		 ...
			//执行Linux指令
		    mkdir("/dev", 0755);
		    mkdir("/proc", 0755);
		    mkdir("/sys", 0755);
	
	      	...
	    	//解析执行init.rc配置文件
	    	init_parse_config_file("/init.rc");
			...
		}

* 在system\core\rootdir\Init.rc中定义好的指令都会开始执行，其中执行了很多bin指令，启动系统服务

		//启动孵化器进程，此进程是Android系统启动关键服务的一个母进程
		service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    	socket zygote stream 666
   	 	onrestart write /sys/android_power/request_state wake
    	onrestart write /sys/power/state on
    	onrestart restart media
    	onrestart restart netd

* 在app_process文件夹下找到app_main.cpp，查看main函数，发现以下代码

		int main(int argc, const char* const argv[])
		{
	   		...
			//启动一个系统服务：ZygoteInit
	        runtime.start("com.android.internal.os.ZygoteInit",startSystemServer);
			...
		}

* 在ZygoteInit.java中，查看main方法

		 public static void main(String argv[]) {
			...
			//加载Android系统需要的类
			preloadClasses();
			...
			if (argv[1].equals("true")) {
				//调用方法启动一个系统服务
                startSystemServer();
            }
			...
		}

* startSystemServer()方法的方法体

		String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,3001,3002,3003",
            "--capabilities=130104352,130104352",
            "--runtime-init",
            "--nice-name=system_server",
            "com.android.server.SystemServer",
        };

		...
		//分叉启动上面字符串数组定义的服务
		 pid = Zygote.forkSystemServer(
         parsedArgs.uid, parsedArgs.gid,
         parsedArgs.gids, debugFlags, null,
         parsedArgs.permittedCapabilities,
         parsedArgs.effectiveCapabilities);
* SystemServer服务被启动

		public static void main(String[] args) {
			...
			//加载动态链接库
			 System.loadLibrary("android_servers");
        	//执行链接库里的init1方法
			init1(args);
			...
		}

* 动态链接库文件和java类包名相同，找到com\_android\_server\_SystemServer.cpp文件
* 在com\_android\_server\_SystemServer.cpp文件中，找到了

		static JNINativeMethod gMethods[] = {
		    /* name, signature, funcPtr */
			//给init1方法映射一个指针，调用system_init方法
		    { "init1", "([Ljava/lang/String;)V", (void*) android_server_SystemServer_init1 },
		};

* android\_server\_SystemServer\_init1方法体中调用了system\_init()，system\_init()没有方法体
* 在system\_init.cpp文件中找到system\_init()方法，方法体中
		//执行了SystemServer.java的init2方法
		runtime->callStatic("com/android/server/SystemServer", "init2");

* 回到SystemServer.java，在init2的方法体中

		//启动一个服务线程
		Thread thr = new ServerThread();
        thr.start();

* 在ServerThread的run方法中
		
		//准备消息轮询器
		Looper.prepare();
		...
		//启动大量的系统服务并把其逐一添加至ServiceManager
		ServiceManager.addService(Context.WINDOW_SERVICE, wm);
		...
		//调用systemReady，准备创建第一个activity
		 ((ActivityManagerService)ActivityManagerNative.getDefault())
                .systemReady(new Runnable(){
				...
		}）；

* 在ActivityManagerService.java中，有systemReady方法，方法体里找到

		//检测任务栈中有没有activity，如果没有，创建Launcher
		mMainStack.resumeTopActivityLocked(null);
* 在ActivityStack.java中，方法resumeTopActivityLocked

		// Find the first activity that is not finishing.
        ActivityRecord next = topRunningActivityLocked(null);
        ...
        if (next == null) {
            // There are no more activities!  Let's just start up the
            // Launcher...
            if (mMainStack) {
                return mService.startHomeActivityLocked();
            }
        }
		...

