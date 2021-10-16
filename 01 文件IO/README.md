> 文件io

1. 打开、创建与关闭文件

(1) open和openat函数：打开文件

```cpp
#include<fcntl.h>
int open(const char* path,int oflag,.../*mode_t mode*/);
int openat(int fd,const char*path,int oflag,.../*mode_t mode */);
```




