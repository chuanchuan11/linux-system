> 守护进程  

    Daemon(精灵)进程是linux中的后台服务进程，通常独立于控制终端并且周期性地执行某种任务或等待处理某些发生的事件。一般采用d结尾的名字  
    
    linux后台的一些系统服务进程，没有控制终端，不能直接和用户交互。不受用户登录、注销的影响，一直在运行着，他们都是守护进程。如：ftp服务器；nfs服务器等  

  (1) 基本概念  
  
      a) 通常在系统启动时运行，仅在系统关闭时终止  
      b) 始终在后台运行，独立于任何终端  
      
   守护进程不能对终端输入和输出，但是不影响其他程序的运行；前台进程可以从终端输入，也可以终端输出；后台进程不可以终端输入，但可以终端输出  
    
    
  (2) 会话 - 进程组  
  
      a) Linux以会话[session]、进程组的方式管理进程  
      b) 多个进程在同一个组，第一个进程默认时进程组的组长      
      c) 会话是一个或多个进程组的集合，通常用户打开一个终端时，系统会创建一个shell会话。在终端运行的第一个shell进程称为会话组长，在该会话下运行的所有进程都属于这个会话，创建会话的时候，组长不可以创建必须是组员创建    
      d) 终端关闭时，所有相关进程会被结束  

  (3) 创建会话  
  
      创建一个会话需要注意以下5点注意事项： 
      
        a) 调用进程不能是进程组组长，该进程变成新会话首进程[session header]  
        b) 该进程成为一个新进程组的组长进程  
        c) 新会话丢弃原有的控制终端，该会话没有控制终端  
        d) 该调用进程是组长进程，则出错返回  
        e) 建立新会话时，先调用fork，父进程终止，子进程调用setsid  
```cpp
//获取进程所属的会话ID  
    pid_t getsid(pid_t pid);  

参数：  

    pid为0表示查看当前进程session ID  
    
返回值：

    成功：  返回调用进程的会话ID  
    失败：  返回-1，设置errno  

命令学习：

  ps ajx 查看系统中的进程
     - a 表示不仅列当前用户的进程，也列出所有其他用户的进程
     - x 表示不仅列有控制终端的进程，也列出所有无控制终端的进程  
     - j 表示列出与作业控制相关的信息  

```

  (4) 守护进程创建模型  
  
      让守护进程独立于终端，避免终端关闭，进程结束  
  
      a) 创建子进程，父进程退出  
          
          所有工作在子进程中进行形式上的脱离了控制终端  
      
      b) 子进程中创建新会话 - setsid()函数
      
          使子进程完全独立出来，脱离原先的控制终端，子进程成为新的会话组长    
      
      c) 改变当前目录为根目录 -  chdir() 函数
          
          守护进程一直在后台运行，其工作目录不能被卸载，应该重新设定当前的工作目录  
          [/和temp 目录对不同的用户读写权限不一样，且守护进程的工作目录应该放置在永远不需要进程卸载的目录]  
            
      d) 重设文件权限掩码 - umask()函数  
      
          一般不希望对守护进程创建的文件权限进行限制，因此将文件权限掩码设置为0，不会屏蔽任何权限位[防止守护进程创建到的文件权限被改变]  
          一般只对当前有影响，不影响其他的进程的权限设置  
          
      e) 关闭文件描述符  
      
          关闭所有从父进程继承的打开文件，因为不会再用到，占用浪费资源  
          已脱离终端, stdin/stdout/stderr 无法再使用  
          
      f) 开始执行守护进程核心工作  
      
      g) 守护进程退出处理程序模型  

示例：
```cpp
    #include <sys/stat.h>
    #include <fcntl.h>  
    #include <string.h>  
    #include <stdlib.h>  
    #include <signal.h>  
    #include <sys/time.h>  
    #include <time.h>  

    #define _FILE_NAME_FORMAT_ "%s/log/mydaemon.%ld"  
    
    void touchfile(int num)
    {
        char *HoneDir = getenv("HOME");
        char strFilename[256] = {0};
        
        sprintf(strFilename, _FILE_NAME_FORMAT_, HomeDir, time(NULL));
        
        int fd = open(strFilename, O_RDWR|O_CREAT, 0666);  //创建文件  
        
        if(fd < 0)
        {
            perror("open err");   
            exit(1);
        }
        close(fd);
    }
    
    int main()
    {
        //1. 创建子进程，父进程退出  
        pid_t pid = fork();
        
        //2. 当会长  
        setsid();
        
        //3. 设置掩码  
        umask(0);
        
        //4. 切换目录  
        chdir(getenv("HOME"));
        
        //5. 关闭文件描述符  
        close(1);
        close(2);
        close(3);
        
        //6. 执行核心逻辑  
        struct itimerval myit = {{60, 0}, {1, 0}};  
        setitimer(ITIMER_REAL, &myit, NULL);  
        
        struct sigaction act;
        act.sa_flags = 0;
        sigemptyset(&act.sa_mask);  
        act.sa_handler = touchfile;  
        
        sigaction(SIGALRM, &act, NULL);  
        while(1){
            //每隔1分钟在/home/itheima/log 下创建文件
            sleep(1);
        }
        
        return 0;
    }

扩展了解：
    通过nohup指令也可以达到守护进程创建的效果  
    nohup cmd [> 1.log] &
    - nohup 指令会让cmd收不到SIGHUP信号   
    - & 代表后台运行  

```

> 线程  

   (1) 线程概念

      a) 线程是轻量级的进程，一个进程内部可以有多个线程，默认情况下一个进程只有一个线程  
    
      b) 共享相同地址空间的多个任务      
      
   (2) 线程特点  
      
      避免了额外的TLB和cache的刷新，大大提高了任务切换的效率，解决进程再切换时系统开销大的问题  
      
      Cache是高速缓存，cpu首先是从cache中访问数据，不同进程访问时，若里边的数据不是自己的，则需要刷新数据，造成开销大  
      
![image](https://user-images.githubusercontent.com/42632290/139525466-73d1dcd9-51ab-4fa8-81e8-398144cda8f1.png)

      
   (3) 线程与进程的区别  
   
      a) 每个进程有自己独立的一块内存空间、一组资源系统，内部数据和状态完全独立[即：拥有自己的PCB]  
      
      b) 线程也有PCB，但没有独立的地址空间, 同一进程下的线程不仅共享进程资源和内存，每个线程还有属于自己的内存空间[栈空间]    
       
       即：独居[进程]，合租[线程]  
    
      c) 线程是操作系统最小的执行单位[即一个cpu情况下，含有多个线程的进程运行时，只有一个线程再运行]，进程是最小的系统资源分配单位 

**命令学习：**
     
     查看LWP号:
         
         ps -Lf pid 查看指定线程的lwp号，即查看一个进程下有几个线程  

  (4) 线程共享资源与私有资源  

      a) 线程资源共享  
         - 文件描述符  
         - 每种信号的处理方式   
         - 当前工作目录  
         - 用户ID和组ID  
         - 内存地址空间[.text/.data/.bss/heap/共享库]，即只有栈不共享  

      b) 线程非共享资源  
         - 线程ID  
         - 处理器现场和栈指针[内核栈]  
         - 独立的栈空间[用户栈空间]  
         - errno变量  
         - 信号屏蔽字  
         - 调度优先级  
         
![image](https://user-images.githubusercontent.com/42632290/139526148-792c706b-f94a-4381-9cb0-8c1786a3bbe2.png)

  (5) 线程优缺点  
      
      a) 优点：提高程序并发性，开销小，数据通信、共享数据方便  
      
      b) 缺点：库函数，不稳定，调试、编写困难，对信号支持不好  

      优点相对突出，缺点均不是硬伤，linux下由于实现方法导致进程，线程差别不是很大  

  (6) 线程中打印错误信息  
  
      线程中使用 char *strerror(int errnum) 函数打印错误信息  
      
      线程中一般不使用perror()打印错误信息  
      
      为什么线程中不使用perror()? 需要待查


  (7) 线程函数使用  
 
 //1. 线程创建
 
```cpp

    #include <pthread.h>  
        int pthread_creat(pthread_t *thread, const pthread_attr_t *attr, void *(*routine)(void *), void *arg);
        
参数：
    thread:   线程对象id，传出参数    
    attr:     线程属性，NULL代表默认属性  
    routine:  线程执行的函数，void *func(void *)
    arg:      传递给routine的参数  
    
返回值：
    成功： 返回0
    失败： 返回errno
 
示例：
    #include <unistd.h>
    #include <pthread.h>
    
    void* threadfunc(void *arg)
    {
        printf("i am a thread! pid = %d, tid=%lu \n", getpid(), pthread_self());
        return NULL;
    }
    
    int main()
    {
        pthread_t pid;
        pthread_creat(&tid, NULL, threadfunc, NULL);  
        printf("i am main thread, pid=%d, tid=%lu \n", getpid(), pthread_self());
        sleep(1);  //主控线程退出代表整个进程退出，其他线程自动退出，所以加延时，方便观察  
        return 0;
    } 
```
 
//2. 线程回收  

```cpp
    #include <pthread.h> 
        int pthread_join(pthread_t pthread, void **retval); //阻塞直到thread结束  

参数：
    thread:   要回收的线程对象  
    *retval： 接收线程thread的返回值  

返回值：
    成功： 返回0
    失败： 返回错误码

示例：
    #include <unistd.h>
    #include <pthread.h> 
    
    void *threadfunc(void *arg)
    {
        printf("i am a thread, tid=%lu \n", pthread_self());
        sleep(5);
        printf("i am a thread, tid=%lu \n", pthread_self());
        return (void*)100;
    }
    
    int main()
    {
        pthread_t tid;
        pthread_creat(&tid, NULL, threadfunc, NULL);  
        void *ret;
        pthread_join(tid, &ret); //线程回收  
        
        printf("ret exit with %d \n", (int)ret);
        
        pthread_exit(NULL);
    }
```

//3. 线程结束 

```cpp
    #include <pthread.h> 
        void pthread_exit(void *retval);  //结束后私有资源被释放  
        
参数：
    retval: 线程结束的返回值   
    
注意事项：
    a) 在线程中使用pthread_exit,代表线程退出
    b) 在线程中使用return，代表线程退出 [主控线程return代表退出进程]  
    c) exit代表退出整个进程 [exit不要使用在线程中]  
```

//4. 杀死线程  

```cpp
    #include <pthread.h>
        int pthread_cancel(pthread_t thread)  
        
参数：
    thread:  要杀掉的线程对象id
    
返回值：
    成功：  返回0
    失败：  返回errno

注意：
    被pthread_cancel杀死的线程，退出状态为PTHREAD_CANCELED
    #define PTHREAD_CANCELED((void *)-1)
    
示例：
    #include <stdio.h>
    #include <unistd.h>
    #include <pthread.h>
    
    void *threadfunc(void *arg)
    {
        while(1){
            printff("i am a thread, tid= %lu \n", pthread_self());
            sleep(1);
        }
        return NULL;
    }
    
    int main()
    {
        pthread_t tid;
        pthread_creat(&tid, NULL, threadfunc, NULL);
        
        sleep(5);
        pthread_cancel(tid); //杀死线程  
        void * ret;
        pthread_join(tid, &ret);
        printf("thread exit with %d \n", (int)ret);
        return 0;
    }
 
```

//5. 线程分离  

    线程分离状态：指定该状态，线程主动与主动线程断开关系，线程结束后，其退出状态不由其他线程获取，而直接自己自动释放。网络，多线程服务器常用  
    一般情况下，线程终止后，其终止状态一直保留到其它线程调用pthread_join获取它的状态为止。但是线程也可以被设置为detach状态，这样的线程一旦终止就立刻回收它占用的所有资源，而不保留终止状态。不能对一个已经处于detach状态的线程调用pthread_join,这样的调用将返回EINVAL错误。也就是说，如果已经对一个线程调用了pthread_detach就不能再调用pthread_join了  
```cpp
    #include <pthread.h>
        int pthread_detach(pthread_t thread);
        
参数：
    pthread_t:  要传入的线程对象id  
    
返回值：
    成功：  返回0
    失败：  返回错误号  

示例：
   #include <stdio.h>
   #include <unistd.h>
   #include <pthread.h>
   #include <string.h>

   void *threadfunc(void *arg)
   {
       printf("i am a thread, tid=%lu \n", pthread_self());
       sleep(4);
       printf("i am a thread, tid=%lu \n", pthread_self());
       return NULL;
   }
   
   int main()
   {
       pthread_t tid;
       pthread_creat(&tid, NULL, threadfunc, NULL);
       
       pthread_detach(tid); //线程分离  
       sleep(5);
       
       int ret = 0;
       if((ret=pthread_join(tid, NULL)) > 0)  //会出错
       {
           printf("join err:%d, %s \n", ret, strerror(ret));
       }
       
       return 0;
   }
   
扩展：
    比较两个线程id是否相等
    int pthread_equal(pthread_t t1, pthread_t t2);

```

//6. 线程属性设置分离  

    线程的分离状态决定一个线程以什么样的方式来终止自己  
    
    非分离状态：线程的默认属性是非分离状态，这种情况下，原有的线程等待创建的线程结束。只有当pthread_join函数返回时，创建的线程才算终止，才能释放自己占用的系统资源  
    分离状态：分离线程没有被其他的线程所等待，自己运行结束了，线程也就终止了，马上释放系统资源。应该根据自己的需要，选择适当的分离状态   
    
    线程分离状态的函数：  
    设置线程属性，分离或非分离  
      int pthread_attr_setdetachstate(pthreadd_attr_t *attr, int detachstate);  
      
    获取线程属性，分离或非分离  
      int pthread_attr_getdetachstate(pthread_attr_t *attr, int *detachstate);

```cpp

//初始化线程属性  
int pthread_attr_init(pthread_attr_t *attr);

//销毁线程属性  
int pthread_attr_destroy(pthread_attr_t *attr);

//设置属性分离态  
int pthread_attr_setdetachstate(pthread_attr_t *attr, int detachstate)  //默认允许回收  

参数：
    attr：  init初始化属性  
    detachstate: 
        - PTHREAD_CREAT_DETACHED 线程分离  
        - PTHREAD_CREAT_JOINABLE 允许回收  

示例：
    #include 
    
    
    
```

**命令学习**：
    在家目录的.bashrc增加快捷键    
    alias echomake='cat ~/bin/makefile.template >> makefile'
    
    使用echomake命令，可以快速建立makefile文件，前提是自己已经写好了模板文件  


> 线程同步  
















