**项目下的资源文件全部拷贝到运行目录中，连同资源文件的目录结构。**
方法1：将资源文件的生成操作设置成 内容 ，复制输出到目录设置成 始终复制 或 如果较新则复制。

```xml
<ItemGroup>
  <Content Include="Assets\IPcfg.ini">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </Content>
  <Content Include="Assets\Snipaste_2022-12-30_19-43-08.png">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </Content>
</ItemGroup>
```

这样过于麻烦，可以这样写

```xml
<ItemGroup>
  <Content Include="Assets\*">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </Content>
</ItemGroup>
```