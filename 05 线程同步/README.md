> 线程同步  

    线程间通信，线程共享同一进程的地址空间，优点是线程间通信很容易，通过全局变量交换数据，缺点是多个线程访问共享数据时需要同步或互斥机制  
    
    线程同步指的是多个任务按照约定的先后次序相互配合完成一件事情  

- 1. 互斥锁  

    临界资源：  一次只允许一个进程或线程访问的共享资源  
    互斥机制： 任务访问临界资源前申请锁，访问完后释放锁  

![image](https://user-images.githubusercontent.com/42632290/139572827-179d977a-88b2-472c-8347-733543bf0ee4.png)

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
  
注意：  
    使用读写锁之前必须初始化：  
        如果是动态分配的读写锁（如通过malloc函数），则必须调用pthread_rwlock_init 函数进行初始化  
        如果是静态分配的读写锁，那么除了调用pthread_rwlock_init函数来初始化，也可以将它设置为常量PTHREAD_RWLOCK_INITALIZER来初始化  
        如果是动态分配的读写锁，那么在free释放内存之前必须调用pthread_rwlock_destroy函数来销毁读写锁。该函数会释放在动态初始化读写锁时动态分配的资源  
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

    条件变量是不同于锁的另一种同步机制，相较于mutex而言，条件变量可以减少竞争  
    
    如果直接使用mutex，除了生产者/消费者之间要竞争互斥量以外，消费者之间也需要竞争互斥量，但如果汇聚中没有数据，消费者之间竞争互斥锁是无意义的。有了条件变量机制后，只有生产者完成生产，才会引起消费者之间的竞争，提高了程序效率。  
    
    条件变量不是锁，要和互斥量组合使用，条件变量与互斥量一起使用时，允许线程以无竞争的方式等待指定的条件发生  

```cpp  
    #include<pthread.h>  
      int pthread_cond_init(pthread_cond_t *restrict cond, const pthread_condattr_t *restrict attr);  //初始化
      int pthread_cond_destroy(pthread_cond_t *cond);  //退出初始化  
      
参数：
    cond：  待初始化/销毁的条件变量的地址
    attr：  条件变量的属性。如果为NULL，那么条件变量设置为默认属性  
返回值：
    成功：  返回0
    失败：   返回错误编号      
  
注意：
    使用条件变量之前必须初始化  
      a) 如果是动态分配的条件变量（如通过malloc函数），则必须调用pthread_cond_init函数进行初始化  
      b) 如果是静态分配的条件变量，那么除了调用pthread_cond_init函数来初始化，也可以将它设置为常量PTHREAD_COND_INITALIZER来初始化  
      c) 如果是动态分配的条件变量，那么在free释放内存之前必须调用pthread_cond_destroy函数来销毁条件变量。该函数会释放在动态初始化条件变量时动态分配的资源  
      
      
    #include<pthread.h>
    #include<time.h>
        int pthread_cond_wait(pthread_cond_t *restrict cond, pthread_mutext_t *restrict mutex);    //等待指定的条件成立；如果指定的条件不成立，则线程阻塞  
        int pthread_cond_timedwait(pthread_cond_t *restrict cond, pthread_mutext_t *restrict mutex, const struct timespect*restrict tsptr);  //指定一个超时时刻。如果在超时时刻到来时，指定条件仍然不满足，则函数返回错误码ETIMEOUT  
        
参数：
    cond：  要等待的条件变量的地址  
    mutex： 与条件变量配套的互斥量的地址  
    tsptr： 指向一个timespec的指针，该timepsec指定了一个绝对时间（并不是相对时间，比如10秒）  
返回值：
    成功： 返回0  
    失败： 返回错误编号  
    
注意：  
    a) 互斥量mutex用于对条件变量cond进行保护。调用者在调用pthread_cond_wait/pthread_cond_timedwait函数之前，必须首先对mutex加锁（即mutex此时必须是处于已锁定状态） 
    
    b) pthread_cond_wait/pthread_cond_timedwait会计算指定的条件是否成立，如果不成立则会自动把调用线程放到等待条件的线程列表中并阻塞线程，然后对互斥量mutex进行解锁【条件成立是否指condition先被触发？，做测试callback先触发】 
【之所以mutex必须是已锁定状态，就是因为这里有“计算条件然后如果不满足条件就把进程投入休眠”的操作。这两步并不是原子的，所以需要利用条件变量保护起来】  

    c) 当从pthread_cond_wait/pthread_cond_timedwait函数返回时，返回之前互斥量mutex再次被锁住 
    
    d) pthread_cond_timedwait被调用时：
        如果指定条件成立，那么函数返回0
        如果指定条件不成立，那么函数将阻塞到tsptr指定的时刻。在到达超时时刻时，pthread_mutex_timedlock返回错误码ETIMEOUT
        【可以使用clock_gettime函数获取timespec结构表示的当前时间。但是目前并不是所有平台都支持这个函数。因此也可以用gettimeofday函数获取timeval结构表示的当前时间，然后将这个时间转换为timespec结构】思考：是否都为单调时间，是否会收到系统时间改变的影响，代码可以测试？【默认为绝对时间等待，当系统时间被修改时，等待函数会受到影响，可以使用pthread_condaddr_setclock(&cond.cattr, CLOCK_MONOTONIC)设置属性为相对时间，避免该问题】  
     
     e) 使用时候注意设置的纳秒级超时时间，与绝对时间基相加后超出秒级的时间，导致永远无法超时的情况[当纳秒级时间超出1s应进位到s级时间上，ns级单位永远不要超过1s]  
        
    #include<pthread.h>  
        int pthread_cond_signal(pthread_cond_t *cond);    //唤醒一个等待cond条件发生的线程  
        int pthread_cond_broadcast(pthread_cond_t *cond); //唤醒等待cond条件发生的所有线程  
        
参数：
    cond：该指针指向的条件变量状态变为：条件成立  
    
返回值：  
    成功： 返回0  
    失败： 返回错误编号  
```

    (经典)生产消费者模型示例：生产者产生数据，消费者获取数据  
          
          让消费者阻塞在条件变量上，而不是争抢锁上边[如果阻塞在锁上，不管是否有数据都会发生竞争，当没有数据时，退出后继续竞争]，阻塞在条件变量上则直接等待，直到被生产者唤醒再去竞争锁  

```cpp 
  #include <stdio.h>  
  #include <unistd.h>  
  #include <pthread.h>  
  #include <stdlib.h>  
  
  pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
  pthread_cond_t cond = PTHREAD_COND_INITIALZER;

  int beginnum = 1000;

  typedef struct _Proinfo{
      int num;
      struct _Prodinfo *next;
  }ProdInfo;

  ProdIndo * Head = NULL;

  void *thr_producter(void * arg)
  {
      //负责在链表添加数据  
      while(1){
          ProdInfo *prod = malloc(sizeof(ProdInfo));
          prod->num = beginnum++;
          printf("----%s-----self=%lu----%d \n", __FUNCTION__, pthread_self(), prod->num);
          pthread_mutex_lock(&mutex);
          //add to list
          prod->next = Head;
          Head = prod;
          pthread_mutex_unlock(&mutex);
          //发起通知  
          pthread_cond_signal(&cond);
          sleep(rand()%2);
      }
      return NULL;
  }
  
  void *thr_customer(void * arg)
  {
      ProdInfo *prod = NULL;
      while(1){
           //负责取链表数据 
          pthread_mutex_lock(&mutex);
          while(Head == NULL){  
              pthread_cond_wait(&cond, &mutex);  //在此之前必须先枷锁  
          }
          prod = Head;
          Head = Head->next;
          printf("----%s-----self=%lu----%d \n", __FUNCTION__, pthread_self(), prod->num);
          pthread_mutex_unlock(&mutex);
          sleep(rand()%4);
          free(prod);
          
      }
      return NULL;
  }
  
  int main()
  {
      pthread_t tid[3];  
      pthread_creat(&tid[0], NULL, thr_producter, NULL);  
      pthread_creat(&tid[1], NULL, thr_customer, NULL);
      pthread_creat(&tid[2], NULL, thr_customer, NULL);  
      
      pthread_join(tid[0], NULL);
      pthread_join(tid[1], NULL);
      pthread_join(tid[2], NULL);
      pthread_mutex_destroy(&mutex);  
      pthread_cond_destroy(&cond); 
  }
  
思考：当生产者比较多时候？
      当消费者比较多时候？
      多个消费者都被唤醒只有一个取到数据，其他消费者什么逻辑？

```

- 5. posix信号量 

    进化版的互斥锁(1->N)  
    
    由于互斥锁的粒度比较大，如果我们希望在多个线程间对某一对象的部分数据进行共享，使用互斥锁是没办法实现的，只能将整个数据对象锁住。这样虽然达到了多线程操作共享数据时保证数据正确性的目的，却无形中导致线程的并发性下降。线程从并行执行，变成了串行执行。与直接使用单进程无异。[即同时允许几个线程访问共享数据]  
    
    信号量，是相对折中的一种处理方式，既能保证同步，数据不混乱，又能提高线程并发。 允许多线程访问共享资源  
  
    posix中定义了两类信号量：无名信号量[基于内存的信号量]和有名信号量，无名信号量一般用于线程之间的同步，有名信号量用于进程和线程都可以  
  
   (1) 信号量基本操作  
      
       sem_wait:   信号量大于0，则信号量--；信号量等于0，造成线程阻塞  
       sem_post:   将信号量++，同时唤醒阻塞在信号量上的线程  
  
  
```cpp
    #include <semaphore.h>
        int sem_init(sem_t *sem, int pshared, unsigned int value);
        int sem_destroy(sem_t *sem);

参数：
    sem:      要初始化/销毁的信号量    
    pshared:  0用于线程间，1用于进程间  
    value:    指定信号量的个数，规定sem不能<0
              信号量的初值，决定了占用信号量的线程的个数  
    
返回值：
    成功：  返回0  
    失败：  返回-1，同时设置errno  

    #include <semaphore.h>
        int sem_wait(sem_t *sem);      //v操作给信号量加锁--  
        int sem_trywait(sem_t *sem);   //尝试对信号量加锁--
        int sem_timedwait(sem_t *sem, const struct timespec *abs_timeout);  //限时对信号量加锁--  

参数：
    sem:      要操作的信号量    
    
返回值：
    成功：  返回0  
    失败：  返回-1，同时设置errno  

    #include <semaphore.h>  
        int post(sem_t *sem)  //p操作给信号量解锁++  

参数：
    sem:      要操作的信号量    
    
返回值：
    成功：  返回0  
    失败：  返回-1，同时设置errno  

```

    信号量实现生产者和消费者模型：  
    
```cpp
  #include <stdio.h>  
  #include <unistd.h>  
  #include <pthread.h>  
  #include <stdlib.h>  
  #include <semaphore.h>  

  sem_t blank, xfull;
  #define _SEM_CNT_ 5
  
  int queue[_SEM_CNT_];  //模拟饼框  
  int beginnum = 100;
  
  void *thr_producter(void *arg)
  {
      int i = 0;
      while(1){
          sem_wait(&blank); //申请资源blank  
          printf("----%s-----self=%lu----%d \n", __FUNCTION__, pthread_self(), beginnum);
          queue[(i=++)%_SEM_CNT_] = beginnum++;
          sem_post(&xfull);  --
          sleep(rand()%3);
      }
      return NULL;
  }
  
  void *thr_producter(void *arg)
  {
      int i = 0;
      int num = 0;
      while(1){
          sem_wait(&xfull); 
          num = queue[(i=++)%_SEM_CNT_];  
          printf("----%s-----self=%lu----%d \n", __FUNCTION__, pthread_self(), num);
          sem_post(&blank);   ++
          sleep(rand()%3);
      }
      return NULL;
  }

  int main()
  {
      sem_init(&blank, 0, _SEM_CNT_); 
      sem_init(&xfull, 0, 0);   //消费者一开始初始化默认没有产品  
      
      pthread_t tid[2];
      pthread_creat(&tid[0], NULL, thr_producter, NULL);  
      pthread_creat(&tid[1], NULL, thr_customer, NULL);
      
      pthread_join(tid[0], NULL);
      pthread_join(tid[1], NULL);
      sem_destroy(&blank);
      sem_destroy(&xfull);
  }
```

- 6. 进程间同步   

    a) 互斥量  
    
        进程间也可以使用互斥锁，来达到同步的目的。但应在pthread_mutex_init初始化之前，修改其属性为进程间共享。mutex的属性修改函数主要有以下几个：
        
```cpp
    pthread_mutexattr_t mattr; //用于定义mutex锁的属性  
    pthread_mutexattr_init(pthread_mutexattr_t *attr);    //初始化一个mutex属性对象  
    pthread_mutexattr_destroy(pthread_mutexattr_t *attr); //销毁mutex属性对象，而非销毁锁  
    pthread_mutexattr_setpshared(pthread_mutexattr_t *attr, int pshared); 
          pshared:  PTHREAD_PROCESS_PRIVATE[mutex的默认属性值即为线程锁]  
                    PTHREAD_PROCESS_SHARED [用于进程间互斥]
```

    b) 文件锁  

    借助fcntl函数来实现锁机制。操作文件的进程没有获得锁时，可以打开，但无法执行read和write操作  
    
    fcntl函数：获取、设置文件访问控制属性    
        int fcntl(int fd, int cmd, .../*arg*/);    
    
    参数2：  
        F_SETK(struct flock *);  设置文件锁[trylock]  
        F_SETLKW(struct flock *);设置文件锁[lock]   
        F_GETLK(struct flock *); 获取文件锁  
    参数3：  
        struct flock{  
            ...  
            short l_type;   锁的类型：F_RDLCK、F_WRLCK、F_UNLCK  
            short l_whence; 偏移位置：SEEK_SET、SEEK_CUR、SEEK_END
            off_t l_start;  起始位置：1000
            off_t l_len;    长度：0表示整个文件加锁
            pid_t l_pid;    持有该锁的进程ID：F_GETLK only
        }  

```cpp
    #include <sys/stat.h>  
    #include <fcntl.h>  
    #include <stdlib.h>  
    
    #define _FILE_NAME_ "/home/itheima/temp.lock"
    
    int main()
    {
        int fd = open(_FILE_NAME_， O_RDWR|O_CREAT, 0666);
        
        struct flock lk;
        lk.l_type = F_WRLCK;
        lk.l_whence = SEEK_SET;
        lk.l_start = 0;
        lk.l_len = 0;
        
        if(fcntl(fd, F_SETLK, &lk) < 0)
        {
            perror("get lock err");
            exit(1);
        }
        
        while(1){
            printf("i am alive \n");
            sleep(1);
        }
        
        return 0;
    }
```

