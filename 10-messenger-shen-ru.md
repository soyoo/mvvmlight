---
description: http://www.cnblogs.com/wzh2010/p/6689423.html
---

# 10 Messenger 深入

**1 Messager交互结构和消息类型**

衔接上篇，Messeger是信使的意思，顾名思义，它的目的是用于 View 和 ViewModel 以及 ViewModel 和 ViewModel 之间的消息通知和接收。

Messenger 类应用于应用程序的通信，接收者只能接收注册的消息类型，另外，目标类型可以被指定，用 Send\<TMessage, TTarget>(TMessage message) 实现，在这种情况下，信息只能被传递如果接收者类型和目标参数类型匹配，message 可以是任何简单或者复杂的对象，你可以用特定的消息类型或者创建你自己的类型（继承自它们）。

交互结构如下所示：

&#x20;\*\*\*\*

消息类型如下表所示：

| **message消息对象类型**         | **说明**                                                                            |
| ------------------------- | --------------------------------------------------------------------------------- |
| MessageBase               | 简单的消息类，携带可选的信息关于消息发布者的                                                            |
| GenericMessage            | 泛型消息                                                                              |
| NotificationMessage       | 用于发送一个string类型通知给接受者                                                              |
| NotificationMessage       | 和上面一样是一个，且具有泛型功能                                                                  |
| NotificationMessage       | 向接受者发送一个通知，允许接受者向发送者回传消息                                                          |
| NotificationMessageAction | NotificationMessage的泛型方式                                                          |
| DialogMessage             | 发送者（通常是View）显示对话，并且传递调用者得回传结果（用于回调），接受者可以选择怎样显示对话框，可以使用标准的 MessageBox 也可以是自定义弹出窗口 |
| PropertyChangedMessage    | 用于广播一个属性的改变在发送者里，和PropertyChanged事件有完全箱体内各的目的，但是是一种弱联系方式                          |

**2 注册消息的模式**

上篇给出了注册的方法，但是注册可以有很多种方式，最常见的就是命名方法调用和Lambda表达式调用的方式：

**2.1 基本的命名方法注册**

```csharp
// 使用命名方法进行注册
Messenger.Default.Register<String>(this, HandleMessage);

// 卸载当前(this)对象注册的所有MVVMLight消息
this.Unloaded += (sender, e) => Messenger.Default.Unregister(this);

private void HandleMessage(String msg)
{
    //Todo
}
```

**2.2 使用 Lambda 注册**

```csharp
Messenger.Default.Register<String>(this, message => {
    // Todo 
}); 
//卸载当前(this)对象注册的所有MVVMLight消息
this.Unloaded += (sender, e) => Messenger.Default.Unregister(this);
```

**3 跨线程访问**

之前在第8篇《[利刃 MVVMLight 8：DispatchHelper在多线程和调度中的使用](http://www.cnblogs.com/wzh2010/p/6633026.html)》中，

我们有讨论过在异步线程中使用事件来执行和获取相关的执行步骤；但是，如果异步线程中的某个方法需要操作主线程（UI线程的时候）的 UI 是不允许的。

Windows 中的控件被绑定到特定的 UI 线程（主线程）中，其它线程是不允许访问的，因为不具备线程安全性和规范性。所以，MVVMLight 才有了调度帮助类（DispatchHelper）来处理不同线程中的调度方案。

从这边可以衍生到异步线程下，对 UI 线程（主线程）的信息发送和接收。所以，之前的代码 DispatchHelper 可以改装如下：

注册模块（ViewModel中）：

```csharp
public class MessengerForDispatchViewModel : ViewModelBase
{
    /// <summary>
    /// 构造函数
    /// </summary>
    public MessengerForDispatchViewModel()
    {
        InitData();
        DispatcherHelper.Initialize();

        ///Messenger：信使
        ///Recipient：收件人  
        Messenger.Default.Register<TopUserInfo>(this, "UserMessenger", FeedBack);
    }
}
```

发送模块（异步线程中代码）：

```csharp
private void Start()
{
    TopUserInfo ui = new TopUserInfo();

    //ToDo：编写创建用户的DataAccess代码
    for (Int32 idx = 1; idx <= 9; idx++)
    {
        Thread.Sleep(1000);
        ui = new TopUserInfo() {
             isFinish = false,
             process = idx * 10,
             userInfo = null
        };

        DispatcherHelper.CheckBeginInvokeOnUI(() =>
        {
            Messenger.Default.Send<TopUserInfo>(ui, "UserMessenger");
        });
    }

    Thread.Sleep(1000);
    ui = new TopUserInfo() {
         isFinish = true,
         process = 100,
         userInfo = up
    };

    DispatcherHelper.CheckBeginInvokeOnUI(() =>
    {
        Messenger.Default.Send<TopUserInfo>(ui, "UserMessenger");
    });
}
```

结果： ![](https://images2015.cnblogs.com/blog/167509/201705/167509-20170522213129757-356644662.png)&#x20;

**4 释放注册信息**

**4.1 基于 View 界面内的 UnRegister 的释放**

为当前视图页面的 Unload 事件附加释放注册信息的功能：

```csharp
// 卸载当前(this)对象注册的所有MVVMLight消息 
this.Unloaded += (sender, e) => Messenger.Default.Unregister(this);
```

**4.2 基于 ViewModel 类中的 UnRegister 释放**

用户在关闭使用页面的时候同时调用该方法，释放注册，这个需要开发人员在关闭视图模型的时候发起：

```csharp
/// <summary>
/// 手动调用释放注册信息（该视图模型内的所有注册信息全部释放）
/// </summary>
public void ReleaseRegister()
{
    Messenger.Default.Unregister(this); 
}
```

**5 释放注册信息和内存处理**

为了避免不必要的内存泄漏， .Net 框架提出了比较实用的 WeakReference（弱引用）对象。该功能允许将对象的引用进行弱存储。如果对该对象的所有引用都被释放了，则垃圾回收机便可回收该对象。

类似的，将所有的注册信息保存在一个弱引用的存储区域，一旦注册信息所寄宿的宿主（View 或者 ViewModel）被释放，引用被清空，该注册信息也会在一定时间内被释放。

下面的表格来源于 [**Laurent Bugnion**](https://msdn.microsoft.com/zh-cn/magazine/ee532098.aspx?sdmr=laurentbugnion\&sdmi=authors) 对 MVVMLight 的说明文档，未取消注册时的内存泄漏风险：

| **可见性**  | **WPF** | **Silverlight** | **Windows Phone 8** | **Windows 运行时** |
| -------- | ------- | --------------- | ------------------- | --------------- |
| 静态       | 无风险     | 无风险             | 无风险                 | 无风险             |
| 公共       | 无风险     | 无风险             | 无风险                 | 无风险             |
| 内部       | 无风险     | 风险              | 风险                  | 无风险             |
| 专用       | 无风险     | 风险              | 风险                  | 无风险             |
| 匿名Lambda | 无风险     | 风险              | 风险                  | 无风险             |

**6 专有信道和广播信道**

**6.1 过滤 Messenger 发送端**

通过判断发送端来确认是否是发送给自己的：

```csharp
public class ForSourceSenderViewModel : ViewModelBase
{
    public ForSourceSenderViewModel(){}

#region 全局命令
    private RelayCommand sendMsg;
    /// <summary>
    /// 发送消息
    /// </summary>
    public RelayCommand SendMsg
    {
        get
        {
            if (sendMsg == null)
                sendMsg = new RelayCommand(() => ExcuteSendMsh());
            return sendMsg;
        }
        set { sendMsg = value; }
    }
#endregion

#region 附属方法
    private void ExcuteSendMsh()
    {
        NotificationMessage nm = new NotificationMessage(this,"发送源消息");
        Messenger.Default.Send<NotificationMessage>(nm);
    }
#endregion
}
```

```csharp
Messenger.Default.Register<NotificationMessage>(this, message =>
{
    if (message.Sender is ForSourceSenderViewModel)
    {
        // 判断来源来接受消息
        MsgInfo = message.Notification;
    }
});
```

**6.2 开设专用的 Messenger 通道**

```csharp
private Messenger myMessenger;
public MessengerForSourceViewModel()
{
    // 构造函数  
    myMessenger = new Messenger();

    // 注入一个Key为MyMessenger的Messenger对象
    SimpleIoc.Default.Register(() => myMessenger, "MyMessenger");

    // 注册myMessenger，开启监听
    myMessenger.Register<NotificationMessage>(this, message => {
        // 判断来源来接受消息
        MsgInfo = message.Notification;
    });
}
```

```csharp
#region 全局命令
    private RelayCommand sendMsg;

    /// <summary>
    /// 发送消息
    /// </summary>
    public RelayCommand SendMsg
    {
        get
        {
            if (sendMsg == null)
                sendMsg = new RelayCommand(() => ExcuteSendMsh());
            return sendMsg;
        }
        set { sendMsg = value; }
    }
#endregion

#region 附属方法
    private void ExcuteSendMsh()
    {
        NotificationMessage nm = new NotificationMessage(this, String.Format("发送消息：{0}", DateTime.Now));
        // 获取已存在的Messenger实例
        Messenger myMessenger = SimpleIoc.Default.GetInstance<Messenger>("MyMessenger");
        // 消息发送
        myMessenger.Send<NotificationMessage>(nm);//消息发送
    }
#endregion
```

**6.3 使用令牌（Token）区分和使用信道**

这是最常用的方式，使用专属 Token，可以区分不同的信道，并提高复用性。

Messenger 中包含一个 Token 参数，发送方和注册方使用同一个 Token，便可保证数据在该专有信道中流通，所以令牌是筛选和隔离消息的最好办法。

```csharp
// 以ViewAlert 为 Tokon 标志，进行消息发送
Messenger.Default.Send<String>("ViewModel通知View弹出消息框", "ViewAlert");
```

```csharp
public MessagerForView()
{
    InitializeComponent();

    //消息标志token：ViewAlert，用于标识只阅读某个或者某些Sender发送的消息，并执行相应的处理，所以Sender那边的token要保持一致
    //执行方法Action：ShowReceiveInfo，用来执行接收到消息后的后续工作，注意这边是支持泛型能力的，所以传递参数很方便。
    Messenger.Default.Register<String>(this, "ViewAlert", ShowReceiveInfo);
    this.DataContext = new MessengerRegisterForVViewModel();

    //卸载当前(this)对象注册的所有MVVMLight消息
    this.Unloaded += (sender, e) => Messenger.Default.Unregister(this);
}

/// <summary>
/// 接收到消息后的后续工作：根据返回来的信息弹出消息框
/// </summary>
/// <param name="msg"></param>
private void ShowReceiveInfo(String msg)
{
    MessageBox.Show(msg);
}
```

**7 使用内置消息**

比如，我们上面用到的 NotificationMessage ，以及PropertyChanged­Message。

Notification 是一种消息通知机制，而 PropertyChanged­Message 主要指的是当属性改变的时候，执行通知操作。

```csharp
public class PropertyChangedViewModel : ViewModelBase
{
    public PropertyChangedViewModel(){}

    // 注册为该属性，该属性变化时进行消息发送
    public const string PropertyName = "UserName";

#region 全局变量
    private String userName;
    /// <summary>
    /// 用户名称
    /// </summary>
    public string UserName
    {
        get { return userName; }
        set
        {
            String oldValue = userName;
            userName = value;
            // 这边相应配置上发送参数
            RaisePropertyChanged(() => UserName, oldValue, value, true);
        }
    }
#endregion
```

```csharp
Messenger.Default.Register<PropertyChangedMessage<String>>(this, message =>
{
    // 接受特定属性值相关信道的消息
    if (message.PropertyName == PropertyChangedViewModel.PropertyName)
    {
        // 输出旧值到新值的内容
        PropertyChangedInfo = (message.OldValue + " --> " + message.NewValue);
    }
});
```

结果：

![](https://images2015.cnblogs.com/blog/167509/201705/167509-20170525121427654-1325564228.png)
