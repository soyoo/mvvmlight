---
description: http://www.cnblogs.com/wzh2010/p/6518834.html
---

# 5 绑定在表单验证上的应用

表单验证是MVVM体系中重要的一块，绑定除了推动 Model-View-ViewModel (MVVM) 模式松散耦合逻辑、数据和 UI 定义的关系之外，还为业务数据验证方案提供强大而灵活的支持。

WPF 中的数据绑定机制包括多个选项，可用于在创建可编辑视图时校验输入数据的有效性。

常见的表单验证机制有如下几种类型：

| **验证类型**              | **说明**                                                                                                                                                                            |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Exception 验证**      | 通过在某个 Binding 对象上设置 ValidatesOnExceptions 属性，如果源对象属性设置已修改的值的过程中引发异常，则抛出错误并为该 Binding 设置验证错误。                                                                                      |
| **ValidationRule 验证** | Binding 类具有一个用于提供 ValidationRule 派生类实例的集合的属性。这些 ValidationRules 需要覆盖某个 Validate 方法，该方法由 Binding 在每次绑定控件中的数据发生更改时进行调用。如果 Validate 方法返回无效的 ValidationResult 对象，则将为该 Binding 设置验证错误。 |
| **IDataErrorInfo 验证** | 通过在绑定数据源对象上实现 IDataErrorInfo 接口，并在 Binding 对象上设置 ValidatesOnDataErrors 属性，Binding 将调用从绑定数据源对象公开的 IDataErrorInfo API。如果从这些属性调用返回非 null 或非空字符串，则将为该 Binding 设置验证错误。                 |

验证交互的关系模式如图：

![](https://images2015.cnblogs.com/blog/167509/201703/167509-20170314132729448-543973432.png)

我们在使用 WPF 中的数据绑定来呈现业务数据时，通常会使用 Binding 对象在目标控件的单个属性与数据源对象属性之间提供数据管道。

如果要使得绑定验证有效，首先需要进行 TwoWay 数据绑定。这表明，除了从源属性流向目标属性以进行显示的数据之外，编辑过的数据也会从目标流向源。

这就是伟大的双向数据绑定的精髓，所以在MVVM中做数据校验，会容易的多。

当 TwoWay 数据绑定中输入或修改数据时，将启动以下工作流：

1. 用户通过键盘、鼠标、手写板或者其他输入设备来输入或修改数据，从而改变绑定的目标信息；
2. 设置源属性值；
3. 触发 Binding.SourceUpdated 事件；
4. 如果数据源属性上的 setter 引发异常，则异常会由 Binding 捕获，并可用于指示验证错误；
5. 如果实现了 IDataErrorInfo 接口，则会对数据源对象调用该接口的方法获得该属性的错误信息；
6. 向用户呈现验证错误指示，并触发 Validation.Error 附加事件。

绑定目标向绑定源发送数据更新的请求，而绑定源则对数据进行验证，并根据不同的验证机制进行反馈。&#x20;

下面我们用实例来对比下这几种验证机制，在此之前，我们先做一个事情，就是写一个错误触发的样式，来保证错误触发的时候直接、清晰的向用户反馈出去。

我们新建一个 ResourceDictionary 文件，命名为 TextBox.xaml，下面这个是 ResourceDictionary 文件的内容，TargetType 是 TextBoxBase 基础的控件，如 TextBox 和RichTextBox。

代码比较简单，注意标红的内容，设计一个红底白字的提示框，当源属性触发错误验证的时候，把验证对象集合中的错误内容显示出来。

```xml
<ResourceDictionary
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <Style x:Key="{x:Type TextBoxBase}" TargetType="{x:Type TextBoxBase}" BasedOn="{x:Null}">
        <Setter Property="BorderThickness" Value="1"/>
        <Setter Property="Padding" Value="2,1,1,1"/>
        <Setter Property="AllowDrop" Value="true"/>
        <Setter Property="FocusVisualStyle" Value="{x:Null}"/>
        <Setter Property="ScrollViewer.PanningMode" Value="VerticalFirst"/>
        <Setter Property="Stylus.IsFlicksEnabled" Value="False"/>
        <Setter Property="SelectionBrush" Value="{DynamicResource Accent}" />
        <Setter Property="Validation.ErrorTemplate">
            <Setter.Value>
                <ControlTemplate>
                    <StackPanel Orientation="Horizontal">
                        <!-- 标红 -->
                        <Border BorderThickness="1" BorderBrush="#FFdc000c" VerticalAlignment="Top">
                            <Grid>
                                <AdornedElementPlaceholder x:Name="adorner" Margin="-1"/>
                            </Grid>
                        </Border>
                        <Border x:Name="errorBorder" Background="#FFdc000c" Margin="8,0,0,0"
                                Opacity="0" CornerRadius="0"
                                IsHitTestVisible="False"
                                MinHeight="24" >
                            <TextBlock Text="{Binding ElementName=adorner, Path=AdornedElement.(Validation.Errors)[0].ErrorContent}"
                                       Foreground="White" Margin="8,2,8,3" TextWrapping="Wrap" VerticalAlignment="Center"/>
                        </Border>
                    </StackPanel>
                    <ControlTemplate.Triggers>
                        <DataTrigger Value="True">
                            <!-- 标红 -->
                            <DataTrigger.Binding>
                                <Binding ElementName="adorner" Path="AdornedElement.IsKeyboardFocused" />
                            </DataTrigger.Binding>
                            <DataTrigger.EnterActions>
                                <BeginStoryboard x:Name="fadeInStoryboard">
                                    <Storyboard>
                                        <DoubleAnimation Duration="00:00:00.15"
                                                         Storyboard.TargetName="errorBorder"
                                                         Storyboard.TargetProperty="Opacity" To="1"/>
                                    </Storyboard>
                                </BeginStoryboard>
                            </DataTrigger.EnterActions>
                            <DataTrigger.ExitActions>
                                <StopStoryboard BeginStoryboardName="fadeInStoryboard"/>
                                <BeginStoryboard x:Name="fadeOutStoryBoard">
                                    <Storyboard>
                                        <DoubleAnimation Duration="00:00:00" Storyboard.TargetName="errorBorder" Storyboard.TargetProperty="Opacity" To="0"/>
                                    </Storyboard>
                                </BeginStoryboard>
                            </DataTrigger.ExitActions>
                        </DataTrigger>
                    </ControlTemplate.Triggers>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="{x:Type TextBoxBase}">
                    <Border x:Name="Bd" BorderThickness="{TemplateBinding BorderThickness}" BorderBrush="{TemplateBinding BorderBrush}" Background="{TemplateBinding Background}" Padding="{TemplateBinding Padding}" SnapsToDevicePixels="true">
                        <ScrollViewer x:Name="PART_ContentHost" RenderOptions.ClearTypeHint="Enabled" SnapsToDevicePixels="{TemplateBinding SnapsToDevicePixels}"/>
                    </Border>
                    <ControlTemplate.Triggers>
                        <Trigger Property="IsEnabled" Value="false">
                            <Setter Property="Foreground" Value="{DynamicResource InputTextDisabled}"/>
                        </Trigger>
                        <Trigger Property="IsReadOnly" Value="true">
                            <Setter Property="Foreground" Value="{DynamicResource InputTextDisabled}"/>
                        </Trigger>
                        <Trigger Property="IsFocused" Value="true">
                            <Setter TargetName="Bd" Property="BorderBrush" Value="{DynamicResource Accent}" />
                        </Trigger>
                        <MultiTrigger>
                            <MultiTrigger.Conditions>
                                <Condition Property="IsReadOnly" Value="False"/>
                                <Condition Property="IsEnabled" Value="True"/>
                                <Condition Property="IsMouseOver" Value="True"/>
                            </MultiTrigger.Conditions>
                            <Setter Property="Background" Value="{DynamicResource InputBackgroundHover}"/>
                            <Setter Property="BorderBrush" Value="{DynamicResource InputBorderHover}"/>
                            <Setter Property="Foreground" Value="{DynamicResource InputTextHover}"/>
                        </MultiTrigger>
                    </ControlTemplate.Triggers>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>
    <Style BasedOn="{StaticResource {x:Type TextBoxBase}}" TargetType="{x:Type TextBox}"></Style>
    <Style BasedOn="{StaticResource {x:Type TextBoxBase}}" TargetType="{x:Type RichTextBox}"></Style>
</ResourceDictionary>
```

然后在 App.Xaml 中全局注册到整个应用中。

```xml
<Application x:Class="MVVMLightDemo.App"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml" 
    StartupUri="View/BindingFormView.xaml"
    xmlns:d="http://schemas.microsoft.com/expression/blend/2008" 
     d1p1:Ignorable="d"
    xmlns:d1p1="http://schemas.openxmlformats.org/markup-compatibility/2006"
    xmlns:vm="clr-namespace:MVVMLightDemo.ViewModel"
    xmlns:Common="clr-namespace:MVVMLightDemo.Common">
    <Application.Resources>
        <ResourceDictionary>
            <ResourceDictionary.MergedDictionaries>
                <ResourceDictionary Source="/MVVMLightDemo;component/Assets/TextBox.xaml" />
            </ResourceDictionary.MergedDictionaries>
            <vm:ViewModelLocator x:Key="Locator" d:IsDataSource="True" />
            <Common:IntegerToSex x:Key="IntegerToSex" d:IsDataSource="True" />
        </ResourceDictionary>
    </Application.Resources>
</Application>
```

达到的效果如下：

![](https://images2015.cnblogs.com/blog/167509/201704/167509-20170414095844173-2104154642.png)

**下面详细描述下这三种验证模式**

**1、Exception 验证：**

正如说明中描述的那样，在具有绑定关系的源字段模型上做验证异常的引发并抛出，在 View中的 Xaml 对象上设置 ExceptionValidationRule 属性，响应捕获异常并显示。

View 代码：

```xml
<GroupBox Header="Exception 验证" Margin="10 10 10 10" DataContext="{Binding Source={StaticResource Locator},Path=ValidateException}" >
    <StackPanel x:Name="ExceptionPanel" Orientation="Vertical" Margin="0,10,0,0" >
        <StackPanel>
            <Label Content="用户名" Target="{Binding ElementName=UserNameEx}"/>
            <TextBox x:Name="UserNameEx" Width="150">
                <TextBox.Text>
                    <Binding Path="UserNameEx" UpdateSourceTrigger="PropertyChanged">
                        <Binding.ValidationRules>
                            <ExceptionValidationRule></ExceptionValidationRule>
                        </Binding.ValidationRules>
                    </Binding>
                </TextBox.Text>
            </TextBox>
        </StackPanel>
    </StackPanel>
</GroupBox>
```

ViewModel 代码：

```csharp
/// <summary>
/// Exception 验证
/// </summary>
public class ValidateExceptionViewModel : ViewModelBase
{
    public ValidateExceptionViewModel()
    {
    }

    private String userNameEx;
    /// <summary>
    /// 用户名称（不为空）
    /// </summary>
    public string UserNameEx
    {
        get { return userNameEx; }
        set
        {
            userNameEx = value;
            RaisePropertyChanged(() => UserNameEx);
            if (string.IsNullOrEmpty(value))
            {
                throw new ApplicationException("该字段不能为空！");
            }
        }
    }
}
```

结果如图：

![](https://images2015.cnblogs.com/blog/167509/201703/167509-20170314141750541-1059741509.png)

将验证失败的信息直接抛出来，这无疑是最简单粗暴的，实现也很简单，但是只是针对单一源属性进行验证， 复用性不高。

而且在组合验证（比如同时需要验证非空和其他规则）情况下，会导致 Model 中写过重、过臃肿的代码。

**2、ValidationRule 验证：**

通过继承 ValidationRule 抽象类，并重写他的 Validate 方法来扩展编写我们需要的验证类。该验证类可以直接使用在我们需要验证的属性上。

View 代码：

```xml
<GroupBox Header="ValidationRule 验证" Margin="10 20 10 10" DataContext="{Binding Source={StaticResource Locator},Path=ValidationRule}" >
    <StackPanel x:Name="ValidationRulePanel" Orientation="Vertical" Margin="0,20,0,0">
        <StackPanel>
            <Label Content="用户名" Target="{Binding ElementName=UserName}"/>
            <TextBox Width="150" >
                <TextBox.Text>
                    <Binding Path="UserName" UpdateSourceTrigger="PropertyChanged">
                        <Binding.ValidationRules>
                            <app:RequiredRule />
                        </Binding.ValidationRules>
                    </Binding>
                </TextBox.Text>
            </TextBox>
        </StackPanel>
        <StackPanel>
            <Label Content="用户邮箱" Target="{Binding ElementName=UserEmail}"/>
            <TextBox Width="150">
                <TextBox.Text>
                    <Binding Path="UserEmail" UpdateSourceTrigger="PropertyChanged">
                        <Binding.ValidationRules>
                            <app:EmailRule />
                        </Binding.ValidationRules>
                    </Binding>
                </TextBox.Text>
            </TextBox>
        </StackPanel>
    </StackPanel>
</GroupBox>
```

重写两个 ValidationRule，代码如下：

```csharp
public class RequiredRule : ValidationRule
{
    public override ValidationResult Validate(object value,
                                              CultureInfo cultureInfo)
    {
        if (value == null)
            return new ValidationResult(false, "该字段不能为空值！");
        if (string.IsNullOrEmpty(value.ToString()))
            return new ValidationResult(false, "该字段不能为空字符串！");
        return new ValidationResult(true, null);
    }
}

public class EmailRule : ValidationRule
{
    public override ValidationResult Validate(object value,
                                              CultureInfo cultureInfo)
    {
        Regex emailReg = new Regex("^\\s*([A-Za-z0-9_-]+(\\.\\w+)*@(\\w+\\.)+\\w{2,5})\\s*$");

        if (!String.IsNullOrEmpty(value.ToString()))
        {
            if (!emailReg.IsMatch(value.ToString()))
            {
                return new ValidationResult(false, "邮箱地址不准确！");
            }
        }
        return new ValidationResult(true, null);
    }
}
```

创建了两个类，一个用于验证是否为空，一个用于验证是否符合邮箱地址标准格式。

ViewModel 代码：

```csharp
public class ValidationRuleViewModel : ViewModelBase
{
    public ValidationRuleViewModel()
    {
    }

#region 属性
    private String userName;
    /// <summary>
    /// 用户名
    /// </summary>
    public String UserName
    {
        get { return userName; }
        set { userName = value; RaisePropertyChanged(() => UserName); }
    }

    private String userEmail;
    /// <summary>
    /// 用户邮件
    /// </summary>
    public String UserEmail
    {
        get { return userEmail; }
        set { userEmail = value; RaisePropertyChanged(() => UserName); }
    }
#endregion
```

结果如下：

![](https://images2015.cnblogs.com/blog/167509/201703/167509-20170314144517682-1017781868.png)

**说明：** 相对来说，这种方式是比较不错的，独立性、复用性都很好，从松散耦合角度来说也是比较恰当的。

可以预先写好一系列的验证规则类，View 视图编码人员可以根据需求直接使用这些验证规则，服务端无需额外的处理。

但是仍然有缺点，扩展性差，如果需要个性化反馈消息也需要额外扩展。不符合日益丰富的前端验证需求。

**3、IDataErrorInfo 验证:**

**3.1、在绑定数据源对象上实现 IDataErrorInfo 接口**

**3.2、在 Binding 对象上设置 ValidatesOnDataErrors 属性**

Binding 将调用从绑定数据源对象公开的 IDataErrorInfo API。如果从这些属性调用返回非 null 或非空字符串，则将为该 Binding 设置验证错误。

View 代码：

```xml
<GroupBox Header="IDataErrorInfo 验证" Margin="10 20 10 10" DataContext="{Binding Source={StaticResource Locator},Path=BindingForm}" >
    <StackPanel x:Name="Form" Orientation="Vertical" Margin="0,20,0,0">
        <StackPanel>
            <Label Content="用户名" Target="{Binding ElementName=UserName}"/>
            <TextBox Width="150" Text="{Binding UserName, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged, ValidatesOnDataErrors=True}" ></TextBox>
        </StackPanel>
        <StackPanel>
            <Label Content="性别" Target="{Binding ElementName=RadioGendeMale}"/>
            <RadioButton Content="男" />
            <RadioButton Content="女" Margin="8,0,0,0" />
        </StackPanel>
        <StackPanel>
            <Label Content="生日" Target="{Binding ElementName=DateBirth}" />
            <DatePicker x:Name="DateBirth" />
        </StackPanel>
        <StackPanel>
            <Label Content="用户邮箱" Target="{Binding ElementName=UserEmail}"/>
            <TextBox Width="150" Text="{Binding UserEmail, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged, ValidatesOnDataErrors=True}" />
        </StackPanel>
        <StackPanel>
            <Label Content="用户电话" Target="{Binding ElementName=UserPhone}"/>
            <TextBox Width="150" Text="{Binding UserPhone, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged, ValidatesOnDataErrors=True}" />
        </StackPanel>
    </StackPanel>
</GroupBox>
```

ViewModel 代码：

```csharp
public class BindingFormViewModel : ViewModelBase, IDataErrorInfo
{
    public BindingFormViewModel()
    {
    }

#region 属性
    private String userName;
    /// <summary>
    /// 用户名
    /// </summary>
    public String UserName
    {
        get { return userName; }
        set { userName = value; }
    }

    private String userPhone;
    /// <summary>
    /// 用户电话
    /// </summary>
    public String UserPhone
    {
        get { return userPhone; }
        set { userPhone = value; }
    }

    private String userEmail;
    /// <summary>
    /// 用户邮件
    /// </summary>
    public String UserEmail
    {
        get { return userEmail; }
        set { userEmail = value; }
    }
#endregion

    public String Error
    {
        get { return null; }
    }

    public String this[string columnName]
    {
        get
        {
            Regex digitalReg = new Regex(@"^[-]?[1-9]{8,11}\d*$|^[0]{1}$");
            Regex emailReg = new Regex("^\\s*([A-Za-z0-9_-]+(\\.\\w+)*@(\\w+\\.)+\\w{2,5})\\s*$");

            if (columnName == "UserName" &&
                String.IsNullOrEmpty(this.UserName))
            {
                return "用户名不能为空";
            }

            if (columnName == "UserPhone" &&
                !String.IsNullOrEmpty(this.UserPhone))
            {
                if (!digitalReg.IsMatch(this.UserPhone.ToString()))
                {
                    return "用户电话必须为8-11位的数值！";
                }
            }

            if (columnName == "UserEmail" &&
                !String.IsNullOrEmpty(this.UserEmail))
            {
                if (!emailReg.IsMatch(this.UserEmail.ToString()))
                {
                    return "用户邮箱地址不正确！";
                }
            }
            return null;
        }
    }
}
```

在继承 IDataErrorInfo 接口后，实现方法的两个属性：Error 属性用于指示整个对象的错误信息，而索引器用于指示单个属性级别的错误信息。

每次属性值发生变化后，索引器进行一次检查，查看是否有验证错误的信息返回。

其实，两者的工作原理相同，如果返回非 null 或非空字符串，则表示存在验证错误。否则，返回的字符串用于向用户显示错误。&#x20;

结果如图：

![](https://images2015.cnblogs.com/blog/167509/201703/167509-20170314153311166-679326127.png)

利用 IDataErrorInfo 的好处是它可用于轻松地处理交叉耦合属性。但也具有一个很大的弊端：索引器的实现通常会导致较大的 switch-case 语句（对象中的每个属性名称都对应于一种情况），必须基于字符串进行切换和匹配，并返回指示错误的字符串。而且，在对象上设置属性值之前，不会调用 IDataErrorInfo 的实现。

为了避免出现大量的 switch-case，并且将校验逻辑进行分离提高代码复用，将验证规则和验证信息独立化于每个模型对象中，使用 DataAnnotations 无疑是最好的的方案 ；所以，我们进行改良一下。

View 代码，跟上面那个一样：

```xml
<GroupBox Header="IDataErrorInfo+ 验证" Margin="10 20 10 10" DataContext="{Binding Source={StaticResource Locator},Path=BindDataAnnotations}" >
    <StackPanel Orientation="Vertical" Margin="0,20,0,0">
        <StackPanel>
            <Label Content="用户名" Target="{Binding ElementName=UserName}"/>
            <TextBox Width="150" Text="{Binding UserName,UpdateSourceTrigger=PropertyChanged,ValidatesOnDataErrors=True}" ></TextBox>
        </StackPanel>
        <StackPanel>
            <Label Content="性别" Target="{Binding ElementName=RadioGendeMale}"/>
            <RadioButton Content="男" />
            <RadioButton Content="女" Margin="8,0,0,0" />
        </StackPanel>
        <StackPanel>
            <Label Content="生日" Target="{Binding ElementName=DateBirth}" />
            <DatePicker />
        </StackPanel>
        <StackPanel>
            <Label Content="用户邮箱" Target="{Binding ElementName=UserEmail}"/>
            <TextBox Width="150" Text="{Binding UserEmail, UpdateSourceTrigger=PropertyChanged, ValidatesOnDataErrors=True}" />
        </StackPanel>
        <StackPanel>
            <Label Content="用户电话" Target="{Binding ElementName=UserPhone}"/>
            <TextBox Width="150" Text="{Binding UserPhone,UpdateSourceTrigger=PropertyChanged, ValidatesOnDataErrors=True}" />
        </StackPanel>
        <Button Content="提交" Margin="100,16,0,0" HorizontalAlignment="Left" Command="{Binding ValidFormCommand}" />
    </StackPanel>
</GroupBox>
```

VideModel代码：

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.ComponentModel;
using System.ComponentModel.DataAnnotations;
using System.Windows;
using GalaSoft.MvvmLight;
using GalaSoft.MvvmLight.Command;

namespace MVVMLightDemo.ViewModel
{
    [MetadataType(typeof(BindDataAnnotationsViewModel))]
    public class BindDataAnnotationsViewModel : ViewModelBase, IDataErrorInfo
    {
        public BindDataAnnotationsViewModel()
        {
        }

#region 属性 
        /// <summary>
        /// 表单验证错误集合
        /// </summary>
        private Dictionary<String, String> dataErrors = new Dictionary<String, String>();

        private String userName;
        /// <summary>
        /// 用户名
        /// </summary>
        [Required]
        public String UserName
        {
            get { return userName; }
            set { userName = value; }
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
            set { userPhone = value; }
        }

        private String userEmail;
        /// <summary>
        /// 用户邮件
        /// </summary>
        [Required]
        [StringLength(100, MinimumLength=2)]
        [RegularExpression("^\\s*([A-Za-z0-9_-]+(\\.\\w+)*@(\\w+\\.)+\\w{2,5})\\s*$", ErrorMessage = "请填写正确的邮箱地址.")]
        public String UserEmail
        {
            get { return userEmail; }
            set { userEmail = value; }
        }
#endregion

#region 命令
        private RelayCommand validFormCommand;
        /// <summary>
        /// 验证表单
        /// </summary>
        public RelayCommand ValidFormCommand
        {
            get
            {
                if (validFormCommand == null)
                    return new RelayCommand(() => ExcuteValidForm());
                return validFormCommand;
            }
            set { validFormCommand = value; }
        }

        /// <summary>
        /// 验证表单
        /// </summary>
        private void ExcuteValidForm()
        {
            if (dataErrors.Count == 0)
                MessageBox.Show("验证通过！");
            else
                MessageBox.Show("验证失败！");
        }
#endregion

        public string this[string columnName]
        {
            get
            {
                ValidationContext vc = new ValidationContext(this, null, null);
                vc.MemberName = columnName;
                var res = new List<ValidationResult>();
                var result = Validator.TryValidateProperty(this.GetType().GetProperty(columnName).GetValue(this, null), vc, res);
                if (res.Count > 0)
                {
                    AddDic(dataErrors,vc.MemberName);
                    return string.Join(Environment.NewLine, res.Select(r => r.ErrorMessage).ToArray());
                }
                RemoveDic(dataErrors,vc.MemberName);
                return null;
            }
        }

        public string Error
        {
            get { return null; }
        }

#region 附属方法
        /// <summary>
        /// 移除字典
        /// </summary>
        /// <param name="dics"></param>
        /// <param name="dicKey"></param>
        private void RemoveDic(Dictionary<String, String> dics, String dicKey)
        {
            dics.Remove(dicKey);
        }

        /// <summary>
        /// 添加字典
        /// </summary>
        /// <param name="dics"></param>
        /// <param name="dicKey"></param>
        private void AddDic(Dictionary<String, String> dics, String dicKey)
        {
            if (!dics.ContainsKey(dicKey)) dics.Add(dicKey, "");
        }
#endregion
    }
}
```

DataAnnotations 相信很多人很熟悉，可以使用数据批注来自定义用户的模型数据，记得引用 System.ComponentModel.DataAnnotations。

它包含如下几种验证类型：&#x20;

| **验证属性**                   | **说明**                  |
| -------------------------- | ----------------------- |
| CustomValidationAttribute  | 使用自定义方法进行验证。            |
| DataTypeAttribute          | 指定特定类型的数据，如电子邮件地址或电话号码。 |
| EnumDataTypeAttribute      | 确保值存在于枚举中。              |
| RangeAttribute             | 指定最小和最大约束。              |
| RegularExpressionAttribute | 使用正则表达式来确定有效的值。         |
| RequiredAttribute          | 指定必须提供一个值。              |
| StringLengthAttribute      | 指定最大和最小字符数。             |
| ValidationAttribute        | 用作验证属性的基类。              |

这边我们使用到了 RequiredAttribute、StringLengthAttribute、RegularExpressionAttribute 三项，如果有需要进一步了解 DataAnnotations 的，可以参考微软官网：[Using Data Annotations to Customize Data Classes | Microsoft Docs](https://msdn.microsoft.com/en-us/library/dd901590\(VS.95\).aspx)

在使用 DataAnnotions 之后，Model 变的更加简洁，校验也更加灵活，可以叠加组合验证 , 面对复杂验证模式的时候，也可以自由的使用正则来验证。

默认情况下，框架会提供相应需要反馈的消息内容，当然也可以自定义错误消息内容：ErrorMessage 。

这里，我们还加了个全局的错误集合收集器：dataErrors，在提交判断的时候判断是否验证通过。

这里，我们也进一步封装了索引器，并且通过反射技术读取当前字段下的属性进行验证。

结果如下：

![](https://images2015.cnblogs.com/blog/167509/201703/167509-20170314170431651-1779258892.png)

**封装 ValidateModelBase类：**

上面的验证比较合理了，不过相对于开发人员还是太累赘了，开发人员更关心的是 Model 的DataAnnotations 的配置，而不是关心在这个 ViewModel 要如何做验证处理，所以，我们需要进一步抽象。

编写一个 ValidateModelBase，把需要处理的工作都放在里面。需要验证属性的 Model 去继承这个基类。如下：

![](https://images2015.cnblogs.com/blog/167509/201703/167509-20170315222010088-945609008.png)

ValidateModelBase 类，请注意标红部分：

```csharp
public class ValidateModelBase : ObservableObject, IDataErrorInfo
{
    public ValidateModelBase()
    {
    }

#region 属性 
    /// <summary>
    /// 表当验证错误集合
    /// </summary>
    private Dictionary<String, String> dataErrors = new Dictionary<String, String>();

    /// <summary>
    /// 是否验证通过
    /// </summary>
    public Boolean IsValidated
    {
        get
        {
            if (dataErrors != null && dataErrors.Count > 0)
            {
                return false;
            }
            return true;
        }
    }
#endregion

    public string this[string columnName]
    {
        get
        {
            ValidationContext vc = new ValidationContext(this, null, null);
            vc.MemberName = columnName;
            var res = new List<ValidationResult>();
            var result = Validator.TryValidateProperty(this.GetType().GetProperty(columnName).GetValue(this, null), vc, res);
            if (res.Count > 0)
            {
                AddDic(dataErrors, vc.MemberName);
                return string.Join(Environment.NewLine, res.Select(r => r.ErrorMessage).ToArray());
            }
            RemoveDic(dataErrors, vc.MemberName);
            return null;
        }
    }

    public string Error
    {
        get { return null; }
    }

#region 附属方法
    /// <summary>
    /// 移除字典
    /// </summary>
    /// <param name="dics"></param>
    /// <param name="dicKey"></param>
    private void RemoveDic(Dictionary<String, String> dics, String dicKey)
    {
        dics.Remove(dicKey);
    }

    /// <summary>
    /// 添加字典
    /// </summary>
    /// <param name="dics"></param>
    /// <param name="dicKey"></param>
    private void AddDic(Dictionary<String, String> dics, String dicKey)
    {
        if (!dics.ContainsKey(dicKey)) dics.Add(dicKey, "");
    }
#endregion
}
```

验证的模型类，继承自 ValidateModelBase：

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
}
```

ViewModel 代码如下：

```csharp
public class PackagedValidateViewModel : ViewModelBase
{
    public PackagedValidateViewModel()
    {
        ValidateUI = new Model.ValidateUserInfo();
    }

#region 全局属性
    private ValidateUserInfo validateUI;
    /// <summary>
    /// 用户信息
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
    public RelayCommand SubmitCmd
    {
        get
        {
            if(submitCmd == null)
                return new RelayCommand(() => ExcuteValidForm());
            return submitCmd;
        }
        set { submitCmd = value; }
    }
#endregion

#region 附属方法
    /// <summary>
    /// 验证表单
    /// </summary>
    private void ExcuteValidForm()
    {
        if (ValidateUI.IsValidated)
            MessageBox.Show("验证通过！");
        else
            MessageBox.Show("验证失败！");
    }
#endregion
}
```

结果如下：

![](https://images2015.cnblogs.com/blog/167509/201703/167509-20170315221357198-1295761813.png)

