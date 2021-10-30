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
         
![Uploading image.png…]()

> 线程同步  
















