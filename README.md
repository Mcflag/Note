# 笔记大纲

## 1、Java 相关

1. 容器（HashMap、HashSet、LinkedList、ArrayList、数组等）

	需要了解其实现原理，还要灵活运用，如：自己实现 LinkedList、两个栈实现一个队列，数组实现栈，队列实现栈等。

2. 内存模型
3. 垃圾回收算法（JVM）
4. 类加载过程（需要多看看，重在理解，对于热修复和插件化比较重要）
5. 反射
6. 多线程和线程池
7. HTTP、HTTPS、TCP/IP、Socket通信、三次握手四次挥手过程
8. 设计模式（六大基本原则、项目中常用的设计模式、手写单例等）
9. 断点续传
10. volatile理解，JMM中主存和工作内存到底是啥？和JVM各个部分怎么个对应关系？[参考link](https://www.cnblogs.com/dolphin0520/p/3613043.html)
11. 序列化，Serializable在序列化时使用了反射，从而导致GC的频繁调用。[参考link](https://www.cnblogs.com/yezhennan/p/5527506.html)

12. 可见性，原子性，有序性
* 可见性volatile，一个线程的修改对另外一个线程是马上可见的。
* 原子性CAS操作，要么都做要么都不做
* 有序性synchronized通过进入和退出Monitor(观察器)实现，CPU可能会乱序执行指令，如果在本线程内观察，所有操作都是有序的，如果在一个线程中观察另一个线程，所有操作都是无序的。[参考link](https://blog.csdn.net/qq_33689414/article/details/73527438)

13. Java锁机制，java锁机制其实是锁总线
14. Java的常量池？不同String赋值方法，引用是否相等？字面值是常量，在字节码中使用id索引，equals相等引用不一定相等，Android上String的构造函数会被虚拟机拦截，重定向到StringFactory
15. HashMap的实现？树化阈值？负载因子？数组加链表加红黑树，默认负载因子0.75，树化阈值8，这部分比较常考，建议专门准备。
16. Java实现无锁同步，CAS的实现如AtomicInteger等等
17. 单例模式
* 双重检查

```java
public class Singleton {

    private static volatile Singleton singleton;

    private Singleton() {}

    public static Singleton getInstance() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```

* 反序列化安全，反射安全。枚举级单例，类加载时由JVM保证单例，反序列化不会生成新对象，另外一种反射安全是在构造函数中对单例进行检查如果存在则抛出异常。

18. 锁的条件变量

信号量要与一个锁结合使用，当前线程要先获得这个锁，然后等待与这个锁相关联的信号量，此时该锁会被解锁，其他线程可以抢到这个锁，如果其他线程抢到了这个锁，那他可以通知这个信号量，然后释放该锁，如果此时第一个线程抢到了该锁，那么它将从等待处继续执行(应用场景，将异步回调操作封装为变为同步操作，避免回调地狱)。

信号量与锁相比的应用场景不同，锁是服务于共享资源的，而信号量是服务于多个线程间的执行的逻辑顺序的，锁的效率更高一些。

19. ThreadLocal原理

线程上保存着ThreadLocalMap，每个ThreadLocal使用弱引用包装作为Key存入这个Map里，当线程被回收或者没有其他地方引用ThreadLocal时，ThreadLocal也会被回收进而回收其保存的值。

20. 软引用，弱引用，虚引用
* 软引用内存不够的时候会释放。
* 弱引用GC时释放。
* 虚引用，需要和一个引用队列联系在一起使用，引用了跟没引用一样，主要是用来跟GC做一些交互。

21. ClassLoader双亲委派机制

简单来说就是先把加载请求转发到父加载器，父加载器失败了，再自己试着加载。

22. GC Roots有这些

通过System Class Loader或者Boot Class Loader加载的class对象，通过自定义类加载器加载的class不一定是GC Root：

* 处于激活状态的线程
* 栈中的对象
* JNI栈中的对象
* JNI中的全局对象
* 正在被用于同步的各种锁对象
* JVM自身持有的对象，比如系统类加载器等。

23. GC算法

|名称|描述|优点|缺点|
|:----:|:----:|:----:|:----:|
|标记-清除算法|暂停除了GC线程以外的所有线程,算法分为“标记”和“清除”两个阶段,首先从GC Root开始标记出所有需要回收的对象，在标记完成之后统一回收掉所有被标记的对象。||标记-清除算法的缺点有两个：首先，效率问题，标记和清除效率都不高。其次，标记清除之后会产生大量的不连续的内存碎片，空间碎片太多会导致当程序需要为较大对象分配内存时无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作|
|复制算法|将可用内存按容量分成大小相等的两块，每次只使用其中一块，当这块内存使用完了，就将还存活的对象复制到另一块内存上去，然后把使用过的内存空间一次清理掉。|这样使得每次都是对其中一块内存进行回收，内存分配时不用考虑内存碎片等复杂情况，只需要移动堆顶指针，按顺序分配内存即可，实现简单，运行高效|复制算法的缺点显而易见，可使用的内存降为原来一半|
|标记-整理算法|标记-整理算法在标记-清除算法基础上做了改进，标记阶段是相同的,标记出所有需要回收的对象，在标记完成之后不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，在移动过程中清理掉可回收的对象，这个过程叫做整理。|标记-整理算法相比标记-清除算法的优点是内存被整理以后不会产生大量不连续内存碎片问题。复制算法在对象存活率高的情况下就要执行较多的复制操作，效率将会变低，而在对象存活率高的情况下使用标记-整理算法效率会大大提高||
|分代收集算法|是java的虚拟机的垃圾回收算法.基于编程中的一个事实,越新的对象的生存期越短,根据内存中对象的存活周期不同，将内存划分为几块，java的虚拟机中一般把内存划分为新生代和年老代，当新创建对象时一般在新生代中分配内存空间，当新生代垃圾收集器回收几次之后仍然存活的对象会被移动到年老代内存中，当大对象在新生代中无法找到足够的连续内存时也直接在年老代中创建|||

## 2、Android 基础

1. 自定义 View [参考链接：自定义View，有这一篇就够了 - 简书、Android 自定义 View](https://www.jianshu.com/p/c84693096e41)
2. 事件拦截分发 [Android事件分发机制，大表哥带你慢慢深入 - 简书](https://www.jianshu.com/p/fc0590afb1bf)
3. 解决过的一些性能问题，在项目中的实际运用。
4. 性能优化工具  (TraceView、Systrace、调试 GPU 过度绘制 & GPU 呈现模式分析、Hierarchy Viewer、MAT、Memory Monitor & Heap Viewer & Allocation Tracker 等）
5. 性能优化
* （1）网络：API 优化、流量优化、弱网优化
* （2）内存：OOM 处理、内存泄漏、内存检测、分析、Bitmap 优化
* （3）绘制
* （4）电量：WeakLock 机制、JobScheduler 机制
* （5）APK 瘦身
* （6）内存抖动
* （7）内存泄漏
* （8）卡顿
* （9）性能优化：布局优化、过度渲染处理、ANR 处理、监控、埋点、Crash 上传。
6. IntentService 原理及应用
7. 缓存自己如何实现（LRUCache 原理）
8. 图形图像相关：OpenGL ES 管线流程、EGL 的认识、Shader 相关
9. SurfaceView、TextureView、GLSurfaceView 区别及使用场景
10. 动画、差值器、估值器 [Android中的View动画和属性动画 - 简书、Android 动画 介绍与使用](https://www.jianshu.com/p/b117c974deaf)
11. MVC、MVP、MVVM
12. Handler、ThreadLocal、AsyncTask
13. Gradle（Groovy 语法、Gradle 插件开发基础）
14. 热修复、插件化

## 3、Android Framework

1. AMS 、PMS
2. Activity 启动流程
3. Binder 机制（IPC、AIDL 的使用）
4. 为什么使用 Parcelable，好处是什么？
5. Android 图像显示相关流程，Vsync 信号等

## 4、三方源码

1. Glide ：加载、缓存、LRU 算法
2. EventBus
3. LeakCanary
4. ARouter
5. 插件化（不同插件化机制原理与流派，优缺点。局限性）
6. 热修复
7. RXJava
8. Retrofit

## 5、算法与数据结构

1. 单链表：反转、插入、删除
2. 双链表：插入、删除
3. 手写常见排序、归并排序、堆排序
4. 二叉树前序、中序、后序遍历
5. 最大 K 问题
6. 广度、深度优先搜索算法
7. 可以去刷一下 LeetCode ,对自己提升也会比较大。


## 另外的

2.2 Android
2.2.1 Handler、MessageQueue等一套东西讲一下，详细说了下源码。为什么主线程loop不会ANR？

Android线程模型就是消息循环,Looper关联MessageQueue,不断尝试从MessageQueue取出Message来消费,这个过程可能会被它自己阻塞.
而Handler最终都调用enqueueMessage(Message,when)入队的,延迟的实现时当前是时间加上延迟时间给消息指定一个执行的时间点,然后在MessageQueue找到插入位置,此时会判断是否需要唤醒线程来消费消息以及更新下次需要暂停的时间.
Message知道要发到哪个Handler是因为Message把Handler保存到了target.
Message内部使用链表进行回收复用

2.2.2 View事件以及View体系相关知识。
建议看《Android开发艺术探索》,这玩意三言两语讲不清楚
2.2.3 Android中使用多线程的方法

裸new一个Thread(失控线程,不推荐)
RxJava的调度器(io(优先级比低),密集计算线程(优先级比高,用于执行密集计算任务),安卓主线程, Looper创建(实际上内部也是创建了Handler))
Java Executor框架的Executors#newCachedThreadPool(),不会造成资源浪费,60秒没有被使用的线程会被释放
AsyncTask,内部使用FutureTask实现,通过Handler将结果转发到主线程,默认的Executor是共用的,如果同时执行多个AsyncTask,就可能需要排队,但是可以手动指定Executor解决这个问题,直接new匿名内部类会保存外部类的引用,可能会导致内存泄漏
Android线程模型提供的Handler和HandlerThread
使用IntentService
IntentService和Service的区别——没什么区别,其实就是开了个HandlerThread,让它不要在主线程跑耗时任务

2.2.4 RecyclerView复用缓存
建议看一下,这个可能会被问,不过我运气好没被问到.
2.2.5 JNI(不常考)

可避免的内存拷贝,直接传递对象,到C层是一个jobject的指针,可以使用jmethodID和jfiledID访问方法和字段,无需进行内存拷贝,使用直接缓冲区也可以避免内存拷贝.
无法避免的内存拷贝,基本类型数组,无法避免拷贝,因为JVM不信任C层的任何内存操作,特别是字符串操作,因为Java的字符串与C/C++的字符串所使用的数据类型是不一样的C/C++使用char一个字节(1字节=8位)或wchar_t是四字节.而jstring和jchar使用的是UTF-16编码使用双字节.(Unicode是兼容ASCII,但不兼容GBK,需要自己转换)
自己创建的局部引用一定要释放,否则一直持有内存泄漏
非局部引用方法返回后就会失效,除非创建全局引用,jclass是一个jobject,方法外围使用时需要创建全局引用,jmethodID和jfiledID不需要.
JNI是通过Java方法映射到C函数实现的,如果使用这种方法,函数必须以C式接口导出(因为C++会对名字做修饰处理),当然也可以在JNI_OnLoad方法中注册.
JNIEnv是线程独立的,JNI中使用pthread创建的线程没有JNIEnv,需要AttachCurrentThread来获取JNIEnv,不用时要DetachCurrentThread

2.3专业课
2.3.1 TCP和UDP的根本区别？
数据报,流模式,TCP可靠,包序不对会要求重传,UDP不管,甚至不能保证送到
2.3.2 TCP三次握手
这个被问的几率非常的大,几乎等于必问,建议专门花时间去看.
2.3.3 Http和Https
CA证书,中间机构,公钥加密对称秘钥传回服务端,一个明文一个加密,SSL层,中间人攻击,参考link
2.4 ACM
对于ACM,比较常考链表的题,不常刷算法的同学一定不要对其有抵触心理.
你可能会问为什么要ACM?网上答案说的什么提高代码质量,能够更好地阅读别人的代码这些理由有一定道理,但对于我们去面试的人而言最重要的是ACM是面试官考察你编码能力的最直接的手段,所以不用说这么多废话刷题就够了.
刷题的话,建议去刷leetcode,题号在200以内的,简单和中等难度,不建议刷困难,因为面试的时候基本就不会出,没人愿意在那里等你想一个半个小时的.
在面试官面前现场白板编程时,可以先把思路告诉面试官,写不写得出来是另外一回事,时间复杂度和空间复杂度是怎么来的一定要搞清楚.在编码时也不一定要写出最佳的时间和空间的算法,但推荐你写出代码量最少,思路最清晰的,这样面试官看得舒服,你讲得也舒服.
下面是我在网上收集或者是在实际中遇到过的ACM题,基本上在leetcode上也都有类似的.
2.4.1 数组、链表

链表逆序(头条几乎是必考的)

    public ListNode reverseList(ListNode head)
    {
        if (head == null)
        {
            return null;
        }
        if (head.next == null)
        {
            return head;
        }
        ListNode prev = null;
        ListNode current = head;
        while (current != null)
        {
            ListNode next = current.next;
            current.next = prev;
            prev = current;
            current = next;
        }
        return prev;
    }
复制代码
删除排序数组中的重复项

    public int removeDuplicates(int[] nums)
    {
        int length = nums.length;
        if (length == 0 || length == 1)
        {
            return length;
        }
        int size = 1;
        int pre = nums[0];
        for (int i = 1; i < length; )
        {
            if (nums[i] == pre)
            {
                i++;
            } else
            {
                pre = nums[size++] = nums[i++];
            }
        }
        return size;
    }
复制代码
数组中找到重复元素
n个长为n的有序数组，求最大的n个数
用O(1)的时间复杂度删除单链表中的某个节点
把后一个元素赋值给待删除节点，这样也就相当于是删除了当前元素,只有删除最后一个元素的时间为o(N)平均时间复杂度仍然为O(1)

      public void deleteNode(ListNode node) {
          ListNode next = node.next;
          node.val = next.val;
          node.next = next.next;
      }
复制代码
删除单链表的倒数第N个元素
两个指针,第一个先走N步第二个再走,时间复杂度为O(N),参考link

      public ListNode removeNthFromEnd(ListNode head, int n) {
          if (head == null)
          {
              return null;
          }
          if (head.next == null)
          {
              return n == 1 ? null : head;
          }
          int size = 0;
          ListNode point = head;
          ListNode node = head;
          do
          {
              if (size >= n + 1)
              {
                  point = point.next;
              }
              node = node.next;
              size++;
          } while (node != null);
          if (size == n)
          {
              return head.next;
          }
          node = point.next;
          point.next = node == null ? null : node.next;
          return head;
      }
复制代码
从长序列中找出前K大的数字
用数组实现双头栈

  public static class Stack<T>
      {
          
          public Stack(int cap)
          {
              if (cap <= 0)
              {
                  throw new IllegalArgumentException();
              }
              array = new Object[cap];
              left = 0;
              right = cap - 1;
          }
  
          private Object[] array;
          private int left;
          private int right;
          
          public void push1(T val)
          {
              int index = left + 1;
              if (index < right)
              {
                  array[index] = val;
              }
              left = index;
          }
          
          @SuppressWarnings("unchecked")
          public T pop1()
          {
              if (left > 0)
              {
                  return (T)array[left--];
              }
              return null;
          }
          
          public void push2(T val)
          {
              int index = right - 1;
              if (index > left)
              {
                  array[index] = val;
              }
              right = index;
          }
  
          @SuppressWarnings("unchecked")
          public T pop2()
          {
              if (right < array.length)
              {
                 return (T)array[right++];
              }
              return null;
          }
      }
复制代码
两个链表求和，返回结果也用链表表示 1 -> 2 -> 3 + 2 -> 3 -> 4 = 3 -> 5 -> 7

      public ListNode addTwoNumbers(ListNode node1, ListNode node2)
      {
          ListNode head = null;
          ListNode tail = null;
          boolean upAdd = false;
          while (!(node1 == null && node2 == null))
          {
              ListNode midResult = null;
              if (node1 != null)
              {
                  midResult = node1;
                  node1 = node1.next;
              }
              if (node2 != null)
              {
                  if (midResult == null)
                  {
                      midResult = node2;
                  } else
                  {
                      midResult.val += node2.val;
                  }
                  node2 = node2.next;
              }
              if (upAdd)
              {
                  midResult.val += 1;
              }
              if (midResult.val >= 10)
              {
                  upAdd = true;
                  midResult.val %= 10;
              }
              else
              {
                  upAdd = false;
              }
              if (head == null)
              {
                  head = midResult;
                  tail = midResult;
              } else
              {
                  tail.next = midResult;
                  tail = midResult;
              }
          }
          if (upAdd)
          {
              tail.next = new ListNode(1);
          }
          return head;
      }
复制代码
交换链表两两节点

      public ListNode swapPairs(ListNode head)
      {
          if (head == null)
          {
              return null;
          }
          if (head.next == null)
          {
              return head;
          }
          ListNode current = head;
          ListNode after = current.next;
          ListNode nextCurrent;
          head = after;
          do
          {
              nextCurrent = after.next;
              after.next = current;
              if (nextCurrent == null)
              {
                  current.next = null;
                  break;
              }
              current.next = nextCurrent.next;
              after = nextCurrent.next;
              if (after == null)
              {
                  current.next = nextCurrent;
                  break;
              }
              current = nextCurrent;
          } while (true);
          return head;
      }
复制代码
找出数组中和为给定值的两个元素，如：[1, 2, 3, 4, 5]中找出和为6的两个元素。

      public int[] twoSum(int[]mun,int target)
      {
          Map<Integer, Integer> table = new HashMap<>();
          for (int i = 0; i < mun.length; ++i)
          {
              Integer value = table.get(target - mun[i]);
              if (value != null)
              {
                  return new int[]{i, value};
              }
              table.put(mun[i], i);
          }
          return null;
      }
复制代码2.4.2 树

二叉树某一层有多少个节点

2.4.3 排序

双向链表排序(这个就比较过分了,遇到了就自求多福吧)

  public static void quickSort(Node head, Node tail) {
  		if (head == null || tail == null || head == tail || head.next == tail) {
  			return;
  		}
  		
  		if (head != tail) {
  			Node mid = getMid(head, tail);
  			quickSort(head, mid);
  			quickSort(mid.next, tail);
  		}
  	}
  	
  	public static Node getMid(Node start, Node end) {
  		int base = start.value;
  		while (start != end) {
  			while(start != end && base <= end.value) {
  				end = end.pre;
  			}
  			start.value = end.value;
  			while(start != end && base >= start.value) {
  				start = start.next;
  			}
  			end.value = start.value;
  		}
  		start.value = base;
  		return start;
  	}
  	
  	/**
  	 * 使用如内部实现使用双向链表的LinkedList容器实现的快排 
  	 */
  	public static void quickSort(List<Integer> list) {
  		if (list == null || list.isEmpty()) {
  			return;
  		}
  		quickSort(list, 0, list.size() - 1);
  	}
  	
  	private static void quickSort(List<Integer> list, int i, int j) {
  		if (i < j) {
  			int mid = partition(list, i, j);
  			partition(list, i, mid);
  			partition(list,mid + 1, j);
  		}
  	}
  	
  	private static int partition(List<Integer> list, int i, int j) {
  		int baseVal = list.get(i);
  		while (i < j) {
  			while (i < j && baseVal <= list.get(j)) {
  				j--;
  			}
  			list.set(i, list.get(j));
  			while (i < j && baseVal >= list.get(i)) {
  				i++;
  			}
  			list.set(j, list.get(i));
  		}
  		list.set(i, baseVal);
  		return i;
  	}
复制代码
常见排序,如堆排序,快速,归并,冒泡等,还得记住他们的时间复杂度.

2.5 项目
2.5.1 视频聊天使用什么协议？
不要答TCP,答RTMP实时传输协议,RTMP在Github也有很多开源实现,建议面这方面的同学可以去了解一下.
2.5.2 你在项目中遇到的一些问题,如何解决,思路是什么?
这一块比较抽象,根据你自己的项目来,着重讲你比较熟悉,有把握的模块,一般面试官都会从中抽取一些问题来向你提问.
