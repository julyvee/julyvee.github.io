---
layout: post
title:  "4 Dos And Don'ts For Efficient Unit Testing"
date:   2022-08-17 15:00:00 +0200
img: 1.jpg
tags: [development,testing]
description: "Adhering to these principles will save you time while writing your tests and make them more meaningful."
---
Unit testing is an important part of the software development lifecycle since it ensures that your code works as expected and that future changes don’t break existing features. When writing unit tests, we want to write as many as necessary to cover all the important logic in the code but at the same time write as little as possible to minimize dependencies and keep the tests maintainable. To achieve efficient unit testing, here are some Dos and Don’ts to consider.

# 1. Mocking

In a unit test, we want to test the logic inside the class or method we are testing. We don’t want to test logic that is part of other classes or third-party libraries. To achieve this, we can mock any objects or functions that contain logic outside our class or method. This is more efficient as you can spend less time worrying about the logic outside of your testing scope.

## Don’t

Don’t call on any real objects or third-party libraries outside of your testing scope. This would implicitly test their logic as well as yours and make your unit test less reliable. You’ll be dependent on the functionality of the other class and changes you have no control over might break your tests even if the logic you wrote is working well. 
This is especially true for objects that will connect to other systems, such as databases, APIs, etc. If you use real objects and connect to these systems during your tests, you will be dependent on their availability. They will also most likely slow down your test execution and you’ll have less control over the data these systems will return.

## Do

Instead, use mocks to inject a fake object which takes the place of the real one. On the mock, you can control what kind of data is returned for which method call and you are not dependent on the internal logic of other classes or any system availability. For example:

{% highlight cs %}
var mockObject = new Mock<ApiClient>();
mockObject.Setup(x => x.getName()).Returns("Will Smith");
{% endhighlight %}

Instead of a mock, it is also possible to use a fake, which is an object with a working implementation that takes shortcuts for testing purposes, like having hard coded return values. In this example, a fake api client wouldn't actually call the api and instead always return "Will Smith". Setting up a fake takes more time, but doesn't require a framework like mocks do. Depending on your exact testing scenario a fake might be more suitable.

# 2. Dependency Injection

If you don’t keep testability in mind from the beginning, once you start writing your tests, you might realize you have to do a time-intensive refactor to make the code unit testable. A common problem that leads to non-testable code is not using dependency injection. To avoid this, remember the following while you’re writing your code.

## Don’t

Don’t use `new`-statements for objects you want to mock later such as clients, configurations, or other services your class uses. Example:

{% highlight cs %}
var apiClient = new ApiClient(apiBaseUrl);
var name = apiClient.getCompanyName();
{% endhighlight %}

This code is not unit testable because you have no way of injecting a mock that can be used instead of the api client. 

## Do

Use dependency injection so that you can inject a mock instead of a real object when testing. In this case, we’re going to inject mock api client.

{% highlight cs %}
public SomeClass(ApiClient apiClient) {
   this.apiClient = apiClient;
}
{% endhighlight %}        

The code from the above example now looks like this:

{% highlight cs %}
var name = this.apiClient.getCompanyName();
{% endhighlight %}

This is unit testable since the api client can be injected as a mock and can then be configured to return a pre-defined value.

{% highlight cs %}
mockApiClient = new Mock<ApiClient>();

someObject= new SomeClass(mockApiClient.Object);

mockApiClient.Setup(x => x.getCompanyName()).Returns("My Test Company");
{% endhighlight %}

Now you can easily set up the mock api client to return predefined test data and continue testing your class logic without connecting to a real api. If you keep this in mind from the beginning, you’ll write great unit testable code and can start testing immediately without spending much time on refactoring. Very efficient!

# 3. Assertions

When it comes to assertions in unit tests you want to make sure that you assert the right things, not necessarily lots of things. While we are trying to test the logic inside a certain class, some assertions can be inefficient and not give you the confidence you need in the test result. Consider the following:

## Don’t

Don’t assert the return values of mocks. Because if you do, you’re mainly asserting whether you set up the mock correctly. This is especially true when the method you are testing passes the mock result directly as a return value without significant changes. For a very simple example, look at this class:

{% highlight cs %}

public class SearchController : ControllerBase {

   public SearchClient SearchClient { get; }

   public SearchController(SearchClient searchClient)
   {
      SearchClient = searchClient;
   }

   public String GetName(string id)
   {
      return this.SearchClient.GetName(id);
   }
}
{% endhighlight %}

After mocking the search client, we can set it up to return a certain value. Then, it’s easy to assert that the return value is, in fact, this value from the mock. 

{% highlight cs %}
mockSearchClient.Setup(x => x.GetName(id)
   .ReturnsAsync("myResult");
var result = searchController.GetName(id);
Assert.Equal("myResult",result.Value);
{% endhighlight %}

But now, your method could look like this, and the test would still pass:

{% highlight cs %}
public String GetName(string id)
{
   return "myResult";
}
{% endhighlight %}

Similarly, if you set up your mock wrong, the test would fail even though the logic inside the method is sound. 

## Do

For efficient assertions that will give you confidence in the logic, you are testing, make them only on the logic in the method you are testing. The simple example above doesn’t have a lot of logic, but we want to make sure that it calls the search client to retrieve the result. For this, we can use the verify method to make sure the search client was called using the right parameters even though we don’t care about the result.

{% highlight cs %}
mockSearchClient.Verify(mock => mock.GetName(id), Times.Once());
{% endhighlight %}

If there is more logic, you can make assertions on the part of the result that was modified by the method you are testing.

# 4. Callbacks

It can be time-consuming to set up mocks if you want to make sure they are being called with the right parameters, especially if the parameters are complex. To make your testing more efficient, consider using callbacks to make assertions on the parameters after a method was called.

## Don’t

Don’t spend a lot of time setting up a complex mock method call. Often you don’t care about all the parameters but only a few, or even only parts of them if the parameters are also objects. It’s easy to make a small mistake in the creation of the parameter, like missing an attribute that the actual method sets, and then your mock won’t be called, even though you might not care about this attribute at all. 

## Do

Define only the most relevant parameters to differentiate between method calls and use an `any`-statement for the others. In this example, the method has a complex search options parameter which would take a lot of time to set up manually. Since we only care about 2 attributes in the search options, we use an `any`-statement and store the options in a callback for later assertions.

{% highlight cs %}
var actualOptions = new SearchOptions();

mockSearchClient
   .Setup(x => 
      x.Search(
         myQuery, 
         It.IsAny<SearchOptions>()
      ) 
   )
   .Returns(mockResults)
   .Callback<string, SearchOptions>((query, searchOptions) =>
     {
       actualOptions = searchOptions;
     }
   );
{% endhighlight %}

Since we want to test our method logic, we care only about the parts of the parameter which are influenced by our method, in this example, let's say the search mode and the search query type. So, with the variable we stored in the callback, we can make assertions on only these two attributes.

{% highlight cs %}
Assert.Equal(SearchMode.All, actualOptions.SearchMode);
Assert.Equal(SearchQueryType.Full, actualOptions.QueryType);
{% endhighlight %}
 
This makes the test more explicit since it shows which parts of the logic you care about. It’s also more efficient since you don’t have to spend a lot of time setting up the parameters for the mock.

# Conclusion

To make your unit tests more efficient, keep a few principles in mind:

1.	Mock any logic outside of your testing scope
2.	Use dependency injection to make objects easily mockable
3.	Make assertions on your logic, not mock setups
4.	Use callbacks to validate complex parameters

Of course there might be scenarios in which some of these concepts don't apply. But in general, adhering to these principles will save you time while writing your tests and make them more meaningful. An all-around win! 

# Supporting Information

- The code in this blog post was written in C# (.NET Framework) using Xunit and the Moq framework. However, the concepts also apply to any other testing frameworks for object-oriented programming languages.
- [Click here for more Information on Dependency Injection](https://en.wikipedia.org/wiki/Dependency_injection)
- [Click here for more Information on Dependency Injection in .NET](https://docs.microsoft.com/en-us/dotnet/core/extensions/dependency-injection)
- [Click here for more Information on Mocking](https://microsoft.github.io/code-with-engineering-playbook/automated-testing/unit-testing/mocking/)