> 文件io

1. 打开、创建与关闭文件

(1) open和openat函数：打开文件

```cpp  
    #include<fcntl.h>
    int open(const char* path,int oflag,.../*mode_t mode*/);
    int openat(int fd,const char*path,int oflag,.../*mode_t mode */);  
参数：  
    path:要打开或者创建文件的名字  
    oflag：用于指定函数的操作行为  
           O_RDONLY常量：文件只读打开  
           O_WRONLY常量：文件只写打开  
           O_RDWR常量：文件读、写打开  
           O_EXEC常量：只执行打开  
           O_SEARCH常量：只搜索打开（应用于目录）。本文涉及的操作系统都没有支持该常量  
    在上面五个常量中必须指定且只能指定一个。下面的常量是可选的（进行或运行）：  
           O_APPEND：每次写时都追加到文件的尾端  
           O_CLOEXEC：将FD_CLOEXEC常量设置为文件描述符标志  
           O_CREAT：若此文件不存在则创建它。在使用此选项时，需要同时说明参数mode（指定该文件的访问权限）  
           O_DIRECTORY：若path引用的不是目录，则出错  
           O_EXCL：若同时指定了O_CREAT时，且文件已存在则出错。根据此可以测试一个文件是否存在。若不存在则创建此文件。这使得测试和创建两者成为一个原子操作  
           O_NOCTTY：若path引用的是终端设备，则不将该设备分配作为此进程的控制终端  
           O_NOFOLLOW：若path引用的是一个符号链接，则出错  
           O_NONBLOCK：如果path引用的是一个FIFO、一个块特殊文件或者一个字符特殊文件，则文件本次打开操作和后续的 I/O 操作设为非阻塞模式。  
           O_SYNC：每次 write 等待物理 I/O 完成，包括由 write 操作引起的文件属性更新所需的 I/O  
           O_TRUNC： 如果此文件存在，且为O_WRONLY或者O_RDWR成功打开，则将其长度截断为0  
           O_RSYNC：使每一个read操作等待，直到所有对文件同一部分挂起的写操作都完成。  
           O_DSYNC：每次 write 等待物理 I/O 完成，但不包括由 write 操作引起的文件属性更新所需的 I/O  
    mode：文件访问权限。文件访问权限常量在 <sys/stat.h> 中定义，有下列九个：  
           S_IRUSR：用户读  
           S_IWUSR：用户写  
           S_IXUSR：用户执行  
           S_IRGRP：组读  
           S_IWGRP：组写  
           S_IXGRP：组执行  
           S_IROTH：其他读  
           S_IWOTH：其他写  
           S_IXOTH：其他执行  
    对于openat函数，被打开的文件名由fd和path共同决定：  
           如果path指定的是绝对路径，此时fd被忽略。openat等价于open  
           如果path指定的是相对路径名，则fd是一个目录打开的文件描述符。被打开的文件的绝对路径由该fd描述符对应的目录加上path组合而成  
           如果path是一个相对路径名，而fd是常量AT_FDCWD，则path相对于当前工作目录。被打开文件在当前工作目录中查找。  
返回值：  
    成功：返回文件描述符。  
    失败：返回 -1  
    由 open/openat 返回的文件描述符一定是最小的未使用的描述符数字  
```


(2) creat函数：创建一个新文件  

```cpp  
    #include<fcntl.h>  
    int creat(const char*path,mode_t mode);  
    
参数：  
    path:要创建文件的名字  
    mode：指定该文件的访问权限文件访问权限常量在 <sys/stat.h> 中定义，有下列九个：  
          S_IRUSR：用户读  
          S_IWUSR：用户写  
          S_IXUSR：用户执行  
          S_IRGRP：组读  
          S_IWGRP：组写  
          S_IXGRP：组执行  
          S_IROTH：其他读  
          S_IWOTH：其他写  
          S_IXOTH：其他执行  
返回值：    
    成功： 返回O_WRONLY打开的文件描述符  
    失败： 返回 -1  
该函数等价于open(path,O_WRONLY|O_CREAT|O_TRUNC,mode)。注意：  
    它以只写方式打开，因此若要读取该文件，则必须先关闭，然后重新以读方式打开。
    若文件已存在则将文件截断为0。
```

(3) close函数：关闭文件  

```cpp  
    #include<unistd.h>
    int close(int fd);
    
参数：  
    fd：待关闭文件的文件描述符  
返回值：  
    成功：返回 0  
    失败：返回 -1  
注意：  
    进程关闭一个文件会释放它加在该文件上的所有记录锁。  
    当一个进程终止时，内核会自动关闭它所有的打开的文件。  
```

2. 定位、读、写文件

(1) lseek函数：设置打开文件的偏移量

```cpp 
    #include<unistd.h>
    off_t lseek(int fd, off_t offset,int whence);
    
参数：  
    fd：打开的文件的文件描述符  
    whence：必须是 SEEK_SET、SEEK_CUR、SEEK_END三个常量之一  
    offset：  
        如果 whence 是 SEEK_SET，则将该文件的偏移量设置为距离文件开始处offset个字节  
        如果 whence 是 SEEK_CUR，则将该文件的偏移量设置为当前值加上offset个字节，offset可正，可负  
        如果 whence 是 SEEK_END，则将该文件的偏移量设置为文件长度加上offset个字节，offset可正，可负  
返回值：  
    成功： 返回新的文件偏移量  
    失败：返回 -1  
    
    每个打开的文件都有一个与其关联的“当前文件偏移量”。它通常是个非负整数，用于度量从文件开始处计算的字节数。  
通常读、写操作都从当前文件偏移量处开始，并且使偏移量增加所读写的字节数。注意：  

    a) 打开一个文件时，除非指定O_APPEND选项，否则系统默认将该偏移量设为0  
    b) 如果文件描述符指定的是一个管道、FIFO、或者网络套接字，则无法设定当前文件偏移量，则lseek将返回 -1 ，并且将 errno 设置为 ESPIPE。  
    c) 对于普通文件，其当前文件偏移量必须是非负值。但是某些设备运行负的偏移量出现。因此比较lseek的结果时，不能根据它小于0 就认为出错。要根据是否等于 -1 来判断是否出错。  
```

(2) read函数：读取文件内容

```cpp
    #include<unistd.h>  
    ssize_t read(int fd,void *buf,size_t nbytes);  
    
参数：  
    fd：打开的文件的文件描述符  
    buf：存放读取内容的缓冲区的地址（由程序员手动分配）  
    nbytes：期望读到的字节数  
    
返回值：  
    成功：返回读到的字节数，若已到文件尾则返回 0  
    失败：返回 -1  
    
读操作从文件的当前偏移量开始，在成功返回之前，文件的当前偏移量会增加实际读到的字节数。有多种情况可能导致实际读到的字节数少于期望读到的字节数：    
    读普通文件时，在读到期望字节数之前到达了文件尾端  
   a) 当从终端设备读时，通常一次最多读取一行（终端默认是行缓冲的）  
   b) 当从网络读时，网络中的缓存机制可能造成返回值小于期望读到的字节数  
   c) 当从管道或者FIFO读时，若管道包含的字节少于所需的数量，则 read只返回实际可用的字节数  
   d) 当从某些面向记录的设备（如磁带）中读取时，一次最多返回一条记录  
   e) 当一个信号造成中断，而已读了部分数据时  
```

(3) write函数：想文件写数据  

```cpp
    #include<unistd.h>  
    ssize_t write(int fd,const void *buf,size_t nbytes);  
    
参数：  
    fd：    打开的文件的文件描述符  
    buf：   存放待写的数据内容的缓冲区的地址（由程序员手动分配）  
    nbytes：期望写入文件的字节数  
返回值：  

    成功：返回已写的字节数  
    失败：返回 -1  
    
write的返回值通常都是与nbytes相同。否则表示出错。write出错的一个常见原因是磁盘写满，或者超过了一个给定进行的文件长度限制  
    对于普通文件，写操作从文件的当前偏移量处开始。  
    如果打开文件时指定了O_APPEND选项，则每次写操作之前，都会将文件偏移量设置在文件的当前结尾处。在一次成功写之后，该文件偏移量增加实际写的字节数。
```

(4) fcntl函数：改变已经打开的文件的属性  

```cpp  
    #include<fcntl.h>  
    int fcntl(int fd,int cmd,.../* int arg */);  
    
参数：
    fd：已打开文件的描述符  
    cmd：有下列若干种：  
         F_DUPFD：复制文件描述符 fd。新文件描述符作为函数值返回。它是尚未打开的文件描述符中大于或等于arg中的最小值。新文件描述符与fd共享同一个文件表项，但是新描述符有自己的一套文件描述符标志，其中FD_CLOEXEC文件描述符标志被清除  
         F_DUPFD_CLOEXEC：复制文件描述符。新文件描述符作为函数值返回。它是尚未打开的个描述符中大于或等于arg中的最小值。新文件描述符与fd共享同一个文件表项，但是新描述符有自己的一套文件描述符标志，其中FD_CLOEXEC文件描述符标志被设置  
         F_GETFD：对应于fd的文件描述符标志作为函数值返回。当前只定义了一个文件描述符标志FD_CLOEXEC  
         F_SETFD：设置fd的文件描述符标志为arg  
         F_GETFL：返回fd的文件状态标志。文件状态标志必须首先用屏蔽字 O_ACCMODE 取得访问方式位，然后与O_RDONLY、O_WRONLY、O_RDWR、O_EXEC、O_SEARCH比较（这5个值互斥，且并不是各占1位）。剩下的还有：O_APPEND、O_NONBLOCK、O_SYNC 、O_DSYNC、O_RSYNC、F_ASYNC、O_ASYNC  
         F_SETFL：设置fd的文件状态标志为 arg。可以更改的标志是： O_APPEND、O_NONBLOCK、O_SYNC、O_DSYNC、O_RSYNC、F_ASYNC、O_ASYNC  
         F_GETOWN：获取当前接收 SIGIO和SIGURG信号的进程 ID或者进程组 ID  
         F_SETOWN：设置当前接收 SIGIO和SIGURG信号的进程 ID或者进程组 ID为arg。若 arg是个正值，则设定进程 ID；若 arg是个负值，则设定进程组ID  
         F_GETLK、F_SETLK、F_SETLKW：获取/设置文件记录锁  
    arg：依赖于具体的命令  
    
返回值：
    成功： 依赖于具体的命令  
    失败： 返回 -1  
```

> 目录操作

(1) link/linkat函数：创建一个指向现有文件的硬链接  

```
    #include<unistd.h>
	  int link(const char *existingpath, const char *newpath);
	  int linkat(int efd, const char*existingpath, int nfd, const char *newpath, int flag);

参数：
		existingpath：现有的文件的文件名（新创建的硬链接指向它）  
		newpath：新创建的目录项   

返回值：
    如果newpath已存在，则返回出错  
    只创建newpath中的最后一个分量，路径中的其他部分应当已经存在  
			  > 假设 newpath为：/home/aaa/b/c.txt，则要求 /home/aaa/b已经存在，只创建c.txt  

对于linkat函数：  
    现有的文件名是通过 efd 和 existingpath 指定  
		    若 existingpath 是绝对路径，则忽略efd  
		    若 existingpath 是相对路径，则：  
		        若 efd=AT_FDCWD，则existingpath是相对于当前工作目录来计算  
		        若 efd 是一个打开的目录文件的文件描述符，则existingpath是相对于efd对应的目录文件  
    新建的文件名是通过nfd和newpath指定。   
		    若 newpath 是绝对路径，则忽略nfd  
		    若 newpath 是相对路径，则：  
		        若 nfd=AT_FDCWD，则newpath是相对于当前工作目录来计算  
		        若 nfd是一个打开的目录文件的文件描述符，则newpath是相对于nfd对应的目录文件  
    flag：当现有文件是符号链接时的行为：  
		    flag=AT_SYMLINK_FOLLOW：创建符号链接指向的文件的硬链接（跟随行为）  
		    flag=!AT_SYMLINK_FOLLOW:创建符号链接本身的硬链接（默认行为）  

返回值：  
		成功： 返回 0  
		失败： 返回 -1  


(2) unlink函数：删除一个现有的目录项

```cpp
    #include<unistd.h>
    int unlink(const char*pathname);
    int unlinkat(int fd,const char*pathname,int flag);
```

参数：
    pathname：现有的、待删除的目录项的完整路径名。  
		对于 unlinkat 函数, 现有的文件名是通过 fd 和 pathname 指定。  
		    若 pathname 是绝对路径，则忽略 fd  
		    若 pathname 是相对路径，则：  
		        若 fd=AT_FDCWD，则 pathname 是相对于当前工作目录来计算  
		        若 fd 是一个打开的目录文件的文件描述符，则 pathname 是相对于 fd 对应的目录文件  
		flag：  
        flag=AT_REMOVEDIR：可以类似于rmdir一样的删除目录  
        flag=!AT_REMOVEDIR:与unlink执行同样的操作  
返回值：  
		成功： 返回 0  
		失败： 返回 -1  

(3) remove函数：解除对一个目录或者文件的链接。

```cpp
    #include<stdio.h>
    int remove(const char *pathname);

参数:  
		pathname：文件名或者目录名  
返回值：  
		成功：返回0  
		失败：返回 -1  

	对于文件，remove功能与unlink相同；对于目录，remove功能与rmdir相同  
```

(4) symlink/symlinkat函数：创建一个符号链接    

```cpp 
    #include<unistd.h>
    int symlink(const char*actualpath,const char *sympath);
    int symlinkat(const char*actualpath,int fd,const char*sympath);
```

参数：
		actualpath：符号链接要指向的文件或者目录（可能尚不存在）  
		sympath：符号链接的名字  
		         > 二者不要求位于同一个文件系统中  

    对于 symlinkat 函数：
    符号链接的名字是通过 fd 和 sympath 指定  
        若 sympath 是绝对路径，则忽略 fd  
        若 sympath 是相对路径，则：  
            若 fd=AT_FDCWD，则 sympath 是相对于当前工作目录来计算  
            若 fd 是一个打开的目录文件的文件描述符，则 sympath 是相对于 fd 对应的目录文件  

返回值：
		- 成功： 返回 0  
		- 失败： 返回 -1  

(5) umask函数：设置文件模式创建屏蔽字  

```cpp
    #include <sys/stat.h>
    mode_t umask(mode_t cmask);
    
参数：  
    cmask:    访问权限位  
          S_IRUSR  用户读  
          S_IWUSR  用户写  
          S_IXUSR  用户执行  
          S_IRUSR  组读  
          S_IWUSR  组写  
          S_IXUSR  组执行  
          S_IRUSR  其他读  
          S_IWUSR  其他写  
          S_IXUSR  其他执行  
```

> 标准IO

(1) 标准IO库与文件IO区别  

    标准IO库处理很多细节，如缓冲区分片、以优化的块长度执行IO等  
          a) 文件IO函数都是围绕文件描述符进行。首先打开一个文件，返回一个文件描述符；后续的文件IO操作都使用该文件描述符  
          b) 标准IO库是围绕流进行的。当用标准IO库打开或者创建一个文件时，就有一个内建的流与之相关联  
          c) 标准IO库的函数很多都是以 f 开头，如 fopen、fclose  

(2) 操作系统对每个进程与定义了3个流，并且这3个流可以自动地被进程使用，他们都是定义在<stdio.h>中：  

    标准输入：预定义的文件指针为stdin，它内部的文件描述符就是STDIN_FILENO  
    标准输出：预定义的文件指针为stdout，它内部的文件描述符就是STDOUT_FILENO  
    标准错误：预定义的文件指针为stderr，它内部的文件描述符就是STDERR_FILENO  

(3) fopen函数：打开标准IO流  

```cpp
    #include<stdio.h>
    FILE *fopen(const char*restrict pathname, const char*restrict type);
    
参数：
    type：指定对该IO流的读写方式：  
          "r"： 为读打开  
          "w"： 写打开。若文件存在则把文件截断为0长；若文件不存在则创建然后写  
          "a"： 追加写打开；若文件存在每次都定位到文件末尾；若文件不存在则创建然后写  
          "r+"：为读和写打开  
          "w+"：若文件存在则文件截断为0然后读写；若文件不存在则创建然后读写  
          "a+"：若文件存在则每次都定位到文件末尾然后读写；若文件不存在则创建然后读写  
    创建文件时，无法指定文件访问权限位。POSIX默认要求为：S_IRUSR|S_IWUSR|S_IRGRP|S_IWGRP|S_IROTH|S_IWOTH  
    pathname：待打开文件的路径名  

返回值：
    成功： 返回文件指针  
    失败： 返回NULL  
```

(4) fclose：关闭一个打开的流  

```cpp
    #include<stdio.h>
    int fclose(FILE *fp);
    
参数：  
    fp：待关闭的文件指针  
    
返回值：
    成功： 返回 0  
    失败： 返回 -1  
    
    在该文件被关闭之前：  
        fclose会自动冲洗缓冲中的输出数据  
        缓冲区中的输入数据被丢弃  
        若该缓冲区是标准IO库自动分配的，则释放此缓冲区  
    当一个进程正常终止时（直接调用exit函数，或者从main函数返回）：  
        所有带未写缓存数据的标准IO流都被冲洗  
        所有打开的标准IO流都被关闭  
```

(5) getc/fgetc/getchar 函数：一次读一个字符：  

```cpp
    #include<stdio.h>  
    int getc(FILE*fp);  
    int fgetc(FILE*fp);  
    int getchar(void);  
     
参数：  
    fp：打开的文件对象指针  
    
返回值：  
    成功：则返回下一个字符  
    到达文件尾端：返回EOF  
    失败：返回EOF  
    
```

(6) putc/fputc/putchar函数：一次写一个字符

```cpp
    #include<stdio.h>
    int putc(int c,FILE*fp);
    int fputc(int c,FILE*fp);
    int putchar(int c);
    
参数：  
    c：待写字符转换成的整数值  
    fp：打开的文件对象指针  
    
返回值：
    成功：则返回 c  
    失败：返回EOF  
```

(7) fgets/gets函数：一次读一行字符：

```cpp
    #include<stdio.h>
    char *fgets(char *restrict buf,int n, FILE* restrict fp);
    char *gets(char *buf);
    
参数：  
    buf：存放读取到的字符的缓冲区地址  
    对于 fgets函数：  
        n：缓冲区长度  
        fp：打开的文件对象指针   
返回值：  
    成功：则返回buf  
    到达文件尾端：返回NULL  
    失败：返回NULL  
```

(8) fputs/puts函数：一次写一行字符：

```cpp
    #include<stdio.h>
    int fputs(const char* restrict str,FILE*restrict fp);
    int puts(const char*str);
    
参数：  
    str：待写的字符串  
    fp：打开的文件对象指针  
    
返回值：  
    成功：则返回非负值  
    失败：返回EOF  
```

(9) fread/fwrite函数：执行二进制读写IO

```cpp
    #include<stdio.h>
    size_t fread(void *restrict ptr, size_t size, size_t nobj, FILE *restrict fp);
    size_t fwrite(const void*restrict ptr, size_t size, size_t nobj, FILE *restrict fp);
    
参数：
    ptr:存放二进制数据对象的缓冲区地址  
    size：单个二进制数据对象的字节数（比如一个struct的大小）  
    nobj：二进制数据对象的数量  
    fp：打开的文件对象指针  
     
返回值：
    成功或失败： 读/写的对象数  
    对于读：如果出错或者到达文件尾端，则此数字可以少于nobj。此时应调用ferror或者feof来判断究竟是那种情况  
    对于写：如果返回值少于nobj，则出错  
```

(10) 格式化输出函数：

```cpp
    #include<stdio.h>
    int printf(const char *restrict format,...);
    int fprintf(FILE *restrict fp,const char*restrict format,...);
    int dprintf(int fd,const char *restrict format,...);
    int sprintf(char *restrict buf,const char*restrict format,...);
    int snprintf(char *restrict buf,size_t n,const char *restrict format,...);
参数：
    format,...：输出的格式化字符串    
    
    对于fprintf：  
        fp：打开的文件对象指针。格式化输出到该文件中  
    对于dprintf：  
        fd：打开文件的文件描述符。格式化输出到该文件中  
    对于sprintf:  
        buf：一个缓冲区的指针。格式化输出到该缓冲区中  
    对于snprintf:  
        buf：一个缓冲区的指针。格式化输出到该缓冲区中  
        n：缓冲区的长度。格式化输出到该缓冲区中  
返回值：  
    成功：返回输出字符数（不包含null字节）  
    失败：返回负数  
    
格式说明：%[flags][fldwidth][precision][lenmodifier]convtype

标志flags有：
    ' : 撇号，将整数按照千位分组字符
    - ： 在字段内左对齐输出
    +： 总是显示带符号转换的正负号
    ：空格。如果第一个字符不是正负号，则在其前面加一个空格
    #：指定另一种转换形式（如，对于十六进制格式，加 0x 前缀）
    0：添加前导0（而非空格） 进行填充
    
fldwidth：说明最小字段宽度。转换后参数字符如果小于宽度，则多余字符位置用空格填充。
    字段宽度是一个非负十进制数，或者是一个星号 *
    
precision：说明整型转换后最少输出数字位数、浮点数转换后小数点后的最少位数、字符串转换后最大字节数。
    精度是一个点.后跟随一个可选的非负十进制数或者一个星号*
    宽度和精度可以为*，此时一个整型参数指定宽度或者精度的值。该整型参数正好位于被转换的参数之前

lenmodifier：说明参数长度。可以为：
    hh：将相应的参数按照signed char或者unsigned char类型输出
    h：将相应的参数按照signed short或者unsigned short类型输出
    l：将相应的参数按照signed long或者unsigned long或者宽字符类型输出
    ll：将相应的参数按照signed longlong或者unsigned longlong类型输出
    j：intmax_t或者uintmax_t
    z：size_t
    t：ptrdiff_t
    L：long double
convtype：控制如何解释参数
    d或者i：有符号十进制
    o：无符号八进制
    u：无符号十进制
    x或者X：无符号十六进制
    f或者F：双精度浮点数
    e或者E：指数格式双精度浮点数
    g或者G：根据转换后的值解释为f、F、e、E
    a或者A：十六进制指数格式双精度浮点数
    c：字符（若带上长度修饰符l,则为宽字符）
    s：字符串（若带上长度修饰符l,则为宽字符）
    p：指向void的指针
    n：到目前位置，此printf调用输出的字符的数目将被写入到指针所指向的带符号整型中
    %：一个%字符
    C：宽字符，等效于lc
    S：宽字符串，等效于ls  
```

(11) 格式化输入函数：

```cpp
    #include<stdio.h>
    int scanf(const char*restrict format,...);
    int fscanf(FILE *restrict fp,const char *restrict format,...);
    int sscanf(const char *restrict buf,const char *restrict format,...);
    
参数：
    format,...：格式化字符串
    
    对于fscanf：
        fp：打开的文件对象指针。从流中读取输入
    对于sscanf：
        buf：一个缓冲区指针。从该缓冲区中读取输入
        
返回值：
    成功：返回赋值的输入项数
    提前到达文件尾端：返回EOF
    失败：返回EOF

转换说明的格式为：%[*][fldwidth][m][lenmodifier]convtype：
    *：用于抑制转换。按照转换说明的其余部分对输入进行转换，但是转换结果不存放在参数中而是抛弃
    fldwidth：说明最大宽度，即最大字符数
    lenmodifier：说明要转换结果赋值的参数大小。见前述说明
    convtype：类似前述说明。但是稍有区别：输入中的带符号的数值可以赋给无符号类型的变量
    m：用于强迫内存分配。当%c,%s时，如果指定了m，则会自动分配内存来容纳转换的字符串。同时该内存的地址会赋给指针类型的变量（即要求对应的参数必须是指针的地址）。同时要求程序员负责释放该缓冲区（通过free函数）  
    
  (11) dup和dup2函数：复制文件描述符   
    
```cpp
    #include <unistd.h>
        int dup(int oldfd);
	int dup2(int oldfd, int targetfd);

参数：  
    针对dup函数：用来复制oldfd所指的文件描述符。  
        当复制成功时：返回最小的尚未被使用过的文件描述符  
        失败时：      返回-1  
        返回的的新文件描述符和参数oldfd指向同一个文件，共享同一个数据结构，所有的锁定，读写指针和各项权限及标志位  
    针对dup2函数：与dup相似，但是允许调用者指定目标描述符的数值。
                  若参数nwefd已经被程序使用，则系统就会将newfd所指向的文件关闭，若newfd等于oldfd则返回newfd，而不关闭newfd所指向的文件  
        成功时：目标描述符[第二个参数]将变成源描述符[第一个参数]的复制品  
        失败：  返回-1  
	返回的的新文件描述符和参数oldfd指向同一个文件，共享同一个数据结构，所有的锁定，读写指针和各项权限及标志位  
	
使用场景：  
    经常用来重定向进程的stdin,stdout和stderr，示例：   

```
   
   ![image](https://user-images.githubusercontent.com/42632290/137614339-46a8ded9-5fc7-4e54-a7d3-1cf5e2770d1c.png)
