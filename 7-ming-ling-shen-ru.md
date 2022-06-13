---
description: http://www.cnblogs.com/wzh2010/p/6607702.html
---

# 7 命令深入

在上面一篇中，我们大致了解了命令的基本使用方法和基础原理，但是，实际在运用命令的时候会复杂的多，并且会遇到各种各样的情况。

**一、命令带参数的情况**

如果视图控件所绑定的命令想要传输参数，需要配置 CommandParameter 属性，用来传输参数出去。而继承自 ICommand 接口的 RelayCommand 又具有支持泛型的能力，这样就可以接受来自客户端请求的参数。

```csharp
public RelayCommand(Action<T> execute);
```

构造函数传入的是委托类型的参数，Execute 和 CanExecute 执行委托方法。

所以，修改上篇的代码如下：

View代码：

```xml
<StackPanel Margin="10,20,0,50">
    <TextBlock Text="传递单个参数" FontWeight="Bold" FontSize="12" Margin="0,5,0,5" ></TextBlock>
    <DockPanel x:Name="ArgStr" >
        <StackPanel DockPanel.Dock="Left" Width="240" Orientation="Horizontal" >
            <TextBox x:Name="ArgStrFrom" Width="100" Margin="0,0,10,0"></TextBox>
            <Button Content="传递参数" Width="100" HorizontalAlignment="Left" Command="{Binding PassArgStrCommand}"
                    CommandParameter="{Binding ElementName=ArgStrFrom,Path=Text}" ></Button>
        </StackPanel>
        <StackPanel DockPanel.Dock="Right" Width="240" Orientation="Horizontal">
            <TextBlock Text="{Binding ArgStrTo,StringFormat='接收到参数：\{0\}'}" ></TextBlock>
        </StackPanel>
    </DockPanel>
</StackPanel>
```

ViewModel 代码：

```csharp
#region 传递单个参数
    private String argStrTo;
    /// <summary>
    /// 目标参数
    /// </summary>
    public String ArgStrTo
    {
        get { return argStrTo; }
        set
        {
            argStrTo = value;
            RaisePropertyChanged(() => ArgStrTo);
        }
    }
#endregion

#region 命令
    private RelayCommand<String> passArgStrCommand;
    /// <summary>
    /// 传递单个参数命令
    /// </summary>
    public RelayCommand<String> PassArgStrCommand
    {
        get
        {
            if (passArgStrCommand == null)
                passArgStrCommand = new RelayCommand<String>((p) => ExecutePassArgStr(p));
            return passArgStrCommand;
        }
        set { passArgStrCommand = value; }
    }
    private void ExecutePassArgStr(String arg)
    {
        ArgStrTo = arg;
    }
#endregion
```

结果如下：

![](https://images2015.cnblogs.com/blog/167509/201703/167509-20170326164001236-1033661552.png)

**二、多个参数的情况**

上面是单个参数传输的，如果需要传入多个参数，可能就需要以参数对象方式传入，如下所示：

Model 代码：

```csharp
public class UserParam 
{
    public String UserName { get; set; }

    public String UserPhone { get; set; }

    public String UserAdd { get; set; }

    public String UserSex { get; set; }
}
```

View代码：

```xml
xmlns:model="clr-namespace:MVVMLightDemo.Model"
```

```xml
<StackPanel Margin="10,0,0,50">
    <TextBlock Text="传递对象参数" FontWeight="Bold" FontSize="12" Margin="0,5,0,5" ></TextBlock>
    <DockPanel>
        <StackPanel DockPanel.Dock="Left" Width="240">
            <Button Command="{Binding PassArgObjCmd}" Content="传递多个参数" Height="23" HorizontalAlignment="Left" Width="100">
                <Button.CommandParameter>
                    <model:UserParam UserName="Brand" UserPhone="88888888" UserAdd="地址" UserSex="男" ></model:UserParam>
                </Button.CommandParameter>
            </Button>
        </StackPanel>
        <StackPanel DockPanel.Dock="Right" Width="240" Orientation="Vertical">
            <TextBlock Text="{Binding ObjParam.UserName,StringFormat='姓名：\{0\}'}" ></TextBlock>
            <TextBlock Text="{Binding ObjParam.UserPhone,StringFormat='电话：\{0\}'}" ></TextBlock>
            <TextBlock Text="{Binding ObjParam.UserAdd,StringFormat='地址：\{0\}'}" ></TextBlock>
            <TextBlock Text="{Binding ObjParam.UserSex,StringFormat='性别：\{0\}'}" ></TextBlock>
        </StackPanel>
    </DockPanel>
</StackPanel>
```

ViewModel 代码：

```csharp
#region 传递参数对象
    private UserParam objParam;
    public UserParam ObjParam
    {
        get { return objParam; }
        set
        {
            objParam = value;
            RaisePropertyChanged(() => ObjParam);
        }
    }
#endregion

#region 命令
    private RelayCommand<UserParam> passArgObjCmd;
    public RelayCommand<UserParam> PassArgObjCmd
    {
        get
        {
            if (passArgObjCmd == null)
                passArgObjCmd = new RelayCommand<UserParam>((p) => ExecutePassArgObj(p));
            return passArgObjCmd;  
        }
        set { passArgObjCmd = value; }
    }
    private void ExecutePassArgObj(UserParam up)
    {
        ObjParam = up;
    }
#endregion
```

结果如下：

![](https://images2015.cnblogs.com/blog/167509/201703/167509-20170327110155764-1184657303.png)

**三、动态绑定多个参数情况**

现在，参数可以过来了，但是，我们会发现这样的参数是我们硬编码在代码中的，比较死；一般情况下，需要动态绑定参数进行传递，所以，我们修改上面的代码如下：

```xml
<StackPanel DockPanel.Dock="Left" Width="240">
    <Button Command="{Binding PassArgObjCmd}" Content="传递多个参数" Height="23" HorizontalAlignment="Left" Width="100">
        <Button.CommandParameter>
            <model:UserParam UserName="{Binding ElementName=ArgStrFrom,Path=Text}" UserPhone="88888888" UserAdd="地址" UserSex="男" ></model:UserParam>
        </Button.CommandParameter>
    </Button>
</StackPanel>
```

这时候编译运行，它会提示：不能在“UserParam”类型的“UserName”属性上设置“Binding”。只能在 DependencyObject 的 DependencyProperty 上设置“Binding”。

原来，我们的绑定属性只能用在 DependencyObject 类型的控件对象上。像我们的 TextBox、Button、StackPanel 等等控件都是

&#x20;System.Windows.FrameworkElement => System.Windows.UIElement=>  System.Windows.Media.Visual => System.Windows.DependencyObject  这样的一种继承方式。所以支持绑定特性。

![](https://images2015.cnblogs.com/blog/167509/201704/167509-20170425112707084-477614291.png)

WPF 的所有 UI 控件都是 DependencyObject 类型的对象。

一种方式就是将 UserParam 类改成继承自 DependencyObject 的类，如下所示：

```csharp
/// <summary>
/// 自定义类型
/// </summary>
public class UserParam : FrameworkElement // 继承于FrameworkElement
{
    /// <summary>
    /// .net属性封装
    /// </summary>
    public int Age
    {
        get { return (int)GetValue(AgeProperty); }
        set { SetValue(AgeProperty, value); }
    }

    /// <summary>
    /// 声明并创建依赖项属性
    /// </summary>
    public static readonly DependencyProperty AgeProperty =
    DependencyProperty.Register("Age", typeof(int), typeof(CustomClass), new PropertyMetadata(0, CustomPropertyChangedCallback), CustomValidateValueCallback);

    /// <summary>
    /// 属性值更改回调方法
    /// </summary>
    /// <param name="d"></param>
    /// <param name="e"></param>
    private static void CustomPropertyChangedCallback(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
    }

    /// <summary>
    /// 属性值验证回调方法
    /// </summary>
    /// <param name="value"></param>
    /// <returns></returns>
    private static bool CustomValidateValueCallback(object value)
    {
        return true;
    }
}
```

但是这种方式不建议。仅仅是为了传输参数而大费周章，写一堆额外的功能，而且通用性差，几乎每个实例都要写一个对象，也破坏了 WPF 文档树的设计结构。

更建议的方式如下所示，采用多绑定的方式。将多绑定的各个值转换成你想要的对象或者实例模型，再传递给 ViewModel。

View代码：

```xml
xmlns:common="clr-namespace:MVVMLightDemo.Common"
```

```xml
<Grid.Resources>
    <common:UserInfoConvert x:Key="uic" />
</Grid.Resources>
```

```xml
<StackPanel Margin="10,0,0,50">
    <TextBlock Text="动态参数传递" FontWeight="Bold" FontSize="12" Margin="0,5,0,5" ></TextBlock>
    <StackPanel Orientation="Horizontal">
        <StackPanel Orientation="Vertical" Margin="0,0,10,0">
            <StackPanel Orientation="Horizontal" Margin="0,0,0,5">
                <TextBlock Text="姓名" Width="80"></TextBlock>
                <TextBox x:Name="txtUName" Width="200"/>
            </StackPanel>
            <StackPanel Orientation="Horizontal" Margin="0,0,0,5">
                <TextBlock Text="电话" Width="80"></TextBlock>
                <TextBox x:Name="txtUPhone" Width="200"/>
            </StackPanel>
            <StackPanel Orientation="Horizontal" Margin="0,0,0,5">
                <TextBlock Text="地址" Width="80"></TextBlock>
                <TextBox x:Name="txtUAdd" Width="200"/>
            </StackPanel>
            <StackPanel Orientation="Horizontal" Margin="0,0,0,5">
                <TextBlock Text="性别" Width="80"></TextBlock>
                <TextBox x:Name="txtUSex" Width="200"/>
            </StackPanel>
        </StackPanel>
        <StackPanel>
            <Button Content="点击传递" Command="{Binding DynamicParamCmd}">
                <Button.CommandParameter>
                    <MultiBinding Converter="{StaticResource uic}">
                        <Binding ElementName="txtUName" Path="Text"/>
                        <Binding ElementName="txtUSex" Path="Text"/>
                        <Binding ElementName="txtUPhone" Path="Text"/>
                        <Binding ElementName="txtUAdd" Path="Text"/>
                    </MultiBinding>
                </Button.CommandParameter>
            </Button>
        </StackPanel>
        <StackPanel Width="240" Orientation="Vertical" Margin="10,0,0,0">
            <TextBlock Text="{Binding ArgsTo.UserName,StringFormat='姓名：\{0\}'}"></TextBlock>
            <TextBlock Text="{Binding ArgsTo.UserPhone,StringFormat='电话：\{0\}'}"></TextBlock>
            <TextBlock Text="{Binding ArgsTo.UserAdd,StringFormat='地址：\{0\}'}"></TextBlock>
            <TextBlock Text="{Binding ArgsTo.UserSex,StringFormat='性别：\{0\}'}"></TextBlock>
        </StackPanel>
    </StackPanel>
</StackPanel>
```

转换器 UserInfoConvert 代码：

```csharp
public class UserInfoConvert : IMultiValueConverter
{
    /// <summary>
    /// 对象转换
    /// </summary>
    /// <param name="values">所绑定的源的值</param>
    /// <param name="targetType">目标的类型</param>
    /// <param name="parameter">绑定时所传递的参数</param>
    /// <param name="culture"><系统语言等信息</param>
    /// <returns></returns>
    public object Convert(object[] values,
                          Type targetType,
                          object parameter,
                          System.Globalization.CultureInfo culture)
    {
        if (!values.Cast<string>()
        .Any(text => string.IsNullOrEmpty(text)) && values.Count() == 4)
        {
            UserParam up = new UserParam() {
                           UserName = values[0].ToString(),
                           UserSex = values[1].ToString(),
                           UserPhone = values[2].ToString(),
                           UserAdd = values[3].ToString() };
            return up;
        }
        return null;
    }

    public object[] ConvertBack(object value,
                                Type[] targetTypes,
                                object parameter,
                                System.Globalization.CultureInfo culture)
    {
        throw new NotImplementedException();
    }
}
```

ViewModel 代码：

```csharp
#region 动态参数传递
    private UserParam argsTo;
    /// <summary>
    /// 动态参数传递
    /// </summary>
    public UserParam ArgsTo
    {
        get { return argsTo; }
        set
        {
            argsTo = value;
            RaisePropertyChanged(() => ArgsTo);
        }
    }
#endregion

    private RelayCommand<UserParam> dynamicParamCmd;
    /// <summary>
    /// 动态参数传递
    /// </summary>
    public RelayCommand<UserParam> DynamicParamCmd
    {
        get
        {
            if (dynamicParamCmd == null)
                dynamicParamCmd = new RelayCommand<UserParam>(p => ExecuteDynPar(p));
            return dynamicParamCmd;
        }
        set { dynamicParamCmd = value; }
    }

    private void ExecuteDynPar(UserParam up)
    {
        ArgsTo = up;
    }
```

效果如下：

![](https://images2015.cnblogs.com/blog/167509/201703/167509-20170328101308967-515292876.png)

到这里，命令参数绑定相关的应该就比较清楚了，这种方式也比较好操作。

**个人观点：从MVVM的模式来说，其实命令中的参数传递未必是必要的。MVVM 的目标就是消除 View 和 ViewModel 开发人员之间过于频繁的数据交互。**

**去维护一段额外的参数代码，还不如把所有的交互参数细化成在当前 DataContext 下的全局属性。View 开发人员和 ViewModel 开发人员共同维护好这份命令清单和属性清单即可。**

**而微软的很多控件也提供了类似  SelectedItem 和 SelectedValue 之类的功能属性来辅助开发。**

**四、传递原事件参数**

如果在一些特殊环境里，我们需要传递原事件的参数，那也很简单，只要设置 PassEventArgsToCommand="True" ，在 ViewModel 中对应接收参数即可。

```csharp
    private RelayCommand<DragEventArgs> dropCommand;
    /// <summary>
    /// 传递原事件参数
    /// </summary>
    public RelayCommand<DragEventArgs> DropCommand
    {
        get
        {
            if (dropCommand == null)
                dropCommand = new RelayCommand<DragEventArgs>(e => ExecuteDrop(e));
            return dropCommand;
        }
        set { dropCommand = value; }
    }

    private void ExecuteDrop(DragEventArgs e)
    {
        FileAdd = ((System.Array)e.Data.GetData(System.Windows.DataFormats.FileDrop)).GetValue(0).ToString(); 
    }
```

结果如下(将文件拖拽至红色区域内，会获取到拖拽来源，并解析参数显示出来)：

![](https://images2015.cnblogs.com/blog/167509/201703/167509-20170328105541201-1327795725.png)

**五、EventToCommand**

在WPF中，并不是所有控件都有 Command，例如 TextBox，那么当文本改变，我们需要处理一些逻辑，这些逻辑在 ViewModel 中，没有 Command 如何绑定呢？

这个时候我们就用到 EventToCommand 事件转命令，可以将一些事件，例如TextChanged，Checked 等转换成命令的方式。接下来，我们就以下拉控件为例子，来看看具体的实例：

View代码：

```textile
xmlns:mvvm="http://www.galasoft.ch/mvvmlight"
xmlns:i="clr-namespace:System.Windows.Interactivity;assembly=System.Windows.Interactivity"
```

```xml
<StackPanel Margin="10,0,0,50">
    <TextBlock Text="事件转命令执行" FontWeight="Bold" FontSize="12" Margin="0,5,0,5" ></TextBlock>
    <DockPanel x:Name="EventToCommand" >
        <StackPanel DockPanel.Dock="Left" Width="240" Orientation="Horizontal" >
            <ComboBox Width="130" ItemsSource="{Binding ResType.List}" DisplayMemberPath="Text" SelectedValuePath="Key" SelectedIndex="{Binding ResType.SelectIndex}" >
                <i:Interaction.Triggers>
                    <i:EventTrigger EventName="SelectionChanged">
                        <mvvm:EventToCommand Command="{Binding SelectCommand}"/>
                    </i:EventTrigger>
                </i:Interaction.Triggers>
            </ComboBox>
        </StackPanel>
        <StackPanel DockPanel.Dock="Right" Width="240" Orientation="Horizontal">
            <TextBlock Text="{Binding SelectInfo,StringFormat='选中值：\{0\}'}" ></TextBlock>
        </StackPanel>
    </DockPanel>
</StackPanel>
```

这里声明了 i 特性和 mvvm 特性，一个是为了拥有触发器和行为附加属性的能力，当事件触发时，会去调用相应的命令，EventName 代表触发的事件名称；一个是为了使用 MVVMLight 中 EventToCommand 功能。

这里就是当 ComboBox 执行 SelectionChanged 事件的时候，会相应去执行 SelectCommand 命令。

ViewModel 代码：

```csharp
    private RelayCommand selectCommand;
    /// <summary>
    /// 事件转命令执行
    /// </summary>
    public RelayCommand SelectCommand
    {
        get
        {
            if (selectCommand == null)
                selectCommand = new RelayCommand(() => ExecuteSelect());
            return selectCommand;
        }
        set { selectCommand = value; }
    }

    private void ExecuteSelect()
    {
        if (ResType != null && ResType.SelectIndex > 0)
        {
            SelectInfo = ResType.List[ResType.SelectIndex].Text;
        }
    }
```

结果如下：

![](https://images2015.cnblogs.com/blog/167509/201703/167509-20170328112035451-1042535937.png)

