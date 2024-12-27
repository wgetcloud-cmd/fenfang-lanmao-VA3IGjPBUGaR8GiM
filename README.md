
在多线程编程中，死锁是一种非常常见的问题，稍不留神可能就会产生死锁，今天就和大家分享死锁产生的原因，如何排查，以及解决办法。


线程死锁通常是因为两个或两个以上线程在资源争夺中，形成循环等待，导致它们都无法继续执行各自后续操作的现象。


我们结合下图简单举个例子，线程1拥有资源A同时使用锁A进行锁定，并等待获取资源B；与此同时线程2拥有资源B同时使用锁B进行锁定，并等待获取资源A。此时便形成了线程1和线程2相互等待对方先释放锁的现象，形成了死循环，最终导致死锁。


![](https://img2024.cnblogs.com/blog/386841/202412/386841-20241226221756275-1517063144.png)


# ***01***、产生死锁的必要条件


根据死锁产生的原因，可以总结出以下四个死锁产生的必要条件。


## 1、互斥条件


互斥即非此即彼，一个资源要不是我拥有，要不是你拥有，就是不能我们俩同时拥有。也就是互斥条件是指至少有一个资源处于非共享状态，一次只能有一个线程可以访问该资源。


## 2、占有并等待条件


该条件是指一个线程在拥有至少一个资源的同时还在等待获取其他线程拥有的资源。


## 3、不可剥夺条件


该条件是指一个线程一旦获取了某个资源，则不可被强行剥夺对该资源的所有权，只能等待该线程自己主动释放。


## 4、循环等待条件


循环等待是指线程等待资源形成的循环链，比如线程A等待资源B，线程B等待资源C，线程C等待资源A，但是资源A被线程A拥有，资源B被线程B拥有，资源C被线程C拥有，如此形成了依赖死循环，都在等待其他线程释放资源。


# ***02***、代码示例


下面我们实现一个简单的死锁代码示例，代码如下：



```
//锁1
private static readonly object lock1 = new();
//锁2
private static readonly object lock2 = new();
//模拟两个线程死锁
public static void ThreadDeadLock()
{
    //线程1
    var thread1 = new Thread(Thread1);
    //线程2
    var thread2 = new Thread(Thread2);
    //线程1 启动
    thread1.Start();
    //线程2 启动
    thread2.Start();
    //等待 线程1 执行完毕
    thread1.Join();
    //等待 线程2 执行完毕
    thread2.Join();
}
//线程1
public static void Thread1()
{
    //线程1 首先获取 锁1
    lock (lock1)
    {
        Console.WriteLine("线程1: 已获取 锁1");
        //模拟一些操作
        Thread.Sleep(1000);
        Console.WriteLine("线程1: 等待获取 锁2");
        //线程1 等待 锁2
        lock (lock2)
        {
            Console.WriteLine("线程1: 已获取 锁2");
        }
    }
}
//线程2
public static void Thread2()
{
    //线程2 首先获取 锁2
    lock (lock2)
    {
        Console.WriteLine("线程2: 已获取 锁2");
        //模拟一些操作
        Thread.Sleep(1000);
        Console.WriteLine("线程2: 等待获取 锁1");
        //线程2 等待 锁1
        lock (lock1)
        {
            Console.WriteLine("线程2: 已获取 锁1");
        }
    }
}

```

在上面的代码中，thread1 先拥有lock1，然后尝试获取lock2；thread2 先拥有锁住 lock2，然后尝试获取lock1；由于线程间相互等待对方释放资源，所以导致死锁。


下面我们看看上面代码执行效果：


![](https://img2024.cnblogs.com/blog/386841/202412/386841-20241226221742369-1174843247.png)


可以发现线程1和线程2都在等待彼此所拥有的锁。


# ***03***、排查死锁


上一节中我们编写了一个简单的死锁代码示例，但是实际研发过程中代码不可能这么简单直观，一眼就能看出来问题所在。因此如何排查发生死锁呢？


其实我们的开发工具Visual Studio就可以查看。可以通过调试菜单中窗口下的线程、调用堆栈、并行堆栈等调试窗口查看。


上面代码正常运行后，编辑器为如下状态，也没有报错，啥也看不出来。


![](https://img2024.cnblogs.com/blog/386841/202412/386841-20241226221729383-228382814.png)


在默认状态下是无法看出东西，此时我们只需要点击全部中断按钮，则死锁的相关信息都会展示出来，如下图。


![](https://img2024.cnblogs.com/blog/386841/202412/386841-20241226221720577-1119521864.png)


可以看到已经提示检测到死锁了，同时在调用堆栈窗口中还可以通过双击切换具体发生死锁的代码。


我们再切换至并行堆栈调试窗口，和调用堆栈相比，并行堆栈窗口更偏向图形化，并且发生死锁的两个线程方法都有体现出来，同样可以通过双击切换到具体代码，如下图：


![](https://img2024.cnblogs.com/blog/386841/202412/386841-20241226221712378-1284068743.png)


下面我们再来看看线程调试窗口，如下图，可以发现前面有两个箭头，其中黄色箭头表示当前选中的发生死锁的代码，图中绿色选中代码，灰色箭头表示第一个发生死锁的代码。可以通过双击当前窗口中行进行发生死锁代码的切换，如下图：


![](https://img2024.cnblogs.com/blog/386841/202412/386841-20241226221704432-1870755249.png)


当然还可以通过其他方式排查死锁，比如分析dump文件，这里就不深入了，后面有机会再单独讲解。


# ***04***、解决办法


下面介绍几种避免死锁的指导思想。


## 1、顺序加锁


顺序加锁就是为了避免产生循环等待，如果大家都是先锁定lock1，再锁定lock2，则就不会产生循环等待。


看看如下代码：



```
//线程1
public static void Thread1New()
{
    //线程1 首先获取 锁1
    lock (lock1)
    {
        Console.WriteLine("线程1: 已获取 锁1");
        //模拟一些操作
        Thread.Sleep(1000);
        Console.WriteLine("线程1: 等待获取 锁2");
        //线程1 等待 锁2
        lock (lock2)
        {
            Console.WriteLine("线程1: 已获取 锁2");
        }
    }
}
//线程2
public static void Thread2New()
{
    //线程2 首先获取 锁2
    lock (lock1)
    {
        Console.WriteLine("线程2: 已获取 锁2");
        //模拟一些操作
        Thread.Sleep(1000);
        Console.WriteLine("线程2: 等待获取 锁1");
        //线程2 等待 锁1
        lock (lock2)
        {
            Console.WriteLine("线程2: 已获取 锁1");
        }
    }
}

```

我们看看代码执行结果。


![](https://img2024.cnblogs.com/blog/386841/202412/386841-20241226221652439-48743510.png)


## 2、使用尝试锁


我们可以使用一些其他锁机制，比如使用Monitor.TryEnter方法尝试获取锁，如果在指定时间内没有获取到锁，则释放当前所拥有的锁，以此来避免死锁。


## 3、使用超时机制


我们可以通过Thead结合CancellationToken实现超时机制，避免线程无限等待。当然可以直接使用Task，因为Task本身就支持CancellationToken，提供了内置的取消支持使用起来更方便。


## 4、避免嵌套使用锁


一个线程在拥有一个锁的同时尽量避免再去申请另一个锁，这样可以避免循环等待。


上面是使用Thread实现的示例，现在大家直接使用Thread可能比较少，大多数都是使用Task，最后给大家一个Task死锁示例，代码如下：



```
//锁1
private static readonly object lock1 = new();
//锁2
private static readonly object lock2 = new();
//模拟两个任务死锁
public static async Task TaskDeadLock()
{
    //启动 任务1
    var task1 = Task.Run(() => Task1());
    //启动 任务2
    var task2 = Task.Run(() => Task2());
    //等待两个任务完成
    await Task.WhenAll(task1, task2);
}
//任务1
public static async Task Task1()
{
    //任务1 首先获取 锁1
    lock (lock1)
    {
        Console.WriteLine("任务1: 已获取 锁1");
        //模拟一些操作
        Task.Delay(1000).Wait();
        //任务1 等待 锁2
        Console.WriteLine("任务1: 等待获取 锁2");
        lock (lock2)
        {
            Console.WriteLine("任务1: 已获取 锁2");
        }
    }
}
//任务2
public static async Task Task2()
{
    //线程2 首先获取 锁2
    lock (lock2)
    {
        Console.WriteLine("任务2: 已获取 锁2");
        //模拟一些操作
        Task.Delay(100).Wait();
        // 任务2 等待 锁1
        Console.WriteLine("任务2: 等待获取 锁1");
        lock (lock1)
        {
            Console.WriteLine("任务2: 获取 锁1");
        }
    }
}

```

***注***：测试方法代码以及示例源码都已经上传至代码库，有兴趣的可以看看。[https://gitee.com/hugogoos/Planner](https://github.com)


 本博客参考[slower加速器](https://jisuanqi.org)。转载请注明出处！
