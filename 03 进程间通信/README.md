> 进程间通信

- 1. 进程间通信方式  

IPC：进程间通信，通过内核提供的缓冲区进行数据交换的机制。  

    早期UNIX进程间通信方式:  
      (1) 无名管道(pipe) --- 最简单  
      (2) 有名管道(fifo)  
      (3) 信号(signal) --- 携带信息量最小  
      
    System V IPC  
      (4) 共享内存(share memry) --- 速度最快  
          shmget 实现  
          mmap 实现[为了做比较，非System V IPC]  
      (5) 消息队列(message queue)  
      (6) 信号量(semaphore set)  
      
    Socket 套接字  
      (7) 本地socket --- 最稳定  
          多用于跨核网络通信，也可用于本地进程间通信  
          
- 2. 无名管道(pipe)  

    (1) 无名管道具特点  
        a) 只能用于具有亲缘关系的进程之间的通信(如：父子进程，兄弟进程)  
        b) 单工通信模式，即单项数据传递，具有固定的读端和写端  
        c) 它可以看成是一种特殊的文件，对于它的读写也可以使用普通的read、write 等函数，但它不是普通的文件，不属于任何文件系统，只存在于内存中  
        
![image](https://user-images.githubusercontent.com/42632290/137627186-70f17449-188d-418e-80e9-fee66510d50e.png)

![image](https://user-images.githubusercontent.com/42632290/137627156-3b929bb7-b8c3-4bfe-a245-3bcb60930aa6.png)


    (2) 无名管道创建  

```cpp
    #include <unistd.h>
        int pipe(int pfd[2]);

参数：
    pfd:   包含两个元素的整形数组，用来保存文件描述符，pfd[0]用于读管道，pfd[1]用于写管道

返回值：  
    成功：返回0  
    失败：返回-1  
```

    问题：为什么无名管道只能用于具有血缘关系的进程通信？  
        无名管道只存在与内存中，通过fork出来的进程继承了创建好的管道，所以只能用于亲属关系的进程通信 
    
    注意：
         无名管道只能实现单向通信，如果要实现双向通信，则可以通过创建两个管道  
         无名管道中的数据读出来后，则数据就不存在了  

示例1：    
```cpp
#include <stdio.h>
#include <unistd.h>
int main(){
    int fd[2];
    pipe(fd);
    pid_t pid=fork();
    if(pid==0){
        //son
        write(fd[1],"hello",5);
    }else if(pid>0){
        //parent
        char buf[12]={0};
        int ret = read(fd[0],buf,sizeof(buf));
        if (ret>0){
            write(STDOUT_FILENO,buf,ret);
        }
    }
    return 0;
}
```  
示例2：ps | grep 命令实现
```cpp
#include <stdio.h>
#include <unistd.h>

int main()
{
    int fd[2];
    pipe(fd);     //创建管道

    pid_t pid = fork(); 
    if(pid == 0){
        //son -- > ps 
        close(fd[0]);   //关闭 读端
        //1. 先重定向
        dup2(fd[1],STDOUT_FILENO); //标准输出重定向到管道写端（获取输出的内容，写入管道）
        //2. execlp 
        execlp("ps","ps","aux",NULL);  //execlp输出到终端的内容，会重定向到管道中
    }else if(pid > 0){
        //parent
        close(fd[1]);  //关闭写端
        //1. 先重定向，标准输入重定向到管道读端(读取输入的内容)
        dup2(fd[0],STDIN_FILENO);
        //2. execlp 
        execlp("grep","grep","bash",NULL); //将管道读端的内容打印到屏幕终端
    }
    return 0;
}

注意：  
    代码缺点：无法回收子进程，导致僵尸进程出现  
    解决方式：父进程只负责创建进程，回收进程。可以创建两个子进程只负责通信  

```  

    (3) 无名管道读写特性--读无名管道  
![image](https://user-images.githubusercontent.com/42632290/137626983-f5c6e5f4-d295-4390-9ecc-3870478faeba.png)  
![image](https://user-images.githubusercontent.com/42632290/137626990-f4f2d6cc-a2f8-49ec-bf00-6cfc0f6fb501.png)  
 
注意：对于跨越fork的调用管道，会有两个写文件描述符，即子父进程各有一个，只有两个写文件描述符都关闭，程序才认为关闭，read才会返回0

    (4) 无名管道读写特性--写无名管道  

![image](https://user-images.githubusercontent.com/42632290/137627061-620c2184-afb4-47d4-aaf3-2242e9e1de81.png)  
        注意：无空间时候（或空间不足以写入所有字节），则写进程阻塞，直到出现空间把剩余字节写完为止  
        思考：如果获取无名管道的大小？
              循环写入管道，直接阻塞，统计循环次数   
              long fpathconf(int fd,int name)  该函数能直接获取管道的属性  
代码示例：  
```cpp
#include <unistd.h>
#include <stdlib.h>

int main()
{
    int count = 0, pfd[2];
    char buf[1024];
    
    while(1)
    {
        write(pfd[1], buf, 1024);
        printf("wrote %dk bytes\n", ++count);
    }  
}
```  
![image](https://user-images.githubusercontent.com/42632290/137627324-62dcdff0-d452-44f9-92f0-831ae829de5f.png)  

注意：系统不允许进程写一个读端不存在的管道，否则出现进程异常结束（即管道断裂）,会产生一个SIGPIPE信号

思考：如何验证管道破裂?  
    子进程写管道，父进程回收  
代码示例：  
![image](https://user-images.githubusercontent.com/42632290/137627368-c3064a2c-c9c8-47df-a393-77d4c4f37815.png)  
./pipe_break  
     unformal  
     0  

    (5) popen和pclose函数  
        作用：常常需要创建一个连接到另外一个进程的管道，然后读其输出或向其输入端发送数据，为此有两个标准IO库函数popen和pclose。其实现的操作是创建一个管道，fork一个子进程，关闭未使用的管道端，执行一个shell命令，然后等待命令终止  

```cpp
    #include <stdio.h>
        FILE *popen(const char* cmdstring, const char* type);  
        
返回值：
    成功：返回文件指针    
    失败：返回NULL    
    
        int pclose(FILE *fp); 
        
 返回值：
    成功：返回cmdstring的终止状态    
    失败：返回-1         

```  
![image](https://user-images.githubusercontent.com/42632290/137627694-ea772d1c-9bf6-492e-a9bf-189d52062520.png)


- 3. 有名管道(FIFO)  

  (1) 有名管道特点  
        a) 对应管道文件，可用于任意进程之间进行通信[解决pipe只能在亲属进程间通信的问题]   
        b) 打开管道时可指定读写方式  
        c) 通过文件IO操作，内容存放在内存中  
        
    FIFO是linux基础文件类型中的一种(伪文件)。但FIFO文件在磁盘上没有数据块[文件大小为0]，仅仅用来标识内核中一条通道。各进程可以打开这个文件进行read/write，实际上在读写内核通道，这样就实现了进程间通信  
    注意：（1）不论是有名管道还是无名管道，当读端和写端都关闭的时候，存放在内存中的数就被释放掉了  
  
  (2) 有名管道创建  
    
```cpp
    #include <unistd.h>  
    #include <fcntl.h>  
        int mkfifo(const char *path, mode_t mode);

参数：  
    path:   创建的管道文件路径  
    mode:   管道文件的权限，如0666  

返回值：  
    成功：  返回0  
    失败：  返回-1  
    
注意：  
    （1）可以是相对路径，也可以是绝对路径，没有路径，则表示在当前文件下创建  
    （2）管道文件主要是读写权限  

```

代码示例：  
```cpp  
//创建fifo
int main(void) 
{
    if(mkfifo(“./myfifo”, 0666) < 0)  //执行成功后，在当前目录下会出现一个myfifo的文件
    {
          perror(“mkfifo”);exit(-1);
    }
     return 0;
}

//写fifo
int main(void) 
{
     char buf[32];
     int pfd;
     if ((pfd = open(“myfifo”, O_WRONLY)) < 0)
    {
          perror(“open”);  exit(-1);
     }
     while ( 1 ) 
    {
          fgets(buf, 32, stdin);
          write(pfd, buf, 32); 写入管道
    }
     close(pfd);//关闭打开的管道
     return 0;
  }

//读fifo
int main(void) 
{
     char buf[32];
     int pfd;
     if ((pfd = open(“./myfifo”, O_RDONLY)) < 0) 
    {
          perror(“open”); exit(-1);
     }
     while (read(pfd, buf, 32) > 0) 
    {
          printf(“the length of string is %d\n”,strlen(buf));
     }
     close(pfd);
     return 0;
  }
```  

  (3) 有名管道读写特性  

     a) 当写端退出，读端读管道时若没有数据，会返回0  
        当读端存在，读端读管道时若没有数据，则进入阻塞状态  
     b) 当进程打开管道时，此时只有读端或只有写端存在，则程序直接进入阻塞状态[即阻塞在open调用处]直到另外一端被打开  
         即：打开fifo文件的时候，read端会阻塞等待write端open，同理，write端也会阻塞等待read端open  

- System V IPC通信

    linux系统中进行进程间通信时，会发现有System V IPC 以及POXIS IPC 两种类型:  
       (1) POSIX(Portable Operating System Interface)可移植操作系统接口。它是由IEEE(电子和电气工程师协会)开发，由ANSI(美国国家标准化学会)和OSI(国际标准化组织)两个机构标准化。由于早起各厂家对UNIX的开发各自为政，互相竞争，造成UNIX版本混乱，给软件移植造成困难，不利于UNIX长期发展，基于此，IEEE开发了POSIX，在源码级别定义了一组UNIX操作系统接口  
       (2) System V(System Five),是Unix操作系统众多版本中的一支，最初由 AT&T 开发，在1983年第一次发布  
    总结来说：System V 和 POXIS 是一种应用于系统的接口协议, 用于进程间通信的一种机制   

    System V IPC 特点:    
        每个IPC对象有唯一的ID  
        IPC对象创建后一直存在，直接被显式删除  
        每个IPC对象有一个关联的**KEY** 

   **注意**：
       (1) IPC对象创建后，一般只有创建IPC对象的进程能够识别到ID，为了让的进程也能获取，因此只能通过KEY来访问
       (2) 每个进程必须生成相同的KEY，这样才能通过相同的KEY找到对应的IPC对象

   **KEY的创建**  
       (1) 通过函数创建key  
       (2) 私有key（IPC_PRIVATE）则表示只有当前进程自己能够访问共享空间  
       
![image](https://user-images.githubusercontent.com/42632290/137745911-4f4f43e1-558e-45c6-a717-9b66ba5ed12b.png)  

   **相关函数**  

```cpp
     #include <sys/types.h>
     #include <sys/ipc.h>
         key_t ftok(const char *path, int proj_id);

参数：
    path：    存在且可访问的文件路径  
    proj_id:  用于生成key的数字，不能为0  

返回值：  
    成功：返回合法key值  
    失败：返回-1  
    
代码示例：

int main()
{
   key_t  key;
   if ((key = ftok(“.”, ‘a’)) == -1) //创建key值
   {
      perror(“key”);
      exit(-1);
}   
```

- 4. 共享内存(shmget)  

    共享内存允许两个或多个进程共享一给定的存储区，因为数据不需要来回复制，所以是最快的一种进程间通信机制。共享内存可以通过mmap()映射普通文件 （特殊情况下还可以采用匿名映射）机制实现，也可以通过systemV共享内存机制实现。应用接口和原理很简单，内部机制复杂。为了实现更安全通信，往往还与信号灯等同步机制共同使用  
    
  (1) 共享内存特点：  
    
     a) 共享内存是最快的一种IPC方式，进程可以直接读写内存，而不需要任何数据的拷贝  
     b) 共享内存在内核空间创建，可**被进程映射到用户空间**访问，使用灵活  
     c) 由于多个进程可同时访问共享内存，因此需要同步和互斥机制配置使用  
        
     与管道相比：管道使用时候需要在内核空间和用户空间进行切换，数据的拷贝，内存开销大；因此共享内存的效率最高  
     
  **共享内存使用步骤：**  
     a) 创建/打开共享内存  
     b) 映射共享内存，即把指定的共享内存映射到进程的地址空间用于访问  
     c) 读写共享内存  
     d) 撤销共享内存映射  
     e) 删除共享内存对象  

 **注意：**  
     a) 共享内存创建好之后，需要进行映射后才可以访问（即把指定的共享内存映射到进程的地址空间用于访问）  
     b) 类似于malloc，IPC创建的一个特殊的地址范围，进程可以将它连接到自己的地址空间进行访问  
     c) 不删除，则共享内存一直存在  

  (2) 共享内存-创建

```cpp
    #include  <sys/ipc.h>  
    #include <sys/shm.h>  
      int  shmget(key_t key,  int size, int shmflg);  

参数：  
    key：    和共享内存关联的key，IPC_PRIVATE 或 ftok生成  
    Size：   创建的共享内存大小  
    shmflg： 共享内存标志位  IPC_CREAT | 0666
             IPC_CREAT 表示如果与key关联的共享不存在则新建，若存在直接打开  
返回值：  
    成功： 返回共享内存的id
    失败： 返回-1 

示例1：创建/打开一个和key关联的共享内存，大小为1024字节，权限为0666

  key_t key;
  int shmid;
 
  if ((key = ftok(“.”, ‘m’)) == -1) {
     perror(“ftok”); exit(-1);
  }
  if ((shmid = shmget(key, 1024, IPC_CREAT|0666)) < 0) {
     perror(“shmget”); exit(-1);
  }
```  

  (3) 共享内存-映射与读写

```cpp
    #include <sys/types.h>
    #include <sys/shm.h>  
        void  *shmat(int shmid,  const void *shmaddr,  int shmflg);  

参数：  
    shmid：       要映射的共享内存id  
    shmaddr：     映射后的地址， NULL表示由系统自动映射 
    shmflg：      标志位 0表示可读写；SHM_RDONLY表示只读
    
返回值：  
    成功：返回映射后的地址
    失败： 返回(void *)-1  
  
注意：  
    将共享内存连接到当前进程的地址空间，返回地址的类型根据存放的数据类型进行确定

代码示例：  

char *addr;
int  shmid;

if ((addr = (char *)shmat(shmid, NULL, 0)) == (char *)-1) {
     perror(“shmat”); exit(-1);
}
fgets(addr, N, stdin); //往共享内存存放键盘输入的字符串

```

  (4) 共享内存-撤销映射

```cpp
    #include <sys/types.h>
    #include <sys/shm.h>  
        int  shmdt(void * shmaddr);  
        
参数:
    shmaddr:  撤销的共享内存地址  

返回值：  
    成功:  返回0  
    失败:  返回-1  

注意：
    不使用共享内存时，应撤销映射  
    进程结束时自动撤销  
    共享内存撤销并没有删除，只是对当前进程不再可访问
```

  (5) 共享内存-控制

```cpp
    #include <sys/ipc.h>
    #include <sys/shm.h>  
        int  shmctl(int shmid, int cmd, struct shmid_ds * buf); 
参数：  
    shmid：  要操作的共享内存的id  
    cmd：    要执行的操作，取值如下：  
             IPC_STAT    获取共享内存属性  
             IPC_SET     设置共享内存属性  
             IPC_RMID    删除共享内存ID  
    buf:     保存或设置共享内存属性的地址 
              Struct  shmid_ds{
                   uid_t  shm_perm.uid;
                   uid_t  shm_perm.gid;
                   mode_t  shm_perm.mode;
              }
返回值：   
  成功：  返回0  
  失败：  返回-1

注意事项：  
  a) 每块共享内存大小有限制   
  b) ipcs -l 查看共享内容默认属性  
  c) cat /proc/sys/kernel/shmmax 可以改变共享内容属性设置  
  d) shmctl(shmid, IPC_RMID, NULL) 删除共享内存  
  e) nattach 变成0时真正删除  
  f) 通常删除的共享内存段还能继续使用，直到从最后一个进程中分离为止，但这个行为并未在规范中定义，最好不要依赖它  

代码示例：  
// 进程1读共享内存  
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <unistd.h>
#include <sys/shm.h>

struct shared_use_st
{  
    int rw_flag;   //作为一个标志，1：表示可读，0表示可写 
    char text[256];//记录写入和读取的文本
};

int main()
{
    key_t key;
    int shmid;
    void * addr = NULL;//存放字符串
    struct shared_use_st * shared = NULL;

    //1. through ftok generate key
    key=ftok(".", 'a'))
    //2. creat share memory
    shmid=shmget(key, 1024, IPC_CREAT|0666))
    //3. map 
    addr=shmat(shmid, NULL, 0)

    shared = (struct shared_use_st *)addr; 
    while(1)
    {
        if(shared->rw_flag == 1)
        {
            printf("your inputs is:%s", shared->text);
            shared->rw_flag = 0;
            if(strncmp(shared->text, "end", 3) == 0)break; 
        }
        else
        {
            sleep(1);
        }
        
    }

    //4. 把共享内存从当前进程中分离，撤销共享内存
    shmdt(shared)
    //5. last process 删除共享内存
    shmctl(shmid, IPC_RMID, 0)  

    return 0;
}
//进程2读共享内存  
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <unistd.h>
#include <sys/shm.h>

struct shared_use_st
{  
    int rw_flag;   //作为一个标志，1：表示可读，0表示可写 
    char text[256];//记录写入和读取的文本
};

int main()
{
    key_t key;
    int shmid;
    void * addr = NULL;//存放字符串
    struct shared_use_st * shared = NULL;

    //1. through ftok generate key
    key=ftok(".", 'a')
    //2. creat share memory  
    shmid=shmget(key, 1024, IPC_CREAT|0666)
    //3. map 
    addr=shmat(shmid, NULL, 0)

    shared = (struct shared_use_st *)addr; 
    shared->rw_flag = 0;
    while(1)
    {
        if(shared->rw_flag == 0)
        {
            printf("please inputs:");
            fgets(shared->text, 256, stdin);//write share memory
            shared->rw_flag = 1;
            if(strncmp(shared->text, "end", 3) == 0)break; 
        }
        else
        {
            sleep(1);
        }
    }
    //4. 把共享内存从当前进程中分离
    shmdt(shared)
    return 0;
}
```

- 5. 共享内存(mmap) 

    mmap()系统调用使得进城之间通过映射同一个普通文件实现共享内存（特殊情况下还可以采用匿名映射）。普通文件被映射到进程地址空间后，进程可以像访问普通内存一样对文件进行访问，不必再调用read,write等操作  
    
![image](https://user-images.githubusercontent.com/42632290/137925634-6d2199a4-55ca-4e79-84c6-240b670afbe3.png)

  (1) mmap()函数: 创建内存映射区   

```cpp
    #include <sys/mman.h>
        void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);

参数：  
    addr:     传NULL
    length:   映射到调用进程地址空间的字节数，它从被映射文件开头offset个字节开始算起，一般offset设置为0，表示从文件开始映射  
    prot:     指定共享内存的访问权限    
              PROT_READ（可读）   
              PROT_WRITE （可写）  
              PROT_EXEC （可执行）  
              PROT_NONE（不可访问）  
    flags:    共享内存标志位，常见值如下：  
              MAP_SHARED   共享映射，进程共享映射区，对内存的修改会影响到源文件    
              MAP_PRIVATE  私有映射，进程各自独占映射区  
    fd:       fd为即将映射到进程空间的文件描述字，一般由open()返回  
              fd可以指定为-1，此时须指定flags参数未MAP_ANON，表明进行的是匿名映射（不涉及具体的文件名，避免了文件的创建及打开，很显然只能用于具有亲缘关系的进程间通信）  
    offset:   偏移量，一般设为 0 表示从文件头开始映射  

返回值：   
    成功：  返回可用的内存首地址  
    失败：  返回MAP_FAILED  
```
    
  (2) munmap()函数：解除内存映射区   
  
```cpp
    #include <sys/mman.h>
        int munmap(void* addr, size_t length);

参数：  
    addr:    调用mmap()时返回的地址  
    length:  创建映射区的大小

返回值：  
    成功： 返回0  
    失败： 返回-1  

注意：  
      当映射关系解除后，对原来映射地址的访问将导致段错误发生   
```

![image](https://user-images.githubusercontent.com/42632290/138296384-429f37bd-25d4-4393-b303-1bd42ddf3b71.png)

  (3) 示例1：修改MAP_SHARED属性的共享内存，能影响到文件  

```cpp
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <string.h>

int main()
{
    int fd = open("mem.txt",O_RDWR);     1. //创建并且截断文件
    //int fd = open("mem.txt",O_RDWR|O_CREAT|O_TRUNC,0664);//创建并且截断文件
    //2. 设置文件大小  
    ftruncate(fd,8);   
    //3. 创建映射区
   char *mem = mmap(NULL,20,PROT_READ|PROT_WRITE,MAP_SHARED,fd,0);
    //char *mem = mmap(NULL,8,PROT_READ|PROT_WRITE,MAP_PRIVATE,fd,0);

    if(mem == MAP_FAILED){
        perror("mmap err");
        return -1;
    }
    close(fd);
    //4. 拷贝数据
    strcpy(mem,"helloworld");  //通过cat读文件，发现文件内容发生了变化  

    //5. 释放mmap
    if(munmap(mem,20) < 0){
        perror("munmap err");
    }
    return 0;
}
```

  (4) 示例2：匿名映射  
   
```cpp
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/mman.h>
#include <fcntl.h>
#include <sys/wait.h>

int main()
{
    //1. 创建内存映射区
    int *mem = mmap(NULL,4,PROT_READ|PROT_WRITE,MAP_SHARED|MAP_ANON,-1,0);
	
    //是否创建成功
    if(mem == MAP_FAILED){
        perror("mmap err");
        return -1;
    }
	
    //2. 创建子进程
    pid_t pid = fork();

    //3. 针对父子进程处理
    if(pid == 0 ){
        //son 
        *mem = 101;
        printf("child,*mem=%d\n",*mem);
        sleep(3);
        printf("child,*mem=%d\n",*mem);
    }else if(pid > 0){
        //parent 
        sleep(1);
        printf("parent,*mem=%d\n",*mem);
        *mem = 10001;
        printf("parent,*mem=%d\n",*mem);
        wait(NULL); //回收子进程
    }

    munmap(mem,4);
    return 0;
}

优点：创建映射区不需要依赖于文件，比较方便  
缺点：由于没有产生文件，只能进行有血缘关系进程之间的通信 
注意：
    MAP_ANON或ANONYMOUS这两个宏在有些unix系统没有  
    可以借助/dev/zero 进行实现[聚宝盆文件，可以随意映射，/dev/null为黑洞文件 可以放置任何垃圾信息]  
    
```

  (5) 示例3：任意进程之间通信  

```cpp
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/mman.h>
#include <fcntl.h>
#include <sys/wait.h>

typedef struct  _Student{
    int sid;
    char sname[20];
}Student;

int main(int argc,char *argv[])
{
    if(argc != 2){
        printf("./a.out filename\n");
        return -1;
    }
    
    // 1. open file 
    int fd = open(argv[1],O_RDWR|O_CREAT|O_TRUNC,0666);
    int length = sizeof(Student);

    ftruncate(fd,length);

    // 2. mmap
    Student * stu = mmap(NULL,length,PROT_READ|PROT_WRITE,MAP_SHARED,fd,0);
    
	//判断返回值
    if(stu == MAP_FAILED){
        perror("mmap err");
        return -1;
    }
    int num = 1;
    // 3. 修改内存数据
    while(1){
        stu->sid = num;
        sprintf(stu->sname,"xiaoming-%03d",num++);
        sleep(1);//相当于没隔1s修改一次映射区的内容
    }
    // 4. 释放映射区和关闭文件描述符
    munmap(stu,length);
    close(fd);

    return 0;
}

注意：
    如果进程要通信，flags必须设置为MAP_SHARED, MAP_PRIVATE则不会通信  
    
学习一个命令技巧：  
    head -n oldfiile > newfile  [将oldfile开头n行内容输入到newfile]  
```
  (6)mmap八问：

```cpp
    a) 如果更改mem变量的地址，释放的时候munmap，传入mem还能成功吗？  不能！

    b) 如果对mem越界操作会怎么样？文件的大小对映射区操作有影响，尽量避免 [文件较大，即使越界也能写入，文件较小，则只写实际大小]  

    c) 如果文件偏移量随便填个数会怎么样？ offset必须是 4k 的整数倍，否则报错

    d) 如果文件描述符先关闭，对mmap映射有没有影响？ 没有影响 [只要做了，mmap操作则已经在内存中申请了映射空间，即使fd提前关闭，对读写并没有影响]

    e) open的时候，可以新创建一个文件来创建映射区吗？ 不可以用大小为0的文件 [可以使用**ftruncate**设置大小]  

    f) 文件选择O_WRONLY，可以吗？ 不可以： Permission denied [在调用mmap做映射时候，隐含了一次读操作] 

    g) 当选择MAP_SHARED的时候，open文件选择O_RDONLY，prot可以选择 PROT_READ|PROT_WRITE吗？ Permission denied [SHARED的时候，映射区的权限<= open文件的权限]

    h) 如果不判断返回值会怎么样？ 必须判断返回值，因为很容易报错误,使用mmap时候一定要判断  

```

  (7)mmap与shmget的区别   

    mmap的机制：就是在磁盘上建立一个文件，每个进程存储器里面，单独开辟一个空间来进行映射。如果多进程的话，那么不会对实际的物理存储器（主存）消耗太大  
    
    mmap, 它把文件内容映射到一段内存上(准确说是虚拟内存上), 通过对这段内存的读取和修改, 实现对文件的读取和修改,mmap()系统调用使得进程之间可以通过映射一个普通的文件实现共享内存。普通文件映射到进程地址空间后，进程可以向访问内存的方式对文件进行访问，不需要其他系统调用(read,write)去操作。 mmap图示例：  

![image](https://user-images.githubusercontent.com/42632290/138295964-4e1cbd2b-c3b1-469d-b3e4-46ff84205721.png)  


    shm的机制：每个进程的共享内存都直接映射到实际内存里面  

![image](https://user-images.githubusercontent.com/42632290/138296707-d295fc8a-6704-45f4-85b2-eee3c1811957.png)  

    总结：  
        (1) mmap保存到实际硬盘，实际存储并没有反映到主存上。优点：储存量可以很大（多于主存）；缺点：进程间读取和写入速度要比主存的要慢  
	(2) shm保存到物理存储器（主存），实际的储存量直接反映到主存上。优点，进程间访问速度（读写）比磁盘要快；缺点，储存量不能非常大（多于主存）  
	(3) 二者本质上是类似的，mmap可以看到文件的实体，而 shmget 对应的文件在交换分区上的 shm 文件系统内，无法直接 cat 查看  
	
    如果分配的存储量不大，那么使用shm；如果存储量大，那么使用mmap  

    持续性：  
         进程挂了重启不丢失内容，二者都可以做到  
         机器挂了重启，mmap 可以不丢失内容（文件内保存了OS同步过的映像），而 shmget 会丢失  

- 6. 消息队列(msg) 

     消息队列是System V IPC对象的一种，消息队列由消息队列ID来唯一标识，消息队列就是一个消息的列表，用户可以在消息队列中添加消息、读取消息等  

  (1) 消息队列特点  
    
      a) 消息队列是消息的链接表，存放与内核中  
      b) 消息队列可以实现消息的随机查询，消息不一定要以先进先出的次序读取，也可以按照类型来发送/接收消息读取 
      c) 与管道例子不同，消息队列不需要由自己进程提供同步方法（这是消息队列相对于管道的一个明显优势） 
      
      消息队列与命名管道类似，但少了打开和关闭管道方面的复杂性。使用消息队列并未解决我们在使用命名管道时遇到的一些问题，如管道满时的阻塞问题。消息队列提供了一种在两个不相关进程间传递数据的简单有效的方法。与命名管道相比：消息队列的优势在于，它独立于发送和接收进程而存在，这消除了在同步命名管道的打开和关闭时可能产生的一些困难。消息队列提供了一种从一个进程向另一个进程发送一个数据块的方法。而且每个数据块被认为含有一个类型，接收进程可以独立地接收含有不同类型值的数据块  

      优点：
          A. 我们可以通过发送消息来几乎完全避免命名管道的同步和阻塞问题  
          B. 我们可以用一些方法来提前查看紧急消息  

      缺点：
          A. 与管道一样，每个数据块有一个最大长度的限制  
          B. 系统中所有队列所包含的全部数据块的总长度也有一个上限  

     Linux系统中有两个宏定义:  
         MSGMAX, 以字节为单位，定义了一条消息的最大长度  
         MSGMNB, 以字节为单位，定义了一个队列的最大长度  

     限制：  
     由于消息缓冲机制中所使用的缓冲区为共用缓冲区，因此使用消息缓冲机制传送数据时，两通信进程必须满足如下条件  
     （1）在发送进程把写入消息的缓冲区挂入消息队列时，应禁止其他进程对消息队列的访问，否则，将引起消息队列的混乱。同理，当接收进程正从消息队列中取消息时，也应禁止其他进程对该队列的访问  
     （2）当缓冲区中无消息存在时，接收进程不能接收任何消息；而发送进程是否可以发送消息，则只由发送进程是否能够申请缓冲区决定         
  
![image](https://user-images.githubusercontent.com/42632290/138537901-df4f3a68-b32e-4dfd-a1ef-f7962c6a0a17.png)

  (2) 消息队列使用步骤  

    创建/打开消息队列     msgget  
    向消息队列发送消息    msgsnd  
    从消息队列接收消息    msgrcv  
    控制消息队列          msgctl  

![image](https://user-images.githubusercontent.com/42632290/138537966-d2438cbd-232c-4a6b-9fab-1456fa2e9280.png)

  - 消息队列创建/打开 - msgget 
 
```cpp

    #include <sys/ipc.h>
    #include <sys/msg.h>  
        int msgget(key_t key, int msgflg);

参数： 
    key：    消息队列关系的key，取值为IPC_PRIVATE 或 通过ftok()创建  
             私有消息队列只能在一个进程中使用，通过ftok函数创建的则可以在多进程通信  
    msgflg： 标志位 IPC_CREAT | 0666  
             有文件存在则直接检查权限，没有则创建  

返回值：
    成功：    返回消息队列id  
    失败：    返回-1  

//示例
int main(){
   key_t  key;
   int msgid;
   if ((key = ftok(“.”, ‘q’)) == -1){
        perror(“key error”); exit(-1);
   } 
   if((msgid = msgget(key, IPC_CREAT|0666)) < 0){
        perror(“msgget error”); exit(-1);
   }  
```

  - 消息发送  - msgsnd  
  
```cpp
    #include <sys/ipc.h>
    #include <sys/msg.h>  
        int  msgsnd(int msgid,  const void *msgp,  size_t  size,  int  msgflg);  

参数： 
    msgid:    消息队列id  
    msgp:     消息缓冲区地址  
              自定义的消息结构体，存放消息  
    size:     消息正文长度  
    msgflg:   标志位0 或 IPC_NOWAIT 
              0：  发送成功时返回  
	      IPC_NOWAIT:  不论是否成功，立马返回结果  

返回值：  
    成功：  返回0
    失败：  返回-1

//示例  
struct{
    long type;
    char text[64];
}buf;
#define len  ( sizeof(MSG) – sizeof(long) )  //消息正文长度  

Int main(){
    …创建/打开消息队列
    bug.type = 100;  //消息类型一般为正整数  
    fgets(buf.text,  64,  stdin);  //存放数据  
    msgsnd(msgid,  &buf,  len,  0);  //发送数据，成功才返回结果  
    …
}

注意： 一般第一个进程创建消息队列，后边的进程根据key去操作消息队列  
```

  - 消息接收  - msgrcv  

```cpp
    #include <sys/ipc.h>
    #include <sys/msg.h>  
        int  msgrcv(int msgid, void *msgp, size_t size, long msgtype, int msgflg);  

参数：  
    msgid    消息队列id  
    msgp     消息缓冲区地址  
    size     指定接收的消息长度  
    Msgtype  指定接受的消息类型  
    msgflg   标志位0 或 IPC_NOWAIT  

返回值：  
    成功：  返回收到的消息长度  
    失败：  返回-1  

//示例
struct{
    long type;
    char text[64];
}buf;
#define len  ( sizeof(MSG) – sizeof(long) )

Int main(){
    …打开消息队列
    if(msgrcv(msgid,  &buf,  len,  100,  0) < 0){
         Perror(“msgrcv error”); exit(-1);
    }
    …
}
```

  - 消息队列控制 - msgctl  

```cpp
    #include <sys/ipc.h>
    #include <sys/msg.h>  
        int  msgctl(int msgid, int cmd, struct msqid_ds * buf);  

参数：  
    msgid：  消息队列id  
    cmd：    要执行的操作，取值如下
             IPC_STAT：    获取消息队列信息   
	     IPC_SET：     设置消息队列信息  
	     IPC_RMID ：   删除消息队列信息  
    
    buf：    存放消息队列属性的地址  
             Struct  msqid_ds{  
                 uid_t  msg_perm.uid;  
                 uid_t  msg_perm.gid;  
                 mode_t  msg_perm.mode;  
             }  

返回值：  
    成功：  返回0
    失败：  返回-1

注意：  
    一般最后一个结束的进程负责删除消息信息，删除操作不需要buf参数  
    
```

  - 代码练习  
    
    两个进程通过消息队列轮流将键盘输入的字符串发送给对方，接收并打印对方发送的消息  

```cpp
//进程a
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>

typedef struct 
{
    long mytype;
    char mytest[64];
}MSG;

#define LEN  (sizeof(MSG)-sizeof(long))
#define Type_A 100
#define Type_B 200

int main()
{
    key_t key;
    MSG buf;
    int msgid;    

    key=ftok(".", 'q')
    
    msgid=msgget(key, IPC_CREAT|0666) //creat message

    while(1)
    {
        buf.mytype = Type_B;
        printf("=====>input contents to B:");
	fgets(buf.mytest, 64, stdin);
	
	msgsnd(msgid, (void *)&buf, LEN, 0)  //send message
 
	if(strcmp(buf.mytest, "quit\n") ==0)
	{
	    break;
	}
        printf("wait receive form B...\n");
	
	msgrcv(msgid, (void *)&buf, LEN, Type_A, 0) //receive message

	if(strcmp(buf.mytest, "quit\n") == 0)
	{
	   msgctl(msgid, IPC_RMID, 0); //the last process remove message
	   exit(0);
	}
        printf("receive message:%s\n", buf.mytest);

    }

    return 0;
}

//进程B
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>

typedef struct 
{
    long mytype;
    char mytest[64];
}MSG;

#define LEN  (sizeof(MSG)-sizeof(long))
#define Type_A 100
#define Type_B 200

int main()
{
    key_t key;
    MSG buf;
    int msgid;    

    key=ftok(".", 'q')

    msgid=msgget(key, IPC_CREAT|0666) //creat message

    while(1)
    {
        printf("wait receive from A\n");
	
        msgrcv(msgid, (void *)&buf, LEN, Type_B, 0) //receive

        if(strcmp(buf.mytest, "qiut\n") == 0)
        {
             msgctl(msgid, IPC_RMID, 0); //last process remove message
             exit(0);
        }
        printf("receive message:%s\n", buf.mytest);

        buf.mytype = Type_A;
        printf("==========>input content to A:");
        fgets(buf.mytest, 64, stdin);
	
        msgsnd(msgid, (void *)&buf, LEN, 0)  //send message
  
        if(strcmp(buf.mytest, "qiut\n") ==0)
        {
            break;
        }
    }

    return 0;
}  



```  

  - 命令学习  
  
    消息队列独立于进程，故进程结束可以查看队列信息  

    a) ipcs –q：查看系统中存在的消息队列  
    b) ipcrm  -q  msgid：删除某个消息队列  

- 7. 共享内存与消息队列的比较   

     共享内存区是最快的可用IPC形式，一旦这样的内存区映射到共享它的进程的地址空间，这些进程间数据的传递就不再通过执行任何进入内核的系统调用来传递彼此的数据，节省了时间  
     共享内存和消息队列，FIFO，管道传递消息的区别：  
       ——消息队列，FIFO，管道的消息传递方式一般为  
          1：服务器得到输入  
          2：通过管道，消息队列写入数据，通常需要从进程拷贝到内核  
          3：客户从内核拷贝到进程  
          4：然后再从进程中拷贝到输出文件 
	  
      上述过程通常要经过4次拷贝，才能完成文件的传递
      
       ——共享内存只需要  
           1:从输入文件到共享内存区  
           2:从共享内存区输出到文件  

      上述过程不涉及到内核的拷贝，所以花的时间较少，这也是共享内存效率高的原因  

- 8. 信号量  

     信号量与管道、FIFO以及消息队列不同，它是一个计数器，主要提供对进程间共享资源访问控制机制，实现进程间的互斥与同步，而不是用于存储进程间通信数据  
  
     (1) 信号量类型  
         
	 a) posix 信号量[参考UNIX环境高级编程]
	 b) system V  信号量[本文只讲此信号量]  

     (2) 信号量特点  
         
	 a) System V 信号量是一个或多个计数信号灯的集合  
	 b) 可同时操作集合中的多个信号灯  
	 c) 申请多个资源时避免死锁问题的产生  
            进程1在申请时候AB时候，可能进程2也在申请，造成P1有A，P2有B，然后两个进程都进入等待状态，发生死锁  
	    
![image](https://user-images.githubusercontent.com/42632290/138544137-31306425-ac27-4a2e-908e-0914f73b1e84.png)  

         信号量执行过程： 
	     a) 若此信号量的值为正，则进程可以使用该资源。在此情况下，进程会将信号值减1表示它使用了一个资源单位  
	     b) 若信号量的值为0，则进程进入休眠状态，直至信号量值大于0，进程被唤醒后，返回至第a步  

     (3) 信号量使用步骤  
     
         a) 创建/打开信号灯    semget  
         b) 信号灯初始化       semctl  
         c) P/V操作            semop  
         d) 删除信号灯         semctl  

- 信号量的创建/打开  
```cpp
    #include <sys/ipc.h>
    #include <sys/sem.h>  
        int  semget(ket_t  key,  int  nsems,  int  semflg); 

参数：  
    Key：    和信号量关联的key, IPC_PRIVATE 或 ftok   
    Nsems:   集合中包含的计数信号灯数目   
    Semflg:  标志位  IPC_CREAT|0666|IPC_EXCL  
             IPC_CREAT： 信号量不存在，建立新的信号量，存在则获取    
             IPC_EXCL  ：信号量不存在，建立新的信号量，存在则发送错误码    

返回值：
    成功: 返回信号灯id
    失败: 返回-1

示例：
if((key = ftok(".",'s')) == -1){
        perror("ftok");  exit(-1);
    }
if((semid = semget(key, 2, IPC_CREAT|0666)) < 0){
        perror("semget"); exit(-1);
    }

```

- 信号量的初始化    
```cpp  
    #include <sys/ipc.h>  
    #include <sys/sem.h>  
        int  semctl(int semid, int semnum, int cmd, …);  

参数：  
    semid：     要操作的信号灯集id  
    semnum：    要操作的集合中信号灯编号  
    cmd：       执行的操作, SETVAL或IPC_RMID  
    union semun：  取决于cmd  
                   union semun {  
                         int val;  
                         struct semid_ds *buf;   
                         unsigned short  *arry;   
                         struct seminfo  *_buf;   
                         };  
			 
    semun联合体必须由程序员自己初始化，且至少包含以上几个成员

返回值：
    成功时返回0，失败时返回-1  
    
示例：  
    //两个信号量，第一个初始化为1，第二个初始化为0  
union semun  myun;  //定义一个共用体变量
myun.val = 1;
if (semctl(semid, 0, SETVAL, myun) < 0) {
     perror(“semctl”);     exit(-1);
}
myun.val = 0;
if (semctl(semid, 1, SETVAL, myun) < 0) {
     perror(“semctl”);     exit(-1);
}
```
     
- 信号灯P/V操作       
```cpp
    #include <sys/ipc.h>
    #include <sys/sem.h>  
        int  semop(int semid, struct sembuf  *sops, unsigned nsops)

参数：
    semid:     要操作的信号灯集id  
    sops:      描述对信号灯操作的结构体数组：  
                Struct  sembuf{
                  short  semnum;   //信号灯编号
                  short  sem_op;   //-1: P操作  1: V操作
                  short  sem_flg;  //
                };    
     nsops:    要操作的信号灯的个数    

解释：sem_flg：  
      0 ：        表示阻塞等待  
      IPC_NOWAIT：表示非阻塞操作  
      SEM_UNDO：  表示则当进程退出的时候会还原该进程的信号量操作，比如某进程做了P操作得到资源，但还没来得及做V操作时就异常退出了，此时，其他进程就只能都阻塞在P操作上，于是造成了死锁。若采取SEM_UNDO标志，就可以避免因为进程异常退出而造成的死锁    

返回值：
    成功时返回0，失败时返回-1

示例：  
Struct  sem_buf  buf[3];
Buf[0].semnum = 0;   //0号  
Buf[0].sem_op = -1;  //P操作   
Buf[0].sem_flg = 0;  //成功时返回  
Buf[1].semnum = 2;   //2号--> error, 应该连续存储
...
Semop(semid,  buf,  2) //三个信号灯，同时操作2个，则buf应该操作的是连续的两个，buf[0]=0和buf[1]=1
```
     
  (4) 代码练习      

需求：父子进程通过system V信号灯同步对共享内存的读写  
      父进程从键盘输入带有空格的字符串到共享内存  
      子进程删除字符串中的空格并打印  
      父进程输入quit后删除共享内存和信号灯集，程序结束  
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <sys/sem.h>

#define READ 0    //信号灯0作为读信号灯
#define WRITE 1   //信号灯1作为写信号灯
union semun       //self define semun
{
    int val;
    struct semid_ds *buf;
    unsigned short *arry;
    struct seminfo *_buf;
};

void init_sem(int semid,int s[], int n)
{
    int i;
    union semun myun;
    for(i=0; i<n; i++)
    {
        myun.val=s[i];
	semctl(semid, i, SETVAL, myun);
    }
}

void pv(int semid, int num, int op)
{
    struct sembuf buf;

    buf.sem_num = num; //operation which write or read
    buf.sem_op = op;
    buf.sem_flg = 0;
    semop(semid, &buf, 1);

}

int main()
{
    int shmid, semid, s[]={0, 1};
    pid_t pid;
    key_t key;
    char * shmaddr;

    //1. key
    key=ftok(".", 'q')

    //2. creat sharm memory
    shmid=shmget(key, 64, IPC_CREAT|0666)

    //3. creat signal set
    semid=semget(key, 2, IPC_CREAT|0666)

    //4. signal set init
    init_sem(semid, s, 2);
    
    //5. mapping 
    if((shmaddr=(char *)shmat(shmid, NULL, 0)) == (char *)-1)
    {
        perror("shmat error");
	goto _error2;
    }
    
    //6. creat process
    if((pid=fork()) < 0)
    {
        perror("fork error");
	goto _error2;
    }
    else if(pid ==0)//child process 
    {
       char *q, *p;
       while(1)
       {
           pv(semid, READ, -1); // p operation read  申请资源  
           p = q = shmaddr;
	   while(*q)
	   {
	       if(*q != ' ')
	       {
	           *p++ = *q;
	       }
	       q++;
	   }
	   *p = '\0';
	   printf("%s", shmaddr);
	   pv(semid, WRITE, 1); // v operation write  释放资源
       }
    }
    else //father process input
    {
       while(1)
       {
           pv(semid, WRITE, -1);//p operation write  申请资源  
	   printf("input characters with space:");
	   fgets(shmaddr, 64, stdin);
	   if(strcmp(shmaddr, "quit\n") ==0)
	   {
	       break;
	   }
	   pv(semid, READ, 1); //v operation read  释放资源  
       }
       kill(pid, SIGUSR1);
    }

_error2:
    semctl(semid, 0, IPC_RMID);   //如果映射共享内存失败，删除创建的信号灯集
_error1:
    shmctl(shmid, IPC_RMID, NULL);//如果创建信号灯失败，则删除共享内存

    return 0;
}
```
     
- 9. 进程间通信总结         








参考：  
(1) https://www.cnblogs.com/jiangknownet/p/14516630.html
(2) https://www.cnblogs.com/guxuanqing/p/5978161.html


- 10. 信号(signal) 

    A给B发送信号，B收到信号之前执行自己的代码，收到信号后，不管执行到程序的什么位置，都要暂停运行，去处理信号，处理完毕再继续运行。与硬件中断类型---异步模式。但信号是软件层面上实现的中断，早期常称为"软中断"  
     由于信号是通过软件方法实现，其实现手段导致信号有很强的延时性，但对于用户来说，这个延时时间非常短，不易察觉  
    
  (1)信号基本概念  
      
      a) 信号是在软件层上中断机制的一种模拟，是一种异步通信方式[异步：进程不需要进行特殊处理，随时可以接收信号]  
      b) linu内核通过信号通知用户进程，不同的信号类型代表不同的事件  
      c) 每个进程收到的所有信号，都是由内核负责发送的，内核处理  

  (2)信号状态  
      
      产生：  
          a) 按键产生，如：ctrl+c, ctrl+z, ctrl+\  
	  b) 系统调用产生，如: kill, raise, abort\, settimer  
	  c) 软件条件产生，如：定时器 alarm  
	  d) 硬件异常产生，如：非法访问内存(段错误)，除0，内存对齐出错[总线错误]  
	  e) 命令产生，如：kill命令  
      递达：  
          递达并且到达进程  
      未决:  
          产生和递达之间的状态，主要由于阻塞导致该状态  

  (3) 信号处理方式  
  
      a) 执行默认动作    
      b) 忽略[丢弃]  
      c) 捕捉[收到信号时，去调用用户提前设置好的处理函数]  
      
      常用信号[man 7 signal 可以查看]：
      
![image](https://user-images.githubusercontent.com/42632290/138557677-94c77b2d-cd4a-4dcd-9d28-16539661d2d7.png)  
![image](https://user-images.githubusercontent.com/42632290/138557685-bf0e2cb7-2542-4893-81c3-487ad4b6afcf.png)  
 
  (4) 信号四要素  

![image](https://user-images.githubusercontent.com/42632290/138557731-b3dbc42c-f39e-4f6e-af09-46bd58d1563b.png)  

    9和19号信号不能捕捉，不能忽略，甚至不能阻塞  
    
    linux内核的进程控制块PCB是一个结构体，task_struct除了包含进程id状态，工作目录，用户id，组id，文件描述符表，还包含了信号相关的信息，主要指阻塞信号集和未决信号集  
 
![image](https://user-images.githubusercontent.com/42632290/138557762-dd56c06e-7640-4e7c-b81e-89920847cdce.png)  
 
  (5) 信号函数  

    a) kill函数 -- 发送信号给进程或进程组  
```cpp
    #include<signal.h>
        int kill(pid_t pid,int signo);

参数：  
    pid：   接受信号的进程或者进程组。分为四种情况：  
            pid >0：将信号发送给进程ID为pid的进程  
            pid==0：将信号发送给与发送进程属于同一个进程组的所有其他进程（这些进程的进程组ID等于发送进程的进程组ID ），且发送进程必须要有权限向这些进程发送信号  
                    注意：这里的所有其他进程不包括内核进程和init进程  

            pid<0：将该信号发送给其进程组ID等于pid绝对值，且发送进程具有权限向其发送信号的所有进程。  
                    注意：这里的所有其他进程不包括内核进程和init进程  

            pid==-1：将信号发送给发送进程有权限向他们发送信号的所有进程。  
                    注意：这里的所有其他进程不包括内核进程和init进程
返回值：
    成功： 返回 0
    失败： 返回 -1
    
kill有两种经典应用场景：
    如果kill的signo参数为 0 ，则kill仍然执行正常的错误检查，但是不发送信号  
        - 这常用于确定一个特定的进程是否仍然存在。如果向一个并不存在的进程发送空信号，则kill返回 -1， errno设置为ESRCH  
        - 但是注意：UNIX系统每经过一定时间会重新使用销毁的进程的进程ID，所以可能存在某个给定进程ID的进程并不是你所期望的那个进程  
    如果调用kill为本进程产生信号，而且此信号是不被阻塞的，那么可以确保在kill返回之前，任何其他未决的、非阻塞信号（包括signo信号）都被递送到本进程  
```

    b) raise和abort函数 --- 函数向进程自己发送信号
    
```cpp
    #include<signal.h>
        int raise(int signo);    //给自己发送指定信号  

参数：  
    signo：要发送信号的信号编号  

返回值：  
    成功：返回 0  
    失败： 返回 -1  
    
    #include<stdlib.h>
        void abort(void);    //给自己发送异常终止信号 6[SIGABRT],终止并产生core文件  

```

    c) alarm 函数  -- 为进程设置定时器
    
```cpp

```


    d) settimer函数  --



