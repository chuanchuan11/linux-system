> 线程同步  

    线程间通信，线程共享同一进程的地址空间，优点是线程间通信很容易，通过全局变量交换数据，缺点是多个线程访问共享数据时需要同步或互斥机制  
    
    线程同步指一个线程发出某一功能调用时，在没有得到结果之前，该调用不返回。同时其它线程为保证数据一致性，不能调用该功能  

- 1. 互斥锁  

    (1) 互斥量使用步骤：
    
          - 初始化  
          - 加锁  
          - 执行逻辑--操作共享数据  
          - 解锁  
     **注意事项：加锁需要最小粒度，不要一直占用临界区！**  

    (2) 相关函数  

```cpp
    #include<pthread.h>  
        int pthread_mutex_init(pthread_mutex_t *restrict mutex, const pthread_mutexattr_t *restrict attr);  //初始化锁  
        int pthread_mutex_destroy(pthread_mutex_t *mutex);  //销毁锁  

参数：
    mutex：  待初始化/释放的互斥量的地址  
    attr：   互斥量的属性。如果为NULL，那么互斥量设置为默认属性  
    
返回值：
    成功：返回0  
    失败： 返回错误编号 
    
注意：
    使用互斥量之前必须初始化：  
        - 如果是动态分配的互斥量（如通过malloc函数），则必须调用pthread_mutex_init函数进行初始化  
        - 如果是静态分配的互斥量，那么除了调用pthread_mutex_init函数来初始化，也可以将它设置为常量PTHREAD_MUTEX_INITALIZER来初始化  
    如果是动态分配的互斥量，那么在free释放内存之前必须调用pthread_mutex_destroy函数来销毁互斥量。该函数会释放在动态初始化互斥量时动态分配的资源  
    
    #include<pthread.h>  
        int pthread_mutex_lock(pthread_mutex_t *mutex);     //加锁  
        int pthread_mutex_trylock(pthread_mutex_t *mutex);  //尝试加锁  
        int pthread_mutex_unlock(pthread_mutex_t *mutex);   //解锁  
        
参数：  
    mutex：待加锁/解锁的互斥量的地址  
    
返回值：
    成功：返回0  
    失败： 返回错误编号  
    
用法：  
    pthread_mutex_lock    如果互斥量已经上锁，则调用线程将阻塞直到互斥量解锁  
    pthread_mutex_unlock  用于对互斥量进行解锁   
    pthread_mutex_trylock 尝试对互斥量进行加锁  
        - 如果调用时，互斥量处于未锁定状态，那么函数将锁住互斥量并返回0    
        - 如果调用时，互斥量处于锁定状态，则函数调用失败，立即返回EBUSY而不是阻塞   
        
   #include<pthread.h>
   #include<time.h>
        int pthread_mutex_timedlock(pthread_mutex_t *restrict mutex, const struct timespec *restrict tsptr);  
   
参数：  
    mutex：  待加锁的互斥量的地址  
    tsptr：  指向一个timespec的指针，该timepsec指定了一个绝对时间（并不是相对时间，比如10秒）  
    
返回值：  
    成功：返回0  
    失败： 返回错误编号   
   
注意：  
    pthread_mutex_timedlock被调用时：  
        - 如果互斥量处于未锁定状态，那么函数将锁住互斥量并返回0  
        - 如果互斥量处于锁定状态，那么函数将阻塞到tsptr指定的时刻。在到达超时时刻时，pthread_mutex_timedlock不再试图对互斥量进行加锁，而是返回错误码ETIMEOUT  
    可以使用clock_gettime函数获取timespec结构表示的当前时间。但是目前并不是所有平台都支持这个函数。因此也可以用gettimeofday函数获取timeval结构表示的当前时间，然后将这个时间转换为timespec结构  
```

    示例：

```cpp
    #include<stdio.h>
    #include<unistd.h>
    #include<pthread.h>
    #include<stdlib.h>

    pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
    int sum=0;

    void * thr1 (void * arg){
        while(1){
                //先上锁
                pthread_mutex_lock(&mutex);

            	  //临界区
                printf("hello");
                sleep(rand()%3);       //这种函数尽量不要加在临界区，可以加快效率  
                printf("world\n");
            
                //释放锁
                pthread_mutex_unlock(&mutex);
                sleep(rand()%3);
        }
    }
    void *thr2(void*arg){
        while(1){
                //先上锁
                pthread_mutex_lock(&mutex);

                printf("HELLO");
                sleep(rand()%3);
                printf("WORLD\n");
                //释放锁
                pthread_mutex_unlock(&mutex);
                sleep(rand()%3);    //如果sleep，释放锁后马上再次占用，会导致一直占用临界区，其他线程无法访问  
        }
    }

    int main(){
        pthread_t tid[2];
        pthread_create(&tid[0],NULL,thr1,NULL);
        pthread_create(&tid[1],NULL,thr2,NULL);
        pthread_join(tid[0],NULL);

        pthread_join(tid[1],NULL);
        return 0;
    }
```


- 2. 死锁  

        a) 同一个互斥量加锁两次   
           第二次lock不会成功，会阻塞，第一次lock必须由自己unlock  
           解决：锁使用简单点 或 trylock  
           
        b) 两把锁
           拿着A锁请求B锁，拿着B锁请求A锁  
           解决：pthread_mutex_trylock拿不到所有锁释放已经占有的锁 或 规定不同线程访问时申请锁的顺序一致    


- 3. 读写锁  

        a) 读写锁特点：读共享，写独占，写优先级高

        b) 读写锁仍然是一把锁，有不同的状态：
           
               未加锁  
               读锁  
               写锁  
        c) 场景：
           线程A加写锁成功，线程B请求读锁：B堵塞  
           线程A持有读锁，线程B请求写锁：B阻塞  
           线程A持有读锁，线程B请求读锁：B加锁成功  
           线程A持有读锁，线程B请求写锁，线程C请求读锁：BC阻塞；A释放后B加锁；B释放后C加锁
           线程A持有写锁，线程B请求读锁，线程C请求写锁：BC阻塞；A释放后C加锁；C释放后B加锁
           
        d) 读写锁使用场景适合读的线程多

```cpp
    #include<pthread.h>
        int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock, const pthread_rwlockattr_t *restrict attr);  
        int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);  

参数：
    rwlock：   待初始化/销毁的读写锁的地址  
    attr：     读写锁的属性。如果为NULL，那么读写锁设置为默认属性  
    
返回值：  
    成功： 0  
    失败： 返回错误编号  
    
    #include<pthread.h>
        int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);  
        int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);  
        int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);  

参数：
    rwlock：待加锁/解锁的读写锁的地址  
    
返回值： 
    成功： 0
    失败： 返回错误编号  
    
用法：
    pthread_rwlock_rdlock  加读锁。如果读写锁当前是未加锁的，或者是读锁定的，则加锁成功；如果读写锁当前是写锁定的，则阻塞线程  
    pthread_rwlock_wrlock  加写锁。如果读写锁当前是未加锁的，则加锁成功；如果读写锁当前是读锁定或者写锁定的，则阻塞线程  
    pthread_rwlock_unlock  解锁，无论读写锁当前状态是处于读锁定还是写锁定  
    
注意：有的实现对读写锁同时加读锁的数量有限制。并不是无限制的加读锁  

    #include<pthread.h>
        int pthread_rwlock_tryrdlock(pthread_rwlock_t *rwlock);  
        int pthread_rwlock_trywrlock(pthread_rwlock_t *rwlock);  

参数：
    rwlock：待加锁的读写锁的地址  
    
返回值：
    成功： 返回0  
    失败： 返回错误编号  
    
注意：当可以加锁时，这两个函数返回0。否则它们返回错误EBUSY而不是阻塞线程  

    #include<pthread.h>  
    #include<time.h>  
        int pthread_rwlock_timedrdlock(pthread_rwlock_t *rwlock, const struct timespect*restrict tsptr);   
        int pthread_rwlock_timedwrlock(pthread_rwlock_t *rwlock, const struct timespect*restrict tsptr);  
        
参数：
    rwlock：  待加锁的读写锁的地址
    tsptr：   指向一个timespec的指针，该timepsec指定了一个绝对时间（并不是相对时间，比如10秒）  
    
返回值：  
    成功： 返回0
    失败： 返回错误编号
    
这两个函数被调用时：
    如果允许加锁，那么函数将对读写锁加锁并返回0
    如果不允许加锁，那么函数将阻塞到tsptr指定的时刻。在到达超时时刻时，pthread_mutex_timedlock不再试图对读写锁进行加锁，而是返回错误码ETIMEOUT  
    
可以使用clock_gettime函数获取timespec结构表示的当前时间。但是目前并不是所有平台都支持这个函数。因此也可以用gettimeofday函数获取timeval结构表示的当前时间，然后将这个时间转换为timespec结构  
```

    示例：  
    
```cpp
    #include<stdio.h>  
    #include<unistd.h>  
    #include<pthread>  

    pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;  //初始化
    int beginnum = 1000;  

    void * thr_write(void *arg){
    while(1){
        pthread_rwlock_wrlock(&rwlock);
        printf("-%s--self--%lu--beginnum=%d\n",__FUNCTION__,pthread_self,++beginnum);
        usleep(2000);//模拟占用时间
        pthread_rwlock_wrlock(&rwlock);
        usleep(4000);
    }
    return NULL;
}

    void * thr_read(void *arg){
    while(1){
        pthread_rwlock_rdlock(&rwlock);
        printf("--%s--self--%lu--beginnum=%d\n",__FUNCTION__,pthread_self,beginnum);
        usleep(2000);//模拟占用时间
        pthread_rwlock_rdlock(&rwlock);
        usleep(2000);
    }
    return NULL;
}

    int main(){
    int n=8;i=0;
    pthread_t tid[8];
    for(i=0;i<5;i++){
        pthread_create(&tid[i],NULL,thr_read,NULL);
    }
    for(;i<8;i++){
        pthread_create(&tid[i],NULL,thr_read,NULL);
    }
    for(i=0;i<8;i++){
        pthread_join(tid[i],NULL);
    }
}
```

- 4. 条件变量  

    条件变量是不同于锁的另一种同步机制   






