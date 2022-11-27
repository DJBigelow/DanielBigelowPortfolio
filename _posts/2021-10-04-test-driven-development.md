---
layout: single
title:  "Test Driven Development"
date:   2021-10-04 18:40:03 -0600
categories: unit-testing tdd c# dotnet
---

## What is Test-Driven Development (TDD)?

Test-driven development is a programming technique where tests for your code are written before the code itself. Once a test is written, it's run and verified that it fails, then the smallest amount of code to make that test is written. This process is repeated over and over until a complete program is put together. If refactoring needs to be done, which is likely, then the tests are run for each small change made to the code.

## Why use TDD?

Coding without writing tests first is like placing the cart before the horse, or making a building without any blueprints. You're trying to meet your project's requirements without first properly defining them. Test-driven development isn't a way to verify your program's functionality, it's a way to think through what your program needs to do before you write it. TDD is also helpful in making sure that breaking changes made by refactors don't go unnoticed.

## Demonstration

To showcase the process of test-driven development, we'll be taking a problem from [Project Euler](https://projecteuler.net/), specifically [problem one](https://projecteuler.net/problem=1). We'll be writing a program that finds the sum of all multiples of 3 and 5 below 1000.

### Setup

In Visual Studio, create a new C# .NET Core console project in a new solution. Then go to your extensions and install the NUnit 3 Test Adapter extension if you don't already have it. Now add a new NUnit .NET Core test project to your solution. If you've done everything right, you should have a solution explorer view that looks like this: 

![Starting Solution Explorer](https://i.imgur.com/8SfcDGb.png)

The last step is to go to your test project and add a project reference to your console project.

## The First Test

Our first order of business when following TDD principles is to consider something our program should be able to do and write a test for it. You may think at this point that we should write a test that shows our program can calculate the sum of every multiple of 3 and 5 below 1000. We'll get to that eventually, but right now there are smaller things are program should have to do in order to come to the final solution, such as determine if a number is a multiple of three or five. Let's start there by writing a test for a method that determines if an input number is a multiple of three or not.
Open up the UnitTest1.cs file in the test project. You should notice two methods that look like this:

{% highlight csharp %}
using NUnit.Framework;

namespace Euler1.Test
{
    public class Tests
    {
        [SetUp]
        public void Setup()
        {
        }

        [Test]
        public void Test1()
        {
            Assert.Pass();
        }
    }
} 
{% endhighlight %}

Right now, we're interested in Test1. Go ahead and rename it something more descriptive. Here's what I named mine:

{% highlight csharp %}
[Test]
public void IsMultipleOfThreeReturnsTrueIfGivenNumberIsMultipleOfThree()
{
    Assert.Pass();
}
{% endhighlight %}


You may be thinking that my test name is absurdly long. It may seem a bit overkill in this instance, but its best practice to make your test names as descriptive as possible as that's the easiest way to remind yourself and tell others what the test is for.
Now go into your console project and add an *IsMultipleOfThree* method to your Program.cs file, like this:

{% highlight csharp %}
public static bool IsMultipleOfThree(int number)
{
    throw new NotImplementedException();
}
{% endhighlight %}


We don't want to write our code before our tests, so for the time being our method will throw a new NotImplementedException when called. 
Go back into the test project, and we can go ahead and write our first tests:

{% highlight csharp %}
[Test]
public void IsMultipleOfThreeReturnsTrueIfGivenNumberIsMultipleOfThree()
{
    Assert.True(Program.IsMultipleOfThree(3));
}

public void IsMultipleOfThreeReturnsFalseIfGivenNumberIsNotMultipleOfThree()
{
    Assert.False(Program.IsMultipleOfThree(2));
}
{% endhighlight %}


A little simplistic, isn't it? We'll build on it in the future, but for now let's run the tests, make sure that they fail (it would actually be concerning at this point if they somehow did), and then write the smallest amount of code to make the tests pass.

![First Failing Test](https://i.imgur.com/UoiuR4x.png)

Our tests did indeed fail, so new let's go to the console project and implement the logic to make them pass:

{% highlight csharp %}
public static bool IsMultipleOfThree(int number)
{
    if (number == 3)
    {
        return true;
    }
    else
    {
        return false;
    }
}
{% endhighlight %}


You should immediately notice that this solution will fail if we pass in a multiple of three that isn't three itself. That's fine. In Test-Driven Development, part of the process is building a solution in as small of increments as possible. It may seem silly in this context, but in a more complex project, it can be very beneficial to take things one step at a time like we're doing here. Rushing head-first into a problem rarely doesn't have problematic consequences.
Let's run our tests and make sure they pass:

![First Passing Test](https://i.imgur.com/dxFqmt8.png)

And indeed they passed. Our next step is to repeat the process: write a failing test and then implement the smallest amount of logic to make it pass.
We don't want to leave our current, overly-simplistic solution as is, so let's write a test for it that makes it fail:
{% highlight csharp %}
[TestCase(3)]
[TestCase(6)]
[TestCase(9)]
[TestCase(12)]
[TestCase(30)]
[TestCase(75)]
[TestCase(99)]
[TestCase(300)]
public void IsMultipleOfThreeReturnsTrueIfGivenNumberIsMultipleOfThree(int number)
{
    Assert.True(Program.IsMultipleOfThree(number));
}

[TestCase(1)]
[TestCase(2)]
[TestCase(5)]
[TestCase(13)]
[TestCase(16)]
[TestCase(29)]
[TestCase(103)]
[TestCase(301)]
public void IsMultipleOfThreeReturnsFalseIfGivenNumberIsNotMultipleOfThree(int number)
{
    Assert.False(Program.IsMultipleOfThree(number));
}
{% endhighlight %}


I've actually modified our tests to handle multiple test cases, which technically gives us several new failing tests. Let's run them:

![Failing Test Cases](https://i.imgur.com/vDdE4zq.png)

At this point, we could simply expand on our current solution and simply hard-code it to return true for every given number in our test cases, but that wouldn't be following TDD principles. We want to write the least amount of logic possible to make our tests pass, and the previous suggested strategy certainly wouldn't be the simplest. Here's the solution that I went with:

{% highlight csharp %}
public static bool IsMultipleOfThree(int number)
{
    if (number % 3 == 0)
    {
        return true;
    }
    return false;
}
{% endhighlight %}


Isn't that much better? Now let's run our tests and see if what we did worked:

![Passing Test Cases](https://i.imgur.com/Y7DRGUD.png)

That's what we like to see. The next step is to do the same but for multiples of five. The process is largely the same, so If you're coding along, I'll leave that to you as an exercise. Let's fast-forward to the test for our final solution to the problem:

{% highlight csharp %}
[TestCase(3, 3)]
[TestCase(5, 8)]
[TestCase(10, 33)]
[TestCase(15, 60)]
[TestCase(999, 233168)]
public void CalculatesSumOfMultiplesOfThreeAndFiveUpToLimit(int limit, int expected)
{
    var actual = Program.CalcSumOfMultiplesOfFiveAndThree(limit);
    Assert.AreEqual(expected, actual);
}
{% endhighlight %}


And now the solution itself:

{% highlight csharp %}
public static int CalcSumOfMultiplesOfFiveAndThree(int limit)
{
    int sum = 0;

    for (int i = 0; i <= limit; ++i)
    {
        if (IsMultipleOfThree(i) && IsMultipleOfFive(i))
        {
            sum += i;
        }
    }

    return sum;
}
{% endhighlight %}


![Failing Solution Test](https://i.imgur.com/7VB6tPL.png)

Uh oh. It looks like we made a mistake somewhere. Luckily, we had tests in place to help us catch the error. Upon inspection of my code, it looks like I'm only summing numbers that are both multiples of three *and* five, not three *or* five. Let's fix the problem and run our tests again.

{% highlight csharp %}
public static int CalcSumOfMultiplesOfFiveAndThree(int limit)
{
    int sum = 0;

    for (int i = 0; i <= limit; ++i)
    {
        //                        |
        // Problem was right here v
        if (IsMultipleOfThree(i) || IsMultipleOfFive(i))
        {
            sum += i;
        }
    }

    return sum;
}
{% endhighlight %}


![Passing Solution](https://i.imgur.com/sdf2N6L.png)

Isn't green such a lovely color? Testing really made our lives easier there. Instead of having to bust out a calculator to verify whether our fix worked or not like we would have if we weren't testing, we simply had to run our tests again after making changes.
All that's left is to log the output of CalcSumOfMultiplesOfFiveAndThree() to manually verify the results ourselves:

{% highlight csharp %}
static void Main(string[] args)
{
    Console.WriteLine("Answer: " + CalcSumOfMultiplesOfFiveAndThree(999));
}
{% endhighlight %}


![Console Output](https://i.imgur.com/sdf2N6L.png)

Indeed, the answer should be 233168, meaning we've solved the problem. Hopefully you've seen how Test-Driven Development gives you a structured way to think through problems and can see how on more complex projects it reduces complexity. If not, give TDD a try in your own projects, and I guarantee you'll see it then.

