---
title:  "Unit Testing Static Methods In Swift"
date:   2020-05-28 12:24:03 -0600
categories: swift 
tags: swift json decodable
header:
   overlay_image: /assets/images/statictestsplash.jpg
   overlay_filter: 0.5
---

Understanding How To Unit Test Static Methods Written In Swift Or Objective-C

One of the challenges I've run into as an Objective-C dev transitioning to Swift is the additional difficulty of unit testing in Swift.  Objective-C is a highly dynamic language, which makes it easy to mock, stub and spy on just about anything.  Swift -- not so much.  The easy one-line doubles suddenly become manual mocks.  This is especially true for singletons and static methods.  Many times, our Swift code will need to use third-party libraries that leverage singletons and static  methods, or even access them from our legacy codebase.  Fortunately, these are still testable, with a few caveats.  

To begin, we will start with an admittedly contrived example (Available on GitHub: [Static Methods on Github][github]).  The UI features a label and a button.  When you press the button, a manager class returns a string that welcomes the user as an American or a Briton based on where they are located and the local time.  The manager class itself is not really necessary.  What it is doing could be done more easily directly in the ViewController, but separating it will let us focus on the unit test of static methods without UI unit testing getting in the way.

### Setting Up An Example To Test

The pieces of this system are as follows:

**ObjectiveCStaticUtils** - Objective-C class that contains a static method which tells us whether this is the US or UK version of the app and returns the result as an NSString.

{% highlight objectivec %}
+ (NSString *)getAppVersionType
{
    return @"US";
    
//     return @"UK";
}
{% endhighlight %}

**SwiftStaticUtils** - Swift class that has static methods which return the welcome message for each version type, customized for whether the current time is AM or PM as a Swift String.  

{% highlight swift %}
class SwiftStaticUtils {
    static func getUSMessage() -> String {
        guard let hour = SwiftStaticUtils.getHour() else {
            return "Welcome American"
        }
        
        if hour <= 12 {
            return "Welcome American AM"
        } else {
            return "Welcome American PM"
        }
        
    }
    
    static func getUKMessage() -> String {
        guard let hour = SwiftStaticUtils.getHour() else {
            return "Welcome Briton"
        }
        
        if hour <= 12 {
            return "Welcome Briton AM"
        } else {
            return "Welcome Briton PM"
        }
    }
    
    static func getHour() -> Int? {
        let date = Date()
        let dateComponents = Calendar.current.dateComponents([.hour], from: date)
        return dateComponents.hour
    }
}
{% endhighlight %}

**MessageManager** - Swift class that has an instance method that returns the message as directed by the two utility classes.

{% highlight swift %}
class MessageManager {
    func getMessage() -> String {
        if ObjectiveCStaticUtils.getAppVersionType() == "US" {
            return SwiftStaticUtils.getUSMessage()
        } else if ObjectiveCStaticUtils.getAppVersionType() == "UK" {
            return SwiftStaticUtils.getUKMessage()
        }
        
        return "Welcome Mystery User"
    }
}
{% endhighlight %}

The ViewController creates an instance of MessageManager and calls getMessage() on the button tap handler to get the string for the label.  Simple enough.  We will assume that ObjectiveCStaticUtils and SwiftStaticUtils are a library we don't own and both have previously been unit tested.

### Setting Up A Prototype For The Tests

To begin, we setup a new unit test case class for MessageManager, called MessageManagerTests.swift.  We can add one prototype test and be sure that the setup works.  This test should pass.

{% highlight swift %}
import XCTest
@testable import StaticMethods

class MessageManagerTests: XCTestCase {
    func test_getMessage() {
        XCTAssert(true)
    }
}
{% endhighlight %}

Now, if we look at the getMessage method, there are three cases we need to test for:

1. The app version type is US
2. The app version type is UK
3. The app version type is something else

Since we made the assumption that the other methods are unit tested, we know we need three tests.  We will have to control the output from getAppVersionType, getUSMessage, and getUKMessage in order to write reliable tests. 

{% highlight swift %}
class MessageManagerTests: XCTestCase {
    func test_getMessage_us() {
        XCTAssert(true)
    }
    
    func test_getMessage_uk() {
        XCTAssert(true)
    }
    
    func test_getMessage_somethingElse() {
        XCTAssert(true)
    }
}
{% endhighlight %}

### Faking The Swift Static Methods

Let's start inside out, and control the output of the static methods written in Swift.  "Dependency injection" can be used to allow us to replace the static methods the MessageManager uses with custom ones specific for our test.  Dependency injection simply means to write your code in a way that you can change (or "inject") your dependencies.  In order to do this, we will need to inject SwiftStaticUtils as a type (as opposed to an instance of some type, because SwiftStaticUtils is just a class with static methods).  Swift provides a special [MetaType type][swift-metatype] that can help us achieve this.  The metatype can represent any class, enumeration or structure type.  In our case, the metatype for SwiftStaticUtils is SwiftStaticUtils.Type.  Given this, we now have a hook we can use to inject SwiftStaticUtils as a type, so we can control our class methods.

{% highlight swift %}
class MessageManager {
    var swiftStaticUtils:SwiftStaticUtils.Type = SwiftStaticUtils.self
    
    func getMessage() -> String {
        if ObjectiveCStaticUtils.getAppVersionType() == "US" {
            return swiftStaticUtils.getUSMessage()
        } else if ObjectiveCStaticUtils.getAppVersionType() == "UK" {
            return swiftStaticUtils.getUKMessage()
        }
        
        return "Welcome Mystery User"
    }
}
{% endhighlight %}

To prepare for injecting the type, we create and initialize a member variable in our MessageManager to hold the metatype.  We can then use the member variable to call the static methods.  So, the injection is ready.  Now we need to write a fake.  In order to write a fake, we need create a [protocol extension][protocol-extension] that both our fake and SwiftStaticUtils conform to.  To do this, we create a new protocol, SwiftStaticUtilsProtocol.  SwiftStaticUtilsProtocol belongs to our code, not our tests.

{% highlight swift %}
protocol SwiftStaticUtilsProtocol {
    static func getUSMessage() -> String
    static func getUKMessage() -> String
}

extension SwiftStaticUtils : SwiftStaticUtilsProtocol {}
{% endhighlight %}

Here, we define a new protocol with two static methods (Note, we left out getHour() because we don't need to control what it returns for our test).  We then add an extension to SwiftStaticUtils which means that SwiftStaticUtils conforms to SwiftStaticUtilsProtocol.  Said another way, it means that SwiftStaticUtils has a getUSMessage() and a getUKMessage() method, and we can refer to an instance of SwiftStaticUtils as an instance of SwiftStaticUtilsProtocol.  We can change our member variable in MessageManager to reflect this:

{% highlight swift %}
var swiftStaticUtils:SwiftStaticUtilsProtocol.Type = SwiftStaticUtils.self
{% endhighlight %}

If you rerun the code, it still works.  Now we can make a manual fake that conforms to SwiftStaticUtilsProtocol and inject it.  We will add FakeSwiftStaticUtils to our test code:

{% highlight swift %}
import Foundation
@testable import StaticMethods

class FakeSwiftStaticUtils : SwiftStaticUtilsProtocol {
    static func getUSMessage() -> String {
        return "Testing US"
    }
    
    static func getUKMessage() -> String {
        return "Testing UK"
    }
} 
{% endhighlight %}

We have to import the StaticMethods module so we can see SwiftStaticUtilsProtocol, and implement the two static methods from the protocol.  I give them different return values than the actual code so I am 100% sure the data is coming from my tests, not from the code.  We will now be able to create an instance of FakeSwiftStaticUtils and assign it to the swiftStaticUtils member variable in MessageManager for our tests.  

### Stubbing The Objective-C Static Method

Before we can put it all together, we still need to deal with the Objective-C static method.  Our code in MessageManager can react to the value of getAppVersionType in three ways.  It behaves differently if getAppVersionType returns "US", "UK", or something else.  In order to test all these scenarios, we need to force the static method to return a different value in each of our tests.  We will start by creating another protocol extension and fake.  In the code, we create ObjectiveCStaticUtilsProtocol. 

{% highlight swift %}
protocol ObjectiveCStaticUtilsProtocol {
    static func getAppVersionType() -> String
}

extension ObjectiveCStaticUtils : ObjectiveCStaticUtilsProtocol {}
{% endhighlight %}

Because ObjectiveCStaticUtils is an Objective-C class, you can expect some automatic renaming and type shuffling to occur, as it would if you imported your Objective-C class to Swift.  It can sometimes be handy to just temporarily override your Objective-C class to get Xcode to show you the method signatures.  In this case, note that getAppVersionType returns String, not NSString.  

With the protocol in place, we can again change MessageManager to use it:

{% highlight swift %}
class MessageManager {
    var swiftStaticUtils:SwiftStaticUtilsProtocol.Type = SwiftStaticUtils.self
    var objectiveCStaticUtils:ObjectiveCStaticUtilsProtocol.Type = ObjectiveCStaticUtils.self
    
    func getMessage() -> String {
        if objectiveCStaticUtils.getAppVersionType() == "US" {
            return swiftStaticUtils.getUSMessage()
        } else if objectiveCStaticUtils.getAppVersionType() == "UK" {
            return swiftStaticUtils.getUKMessage()
        }
        
        return "Welcome Mystery User"
    }
}
{% endhighlight %}

...and create a fake, FakeObjectiveCStaticUtils in the test code...

{% highlight swift %}
import Foundation
@testable import StaticMethods

class FakeObjectiveCStaticUtils : ObjectiveCStaticUtilsProtocol {
    static var stubbedAppVersion:String?
    
    static func getAppVersionType() -> String {
        return stubbedAppVersion!
    }
}
{% endhighlight %}

In addition to implementing getAppVersionType(), we also add a static variable called the "stubbedAppVersion."  We can set this static variable before calling getAppVersionType() to control what getAppVerisonType() returns.  It has to be static so our static method can access it.  Now, all we have to do is inject our two fakes and write our test methods.

### Putting It All Together

Given that all of our tests have the same dependencies, we can avoiding repeating ourselves (some folks will refer to this as DRY - Don't Repeat Yourself) and initialize our fakes in our setup and teardown methods of our test case.  The test will be run in this order:  setup() is called, the test is executed, tearDown() is called.  This happens with each test. 

{% highlight swift %}
 var fakeSwiftStaticUtils: FakeSwiftStaticUtils.Type!
 var fakeObjectiveCStaticUtils: FakeObjectiveCStaticUtils.Type!
    
 var messageManager: MessageManager!
    
 override func setUp() {
     messageManager = MessageManager()
     
     fakeSwiftStaticUtils = FakeSwiftStaticUtils.self
     messageManager.staticUtils = fakeSwiftStaticUtils
     
     fakeObjectiveCStaticUtils = FakeObjectiveCStaticUtils.self
     messageManager.objectiveCStaticUtils = fakeObjectiveCStaticUtils
 }
 
 override func tearDown() {
     fakeObjectiveCStaticUtils.stubbedAppVersion = "NONE"
     
     fakeSwiftStaticUtils = nil
     fakeObjectiveCStaticUtils = nil
 }
{% endhighlight %}

Much like in the code, we create two member variables using the metatype of the fake classes.  We also create the messageManager variable that is the instance we will be testing (call me crazy, but I find 'sut' more jarring to read than a named variable with context -- do as you will).  In the setUp method, we initialize the metatype variables to the types of our two fakes and inject them into the MessageManager.  The types in MessageManager are protocol types, so this is entirely valid.  Because we defined the variables in the MessageManager with initial values, this code will override those initial values, and inject what we want.  

For cleanup, we set our variables back to nil.  I also like to set the stubbedAppVersion in the FakeObjectiveCStaticUtils class to a nonsense value.  Usually when we stub, we are creating brand new instances of objects, and can let local scoping or the setUp and tearDown methods clean up for us.  In this case, because we are stubbing a static, the variable that holds the stub value has to be static too, so it is on us to be sure it gets re-initialized to a value we aren't testing for.  

Now let's look at the tests:

{% highlight swift %}
func test_getMessage_us() {
  fakeObjectiveCStaticUtils.stubbedAppVersion = "US"
  
  XCTAssertEqual(messageManager.getMessage(), "Testing US")
}

func test_getMessage_uk() {
  fakeObjectiveCStaticUtils.stubbedAppVersion = "UK"
  
  XCTAssertEqual(messageManager.getMessage(), "Testing UK")
}

func test_getMessage_somethingElse() {
  fakeObjectiveCStaticUtils.stubbedAppVersion = "NOWHERE"
  
  XCTAssertEqual(messageManager.getMessage(), "Welcome Mystery User")
}
{% endhighlight %}

The tests are easy now!  We just need to verify the message returns what we expect the stub to return, and include the error case.  We have successfully injected a fake for our Swift and Objective-C class of static methods and unit tested the method that uses them!

### The Downside To This

One of the general benefits to unit testing is the idea that if our code is testable, it is better code.  In this case, I believe we have made the code itself worse in order to test it.  Some will claim that using protocol extensions for injectable dependencies such as these makes a more flexible system, which is true.  However, that flexibility comes at the expense of extra indirection and complexity.  This makes the code needlessly complex and more difficult to understand for flexibility we likely will never need.  The chances are far higher that this code will confuse a fellow developer than it will provide a magic solution to a future problem we don't have.  That said, if we want to unit test, this is our choice.  

The obvious answer is to avoid static methods like this.  This is not always possible when dealing with systems that have libraries you don't control and are full of legacy code.  As with most things, there is no free lunch.  You can test this, but it does come with a few drawbacks.  As engineers, we have to balance those tradeoffs to make our codebase better.

[github]: https://github.com/weitzel926/StaticMethods
[swift-metatype]: https://docs.swift.org/swift-book/ReferenceManual/Types.html#ID455
[protocol-extension]: https://docs.swift.org/swift-book/LanguageGuide/Protocols.html#ID521