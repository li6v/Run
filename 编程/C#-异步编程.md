# AsyncFixer

使用async和await进行异步编程，我们容易由于一知半解或疏忽大意，编写出有毛病（如存在死锁可能性，低效率，忽略了异步异常等）的代码。为项目添加Nuget包`AsyncFixer`可以检查出这些毛病以编译错误的方式提醒我们去纠正。

# 用GetAwaiter().GetResult()不要用Wait()

Task会隐藏自己的异常，直到等待Task结束时抛出。等待Task结束有两种方式。

方式一：task.Wait();

```csharp
static void Main(string[] args)
{
    try
    {
        Task task = Task.Run(() => { throw new InvalidOperationException("非法操作"); });
        task.Wait();
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.ToString());
    }
}
```

输出

```tex
System.AggregateException: One or more errors occurred. (非法操作)
 ---> System.InvalidOperationException: 非法操作
   at 异步测试.Program.<>c.<Main>b__0_0() in C:\Users\megarobo\Desktop\测试--n\异步测试\Program.cs:line 12
   at System.Threading.Tasks.Task`1.InnerInvoke()
   at System.Threading.Tasks.Task.<>c.<.cctor>b__274_0(Object obj)
   at System.Threading.ExecutionContext.RunFromThreadPoolDispatchLoop(Thread threadPoolThread, ExecutionContext executionContext, ContextCallback callback, Object state)
--- End of stack trace from previous location where exception was thrown ---
   at System.Threading.ExecutionContext.RunFromThreadPoolDispatchLoop(Thread threadPoolThread, ExecutionContext executionContext, ContextCallback callback, Object state)
   at System.Threading.Tasks.Task.ExecuteWithThreadLocal(Task& currentTaskSlot, Thread threadPoolThread)
   --- End of inner exception stack trace ---
   at System.Threading.Tasks.Task.ThrowIfExceptional(Boolean includeTaskCanceledExceptions)
   at System.Threading.Tasks.Task.Wait(Int32 millisecondsTimeout, CancellationToken cancellationToken)
   at System.Threading.Tasks.Task.Wait()
   at 异步测试.Program.Main(String[] args) in C:\Users\megarobo\Desktop\测试--n\异步测试\Program.cs:line 13
```

方式二：task.GetAwaiter().GetResult()

```csharp
static void Main(string[] args)
{
    try
    {
        Task task = Task.Run(() => { throw new InvalidOperationException("非法操作"); });
        task.GetAwaiter().GetResult();
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.ToString());
    }
}
```

输出

```tex
System.InvalidOperationException: 非法操作
   at 异步测试.Program.<>c.<Main>b__0_0() in C:\Users\megarobo\Desktop\测试--n\异步测试\Program.cs:line 12
   at System.Threading.Tasks.Task`1.InnerInvoke()
   at System.Threading.Tasks.Task.<>c.<.cctor>b__274_0(Object obj)
   at System.Threading.ExecutionContext.RunFromThreadPoolDispatchLoop(Thread threadPoolThread, ExecutionContext executionContext, ContextCallback callback, Object state)
--- End of stack trace from previous location where exception was thrown ---
   at System.Threading.ExecutionContext.RunFromThreadPoolDispatchLoop(Thread threadPoolThread, ExecutionContext executionContext, ContextCallback callback, Object state)
   at System.Threading.Tasks.Task.ExecuteWithThreadLocal(Task& currentTaskSlot, Thread threadPoolThread)
--- End of stack trace from previous location where exception was thrown ---
   at 异步测试.Program.Main(String[] args) in C:\Users\megarobo\Desktop\测试--n\异步测试\Program.cs:line 13
```

总结

1. Wait( )会把Task抛出的异常封装进AggregateException，隐藏了真正的异常，不利于排查问题。
2. GetAwaiter( ).GetResult( )会抛出真正的异常，优于Wait()。

# await优于GetAwaiter().GetResult()

await会使宿主方法提前返回，让当前线程执行调用者的下一步，并没有发生线程阻塞，没有上下文切换。GetAwaiter().GetResult()会导致当前线程阻塞，发生了上下文切换，并多产生一个空闲的线程什么也不干。

磁盘和网络IO操作，既不需要线程，也不需要CPU。

# 无await则勿用async



# 异步方法的返回值不允许是void

如果异步方法的返回值是void，则调用者无法知道异步方法何时执行结束，以及执行成功还是失败，并且异步方法如果抛出异常，此异常无法被Try Catch捕获，直接抛到CLR导致进程终止。可以用`AppDomain.CurrentDomain.UnhandledException`在进程终止前打印这种异常。

```csharp
static void Main(string[] args)
{
    AppDomain.CurrentDomain.UnhandledException += CurrentDomain_UnhandledException;
    try
    {
        TestAsync();
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex);
    }

    Console.ReadLine();
}

private static void CurrentDomain_UnhandledException(object sender, UnhandledExceptionEventArgs e)
{
    Console.WriteLine((e.ExceptionObject as Exception).ToString());
    Console.ReadLine();
}

static async void TestAsync()
{
    throw new Exception("异常1");
    await Task.Run(() =>
    {
        throw new Exception("异常2");
    });
}
```

```tex
在异步方法TestAsync()中，无论抛出异常1还是异常2，Main()方法中的Try Catch均无法捕获。AppDomain的全局异常处理程序执行结束后，异常被抛到CLR从而进程终止。
```

# 为什么仍旧存在 async void方法



在事件模型中，事件的发布者不关心订阅者的执行结果，所以事件的委托签名往往都是void。同时，事件模型充斥着整个.NET。

以Button的Click事件为例。

```csharp
public delegate void RoutedEventHandler(object sender, RoutedEventArgs e)
```

我们的事件处理程序对应的方法被迫是void，如果调用







await的方法并没有完成，后续要检查成功或失败，void则没有机会，且执行失败，异常直接提交到CLR，导致进程崩溃。

但是一些事件委托允许的只有void方法，迫不得async void，那么应当，try catch方法的所有代码，除非你确保方法不可能出现异常，或者知道该异常出现时是不可思议的，防御性编程。

# 避免在同步方法中调用异步方法

同步方法必然是void，且无async，所以不能使用await，那么只能wait，但是wait会可能会导致死锁。

































通常，我会尽力避免对异步任务进行同步阻塞。但是，在少数情况下，我确实违反了该准则。在那些罕见的情况下，我的首选方法是`GetAwaiter().GetResult()`因为它保留任务异常，而不是将它们包装在中`AggregateException`。











# 异步方法的返回值不允许是void

Button的Click事件是void，怎么办？

加 async，内部用await，但是会抛出异常，终止程序，所以内部要try catch。

不加async, GetWaiter.GetResult().

# 同步方法如何调用异步方法













# return Task 比 return await Task效率高











# 什么样的代码块适合异步方法

1. 不需要CPU和线程资源的IO操作，如磁盘，网络。
2. 耗时操作，且明知道



# ReaderWriterLockSlim

## 如何选择Lock和ReaderWriterLockSlim

- `lock`比`ReaderWriterLockSlim`写起来更简洁
- 对临界资源的访问，读比写的频率越高，越推荐`ReaderWriterLockSlim`
- EnterUpgradeableReadLock/ExitUpgradeableReadLock是进一步优化读并发能力的手段。

---

## 临界资源

在程序的运行过程中,可能发生多个线程同时访问的资源称之为临界资源。如果所有线程都是读临界资源，那么资源无需被保护，不需要加锁。如果存在写线程与读线程，写线程与写线程并发的可能，则必须对临界资源加锁。

什么是临界资源呢？

① 多个线程累加变量 sum，则变量sum就是临界资源。

② 文件是临界资源，操作文件的实例便是临界资源，如FileStream，StreamReader，StreamWriter。

如何加锁？

保护临界资源的原则是①读临界资源期间，不会发生写临界资源；②写临界资源时，不会发生读临界资源；③允许多个线程同时读临界资源。其中①和②必须要满足，③不是必须的，但是如果满足③可以大幅度提高程序的并发性能和吞吐量。具体措施就是表示临界资源的句柄的周围都要加锁，无论通过句柄是写还是读临界资源。

## lock

读写临界资源时，需要采用`lock`关键字保护起来，如 

```csharp
lock()
{
    // 读临界资源
}
lock()
{
    // 写临界资源
}
lock()
{
    // 读写临界资源
}
```

只要出现指向临界资源的变量或引用都需要用`lock`圈住。这会导致某一时刻仅有一个线程运行，其他线程都被阻塞。

`lock`的缺点是满足了条件①和②，虽然保证了安全性，但是因为不满足条件③，导致`读比写多`的场景下效率很低。

`ReaderWriterLockSlim`同时满足条件①②③，其性能比`lock`更高，尤其是读远比写多的场景。

## `ReaderWriterLockSlim`的基本使用原则

- 读临界资源的代码使用`EnterReadLock`,写临界资源的代码使用`EnterWriteLock`,先读临界资源，满足一定条件再决定是否写临界资源的场景使用`EnterUpgradeableReadLock`.
- 对于同一把锁、多个线程可同时进入读模式，同时只允许一个线程进入写模式。如果一个线程进入可升级的读模式，则不允许其他线程进入写或可升级读模式，但仍旧可以进入读模式。
- 可升级的读模式，进一步加强了读并发的能力。针对先读临界资源满足一定条件再决定是否写临界资源的场景，如果没有可升级的读模式，直接使用写模式，即使后续没有发生写临界资源，也会导致所有读临界资源的线程被阻塞，这是没必要的。可升级的读模式，可以避免这种情况。**先读但是后续发生写操作的可能性越低，可升级的读模式越有优势。**
- 可升级的读模式，未升级前，允许读线程，禁止写线程和可升级的读模式线程；升级时，可能要等待其他的读线程全部退出或部分退出后，才能进入写模式。升级后进入写模式，禁止读，写，可升级读模式。
- 一个线程进入可升级的读模式后，直到退出前，不可能有其他线程对临界资源进行了写操作。
- 读写锁具有线程关联性，即两个线程间拥有的锁的状态相互独立不受影响、并且不能相互修改其锁的状态。

## 递归规则

`ReaderWriterLockSlim`支持递归和非递归，可以在构造函数内传入递归策略。

```csharp
ReaderWriterLockSlim Lock = new ReaderWriterLockSlim(LockRecursionPolicy.NoRecursion); // 不传参默认是不递归
```

非递归情况下，同一个线程必须先Enter后Exit，如果出现连续两次Enter，便会抛出异常。这就要求我们在编写方法时，一定保证先Enter后Exit的次序，尤其注意在finally里面Exit。可升级锁是个例外，可以先EnterUpgradeableReadLock，再EnterWriteLock。

递归情况下，锁会保存线程和该线程进入锁的次数的计数信息，从而允许在同一个线程中连续运行两次EnterReadLock，或者连续两次EnterWriteLock,但也不可以在进入读模式后再次进入写模式或者可升级的读模式（在这之前必须退出读模式）。

**强烈建议永远不要使用递归策略，容易发生死锁，同时锁的性能更低。**

经典使用案例

```c#
public class SynchronizedCache 
{
    private ReaderWriterLockSlim cacheLock = new ReaderWriterLockSlim();
    private Dictionary<int, string> innerCache = new Dictionary<int, string>();

    public int Count
    { get { return innerCache.Count; } }

    public string Read(int key)
    {
        cacheLock.EnterReadLock();
        try
        {
            return innerCache[key];
        }
        finally
        {
            cacheLock.ExitReadLock();
        }
    }

    public void Add(int key, string value)
    {
        cacheLock.EnterWriteLock();
        try
        {
            innerCache.Add(key, value);
        }
        finally
        {
            cacheLock.ExitWriteLock();
        }
    }

    public bool AddWithTimeout(int key, string value, int timeout)
    {
        if (cacheLock.TryEnterWriteLock(timeout))
        {
            try
            {
                innerCache.Add(key, value);
            }
            finally
            {
                cacheLock.ExitWriteLock();
            }
            return true;
        }
        else
        {
            return false;
        }
    }

    public AddOrUpdateStatus AddOrUpdate(int key, string value)
    {
        cacheLock.EnterUpgradeableReadLock();
        try
        {
            string result = null;
            if (innerCache.TryGetValue(key, out result))
            {
                if (result == value)
                {
                    return AddOrUpdateStatus.Unchanged;
                }
                else
                {
                    cacheLock.EnterWriteLock();
                    try
                    {
                        innerCache[key] = value;
                    }
                    finally
                    {
                        cacheLock.ExitWriteLock();
                    }
                    return AddOrUpdateStatus.Updated;
                }
            }
            else
            {
                cacheLock.EnterWriteLock();
                try
                {
                    innerCache.Add(key, value);
                }
                finally
                {
                    cacheLock.ExitWriteLock();
                }
                return AddOrUpdateStatus.Added;
            }
        }
        finally
        {
            cacheLock.ExitUpgradeableReadLock();
        }
    }

    public void Delete(int key)
    {
        cacheLock.EnterWriteLock();
        try
        {
            innerCache.Remove(key);
        }
        finally
        {
            cacheLock.ExitWriteLock();
        }
    }

    public enum AddOrUpdateStatus
    {
        Added,
        Updated,
        Unchanged
    };

    ~SynchronizedCache()
    {
       if (cacheLock != null) cacheLock.Dispose();
    }
}
```





























