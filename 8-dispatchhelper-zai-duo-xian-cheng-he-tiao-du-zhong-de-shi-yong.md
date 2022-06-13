---
description: http://www.cnblogs.com/wzh2010/p/6633026.html
---

# 8 DispatchHelper在多线程和调度中的使用

在应用程序中，线程可以被看做是应用程序的一个较小的执行单位。每个应用程序都至少拥有一个线程，我们称为主线程，这是在启动时调用应用程序的主方法时由操作系统分配启动的线程。

当调用和操作主线程的时候，该操作将动作添加到一个队列中。每个操作均按照将它们添加到队列中的顺序连续执行，但是，可以通过为这些动作指定优先级来影响执行顺序，而负责管理此队列的对象称之为线程调度程序。

在很多情况下，我们启动新的线程主要目的是执行操作（或等待某个操作的结果），而不会导致应用程序的其余部分被阻塞。密集型计算操作、高并发I/O操作等都是这种情况，所以，现在的复杂应用程序日益多线程化了。

当我们启动一个应用程序并创建对象时，就会调用构造函数方法所在的线程，对于 UI 元素而言，在加载 XAML 文档时，XAML 分析器会创建基于这些 UI 元素的对象。所以，所有的对象（包括UI元素）的创建都归属于当前的主线程，当然也只有主线程可以访问他们。但在实际情况中，有很多情况是要假手其它线程来处理的。

比如，在一个长交互中，我们可能需要额外的线程来处理复杂的执行过程，以避免造成线程阻塞，给用户造成界面卡死的错觉。

![](https://images2015.cnblogs.com/blog/167509/201705/167509-20170504195030851-1202840805.png)

比如下面这个例子，我们使用委托的方式模拟用户执行数据创建的操作：

调用 CreateUserInfoHelper 帮助类 和 执行 CreateProcess方法 的代码如下：

```csharp
UserParam up = new UserParam() {
                   UserAdd = txtUserAdd.Text,
                   UserName = txtUserName.Text,
                   UserPhone = txtUserPhone.Text,
                   UserSex = txtUserSex.Text };
CreateUserInfoHelper creatUser = new CreateUserInfoHelper(up);
creatUser.CreateProcess += new EventHandler<CreateUserInfoHelper.CreateArgs>(CreateProcess); //注册事件
creatUser.Create();
processPanel.Visibility = Visibility.Visible;
```

```csharp
//响应时间执行
private void CreateProcess(object sender, CreateUserInfoHelper.CreateArgs args)
{
    processBar.Value = args.process;
    processInfo.Text = String.Format("创建进度：{0}/100",args.process);
    if (args.isFinish)
    {
        if (args.userInfo != null)
        {
            ObservableCollection<UserParam> data = (ObservableCollection<UserParam>)dg.DataContext;
            data.Add(args.userInfo);
            dg.DataContext = data;
        }
        processPanel.Visibility = Visibility.Hidden;
        ClearForm();
    }
}
```

CreateUserInfoHelper 帮助类代码如下：

```csharp
public class CreateUserInfoHelper
{
    //执行进度事件（响应注册的事件）
    public event EventHandler<CreateArgs> CreateProcess;

    //待创建信息
    public UserParam up { get; set; }  

    public CreateUserInfoHelper(UserParam _up)
    {
        up = _up;
    }

    public void Create()
    {
        Thread t = new Thread(Start);//抛出一个行线程
        t.Start();
    }

    private void Start()
    {
        try
        {
            //ToDo：编写创建用户的DataAccess代码
            for (Int32 idx = 1; idx <= 10; idx++)
            {
                CreateProcess(this, new CreateArgs()
                {
                    isFinish = ((idx == 10) ? true : false),
                    process = idx * 10,
                    userInfo =null
                });
                Thread.Sleep(1000);
            }

            CreateProcess(this, new CreateArgs()
            {
                isFinish = true,
                process = 100,
                userInfo =up
            });
        }
        catch (Exception ex)
        {
            CreateProcess(this, new CreateArgs()
            {
                isFinish = true,
                process = 100,
                userInfo = null
            });
        }
    }

    /// <summary>
    /// 创建步骤反馈参数
    /// </summary>
    public class CreateArgs : EventArgs
    {
        /// <summary>
        /// 是否创建结束
        /// </summary>
        public Boolean isFinish { get; set; }

        /// <summary>
        /// 进度
        /// </summary>
        public Int32 process { get; set; }

        /// <summary>
        /// 处理后的用户信息
        /// </summary>
        public UserParam userInfo { get; set; }
    }
}
```

目的很简单：就是在创建用户信息的时候，使用另外一个线程执行创建工作，最后将结果呈现在视图列表上，而在这个创建过程中会相应的呈现进度条。

来看下效果：

![](https://images2015.cnblogs.com/blog/167509/201704/167509-20170406171505363-1751174238.png)

这里立马报错了，原因很简单，在创建对象时，该操作发生在调用 CreateUserInfoHelper 帮助类方法所在的线程中。

对于 UI 元素，在加载 XAML 文档时，XAML 分析器会创建对象。所有这一切都在主线程上进行。因此，所有这些 UI 元素都属于主线程，这也通常称为 UI 线程。

当先前代码中的后台线程尝试修改 UI 主线程的元素属性时，会导致非法的跨线程访问。因此会引发异常。

解决办法就是去通知主线程来处理 UI, 通过向主线程的 Dispatcher 队列注册工作项，来通知UI 线程更新结果。

Dispatcher 提供两个注册工作项的方法：Invoke 和 BeginInvoke。

这两个方法均调度一个委托来执行。Invoke 是同步调用，也就是说，直到 UI 线程实际执行完该委托它才返回。BeginInvoke是异步的，将立即返回。

所以，我们修改上面的代码如下：

```csharp
private void CreateProcess(object sender, CreateUserInfoHelper.CreateArgs args)
{
    this.Dispatcher.BeginInvoke((Action)delegate()
    {
        processBar.Value = args.process;
        processInfo.Text = String.Format("创建进度：{0}/100",args.process);
        if (args.isFinish)
        {
            if (args.userInfo != null)
            {
                ObservableCollection<UserParam> data = (ObservableCollection<UserParam>)dg.DataContext;
                data.Add(args.userInfo);
                dg.DataContext = data;
            }
            processPanel.Visibility = Visibility.Hidden;
            ClearForm();
        }
    });
}
```

结果如下：

![](https://images2015.cnblogs.com/blog/167509/201704/167509-20170406174212097-438852613.png)

![](https://images2015.cnblogs.com/blog/167509/201704/167509-20170406174243832-551678684.png)

实现异步执行的结果。

**MVVM 应用程序中的调度**

当从 ViewModel 执行后台操作时，情况略有不同。通常情况下，ViewModel 不从 DispatcherObject 继承。它们是继承 INotifyPropertyChanged 接口的 Plain Old CLR Objects (POCO)。

因为 ViewModel 是一个 POCO，它不能访问 Dispatcher 属性，因此我需要通过另一种方式来访问主线程，以将操作加入队列中。这正是 MVVMLight DispatcherHelper 组件的作用。

实际上，该类所做的是将主线程的调度程序保存在静态属性中，并公开一些实用工具方法，以便通过便捷且一致的方式访问。为了实现正常功能，需要在主线程上初始化该类。

最好应在应用程序生命周期的初期进行此操作，使应用程序一开始便能够访问这些功能。通常，在 MVVMLight 应用程序中，DispatcherHelper 在 App.xaml.cs 中进行初始化，App.xaml.cs 是定义应用程序启动类的文件。

在 Windows Phone 中，在应用程序的主框架刚刚创建之后，在 InitializePhoneApplication 方法中调用 Dispatcher­Helper.Initialize；

在 WPF 中，该类是在 App 构造函数中进行初始化的；

在 Windows 8 中，在窗口激活之后便立刻在 OnLaunched 中调用 Initialize 方法。

在完成了对 DispatcherHelper.Initialize 方法的调用后，DispatcherHelper 类的 UIDispatcher 属性包含对主线程的调度程序的引用。相对而言，很少直接使用该属性，但如果需要可以这样做。但是，最好使用 CheckBeginInvokeOnUi 方法。此方法将委托视为参数。

所以，将上述代码修改为：

View代码（学过Bind和Command之后应该很好理解下面这段代码，没什么特别的）：

```xml
<Grid>
    <Grid.Resources>
        <Style TargetType="{x:Type Border}" x:Key="ProcessBarBorder">
            <Setter Property="BorderBrush" Value="LightGray" ></Setter>
            <Setter Property="BorderThickness" Value="1" ></Setter>
            <Setter Property="Background" Value="White" ></Setter>
        </Style>
    </Grid.Resources>
    <!-- 延迟框 -->
    <Grid HorizontalAlignment="Stretch" VerticalAlignment="Stretch" >
        <Border Style="{StaticResource ProcessBarBorder}" Padding="5" Visibility="{Binding IsWaitingDisplay,Converter={StaticResource boolToVisibility}}" Panel.ZIndex="999" HorizontalAlignment="Center" VerticalAlignment="Center" Height="50">
            <StackPanel Orientation="Vertical" VerticalAlignment="Center" >
                <ProgressBar Value="{Binding ProcessRange}" Maximum="100" Width="400" Height="5" ></ProgressBar>
                <TextBlock Text="{Binding ProcessRange,StringFormat='执行进度:\{0\}/100'}" Margin="0,10,0,0" ></TextBlock>
            </StackPanel>
        </Border>
    </Grid>
    <StackPanel Orientation="Vertical" IsEnabled="{Binding IsEnableForm}" >
        <StackPanel>
            <DataGrid ItemsSource="{Binding UserList}" AutoGenerateColumns="False" CanUserAddRows="False" CanUserSortColumns="False" Margin="10" AllowDrop="True" IsReadOnly="True" >
                <DataGrid.Columns>
                    <DataGridTextColumn Header="学生姓名" Binding="{Binding UserName}" Width="100" />
                    <DataGridTextColumn Header="学生家庭地址" Binding="{Binding UserAdd}" Width="425" >
                        <DataGridTextColumn.ElementStyle>
                            <Style TargetType="{x:Type TextBlock}">
                                <Setter Property="TextWrapping" Value="Wrap"/>
                                <Setter Property="Height" Value="auto"/>
                            </Style>
                        </DataGridTextColumn.ElementStyle>
                    </DataGridTextColumn>
                    <DataGridTextColumn Header="电话" Binding="{Binding UserPhone}" Width="100" />
                    <DataGridTextColumn Header="性别" Binding="{Binding UserSex}" Width="100" />
                </DataGrid.Columns>
            </DataGrid>
        </StackPanel>
        <StackPanel Orientation="Horizontal" Margin="10,10,10,10">
            <StackPanel Orientation="Vertical" Margin="0,0,10,0" >
                <StackPanel Orientation="Horizontal" Margin="0,0,0,5">
                    <TextBlock Text="学生姓名" Width="80"></TextBlock>
                    <TextBox Text="{Binding User.UserName}" Width="200"/>
                </StackPanel>
                <StackPanel Orientation="Horizontal" Margin="0,0,0,5">
                    <TextBlock Text="学生电话" Width="80" ></TextBlock>
                    <TextBox Text="{Binding User.UserPhone}" Width="200"/>
                </StackPanel>
                <StackPanel Orientation="Horizontal" Margin="0,0,0,5">
                    <TextBlock Text="学生家庭地址" Width="80"></TextBlock>
                    <TextBox Text="{Binding User.UserAdd}" Width="200"/>
                </StackPanel>
                <StackPanel Orientation="Horizontal" Margin="0,0,0,5">
                    <TextBlock Text="学生性别" Width="80" ></TextBlock>
                    <TextBox Text="{Binding User.UserSex}" Width="200"/>
                </StackPanel>
                <StackPanel>
                    <Button Content="提交" Width="100" Command="{Binding AddRecordCmd}"></Button>
                </StackPanel>
            </StackPanel>
        </StackPanel>
    </StackPanel>
</Grid>
```

ViewModel代码：

（先初始化 DispatcherHelper，再调用 CheckBeginInvokeOnUI 方法来实现对 UI 线程的调度）

```csharp
public class DispatcherHelperViewModel : ViewModelBase
{
    /// <summary>
    /// 构造行数
    /// </summary>
    public DispatcherHelperViewModel()
    {
        InitData();
        DispatcherHelper.Initialize();
    }

#region 全局属性
    private ObservableCollection<UserParam> userList;
    /// <summary>
    /// 数据列表
    /// </summary>
    public ObservableCollection<UserParam> UserList
    {
        get { return userList; }
        set
        {
            userList = value;
            RaisePropertyChanged(() => UserList);
        }
    }

    private UserParam user;
    /// <summary>
    /// 当前用户信息
    /// </summary>
    public UserParam User
    {
        get { return user; }
        set
        {
            user = value;
            RaisePropertyChanged(() => User);
        }
    }

    private Boolean isEnableForm;
    /// <summary>
    /// 是否表单可用
    /// </summary>
    public bool IsEnableForm
    {
        get { return isEnableForm; }
        set
        {
            isEnableForm = value;
            RaisePropertyChanged(() => IsEnableForm);
        }
    }

    private Boolean isWaitingDisplay;
    /// <summary>
    /// 是都显示延迟旋转框
    /// </summary>
    public bool IsWaitingDisplay
    {
        get { return isWaitingDisplay; }
        set
        {
            isWaitingDisplay = value;
            RaisePropertyChanged(() => IsWaitingDisplay);
        }
    }

    private Int32 processRange;
    /// <summary>
    /// 进度比例
    /// </summary>
    public int ProcessRange
    {
        get { return processRange; }
        set
        {
            processRange = value;
            RaisePropertyChanged(() => ProcessRange);
        }
    }
#endregion

#region 全局命令
    private RelayCommand addRecordCmd;
    /// <summary>
    /// 添加资源
    /// </summary>
    public RelayCommand AddRecordCmd
    {
        get
        {
            if (addRecordCmd == null)
                addRecordCmd = new RelayCommand(()=>ExcuteAddRecordCmd());  
            return addRecordCmd;
        }
        set { addRecordCmd = value; }
    }
#endregion

#region 辅助方法
    /// <summary>
    /// 初始化数据
    /// </summary>
    private void InitData()
    {
        UserList = new ObservableCollection<UserParam>(){
                   new UserParam(){
                       UserName="周杰伦",
                       UserAdd="周杰伦地址",
                       UserPhone ="88888888",
                       UserSex="男" },
                   new UserParam(){
                       UserName="刘德华",
                       UserAdd="刘德华地址",
                       UserPhone ="88888888",
                       UserSex="男" },
                   new UserParam(){
                       UserName="刘若英",
                       UserAdd="刘若英地址",
                       UserPhone ="88888888",
                       UserSex="女" }};
        User = new UserParam();
        IsEnableForm = true;
        IsWaitingDisplay = false;
    }

    /// <summary>
    /// 执行命令
    /// </summary>
    private void ExcuteAddRecordCmd()
    {
        UserParam up = new UserParam { UserAdd = User.UserAdd, UserName = User.UserName, UserPhone = User.UserPhone, UserSex = User.UserSex };
        CreateUserInfoHelper creatUser = new CreateUserInfoHelper(up);
        creatUser.CreateProcess += new EventHandler<CreateUserInfoHelper.CreateArgs>(CreateProcess);
        creatUser.Create();
        IsEnableForm = false;
        IsWaitingDisplay = true;
    }

    /// <summary>
    /// 创建进度
    /// </summary>
    /// <param name="sender"></param>
    /// <param name="args"></param>
    private void CreateProcess(object sender, CreateUserInfoHelper.CreateArgs args)
    {
        DispatcherHelper.CheckBeginInvokeOnUI(() => {
            if (args.isFinish)
            {
                if (args.userInfo != null)
                {
                    UserList.Add(args.userInfo);
                }
                IsEnableForm = true;
                IsWaitingDisplay = false;
            }
            else
            {
                ProcessRange = args.process;
            }
        });
    }
#endregion
}
```

结果如下：

![](https://images2015.cnblogs.com/blog/167509/201704/167509-20170407143707175-931686154.png)

