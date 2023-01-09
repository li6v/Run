# SVG
## 一般用法
1. 将SVG复制到文件夹下，并设置SVG的文件属性 资源 + 不复制到输出目录 或 内容 + 复制到输出目录
![image](D:\OneDrive\Run\images\1037641-20220910222405086-499320736.png)
2. 添加Nuget包：sharpvectors
![image](D:\OneDrive\Run\images\1037641-20220910222732114-2066475208.png)
3. 添加命名空间 
`xmlns:svgc="http://sharpvectors.codeplex.com/svgc/"`
4. 使用
```xml
<Grid>
    <svgc:SvgViewbox IsHitTestVisible="False" Source="/SvgIcons/下载.svg"  Height="50" Width="50" />
</Grid>
```
说明：
（1）sharpvectors中的SvgViewbox继承自ViewBox，是一个元素，可放到内容控件和布局组件中，且能任意设置宽度和高度决定SVG显示的尺寸。
（2）上面示例的SvgViewbox的Source的URI路径是相对路径

## 集中管理SVG
一般用法情况下，SVG会散落在各个DLL中，不便于管理整个应用程序的SVG。可以创建一个DLL存放所有的SVG.

1. 新建WPF类库(假设类库名称叫Asserts)，用于存放整个应用程序使用的SVG图片，便于管理。
<img src="D:\OneDrive\Run\images\1037641-20220910170008040-892132555.png" alt="image" style="zoom:80%;" />
2. 将SVG图片复制到WPF类库中的文件夹下面。（SVG比较多的时候，建议使用文件夹分门别类便于浏览和管理）
![image](D:\OneDrive\Run\images\1037641-20220910170206335-1478611756.png)
3. 设置SVG的文件属性，生成操作为资源，不复制到输出目录
![image](D:\OneDrive\Run\images\1037641-20220910170303292-1869881347.png)
4. 需要使用SVG的程序集引用Asserts，并下载Nuget：sharpvectors
<img src="D:\OneDrive\Run\images\1037641-20220910170718054-986762928.png" alt="image" style="zoom:80%;" />
5. 引入命名空间
`xmlns:svgc="http://sharpvectors.codeplex.com/svgc/"`
6. 使用
```xml
<StackPanel>
    <svgc:SvgViewbox IsHitTestVisible="False" Source="pack://application:,,,/Assets;v1.0.0.0;component/SvgIcons/系统配置.svg" Height="50" Width="50"/>
</StackPanel>
```
<img src="D:\OneDrive\Run\images\1037641-20220910170932693-152792660.png" alt="image" style="zoom: 80%;" />

说明：
（1）SvgViewbox的Source的URI路径是绝对路径，引用的是其他程序集资源，注意不要写错！
（2）存放图片的类库必须是WPF类库，普通的类库缺少WPF相关的DLL，无法加载SVG.
## 多语言情况下使用SVG
切换不同的语言时，程序所使用的SVG可能也需要更换。
1. 在SVG DLL中封装SVG为属性
```csahrp
public class ChinaSvgLocator
{
    public string Download => "pack://application:,,,/MyAssets;v1.0.0.0;component/svgicons/下载.svg";
}
```
2. 在Application.Resource中实例化ChinaSvgLocator
```xml
<Application.Resources>
    <local:ChinaSvgLocator x:Key="svgLocator" />
</Application.Resources>
```
3. 使用数据绑定指定SvgViewBox的Source
```xml
<svgc:SvgViewbox IsHitTestVisible="False" Source="{Binding Source={StaticResource svgLocator}, Path=Download}"  Height="50" Width="50" />
```
可以添加USSvgLocator,JapanSvgLocator等，切换语言时，替换x:Key="svgLocator"对应的实例即可。


# IconFont
在https://www.iconfont.cn/ 搜索下载字体图标。
1. 添加到购物车
<img src="D:\OneDrive\Run\images\1037641-20220910231436232-2094093683.png" alt="image" style="zoom:50%;" />
2. 下载
<img src="D:\OneDrive\Run\images\1037641-20220910231455390-1527141002.png" alt="image" style="zoom:50%;" />
3. 解压下载得到的压缩包
![image](D:\OneDrive\Run\images\1037641-20220910231525242-904674498.png)
4. 新建文件夹，然后将ttf文件复制到此文件夹下面。
【必须先创建一个文件夹，然后复制ttf到文件夹下面，否则在设计模式下无法实时预览字体图标，只能启动后才能看到图标，这应该是WPF的一个bug】
![image](D:\OneDrive\Run\images\1037641-20220910231554109-2081122094.png)
在Application.Resource中添加FontFamily资源。
```xml
<Application.Resources>
    <FontFamily x:Key="iconfont">
        Fonts/#iconfont
    </FontFamily>
</Application.Resources>
```
5. Text指定字体图标在ttf文件中的Unicode代码。Unicode代码可以在下载的html网页中看到。
```xml
<TextBlock FontFamily="{StaticResource iconfont}" Text="&#xe65e;"  Foreground="Orange" FontSize="50" />
```
可以通过Foreground和FontSize指定字体图标的颜色和大小。