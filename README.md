# nodejs-memory

## V8的垃圾回收机制与内存限制

  * Node与V8

    > Node选择了V8引擎，基于事件驱动、非阻塞I/O模型。  

  * V8的内存限制

    > 64位系统约为1.4GB，32位系统约为0.7GB，在这样限制下，将会导致Node无法直接操作大内存对象，比如无法将一个2GB的文件读入内存中进行字符串分析处理，即使物理内存有32GB，这样在单个Node进程的情况下，计算机的内存资源无法得到充足的使用。要知晓V8为何限制了内存的用量，则需要回归到V8在内存使用上的策略。

  * V8的对象分配

    > 在V8中，所有的JS对象都是通过堆来进行分配的。

    __Node提供V8内存使用量查看方式：__
    
    ```js
    $ node
    $ process.memoryUsage();
    {
      rss: 18702336,
      heapTotal: 10295296,
      heapUsed:5409936
    }
    ```

    * heapTotal：已申请到的堆内存；
    * heapUsed：当前使用的量。

    * V8的堆示意图如下：

      ![Alt text](https://raw.githubusercontent.com/zqjflash/nodejs-memory/master/node_v8_heap.png)
    
    __JS声明变量并赋值时，所使用对象的内存就分配在堆中。如果已申请的堆空闲内存不够分配新的对象，将继续申请堆内存，直到对的大小超过V8的限制为止。__

    __至于V8为何要限制堆的大小，表层原因：V8最初为浏览器而设计，不太可能遇到用大量内存的场景。深层原因：V8的垃圾回收机制的限制。官方说法，以1.5GB的垃圾回收堆内存为例，V8做一次小的垃圾回收需要50毫秒以上，做一次非增量式的垃圾回收甚至要1秒以上。这是垃圾回收中引起JS线程暂停执行的时间，在这样时间花销下，应用的性能和响应能力都会直线下降。__

    __V8提供选择来调整内存大小的配置，需要在初始化时候配置生效，遇到Node无法分配足够内存给JS对象的情况，可以用如下办法来放宽V8默认内存限制。避免执行过程内存用的过多导致崩溃__

    ```js
    node --max-old-space-size=1700 index.js
    node --max-new-space-size=1024 index.js
    ```

  * V8的垃圾回收机制

    * V8主要的垃圾回收算法

      __V8垃圾回收策略主要基于分代式垃圾回收机制。现代的垃圾回收算法中按对象的存活时间将内存的垃圾回收进行不同的分代，然后分别对不同分代的内存施以更高效的算法。__

      * V8的内存分代

        __在V8中，主要将内存分为新生代和老生代，新生代的对象为存活时间较短的对象，老生代的对象为存活时间较长或常驻内存的对象，如下图：__

          ![Alt text](https://raw.githubusercontent.com/zqjflash/nodejs-memory/master/v8_generation.png)

        * V8堆的整体大小就是新生代所用内存空间加上老生代的内存空间。
        * V8分配内存大小，可以通过源码找到相关设置，Page::kPageSize的值为1MB。
        
        ```js
        // a multiple of Page::kPageSize
        `#`if defined(V8_TARGET_ARCH_X64)
        `#`define LUMP_OF_MEMORY ( 2 * MB)
            code_range_size_(512*MB),
        `#`else
        `#`define LUMP_OF_MEMORY MB
            code_range_size_(0),
        `#`endif
        `#`if defined(ANDROID)
            reserved_semispace_size_(4 * Max(LUMP_OF_MEMORY, Page::kPageSize)),
            max_semispace_size_(4 * Max(LUMP_OF_MEMORY, Page::kPageSize)),
            initial_semispace_size_(Page::kPageSize),
            max_old_generation_size_(192*MB),
            max_executable_size_(max_old_generation_size_),
        `#`else
            reserved_semispace_size_(8 * Max(LUMP_OF_MEMORY, Page::kPageSize)),
            max_semispace_size_(8 * Max(LUMP_OF_MEMORY, Page::kPageSize)),
            initial_semispace_size_(Page::kPageSize),
            max_old_generation_size_(700ul * LUMP_OF_MEMORY),
            max_executable_size_(256l * LUMP_OF_MEMORY),
        `#`endif
        ```

        __新生代内存由两个reserved_semispace_size_所构成，最大值在64位系统和32位系统上分别为32MB和16MB。__

    * V8堆内存的最大保留空间可以从下面代码中看出来，其公式为4 * reserved_semispace_size_ + max_old_generation_size_:

      ```js
      // Returns the maxmum amount of memory reserved for the heap. For
      // the young generation, we reserve 4 times the amount needed for a
      // semi space. The young generation consists of two semi spaces and
      // we reserve twice the amount need for those in order to ensure
      // that new space can be aligned to its size
      intptr_tMaxReserved() {
        return 4 * reserved_semispace_size_ + max_old_generation_size_;
      }
      ```

      __因此，默认情况下，V8堆内存的最大值在64位系统上为1464MB，32位系统上则为732MB。这个数值可以解释为何在64位系统下只能使用约1.4GB内存，在32位系统下只能使用约0.7GB内存。__

    * Scavenge算法

      __在分代基础上，新生代中的对象主要通过Scavenge算法进行垃圾回收。在Scavenge的具体实现中，主要采用了Cheney算法__

      > Cheney算法是一种采用复制的方式实现的垃圾回收算法。它将堆内存一分为二，每一部分空间称为semispace。在这两个semispace空间中，只有一个处于使用中，另一个处于闲置状态。处于使用状态的semispace空间称为From空间，处于闲置状态的空间称为To空间。当我们分配对象时，先是在From空间中进行分配。当开始进行垃圾回收时，会检查From空间中的存活对象，这些存活对象将被复制到To空间中，而非存活对象占用的空间将会被释放。完成复制后，From空间和To空间的角色发生兑换。简而言之，在垃圾回收过程中，就是通过将存活对象在两个semispace空间之间进行复制。

      * Scavenge的缺点是只能使用堆内存中的一半，这是由划分空间和复制机制所决定的。但Scavenge由于只复制存活的对象，并且对于生命周期短的场景存活对象只占少部分，所以它在时间效率上有优异的表现。

        __由于Scavenge是典型的牺牲空间换取时间的算法，所以无法大规模地应用到所有的垃圾回收中。但可以发现，Scavenge非常适合应用在新生代中，因为新生代中对象的生命周期较短，恰恰适合这个算法。__

      * V8堆内存示意图：

        ![Alt text](https://raw.githubusercontent.com/zqjflash/nodejs-memory/master/v8_heap_divide.png)

        __实际使用的堆内存是新生代的两个semispace空间大小和老生代所用内存大小之和。当一个对象经过多次复制依然存活时，它将会被认为是生命周期较长的对象。这种较长生命周期的对象随后会被移动到老生代中，采用新的算法进行管理。对象从新生代中移动到老生代中的过程称为晋升。__

      * 在单纯的Scavenge过程中，From空间中的存活对象会被复制到To空间中去，然后对From空间和To空间进行角色对换（又称翻转）。但在分代式垃圾回收前提下，From空间中的存活对象在复制到To空间之前需要进行检查。在一定条件下，需要将存活周期长的对象移动到老生代中，也就是完成对象晋升。

      * 对象晋升的条件主要有两个，一个是对象是否经历过Scavenge回收，一个是To空间的内存占用比超过限制。
      
        __在默认情况下，V8的对象分配主要集中在From空间中。对象从From空间中复制到To空间时，会检查它的内存地址来判断这个对象是否已经经历过一次Scavenge回收。如果已经经历过了，会将该对象从From空间复制到老生代空间中，如果没有，则复制到To空间中。这个晋升流程如图：5-4所示：__

        ![Alt text](https://raw.githubusercontent.com/zqjflash/nodejs-memory/master/semi_space_scavenge.png)

      * 另一个判断条件是To空间的内存占用比。当要从From空间复制一个对象到To空间时，如果To空间已经使用了超过25%，则这个对象直接晋升到老生代空间中，这个晋升的判断示意图如下图：
 
        ![Alt text](https://raw.githubusercontent.com/zqjflash/nodejs-memory/master/semi_space_to.png)

        __设置25%这个限制值的原因是当这次Scavenge回收完成后，这个To空间将变成From空间，接下来的内存分配将在这个空间中进行。如果占比过高，会影响后续的内存分配。__

  * 查看垃圾回收日志

## 高效使用内存

  * 作用域
  * 闭包
  * 小结

## 内存指标

  * 查看内存使用情况
  * 堆外内存
  * 小结

## 内存泄漏

  * 慎将内存当作缓存
  * 关注队列状态

## 内存泄漏排查

  * node-heapdump
  * node-memwatch
  * 小结

## 大内存应用

## 总结

## 参考资源
