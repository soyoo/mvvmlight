---
description: http://www.cnblogs.com/wzh2010/p/6679025.html
---

# 9 Messenger基础

MVVM的目标之一就是为了解耦 View 和 ViewModel。View负责视图展示，ViewModel负责业务逻辑处理，尽量保证 View.xaml.cs 中的简洁，不包含复杂的业务逻辑代码。但是，在实际情况中，View 和 ViewModel 之间的交互方式还是比较复杂的，View 和 ViewModel 的分离并不是界定的那么清晰。

**比如以下两种场景：**

1.  如果需要某张视图页面弹出对话框、弹出子窗体、处理界面元素，播放动画等。如果这些操作都放在 ViewModel 中，就会导致 ViewModel 还是要去处理 View 级别的元素，造成 View 和 ViewModel 的依赖。

    最好的办法就是，ViewModel 通知 View 应该做什么，而 View 监听接收到命令，并去处理这些界面需要处理的事情。
2. ViewModel 和 ViewModel 之间也需要通过消息传递来完成一些交互。

**MVVMLight 的 Messenger类，提供了解决了上述两个问题的能力**

Messenger 类用于应用程序的通信，接受者只能接受注册的消息类型，另外目标类型可以被指定，用Send\<TMessage, TTarget>(TMessage message) 实现；在这种情况下，信息只能被传递，如果接受者类型和目标参数类型匹配，message可以是任何简单或者复杂的对象，你可以用特定的消息类型或者创建你自己的类型（继承自它们）。

Messager 类的主要交互模式就是信息接收和发送（可以理解为“发布消息服务”和“订阅消息服务”），是不是想到观察者模式了，哈哈哈。

MVVMLight Messenger 旨在通过简单的设计模式来精简此场景：任何对象都可以是接收端；任何对象都可以是发送端；任何对象都可以是消息。

如图：&#x20;

&#x20;

**1、View 和 ViewModel 之间的消息交互**

在 View 和 ViewModel 中进行消息注册，相当于订阅服务。包含消息标志、消息参数和消息执行方法。如下所示：

**消息标志Token：ViewAlert**，用于标识只阅读某个或者某些 Sender 发送的消息，并执行相应的处理，所以 Sender 那边的 token 要保持一致；

**执行方法Action：ShowReceiveInfo**，用来执行接收到消息后的后续工作，注意这边是支持泛型能力的，所以传递参数很方便。

View.xaml.cs 代码如下：

```csharp
public partial class NessagerForView : Window
{
    public NessagerForView()
    {
        InitializeComponent();

        //消息标志token：ViewAlert，用于标识只阅读某个或者某些Sender发送的消息，
        //并执行相应的处理，所以Sender那边的token要保持一致
        //执行方法Action：ShowReceiveInfo，用来执行接收到消息后的后续工作，
        //注意这边是支持泛型能力的，所以传递参数很方便。
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
}
```

ViewModel代码：

```csharp
public class MessengerRegisterForVViewModel : ViewModelBase
{
    public MessengerRegisterForVViewModel()
    {
    }

#region 命令
    private RelayCommand sendCommand;
    /// <summary>
    /// 发送命令
    /// </summary>
    public RelayCommand SendCommand
    {
        get
        {
            if (sendCommand == null)
                sendCommand = new RelayCommand(() => ExcuteSendCommand());
            return sendCommand;
        }
        set { sendCommand = value; }
    }

    private void ExcuteSendCommand()
    {
        Messenger.Default.Send<String>("ViewModel通知View弹出消息框", "ViewAlert"); //注意：token参数一致
    }
#endregion
}
```

结果：

![](https://images2015.cnblogs.com/blog/167509/201704/167509-20170410154904016-1292220730.png)

**2、ViewModel 和 ViewModel 之间的消息交互**

ViewModel 和ViewModel 在很多种场景下，也需要通过消息传递来完成一些交互。比如，我打开了两个视图，一个视图是用户信息列表，一个视图是用户信息添加页面，如果想要达到添加信息之后，用户信息列表视图实时刷新，用消息通知无疑是一个很棒的体验。

我们来模拟一下：

MessengerRegisterViewModel代码：

```csharp
public class MessengerRegisterViewModel : ViewModelBase
{
    public MessengerRegisterViewModel()
    {
        ///Messenger：信使
        ///Recipient：收件人
        Messenger.Default.Register<String>(this, "Message", ShowReceiveInfo);
    }

#region 属性
    private String receiveInfo;
    /// <summary>
    /// 接收到信使传递过来的值
    /// </summary>
    public String ReceiveInfo
    {
        get { return receiveInfo; }
        set
        {
            receiveInfo = value;
            RaisePropertyChanged(() => ReceiveInfo);
        }
    }
#endregion

#region 启动新窗口
    private RelayCommand showSenderWindow;
    public RelayCommand ShowSenderWindow
    {
        get
        {
            if (showSenderWindow == null)
                showSenderWindow = new RelayCommand(() => ExcuteShowSenderWindow());
            return showSenderWindow; 
        }
        set { showSenderWindow = value; }
    }

    private void ExcuteShowSenderWindow()
    {
        MessengerSenderView sender = new MessengerSenderView();
        sender.Show();
    }
#endregion 

#region 辅助函数
    /// <summary>
    /// 显示收件的信息
    /// </summary>
    /// <param name="msg"></param>
    private void ShowReceiveInfo(String msg)
    {
        ReceiveInfo += msg + "\n";
    }
#endregion
}
```

MessengerSenderViewModel代码：

```csharp
public class MessengerSenderViewModel : ViewModelBase
{
    public MessengerSenderViewModel()
    {
    }

#region 属性
    private String sendInfo;
    /// <summary>
    /// 发送消息
    /// </summary>
    public String SendInfo
    {
        get { return sendInfo; }
        set
        {
            sendInfo = value;
            RaisePropertyChanged(() => SendInfo);
        }
    }
#endregion

#region 命令
    private RelayCommand sendCommand;
    /// <summary>
    /// 发送命令
    /// </summary>
    public RelayCommand SendCommand
    {
        get
        {
            if (sendCommand == null)
                sendCommand = new RelayCommand(() => ExcuteSendCommand());
            return sendCommand;
        }
        set { sendCommand = value; }
    }

    private void ExcuteSendCommand()
    {
        Messenger.Default.Send<String>(SendInfo, "Message");
    }
#endregion
}
```

结果如下：

