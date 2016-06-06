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

In the following code snippet, the `Store` class has 2 dependencies: `NetworkManager` and `NSNotificationCenter`.

```javascript
class Store {
    private let networkManager: NetworkManager
    private let notificationCenter: NSNotificationCenter

    init(networkManager: NetworkManager = NetworkManager(), notificationCenter: NSNotificationCenter = NSNotificationCenter.defaultCenter()) {
        self.networkManager = networkManager
        self.notificationCenter = notificationCenter
    }
}
```
We would like to test them without all the risks of subclassing. By creating a protocol that encapsulates the interfaces we can communicate between classes by their interfaces and not by their actual type.

`NetworkManager` only has one public method, so let's create a protocol with that and make our `NetworkManager` class conform to it.

```javascript
protocol NetworkManagerProtocol {
    func requestWithPath(path: String) -> NSDictionary
}

class NetworkManager: NetworkManagerProtocol {
    func requestWithPath(path: String) -> NSDictionary { return [...] }
}
```

So now, our `Store` class only knows about a `NetworkManagerProtocol` protocol.

```javascript
class Store {
    private let networkManager: NetworkManagerProtocol
    ...
}
```

But now, `NSNotificationCenter` is a Foundation class. We can use protocol extensions to make `NSNotificationCenter` conform to our protocol. For example, we would like to use these 4 basic methods: `defaultCenter`, `addObserver`, `removeObserver`, `postNotification`.

The `defaultCenter` is a class method, so we use the `static` keyword in the protocol.

```javascript
protocol NotificationCenterProtocol {
    static func defaultCenter() -> NSNotificationCenter
    func addObserver(observer: AnyObject, selector aSelector: Selector, name aName: String?, object anObject: AnyObject?)
    func postNotification(notification: NSNotification)
    func removeObserver(observer: AnyObject)
}

extension NSNotificationCenter: NotificationCenterProtocol { }
```

But the return type is `NSNotificationCenter`, and we want to make this protocol independent of that class. So we can use that protocol extension with a default implementation to create a similar method and return our protocol.

```javascript
protocol NotificationCenterProtocol {
    static func defaultNotificationCenter() -> NotificationCenterProtocol
    ...
}

extension NSNotificationCenter: NotificationCenterProtocol {
    static func defaultNotificationCenter() -> NotificationCenterProtocol {
        return defaultCenter()
    }
}
```

Note that we don't have to provide an implementation all those methods, they are already implemented in `NSNotificationCenter`, which conforms to our new protocol.

Our Store classes are now fully testable.

```javascript
class Store {
    private let networkManager: NetworkManagerProtocol
    private let notificationCenter: NotificationCenterProtocol

    init(networkManager: NetworkManagerProtocol = NetworkManager(), notificationCenter: NotificationCenterProtocol = NSNotificationCenter.defaultNotificationCenter()) {
        self.networkManager = networkManager
        self.notificationCenter = notificationCenter
    }
}

class BookStore: Store {
    func getBook(code: String) -> Book? {
        let bookJSON = networkManager.requestWithPath("book/\(code)")
        return Book(bookJSON)
    }
}

```

The `Store` class only knows it has an member that conforms to the `NetworkManagerProtocol` protocol. Then `BookStore` uses the method `requestWithPath`, that belongs to that protocol, and it's not aware of the actual class that implements it. So we can create a mock class that conforms to that protocol and doesn't mess with any internal implementation.

###A dependency chaos

This way of mocking has two major disadvantages.

First of all, we have to **create a protocol for each class** that we want to test. Each time we want to modify some class member we have to do it in two places and, of course, it leads to duplicated code.
I don't think this is completely a bad idea, but it should be delimited. When we don't know the class implementation, creating protocols is almost required. But if the class is ours and we know how it works, it's not necessary to create a completely new protocol but override de right members.

Then we have a much bigger problem: **Extreme long initializers**. Although we use default parameters in the initializers, in our testing code we must provide our mocks by parameter. Even small project classes have lots of dependencies. Just consider the following example with 5 `structs`.

```javascript
struct A { let b: B, c: C }
struct B { let d: D, e: E }
struct C { let d: D, e: E }
struct D { }
struct E { }
```

If we want to instantiate `A` we end up with the following initialization

```javascript
let a = A(b: B(d: D(), e: E()), c: C(d: D(), e: E()))
```

Testing this way would become instantly insane.

###Narrow light of hope

With all those options analyzed, it was clear that a more comprehensive solution was needed. So we tried **dependency injection**. There's an awesome framework by [Yoichi Tagaya](https://github.com/yoichitgy) called [Swinject](https://github.com/Swinject/Swinject). I'll use this framework to show how to test these classes using dependency injection and how to put all the previous techniques in use.

#####The syntax

Although the framework is really flexible, the following explanation only grasps the ground of what can be done with it. You can read more about the it on the [Swinject's Github page](https://github.com/Swinject/Swinject)

We start with these types

```javascript
struct Book {
    var code: String
}

struct Library {
    var books: [Book]
}
```

And create a `Swinject` container

```javascript
let container = Container()
```
We will associate protocols or type declarations with type constructors in containers. For example, the Book type is registered with a Book constructor associated that has the "KPSRC" code.

```javascript
container.register(Book.self) { _ in Book(code: "KPSRC") }
```

And we can get the actual instance by using the resolve method on the container

```javascript
container.resolve(Book.self)!.code //returns "KPSRC"
```

We could register the Library the following way

```javascript
container.register(Library.self) { r in Library(books: [r.resolve(Book.self)!]) }
container.resolve(Library.self)!.books.first!.code //returns "KPSRC"
```

#####The implementation

We need to access the container from everywhere in the app so it will be a `global` variable.
As described above, we'll use the `NotificationCenterProtocol` protocol and subclass `NetworkManager`. So we have

```javascript
var container = Container()

container.register(NotificationCenterProtocol.self) { _ in NSNotificationCenter.defaultNotificationCenter() }
container.register(NetworkManager.self) { _ in NetworkManager() }
```

And we just resolve our dependencies inside the class, as private constant properties.

```javascript
class Store {
    private let networkManager = container.resolve(NetworkManager.self)!
    private let notificationCenter = container.resolve(NotificationCenterProtocol.self)!
}
```

This way we get a centralized dependencies manager and avoid overloading our initializers.

#####The testing

Testing this way becomes extremely simple. In the `setup` of each test we will register each mock dependency in a new container, and they will be resolved in the test.

```javascript
class BookStoreTests: XCTestCase {

    override func setUp() {
        super.setUp()
        var auxContainer = Container()

        container.register(NotificationCenterProtocol.self) { _ in NSNotificationCenterMock.defaultNotificationCenter() }
        container.register(NetworkManager.self) { _ in NetworkManagerMock() }
    }

    func testBookStore() {
        [...]
        //Here you can use your mock classes
    }
}
```

In the test method we are able to resolve the dependencies and instantiate any other class using them. This allows us to easily choose which ones we want in each test.

###Every journey comes to an end

This was my experience finding the best way to test our new Swift code. There are still lots of things I was not able to cover in this post. Probably a next one will cover those.

There are some interesting issues that came across during development that it's useful to be aware of are:

* Check your code for ***circular dependencies***. In those cases you might need to check your app's architecture first to avoid them when possible.

* ***Class methods should not have dependencies***. If you have dependencies in those classes you probably want to use a `singleton` instead.

* Be clear on ***what you want to test***. Particular approaches should be considered depending on your app's architecture and the kind of classes you want to test.
