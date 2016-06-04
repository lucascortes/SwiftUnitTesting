#A Journey into Swift Unit Testing


Swift is a clean, safe and modern language that helps us build much better code in many ways. But testing is a dark corner in that new world that is often left forgotten. My aim with this article is to show all the different approaches I found while working with Objective-C and Swift.

Objective-C provides a really flexible and vast way of dealing with classes, allowing easy creation of stubs, mocks, and method swizzling. There are also widely used tools like OCMock and Expecta that work exclusively with objc to make testing extremely easy. So basically, with objc, developers have the tools they need to create their test suite.

###The (tough) transition

Since Swift was released and developers started migrating their objc code I've seen all kinds of attempts to provide code coverage to their new swift classes. And it also revealed some bad practices that all that objc flexibility allowed or even encouraged.

While I was working on the Restorando iOS app, we decided to start migrating a few classes to swift. All the test suite was written in objc using a testing framework with absolutely no Swift support. After some research we realized that there were no useful alternatives and we found an easy way out: keep testing in objc.

We just had to expose the swift classes to objc by subclassing NSObject or using the @objc directive. Also we had to mark most class members in the Swift classes as `dynamic` to require that access to them be dynamically dispatched through the objc runtime. So we ended up with code like the following:

```dynamic``` _keyword is not widely understood. should we also add a diagram to ilustrate what the this keyword allows?_

```javascript
class KittenViewController: UIViewController {
  private dynamic let kittens: [Kitten]
  [...]
}
```
```javascript
@interface KittenViewController (KittenViewControllerTest)
@property (nonatomic, strong) NSArray<Kitten>* __nonnull kittens;
@end

@implementation KittenViewControllerTests
- (void)testKittens {
  [...]
}
@end
```

So that was the first attempt. It didn't look that bad. We were able to test our swift classes and even expose private members.

But quickly Swift evolved and we started to embrace it full potential. Structs, Enums with associated values, tuples, and native types. There was no way to expose that in objc _very syntethically, but why?_. Then we knew it was time to find a better solution.

###Embracing Swift

So we started our research. The following snippet has a simple networking system. The `NetworkManager` class handles all the internet communication, while the `Store` subclasses provide an abstract way of requesting models.

```javascript
class NetworkManager {
    func requestWithPath(path: String) -> NSDictionary { [...] }
}

class Store {
    private let networkManager = NetworkManager()
}

class BookStore: Store {
    func getBook(code: String) -> Book? {
        let bookJSON = networkManager.requestWithPath("book/\(code)")
        return Book(bookJSON)
    }
}
```

Now if we want to test our `BookStore` class we can simply call `getBook`

```javascript
let bookStore = BookStore()
let book = bookStore.getBook("KRXPU")
```

That probably will generate a network request. That not only will take time, it will also make us dependent of external services that can fail and affect our test. Also, we won't be able to test different scenarios, like getting a specific book or not founding at all.

So we need to be able to mock `NetworkManager` to provide our customized behavior. The following implementation addresses that _I would also add some reference to this article somewhere http://martinfowler.com/articles/mocksArentStubs.html particularly on what's a stub and what's a mock_

```javascript
class Store {
    private let networkManager: NetworkManager

    init(networkManager: NetworkManager) {
        self.networkManager = networkManager
    }

}
```

So now we can make

```javascript
class NetworkManagerMock: NetworkManager {
//warning worth mentioning here: mocking by subclassing it's not a garantee that sideefects will not be triggered...
//Take into account what wasn't overriden or what could be extended in the real class in the future ;)

    var lastPath: String?
    var dictionaryToReturn = NSDictionary()

    override func requestWithPath(path: String) -> NSDictionary {
        lastPath = path
        return dictionaryToReturn
    }
}
```

And test that the right path is sent and the right book is returned.

```javascript
let networkManagerMock = NetworkManagerMock()
let bookStore = BookStore(networkManager: networkManagerMock)
let bookDictionary = [...] //Custom JSON
networkManagerMock.dictionaryToReturn = bookDictionary
let book = bookStore.getBook("KRXPU")


XCTAssertEqual(networkManagerMock.lastPath, "book/KRXPU")
XCTAssertEqual(Book(bookDictionary), book)
```

And this way we were able to mock `NetworkManager` to test `BookStore`.


Actually, we can avoid having to pass the dependency by parameter each time, since we can use default parameters. This way we only have include them when using the mock instance. So we get the following initializer.

```javascript
init(networkManager: NetworkManager = NetworkManager()))
```

###Beware of unknown implementations

A lot of the time we will want to mock library classes that we don't know how are implemented. For example, there are several libraries that provide networking access. In these scenarios, the internal implementation may be doing lot of stuff we have no control, like notifications, network requests, etc. For example, the following code

```javascript
class PrivateNetworkManager {
  init() {
    //do some networking here.
  }
}
```

Given that we are mocking by subclassing, if we don't override the initializer, this class will make a network request.

So we try a new approach

###A protocol oriented solution

We would like to use a custom mock without all the risks of subclassing.
Let's analyze this example

```javascript
protocol NetworkManagerProtocol {
    func requestWithPath(path: String) -> NSDictionary
}

class NetworkManager: NetworkManagerProtocol {
    func requestWithPath(path: String) -> NSDictionary { return [...] }
}

class Store {
    private let networkManager: NetworkManagerProtocol

    init(networkManager: NetworkManagerProtocol = NetworkManager()) {
        self.networkManager = networkManager
    }
}

class BookStore: Store {
    func getBook(code: String) -> Book? {
        let bookJSON = networkManager.requestWithPath("book/\(code)")
        return Book(bookJSON)
    }
}

```

The `Store` class only knows it has an instance that conforms to the `NetworkManagerProtocol` protocol. Then `BookStore` uses the method `requestWithPath`, that belongs to that protocol, and it's not aware of the actual class that implements it. So we can create a mock class that conforms to that protocol and doesn't mess with any internal implementation.

For example, the following snippet mocks some methods of `NSNotificationCenter`

```javascript
protocol NotificationCenterProtocol {
    func addObserver(observer: AnyObject, selector aSelector: Selector, name aName: String?, object anObject: AnyObject?)
    func removeObserver(observer: AnyObject)
}

extension NSNotificationCenter: NotificationCenterProtocol { }
```


###A dependency chaos

//Extremely long injections

//hard to deal with circular injections

//class methods dependencies


###Narrow light of hope

// dependency injection

###Pros and Cons
