### IsAsync

View是订阅者，ViewModel是发布者。

---

View的事件处理程序大致是：

```tex
UI.Dispatcher.Invoke
{
    （1）通过反射获取源属性的值 （getter）。
    （2）值转换器
    （3）控件.依赖属性 = 新值
}
```
可以看到getter和值转换器都不能有耗时操作，否则会卡UI。

---

ViewModel发布事件

```csharp
PropertyChanged?.Invoke(this,new PropertyChangedEventArgs(nameof(Property)));
```

ViewModel的PropertyChanged?.Invoke( )内部执行事件处理程序，执行完毕后才会返回。

View通过Binding自动完成事件注册。

---

发现问题：

如果源属性的 getter比较耗时，比如从web请求得到属性值，那么会阻塞UI线程，导致UI加载显示会比较慢，ViewModel调用PropertyChanged?.Invoke( )刷新UI时，界面卡顿。

可以使用Binding. IsAsync 设置为 true 可以避免阻塞UI。它的作用相当于：

```tex
UI.Dispatcher.Invoke
{
     (0) 控件.依赖属性 = 默认值、继承值
}


后台线程
（1）通过反射获取源属性的值 （getter）。

UI.Dispatcher.Invoke
{
    （2）值转换器
    （3）控件.依赖属性 = 新值
}
```

getter直接返回，UI线程为控件的依赖属性暂时是默认值或继承值。getter内的逻辑会放到后台线程继续执行，等到getter计算出属性值后，在UI线程完成值转换器，赋值到控件的依赖属性。

可以为Binding指定FallbackValue，让控件在getter未返回前显示指定的值。

另外，`IsAsync = True`时，getter肯定不应当有必须在UI线程执行的代码！

---

**代码案例**

View

```xaml
<Button x:Name="btn" Content="{Binding Property, Mode=OneWay,
    IsAsync=True, Converter={StaticResource converter},FallbackValue=行程卡}"
        Command="{Binding Command}">
</Button>
```

ViewModel

```csharp
internal class ViewModel : ObservableObject
{
    public ViewModel()
    {
        Command = new RelayCommand(() =>
        {
            OnPropertyChanged(nameof(Property));
        });
    }

    public string Property
    {
        get
        {
            Console.WriteLine(Thread.CurrentThread.ManagedThreadId);
            Thread.Sleep(3000);
            return DateTime.Now.ToString(CultureInfo.InvariantCulture);
        }
    }

    public RelayCommand Command { get; set; }
}

class Converter : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter, CultureInfo culture)
    {
        Console.WriteLine($"转换器：{Thread.CurrentThread.ManagedThreadId}");
        return value.ToString();
    }

    public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture)
    {
        throw new NotImplementedException();
    }
}
```

<img src="D:\OneDrive\Run\images\动画-1670955010832-1.gif" alt="动画" style="zoom: 50%;" />

如果不加`IsAsync=True`，那么界面需要加载3秒才能展示出来，每点击一次按钮，界面都会卡死3秒。

加了`IsAsync=True`后，界面立刻展示出来，按钮显示“行程卡”，3秒后，按钮显示当前时间。每点一次按钮，界面上的按钮立刻显示“行程卡”，3秒后，按钮显示当前时间，界面不会出现卡顿。



