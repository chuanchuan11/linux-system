  > 进程基础概念
  
  - (1) PCB概念   
    用户空间 0-3G  
    内核空间 3G-4G  
    PCB属于内核空间，中有file_struct数组指针，存储该进程打开的设备文件列表(fopen打开文件)  

  - (2) C程序存储空间布局
    C程序的存储空间布局：C程序一直由下列几部分组成：  
        正文段：这是由CPU执行的机器指令部分  
               通常正文段是可以共享的。一个程序的可以同时执行N次，但是该程序的正文段在内存中只需要有一份而不是N份  
               通常正文段是只读的，以防止程序由于意外而修改其指令  
        初始化数据段：通常将它称作数据段。  
               它包含了程序中明确地赋了初值的变量：包括函数外的赋初值的全局变量、函数内的赋初值的静态变量  
        未初始化数据段：通常将它称作bss段。在程序开始执行之前，内核将此段中的数据初始化为0或者空指针。
               它包含了程序中未赋初值的变量：包括函数外的未赋初值的全局变量、函数内的未赋初值的静态变量  
        栈段：临时变量以及每次函数调用时所需要保存的信息都存放在此段中。
               每次函数调用时，函数返回地址以及调用者的环境信息（如某些CPU 寄存器的值）都存放在栈中  
               最新的正被执行的函数，在栈上为其临时变量分配存储空间  
               通过这种方式使用栈，C 递归函数可以工作。递归函数每次调用自身时，就创建一个新的栈帧，因此某一次函数调用中的变量不影响下一次调用中的变量  
        堆段：通常在堆中进行动态存储分配。  
               由于历史习惯，堆位于未初始化数据段和栈段之间  
               
![image](https://user-images.githubusercontent.com/42632290/137581091-6a873cfd-6170-47de-aebc-0ed66e996867.png)

注意：  

    栈从高地址向低地址增长。堆顶和栈顶之间未使用的虚拟地址空间很大  
    未初始化数据段的内容并不存放在磁盘程序文件中。需要存放在磁盘程序文件中的段只有正文段和初始化数据段  
        因为内核在程序开始运行前将未初始化数据段设置为 0  

  - (3) 环境变量

    环境变量：是指操作系统中用来指定操作系统运行环境的一些参数  
              命令：env，可以看到所有的环境变量
    相关函数：  
```cpp
    #include<stdlib.h>
    char *getenv(const char*name);

描述：获取环境的值  

参数：  
    name：环境变量名  
    
返回值：  
    成功：指向与name关联的value的指针  
    失败：返回NULL  
    
常用的环境变量民有：
    "HOME"：home目录
    "LANG":语言
    "LOGNAME"：登录名
    "PATH"：搜索路径
    "PWD"：当前工作目录的绝对路径名
    "SHELL"：用户首选的SHELL
    "TERM"：终端类型
    "TMPDIR"：在其中创建临时文件的目录路径名
    "TZ"：时区信息

    #include<stdlib.h>
    int putenv(char *str);
    int setenv(const char *name,const char *value,int rewrite);
    int unsetenv(const char *name);

描述:设置环境变量的值

参数：  
    对于putenv函数：  
        str：形式为name=value的字符串，将其放置到进程的环境表中。如果name已经存在，则先删除其原来的语义  
    对于setenv函数：  
        name：环境变量名  
        value：环境变量的值  
        rewrite：指定覆写行为。  
                 如果它为0，则如果name在环境表中已存在，则直接返回而不修改。同时也不报错   
                 如果它非0，则如果name在环境表中已存在，则首先删除它现有的定义，然后添加新的定义  
    对于unsetenv函数：  
        name：环境变量名  
返回值：  
    对于 putenv函数：  
        成功：返回0  
        失败：返回非0  
    对于setenv/unsetenv函数：  
        成功： 返回 0  
        失败： 返回 -1  

设置环境变量，也可以用命令export key=value
```

  - (4) 进程控制块  

        每个进程在内核中都有一个进程控制块（PCB）来维护进程相关的信息，linux内核的进程控制块是      task_stuct结构体

        a) 进程id，pid_t类型，非负整数  
        b) 进程的状态，初始、就绪、运行、挂起、停止等状态  
        c) 进程切换时需要保存和恢复的一些CPU寄存器  
        d) 描述虚拟地址空间的信息  
        e) 描述控制终端的信息  
        f) 当前工作目录  
        g) umask掩码  
        h) 文件描述符表，包含很多指向file结构体的指针  
        i) 和信号相关的信息  
        j) 用户id和组id  
        k) 会话（Session）和进程组  
        l) 进程可以使用的资源上限 (可以使用ulimit -a 查看所有进程使用的资源设置)  

  - (5) 进程状态  

     进程状态有： 初始、就绪、运行、挂起、终止
 
     就绪状态----------------->（获得CPU）--------------------------->运行状态  
     运行状态----------------->（获得的时间片结束）------------------> 就绪状态  
     运行状态----------------->（因等待其他资源，主动交出CPU）-------> 挂起状态  
     挂起状态----------------->（等待的非CPU资源到位了）-------------> 就绪状态  
     就绪|运行|挂起状态----------------------------------------------> 终止状态  
 
![image](https://user-images.githubusercontent.com/42632290/137582317-998adf6f-63fb-4648-a892-43e91a92696e.png)
  
    就绪态：进程准备运行  
    运行态：进程正在运行  
    挂起态：进程在等待一个事件的发生或某种系统资源  
    终止态：已终止的进程，但pcb没有被释放  

  - (6) 常用进程命令

        ps：查看系统进程信息（可以加|grep test根据关键字查看进程）  
            -ef   显示所有进程信息  
            -aux  显示进程当前状态 
            -ajx  可以追溯进程之间的血缘关系  
            
        top：查看进程动态信息  
        
        /proc：查看进程详细信息  
        
        nice：按用户指定的优先级运行进程（nice –n number ./test）  
             【-20,19】-20最高优先级，19最低，普通用户只能降优先级，不能升  
             
        renice：改变正在运行进程的优先级  
                管理员可以随便改变优先级，普通用户只能降低优先级  
                
        jobs：查看后台进程（./test &后台运行）  
              kill %pid 杀死后台进程  
              
        bg：将挂起的进程在后台运行（bg n）  
            ctrl+z：将前台进程放进后台并挂起  
            
        fg：把后台的进程放到前台运行（fg n）  

        kill:杀死进程  
            -l 查看所有信号  
            -9 pid 给进程发送9号信号，即关闭进程  
            pid 可以直接关闭进程  
> 进程控制

  - (7) 进程创建

```cpp
    #include <unistd.h>
    pid_t fork(void);
    
返回值：
    失败： -1  
    成功： 返回两次  
           父进程返回子进程的id  
           子进程返回0  
           
    #include<unistd.h>
    pid_t getpid(void);  // 返回值：调用进程的进程ID
    pid_t getppid(void); // 返回值：调用进程的父进程ID
    uid_t getuid(void);  // 返回值：返回进程的实际用户ID
    uid_t geteuid(void); // 返回值：返回进程的有效用户ID
    gid_t getgid(void);  // 返回值：返回进程的实际组ID
    gid_t getegid(void); // 返回值：返回进程的有效组ID

```
    注意：
        a) 若父进程先结束  
            子进程成为孤儿进程，被init进程收养  
            子进程编程后台进程  
            
        b) 若子进程先结束  
            父进程如果没有及时回收，子进程变成僵尸进程  
            
        c) 子进程从fork()函数的下一条语句开始执行  

        d) 父子进程(即父进程创建完成后)谁先执行，看操作系统先调用谁，谁就先执行  


  - (8) 进程共享 
 
        a) 父子进程有独立的地址空间，互不影响  
           子进程是父进程的一份一模一样的拷贝，如子进程获取了父进程数据空间、堆、栈的副本  
           父子进程共享正文段（因为正文段是只读的）  
           父子进程并不共享这些数据空间、堆、栈 
           
        b) 由于标准IO库是带缓冲的，因此在fork调用之后，这些缓冲的数据也被拷贝到子进程中

        c) 父进程的所有打开的文件描述符都被复制到子进程中。父进程和子进程每个相同的打开描述符共享同一个文件表项  
           更重要的是：父进程和子进程共享同一个文件偏移量  
                       如果父进程和子进程写同一个描述符指向的文件，但是又没有任何形式的同步，则它们的输出会相互混合  
                       如果父进程fork之后的任务就是等待子进程完成，而不作任何其他的事情，则父进程和子进程无需对打开的文件描述符做任何处理。因为此时只有子进程处理文件  
                       如果父进程fork之后，父进程与子进程都有自己的任务要处理，则此时父进程和子进程需要各自关闭它们不需要使用的文件描述符，从而避免干扰对方的文件操作  

![image](https://user-images.githubusercontent.com/42632290/137587640-f01f1bcf-955b-4b8e-8ecc-7e7db442b12b.png)

        d) 由于创建子进程的目的通常是为了完成某个任务，因此fork之后经常跟随exec，所以很多操作系统的实现并不执行一个父进程数据段、堆和栈的完全拷贝，而是使用"读时共享，写时复制"技术（copy-on-write:COW）  
           这些区域由父进程和子进程共享，而且内核将它们的访问权限改变为只读  
           如果父子进程中有一个试图修改这些区域，则内核只为修改区域的那块内存制作一个副本  

![image](https://user-images.githubusercontent.com/42632290/137587433-e7f7834d-7330-4049-a79a-7e38e66da540.png)

        e) 除了打开的文件描述符之外，子进程还继承了父进程的下列属性：实际用户ID、实际组ID、有效用户ID、有效组ID、附属组ID、进程组ID、会话ID、控制终端、设置用户ID标志和设置组ID标志、当前工作目录、根目录、文件模式创建屏蔽字、信号屏蔽和信号处理、对任一打开文件描述符的执行时关闭标志、环境、连接的共享存储段、存储映像、资源限制  

        f) 父进程和子进程的区别为：  
           fork返回值不同
           进程ID不同
           子进程的tms_utime,tms_stime,tms_cutime,tms_ustime的值设置为0
           子进程不继承父进程设置的文件锁
           子进程的未处理闹钟被清除
           子进程的未处理信号集设置为空集
           
        g) fork失败的零个主要原因：  
           系统已经有了太多的进程  
           实际用户ID的进程总数超过了系统的限制  

  - (9) exec函数族

        当进程调用一种exec函数时，该进程执行的程序完全替换成新程序，而新程序则从main函数开始执行  
            - 调用exec前后，进程ID并未改变。因为exec并不创建新进程  
            - exec只是用磁盘上的一个新程序替换了当前进程的正文段、数据段、堆段和栈段 
        
        目的：   
            实现让父子进程执行不同的程序  
                - 父进程创建子进程  
                - 子进程调用exec函数族  
                - 父进程不受影响  
            在当前进程中执行其他的程序，则当前进程除了进程号不变，其执行内容被执行的程序替换
```cpp
    #include <unistd.h>
    //执行其他程序  
    int execl(const char*path,const char *arg,...);

    //执行程序的时候，使用PATH环境变量，执行的程序可以不用加路径  
    int execlp(const char*file,const char *arg,...);

参数：
    path: 执行的程序名称，包含路径
    file: 执行的程序的名称，在PATH中查找
    arg:  传递给执行的程序的参数列表

返回值：  
    若成功：执行指定的程序，不返回  
    若失败：返回 -1  
 
 示例：
     if(execl("/bin/ls", "ls", "-a", "-l", "/etc", NULL)<0)
         perror("execl")
         
     if(execp("ls", "ls", "-a", "-l", "/etc", NULL)<0)
         perror("execl")
 
 注意：
     最后一个参数必须为NULL, 否则会出错  
    
   int execv(const char* path, char* const argv[]);
   int execvp(const char* file, char* const argv[]);
    
返回值：
    成功： 成功时执行指定的程序，失败返回-1

示例：
     char *arg[] = {"ls", "-a", "-l", "/etc", NULL}
     if(execl("/bin/ls", arg)<0)
         perror("execv")
         
     if(execp("ls", arg)<0)
         perror("execvp") 
         
    #include<stdlib.h>
    int system(const char *cmdstring);  
    
返回值：
    成功时返回命令command的返回值；失败时返回-1
    当前进程等待command执行结束后才继续执行
    
```

  - (10) 进程终止  

    a) 进程有 8 种方式使得进程终止，其中 5 种为正常终止，3 种异常终止：  
       正常终止方式：  
          从main函数返回，等效于exit  
          调用exit函数。exit会调用各终止处理程序，然后关闭所有标准IO流  
          调用_exit函数。它们不运行终止处理程序，也不冲洗标志IO流  
          多线程的程序中，最后一个线程从其启动例程返回。但是该线程的返回值并不用做进程的返回值，进程是以终止状态 0 返回的  
          多线程的程序中，从最后一个线程调用pthread_exit函数。进程也是以终止状态 0 返回的  
       异常终止方式：  
          调用abort函数。它产生SIGABRT信号  
          接收到一个信号  
          多线程的程序中，最后一个线程对取消请求作出响应  

    b) 如果父进程在子进程之前终止，那么内核会将该子进程的父进程改变为init进程，称作由init进程收养。其原理为：  
       在一个进程终止时，内核逐个检查所有活动进程，以判断这些活动进程是否是正要终止的进程的子进程  
       如果是，则该活动进程的父进程ID就改为 1  
       这种方式确保了每个进程都有一个父进程

    c) 相关函数

```cpp
    #include <stdib.h>
    #include <unistd.h>
        void exit(int status)
        void _exit(int status)

返回值：  
    结束当前的进程并将status返回  
    exit结束进程时会刷新流缓冲区  

示例：
    int main()
    {
        printf("this pricess will exit");
        exit(0);
        printf(never be bedisplayed);
    }
    
    ./a.out
    this process will exit

解释：  
  printf（）中没有换行符，因此只是写到输出流的缓冲区，不会输出到终端  
  用exit（）退出进程时，会首先刷新流缓冲区，则将会输出上一句的内容  
  最后一句不会输出，因为进程已经结束了  
```

注意：return和exit的区别   
    a) return返回函数值，是关键字；exit 是一个函数  
    b) return是语言级别的，它表示了调用堆栈的返回；而exit是系统调用级别的，它表示了一个进程的结束  
    c) return是函数的退出(返回)；exit是进程的退出。  
    d) return用于结束一个函数的执行，将函数的执行信息传出给其调用函数使用；exit函数是退出应用程序，删除进程使用的内存空间，并将应用程序的一个状态返回给OS，这个状态标识了应用程序的一些运行信息，这个信息和机器和操作系统有关   

  - (11) 进程回收

    a) 什么是孤儿进程 什么是僵尸进程    
       孤儿进程：父亲死了，子进程被init进程领养。    
       僵尸进程：子进程死了，父进程没有回收子进程的资源（PCB）。    

    b) 相关函数    
```cpp
    #include <unistd.h>
    pid_t wait(int *status)

返回值：  
    成功：返回回收的子进程的进程号  
    失败：返回-1  

示例：
    int status;
    pid_t pid;
    if((pid = fork()) < 0)
        perror("fork");
        exit(-1);
    else if(pid == 0)
        sleep(1);
        exit(2);
    else
        wait(&status);
        printf(%x \n), status;

注意：
    若子进程没有结束，父进程一直阻塞  
    若有多个子进程，哪个先结束就先回收  
    status指定保存子进程返回值和结束方式的地址  
    status为NULL表示直接释放子进程PCB，不接收返回值  

status结束信息可通过宏定义查询：  
    WIFEXITED(status)    判断子进程是否正常结束  
    WEXITSTATUS(status)  获取子进程返回值  
    WIFSIGNALED(status)  判断子进程是否被信号结束  
    WTERMSIG(status)     获取结束子进程的信号类型  


    #include <unistd.h>
    pid_t waitpid(pid_t pid, int *status, int option);

参数：
    pid: 指定的具体要回收的子进程, -1表示回收当前进程的任意一个子进程      
    status： 执行用于保存子进程返回值和结束方式的地址    
    option: 指定回收方式，0或WNOHANG   
            0  表示子进程若没结束，则父进程阻塞直到回收成功才返回  
            WNOHANG 表示不进入阻塞直接返回，若子进程没有结束直接返回0  
     
返回值:  
    成功：返回回收的子进程的pid或0  
    失败：返回-1  

注意：  
    waitpid可以指定具体的子进程和回收方式  
    wait函数不能执行，回收任意子进程  
    返回pid表示进程被成功回收，0表示子进程未结束  

示例：
  
  waitpid(pid, &status, 0)  
  waitpid(pid, &status, WNOHANG)  
  waitpid(-1, &status, 0)  等价于 wait  
  waitpid(-1, &status, WNOHANG)  
  
  
  #include<sys/wait.h>
      int waitid(idtype_t idtype,id_t id,siginfo_t *infop,int options);
  
waitid函数：它类似waitpid，但是提供了更灵活的参数，可以回收的进程组及获取相关进程状态  

  #include<sys/types.h>
  #include<sys/wait.h>
  #include<sys/time.h>
  #include<sys/resource.h>
      pid_t wait3(int *staloc,int options,struct rusage *rusage);
      pid_t wait4(pid_t pid,int *staloc,int options,struct rusage *rusage);

ait3/wait4函数：可以返回终止子进程及其子子进程的资源使用情况  

```

> 进程关系













