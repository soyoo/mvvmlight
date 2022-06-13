---
description: http://www.cnblogs.com/wzh2010/p/6557037.html
---

# 6 命令基础

在 MVVMLight 框架里面，事件是 WPF 应用程序中 UI 与后台代码进行交互的最主要的方式；与传统方式不同，mvvm 中主要通过绑定到命令来进行事件的处理，因此，要了解mvvm中处理事件的方式，就必须先熟悉命令的工作原理。

**RelayCommand命令：** WPF 命令是通过实现 ICommand 接口创建的；ICommand 公开了两个方法（Execute 及 CanExecute）和一个 EventHandler 类型的事件（CanExecuteChanged）。

| **Execute方法**    | 执行与命令关联的操作                                                                                       |
| ---------------- | ------------------------------------------------------------------------------------------------ |
| **CanExecute方法** | 确定是否可以在当前命令目标上执行命令，返回值为true则按钮可用，为false的时候按钮disable；在 MvvmLight 中实现 ICommand 接口的类是 RelayCommand。 |

RelayCommand 通过构造函数初始化 Execute 和 CanExecute 方法，因此，构造函数传入的是委托类型的参数，Execute 和 CanExecute则执行的是委托的方法；如图所示：

![](https://images2015.cnblogs.com/blog/167509/201703/167509-20170320095244143-1683828411.png)

![](https://images2015.cnblogs.com/blog/167509/201703/167509-20170316214337370-1678371819.png)

相对于 CodeBehind 的方式，使用命令会好很多：

最大的特点就是，解耦了 View 和 ViewModel 之间的交互行为，将视图的显示和业务逻辑分开；对 View 上的某个元素进行命令的绑定，触发点击操作的时候，这个按钮实际上完成的是对应 ViewModel 中的所绑定的方法。这里，我们用到 MvvmLight 框架中的RelayCommand。

现在，我们来看一个例子，将我们上篇的那个例子修改一下，加进 CanExcute() 方法和列表数据的呈现。

Model 代码：

```csharp
[MetadataType(typeof(BindDataAnnotationsViewModel))]
public class ValidateUserInfo : ValidateModelBase
{
#region 属性
    private String userName;
    /// <summary>
    /// 用户名
    /// </summary>
    [Required]
    public String UserName
    {
        get { return userName; }
        set { userName = value; RaisePropertyChanged(() => UserName); }
    }

    private String userPhone;
    /// <summary>
    /// 用户电话
    /// </summary>
    [Required]
    [RegularExpression(@"^[-]?[1-9]{8,11}\d*$|^[0]{1}$", ErrorMessage = "用户电话必须为8-11位的数值.")]
    public String UserPhone
    {
        get { return userPhone; }
        set { userPhone = value; RaisePropertyChanged(() => UserPhone); }
    }

    private String userEmail;
    /// <summary>
    /// 用户邮件
    /// </summary>
    [Required]
    [StringLength(100, MinimumLength = 2)]
    [RegularExpression("^\\s*([A-Za-z0-9_-]+(\\.\\w+)*@(\\w+\\.)+\\w{2,5})\\s*$", ErrorMessage = "请填写正确的邮箱地址.")]
    public String UserEmail
    {
        get { return userEmail; }
        set { userEmail = value; RaisePropertyChanged(() => UserEmail); }
    }
#endregion
```

View 代码：

“提交”按钮绑定了一个 Command，这个 Command 指向对应的 ViewModel 中的SubmitCmd 方法；这样确实很赞，SubmitCmd 独立性、复用性很高。

```xml
<Window x:Class="MVVMLightDemo.View.CommandView"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        DataContext="{Binding Source={StaticResource Locator},Path=Command}"
        Title="CommandView" Height="500" Width="800">
    <Grid>
        <StackPanel Orientation="Vertical" >
            <GroupBox Header="命令" Margin="10 20 10 10">
                <StackPanel Orientation="Vertical" Margin="0,10,0,0">
                    <StackPanel.Resources>
                        <Style TargetType="StackPanel">
                            <Setter Property="Orientation" Value="Horizontal"/>
                            <Setter Property="Margin" Value="0,0,0,4"/>
                        </Style>
                        <Style TargetType="Label" BasedOn="{StaticResource {x:Type Label}}">
                            <Setter Property="Width" Value="100"/>
                            <Setter Property="VerticalAlignment" Value="Center"/>
                        </Style>
                        <Style TargetType="CheckBox" BasedOn="{StaticResource {x:Type CheckBox}}">
                            <Setter Property="Padding" Value="0,3"/>
                        </Style>
                        <Style TargetType="RadioButton" BasedOn="{StaticResource {x:Type RadioButton}}">
                            <Setter Property="Padding" Value="0,3"/>
                        </Style>
                    </StackPanel.Resources>
                    <StackPanel>
                        <Label Content="用户名" Target="{Binding ElementName=UserName}"/>
                        <TextBox Width="150" Text="{Binding ValidateUI.UserName,UpdateSourceTrigger=PropertyChanged,ValidatesOnDataErrors=True}"></TextBox>
                    </StackPanel>
                    <StackPanel>
                        <Label Content="用户邮箱" Target="{Binding ElementName=UserEmail}"/>
                        <TextBox Width="150" Text="{Binding ValidateUI.UserEmail, UpdateSourceTrigger=PropertyChanged, ValidatesOnDataErrors=True}"/>
                    </StackPanel>
                    <StackPanel>
                        <Label Content="用户电话" Target="{Binding ElementName=UserPhone}"/>
                        <TextBox Width="150" Text="{Binding ValidateUI.UserPhone,UpdateSourceTrigger=PropertyChanged, ValidatesOnDataErrors=True}"/>
                    </StackPanel>
                    <StackPanel>
                        <Label Foreground="Red" Content="提示：验证全部通过的时候提交按钮可操作！" Width="250" ></Label>
                    </StackPanel>
                    <Button Content="提交" Margin="100,16,0,0" HorizontalAlignment="Left" Command="{Binding SubmitCmd}"/>
                </StackPanel>
            </GroupBox>
            <StackPanel>
                <DataGrid x:Name="dg1" ItemsSource="{Binding List}" AutoGenerateColumns="False" CanUserAddRows="False" 
                          CanUserSortColumns="False" Margin="10" AllowDrop="True" IsReadOnly="True">
                    <DataGrid.Columns>
                        <DataGridTextColumn Header="用户姓名" Binding="{Binding UserName}" Width="100"/>
                        <DataGridTextColumn Header="邮箱" Binding="{Binding UserEmail}" Width="400" />
                        <DataGridTextColumn Header="电话" Binding="{Binding UserPhone}" Width="100" />
                    </DataGrid.Columns>
                </DataGrid>
            </StackPanel>
        </StackPanel>
    </Grid>
</Window>
```

ViewModel代码：

这里需要注意的是，用户在界面上点击“提交”按钮的时候，去 ViewModel 里面寻找名为SubmitCmd 的 RelayCommand 命令对象，如果找不到，则执行无效果，所以名称一定要对应上，而且需要是 public 的访问级别。

CanExcute 方法这边用表单是否验证通过来判断命令是否执行，如果返回的是false，则该命令不执行，这时候“提交”按钮也是不可用（Disable）的。

```csharp
using GalaSoft.MvvmLight;
using GalaSoft.MvvmLight.CommandWpf;
using MVVMLightDemo.Model;
using System.Collections.ObjectModel;

namespace MVVMLightDemo.ViewModel
{
    public class CommandViewModel : ViewModelBase
    {
        public CommandViewModel()
        {
            //构造函数
            ValidateUI = new ValidateUserInfo();
            List = new ObservableCollection<ValidateUserInfo>();
        }

#region 全局属性
        private ObservableCollection<ValidateUserInfo> list;
        /// <summary>
        /// 用户数据列表
        /// </summary>
        public ObservableCollection<ValidateUserInfo> List
        {
            get { return list; }
            set { list = value; }
        }

        private ValidateUserInfo validateUI;
        /// <summary>
        /// 当前操作的用户信息
        /// </summary>
        public ValidateUserInfo ValidateUI
        {
            get { return validateUI; }
            set
            {
                validateUI = value;
                RaisePropertyChanged(() => ValidateUI);
            }
        }
#endregion

#region 全局命令
        private RelayCommand submitCmd;
        /// <summary>
        /// 执行提交命令的方法
        /// </summary>
        public RelayCommand SubmitCmd
        {
            get
            {
                if (submitCmd == null)
                    return new RelayCommand(() => ExcuteValidForm(),
                                            CanExcute);
                return submitCmd;
            }
            set { submitCmd = value; }
        }
#endregion

#region 附属方法
        /// <summary>
        /// 执行提交方法
        /// </summary>
        private void ExcuteValidForm()
        {
            List.Add(new ValidateUserInfo(){
                         UserEmail = ValidateUI.UserEmail,
                         UserName = ValidateUI.UserName,
                         UserPhone = ValidateUI.UserPhone });
        }

        /// <summary>
        /// 是否可执行（这边用表单是否验证通过来判断命令是否执行）
        /// </summary>
        /// <returns></returns>
        private bool CanExcute()
        {
            return ValidateUI.IsValidated;
        }
#endregion
    }
}
```

结果如下：

![](https://images2015.cnblogs.com/blog/167509/201703/167509-20170320153243174-645425607.png)

这是最简单的命令操作了，下篇我们来深入了解下命令和 EventToCommand 的相关内容。
