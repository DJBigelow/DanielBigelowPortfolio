---
layout: single
title:  "Model View Viewmodel Architecture"
date:   2021-11-22 18:40:03 -0600
categories: mvvm prism c# tdd unit-testing
---

## What is MVVM?

MVVM, short for Model--View--Viewmodel, is a design pattern focused on separating business logic from presentation, allowing for loose coupling of components.
In MVVM, there are three components, the model, the view, and the viewmodel.
The model component is simply data and contains no logic.
The view is what's presented to the user. It recieves user inputs and forwards them to the viewmodel
The viewmodel contains the business logic of the app. It responds to events triggered by user input in the view and acts upon the model accordingly.

## Why MVVM?  

As was mentioned mentioned earlier, MVVM allows for loose coupling of components, which has several benefits. One is that the view and viewmodel can be developed simultaneously by multiple developers because of their loose coupling. Another is that the viewmodel can be easily tested because a GUI isn't required to enact any of its logic. MVVM also allows for easy reuse components and gives you the flexibility of changing your UI without having to refactor the code behind it.

## MVVM Frameworks and Prism

You can build your own MVVM architecture from scratch, but often times there's no need unless you have some special needs that can't be met by an MVVM framework.
Using a framework to get started allows you to focus less on the low-level details of MVVM, making it much quicker to get something up and running. There are many MVVM frameworks out there, but for the .NET platform alone we have frameworks such as Prism, MVVM Light Toolkit, Catel, and ClientUI, just to name a few. Most of my experience with MVVM comes from using Prism, so that's what I'll be showcasing later.
Prism is open-source framework with support from Microsoft. It contains support for both WPF and Xamarin Forms apps and supports some useful features such as dependency injection, commands, and Event Aggregator.

## Demonstration

The quickest way to get started is by using the Prism Template Pack extension for Visual Studio. If you want to follow along, go ahead and add that to your VS installation, then create a new project. When you search for Prism in your project templates, you should find a *Prism Blank App (WPF)* template. Go ahead and use that one, then name your project whatever you like. When your project is up, you should have a solution explorer containing ViewModels and Views folders. Let's first take a look at the MainWindowViewModel.cs file inside the ViewModels folder:

{% highlight csharp %}
using Prism.Mvvm;

namespace PrismCalculator.ViewModels
{
    public class MainWindowViewModel : BindableBase
    {
        private string _title = "Prism Application";
        public string Title
        {
            get { return _title; }
            set { SetProperty(ref _title, value); }
        }

        public MainWindowViewModel()
        {

        }
    }
}
{% endhighlight %}

Take notice of the Title property in the MainViewModel class. It'll come into play very soon. Now let's look at the MainWindow.xaml file in the Views folder:

{% highlight xml %}
<Window x:Class="PrismCalculator.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:prism="http://prismlibrary.com/"
        prism:ViewModelLocator.AutoWireViewModel="True"
        Title="{Binding Title}" Height="350" Width="525">
    <Grid>
        <ContentControl prism:RegionManager.RegionName="ContentRegion" />
    </Grid>
</Window>
{% endhighlight %}


Now take notice of these lines in particular:

{% highlight xaml %}
prism:ViewModelLocator.AutoWireViewModel="True"
Title="{Binding Title}" Height="350" Width="525">
{% endhighlight %}

The first of these lines tells Prism to automatically wire up this view to a viewmodel that follows a certain naming convention. Since this view is named MainWindow, Prism will search our ViewModels folder for any viewmodels with the same name but with ViewModel appended to the end.
The second line tells prism to bind the title of the view's window to any properties named *Title* in our viewmodel. When we fire up our app, we can see this in action:

![PrismAppFirstScreen]({{site.baseurl}}/assets/img/mvvm/PrismAppFirstScreen.png)

We can change the title of the window by going into our viewmodel and changing the value assigned to the title field, like this:

{% highlight csharp %}
.
.
.
private string _title = "Hello World";
public string Title
{
    get { return _title; }
    set { SetProperty(ref _title, value); }
}
.
.
.
{% endhighlight %}

When we run the app again, we can see that our changes have resulted in a new title for the window:

![Hello World Window]({{site.baseurl}}/assets/img/mvvm/HelloWorldWindow.png)

As you can see, data binding is fairly easy with Prism. We'll see more of data binding later, but first I want to do some setup.
To see the automatic wiring of views and viewmodels done by Prism, go ahead and create a new class named CalculatorViewModel in the ViewModels folder and copy and paste this code inside of it:

{% highlight csharp %}
using Prism.Mvvm;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace PrismCalculator.ViewModels
{
    public class CalculatorViewModel : BindableBase
    {
        private string text = "Hello World";
        public string Text
        {
            get { return text; }
            set { SetProperty(ref text, value); }
        }
    }
}
{% endhighlight %}


Now add a new *Window (WPF)* item named Calculator to your views folder, then copy and paste this XAML inside of it:

{% highlight xml %}
<Window x:Class="PrismCalculator.Views.Calculator"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:prism="http://prismlibrary.com/"
        prism:ViewModelLocator.AutoWireViewModel="True">
    <Grid>
        <TextBlock VerticalAlignment="Center" HorizontalAlignment="Center" Text="{Binding Text}"></TextBlock>
    </Grid>
</Window>
{% endhighlight %}

To launch the app with this view, we need to go inside the code-behind in App.xaml in the root of the project and change the CreateShell() method to this:

{% highlight csharp %}
protected override Window CreateShell()
{
    return Container.Resolve<Calculator>();
}
{% endhighlight %}


Now go ahead and launch the app. You should get the following window:

![New View]({{site.baseurl}}/assets/img/mvvm/NewView.png)

As you can see, wiring up views and viewmodels is as easy as following the correct naming convention, placing your views and viewmodels inside the right folders, and including these lines in your view's xaml:

{% highlight xml %}
xmlns:prism="http://prismlibrary.com/"
prism:ViewModelLocator.AutoWireViewModel="True"
{% endhighlight %}


## Dependency Injection

Depency injection isn't the main focus of this article, so we'll only be covering it briefly, just long enough to show how it can be done in Prism. Dependency injection is a pattern that separates the responsibility of creating an object from the class that depends on it. This allows you to provide an instance of a class implementing an interface to a class without that class having to worry about which implementation of the interface it just recieved. This allows for loose coupling between dependencies and their dependents.
To set up dependency injection in Prism, we first need a dependency. I've gone ahead and created a Services folder in the root of the project and added the following interface and implementation:

{% highlight csharp %}
public interface ICalculator
{
    double Calculate(double argument1, double argument2);
}

...

public class AdditionCalculator : ICalculator
{
    public double Calculate(double argument1, double argument2)
    {
        return argument1 + argument2;
    }
}
{% endhighlight %}


Of course, I wrote tests for the AdditionCalculator before I implemented it. I'll be showing them off soon enough, along with more tests soon.
Now let's go into CalculatorViewModel and add the dependency:

{% highlight csharp %}
public class CalculatorViewModel : BindableBase
{
    private string text = "Hello World";
    private readonly ICalculator calculator;

    public string Text
    {
        get { return text; }
        set { SetProperty(ref text, value); }
    }

    public CalculatorViewModel(ICalculator calculator)
    {
        this.calculator = calculator;
    }
}
{% endhighlight %}

To register a specific implementation of the ICalculator for dependency injection, we need to go into the code-behind of App.xaml again and add the following line to *RegisterTypes()*:

{% highlight csharp %}
protected override void RegisterTypes(IContainerRegistry containerRegistry)
{
    containerRegistry.Register<ICalculator, AdditionCalculator>();
}
{% endhighlight %}


Now we can run the project again. The lack of thrown exceptions is proof enough that we successfully registered a type for dependency injection.

## Defining the Viewmodel's Behavior Through Tesing

As I mentioned ealrier, MVVM allows you to easily test the viewmodel becuase it's decoupled from the view. I'll be doing Test-Driven Development for this project, but I won't show the process step-by-step. If you want to see a project where I do so, click [here]({{site.baseurl}} {% post_url 2021-10-04-test-driven-development%}).
In my CalculatorViewModel, I want to ensure that the user can't make a calculation if either of the two input arguments are less than 0. This means that the command that executes the calculation shouldn't be able to fire if either input argument is invalid, which is something I tested for:

{% highlight csharp %}
using NUnit.Framework;
using PrismCalculator.ViewModels;
using System;
using System.Collections.Generic;
using System.Text;
using Moq;
using PrismCalculator.Services;

namespace PrismCalculator.Test
{
    public class CalculatorViewModelTests
    {
        private CalculatorViewModel vm;
        private Mock<ICalculator> mockCalculator;

        private readonly double calculateReturnValue = 1.0;

        [SetUp]
        public void Setup()
        {
            mockCalculator = new Mock<ICalculator>();
            vm = new CalculatorViewModel(mockCalculator.Object);
        }

        [TestCase(-1.0, -1.0)]
        [TestCase(-1.0, 1.0)]
        [TestCase(1.0, -1.0)]
        public void CantCalculateWithNegativeNumbers(double arg1, double arg2)
        {
            vm.Argument1 = arg1;
            vm.Argument2 = arg2;

            Assert.IsFalse(vm.CalculateCommand.CanExecute());
        }

        [TestCase(1.0, 0.0)]
        [TestCase(0.0, 0.0)]
        [TestCase(12345.0, 11111.0)]
        public void CanCalculateWithPositiveNumbers(double arg1, double arg2)
        {
            vm.Argument1 = arg1;
            vm.Argument2 = arg2;

            Assert.IsTrue(vm.CalculateCommand.CanExecute());
        }
    }
}
{% endhighlight %}

With tests in place, we can create the command that executes the calculation:

{% highlight csharp %}
public DelegateCommand CalculateCommand { get; private set; }

private bool canCalculate()
{
    return argument1 >= 0 && argument2 >= 0;
}

private void calculate()
{
    
}

public CalculatorViewModel(ICalculator calculator)
{
    this.calculator = calculator;

    CalculateCommand = new DelegateCommand(calculate, canCalculate);
}
{% endhighlight %}

The DelegateCommand class is provided by Prism. There are several ways to use it, but the way I'm using it is one of the simpler ways. When instantiating the DelegateCommand, we pass it a method to run when the command is executed and a method that the command first runs to tell if it can be executed or not. With that done, we get passing tests, which means it's time to start implementing more stuff.
At this point we can write tests where our command actually executes, so let's do that:

{% highlight csharp %}
[Test]
public void CalculatorCalculateCalledOnceOnCommandExecute()
{
    vm.CalculateCommand.Execute();

    mockCalculator.Verify(mock => mock.Calculate(0.0, 0.0), Times.Once());
}

[Test]
public void ResultIsUpdatedOnCalculate()
{
    vm.Result = 0.0;

    vm.CalculateCommand.Execute();

    Assert.AreEqual(calculateReturnValue, vm.Result);
}
{% endhighlight %}


Going back to our Viewmodel, we can now implement the logic behind calculate():

{% highlight csharp %}
private double result;
public double Result
{
    get { return result; }
    set { SetProperty(ref result, value); }
}

...

private void calculate()
{
    Result = calculator.Calculate(Argument1, Argument2);
}
{% endhighlight %}


Our tests are passing and the basic logic behind our viewmodel is complete, so now let's focus on the view. Wiring our properties up is very simple:

{% highlight xml %}
<Window x:Class="PrismCalculator.Views.Calculator"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:prism="http://prismlibrary.com/"
        prism:ViewModelLocator.AutoWireViewModel="True"
        SizeToContent="WidthAndHeight">
    <Grid>
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="Auto" />
            <ColumnDefinition Width="Auto" MinWidth="75" />
        </Grid.ColumnDefinitions>
        <Grid.RowDefinitions>
            <RowDefinition Height="20"/>
            <RowDefinition Height="20"/>
            <RowDefinition Height="20"/>
            <RowDefinition Height="20"/>
        </Grid.RowDefinitions>

        <TextBlock Text="Argument 1: " Grid.Row="0" Grid.Column="0"/>
        <TextBlock Text="Argument 2: " Grid.Row="1" Grid.Column="0"/>
        <TextBlock Text="Result: " Grid.Row="3" Grid.Column="0"/>

        <TextBox Text="{Binding Argument1}" Grid.Row="0" Grid.Column="2"/>
        <TextBox Text="{Binding Argument2}" Grid.Row="1" Grid.Column="2"/>
        <Button Command="{Binding CalculateCommand}" Grid.Row="2" Grid.ColumnSpan="2" Grid.Column="0">Calculate</Button>
        <TextBlock Text="{Binding Result}" Grid.Row="3" Grid.Column="2"/>
    </Grid>
</Window>
{% endhighlight %}


We can now run our app and use the calculator. You can see yourself that it doesn't do anything when either argument is set to a negative value, just as intended.

![Calculator]({{site.baseurl}}/assets/img/mvvm/Calculator.png)
