# VContainer

**VContainer** is an DI (Dependency Injection) library running on Unity.
It is extra fast, minimum code size, and GC-free implementation for using reflection that works IL2CPP.

VContainer aims to be a replacement for any existing DI library or your own implementation in Unity.  
Altough it is not intentionally included the features that should be freely exchanged by your application to keep minimum build size.

Features
- Support DI basic features. Such as constructor-injection, method-injection, property-injection field-injection and more.
- Support scheduling your custom types to Unity's PlayerLoopSystem. And some Unity friendly features. (It is greatly inspired by Zenject)
- Support code first nested scope, and scene based auto nested scope.
- VContainer has immutable container. The main benefits are safety, reusability, and improved performance.

**"V"** means making Unity's initial "U" more solid ... !

![](docs/unity_performance_test_result.png)

| |Resolve (Singleton)|Resolve (Transient)|Resolve (Combined)|Resolve (Complex)|Prepare & Register (Complex)|
|:---|:------------------|:------------------|:-----------------|:----------------|:---------------------------|
|VContainer|4.918865|19.2531|49.60295|57.0287|272.3955|
|Zenject|21.209325|79.1066|164.6642|441.1458|683.2439|

This benchmark was run by [Performance Testing Extension for Unity Test Runner](https://docs.unity3d.com/Packages/com.unity.test-framework.performance@1.0/manual/index.html).  
Test cases is [here](./VContainer.Benchmark).

In resolve, We have **zero allocation** (without resolved instances).

VContainer:

![](docs/screenshot_profiler_vcontainer.png)

Zenject:

![](docs/screenshot_profiler_zenject.png)

## Index

- [What is DI ?](#what-is-di-)
- [Installation](#installation)
- [Getting Started](#getting-started)
- [Registering](#registering)
- [Resolving](#resolving)
- [Controlling Object Lifetime](#controlling-object-lifetime)
- [Dispatching Unity Lifecycle](#dispatching-unity-lifecycle)
- [Best Practices and Recommendations](#best-practices-and-recommendations)

## What is DI ?

DI(Dependenciy Injection) は、オブジェクト指向において、Exchangeability, Extensibility, Maintenainability, Testability のある "クラス"のファイルを書くための、よく知られたテクニックです。

In programming all paradigms, weak module coupling and strong module cohesion are important.

How does OOP(Object Oriented Programming) do this?

In OOP(Object Oriented Programming),
an object hides implementation details within itself and communicates with other objects through an exposed interface.

オブジェクト指向においては、オブジェクトは自身の内に実装の詳細を隠蔽し、公開されたインターフェイスを通じて他のオブジェクトをコミュニケーションします。

ところが、オブジェクトが仕事を完遂するためには、隠蔽した状態と引数だけでは不十分です。
他者のオブジェクトの実装に依存します
これが "Dependency" です。

Auto wireing です。

 Unity's `[SerializeField]` について考えてみましょう。"隠蔽された実装" を
 アプリケーションEntiteryに必要な "機能" の公開インターフェイスを 破綻します。
 しばしば、代替として持ちいられるのは static "Singleton" パターンですが、これは実装をハードコードするに等しく、実装のExchangebilityがありません。

## Installation

 wip

## Getting Started

The basic way for integrating VContainer into your application is:

- Create `LifetimeScope` component in your scene. It has one Container and one scope.
- Write C# code to register dependencies. This is the composition root, called `Installer` like Zenject.
- Add the above installer into `LifetimeScope`.
- When playing scene, LifetimeScope automatically build Container and dispatch to the own PlayerLoopSystem.

Note:
Normally, "scope" is repeatedly created and destroyed during the game.  
LifetimeScope assumes this and has a parent-child relationship. We can child a loaded scene or dynamically spawn from code.

**3. Create LifetimeScope**

Right Click inside the Hierarchy tab and select **VContainer -> Lifetime Scope**.

Then attach your installer.

![](docs/screenshot_add_monoinstaller.gif)


**1. Write a class that depends on other classes**

Let's say Hello world.

```csharp
namespace MyGame
{
    public class HelloWorldService
    {
        public void Hello()
        {
            UnityEngine.Debug.Log("Hello world");
        }
    }
}
```

**2. Define composition root**

Next, let's write a setting that can auto-wiring the class. This place is called an Installer.

- Right click in a folder within the Project Tab and Choose **Create -> C# Script**.
- Name it `GameInstaller.cs`.

Note that VContainer will automatically template C# scripts ending in `Installer`.

You instruct `builder` and register the class above.

```diff
using VContainer;
using VContainer.Unity;

namespace MyGame
{
    public class GameInstaller : MonoInstaller
    {
        public override void Install(IContainerBuilder builder)
        {
+            builder.Register<HelloWorldService>(Lifetime.Singleton);
        }
    }
}
```

Note:  
VContainer always required a `Lifetime` argument explicitly.  
This gives us transparency and consistency.


**4. How to use your new HelloWorldService  ?**

Registered objects will automatically have dependency injection.
Like below:

```csharp
using VContainer;
using VContainer.Unity;

namespace MyGame
{
    public class GamePresenter
    {
        readonly HelloWorldService helloWorldService;

        public GamePresenter(HelloWorldService helloWorldService)
        {
            this.helloWorldService = helloWorldService;
        }
    }
}
```

And let's also register this class.

```diff
builder.Register<HelloWorldService>(Lifetime.Singleton);
+ builder.Register<GamePresenter>(Lifetime.Singleton);
```

Note:  
Press Validate button, you can check for missing dependencies.

![](docs/screenshot_validate_button.png)

**5. Execute your registerd object on PlayerLoopSystem**

To write an application in Unity, we have to interrupt Unity's lifecycle events.  
(Typically MonoBehaviour's Start / Update / OnDestroy / etc..)

Objects registered with VContainer can do this independently of MonoBehaviour.  
This is done automatically by implementing and registering some marker interfaces.

```diff
using VContainer;
using VContainer.Unity;

 namespace MyGame
 {
-    public class GamePresenter
+    public class GamePresenter : ITickable
     {
         readonly HelloWorldService helloWorldService;

         public GamePresenter(HelloWorldService helloWorldService)
         {
             this.helloWorldService = helloWorldService;
         }
+
+        void ITickable.Tick()
+        {
+            helloWorldService.Hello();
+        }
     }
 }
```

Now, `Tick()` will be executed at the timing of Unity's Update.

As such, it's a good practice to keep any side effect entry points through the marker interface.

See [](#dispatching-unity-lifecycle) section.

We should register this as `ITickable` marker.

```csharp
builder.Register<GamePresenter>(Lifetime.Singleton)
    .As<ITickable>();
```

Marker interface is a bit noisy to specify, so we will automate it below.

```csharp
builder.Register<GamePresenter>(Lifetime.Singleton)
    .AsImplementedInterfaces();
```

**Recommendation:**  
Registering lifecycle events without relying on MonoBehaciour facilitates decupling of domain logic and presentation !

## Registering

### Scoping overview

- Singleton : Single instance per container (includes all parents and children).
- Transient : Instance per resolving.
- Scoped    : Instance per `LifetimeScope`.
  - If LifetimeScope is single, similar to Singleton.
  - If you create a LifetimeScope child, the instance will be different for each child.
  - When LifetimeScope is destroyed, release references and calls all the registered `IDisposable`.

See more information: [Controlling Object Lifetime](#controlling-object-lifetime)

### Register method family

There are various ways to use Register.  
Let's take the following complex type as an example.

```csharp
class ServiceA : IServiceA, IInputPort, IDisposable { /* ... */ }
```

#### Register Concrete Type

```csharp
builder.Register<ServiceA>(Lifetime.Singleton);
```

It can resolve like this:

```csharp
class ClassA
{
    public ClassA(IServiceA serviceA) { /* ... */ }
}
```

####  Register as Interface

```csharp
builder.Register<IServiceA, ServiceA>();
```

It can resolve like this:

```csharp
class ClassA
{
    public ClassA(IServiceA serviceA) { /* ... */ } 
} 
```

**Register as multiple Interface**

```csharp
builder.Register<ServiceA>(Lifetime.Singleton)
    .As<IServiceA, IInputPort>();
```

It can resolve like this:

```csharp
class ClassA
{
    public ClassA(IServiceA serviceA) { /* ... */ }
}

class ClassB
{
    public ClassB(IInputPort handlerA) { /* ... */ }
}
```

**Register all implemented interfaces automatically**

```csharp
builder.Register<ServiceA>(Lifetime.Singleton)
    .AsImplementedInterfaces();
```

```csharp
class ClassA
{
    public ClassA(IServiceA serviceA) { /* ... */ }
}

class ClassB
{
    public ClassB(IHandlerB handlerA) { /* ... */ }
}
```

**Register all implemented interfaces and concrete type**

```csharp
builder.Register<ServiceA>(Lifetime.Singleton)
    .AsImplementedInterfaces()
    .AsSelf();
```

```csharp
class ClassA
{
    public ClassA(IServiceA serviceA) { /* ... */ }
}

class ClassB
{
    public ClassB(IHandlerB handlerA) { /* ... */ }
}

class ClassB
{
    public ClassB(ServiceA serviceA) { /* ... */ }
}
```

#### Register instance

```csharp
builder.RegisterInstance(obj);
```

Note that `RegisterIntance` always `Scoped` lifetime. So it has no arguments.

**Register instance as interface**

```csharp
builder.RegisterInstance<IInputPort>(serviceA);

builder.RegisterInstance(serviceA)
    .As<IServiceA, IInputPort>;
    
builder.RegisterInstance()
    .AsImplementedInterfaces();    
```

#### Register type-specific parameters

If the types are not unique, but you have a dependency you want to inject at startup, you can use below:

```csharp
builder.Register<SomeService>(lifetime.Singleton)
    .WithParameter<string>("http://example.com");
```

```csharp
class SomeService
{
    public SomeService(string url) { /* ... */ }
}
```

This Register is with only `SomeService`.

```csharp
class OtherClass
{
    // ! Error 
    public OtherClass(string hogehoge) { /* ... */ }
}
```

OR

You can parameter name as a key.

```csharp
builder.Register<SomeService>(Lifetime.Singleton)
    .WithParameter("url", "http://example.com");
```


#### Register `UnityEngine.Object`

**Register from MonoInstaller's `[SerializeField]`**

Also use `RegisterInstance()`.

```csharp
[SerializeField]
YourBehaviour yourBehaviour;

// ...

builder.RegisterInstance(yourBehaviour);
```

**Register from scene with `LifetimeScope`**

```csharp
builder.RegisterComponentInHierarchy<YourBehaviour>();
```

Note that `RegisterComponentInHierarchy` always `.Scoped` lifetime.  
Because lifetime is equal to the scene.

**Register component that Instantiate from prefab when resolving**

```csharp
[SerializeField]
YourBehaviour prefab;

// ...

builder.RegisterComponentInNewPrefab(prefab, Lifetime.Scoped);
```

**Register component that with new GameObject when resolving**

```csharp
builder.RegisterComponentOnNewGameObject<YourBehaciour>(Lifetime.Scoped, "NewGameObjectName");
```

**Register component as interface**

```csharp
builder.RegisterComponentInHierarchy<YourBehaviour>()
    .AsImplementedInterfaces();
```

**Register component to specific parent Transform**

```csharp
builder.RegisterComponentFromInNewPrefab<YourBehaviour>(Lifetime.Scoped)
    .UnderTransform(parent);

```

Or find at runtime.

```csharp
builder.RegisterComponentFromInNewPrefab<YourBehaviour>(Lifetime.Scoped)
    .UnderTransform(() => {
        // ...
        return parent;
    });

```

## Resolving

### Constructor Injection

VContainer automatically collects and calls registered class constructors.

Note:
- At this time, all the parameters of the constructor must be registered.
- Invalid constructor throws exception when validating LifetimeScope or when building Container.

Here is basic idiom with DI.

```csharp
class ClassA
{
    readonly IServiceA serviceA;
    readonly IServiceB serviceB;
    readonly SomeUnityComponent component;
     
    public ClassA(
        IServiceA serviceA, 
        IServiceB serviceB,
        SomeUnityComponent component)
    {
        this.serviceA = serviceA;
        this.serviceB = serviceB;
        this.component = component;
    }
}
```

:warning: Constructors are often stripped in the IL2CPP environment.
To prevent this problem, add the `[Inject]` Attribute explicitly.

```csharp
    [Inject]
    public ClassA(
        IServiceA serviceA, 
        IServiceB serviceB,
        SomeUnityComponent component)
    {
        // ...
    }
```

Note:  
If class has multiple constructors, the one with `[Inject]` has priority.

**Recommendation:**  
Use Constructor Injection whenever possible.
The constructor & readonly field idiom is:
- The instantiated object has a compiler-level guarantee that the dependencies have been resolved.
- No magic in the class code. Instantiate easily without DI container. (e.g. Unit testing)
- If you look at the constructor, the dependency is clear.
  - If too many constructor arguments, it can be considered overly responsible.

### Method Injection

If constructor injection is not available, use method injection.

Typically this is for MonoBehaviour.

```csharp
public class SomeBehaviour : MonoBehaviour
{
    float speed;

    [Inject]
    public void Construct(GameSettings settings)
    {
        speed = settings.speed;
    }
} 
```

**Recommendation:**  
Consider whether injection to MonoBehaviour is necessary.  
In a code base where domain logic and presentation are well decoupled, MonoBehaviour should act as a View component.

In my opinion, View components should only be responsible for rendering and should be flexible.

Of course, In order for the View component to work, it needs to pass state at runtime.  
But the "state" of an object and its dependence on the functionality of other objects are different.  
It's enough to pass the state as arguments instead of `[Inject]`.

### Property / Field Injection

If the object has a local default and Inject is optional,  
Property/Field Injection can be used.

```csharp
class ClassA
{
    [Inject]
    IServiceA serviceA { get; set; } // Will be overwritten if something is registered. 

    public ClassA()
    {
        serviceA = ServiceA.GoodLocalDefault;
    }        
}
```

You can also use field.

```csharp
    [Inject]
    IServiceA serviceA;
```

### Implicit Relationship Types

VContainer supports automatically resolving particular types implicitly to support special relationships.

#### `IEnumerable<T>` / `IReadonlyLIst<T>`

Duplicate registered interfaces can be resolved together with IEnumerable<T> or IReadOnlyList<T>.

```csharp
builder.Register<IDisposable, A>(Lifetime.Scoped);
builder.Register<IDisposable, B>(Lifetime.Scoped);
```

```csharp
class ClassA
{
    public ClassA(IEnumerable<IDisposable> disposables) { /* ... */ }
}
```

OR

```csharp
class ClassA
{
    public ClassA(IReadOnlyList<IDisposable> disposables) { /* ... */ }
}
```

Note:  
This is mainly used by internal functions such as `ITickable` marker etc.

## Controlling Object Lifetime

`LifetimeScope` can build parent-child relationship.

- If registered object is not found, `LifetimeScope` will look for a parent `LifetimeScope`.
- For `Lifetime.Singleton`
  - Parent and child cannot register the same type.
  - Always returns the same instance.
- For `LifeTime.Transient`
  - If parent and child have the same type, child will prioritize itself.
  - Instance will be different for each child.
  - Instance creating for each resolving..
- For `Lifetime.Scoped`:
  - If same child returns same instance.
  - Instance will be different for each child.
  - When a `LifetimeScope` is destroyed, objects with `IDisposable` implemented are called `Dispose()`.

:warning: If scene is alive and only LifetimeScope is destroyed, MonoBehaviour registered as `Lifetime.Scoped` is not automatically destroyed.
If you want to destroy with `LifetimeScope`, make it a child transform of `LifetimeScope` or consider implement IDisposable.

### Create child LifetimeScope

LifetimeScope can create child at runtime.

```csharp
var childLifetimeScope = lifetimeScope.CreateChild(builder => 
{
    builder.Register<ExtraClass>(Lifetime.Scoped);
    builder.RegisterInstance(extraInstance);
    // ...
});
```

```csharp
// It will be calling registered `IDisposable.Dispose()`.
UnityEngine.Object.Destroy(childLifetimeScope.gameObject);
```

You can also use installer.

```csharp
var childLifetimeScope = lifetimeScope.CreateChild(yourInstaller);
```

##



:warning: If

### Set parent

##

### Pass

```csharp
using (LifetimeScope.Push(builder => 
{
    builder.RegisterInstance(extraInstance);
    builder.Register<ExtraType>(Lifetime.Scoped); 
}))
{
    var loading = SceneManager.LoadSceneAsync("NextScene");
    while (!loading.isDone)
    {
        yield return null;
    }
}
```

```csharp
using (LifetimeScope.Push(builder => {
    builder.RegisterInstance(extraInstance);
    builder.Register<ExtraType>(Lifetime.Scoped); 
}))
{
    await SceneManager.LoadSceneAsync("NextScene");
}

```


## Dispatching Unity Lifecycle

VContainer has own PlayerLoop sub systems.

If you register a class that implements the marker interface, it will be scheduled in Unity's PlayerLoop cycle.

The following interfaces and timings are available.

- `IInitializable`     : Nearly `Start`
- `IPostInitializable` : After `Start`
- `IFixedTickable`     : Nearly `FixedUpdate`
- `IPostFixedTickable` : After `FixedUpdate`
- `ITickable`          : Nearly `Update`
- `IPostTickable`      : After `Update`
- `ILateTickable`      : Nearly `LateUpdate`
- `IPostLateTickabl`   : After `LateUpdate`

![](./docs/lifecycle_diagram.png)

## Comparing VContainer to Zenject

Zenject tries to create any object through Zenject, but VContainer doesn't think so.
VContainer thinks that it is enough to inject only functional dependency to the outside.

wip

 | Zenject                               | VContainer                                |
 |:--------------------------------------|:------------------------------------------|
 | Container.Bind\<Service\>()<br>&nbsp;&nbsp;&nbsp;&nbsp;.AsTransient() | builder.Register\<Service\>(Lifetime.Transient) |
 | Container.Bind\<Service\>()<br>&nbsp;&nbsp;&nbsp;&nbsp;.AsCached() | builder.Register\<Service\>(Lifetime.Scoped) |
 | Container.Bind\<Service\>()<br>&nbsp;&nbsp;&nbsp;&nbsp;.AsSingle() | builder.Register\<Service\>(Lifetime.Singleton) |
 | Container.Bind\<IService\>()<br>&nbsp;&nbsp;&nbsp;&nbsp;.To\<Service\>()<br>&nbsp;&nbsp;&nbsp;&nbsp;.AsCache() | builder.Register\<IService, Service\>(Lifetime.Scoped) |
 | Container.Bind(typeof(IInitializable), typeof(IDisposable))<br>&nbsp;&nbsp;&nbsp;&nbsp;.To\<Service\>()<br>&nbsp;&nbsp;&nbsp;&nbsp;.AsCached(); | builder.Register\<Service\>(Lifetime.Scoped)<br>&nbsp;&nbsp;&nbsp;&nbsp;.As\<IInitializable, IDisposable\>() |
 | Container.BindInterfacesTo\<Service\>()<br>&nbsp;&nbsp;&nbsp;&nbsp;.AsCached() | builder.Register\<Service\>(Lifetime.Scoped)<br>&nbsp;&nbsp;&nbsp;&nbsp;.AsImplementedInterfaces() |
 | Container.BindInterfacesAndSelfTo\<Foo\>()<br>&nbsp;&nbsp;&nbsp;&nbsp;.AsCached()| builder.Register\<Service\>(Lifetime.Scoped)<br>&nbsp;&nbsp;&nbsp;&nbsp;.AsImplementedInterfaces()<br>&nbsp;&nbsp;&nbsp;&nbsp;.AsSelf() |
 | Container.BindInstance(obj) | builder.RegisterInstance(obj) |
 | Container.Bind\<IService\>()<br>&nbsp;&nbsp;&nbsp;&nbsp;.FromInstance(obj) | builder.RegisterInstance\<IService\>(obj) |
 | Container.Bind(typeof(IService1), typeof(IService2))<br>&nbsp;&nbsp;&nbsp;&nbsp;.FromInstance(obj) | builder.RegisterInstance(obj)<br>&nbsp;&nbsp;&nbsp;&nbsp;.As\<IService1, IService2\>() |
 | Container.BindInterfacesTo\<Service\>()<br>&nbsp;&nbsp;&nbsp;&nbsp;.FromInstance(obj) | builder.RegisterInstance(obj)<br>&nbsp;&nbsp;&nbsp;&nbsp;.AsImplementedInterfaces() |
 | Container.BindInterfacesAndSelfTo\<Service\>()<br>&nbsp;&nbsp;&nbsp;&nbsp;.FromInstance(obj) | builder.RegisterInstance(obj)<br>&nbsp;&nbsp;&nbsp;&nbsp;.AsImplementedInterfaces()<br>&nbsp;&nbsp;&nbsp;&nbsp;.AsSelf() |


 Component Registration

  | Zenject                               | VContainer                                |
  |:--------------------------------------|:------------------------------------------|
  | Container.Bind\<Foo\>()<br>&nbsp;&nbsp;&nbsp;&nbsp;.FromComponentInHierarchy()<br>&nbsp;&nbsp;&nbsp;&nbsp;.AsCached(); | builder.RegisterComponentInHierarchy\<Foo\>() |
  | Container.Bind\<Foo\>()<br>&nbsp;&nbsp;&nbsp;&nbsp;.FromComponentInNewPrefab(prefab)<br>&nbsp;&nbsp;&nbsp;&nbsp;.AsCached()<br>&nbsp;&nbsp;&nbsp;&nbsp; | builder.RegisterComponentInNewPrefab(prefab, Lifetime.Scoped)
  | Container.Bind\<Foo\>()<br>&nbsp;&nbsp;&nbsp;&nbsp;.FromNewComponentOnNewGameObject()<br>&nbsp;&nbsp;&nbsp;&nbsp;.AsCached()<br>&nbsp;&nbsp;&nbsp;&nbsp;.WithGameObjectName("Foo1") | builder.RegisterComponentOnNewGameObject\<Foo\>(Lifetime.Scoped, "Foo1") |
  | Container.Bind\<Foo\>()<br>&nbsp;&nbsp;&nbsp;&nbsp;.FromComponentInNewPrefabResource("Some/Path/Foo") | **We should load Resources using the `LoadAsync` family**<br>**You can use RegisterInstance() after loading the Resource ** |
  | .UnderTransform(parentTransform) | .UnderTransform(parentTransform) |
  | .UnderTransform(() => parentTransform) | .UnderTransform(() => parentTransform) |


## Best Practices and Recommendations

wip

## License

 MIT