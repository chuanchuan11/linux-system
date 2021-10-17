> 进程间通信

- 1. 进程间通信方式  

IPC：进程间通信，通过内核提供的缓冲区进行数据交换的机制。  

早期UNIX进程间通信方式：   
    (1) 无名管道(pipe) --- 最简单  
    (2) 有名管道(fifo)  
    (3) 信号(signal) --- 携带信息量最小  
System v IPC  
    (4) 共享内存(share memry) --- 速度最快  
        mmap 实现  
        shmat 实现  
    (5) 消息队列(message queue)  
    (6) 信号灯集(semaphore set)  
    (7) 本地socket --- 最稳定  

- 2. 无名管道(pipe)  

    (1) 无名管道具特点   
        a) 只能用于具有亲缘关系的进程之间的通信（如：父子进程，兄弟进程）  
        b) 单工通信模式，即单项数据传递，具有固定的读端和写端  
        c) 它可以看成是一种特殊的文件，对于它的读写也可以使用普通的read、write 等函数，但是它不是普通的文件，不属于任何文件系统，只存在于内存中  
        
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



- 4. 有名管道(FIFO) 

- 5. 有名管道(FIFO) 


- 6. 有名管道(FIFO) 

- 7. 有名管道(FIFO) 

- 8. 有名管道(FIFO) 
