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

        __设置25%这个限制值的原因是当这次Scavenge回收完成后，这个To空间将变成From空间，接下来的内存分配将在这个空间中进行。如果占比过高，会影响后续的内存分配。对象晋升后，将会在老生代空间中作为存活周期较长的对象来对待，接受新的回收算法处理。__

      * Mark-Sweep & Mark-Compact

        > 对于老生代中的对象，由于存活对象占较大比重，再采用Scavenge的方式会有两个问题：一个是存活对象较多，复制存活对象的效率将会很低；另一个问题依然是浪费一半空间的问题。为此，V8在老生代中主要采用Mark-Sweep和Mark-Compact相结合的方式进行垃圾回收。

        * Mark-Sweep是标记清除的意思，它分为标记和清除两个阶段。与Scavenge相比，Mark-Sweep并不将内存空间划分为两半，所以不存在浪费一半空间的行为。与Scavenge复制活着的对象不同，Mark-Sweep在标记阶段遍历堆中所有对象，并标记活着的对象，在随后的清除阶段中，只清除没有被标记的对象。可以看出，Scavenge中只复制活着的对象，而Mark-Sweep只清理死亡对象。活对象在新生代中只占较小部分，死对象在老生代中只占较小部分，这是两种回收方式能高效处理的原因。
        * 下图为Mark-Sweep在老生代空间中标记的示意图，黑色部分标记为死亡对象
        
          ![Alt text](https://raw.githubusercontent.com/zqjflash/nodejs-memory/master/mark_sweep.png)

        * Mark-Sweep最大的问题是在进行一次标记清除回收后，内存空间会出现不连续的状态。这种内存碎片会对后续的内存分配造成问题，因为很可能出现需要分配一个大对象的情况，这时所有的碎片空间都无法完成此次分配，就会提前触发垃圾回收，而这次回收是不必要的。
        * 为了解决Mark-Sweep的内存碎片问题，Mark-Compact被提出来。Mark-Compact是标记整理的意思，是在Mark-Sweep的基础上演变而来的。它们的差别在于对象在标记为死亡后，在整理的过程中，将活着的对象往一端移动，移动完成后，直接清理掉边界外的内存。图5-7为Mark-Compact完成标记并移动存活对象后的示意图，白色格子为存活对象，深色格子为死亡对象，浅色格子为存活对象移动后留下的空洞。

          ![Alt text](https://raw.githubusercontent.com/zqjflash/nodejs-memory/master/mark_compact.png)

        * 完成移动后，就可以直接清除最右边的存活对象后面的内存区域完成回收。
        * Mark-Sweep、Mark-Compact、Scavenge三种主要垃圾回收算法的简单对比
        <table>
            <thead>
                <th>回收算法</th><th>Mark-Sweep</th><th>Mark-Compact</th><th>Scavenge</th>
            </thead>
            <tbody>
                <tr><td>速度</td><td>中等</td><td>最慢</td><td>最快</td></tr>
                <tr><td>空间开销</td><td>少（有碎片）</td><td>少（无碎片）</td><td>双倍空间（无碎片）</td></tr>
                <tr><td>是否移动对象</td><td>否</td><td>是</td><td>是</td></tr>
            </tbody>
        </table>

         * 从表格上看，Mark-Sweep和Mark-Compact之间，由于Mark-Compact需要移动对象，所以它的执行速度不可能很快，所以在取舍上，V8主要使用Mark-Sweep，在空间不足以对从新生代中晋升过来的对象进行分配时才使用Mark-Compact。

         * Incremental Marking

           * 为了避免出现js应用逻辑与垃圾回收器看到的不一致的情况，垃圾回收的3种基本算法都需要将应用逻辑暂停下来，待执行完垃圾回收后再恢复执行应用逻辑，这种行为被称为“全停顿”（stop-the-world）。在V8的分代式垃圾回收中，一次小垃圾回收只收集新生代，由于新生代默认配置得较小，且其中存活对象通常较少，所以即便它是全停顿的影响也不大。但V8的老生代通常配置得较大，且存活对象较多，全堆垃圾回收（full垃圾回收）的标记、清理、整理等动作造成的停顿就会比较可怕，需要设法改善。
           * 为了降低全堆垃圾回收带来的停顿时间，V8先从标记阶段入手，将原本要一口气停顿完成的动作改为增量标记（incremental marking），也就是拆分为许多小“步进”，每做完一“步进”就让js应用逻辑执行一小会，垃圾回收与应用逻辑交替执行直到标记阶段完成。

            ![Alt text](https://raw.githubusercontent.com/zqjflash/nodejs-memory/master/incremental_marking.png)

           * V8在经过增量标记的改进后，垃圾回收的最大停顿时间可以减少到原本的1/6左右。
           * V8后续还引入了延迟清理（lazy sweeping）与增量式整理（incremental compaction），让清理与整理动作也变成增量式的。同时还计划引入并行标记与并行清理，进一步利用多核性能降低每次停顿的时间。

      * 小结

        > 从V8的自动垃圾回收机制的设计角度可以看到，V8对内存使用进行限制的缘由。新生代设计为一个较小的内存空间是合理的，而老生代空间过大对于垃圾回收并无特别意义。V8对内存限制的设置对于Chrome浏览器这种每个选项卡页面使用一个V8实例而言，内存的使用是绰绰有余，对于Node编写的服务器端来说，内存限制也并不影响正常场景下的使用。但是对于V8的垃圾回收特点和js在单线程上的执行情况，垃圾回收是影响性能的因素之一。想要高性能执行效率，需要注意让垃圾回收尽量少地进行，尤其是全堆垃圾回收。

        __以Web服务器中的会话实现为例，一般通过内存来存储，但在访问量大的时候会导致老生代中的存活对象骤增，不仅造成清理/整理过程费时，还会造成内存紧张，甚至溢出__

  * 查看垃圾回收日志

    > 查看垃圾回收日志的方式主要是在启动时添加--trace_gc参数。在进行垃圾回收时，将会从标准输出中打印垃圾回收的日志信息。通过分析垃圾回收日志，可以了解垃圾回收的运行状况，找出垃圾回收的哪些阶段比较耗时。

    * 示例：循环创建对象并将其分配给局部变量a，文件名:test01.js
    
    ```js
    for (var i = 0; i < 1000000; i++) {
        var a = {};
    }
    ```

    __$ node --prof test01.js__

    * 这将会在目录下得到一个v8.log日志文件。该日志文件基本不具备可读性，内容大致如下：

      ![Alt text](https://raw.githubusercontent.com/zqjflash/nodejs-memory/master/isolate_v8_log.png)

    V8提供了linux-tick-processor工具用于统计日志信息。该工具可以从Node源码的deps/v8/tools目录下找到，Windows下对应命令文件为windows-tick-processor.bat。将该目录添加到环境变量PATH中。


## 高效使用内存

  * 作用域

    > 提到如何触发垃圾回收，第一个要介绍的是作用域（scope）。在js中能形成作用域的有函数调用、with以及全局作用域。

    ```js
    var foo = function() {
        var local = {};
    };
    ```

    > foo()函数在每次被调用时会创建对应的作用域，函数执行结束后，该作用域将会销毁。同时作用域中声明的局部变量分配在该作用域上，随作用域的销毁而销毁。只被局部变量引用的对象存活周期较短。在这个示例中，由于对象非常小，将会分配在新生代中的From空间中。在作用域释放后，局部变量local失效，其引用的对象将会在下次垃圾回收时被释放。

    __以上是最基本的内存回收过程。__

    1. 标识符查找

        与作用域相关的即是标识符查找。所谓标识符，可以理解为变量名。在下面的代码中，执行bar()函数时，将会遇到local变量：
      
        ```js
        var bar = function() {
          console.log(local);
        };
        ```
        js在执行时会去查找该变量定义在哪里。它最先查找的是当前作用域，如果在当前作用域中无法找到该变量的声明，将会向上级的作用域里查找，直到查到为止。

    2. 作用域链

        在下面的代码中：

        ```js
        var foo = function() {
          var local = 'local var';
          var bar = function() {
            var local = 'another var';
            var baz = function() {
              console.log(local);
            };
            baz();
          };
          bar();
        };
        foo();
        ``` 
        local变量在baz()函数形成的作用域里查找不到，继而将在bar()的作用域里寻找。如果去掉上述代码bar()中的local声明，将会继续向上查找，一直到全局作用域。这样的查找方式使得作用域像一个链条。由于标识符的查找方向是向上的，所以变量只能向外访问，而不能向内访问。如下图：

          ![Alt text](https://raw.githubusercontent.com/zqjflash/nodejs-memory/master/function_scope.png)

        当我们在baz()函数中访问local变量时，由于作用域中的变量列表中没有local，所以会向上一个作用域中查找，接着会在bar()函数执行得到的变量列表中找到了一个local变量的定义，于是使用它。尽管在上一层的作用域中也存在local的定义，但是不会继续查找了。如果查找一个不存在的变量，将会一直沿着作用域链查找到全局作用域，最后抛出未定义错误。

    3. 变量的主动释放

      > 如果变量是全局变量（不通过var声明或定义在global变量上），由于全局作用域需要直到进程退出才能释放，此时将导致引用的对象常驻内存（常驻在老生代中）。如果需要释放常驻内存的对象，可以通过delete操作来删除引用关系。或者将变量重新赋值，让旧的对象脱离引用关系。举个示例，老生代内存清除和整理的过程中，会被回收释放
      
      ```js
      global.foo = "I am global object";
      console.log(global.foo); // => "I am global object"
      delete global.foo;
      // 或者重新赋值
      global.foo = undefined; // or null
      console.log(global.foo); // => undefined
      ```

      __同样，如果在非全局作用域中，想主动释放变量引用的对象，也可以通过这样的方式。虽然delete操作和重新赋值具有相同的效果，但是在V8中通过delete删除对象的属性有可能干扰V8的优化，所以通过赋值方式解除引用更好。__

  * 闭包

    * 下面代码，local会得到未定义的异常：

      ```js
      var foo = function() {
        (function() {
          var local = "局部变量";
        }());
        console.log(local);
      };
      ```

      __在js中，实现外部作用域访问内部作用域中变量的方法叫做闭包（closure）。这得益于高阶函数的特性：函数可以作为参数或者返回值。__

      ```js
      var foo = function() {
        var bar = function() {
          var local = "局部变量";
          return function() {
            return local;
          };
        };
        var baz = bar();
        console.log(baz());
      };
      ```

      __在bar()函数执行完成后，局部变量local将会随着作用域的销毁而被回收。但是注意这里的特点在于返回值是一个匿名函数，且这个函数中具备了访问local的条件。虽然在后续的执行中，在外部作用域中还是无法直接访问local，但是若要访问它，可以通过中间函数来过渡。__

      __闭包是js的高级特性，利用它可以产生很多巧妙的效果。它的问题在于，一旦有变量引用这个中间函数，这个中间函数将不会释放，同时也会使原始的作用域得不到释放，作用域中产生的内存占用也不会得到释放。除非不再有引用，才会逐步释放。__

  * 小结

    __在正常js执行中，无法立即回收的内存有闭包和全局变量引用这两种情况。由于V8的内存限制，要十分小心此类变量是否无限制地增加，因为它会导致老生代中的对象增多。__

## 内存指标

  * 查看内存使用情况

    __前面用到的process.memoryUsage()可以查看内存使用情况。除此之外，os模块中totalmem()和freemem()方法也可以查看内存使用情况。__

    * 查看进程的内存占用
    
        调用process.memoryUsage()可以看到Node进程的内存占用情况，示例代码如下：

      ```js
      `$` node
      `>` process.memoryUsage()
      {
        rss: 18821120,
        heapTotal: 10295296,
        heapUsed: 5013664
      }
      ```

      __rss是resident set size的缩写，即进程的常驻内存部分。进程的内存总共有几部分，一部分是rss，其余部分在交换区（swap）或者文件系统（filesystem）中。__

      __除了rss外，heapTotal和heapUsed对应的是V8的堆内存信息。heapTotal是堆中总共申请的内存量，heapUsed表示目前堆中使用中的内存量。这3个值的单位都是字节。为了更好地查看效果，我们格式化一下输出结果：__

      ```js
      var showMem = function() {
        var mem = process.memoryUsage();
        var format = function(bytes) {
          return (bytes / 1024 / 1024).toFixed(2) + ' MB';
        };
        console.log('Process: heapTotal ' + format(mem.heapTotal) + ' heapUsed ' + format(mem.heapUsed) + ' rss ' + format(mem.rss));
        console.log('---------------------------------------------------------------------------');
      };
      ```

      同时，写一个方法用于不停地分配内存但不释放内存，相关代码如下：

      ```js
      var useMem = function() {
        var size = 20 * 1024 * 1024;
        var arr = new Array(size);
        for (var i = 0; i < size; i++) {
          arr[i] = 0;
        }
        return arr;
      };
      var total = [];
      for (var j = 0; j < 15; j++) {
        showMem();
        total.push(useMem());
      }
      showMem();
      ```

      将以上代码存为outofmemory.js并执行它，得到的输出结果如下：

      `$` node outofmemory.js

      ![Alt text](https://raw.githubusercontent.com/zqjflash/nodejs-memory/master/outofmemory.png)

      可以看到，每次调用useMem都导致了3个值的增长。在接近1500MB的时候，无法继续分配内存，然后进程内存溢出了，连循环体都无法执行完成，仅执行了9次。

    * 查看系统的内存占用

      > 与process.memoryUsage()不同的是，os模块中的totalmem()和freemem()这两个方法用于查看操作系统的内存使用情况，它们分别返回系统的总内存和闲置内存，以字节为单位。示例代码如下：

      ```js
      `$` node
      `>` os.totalmem()
      `>` 12798062592
      `>` os.freemem()
      `>` 2753875968
      ```

        __从输出信息看，这台电脑总内存为8GB，当前闲置内存大致为4.2GB。__

  * 堆外内存

    > 通过process.memoryUsage()的结果可以看到，堆中的内存用量总是小于进程的常驻内存用量。这意味着Node中的内存使用并非都是通过V8进行分配的。我们将那些不是通过V8分配的内存称为堆外内存。

      这里将前面的useMem()方法稍微改造一下，将Array变为Buffer，将size变大，每一次构造200MB的对象，相关代码如下：

      ```js
      var useMem = function() {
        var size = 200 * 1024 * 1024;
        var buffer = new Buffer(size);
        for (var i = 0; i < size; i++) {
          buffer[i] = 0;
        }
        return buffer;
      };
      ```

      重新执行该代码，得到的输出结果如下所示：

      ![Alt text](https://raw.githubusercontent.com/zqjflash/nodejs-memory/master/out_of_buffer.png)

      __15次循环都完整执行，并且三个内存占用值与前一个示例完全不同。在改造后的输出结果中，heapTotal和heapUsed的变化极小，唯一变化的是rss的值，并且该值已经远远超过V8的限制值。这其中的原因是Buffer对象不同于其他对象，它不经过V8的内存分配机制，所以也不会有堆内存的大小限制。__

      __这意味着利用堆外内存可以突破内存限制的问题。__

      __为何Buffer对象并非通过V8分配？这在于Node并不同于浏览器的应用场景。在浏览器中，js直接处理字符串即可满足绝大多数的业务需求，而Node则需要处理网络流和文件I/O流，操作字符串远远不能满足传输的性能需求。__

  * 小结

    从上面得知，Node内存构成主要由通过V8进行分配的部分和Node自行分配的部分。受V8的垃圾回收限制的主要是V8的堆内存。

## 内存泄漏

  > Node对内存泄漏十分敏感，一旦线上应用流量千万级别，哪怕一个字节的内存泄漏也会造成堆积，垃圾回收过程中将会耗费更多时间进行对象描述，应用响应缓慢，直到进程内存溢出，应用崩溃。

  > 在V8的垃圾回收机制下，在通常的代码编写中，很少会出现内存泄漏的情况。但是内存泄漏通常产生于无意间，较难排查。尽管内存泄漏的情况不尽相同，但其实质只有一个，那就是应当回收的对象出现意外而没有被回收，变成了常驻在老生代中的对象。通常，造成内存泄漏的原因有如下几个。

    * 缓存；
    * 队列消费不及时；
    * 作用域未释放。

  * 慎将内存当作缓存

    缓存访问效率要比I/O的效率高，一旦命中缓存，就可以节省一次I/O的时间。在Node中，一个对象被当作缓存使用时，将会常驻在老生代中。缓存中存储的键越多，长期存活的对象也就越多，这将导致垃圾回收在进行扫描和整理时，对这些对象无法回收。

    另一个问题，js开发者喜欢用对象的键值对来缓存东西，这与严格意义上的缓存有区别，严格意义的缓存有完善的过期策略，而普通对象的键值对没有。

    下面代码虽然利用JS对象创建一个缓存对象，但是受垃圾回收机制的影响，只能小量使用：

    ```js
    var cache = {};
    var get = function(key) {
      if (cache[key]) {
        return cache[key];
      } else {
        // get from otherwise
      }
    };
    var set = function (key, value) {
      cache[key] = value;
    };
    ```

    上述示例，只要限定缓存对象的大小，加上完善的过期策略以防止内存无限制增长，可以用。

    以下是一个可能无意识造成内存泄漏的场景：memoize。underscore对memoize的实现：

    ```js
    _.memoize = function(func, hasher) {
      var memo = {};
      hasher || (hasher = _.identity);
      return function() {
        var key = hasher.apply(this, arguments);
        return _.has(memo, key) ? memo[key] : (memo[key] = func.apply(this, arguments));
      };
    };
    ```
    它的原理是以参数作为键进行缓存，以内存空间换CPU执行时间。这里潜藏的陷阱即是每个被执行的结果都会按参数缓存在memo对象上，不会被清除。这在前端网页这种短时应用场景中不存在大问题，但是执行量大和参数多样性的情况下，会造成内存占用不释放。

    所以在Node中，任何试图拿内存当缓存的行为要小心使用。

    * 缓存限制策略

        为了解决缓存中的对象永远无法释放的问题，需要加入一种策略来限制缓存的无限增长。可以自己编写一个模块实现对键值对数量的限制，示例代码：

      ```js
      var LimitableMap = function(limit) {
        this.limit = limit || 10;
        this.map = {};
        this.keys = [];
      };
      var hasOwnProperty = Object.prototype.hasOwnProperty;
      LimitableMap.prototype.set = function(key, value) {
        var map = this.map;
        var keys = this.keys;
        if (!hasOwnProperty.call(map, key)) {
          if (keys.length === this.limit) {
            var firstKey = keys.shift();
            delete map[firstKey];
          }
          keys.push(key);
        }
        map[key] = value;
      };
      LimitableMap.prototype.get = function (key) {
        return this.map[key];
      };
      ```

      记录键在数组中，一旦超过数量，就以先进先出的方式进行淘汰。 这种淘汰策略并不是十分高效，只能应付小型应用场景。如果需要更高效的缓存，可以采用LRU算法的缓存，有限制的缓存，memoize还是可用。

      为了加速模块的引入，所有模块都会通过编译执行，然后被缓存起来。通过exports导出的函数，可以访问文件模块中的私有变量，这样每个文件模块在编译执行后形成的作用域因为模块缓存的原因，不会被释放。示例代码：

      ```js
      (function(exports, require, module, __filename, __dirname) {
        var local = "局部变量";
        exports.get = function() {
          return local;
        };
      });
      ```

      由于模块的缓存机制，模块是常驻老生代的，在设计模块时，要十分小心内存泄漏的出现，在下面的代码，每次调用leak()方法时，都导致局部变量leakArray不停增加内存的占用，且不被释放：

      ```js
      var leakArray = [];
      exports.leak = function() {
        leakArray.push("leak" + Math.random());
      };
      ```

      如果模块需要这么设计，那么请添加清空队列的相应接口，供调用者释放内存。

    * 缓存解决方案

      直接将内存作为缓存方案要十分谨慎，除了限制缓存的大小，另外要考虑的是，进程之间无法共享内存。如果在进程内使用缓存，这些缓存不可避免地有重复，对物理内存的使用是一种浪费。

      如果使用大量缓存，目前比较好的解决方案是采用进程外的缓存，进程自身不存储状态。外部的缓存软件有良好的缓存过期淘汰策略以及自有的内存管理，不影响Node进程的性能，在Node中主要可以解决一下两个问题：

      * 将缓存转移到外部，减少常驻内存的对象的数量，让垃圾回收更高效；
      * 进程之间可以共享缓存。

      目前，市面上比较好的缓存有Redis和Memcached。

      * Redis：https://github.com/mranney/node_redis。
      * Memcached：https://github.com/3rd-Eden/node-memcached。

  * 关注队列状态

    内存泄漏的另一个情况则是队列。在js中可以通过队列（数组对象）来完成许多特殊的需求，比如Bagpipe。队列在消费者-生产者模型中经常充当中间产物。在大多数应用场景下，消费的速度远大于生产的速度，内存泄漏不易产生，但是一旦消费速度低于生产速度，将会形成堆积。

    举个例子，有的应用会收集日志。如果欠缺考虑，也许会采用数据库来记录日志。日志通常会是海量的，数据库构建在文件系统之上，写入效率远远低于文件直接写入，于是会形成数据库写入操作的堆积，而js相关作用域也不会得到释放，内存占用不会回落，从而出现内存泄漏。

    遇到这种场景，表层解决方案是换用消费速度更高的技术。在日志收集的案例中，换用文件写入日志的方式会更高效。需要注意的是，如果生产速度因为某些原因突然激增，或者消费速度因为突然的系统故障降低，内存泄漏还是可能出现的。

    深度的解决方案应该是监控队列的长度，一旦堆积，应当通过监控系统产生报警并通知相关人员。另一个解决方案是任意异步调用都应该包含超时机制，一旦在限定的时间内未完成响应，通过回调函数传递超时异常，使得任意异步调用的回调都具备可控的响应时间，给消费速度一个下限值。

    对于Bagpipe而言，它提供了超时模式和拒绝模式。启用超时模式时，调用加入到队列中就开始计时，超时就直接响应一个超时错误。启用拒绝模式时，当队列拥塞时，新到来的调用会直接响应拥塞错误。这两种模式都能够有效地防止队列拥塞导致的内存泄漏问题。


## 内存泄漏排查

> 在Node中，由于V8的堆内存大小的限制，它对内存泄漏非常敏感。当在线服务的请求量变大时，哪怕是一个字节的泄漏都会导致内存占用过高。下面介绍一下遇到内存泄漏时的排查方案。

  有一些常见的工具来定位Node应用的内存泄漏

  ```html

    v8-profiler：它可以用于对V8堆内存抓取快照和对CPU进行分析；
    node-heapdump：它允许对V8堆内存抓取快照，用于事后分析；
    node-mtrace：它使用GCC的mtrace工具来分析堆的使用；
    dtrace：有完善的dtrace工具用来分析内存泄漏；
    node-memwatch：来自Mozilla贡献的模块，采用WTFPL许可发布。

  ···

  * node-heapdump

    想要了解node-heapdump对内存泄漏进行排查的方式，需要先构造如下一份包含内存泄漏的代码示例，并将其存为server.js文件：

      ```js
      var leakArray = [];
      var leak = function() {
        leakArray.push("leak" + Math.random());
      };
      http.createServer(function (req, res) {
        leak();
        res.writeHead(200, {'Content-Type': 'text/plain'});
        res.end('Hello World\n');
      }).listen(1337);
      console.log('Server running at http://127.0.0.1:1337/');
      ```

     在上面这段代码中，每次访问服务进程都将引起leakArray数组中的元素增加，而且得不到回收。我们可以用curl工具输入http://127.0.0.1:1337/命令来模拟用户访问。

     * 安装node-heapdump

       ```js
       npm install heapdump
       ```

     * 引入node-heapdump

       ```js
       var heapdump = require('heapdump');
       ```

       引入node-heapdump后，访问多次，leakArray就会具备大量的元素。这个时候我们通过向服务进程发送SIGUSR2信号，让node-heapdump抓拍一份堆内存的快照。发送信号的命令如下：

       ```js
       kill -USR2 <pid>
       ```

       这份抓取的快照将会在文件目录下以heapdump-<sec>.<usec>.heapsnapshot的格式存放。这是一份较大的JSON文件，需要通过chrome的开发者工具打开查看。

       在chrome的开发者工具中选中Profiles面板，右击该文件后，从弹出的快捷菜单中选择Load...选项，打开刚才的快照文件，就可以查看堆内存中的详细信息。

  * node-memwatch

    准备一份内存泄漏代码：

      ```js
      var memwatch = require('memwatch');
      memwatch.on('leak', function(info) {
        console.log('leak:');
        console.log(info);
      });
      memwatch.on('stats', function(stats) {
        console.log('stats:');
        console.log(stats);
      });
      var http = require('http');
      var leakArray = [];
      var leak = function() {
        leakArray.push("leak" + Math.random());
      };
      http.createServer(function(req, res) {
        leak();
        res.writeHead(200, {'Content-Type': 'text/plain'});
        res.end('Hello World\n');
      }).listen(1337);
      console.log('Server running at http://127.0.0.1:1337/');
      ```

    * stats事件

      在进程中使用node-memwatch之后，每次进行全堆垃圾回收时，将会触发一次stats事件，这个事件将会传递内存的统计信息。在对上述代码创建的服务进程进行访问时，某次stats事件打印的数据如下所示，其中每项的意义卸载注释中：

      ```js
      stats:
      {
        num_full_gc: 4, // 第几次全堆垃圾回收
        num_inc_gc: 23, // 第几次增量垃圾回收
        heap_compactions: 4, // 第几次对老生代进行整理
        usage_trend: 0, // 使用趋势
        estimated_base: 7152944, // 预估基数
        current_base: 7152944, // 当前基数
        min: 6720776, // 最小
        max: 7152944 // 最大
      }
      ```

      在这些数据中，num_full_gc和num_inc_gc比较直观地反应了垃圾回收的情况。

    * leak事件

      如果经过连续5次垃圾回收后，内存仍然没有被释放，这意味着有内存泄漏的产生，node-memwatch会触发一个leak事件。某次leak事件得到的数据如下所示：

      ```js
      leak:
      {
        start: Mon Oct 07 2013 13:46:27 GMT+0800 (CST),
        end: Mon Oct 07 2013 13:54:40 GMT+0800 (CST),
        growth: 6222576,
        reason: 'heap growth over 5 consecutive GCs (8m 13s) - 43.33 mb/hr'
      }
      ```

      这个数据能显示5次垃圾回收的过程中内存增长了多少。

    * 堆内存比较

      最终得到的leak事件的信息只能告知我们应用中存在内存泄漏，具体问题产生在何处还需要从V8的堆内存上定位。node-memwatch提供了抓取快照和比较快照的功能，它能够比较堆上对象的名称和分配数量，从而找到导致内存泄漏的元凶。

      下面为一段导致内存泄漏的代码，这是通过node-memwatch获取堆内存差异结果的示例：

      ```js
      var memwatch = require('memwatch');
      var leakArray = [];
      var leak = function() {
        leakArray.push("leak" + Math.random());
      };
      
      // Take first snapshot
      var hd = new memwatch.HeapDiff();
      
      for (var i = 0; i < 10000; i++) {
        leak();
      }

      // Take the second snapshot and compute the diff
      var diff = hd.end();
      console.log(JSON.stringify(diff, null, 2));
      ```

      执行node diff.js，得到的输出结果如下所示：

      ```js
      {
        "before": {
          "nodes": 11719,
          "time": "2013-10-07T06:32:07.000Z",
          "size_bytes": 1493304,
          "size": "1.42 mb"
        },
        "after": {
          "nodes": 31618,
          "time": "2013-10-07T06:32:07.000Z",
          "size_bytes": 2684864,
          "size": "2.56 mb"
        },
        "change": {
          "size_bytes": 1191560,
          "size": "1.14 mb",
          "freed_nodes": 129,
          "allocated_nodes": 20028,
          "details": [
            {
              "what": "Array",
              "size_bytes": 323720,
              "size": "316.13 kb",
              "+": 15,
              "-": 65
            },
            {
              "what": "Code",
              "size_bytes": -10944,
              "size": "-10.69 kb",
              "+": 8,
              "-": 28
            },
            {
              "what": "String",
              "size_bytes": 879424,
              "size": "858.81 kb",
              "+": 20001,
              "-": 1
            }
          ]
        }
      }
      ```
  
    在上面的输出结果中，主要关注change节点下的freed_nodes和allocated_nodes，他们记录了释放的，它们记录了释放的节点数量和分配的节点数量。这里由于有内存泄漏，分配的节点数量远远多于释放的节点数量。在details下可以看到具体每种类型的分配和释放数量，主要问题展现在下面这段输出中：

    ```js
    {
      "what": "String",
      "size_bytes": 879424,
      "size": "858.51 kb",
      "+": 20001,
      "-": 1
    }
    ```

    在上述代码中，加号和减号分别表示分配和释放的字符串对象数量。可以通过上面的输出结果猜测到，有大量的字符串没有被回收。

  * 小结

    排查内存泄漏的原因主要通过对堆内存进行分析而找到。node-heapdump和node-memwatch各有所长。

## 大内存应用

  在Node中，不可避免地还是会存在操作大文件的场景。由于Node的内存限制，不过Node提供了stream模块用于处理大文件。

  stream模块是Node的原生模块，直接引用即可。stream继承自EventEmitter，具备基本的自定义事件功能，同时抽象出标准的事件和方法。它分可读和可写两种。Node中的大多数模块都有stream的应用，比如fs的createReadStream()和createWriteStream()方法可以分别用于创建文件的可读流与可写流，process模块中的stdin和stdout则分别是可读流和可写流的示例。

  由于V8的内存限制，我们无法通过fs.readFile()和fs.writeFile()直接进行大文件的操作，而改用fs.createReadStream()和fs.createWriteStream()方法通过流的方式实现对大文件的操作。下面的代码展示了如何读取一个文件，然后将数据写入到另一个文件的过程：

  ```js
  var reader = fs.createReadStream('in.txt');
  var writer = fs.createWriteStream('out.txt');
  reader.on('data', function (chunk) {
    writer.write(chunk);
  });
  reader.on('end', function () {
    writer.end();
  });
  ```

  可读流提供了管道方法pipe()，封装了data事件和写入操作。通过流的方式，上述代码不会受到V8内存限制的影响，有效地提高了程序的健壮性。

  如果不需要进行字符串层面的操作，则不需要借助V8来处理，可以尝试进行纯粹的Buffer操作，这不会受到V8堆内存的限制。但是这种大片使用内存的情况依然要小心，即使V8不限制堆内存的大小，物理内存依然有限制。

## 总结

  Node将JavaScript的主要应用场景扩展到了服务器端，相应要考虑的细节也与浏览器端不同，需要更严谨地为每一份资源作出安排。总的来说，内存在Node中不能随心所欲地使用。

## 参考资源

  * `https://github.com/joyent/node/wiki/FAQ`
  * `http://www.cs.sunysb.edu/~cse304/Fall08/Lectures/mem-handout.pdf`
  * `http://en.wikipedia.org/wiki/Resident_set_size`
  * `https://github.com/isaacs/node-lru-cache`
  * `https://github.com/mranney/node_redis`
  * `https://github.com/3rd-Eden/node-memcached`
  * `http://nodejs.org/docs/latest/api/stream.html`
  * `http://www.showmuch.com/a/20111012/215033.html`
  * `https://github.com/lloyd/node-memwatch`
  * `https://github.com/bnoordhuis/node-heapdump`
  * `http://www.williamlong.info/archives/3042.html`
  * `https://code.google.com/p/v8/issues/detail?id=847`
  * `http://blog.chromium.org/2011/11/game-changer-for-interactive.html`