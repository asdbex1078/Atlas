malloc(0) 的时候，会分配32位。 => mem_chunck的结构(小内存是32，大的68) => 底层系统调用brk或者mmap
malloc(xx) 区分大小来分配内存

malloc:往操作系统分配堆内存，堆内存又属于虚拟内存，虚拟内存需要映射到真实内存，如果不够就报错






demon进程：setsid 持久化进程？不随着终端的关闭而关闭。能够使redis后台运行。脱离会话组，形成一个自己的进程组。隐藏的最大组？
nohup：屏蔽信号


redis = 数据结构 + 事件循环
redis所有操作均面向redisObject。节约空间至上，最终需要在节约时间和空间上做一个平衡点

// 元数据，描述数据的数据
redisObject{
    unsigned type:4;
    unsigned notused:2;
    unsigned encoding:4;
    unsigned lru:22;
    int refcount; =》引用计数法，标记有多少个地方在调用
    void *ptr; =〉 指向真实数据结构：5中基本数据类型
}

不仅是redis基本数据结构，也是所有语言的函数库必备：
- string
  - hash
  - set 
  - zset
  - list 

怎么在c语言里用一个数据结构来表示以上所有的数据结构？——redisObject 用一个指针指向即可
      1、当前的类型、2、lru的时间 3、当前的编码集 4、*ptr指向真实的
      将多个结构的共同特性抽取成redisObject

zip压缩：节省内存
空间局部性：数据的流向：CPU -> 缓存行 -> 内存。数组和链表对比：
1、数组的数据在一块，缓存行会携带旁边的数据给到cpu,节省时间；但链表物理上不连续，会导致缓存行失效
2、当数据很少的时候，可以压缩到一串连续的空间中进行空间局部性加速 =》redis压缩的核心，利用好缓存行
3、数据量过大毫无意义，一个缓存行64byte，大一点的128？所以这就是redis为什么建议使用小key





idea调试的本质：gdb调试


代码设计思想：高内聚，低耦合。=> 动态链接 => PIC实现了：PLT + GOT
高内聚，高耦合 => 静态链接

对比项：
1、java里，arrayOop??? 也有一个length记录着数组的长度，Arrays.length
2、redis里对象的结构，ziplist的结构