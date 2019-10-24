#MVVM with Swift application 2/3 - Bindable section:
[Article Link](http://swiftyjimmy.com/mvvm-with-swift-application-23/)
_SwiftyJimmy_
February 22, 2017

> Bindable is just a simple way to start observing a value for changes. It is implemented using an observer pattern. `Bindable` is a crucial part of MVVM and it takes care of the data state changes between the view model and the view.

You could use some reactive lib such as **RxSwft**.
But we will follow the Bindable by method by first implementing the `Bindable` class:

``` swift
class Bindable {
    typealias Listener = ((T) -&gt; Void)
    var listener: Listener?
 
    var value: T {
        didSet {
            listener?(value)
        }
    }
 
    init(_ v: T) {
        self.value = v
    }
 
    func bind(_ listener: Listener?) {
        self.listener = listener
    }
 
    func bindAndFire(_ listener: Listener?) {
        self.listener = listener
        listener?(value)
    }
 
}
```

Here we have:
 - `typealias Listener` = just a function that takes a Generic type as a parameter that can be anything. 
 - `var value: T {...` = a variable that is the type of the **Listener** and is an optional 
 - the next parameter is called `value[rad-hl]` and it is the same Generic type that was used before
 - In the values' `[radhl]didSet` function = listener is called with the new value, so every time the value is changed listener is informed.
 
In the `init` function, the value is given both type and the starting value - if `T` is defined as an optional the parameter can be nil.

`Bindable` also has two functions: 
 - `bind`
 - `bindAndFire`

These are the functions that are going to be used inside the view.
The view starts to listen to a value by calling `bind` or `bindAndFire` and defines closure right after it.
Now that the observer is set, every time `Bindable` value is changed view is notified and it can update the data according to the change.

The difference between `bind` and `bindAndFire` is that `bindAndFire` also returns the starting value of the Bindable while bind only returns the value after the first time it changes.



<br>

---


#Bindable Values in Swift
[Article Link](https://www.swiftbysundell.com/posts/bindable-values-in-swift)
_Swift by Sundell_
March 31, 2019

## Intro

It's so important for applications our data models to be synced to our UI.
>Many different patterns and techniques have been invented in order to make it easier to ensure that a UI stays up to date whenever its underlying model changes — everything from notifications, to delegates, to observables. This week, let’s take a look at one such technique — that involves binding our model values to our UI.

>It’s so common to encounter bugs that causes stale data to be rendered, or errors that happen because of conflicts between the UI state and the rest of the app’s logic. 

## Constant Updates

One common way to ensure that our UI is always rendering the latest available data is to simply reload the underlying model whenever the UI is about to be presented (or re-presented) on the screen. For example, if we're building a profile screen for some form of social networking app, we might reload the User that the profile is for every time `viewWillAppear` is called on our `ProfileViewController`:

``` swift
class ProfileViewController: UIViewController {
    private let userLoader: UserLoader
    private lazy var nameLabel = UILabel()
    private lazy var headerView = HeaderView()
    private lazy var followersLabel = UILabel()

    init(userLoader: UserLoader) {
        self.userLoader = userLoader
        super.init(nibName: nil, bundle: nil)
    }

    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)

        // Here we always reload the logged in user every time
        // our view controller is about to appear on the screen.
        userLoader.load { [weak self] user in
            self?.nameLabel.text = user.name
            self?.headerView.backgroundColor = user.colors.primary
            self?.followersLabel.text = String(user.followersCount)
        }
    }
}
```

A few things that could potentially be improved:
  1. We always have to keep references to our various views on our view controller, since we're not able to assign our UI properties until we've loaded the view controller's model.
  2. When using a closure-based API to get access to the loaded model, we have to weakly reference self (or explicitly capture eac view) in order to avoid **retain cycles**.
  3. Each time our view controller is presented on screen, we'll reload the model, even if only seconds have passed since we last did so, and even if another view controller also reloads the same model at the same time - potentially resulting in wasted, or at least unnecessary, network calls.
  
We can address some of the above points by using a different kind of abstraction to give our view controller access to its model.
>Like we took a look at in “Handling mutable models in Swift”, instead of having the view controller itself load its model, we could use something like a UserHolder to pass in an observable wrapper around our core User model.

By doing so we could encapsulate our reloading logic, and do all required updates in a single place, away from our view controllers - resulting in a simplified `ProfileViewController` implementation:

``` swift
class ProfileViewController: UIViewController {
    private let userHolder: UserHolder
    private lazy var nameLabel = UILabel()
    private lazy var headerView = HeaderView()
    private lazy var followersLabel = UILabel()

    init(userHolder: UserHolder) {
        self.userHolder = userHolder
        super.init(nibName: nil, bundle: nil)
    }

    override func viewDidLoad() {
        super.viewDidLoad()

        // Our view controller now only has to define how it'll
        // *react* to a model change, rather than initiating it.
        userHolder.addObserver(self) { vc, user in
            vc.nameLabel.text = user.name
            vc.headerView.backgroundColor = user.colors.primary
            vc.followersLabel.text = String(user.followersCount)
        }
    }
}
```

While the above is a nice improvement over our original implementation, let's see if we can take things further - especially when it comes to the API that we expose to our view controllers - by instead _directly binding_ our model values to our UI.

## From Observable to Bindable

The idea behind **_value binding_** is to enable us to write auto-updating UI code by simply associating each piece of model data with a UI property, in a much more declarative fashion.

To make that happen, we're first going to replace our `UserHolder` type from before with a generic `Bindable` type. This new type will enable _any value_ to be bound to _any UI property_, without requiring specific abstractions to be built for each model. Let's start by declaring `Bindable` and defining properties to keep track of all of its observations, and to enable it to cache the latest value that passed through it, like this:

``` swift
class Bindable<Value> {
    private var observations = [(Value) -> Bool]()
    private var lastValue: Value?

    init(_ value: Value? = nil) {
        lastValue = value
    }
}
```

Next, let's enable `Bindable` to be observed, just like `UserHolder` before it - but with the key difference that we'll keep the observation method private:

``` swift
private extension Bindable {
    func addObservation<O: AnyObject>(
        for object: O,
        handler: @escaping (O, Value) -> Void
    ) {
        // If we already have a value available, we'll give the
        // handler access to it directly.
        lastValue.map { handler(object, $0) }

        // Each observation closure returns a Bool that indicates
        // whether the observation should still be kept alive,
        // based on whether the observing object is still retained.
        observations.append { [weak object] value in
            guard let object = object else {
                return false
            }

            handler(object, value)
            return true
        }
    }
}
```

Finally, we need a way to update `Bindable` instance whenever a new model became available. For that we'll add an `update` method that updates the bindable's `lastValue` and calls each observation through `filter`, in order to remove all observations that have become outdated:

``` swift
extension Bindable {
    func update(with value: Value) {
        lastValue = value
        observations = observations.filter { $0(value) }
    }
}
```

>It may be argued that using filter to apply side-effects (like we do above) isn’t theoretically correct, at least not from a strict functional programming perspective, but in our case it does exactly what we’re looking for — and since we’re not reliant on the order of operations, using filter is quite a good match, and saves us from essentially writing the exact same code ourselves.

With the above in place we can now start using our new `Bindable` type. We'll start by injecting a `Bindable<User>` instance into our `ProfileViewController`, and rather than setting up each of our views using properties on our view controller, we'll instead do all of their individual setup in dedicated methods that we call within `viewDidLoad()`:

``` swift 
class ProfileViewController: UIViewController {
    private let user: Bindable<User>

    init(user: Bindable<User>) {
        self.user = user
        super.init(nibName: nil, bundle: nil)
    }

    override func viewDidLoad() {
        super.viewDidLoad()

        addNameLabel()
        addHeaderView()
        addFollowersLabel()
    }
}
```

Our view controller is already starting to look much simpler, and we're now free to structure our view setup code however we please - since, through value binding, our UI updates no longer have to be defined within the same method.


## Binding Values

So far we've defined all of the underlying infrastructure that we'll need in order to actually start binding values to our UI - but to do that, we need an API to call.

>The reason we kept addObservation private before, is that we’ll instead expose a KeyPath-based API that we’ll be able to use to directly associate each model property with its corresponding UI property.

Like we took a look at in [“The power of key paths in Swift”](https://www.swiftbysundell.com/posts/the-power-of-key-paths-in-swift), key paths can enable us to construct some really nice APIs that give us dynamic access to an object’s properties, without having to use closures. Let’s start by extending Bindable with an API that’ll let us bind a key path from a model to a key path of a view:

``` swift
extension Bindable {
    func bind<O: AnyObject, T>(
        _ sourceKeyPath: KeyPath<Value, T>,
        to object: O,
        _ objectKeyPath: ReferenceWritableKeyPath<O, T>
    ) {
        addObservation(for: object) { object, observed in
            let value = observed[keyPath: sourceKeyPath]
            object[keyPath: objectKeyPath] = value
        }
    }
}
```

Since we'll sometimes want to bind values to an _optional property_ (such as text on UILabel), we'll also need an additional `bind` overload that accepts an `objectKeyPath` for an optional of `T`:

``` swift
extension Bindable {
    func bind<O: AnyObject, T>(
        _ sourceKeyPath: KeyPath<Value, T>,
        to object: O,
        // This line is the only change compared to the previous
        // code sample, since the key path we're binding *to*
        // might contain an optional.
        _ objectKeyPath: ReferenceWritableKeyPath<O, T?>
    ) {
        addObservation(for: object) { object, observed in
            let value = observed[keyPath: sourceKeyPath]
            object[keyPath: objectKeyPath] = value
        }
    }
}
```

With the above in place, we can now start binding model values to our UI, such as directly associating our user's `name` with the `text` property of the UILabel that'll render it:

``` swift
private extension ProfileViewController {
    func addNameLabel() {
        let label = UILabel()
        user.bind(\.name, to: label, \.text)
        view.addSubview(label)
    }
}
```

Pretty cool! And perhaps even cooler is that, since we based our binding API on key paths, we get support for nested properties completely for free. For example, we can now easily bind the nested colors.primary property to our header view's `backgroundColor`:

``` swift
private extension ProfileViewController {
    func addHeaderView() {
        let header = HeaderView()
        user.bind(\.colors.primary, to: header, \.backgroundColor)
        view.addSubview(header)
    }
}
```


## Transforms

>So far, all of our model properties have been of the same type as their UI counterparts, but that’s not always the case. For example, in our earlier implementation we had to convert the user’s followersCount property to a string, in order to be able to render it using a UILabel — so how can we achieve the same thing with our new value binding approach?

One way to do just that would be to introduce yet another `bind` overload that adds a `transform` parameter, containing a function that converts a valute of `T` into the required result type `R` - and to then use that function within our observation to perform the conversion, like this:

``` swift
extension Bindable {
    func bind<O: AnyObject, T, R>(
        _ sourceKeyPath: KeyPath<Value, T>,
        to object: O,
        _ objectKeyPath: ReferenceWritableKeyPath<O, R?>,
        transform: @escaping (T) -> R?
    ) {
        addObservation(for: object) { object, observed in
            let value = observed[keyPath: sourceKeyPath]
            let transformed = transform(value)
            object[keyPath: objectKeyPath] = transformed
        }
    }
}
```

Using the above transformation API, we can now easily bind our `followersCount` property to a UILabel, by passing `String.init` as the transform:

``` swift
private extension ProfileViewController {
    func addFollowersLabel() {
        let label = UILabel()
        user.bind(\.followersCount, to: label, \.text, transform: String.init)
        view.addSubview(label)
    }
}
```

Another approach would've been to introduce a more _specialized version_ of `bind` that directly converts between `Int` and `String` properties, or to base it on the `CustomStringConvertible` protocol (which `Int` and many other types conform to) - but with the above approach we have the flexibility to transform any value in any way we see fit.


## Automatic Updates

While our new `Bindable` type enables quite elegant UI code using key paths, the main purpose of introducing it was to make sure that our UI stays up-to-date automtaically whenever an underlying model was changed, so let's also take a look at the other side - how a model update will actually be triggered.

Here our core `User` model is managed by a **model controller**, which syncs the model with our server every time the app becomes active - and then calls updated on its `Bindable<User>` to propagate any model changes throughout the app's UI:

``` swift
class UserModelController {
    let user: Bindable<User>
    private let syncService: SyncService<User>

    init(user: User, syncService: SyncService<User>) {
        self.user = Bindable(user)
        self.syncService = syncService
    }

    func applicationDidBecomeActive() {
        syncService.sync(then: user.update)
    }
}
```

>What’s really nice about the above is that our UserModelController can be completely unaware of the consumers of its user data, and vice versa — since our Bindable acts as a layer of abstraction for both sides, which enables both a higher degree of testability, and also makes for a more decoupled system overall.


## Conclusion

By binding our model values directly to our UI, we can both end up with simpler UI configuration code that eliminates common mistakes (such as accidentally strongly capturing view objects in observation closures), and also enures that all UI values will be kept up-to-date as their underlying model changes. By introducing an abstraction such as `Bindable`, we can also more clearly separate our UI code from our core model logic.

>The ideas presented in this article are strongly influenced by Functional Reactive Programming, and while more complete FRP implementations (such as RxSwift) take the idea of value binding much further (for example by introducing two-way binding, and enabling the construction of observable value streams) — if all we need is simple unidirectional binding, then something like a Bindable type may do everything that we need, using a much thinner abstraction.


