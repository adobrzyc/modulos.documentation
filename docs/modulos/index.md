# What is Modulos ? 
It's a very simple library that brings a proposal on how to thread application configuration process. With Modulos you can share functionalities over projects/solutions/organizations via modules that affects application in predictible way. It's also usefull to split single application into more readable parts. 

Besides even if you don't like the whole library ;( you still can use some of the exposed functionalities (e.q. pipelines).

# Pipelines
Modulos uses in many places the concept of pipelines, it may be useful if you take a look into [pipeline documentation](modulos/pipeline/index.md)

# Study case
Let's say we prepared functionality to collect the history of CRUD operations and decided to share this using NuGet package.  

To use this functionality it's needed to:
- read some data from configuration to decide about something
- call `AddHistory(options)` with correct options depends on configuration 
- do some actions requires registered dependencies  

Code may looks like listed below.

```csharp
public IConfiguration Configuration { get; }
public Startup(IConfiguration configuration)
{
    Configuration = configuration;
}

public void ConfigureServices(IServiceCollection services)
{
    if (Configuration["history.storage"] == "EF")
    {
        services.AddHistory(options=>{
            options.UseEf();
        });
    }
    else
    {
          services.AddHistory(options=>{
            options.UseInMemory();
        });
    }
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    app.UseHistory(env);
}
```

**It's nothing wrong with this code.** But let's do it for... then more functionalities and use some of third party components. Startup class usually ends with houndreds, sometimes thousand lines of code. Lets do this in all organization projects and propably ends with **dozens startup classes**. Then finally some changes arrives with needs to be maintained in startup class, for every of this dozens projects, for every integration test with their startup class. **This is a place where Modulos came.** And simplify this process to change this behaviour in one place (source of nuget/project/library) and update dependency in another.

```csharp
private readonly ModulosApp modulos = new ModulosApp();
public Startup(IConfiguration configuration)
{
    // executes initialization pipeline
    modulos.Initialize<Startup>(configuration);
}

public void ConfigureServices(IServiceCollection services)
{
    // configure dependecy injection via modules 
    services.AddModulos(modulos); 
    (...)
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env, IHostApplicationLifetime lifetime)
{
    // executes configuration pipeline
    modulos.Configure(app.ApplicationServices);
    (...)
}
```

# MAP - Modulos Application Pipeline
As you can notice Modulos affects the application configuration process in three areas.  

1. **Initialization (INI)**  
    It's a place to prepare Modulos and things like.: configurations, settings, ect.
2. **Configure dependecy injection (CDI)**   
    Its a place to configure dependency injection. 
3. **Configure application (CA)**  
    It's a place to configure application. 


Modulos modularize those areas. That means INI, CDI, and CA are built with some modules. Even if each block is modularized in its own way, then finally it ends with the composition of classes executed one after another.  

**The biggest benefit of Modulos** is that those modules can (don't must) share some data, and there is a common way to affects those modules even from external dependencies (packages, libraries, projects).


# Initialization (INI)
<p style="font-size:1.2em;">How to run</p>

```csharp
var modulosApp = new ModulosApp();
// decide to use NetCore integration
modulosApp.UseNetCore(); 
// execute initialization pipeline
modulosApp.Initialize<ClassFromMainProject>();
```
<p style="font-size:1.2em;">How to affect</p>
Initialization is build with pipeline pattern. Pipeline can be modified directly from 
`Initialize` method or by class inherited from `ModulosApp.IUpdateInitializationPipeline`.
Classes are auto explored and executed during `Initialize` method invocation.

```csharp
public IPipelineResult Initialize<TClassInProject>
(
    // can be used to inline edit pipeline
    Action<IPipeline> updatePipeline, 
    // if you wants to give access to some extra data for pipes
    params object[] additionalParameters

) where TClassInProject : class
```
```csharp
// simple 
modulosApp.Initialize<Startup>(); 

//with some inline changes 
modulosApp.Initialize<Startup>(update => 
{
    update.Add<SomeNewPipe>();
});
```
and/or
```csharp
public class UpdateInitializationPipeline : ModulosApp.IUpdateInitializationPipeline
{
    public void Update(IPipeline pipeline)
    {
        pipeline.Add<SomeNewPipe>();
    }
}
```

Pipes can be defined as simple as inheritance from `IPipe or IOptionalPipe` interfaces. 
```csharp
public class SomeNewPipe : IPipe
{
    public Task<PipeResult> Execute(CancellationToken cancellationToken)
    {
        return Task.FromResult(PipeResult.Continue);
    }
}
```
<p style="font-size:1.2em;">Recomendations and tips</p>
- Use `IUpdateInitializationPipeline` with your libraries to modularize the initialization process. 
- There are various methods to modify pipeline. 
- For better understanding pipeline or initialization process look at available examples.

# Configure dependecy injection (MAP)
<p style="font-size:1.2em;">How to run</p>

```csharp
//AddModulos(...) will explore and load any available module in application
collection.AddModulos(modulosApp);
```
<p style="font-size:1.2em;">How to affect</p>

Modulos brings modules (very similar to Autofac) even for the default Microsoft DI container. Main difference with similar solutions is possibility to order and filter these modules. It's also worth to note that dependecy injection modules (DIM) can easly access to data from initialization process (or/and some extra inline data). 

<p style="font-size:1.2em;">Sample module</p>

```csharp
public class RegisterStorageModule : MicrosoftDiModule
{
    public override LoadOrder Order { get; } = LoadOrder.Project;
    public override bool AutoLoad { get; } = true;

    // config is available from initialization pipeline 
    private readonly IConfiguration config;

    public RegisterStorageModule(IConfiguration config)
    {
        this.config = config;
    }

    public override void Load(IServiceCollection services)
    {
        // it's possible to consume those available data to perform some actions
        if (config["Storage"] == "InMemory")
        {
            services.AddTransient<IStorage, InMemoryStorage>();
            services.AddTransient<InMemoryStorage, InMemoryStorage>();
        }
        else
        {
            services.AddTransient<IStorage, FileStorage>();
            services.AddTransient<FileStorage, FileStorage>();
        }
    }
}
```
**And thats all.** Define modules whenever in application, packages, or other dependencies.

<p style="font-size:1.2em;">Ordering</p>

As you may notice module has defined property `Order`. It's `enum` listed below and it's used
to control ordering process during loading modules. By choosing higher-order you may overwrite previous registration. 

```csharp
public enum LoadOrder
{
    // Reserved for elements located in external libraries.
    Library,

    // Reserved for elements located in solution projects.
    Project,

    // Reserved for elements located in application project (eq.: console app, web api).
    App,

    // Reserved for elements located in test projects.
    Test
}
```

<p style="font-size:1.2em;">Filtering</p>
Filtering is available during `AddModulos` method invocation.
```csharp
sc.AddModulos(modulosApp, module =>
{
    if(module.Instance is RegisterStorageModule)
        module.AutoLoad = false;
});
```
<p style="font-size:1.2em;">Using additional data </p>

Each of module can use passed into `AddModulos` method extra parameters.
```csharp
collection.AddModulos(modulosApp, someExtraData1, someExtraData2);
```

It's not a bad idea to put here data from the initialization process.

```csharp
var iniResult = modulosApp.Initialize<Program>();

sc.AddModulos
(
    modulosApp, 
    // data from initialization pipeline, will be available for DI modules
    iniResult.GetAll() 
);
```

By default each module may obtain (via ctor) below components:
- `ITypeExplorer`
- `IAssemblyExplorer`
- `IAppInfo`
- `Assembly[]`

<p style="font-size:1.2em;">FAQ </p>

- Can I still use the 'standard' way for dependency injection configuration 
  - Yes you can, either mix them.
- Can I use external dependency injection containers like Autofac.
  - Yes you can. Modulos has also some integrations packages (e.q: Autofac).

# Configure application (CA)
Configuration is very similar to initialization, it's built with a pipeline, accept 
additional data, and can use results from previously executed pipes. The only difference 
is that configuration process may use a dependency injection container to obtain data (if
not available from `additionalParameters`). 

```csharp
IPipelineResult Configure
(
    IServiceProvider serviceProvider,
    Action<IPipeline> updatePipeline, 
    params object[] additionalParameters
)
```

<p style="font-size:1.2em;">Pipeline </p>
To modifie pipeline use:
- `ModulosApp.IUpdateConfigPipeline` 
- `Action<IPipeline> updatePipeline`



# Example
For more examples please explore repository (Examples directory).

```csharp
class Program
{
    static void Main()
    {
        // 1. initialize
        var modulosApp = new ModulosApp();
        modulosApp.UseNetCore();
        modulosApp.Initialize<Program>();


        // 2. organize dependency injection 
        var sc = new ServiceCollection();
        sc.AddModulos
        (
            modulosApp, 
            // data from initialization pipeline, will be available for DI containers
            iniResult.GetAll() 
        );
        var sp = sc.BuildServiceProvider();


        // 3. configure after dependency injection 
        modulosApp.Configure(sp);
    }
}
```

**output**
```bat
PrepareConfiguration...
MakeSomeActionBaseOnConfiguration...
[Storage, InMemory]
[AppVersion, 1.0.0]
ConfigureAppWhenInMemoryStorage...
InMemoryStorage
```

**'application components'**

```csharp
//
// Initialization: it can be delivered event from external package
// 
public class UpdateInitializationPipeline : ModulosApp.IUpdateInitializationPipeline
{
    public void Update(IPipeline pipeline)
    {
        pipeline.Add<PrepareConfiguration>();
        pipeline.Add<MakeSomeActionBaseOnConfiguration>();
    }
}

public class PrepareConfiguration : IPipe
{
    public Task<PipeResult> Execute(CancellationToken cancellationToken)
    {
        Console.WriteLine("PrepareConfiguration...");

        var builder = new ConfigurationBuilder();
        builder.Add(new MemoryConfigurationSource
        {
            InitialData = new []
            {
                new KeyValuePair<string, string>("AppVersion","1.0.0"),
                new KeyValuePair<string, string>("Storage","InMemory")
            }
        });
        var config = builder.Build();

        var result = new PipeResult(PipeActionAfterExecute.Continue, config);

            
        return Task.FromResult(result);
    }
}

public class MakeSomeActionBaseOnConfiguration : IPipe
{
    // pipes can use previous pipes data 
    private readonly IConfiguration config;

    public MakeSomeActionBaseOnConfiguration(IConfiguration config)
    {
        this.config = config;
    }

    public Task<PipeResult> Execute(CancellationToken cancellationToken)
    {
        Console.WriteLine("MakeSomeActionBaseOnConfiguration...");
        foreach (var pair in config.AsEnumerable())
        {
            Console.WriteLine(pair);
        }
            
        return Task.FromResult(PipeResult.Continue);
    }
}

//
// DI modules
// 
public class RegisterStorageModule : MicrosoftDiModule
{
    public override LoadOrder Order { get; } = LoadOrder.Project;
    public override bool AutoLoad { get; } = true;

    // config is available from initialization pipeline 
    private readonly IConfiguration config;

    public RegisterStorageModule(IConfiguration config)
    {
        this.config = config;
    }

    public override void Load(IServiceCollection services)
    {
        if (config["Storage"] == "InMemory")
        {
            services.AddTransient<IStorage, InMemoryStorage>();
            services.AddTransient<InMemoryStorage, InMemoryStorage>();
        }
        else
        {
            services.AddTransient<IStorage, FileStorage>();
            services.AddTransient<FileStorage, FileStorage>();
        }
    }
}

public interface IStorage {}

public class InMemoryStorage : IStorage
{
    public override string ToString()
    {
        return GetType().Name;
    }
}

public class FileStorage : IStorage
{
    public override string ToString()
    {
        return GetType().Name;
    }
}

//
// Configuration
// 
public class UpdateConfigPipeline : ModulosApp.IUpdateConfigPipeline
{
    public void Update(IPipeline pipeline)
    {
        pipeline.Add<ConfigureAppWhenInMemoryStorage>();
        pipeline.Add<ConfigureAppWhenFileStorage>();
    }
}

// pipes can be optional, created and executed only if all params in ctor are available 
public class ConfigureAppWhenInMemoryStorage : IOptionalPipe
{
    private readonly InMemoryStorage storage;

    public ConfigureAppWhenInMemoryStorage(InMemoryStorage storage)
    {
        this.storage = storage;
    }

    public Task<PipeResult> Execute(CancellationToken cancellationToken)
    {
        Console.WriteLine($"{GetType().Name}...");
        Console.WriteLine(storage.ToString());
        return Task.FromResult(PipeResult.Continue);
    }
}

// this pipe will not be created either executed, because FileStorage is not available
public class ConfigureAppWhenFileStorage : IOptionalPipe
{
    private readonly FileStorage storage;

    public ConfigureAppWhenFileStorage(FileStorage storage)
    {
        this.storage = storage;
    }

    public Task<PipeResult> Execute(CancellationToken cancellationToken)
    {
        Console.WriteLine($"{GetType().Name}...");
        Console.WriteLine(storage.ToString());
        return Task.FromResult(PipeResult.Continue);
    }
}
```
