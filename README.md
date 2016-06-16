#A Journey into Swift Unit Testing


`Swift` is a clean, safe and modern language that helps us build much better code in many ways. But testing is a dark corner in that new world that is often left forgotten. My aim with this article is to show all the different approaches I found while working with Objective-C and Swift.

`Objective-C` provides a flexible and vast way of working with classes, allowing easy creation of *stubs*, *mocks*, and *method swizzling*. There are also widely used tools like *OCMock* and *Expecta* that work exclusively with objc to make testing extremely easy. So basically, with objc, developers have the tools they need to create their test suite.

###The (tough) transition

Since Swift was released and developers started migrating their objc code I've seen all kinds of attempts to provide code coverage to their new Swift classes. It also revealed some bad practices that all that objc flexibility allowed or even encouraged.

While working on the Restorando iOS app, we decided to start migrating a few classes to Swift. All the tests were written in objc using a testing framework with absolutely no Swift support. After some research we realized that there were no suitable alternatives and we found an easy way out: keep testing in objc.

We just had to expose the Swift classes to objc by subclassing NSObject or using the `@objc` directive. Also we had to mark most class members in the Swift classes as `dynamic` to require that access to them be dynamically dispatched through the objc runtime. Basically, it allows to exchange method implementations, which is the way the testing frameworks work under the hood. So we ended up with code like the following:

```javascript
class BooksViewController: UIViewController {
  private dynamic let books: [Book]
  [...]
}
```
```javascript
@interface BooksViewController (BooksViewController)
@property (nonatomic, strong) NSArray<Book>* __nonnull books;
@end

@implementation BooksViewControllerTests
- (void)testBooks {
  [...]
}
@end
```

So that was the first attempt. It didn't look that bad. We were able to test our Swift classes and even, although not ideal, expose private members.

But quickly Swift evolved and we started to embrace its full potential. Structs, Enums with associated values, tuples, and native types. There is no way to expose that in objc since it doesn't have those features. Then we knew it was time to find a better solution.

###Embracing Swift

So we started our research. I'll use the following snippet to describe the process. The code has a simple networking system. The `NetworkManager` class handles all the internet communication, while the `Store` subclasses provide an abstract way of requesting models.

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

This will generate a network request that not only will take time, it will also make us dependent of external services that can fail and affect our test. Also, we won't be able to test different scenarios, like getting a specific book or not finding it at all.

We need to be able to modify the default behavior of the `getBook` method. Since Swift limitations don't allow to use *mocks* we'll use [stubs](http://martinfowler.com/articles/mocksArentStubs.html).

We'll start by stubbing `NetworkManager` to provide our customized behavior. The following implementation addresses that by providing the `NetworkManager` instance in the initializer. This way we can create a `Store` that uses a specific `NetworkManager`.

```javascript
class Store {
    private let networkManager: NetworkManager

    init(networkManager: NetworkManager) {
        self.networkManager = networkManager
    }

}
```

So now we can subclass `NetworkManager` and use properties to check the expected behavior. (As we'll see later, this doesn't guarantee that side effects will not be triggered)

```javascript
class NetworkManagerStub: NetworkManager {
    var lastPath: String?
    var dictionaryToReturn = NSDictionary()

    override func requestWithPath(path: String) -> NSDictionary {
        lastPath = path
        return dictionaryToReturn
    }
}
```
When we use the `requestWithPath` method, we'll save the requested path in the `lastPath` property and we'll return a custom dictionary. This clearly allow us to test that the right path is sent and the right book is returned as the following example shows:

```javascript
let networkManagerStub = NetworkManagerStub()
let bookStore = BookStore(networkManager: networkManagerStub)
let bookDictionary = [...] //Custom JSON
networkManagerStub.dictionaryToReturn = bookDictionary
let book = bookStore.getBook("KRXPU")


XCTAssertEqual(networkManagerStub.lastPath, "book/KRXPU")
XCTAssertEqual(Book(bookDictionary), book)
```

This way we were able to *stub* `NetworkManager` to test `BookStore`.


Actually, we can avoid passing the dependency by parameter in the initializer each time since we can use default parameters. So we only have include them when using the stub instance. Then we get the following initializer.

```javascript
init(networkManager: NetworkManager = NetworkManager()))
```

###Beware of unknown implementations

A lot of the time we will want to *stub* library classes that we don't know how are implemented. For example, there are several libraries that provide networking access. In these scenarios, the internal implementation may be doing lot of stuff we have no control, like notifications, network requests, etc. For example, consider the following code

```javascript
class PrivateNetworkManager {
  init() {
    //do some networking here.
  }
}
```

Given that we are stubbing by subclassing, if we don't override the initializer, this class will make a network request. This could also happen in property observers, extensions, methods, etc.

So we try a new approach

###A protocol oriented solution

In the following code snippet, the `Store` class has 2 dependencies: `NetworkManager` and `NSNotificationCenter`. We use the default parameters in the initializers as described above and we get the following class:

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
We would like to test `Store` without all the risks of subclassing. By creating a **protocol** that encapsulates the interfaces we can communicate between classes by their interfaces and not by their actual type.

`NetworkManager` only has one public method, so let's create a *protocol* with it and make our `NetworkManager` class conform to this new *protocol*.

```javascript
protocol NetworkManagerProtocol {
    func requestWithPath(path: String) -> NSDictionary
}

class NetworkManager: NetworkManagerProtocol {
    func requestWithPath(path: String) -> NSDictionary { return [...] }
}
```

So now, we make our `Store` class know about a `NetworkManagerProtocol` protocol with the `requestWithPath` method.

```javascript
class Store {
    private let networkManager: NetworkManagerProtocol
    ...
}
```

Our other dependency, `NSNotificationCenter`, is a *Foundation* class. We can use *protocol extensions* to make `NSNotificationCenter` conform to our *protocol*. For example, we would like to use these 4 basic methods: `defaultCenter`, `addObserver`, `removeObserver`, `postNotification`.

Then we create our *protocol* with those methods and then make `NSNotificationCenter` conform to it.

```javascript
protocol NotificationCenterProtocol {
    static func defaultCenter() -> NSNotificationCenter
    func addObserver(observer: AnyObject, selector aSelector: Selector, name aName: String?, object anObject: AnyObject?)
    func postNotification(notification: NSNotification)
    func removeObserver(observer: AnyObject)
}

extension NSNotificationCenter: NotificationCenterProtocol { }
```
The `defaultCenter` is a class method, so we use the `static` keyword in the protocol. One can quickly see that its return type is `NSNotificationCenter`. We want to make this protocol independent of that class (remember we just want to communicate between protocols).

So we can use the *protocol extension* we just created with a default implementation. We will include a similar method called `defaultNotificationCenter` and make our *protocol* its return type. The default implementation must return the usual `defaultCenter` instance.

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

Note that we don't have to provide an implementation all the methods, they are already implemented in `NSNotificationCenter`, which conforms to our new protocol.

Our `Store` classes are now fully testable.

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

The `Store` class knows it has an member that conforms to the `NetworkManagerProtocol` protocol. Then `BookStore` uses the method `requestWithPath`, that belongs to that protocol, and it's not aware of the actual class that implements it. So we can create a *stub class* that conforms to that protocol and doesn't mess with any internal implementation, just by making:

```javascript
class NetworkManagerStub: NetworkManagerProtocol {
    func requestWithPath(path: String) -> NSDictionary {
        //Some implementation
    }
}
```

###A dependency chaos

This way of stubbing has two major disadvantages.

First of all, we have to **create a protocol for each class** that we want to test. Each time we want to modify some class member we have to do it in two places (the *class* and the *protocol*) and, of course, it leads to duplicated code.
I don't think this approach is completely a bad idea, but it should be delimited. When we don't know the class implementation, creating protocols is almost required. But if the class is ours and we know how it works, it's not necessary to create a completely new *protocol* but override the right members.

Then we have a much bigger problem: **Extreme long initializers**. Although we use default parameters in the initializers, in our testing code we must provide our stubbed classes by parameter. Even small project classes have lots of dependencies. Just consider the following example with 5 `structs`.

```javascript
struct A { let b: B, c: C }
struct B { let c: C, d: D }
struct C { let d: D, e: E }
struct D { }
struct E { }
```

If we want to instantiate `A` we end up with the following initialization code:

```javascript
let a = A(b: B(c: C(), d: D()), c: C(d: D(), e: E()))
```

Testing this way would become instantly insane.

###Narrow light of hope

With all those options analyzed, it was clear that a more comprehensive solution was needed. So we tried **dependency injection**. There's an awesome framework by [Yoichi Tagaya](https://github.com/yoichitgy) called [Swinject](https://github.com/Swinject/Swinject). I'll use this framework to show how to test these classes using *dependency injection* and how to put all the previous techniques and learnings in use.

#####The syntax

Although the framework is really flexible, the following explanation only grasps the ground of what can be done with it. You can read more about it on the [Swinject's Github page](https://github.com/Swinject/Swinject)

Just to introduce the basics, I'll show some examples.
We start with 2 types: `Book` and `Library`.

```javascript
struct Book {
    var code: String
}

struct Library {
    var books: [Book]
}
```

And create a `Swinject` container. A *container* holds a set of *dependencies*.

```javascript
let container = Container()
```
We will associate *protocols or type declarations* with *type constructors* in containers. For example, the Book type is **registered** with a Book constructor associated that has the "KPSRC" code.

```javascript
container.register(Book.self) { _ in Book(code: "KPSRC") }
```

And we can get the actual `Book` instance by using the **resolve** method on the container. We need to *unwrap* the return value because if the dependency is not found, it returns nil.

```javascript
container.resolve(Book.self)!.code //returns "KPSRC"
```

We could register the Library by doing

```javascript
container.register(Library.self) { r in Library(books: [r.resolve(Book.self)!]) }
container.resolve(Library.self)!.books.first!.code //returns "KPSRC"
```

#####The implementation

We need to access the container from everywhere in the app so it will be a `global` variable.
We want to inject the dependencies to `Store`.
As described above, for `NSNotificationCenter` we'll use a protocol (e.g. `NotificationCenterProtocol`) because it's a *Foundation* class and for `NetworkManager` we'll use a subclass. So we have:

```javascript
var container = Container()

container.register(NotificationCenterProtocol.self) { _ in NSNotificationCenter.defaultNotificationCenter() }
container.register(NetworkManager.self) { _ in NetworkManager() }
```

And we just resolve our dependencies inside the class, as **private constant properties**.

```javascript
class Store {
    private let networkManager = container.resolve(NetworkManager.self)!
    private let notificationCenter = container.resolve(NotificationCenterProtocol.self)!
}
```

This way, although our dependencies remain less explicit, we handle them in one place and avoid overloading our initializers.

#####The testing

Testing this way becomes extremely simple. In the `setup` of each test we will register each *stub* dependency in a new container, and they will be resolved in the test.

```javascript
class BookStoreTests: XCTestCase {

    override func setUp() {
        super.setUp()
        var auxContainer = Container()

        auxContainer.register(NotificationCenterProtocol.self) { _ in NSNotificationCenterStub.defaultNotificationCenter() }
        auxContainer.register(NetworkManager.self) { _ in NetworkManagerStub() }

        container = auxContainer
    }

    func testBookStoreExample() {
        [...]
        //Here you can use BookStore with the registered dependencies.
        let networkManagerStub = container.resolve(NetworkManager.self)!

        let bookStore = BookStore()
        let bookDictionary = [...] //Custom JSON
        networkManagerStub.dictionaryToReturn = bookDictionary
        let book = bookStore.getBook("KRXPU")


        XCTAssertEqual(networkManagerStub.lastPath, "book/KRXPU")
        XCTAssertEqual(Book(bookDictionary), book)
    }
}
```

In the test method we are able to resolve the dependencies and instantiate any other class using them. This allows us to easily choose which ones we want in each test.

###Every journey comes to an end

This was my experience finding a good way of testing our new Swift code. There are still lots of things I was not able to cover in this post. Probably a next one will cover those.

There are some interesting issues that came across during development that it's useful to be aware of are:

* Check your code for ***circular dependencies***. In those cases you might need to check your app's architecture first to avoid them when possible.

* ***Class methods should not have dependencies***. If you have dependencies in those classes you probably want to use `singleton`s instead.

* Be clear on ***what you want to test***. Particular approaches should be considered depending on your app's architecture and the kind of classes you want to test.
