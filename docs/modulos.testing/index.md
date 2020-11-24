# What is Modulos.Testing ? 
Modulos.Testing is a simple library built to improve integration and unit testing. It also brings a bunch of battled proven patterns for better efficiency during testing process.  


## Prerequisites 
In this documentation and included samples, `xUnit` framework is in use. Basic knowledge on this area is required. 

# Test environemnt 
Below diagram shows simplified schema of how a test environment works.  
```console
+------------------------------------+
| TestEnvironment                    |
|                                    |
|  +---------------------------+     |
|  |                           |     |    +-----------------+
|  |  Setup test environment   |     |    |                 |
|  |  using Blocks (pipeline)  |     |    |  Create test    |
|  |                           |     +--->+   (ITest)       |
|  +------------+--------------+     |    |                 |
|               |                    |    +--------+--------+
|   +-----------+---------------+    |             |
|   |                           |    |             |
|   | Build a test environment  |    |             v
|   | (blocks are executed)     |    |    +--------+----------------+
|   |                           |    |    | Execute functionalities |
|   +---------------------------+    |    | to test.                |
|                                    |    | (resolve it from test)  |
+------------------------------------+    +---------+---------------+
                                                    |
                                                    |
                                            +-------+-----------+
                                            |   Assert results  |
                                            +-------------------+

```

- Test environment is a class used to create tests. 
- **Test environment is built with blocks.** 
- Blocks are executed in a pipeline, one after another.
- Lifetime of the test environment may be:
    - test method
    - single test class 
    - multiple test classes
- Created test are child-scoped instance of the `ITest` interface 

Test environment is represented by `ITestEnvironment` interfce.

```csharp 
/// <summary>
/// Lifetime of this object should be managed by test framework like a xUnit, NUnit.
/// </summary>
public interface ITestEnvironment : IAsyncDisposable
{
    Task<Test> CreateTest(Action<TestOptions> optionsModifier = null);
    Task<ITestEnvironment> Build(params object[] data);
    void SetServiceProvider(IServiceProvider provider);
    IServiceProvider GetServiceProvider();

    // methods to manipulate blocks (Add, Remove, Update, ect.)
    (...)
```

## Dependency injection

**Using DI container is part of Modulos.Testing proposal pattern.**

The below diagram shows how DI container is set and consumed by the test environment.
```console
+------------------------+       +-----------+
|                        +------>+ Building  |
|  Test environment      |       | process   |
|  +-----------------+   |       +----+------+
|  |                 |   |            |
|  | Dependency      |   |            v
|  | Injection       |   |    +-------+-----------+
|  | Container       +<-------+SetServiceProvider |
|  +-------+---------+   |    +-------------------+
|          |             |
+------------------------+
           |
     +-----V-------+
     |CreateTest() |
     |             |
     +------+------+
            |
            v
  +---------+-----------+
  |  ITest is child     |
  |  scope of test      |
  |  environment        |
  |  container          |
  +---------------------+
```

- use `SetServiceProvider` from block to setup DI container for test environment 
- if few blocks call `SetServiceProvider` then the last one win
- test environment uses dependency injection to control lifecycle 
- each of the created test is child scope of test environment container
- after `SetServiceProvider` is called, any further block can resolve dependencies from the DI container
- even if setting up DI container is not required - **for most of the cases is strongly recommended** 


# Defining and building test environment
**Test environment is built with blocks.**  
```csharp
await using var env = new TestEnvironment();
env.Add<InitializeIoc>(); // block
env.Add<DtopAndCreateDb>(); // block
await env.Build();
```
## Blocks
Think about blocks as workers, created and executed to prepare and clean up tests environment.

- Block is represented by `IBlock` interfce.  
```csharp
    public interface IBlock
    {
        // executes during building 
        Task<BlockExecutionResult> Execute(ITestEnvironment testEnv);

        // executes during disposal of test environment 
        Task ExecuteAtTheEnd(ITestEnvironment testEnv);
    }
```

- Blocks are **defined by developers**, for example, the mentioned `InitializeIoc` is not available from a `Modulos.Testing` package. Anyway, feel free to analyze and copy implementation from available examples.  
- Blocks can be add or manipulate using `ITestEnvironment` instance. 


## Setup block 
Block may be updated during the configuration process.

```csharp
// during add 
env.Add<SampleBlock>(block =>
{
    block.SampleBlockProperty  = "ulalla";
});

// during update 
and.Update<SammpleBlock>(block =>
{
    block.SampleBlockProperty  = "ulalla";
});
```

# Creating test
The prepared and built test environment can create tests. It's done by calling `CreateTest` method.

```csharp
(...)
await using ITest test = await env.CreateTest();
```
Created test is represented by `ITest` interface.

```csharp
public interface ITest : IServiceScope, IServiceProvider, IAsyncDisposable
{
    (...)
}
```
The simplest explanation of `ITest` is to say that:  

 - it's an execution scope for the test (controls lifecycle)
- because it's child scope of test environment DI container, it provides registered data

```csharp
await using var test = await env.CreateTest()
{
    // obtain registered dependencies 
    var functionality = test.Resolve<SomeDependency>();
}
// test is disposed, scope is disposed
// (brackets are no required)
```

# Wrappers
Tests can be wrapped by `ITestWrapper`. 

```csharp
public interface ITestWrapper
{
    Task Begin();
    Task Finish();
}
```
`Begin` method is called just after the test is created, `Finish` during the disposal process. Test wrappers can consume registered dependencies from the same scope as a test (ctor injection). 

## Defining wrappers

- It's possible to configure wrapper directly from the test environment. In this situation, each of the created tests will be wrapped.

```csharp 
public class CustomEnvironment : DefaultEnvironment
{
    public CustomEnvironment()
    {
        (...)
        // each test will be wrapped with EmptyTestWrapper
        Wrap<EmptyTestWrapper>();
    }
}
```

- During test creation using `CreateTest` method. In this situation, only a particular invocation will be wrapped.

```csharp
await using var test = await testEnvironment
    .CreateTest(options =>
    {
        options.Wrap<TestWrapper>();
    });
```

# Study case - simplest
**Let's write some integration test.**
```csharp
[Fact]
public async Task GetUserById_test()
{
    // Arrange
    await using var env = new TestEnvironment();
    env.Add<InitializeIoc>(block =>
    {
        block.RegisterServices = collection =>
        {
            collection.AddSingleton<IUserRepository, DbUserRepository>();
            collection.AddTransient<IGetUserById, GetUserById>();
        };
    });
    await env.Build();
    await using var test = await env.CreateTest();
    var functionality = test.Resolve<IGetUserById>();

    // Act
    var result = functionality.Execute(0);

    // Assert
    result.Should().NotBeNull();
    result.Name.Should().Be("Tom");
}

public class GetUserById : IGetUserById
{
    public GetUserById(IUserRepository userStorage)
    (...)
}
```

And now **unit test** with mocked repository. It's done by replacing **one line**.

*this*
```csharp
collection.AddSingleton<IUserRepository, DbUserRepository>();
```
*into this*
```csharp
collection.AddSingleton<IUserRepository, MockedUserRepository>();
```

**You can create integration and unit tests with the same pattern, it only dependes of registration (real object or mocked).** Maybe it does not look like a great piece of art, but in fact, it is a really nice feature. It's possible to create a shared codebase for unit and integration tests. 


# Controlling environment lifetime (xUnit)
In the currently presented examples test environments were created for each of the tests, This approach is generally acceptable, but for various scenarios non needed. For example, if you prepared an environment for some integration test, there is a lot of chance it's fittable to more than one test. In fact, it's not a bad idea to create one environment shared between dozens or even hundreds of tests.


Let's define some custom environments and use `xUnit` to consume and controll its lifetime.

*Define environment*
```csharp
public class CustomEnvironment: TestEnvironment, 
    IAsyncLifetime  // xUnit
{
    public CustomEnvironment()
    {
        Add<InitializeIoc>(block =>
        {
            block.AddTransient<IGetUserById, GetUserById>();
            block.AddTransient<IUserRepository, MockedUserRepository>();
        });
    }

    async Task IAsyncLifetime.InitializeAsync() // xUnit (IAsyncLifetime)
    {
        await Build();
    }

    async Task IAsyncLifetime.DisposeAsync() // xUnit (IAsyncLifetime)
    {
        await DisposeAsync();
    }
}
```
*Consume it, in this example one environment for a class*

```csharp
public class TestClass : IClassFixture<CustomEnvironment> // xUnit 
{
    private readonly CustomEnvironment env;

    public TestClass(CustomEnvironment env)
    {
        this.env = env;
    }

    [Fact]
    public async Task GetUserById_test()
    {
        // Arrange
        await using var test = await env.CreateTest();
        var functionality = test.Resolve<IGetUserById>();

        // Act
        var result = functionality.Execute(0);

        // Assert
        result.Should().NotBeNull();
        result.Name.Should().Be("Tom");
    }
```

*You can either use the same environment in many classes using xUnit collection functionality.*

```csharp
[CollectionDefinition(nameof(CustomEnvironment))]
public class CustomEnvironmentCollection:ICollectionFixture<CustomEnvironment>
{
}
```

*The same environment in many classes.*
```csharp
[Collection(nameof(CustomEnvironment))]
public class TestClass : IClassFixture<CustomEnvironment>
{
    public TestClass(CustomEnvironment env)
    {
        this.env = env;
    }
(...)

[Collection(nameof(CustomEnvironment))]
public class TestClass2 : IClassFixture<CustomEnvironment>
{
    public TestClass2(CustomEnvironment env)
    {
        this.env = env;
    }
(...)
```

**This approach may result in significant performance improvements.**

# Environment inheritance 
**Inherit from test environments is part of Modulos.Testing pattern.** This approach is an essence of reconfigurable and reusable test environments. It was the purpose to write this library. 

Let's see how it works. 

<p style="font-weight: bold;">1.Definitions of common blocks, let's say it's from nuget package.</p>

```csharp
public sealed class InitializeIoc : IBlock, IServiceCollection
{
    private readonly IServiceCollection internalCollection;

    public InitializeIoc()
    {
        internalCollection = new ServiceCollection();
    }

    Task<BlockExecutionResult> IBlock.Execute(ITestEnvironment testEnv)
    {
        var collection = new ServiceCollection();

        foreach (var serviceDescriptor in internalCollection)
        {
            collection.Add(serviceDescriptor);
        }

        var provider = collection.BuildServiceProvider();

        testEnv.SetServiceProvider(provider);

        return Task.FromResult(BlockExecutionResult.EmptyContinue);
    }

    Task IBlock.ExecuteAtTheEnd(ITestEnvironment testEnv)
    {
        ((IServiceCollection)this).Clear();
        return Task.CompletedTask;
    }

    (...) // 'delegate' implementation of IServiceCollection (internalCollection)
}

public sealed class CreateDatabase : IBlock
{
    public bool DropDbAtTheEnd { get; set; } = true;
    public bool RecreateDbAtStart { get; set; } = true;
    (...)
} 

public sealed class Cleanup : IBlock
{
    (...)
} 
```
<p style="font-weight: bold;">2. Default environment from nuget package.</p>

```csharp
public class DefaultEnvironment : TestEnvironment, IAsyncLifetime
{
    public DefaultEnvironment()
    {
        Add<InitializeIoc>();
        Add<CreateDatabase>();
        Add<Cleanup>();
    }

    async Task IAsyncLifetime.InitializeAsync()
    {
        await Build();
    }

    async Task IAsyncLifetime.DisposeAsync()
    {
        await DisposeAsync();
    }
}
```
<p style="font-weight: bold;">3. At last inheritance with updates.</p>

```csharp
public class MyCustomEnvironment : DefaultEnvironment
{
    public MyCustomEnvironment()
    {
        Update<InitializeIoc>(block =>
        {
            // register additional services for MyCustomEnvironment
            block.AddTransient<IUserRepository, InMemoryUserRepository>();
        });

        // change behavior of CreateDatabase block
        Update<CreateDatabase>(block =>
        {
            block.DropDbAtTheEnd = false;
        });
    }
}
```

# Modulos.Testing Pattern (MTP)
- Prepare common blocks and environments to share them between projects, teams, solutions.
- It's a good idea for the test environment to inherit from the existing one
- Use mocks as dependency injection registrations 

**Give a chance for this pattern and soon it's going to be one of your favorite.**

# Examples
Examples are available directly in 'Modulos.Testing` project [github](https://github.com/adobrzyc/modulos.testing/tree/master/examples).