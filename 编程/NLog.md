[TOC]
# 搜索配置文件的规则
* NetCore和NetFramework都可以的配置方法
  （1）配置文件名称必须是 nlog.config
  （2）配置文件必须放到程序目录  application's directionary

  ==启动应用程序，NLog自动去程序目录下寻找nlog.config加载，如果找不到，NLog无法打印日志。（建议在项目下添加文件nlog.config，文件属性设置成：生成操作 -> 内容。复制到输出目录-> 如果较新则复制。这样每次编译项目，nlog.config都会自动被复制到exe目录。）==

* nlog支持在net core的appsettings.json配置NLog，这种方式相比nlog.config的xml配置方式并没有啥优势，闲的时候可以试一下。

# nlog.config智能感知

如果在配置nlog.config时，想利用智能感知，那么不仅要安装NLog，还要安装NLog.Schema，项目的目录下会多出一个NLog.xsd，在xml中引入命名空间，并在定义节点时使用xsi:type。

```xml
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
	<!-- configuration goes here -->
	<targets>
		<target xsi:type="File" ></target>
	</targets>
</nlog>
```



![image-20221102165813431](D:\OneDrive\Run\images\image-20221102165813431.png)

<img src="D:\OneDrive\Run\images\image-20221102165751322.png" alt="image-20221102165751322" style="zoom:50%;" />



# 配置文件配错了怎么办？
throwException="true"，NLog检测到配置错误，会抛出异常。否则，不会抛出异常且不打印任何日志。
所以建议配置成true,有错误及早发现。

```xml
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" throwExceptions="true">
```
# 利用自定义变量简化配置
自定义变量可以保存常量和上下文变量组成的字符串，在配置文件中复用，避免重复配置长字符串显得冗余。
自定义变量的value可包含另一个自定义变量和上下文变量。
<img src="D:\OneDrive\Run\images\1037641-20220808100706901-2085593277.png" alt="image" style="zoom:30%;" />
# 缓冲区
```xml
<targets>
  <target xsi:type="BufferingWrapper"
          name="String"
          bufferSize="Integer"
          flushTimeout="Integer"
          slidingTimeout="Boolean"
          overflowAction="Enum">
    <target xsi:type="wrappedTargetType" ...target properties... />
  </target>
</targets>
```
BufferingWrapper,日志先被存放到一个带锁的缓存队列Queue中，然后Logger.Log()立刻返回，等到队列放满了，会将队列中的日志全部取出，一次性同步(在业务代码线程上)flush到被封装的下一层Target。
`bufferSize`: 缓存队列的长度限制，默认值是100.
`flushTimeout`: 避免队列长时间未满，缓存中的日志长时间滞留在内存中不写入目标，在上一次队列中日志被一次性flush出去后开始计时，计时器到，即使队列没有放满，也会进行一次异步flush。默认值是-1，关闭此功能。单位是毫秒。<font color=red>笔者建议永远关闭此功能，始终使用默认值-1.</font>
`overflowAction`: 有两种值1.Flush，队列满就flush。2.Discard，删除队列中时间最早的一批日志。默认值是flush。<font color=red>笔者建议坚持使用默认值flush.</font>

**1. logger.Log()调用BufferingWrapper的`protected override void Write(AsyncLogEventInfo logEvent)`，每次一条的放入BufferingWrapper的缓存队列中，当检测到队列满时，取走队列中的所有日志flush到被封装的内部Target。被封装的Target调用public void WriteAsyncLogEvents(IList<AsyncLogEventInfo> logEvents)，批量写入，由于Target的批量写入日志的API的默认实现是遍历批量日志，逐条写入Target，所以，被封装的自定义Target最好要重写 Write(IList<logEvents>)才能更好的发挥出缓存队列的作用。另外，BufferingWrapper只是避免瞬时打印大量日志导致缓存满日志被丢弃而设计的，只是延迟将日志写入Target，后期仍旧占用业务线程将缓存的日志flush到Target.**
**2. 日志放入BufferingWrapper，以及取出，以及被封装的Target批量写入日志都发生在与打印日志的业务方法处于同一线程，不存在异步写入日志的情况。(flushTimeout大于0，就会异步，建议不要开启它。)**

# 异步日志

队列放大，批次放大，批次次数放大，定时器放小，定时器的时间会动态调整，队列空会自动暂停，来日志了再开启，队列多，再一个定时任务中，会取若干个批次。 为了避免内存爆炸，建议设置成Block或Discard,监控discard事件. 批次大了，也减少了入Target锁出Target锁的频率。但是队列队列过大，可能导致CloseTarget时，flush超时。

AsyncTargetWrapper会将日志逐条暂存到缓存队列（如果BufferingWrapper包裹的AsyncTargetWrapper会批量写入），它内部有个计时器，每隔指定毫秒就从队列取出若干批次的日志，写入被封装的Target.

```tex
定时器到
		for  取0~N个批次（fullBatchSizeWriteLimit）
				取一批次日志写入被封装的Target（batchSize）
				重新计时
```

`queueLimit`:

缓存队列的长度，默认值是10000. 

如果定时器处理不及缓存日志，队列已满，那么队列有3种运行方式：

1. 丢弃新来的日志。
2. 自动扩增队列容量，继续接收新日志，可能导致占用大量内存,等到日志量下降后，再将队列长度重新恢复到设置值。
3. 阻塞日志API（日志不会丢失，但可能暂时阻塞正常的业务代码运行）

`timeToSleepBetweenBatches`

隔多久从缓存队列中取一次批量日志写入被封装的Target,单位是毫秒，默认值是1。内部使用的定时器是System.Threading.Timer， 每次批量写入日志后重新计时，不会存在多个定时任务同时在线程池中执行的情况。另外，定时器的周期虽然是1ms，但是Windows是非实时操作系统，实际周期可能在20ms-100ms，所以建议使用默认值就好。

`batchSize`

每次定时任务单次从队列中取出多少条日志作为一个批次写入被封装的Target，默认值是100.

`fullBatchSizeWriteLimit`

每次执行定时任务，从队列中取几次批量日志写入被封装的Target，默认值是5。假设取5次，这5次批量写入在定时任务内是串行执行的，全部执行结束，才开始重新计时，等待触发下一次定时任务。

`overflowAction`

缓存队列满了，有3种策略：Discard，Block，Grow。默认值是Discard。

<font color=red>如果使用AsyncTargetWrapper封装了Target,就务必不要再配置`<targets async="true">`，这样会使日志性能大幅下降。</font>

当写入的时候，首先把文件写入到这个队列，然后启动定时器，定时器的工作是从这个队列一次读取一条或者多条数据，把这个数据传递给被封装的Target。为了提高效率会在文件写入读取的时候，会根据队列中的数据动态调整定时器的执行周期。如，队列数据读取完，为空的时候，会禁用定时器。当有数据写入的时候有会开启定时器。

# 缓冲区BufferingWrapper和异步AsyncTargetWrapper的区别

1. 日志API单条写入BufferingWrapper的缓存队列，缓存队列满，取出队列中的全部日志，调用被封装的Target的批量写入日志API进行flush,这些操作都是同步的，与业务代码处于同一线程。
2. 日志API单条写入AsyncTargetWrapper的缓存队列。定时器到，在线程池中的线程中，取出队列中的一部分日志(不大于队列中的全部日志总量)，通过被封装的Target的批量写入日志API进行flush。
3. 由于AsyncTargetWrapper是在线程池中取缓存日志写入Target，不占用主线程，所以，它能提高业务代码的执行效率提高整个日志系统的性能。BufferingWrapper将日志临时存储到缓存队列，让打印日志的API立即返回，但是在后期仍旧占用主线程批量写入日志，所以它有削峰的作用，避免日志被丢弃，但是不会提高业务代码整体的执行效率。
4. AsyncTargetWrapper对整个应用程序的多个并发线程打印日志的场景具有很好的优化效果，如果整个线程只有1个或极少量的线程打印日志，开启异步可能反而让打印日志的性能降低。
5. 被BufferingWrapper和AsyncTargetWrapper封装的Target，最好重写默认的批量写入日志的接口，以发挥缓存后批量写入的威力。但是AsyncTargetWrapper包裹的Target即使未重写批量写入日志的接口，由于是在线程池中写入日志，未占用主线程，AsyncTargetWrapper仍旧能提高整体的日志性能。
6. 自定义Target如果想被BufferingWrapper和AsyncTargetWrapper封装，需要继承TargetWithContext，而不是TargetWithLayout。

# 如何自定义Target

Target写日志方法链

<img src="D:\OneDrive\Run\images\1037641-20220808100813250-881256497.png" alt="image" style="zoom:40%;" />


**自定义Target的核心知识讲解**

自定义Target很简单，重写`void Write(LogEventInfo logEvent)`即可。

如果自定义Target没有被BufferingWrapper和AsyncTargetWrapper包裹，logger.Log()走的是[单条日志写入Target方法链]，logger.Log()在业务线程上执行，`void Write(LogEventInfo logEvent)`未返回，logger.Log()不返回，如果`void Write(LogEventInfo logEvent)`的执行效率很低，会明显阻塞影响业务代码的执行。

多线程调用logger.Log()打印日志，`void WriteAsyncThreadSafe(AsyncEventInfo logEvent)`保证了同一时间始终只有一个logger.Log()占用临界资源Target。所以有3个结论：① 多线程打印日志，经过`void WriteAsyncThreadSafe(AsyncEventInfo logEvent)`间接调用`void Write(LogEventInfo logEvent)`是必须的，即使重写`void WriteAsyncThreadSafe(AsyncEventInfo logEvent)`，也要保证方法内部的第一行使用lock(this.SyncRoot)，独占式占用Target. ② `void WriteAsyncThreadSafe(AsyncEventInfo logEvent)`由于需要入锁出锁，导致多个线程的logger.Log()阻塞到这个锁上等待写入Target，如果我们实现的`void Write(LogEventInfo logEvent)`效率很低，会进一步放大阻塞业务代码的灾难。

如果自定义Target被BufferingWrapper包裹，logger.Log()走的是[批量日志写入Target方法链]，在BufferingWrapper的缓存队列未满时，logger.Log()将日志放入缓存队列立刻返回，不会占用业务线程。如果BufferingWrapper的缓存队列还差1条日志就满时，下一条`logger.Log()`会将日志放入队列，但它并不立刻返回，而是取出队列中的所有日志，调用自定义Target的`void WriteAsyncLogEvents(IList<AsyncLogEventInfo> logEvents)`批量写入，所以调用让队列刚好满载的logger.Log()的业务线程，会被阻塞直到批量写入完成。但是队列内的日志已经被清空，其他线程调用logger.Log()仍旧是直接放到缓存队列立刻返回不会被阻塞。另外，`void WriteAsyncThreadSafe(IList<AsyncLogEventInfo> logEvents)`同样有锁，保证在批量写入Target时是独占Target的，避免刚腾空的缓存又被填满在其他线程执行批量写入Target导致并发写入Target。至此我们可以发现，BufferingWrapper最终还是要占用业务线程的时间，笔者觉得，只有自定义Target重写后的批量写入方法比默认的逐条写入的实现具有较大的性能优势的场景下，BufferingWrapper对性能有提升外，暂时没发现它的明显好处。NLog官方说，BufferingWrapper可以降低日志被丢弃的可能性。

如果自定义Target被AsyncTargetWrapper包裹，logger.Log()走的是[批量日志写入Target方法链]。AsyncTargetWrapper内置了一个队列和定时器，logger.Log()将日志放入队列后立刻返回，定时器定时在线程池线程取走队列中的若干批次的日志，调用其封装的自定义Target的`WriteAsyncLogEvents(IList<AsyncLogEventInfo> logEvents)`批量写入。可以看到，AsyncTargetWrapper可以让任何业务线程的logger.Log()立刻返回，即使自定义的Target的`void Write(LogEventInfo logEvent)`效率较差，也是阻塞在线程池线程上，并不会阻塞业务线程，所以AsyncTargetWrapper能够大幅度的提高日志系统的整体性能。AsyncTargetWrapper的定时器类似于UI定时器，定时任务未完成，不会执行下一次的定时任务，如果`void Write(LogEventInfo logEvent)`的执行效率极低非常耗时，会导致定时器无法及时取走队列中的日志，队列很快就满载，队列满载后，有3种策略，1. Discard，新日志不再放入队列，直接被丢弃，这会造成日志漏打缺失；2. Block，阻塞业务流程中的logger.Log()，这会导致正常的业务代码长时间阻塞影响整个应用程序；3. Grow，自动扩增队列长度，新日志仍旧放入队列，这可能会导致内存爆炸。所以`void Write(LogEventInfo logEvent)`的实现好坏非常重要，注意点有：<font color =red>1. 应当竭尽全能的快。2. 应当有超时机制，长时间执行不完应当取消。3. 不可开个Task去Write就立刻返回，虽然这样会显的很快，但是`WriteAsyncLogEvents(IList<AsyncLogEventInfo> logEvents)`是循环遍历logEvents调用logEvents.Count次`void Write(LogEventInfo logEvent)`逐条写入的，开个Task就返回可能会导致打印的日志乱序，在文本中晚产生日志打印在早产生的日志前面。4. 抛出异常，NLog并不会崩溃，而是忽略此条日志，但是，Write方法抛出异常应当适当的捕获做一些处理，如稍加延时后尝试重新连接网络、重新打开数据库、删除磁盘文件成功后再次重新Write，临时将日志存储到其他位置后续手动导入，记录异常信息便于后续排错，甚至采用另一种可靠的机制通知工程师日志系统出了问题。</font>

自定义Target，一般情况下重写`void Write(LogEventInfo logEvent)`即可，单条日志写入自定义Target，会调用此API，批量日志写入自定义Target的API  `WriteAsyncLogEvents(IList<AsyncLogEventInfo>`具有默认实现:循环遍历logEvents调用logEvents.Count次`void Write(LogEventInfo logEvent)`逐条写入。如果对日志性能要求很高，可以同时重写`WriteAsyncLogEvents(IList<AsyncLogEventInfo>`，最典型的例子就是把日志写到数据库，批量写入比逐条写入的性能高8-50倍。

**重写`WriteAsyncLogEvents(IList<AsyncLogEventInfo>`的注意事项**

```csharp
protected override void Write(IList<AsyncLogEventInfo> logEvents)
{
    string sql = "";
    // 不可以将logEvents分批开多个Task处理，这可能造成并发占用Target导致异常
    foreach (var item in logEvents)
    {
        // 假设这是拼接SQL字符串的操作
        sql += RenderLogEvent(this.Layout, item.LogEvent);
    }
    try
    {
        SendDatabase(sql);
    }
    catch (Exception ex)
    {
        // 实现重连数据库，重新执行SQL
        // 多次尝试都失败，发邮件给工程师，并以本地文件的形式记录SQL，在后期手工导入数据库。
        // 记录异常的类型和信息，后期排错用
    }
}
```

综上：
对日志要求可靠性极高的情况，只重写`void Write(LogEventInfo logEvent)`即可，且不要使用NLog内置的AsyncWrapper、BufferingWrapper、AutoFlushWrapper 、RetryingWrapper、FilteringWrapper、FallbackGroup等外围Target包裹自定义Target。

对日志可靠性要求略高且追求性能的情况，重写`void Write(LogEventInfo logEvent)`，并用AsyncWrapper包裹自定义Target，根据实际情况，考虑是否重写`void Write(IList<AsyncLogEventInfo> logEvents)`来进一步提高自定义Target的性能，假设自定义Target是将日志写入到数据库，就特别有必要重写`void Write(IList<AsyncLogEventInfo> logEvents)`，因为批量行插入和逐行插入Database的速度有着几十倍的差异。

海量高频率的日志可以考虑使用AsyncWrapper->BufferingWrapper->ActualTarget的结构，进一步增强处理瞬时大量日志的能力。logger.Log()将日志逐条放入AsyncWrapper的缓存队列，AsyncWrapper定时取批量日志，调用BufferingWrapper的Write(logevents)逐条写入BufferingWrapper的缓存队列。BufferingWrapper取批量日志，调用ActualTarget的Write(logevents)写入。简而言之，就是为ActualTarget使用两级缓存，logger.Log()将日志逐条放入缓存，最终，从缓存取出批量日志使用ActualTarget的Write(logevents)批量写入。

务必重写`void Write(LogEventInfo logEvent)`，让自定义Target支持同步单条写入,即无BufferWrapper和AsyncWrapper包裹的配置模式，应当实现超时监控，异常记录，异常备份，异常通知，异常重连，异常重写功能，虽然方法抛出异常不影响NLog正常工作，但还是不建议抛出异常。另外，方法返回前，务必要保证方法内的工作已全部完成，以避免晚打印的日志比早打印的日志先流入Target.

由于Write(logEvents)的默认实现方式是在遍历logEvents集合逐条调用Write(logEvent)，性能并不高，如果重写`void Write(IList<AsyncLogEventInfo> logEvents)`,让其内部不调用logEvents.Count次Write(logEvent)，而是拼接好logEvents一下子写入Target的方式更佳，可以考虑也重写`void Write(IList<AsyncLogEventInfo> logEvents)`。同样要遵守实现超时监控，异常记录，异常备份，异常通知，异常重连，异常重写功能，虽然方法抛出异常不影响NLog正常工作，但还是不建议抛出异常。另外，方法返回前，务必要保证方法内的工作已全部完成，以避免晚打印的日志比早打印的日志先流入Target，而且不要在方法内部调用base.Write(logEvents)或this.Write(logEvent)等。

Target的初始化和清理工作放到构造方法，析构方法，InitializeTarget()和CloseTarget()中。

不要使用NLog提供的RetryTarget，它会将批量写入转换成单条写入，妈的，性能太慢了！其他的Target也不建议使用，封装的多会降低性能。

```csharp
/* 自定义一个Target的步骤：
 * 1. 继承TargetWithLayout，如果要为Target添加不确定数量的上下文变量,就继承TargetWithContext
 * 2. 为自定义Target类添加Target特性，参数是target类型标记，用于配置文件  xsi=特性名称
 * 3. 配置文件中配置 extensions节点，将自定义Target所在的程序集配置好，这一步必须做，不然NLog无法利用自定义Target
 */

[Target("network_tcp")]
public class TcpTarget : TargetWithContext
{
    [RequiredParameter] // RequiredParameterAttribute修饰的属性，在配置文件中必须为此Target配置此参数，否则无法正常打印日志
    public string Adderss { get; set; }

    [RequiredParameter] // RequiredParameterAttribute修饰的属性，在配置文件中必须为此Target配置此参数，否则无法正常打印日志
    public int Port { get; set; }
    
    // 可不配置此参数
    public string LineEnding { get; set; }

    // 程序启动时，执行一次
    protected override void InitializeTarget()
    {
	    base.InitializeTarget();
        Console.WriteLine("TcpTarget Initialize");
    }

    // 关闭程序时，执行一次
    protected override void CloseTarget()
    {
        // 一定要调用基类的CloseTarget
        base.CloseTarget();
        Console.WriteLine("TcpTarget Close");
    }

    // 程序启动时，执行一次
    public TcpTarget()
    {
        Console.WriteLine("TcpTarget构造方法");
    }

    protected override void Write(LogEventInfo logEvent)
    {
        // Layout是成员变量。 Address是成员变量  Port是成员变量
        // 如果直接使用成员变量，获取的是配置的字符串，举例：在配置文件中配置Address="${level}"
        // this.Address的值是${level}，是Raw值。获取配置绑定的实际上下文变量，需要利用RenderLogEvent()渲染一下，
        // 把${level}渲染成Error，Warn,Info等。
        // layout表示msg的标记，需要渲染

        string logMessage = RenderLogEvent(this.Layout, logEvent);
		// 第2个参数logEvent也很重要，因为Adderss可能绑定在logEvent携带的上下文变量上
        string address = RenderLogEvent(this.Adderss, logEvent);

        // 定义的成员变量毕竟有限，可以动态添加成员变量。
        // 支持动态数量的上下文环境变量
        object machineName = ContextProperties.First(item => item.Name == "MachineName").RenderValue(logEvent);
        object loggerName = ContextProperties.First(item => item.Name == "LoggerName").RenderValue(logEvent);

        Console.WriteLine($"Address={address}," +
            $" Port={Port}, " +
            $"MachineName={machineName}," +
            $"loggerName={loggerName}"+
            $"Msg={logMessage}");
    }
    
    // 建议重写批量写入日志的API，默认实现遍历logEvents，调用logEvents.Count次void Write(LogEventInfo logEvent)
    protected override void Write(IList<AsyncLogEventInfo> logEvents)
    {
        
    }
}
```

步骤（2）：在配置文件中使用自定义Target
注意事项：
* 自定义Target,自定义renderer，自定义logger，要想让NLog使用这些自定义类，必须要将这些自定义类型所在的dll路径配置到extensions节点，这样NLog在启动后，才能检索这些类型的信息，加以利用。
* 自定义Target的一些[RequiredParameterAttribute]属性要求必须在配置文件中提供值或绑定到NLog内置的上下文变量或通过自定义renderer和logeventinfo的properties添加的上下文变量，否则，自定义Target无法工作。
* 自定义Target若有ContextProperties字典，可配置其name和layout，layout决定绑定到哪个上下文变量或使用普通不变文本，name用于检索。

```xml
<?xml version="1.0" encoding="utf-8" ?>
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

<!--必须配置类型所在的dll路径，否则找不到-->
	<extensions>
		<add assembly="MyTarget"/>
	</extensions>

	<targets>
		<target xsi:type="network_tcp" name="test" Adderss="127.0.0.1" 
				Port="33066" layout="${message}">
			<!--Target的上下文变量-->
			<contextproperty name="MachineName" layout="${machinename}" />
			<contextproperty name="LoggerName" layout="${logger}" />
		</target>
	</targets>

	<rules>
		<logger name="*" writeTo="test" />
	</rules>
</nlog>
```
步骤（3）：打印日志即可
```csharp
static void Main(string[] args)
{
    Logger logger = LogManager.GetCurrentClassLogger();

    for (int i = 0; i < 1000; i++)
    {
        logger.Error($"Hello,Nlog Target{i}...");
    }
}
```

# Remember to flush

养成好习惯，无论您是怎样配置NLog的，一定要在应用程序退出前，调用 `NLog.LogManager.Shutdown();`不然，缓存中的日志可能会丢失。

# 优化写入磁盘文件操作
```xml
<?xml version="1.0" encoding="utf-8" ?>
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

	<targets>
		<target xsi:type="AsyncWrapper"
		        name="asyncfile"
		        queueLimit="10000"
		        timeToSleepBetweenBatches="1"
		        batchSize="200"
		        overflowAction="Block">
			<!--overflowAction=Grow，NLog的并发队列满了，会自动增大缓存队列，这可能导致内存被占满。
			overflowAction=Block，NLog的并发队列满了，会阻塞打印日志的方法，直到并发队列空闲了。这可能导致日志阻塞正常业务代码的执行效率。
			overflowAction=Discard，NLog的并发队列满了，直接丢弃新来的日志缓存。这样虽然不会造成阻塞，但是会造成日志丢失。
			-->
			<target
				xsi:type="File"
				fileName="D:/TESTNLOG/${shortdate}/${logger}.log"
				name="file"
				cleanupFileName="false"
				encoding="utf-8"
				concurrentWrites="false"
				bufferSize="327680"
				keepFileOpen="true"
				autoFlush="false"
				openFileFlushTimeout="3"
				archiveAboveSize="102400000"
				archiveFileName="D:/TESTNLOG/${shortdate}/achieves/${logger}.{###}.log"
				archiveNumbering="Sequence"
				maxArchiveFiles="100"
				layout="[${longdate} %${uppercase:${level:padding=-5}}% ${callsite:fileName=true:include:fileName=true:includeSourcePath=false} ${message}" />
		</target>
	</targets>
	<rules>
		<logger name="*" minlevel="Debug" writeTo="asyncfile" />
	</rules>
</nlog>
```
缓存有两种，一种是NLog层级的，一种是操作系统层级的。
一种是NLog内置的并发队列缓存，达到缓存或定时器到钟，就获取文件句柄打开文件，然后从并发队列中取一个批次的日志，一次性写入。
另一种缓存，是操作系统为写文件搞的缓存。

This FileName string is a layout which may include instances of layout renderers. This lets you use a single target to write to multiple files.

`encoding` : 文件字符集。如果日志含有中文，设置成gbk,utf-8等，设置成ansii会乱码。默认值时utf-8.

`writeBom` : BOM是一种若干字节的标记，放在文件头，文本编辑器读取文件时，能够通过BOM得知文件的字符集，然后解析显示。如果文件BOM，文本编辑器只能读取一点字节尝试得知是什么字符集，存在解析错误的可能性。BOM是一种在Windows平台的古老技术，BOM字节会被当做有效数据，导致浏览器出错，不建议使用。默认值是 false，也就是文件不带BOM。应该使用默认值false。

`lineEnding` : 调用NLog的API写入日志，都是写入一行，结尾自动加换行符。Windows，MacOS，Linux的换行符不同。
CR，CRLF,Default 根据操作系统插入相应的换行符,LF,None 不插入换行符

`keepFileOpen` : 设置成false，NLog每写入一条日志或一个批次的(多条)日志，便会关闭文件，下次写入再重新打开，关闭，打开，关闭.... 显然这没有一直保持占用文件句柄不关闭性能高。其次，设置成false，那么就有可能让别的进程乘机而入占用句柄，导致NLog写入日志失败。
默认是true，建议显示设置成true.

`openFileCacheTimeout` : `keepFileOpen` = true的时候这个参数才有作用。它表示日志文件保持打开多久就自动flush下缓存关闭一次文件。负数表示永远不自动关闭，默认值是-1. 推荐就使用默认值-1.

`openFileCacheSize` : `keepFileOpen` = true的时候这个参数才有作用。 它表示应用中任意一个时刻，`keepFileOpen` = true的文件数量的上限，默认值是5. 它的值越大，对于一个Target但是因为路径中包含上下文变量会写入多个不同的场景有性能提高的好处。但也不宜过大，不建议超过10，建议就使用默认值5.
这里解释一下为什么它的值不能太大：keepFileOpen的文件如果长时间没有写入仍旧处于打开状态还不如flush一下关闭掉好，其次太多保持打开的日志文件会占用太多操作系统资源。


`cleanupFileName` : 表示是否检验配置的文件路径是不是合法的，检查有没有操作系统不允许的路径字符。设置成true，写日志的时候会检查一下，路径非法就抛错。设置成false，不检查，对性能有少许提高，但是如果路径非法，不抛错提醒你，最终你啥日志也没有。默认值是true。 推荐开发者认真配路径，别配错，然后设置成false。

`bufferSize` : 写文件缓存，单位是字节，默认值是32768，32kb。可适当配大点，提高性能。

`autoFlush` : 表示是否写每一条日志后刷新下文件缓存，默认值是true.  推荐设置成false，这样对性能有可观的提高。

`openFileFlushTimeout` : `autoFlush`设置成false这个参数才有作用。它表示周期性的隔多久把操作系统缓存flush到文件。单位是秒。0和负数表示禁用。
推荐设置成 2-5。

`concurrentWrites` : 默认为false。设置成true，则允许多个进程同时写入同一个日志文件。一般不会有多进程写入同一份文件的需求，建议为false，因为开启此功能也会损伤性能。如果设置成false，则`concurrentWriteAttemptDelay`，`concurrentWriteAttempts`也没有了作用。建议有多进程写入同一份日志文件需求时，再仔细研究这3个参数的含义。
# 引入上下文变量



# 结构化日志
```csharp
logger.ForDebugEvent().Message(CultureInfo.InvariantCulture,"{0} + {1} = {2}", 3, 5, 3 + 5).Property("propertyName1",1).Property("propertyName2",2).Callsite(nameof(Program)).Log();
```


// LogEventInfo.Create（）l列出所有的成员属性。

# 消息模板

https://messagetemplates.org/

***NLog消息模板总结***

- 优先使用数字索引。数字索引比字符串索引性能高，因为字符串索引会生成带properties的LogEventInfo。

- 不要混用数字索引和字符串索引。混用时NLog解析消息模板会有额外的开销。

- 数字索引和`{$value}`无法优雅的处理复杂对象，列表，字典。`{value}`能够优雅的处理列表和字典，但仍旧不能优雅的处理复杂对象。`{@value}`可以优雅的处理复杂对象，列表，字典。唯一的缺点是可能有些属性我们不期望被序列化或者对象无法被序列化如Task。

- 数字索引和`{$value}`都是调用变量的Object-ToString()，复杂对象重写Object-ToString()或传入IFormatProvider格式化器，列表和字典等集合需要自己拼接成有意义的字符串再打印，另外，如果列表和字典等集合中的元素是复杂对象，则元素也需要重写Object-ToString()或传入IFormatProvider格式化器。

- `{value}`也是调用调用变量的Object-ToString()，但列表和字典例外，它能自动展开列表和字典打印其元素，而不是打印集合默认的ToString()，如`"System.Collections.Generic.List1[System.Int32]"`，`"System.Collections.Generic.Dictionary`2[System.String,System.Int32]"`。这是

  `{value}`相对于`{$value}`和数字索引的优势。但是如果集合中的元素是复杂对象仍旧需要重写Object-ToString()或传入IFormatProvider格式化器。

- `{@value}`打印变量的JSON字符串。唯一的缺点是有些对象的属性是不可序列化的如Task，以及有些属性是不想让其参与序列化的。



## 数字索引

NLog支持String.Format(......)

```csharp
logger.Debug("{0} + {1} = {2}", 1, 2, "3");
logger.Info("{0}",new User { Id = 100,Name = "Gaoyuanyuan"});
logger.Info("{0}", new List<int> { 0,1,2});
logger.Info("{0} create {1}", new User(1001, "Jack"), new Order(901021, DateTime.Now, new List<string>() { "shoes", "computer", "book" }));
logger.Fatal((A:1, B:2));
```

```xml
layout="${longdate} ${uppercase:${level:padding=-5}} ${logger} ${message}"
```

```tex
2022-08-11 00:15:09.9089 DEBUG Program 1 + 2 = 3
2022-11-15 02:04:57.5808 INFO  Program User
2022-11-15 02:04:57.6874 INFO  Program System.Collections.Generic.List`1[System.Int32]
2022-08-11 00:19:26.3646 INFO  Program User create Order
2022-08-11 00:19:26.3646 INFO  Program (1, 2)
```

列表，字典等集合采用Object-ToString( )，打印出来的字符串并没有什么实际意义，需要开发者遍历元素进行拼接得到有意义的字符串再进行打印。

复杂对象采用的也是Object-ToString( )，默认的ToString( )可能不是我们期望的。

可以重写复杂对象的Object-ToString()解决此问题。

```csharp
public class User
{
    public User(int userId, string name)
    {
        UserId = userId;
        Name = name;
    }

    public int UserId { get; set; }
    public string Name { get; set; }

    public override string ToString()
    {
        return $"UserId:{UserId},Name:{Name}";
    }
}

public class Order
{
    public Order(int id, DateTime time, IList<string> goods)
    {
        Id = id;
        Time = time;
        Goods = goods;
    }

    public int Id { get; set; }
    public DateTime Time { get; set; }
    public IList<string> Goods { get; set; }

    public override string ToString()
    {
        return $"Id:{Id},Time:{Time},Goods:{string.Join('|', Goods)}";
    }
}
```

```tex
2022-08-11 00:24:42.2618 INFO  Program UserId:1001,Name:Jack create Id:901021,Time:2022/8/11 0:24:42,Goods:shoes|computer|book
```

如果复杂对象没有重写ToString()或重写的不是我们期望的，但我们又没有权限编辑复杂对象，这种情况，我们可以传递一个继承IFormatProvider的格式化器。

```csharp
logger.Info(new CustomFormatProvider(),"{0} create {1}", new User(1001, "Jack"), new Order(901021, DateTime.Now, new List<string>() { "shoes", "computer", "book" }));
```
`{0}`，`{1}`，`{2}` 这种数字索引占位符并不是必须要求从0开始，数字索引被数字表示的位置的变量所替换，如{2}表示使用第3个变量替换。

```csharp
logger.Warn("{2}", 1, 2, 3); // 3
```


## 字符串索引

==打印日志是高频操作，如果每次都传入一个IFormatProvider格式化器和拼接字典、列表，太麻烦了！==

NLog支持`{value}`,`{@value}`,`{$value}`.

-----

`{$value}`

打印变量的Object-ToString()，效果同数字索引。

---

`{value}` 

打印变量的Object-ToString()，但针对列表和字典，能够自动展开打印其元素的Object-ToString(). 

列表: item1,item2,item3

字典: key1=value1,key2=value2,key3=value3

```csharp
logger.Info("{value1}", 1);    // 1
logger.Info("{value1}", "1");  // "1"
logger.Info("{value1}", new List<int>() { 1, 2, 3, 4 }); // 1, 2, 3, 4
logger.Info("{value1}", new List<string>() { "1", "2", "3", "4" }); // "1", "2", "3", "4"
logger.Info("{value1}", new User { Id = 100, Name = "GaoYuanyuan" }); // 100GaoYuanyuan
logger.Info("{value1}", new Dictionary<string, int>() { { "one", 1 }, { "two", 2 } }); // "one"=1, "two"=2
```

虽然列表和字典可以自动展开集合中的元素进行打印，但是如果元素是复杂对象，打印的字符串仍旧无实际意义。

```csharp
logger.Info("{users}", new List<User> { new User { Id = 100, Name = "Gaoyuanyuan" }, new User { Id = 101, Name = "Yangmmi" } });
```

```tex
2022-11-15 02:32:06.9039 INFO Program User, User
```

所以如果想利用`{value}`打印有意义的日志，必须还要为复杂对象重写Object-ToString( )或者传入IFormatProvider格式化器。

---

`{@value}`

打印变量的JSON格式，所以能够轻松展现复杂对象，字典，列表的详细信息。

```csharp
logger.Info("{@user} buy {@dic} that cost {@money}￥.", new User(10102, "Bruce"), new Dictionary<int, string>() { { 1, "one" }, { 2, "two" } }, 586);
```

```tex
2022-08-11 01:10:25.9994 INFO  Program {"UserId":10102, "Name":"Bruce"} buy {"1":"one","2":"two"} that cost 586￥.
```

```csharp
logger.Info("{user} buy {dic} that cost {money}￥.", new User(10102, "Bruce"), new Dictionary<int, string>() { { 1, "one" }, { 2, "two" } }, 586);
```

```tex
2022-08-11 01:10:26.1180 INFO  Program UserId:10102,Name:Bruce buy 1="one", 2="two" that cost 586￥.
```

```csharp
logger.Info("Test {@value1}", new { OrderId = 2, Status = "Processing" });
```

```tex
2022-08-11 01:18:01.1571 INFO  Program Test {"OrderId":2, "Status":"Processing"}
```

仍旧存在一些问题，假设`{value}`，`{$value}`，`{@value}`不是我们想要的格式怎么办？比如，`{value}`是字符串类型，输出带双引号，我们不想要双引号；`{$value}`的ToString()是默认的Object-ToString，我们也不想要，但又不想传入IFormatProvider格式化器；`{@value}`Json序列化有我们不想要却有的属性或我们想要但是没有的字段或value是个很重很危险的对象(Thread,Task,CultureInfo等)序列化时可能抛出异常。如果是少量操作，我们可以在打印日志的时候控制变量的传入。

```csharp
logger.Error("{@user} login at {$time}.", new {Name=user.Name}, DateTime.Now.ToString("yyMMddhhmmss"));
```

但是每次打印日志都需要这样，太麻烦了！NLog提供`NLog.IValueFormatter`接口，可以一劳永逸的定制`{value}`，`{$value}`，`{@value}`。

```csharp
public class OverrideValueFormatter : IValueFormatter
{
    private readonly IValueFormatter _originalFormatter;

    public bool StringWithQuotes { get; set; }

    public OverrideValueFormatter(IValueFormatter originalFormatter)
    {
        _originalFormatter = originalFormatter;
    }

    public bool FormatValue(object value, string format, CaptureType captureType, IFormatProvider formatProvider, System.Text.StringBuilder builder)
    {
        // captureType Unknown:Not decided Normal:{x} Serialize:{@x} Stringify:{$x}

        if (!StringWithQuotes && captureType == CaptureType.Normal)
        {
            switch (Convert.GetTypeCode(value))
            {
                case TypeCode.String:
                    {
                        builder.Append((string)value);
                        return true;
                    }
                case TypeCode.Char:
                    {
                        builder.Append((char)value);
                        return true;
                    }
                case TypeCode.Empty:
                    return true;  // null becomes empty string
            }
        }

        if (captureType == CaptureType.Serialize)
        {
            if (value is Task)
            {
                Task t = (Task)value;
                builder.Append(t.Id).Append(t.Status);
                return true;
            }
        }

        return _originalFormatter.FormatValue(value, format, captureType, formatProvider, builder);
    }
}

var oldFormatter = NLog.Config.ConfigurationItemFactory.Default.ValueFormatter;
var newFormatter = new OverrideValueFormatter(oldFormatter);
newFormatter.StringWithQuotes = false;
NLog.Config.ConfigurationItemFactory.Default.ValueFormatter = newFormatter;
```

NLog的`{value}`格式已经实现的足够好了，一般满足我们的需求，不需要定制；只要我们遵守面向对象的开发规范，为自定义类型都重写ToString()，那么`{$value}`也都满足我们的需求，不需要定制。`{@value}` 序列化时经常出现我们不想要的属性，以及Microsoft内置的危险对象在序列化时抛出异常，经常需要定制。我们用上述的IValueFormatter接口能实现定制`{@value}`，但是这个接口也掺杂定制`{value}`和`{$value}`，比较臃肿，所以NLog也提供了另外一个专门定制JSON序列化的方法，在序列化System.Net.WebException时，会将它转换成匿名类，然后对匿名类进行序列化。

```csharp
LogManager.Setup().SetupSerialization(s =>
    s.RegisterObjectTransformation<System.Net.WebException>(ex => new {
        Type = ex.GetType().ToString(),
        Message = ex.Message,
        StackTrace = ex.StackTrace,
        Source = ex.Source,
        InnerException = ex.InnerException,
        Status = ex.Status,
        Response = ex.Response.ToString(),  // Call your custom method to render stream as string
    })
);
```

NLog采用的是Microsoft.System.Text.Json序列化，我们可以改成Newtonsoft.Json，并对Newtonsoft.Json进行一些全局设置，如值为NULL的对象不参与序列化，为特殊类型注册序列化设置等。

```csharp
    internal class JsonNetSerializer : NLog.IJsonConverter
    {
        readonly JsonSerializerSettings _settings;

        public JsonNetSerializer()
        {
           _settings = new JsonSerializerSettings
                {
                    Formatting = Formatting.Indented,
                    ReferenceLoopHandling = ReferenceLoopHandling.Ignore
                };
        }

        /// <summary>Serialization of an object into JSON format.</summary>
        /// <param name="value">The object to serialize to JSON.</param>
        /// <param name="builder">Output destination.</param>
        /// <returns>Serialize succeeded (true/false)</returns>
        public bool SerializeObject(object value, StringBuilder builder)
        {
            try
            {
                var jsonSerializer = JsonSerializer.CreateDefault(_settings);
                var sw = new System.IO.StringWriter(builder, System.Globalization.CultureInfo.InvariantCulture);
                using (var jsonWriter = new JsonTextWriter(sw))
                {
                    jsonWriter.Formatting = jsonSerializer.Formatting;
                    jsonSerializer.Serialize(jsonWriter, value, null);
                }
            }
            catch (Exception e)
            {
                NLog.Common.InternalLogger.Error(e, "Error when custom JSON serialization");
                return false;
            }
            return true;
        }
    }

NLog.Config.ConfigurationItemFactory.Default.JsonConverter = new JsonNetSerializer();
```

额外补充一点：

以`{value}`示例，默认效果如下：

```csharp
logger.Error("wrong reason :{reason}", "invalid input");
```

```tex
2022-08-11 02:17:26.5586 ERROR Program wrong reason :"invalid input"
```

reason是string类型，默认打印带"",不想要怎么办？可以加`{value:l}`

```csharp
logger.Error("wrong reason :{reason:l}", "invalid input");
```

```tex
2022-08-11 02:19:42.2508 ERROR Program wrong reason :invalid input
```

最后，我们看下message的形式

```csharp
logger.Info("{0} + {1} = {2}",0, 1, 1);
```

```csharp
logger.Info("{v1} + {v2} = {v3}",0, 1, 1);
```

```csharp
logger.Info("{0} + {v2} = {1}",0, 1, 1);
```

**理解以上知识，我们就能通过配置文件中的${message}打印出优秀的日志文本信息**

# 可解析的日志

如果我们想打印出结构化日志，先用文本形式存储，后续可以读取文本解析关键字段对业务进行大数据分析，或者直接把日志存储到数据库或者通过网络发送到远端供别的开发者收到后解析，我们可能需要打印完全是JSON格式的日志。

通过指定Target的Layout就可以达到目的。

```xml
<target xsi:type="Console" name="console">
	<layout xsi:type="JsonLayout" includeAllProperties="false">
		<attribute name="time" layout="${longdate}" />
		<attribute name="level" layout="${level:upperCase=true}" />
		<attribute name="message" layout="${message}" />
		<attribute name="sequence" layout="${sequenceid}" />
	</layout>
</target>
```

```csharp
logger.Warn("{0} login website.", new User(10102, "Bruce"));
```

```tex
{ "time": "2022-08-11 06:16:57.1843", "level": "WARN", "message": "User login website.", "sequence": "1" }
```

序列化得到的JSON字符串的格式有以下特点：

* Json的root是对象

* 对象的key是attribute的name,对象的value是layout绑定的上下文变量的ToString()，不管上下文变量是什么类型，恒为字符串

虽然是完整的标准的JSON字符串，但是value全是字符串，这并不太是我们期望的，更希望保持原有的类型。

LogEventInfo的properties属性可以在序列化时保持原有类型。

如果是一次性序列化所有的properties，NLog会保持原类型。如果序列化单个property，那即是序列化一个上下文变量，仍旧是不管它是什么类型都是字符串，幸好，NLog提供format参数为@便会保持原类型。

(1) 自动序列化所有的properties

```xml
<layout xsi:type="JsonLayout" includeAllProperties="true"> <!--includeAllProperties必须是true，才会序列化properties-->
	<attribute name="time" layout="${longdate}" />
	<attribute name="level" layout="${level:upperCase=true}" />
	<attribute name="message" layout="${message}" />
	<attribute name="sequence" layout="${sequenceid}" />
	<attribute name="sequence" layout="${all-event-properties}" /> <!--在这里设置-->
</layout>

```

```csharp
logger.ForWarnEvent().Property("liuwei", new User(10102, "Bruce")).Property("numbers", new List<int>() { 1, 2, 3 }).Log();
```

```csharp
{ "time": "2022-08-11 06:31:52.3707", "level": "WARN", "sequence": "3", "sequence": "liuwei=User, numbers=1, 2, 3", "liuwei": {"UserId":10102, "Name":"Bruce"}, "numbers": [1,2,3] }
```

（2）序列化properties的其中一个元素

```xml
<attribute name="liuwei" layout="${event-properties:item=liuwei:format=@}" />
```

（3）序列化properties的其中一个元素的成员属性

```xml
<attribute name="liuwei" layout="${event-properties:item=liuwei:format=@:objectpath=Name}" />
```

所以，打印JSON格式的结构化日志，为LogEventInfo的Properties添加元素是必须的。但是NLog的日志API：Logger.Debug(),Logger.Info(),Logger.Error()等并为提供传递Properties的参数，需要以其他形式的API.

(1) logger.WithProperty()

```csharp
logger.WithProperty("myProperty", "myValue").Info("Hello");
```

(2) FluentAPI

```csharp
logger.ForInfoEvent().Property("myProperty", "myValue")).Message("hello").Log();
```

(3)  模板参数

```csharp
logger.Info("Say hello to {myProperty}", "myValue");
```

字符串形式的模板参数，自动填入properties，key名称就是模板参数名称。如果既想看普通文本形式的日志，又想以JSON格式把日志输出到数据库，用模板参数更合适。否则，推荐方式(1)。

# 数字索引和字符串索引的区别

1. 不建议混用数字索引和字符串索引。

   要么只用数字索引，要么只使用字符串索引；混用的话，NLog解析消息模板会有额外的开销。

2. 能用数字索引就不要用字符串索引

   字符串索引会引入上下文变量到event-properties，每个LogEventInfo都会携带一个字典类型的properties属性，会有额外的开销。


# 印CSV和Json格式的日志文件
## JsonLayout

## CsvLayout

[CsvLayout · NLog/NLog Wiki (github.com)](https://github.com/NLog/NLog/wiki/CsvLayout)

```csharp
// 记录到日志，NLog自动以CSV格式存储
_logger.Info("{category} {detail}", category, detail);
```

```xaml
<target xsi:type="File" name="csvFileExample" fileName="./${shortdate}.csv" encoding="gbk">
	<layout xsi:type="CsvLayout" delimiter="Comma" withHeader="true">
		<column name="时间" layout="${date:format=yyyy年 MM月 dd日 HH时 mm分 ss秒}" />
		<column name="类别" layout="${event-properties:category}"/>
		<column name="详情" layout="${event-properties:detail}"/>
		<column name="用户" layout="默认"/>
	</layout>
</target>
```

使用NLog自带的CSV布局功能，只是layout比平时复杂一点而已。注意通过结构化日志传入的上下文变量都在LogEventInfo的Properties中，layout的格式是`${event-properties:category}`而不是`${category}`。

# Callsite
layout配置Callsite,可以打印输出日志的方法，类，命名空间，源文件名称，代码行数，这十分有利于定位出问题的代码位置。
Callsite是从dll对应的pdb文件中拿到的，所以你的程序不仅要放dll，还要放pdb文件，没有pdb文件，无法打印源文件行号和源文件路径。
`layout="${longdate} ${level} ${callsite:includeNamespace=true:className=true:methodName=true:fileName=true:includeSourcePath=true} ${message}"`
源代码文件路径太长了，打印了太浪费存储空间了，一般知道源代码文件名就行了，搜一下就找到了，可以不放源代码的路径
`layout="${longdate} ${level} ${callsite:includeNamespace=false:className=true:methodName=true:fileName=true:includeSourcePath=false} ${message}"`
另外，release版本可能会内联小方法，导致打印的行数，调用者与源代码不一致，注意一下就行了，无伤大雅。
Callsite底层实现：
LogEventInfo有CallerClassName,CallerMemberName,CallerFilePath,CallerLineNumber.在打印LogEventInfo时，可以提取这4个成员属性。它们从CallSiteInfo类型的属性获取。
<img src="https://img2022.cnblogs.com/blog/1037641/202208/1037641-20220809195224258-2065582316.png" alt="image" style="zoom:40%;" />
CallSiteInfo也有自己的CallerClassName,CallerMemberName,CallerFilePath,CallerLineNumber，如果这4个属性已经有值(非String.Empty非NULL),便会直接返回，如果4个属性无值（string.Empty或NULL）,就会捕获StackTrace，从StackTrace提取这4个属性。如果每条LogEventInfo都捕获StackTrace,日志性能会迅速大幅度下降。所以我们应该尝试提前把LogEventInfo的属性CallSiteInfo的CallerClassName,CallerMemberName,CallerFilePath,CallerLineNumber赋好值，避免捕获StackTrace。NLog提供了这个API:`public void LogEventInfo.SetCallerInfo(string callerClassName, string callerMemberName, string callerFilePath, int callerLineNumber)`.
CallSiteInfo属性获取方式的源码如下：
<img src="https://img2022.cnblogs.com/blog/1037641/202208/1037641-20220809195630623-66415773.png" alt="image" style="zoom:60%;" />
CallerClassName
<img src="https://img2022.cnblogs.com/blog/1037641/202208/1037641-20220809195755903-705185750.png" alt="image" style="zoom:50%;" />
CallerMemberName
<img src="https://img2022.cnblogs.com/blog/1037641/202208/1037641-20220809195903430-827989893.png" alt="image" style="zoom:50%;" />
CallerFilePath
<img src="D:\OneDrive\Run\images\1037641-20220809195956413-2043505302.png" alt="image" style="zoom:50%;" />
CallerLineNumber
重点：
但是NLog需要查线程的StackTrace，还要利用反射，会使日志性能降低5-7倍。所以建议，Info级别的日志不要打印Callsite，Error,Warn，Fatal，Debug级别的日志可以打印。
如果Info级别也想打印Callsite，推荐继承Logger通过CallerMemeberAttribute和CallerFileAttribute,CallerLineNumberAttribute实现自定义Logger.
不过NLog通过FluentAPI实现了机制。

# 生产环境禁用Console

日志输出到控制台会极大的降低日志性能，强烈建议调试器可以输出到控制台，生产环境禁止输出到控制台。

# 重视Logger Name
```
Logger logger = LogManager.GetCurrentClassLogger();
```
logger的Name是调用上述代码的方法所在的类的类名，Namespace.Class.
```
LogManager.GetLogger("app_module_component");
```
这种方式可以自定义logger的Name.
<br>
Logger的Name可以在NLog.config作为环境变量使用，它有三大重要用处：
1. 在Rule中匹配Logger和Target，Name影响使用模糊匹配的难易程度
2. 决定日志文件名
3. 可用于为每条日志的打分类标签

所以Name不可随意起名，应当按照程序的层级关系或功能模块来命名，既方便分类归纳和查看日志，也利于在Rules中利用模糊匹配将一个模块下的不同组件的日志聚合到一个日志文件中来查看组件间的动作时序。良好的命名示例如下：
* myapp_config
* myapp_login
* myapp_networkMsg_mqtt
* myapp_networkMsg_http
* mMapp_motion_plc_axis
* myapp_motion_robot  

具有良好规范的namespace能够反映solution的功能模块划分和层次关系，如果团队成员都为自己负责的project提供合适的namespace，那么我们可以直接使用`LogManager.GetCurrentClassLogger()`获取Logger，由此API也可以看出NLog作者鼓励开发者这样做。但是如果我们的solution由于各种原因不具备规范的namespace或者我们不想使用namespace，那么我们可以使用`LogManager.GetLogger(string name)`.  

如果我们采取自定义每个Logger的Name这种方案，那么在开发调试期间，难免因为日志组织的不够好，需要频繁调整Logger的Name，如果去每个源文件中改名，那是多么的麻烦且容易疏漏。所以强烈建议，将整个solution使用的Logger的Name组织到一个文件的类中，集中管理Name。当需要获取Logger时，从类中获取Name。这样我们就能极其轻松的修改Name(一改俱改)，甚至还可以清晰的查看整个solution的日志结构。
```
public static class LoggerNameCollection
{
    public const string app_plc_axis = nameof(app_plc_axis);
    public const string app_plc_robot = nameof(app_plc_robot);
    public const string app_plc_robot_msg = nameof(app_plc_robot_msg);
    public const string app_mqtt_raw = nameof(app_mqtt_raw);
    public const string app_mqtt_resolved = nameof(app_mqtt_resolved);
    public const string app_vision = nameof(app_vision);
}

Logger logger = LogManager.GetLogger(LoggerNameCollection.app_plc_axis); // 从LoggerNameCollection获取Name
```
当然在Visual Studio中namespace天然的一改俱改，使用`LogManager.GetCurrentClassLogger()`方案，可以实时调整namespace来达到重新组织日志的目的。

# 获取Logger实例存在时间开销
首先明确一点，允许同时存在多个相同Name的Logger实例，它们打印日志的效果相同。只是既然Name相同的两个Logger实例打印日志的效果相同，NLog就索性以Name唯一标识一个Logger实例，获取指定Name的Logger实例时，不存在则创建并缓存，存在则直接从缓存返回。
`LogManager.GetCurrentClassLogger()`和`LogManager.GetLogger(string name)`的执行步骤：
进入一个被static的锁锁住的缓存(Dictionary<string,Weakreference<Logger>>),如果缓存中已存在与参数name相同Name的Logger且没被垃圾回收，则直接返回此Logger，否则实例化一个Logger再返回。
上述步骤存在两点开销：
* 每一次调静态方法`LogManager.GetCurrentClassLogger()`和`LogManager.GetLogger(string name)`都会进入lock(static object),入锁出锁等待锁存在时间开销。
* 缓存的Logger实例并不是永久存储于缓存中，它是弱引用，可能被垃圾回收，多次从缓存中获取同一个Name的实例仍旧存在多次实例化的时间开销的可能。
所以建议的获取Logger实例的方式是：
1. (务必遵守)类中创建一个Logger类型的字段，该字段在整个类的成员中使用。这样与在成员方法中打日志临时获取logger的方式比，减少了`LogManager.GetCurrentClassLogger()`和`LogManager.GetLogger(string name)`的执行次数，避免了整个日志系统卡在static锁上。
2. 如果某个类在短时间内被高频率实例化，建议保证它的Logger是静态的，进一步减少`LogManager.GetCurrentClassLogger()`和`LogManager.GetLogger(string name)`的执行次数
3. 如果将所有的Logger都声明成静态的，会浪费存储空间，2中的短时间高频率又没有明确的评判标准。所以建议先全部声明成实例字段，后期寻找10个最频繁被实例化的类改成static.


# 新增环境变量
两种方式：自定义LayoutRender或利用LogEventInfo的properties属性
前者：定义注册后，在配置文件中随便使用，甚至可以把定义的类发成一个库，在不同的项目中复用。这个环境变量可以有属性，不同的属性对应不同的格式化方法，灵活多变。如果环境变量的获取比较复杂，或者在很多项目中都可能频繁使用比较常见和刚需，建议使用这种方式添加环境变量
后者：这种方式不用定义类，可以在LogEventInfo中添加，也可以用logger.WithProperty为所有由它写入的的LogEventInfo全部具有property,这种方式的缺点是代码比较零散，而且只能用debug（logeventinfo log）不能用Debug(string log),优点是适合针对性的一次性代码，比如客户端发过来的日志具有callsite和machinename信息，服务器接收后打印日志肯定不能用服务器的machinename和callsite，需要解析消息中的callsite和mahcinename,这时候服务器只需要提起消息中的machinename赋值给logeventinfo即可，配置文件中和用eventproperties : Item="perproty即可"，而不需要自己定义很多Render提取Logevent中的proeprty.


［强制］、［推荐］、［参考］

If a dog is a man’s best friend, logs are software engineer’s best friend.

# 日志最佳实践规范
## 必须使用条件输出形式或者使用占位符的方式。
正例,条件输出形式
```csharp
if (logger.IsDebugEnabled)
{
    string msg = "user id " + id + " login by ip " + ip;
    logger.Debug(msg);
}
```
正例,占位符
```csharp
logger.Debug("user id {0} login by ip {1}", id, ip);
```
反例
```csharp
string msg = "user id " + id + " login by ip " + ip;
logger.Debug(msg);
```
正例保证logger确认此日志级别会被打印才进行字符串拼接；反例不管等级是什么都会进行字符串拼接，可能导致不必要的字符串拼接操作。
## 故障日志必须包含callsiteinfo
Warn,Error,Fatal级别的日志必须包含异常信息，异常类型，异常堆栈，调用者，行号，文件名，这样有利于快速在源代码中定位故障位置。
## 异常实例直接做参数
避免将异常当格式化参数传入，而是将异常当第一个参数显示传入，通过配置文件可以选择打印日志的哪些成员信息。
推荐
try
{
}
catch (Exception ex)
{
    logger.Error(ex, "Something bad happened");
}

避免
try
{
}
catch (Exception ex)
{
    logger.Error("Something bad happened， error message：{0}"，ex); 
}
* 【强制】既要支持将日志分流到不同的类别，也要支持将任意多个类别的日志汇总到一个日志文件。
不同层级，不同模块的日志分流到相对应的类别文件中，有利于归纳整理日志系统，当某个层级，某个模块出现问题，有利于在海量繁杂的日志中精确的找到相关的感兴趣的日志。
不同类别的日志汇总到一处，虽然会让日志存储量翻倍，但有利于观察应用的多个模块的交互时序，便于定位问题。

* 【强制】每个类别的故障日志（Warn,Error,Fatal）汇总输出到一个文件。
针对海量日志，更容易觉察系统已出现故障，分析系统错误。

* 【强制】对日志文件命名能够看出什么应用，什么模块，什么层级，什么目的。
正例
monitor.ip
network.mqtt.rawmsg
login

* 【推荐】开发过程中不断对日志内容进行优化更新，在保证很好的反映系统运行的轨迹，能够快速准确地定位问题的前提下，打印最少量的日志。

* 【强制】谨慎地记录日志和设置日志等级。生产环境禁止输出debug日志；有选择地输出info日志。
大量地输出无效日志，增大磁盘和网络的IO压力，阻塞正常的业务方法，也不利于快速定位错误点，占用大量存储空间，记录日志时请思考：这些日志真的有人看吗？看到这条日志你能做什么？能不能给问题排查带来好处？
非必要情况下，避免在大循环结构中打印日志。


* 【推荐】日志文件大小要合适
日志文件太大，不便查找某个具体时间点的日志，不便从生产环境拷贝到个人笔电。
日志文件太小，某段连续的逻辑日志分散到多个文件，不便搜查关键字，文件也太过零散。
建议100M，最大250M，随着项目的进展不断调整到最适合的大小。


* 【推荐】日志文件不能挤爆磁盘导致系统宕机
日志文件推荐至少保存15天，因为有些异常具备以“周”为频次发生的特点。
自动删除过期日志：（1）日志组件内置删除N久之前的日志文件机制.（2）定时器(如Quartz.NET)定期删除过期日志。