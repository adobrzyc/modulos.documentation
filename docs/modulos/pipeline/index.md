### Idea
A pipeline is an idea to consume a defined block of codes one after another. 
It's also a common issue to share data between executed blocks and return 
result at the end.

```
+--------------------------------------------------+
|                                                  |
|  +-------------+     PIPE1,data1                 |
|  | PIPE1       +---------------------+           |
|  +-------------+                     |           |
|       |                              |           |
|       | PIPE1,data1                  |           |
|       v                    RESULTS   |           | 
|  +-------------+  PIPE2    +---------v--------+  |
|  | PIPE2       +---------->+                  |  |
|  +-------------+           | PIPE1 - instance |  |
|    |                       | data1            |  |
|    | PIPE2, PIPE1          | PIPE2 - instance |  |
|    v data1                 | PIPE3 - instance |  |
|  +-------------+ PIPE3     | data3            |  |
   |             | data3     |                  |  | 
|  | PIPE3       +---------->+                  |  |
|  +-------------+           |                  |  |
|                            +------------------+  |
|                                      |           |
+----------------------+---------------+-----------+
                       |
                       | EXECUTE(..)
                       v
            +----------+------+
            |  RESULTS        |
            +-----------------+
```

### Implementation
A pipeline is represented by `Pipeline` class (`IPipeline` interface). The pipe must inherit 
from `IPipe` or `IOptionalPipe`.

### Defining pipeline
Let's look at the simples possible example. 
```csharp
var pipeline = new Pipeline();
pipeline.Add<Pipe1>();
pipeline.Add<Pipe2>();
await pipeline.Execute(CancellationToken.None);
```
And the pipes definitions.
```csharp

class Pipe1 : IPipe
{
    public ValueTask<PipeResult> Execute(CancellationToken cancellationToken)
    {
        return new ValueTask<PipeResult>(PipeResult.Continue);
    }
}

class Pipe2 : IPipe
{
    public ValueTask<PipeResult> Execute(CancellationToken cancellationToken)
    {
        return new ValueTask<PipeResult>(PipeResult.Continue);
    }
}
```

It's worth to mention that `IPipeline` defines a lot of methods to manipulate pipelines
such as `Remove, AddOrReplace, Insert` and many more.

### Available data 
It's possible to pass into `Pipeline` class some `additional data`.
It's worth to mention that `Pipeline` class accepts `IServiceProvider`
in its constructor. `IServiceProvider' is used in case of not found required 
dependency in pipeline additional data or pipes results. 

### Passing data between pipes
Each executed pipe and its results are available for further pipes 
as simple as parameter in constructor (ctor injection). 

```csharp
class Pipe2 : IPipe
{
    public Pipe2(Pipe1 pipe1) // that's ok Pipe1 was executed 
    {
    } 
    (...)
}
```

During pipe execution process it's possible to pass some extra data.
```csharp
class Pipe1 : IPipe
{
    public string PipeData => "Pipe1 data";

    public ValueTask<PipeResult> Execute(CancellationToken cancellationToken)
    {
        var result = PipeResult.NewContinue("some extra data"); //extra data 
        return new ValueTask<PipeResult>(result);
    }
}
```

**Some facts**
- Each pipe overwites result of the same type.
- Pipeline `additional data` and pipes results are the first choice. Only if not found `IServiceProvider` is used.

### Optional pipes
Pipes can be defined as `IPipe` or `IOptionalPipe`. The `IPipe` in case of 
being unable to resolve its references will throw an exception. On the other hand, 
`IOptionalPipe` will not throw an exception and also 
**will try to execute after any further pipe execution**. 

It's a very simple, but powerful concept. It's possible to define pipes that will 
be waiting till proper data become available. 
```csharp

var pipeline = new Pipeline();
pipeline.Add<OptionalPipe>();
pipeline.Add<Pipe1>();
pipeline.Add<Pipe2>();
pipeline.Add<Pipe3>();
var result = await pipeline.Execute(CancellationToken.None);
result.GetOptional<OptionalPipe>().Should().NotBeNull();
//that's because OptionalPipe breaks pipeline 
result.GetOptional<Pipe3>().Should().BeNull();

class OptionalPipe : IOptionalPipe
{
    // until pipe2 will not show up OptionalPipe won't be created and executed
    public OptionalPipe(Pipe2 pipe2)
    {
        pipe2.Should().NotBeNull();
    }

    public ValueTask<PipeResult> Execute(CancellationToken cancellationToken)
    {
        return new ValueTask<PipeResult>(PipeResult.Break);
    }
}

class Pipe1 : IPipe
{
    public ValueTask<PipeResult> Execute(CancellationToken cancellationToken)
    {
        return new ValueTask<PipeResult>(PipeResult.Continue);
    }
}

class Pipe2 : IPipe
{
    public ValueTask<PipeResult> Execute(CancellationToken cancellationToken)
    {
        return new ValueTask<PipeResult>(PipeResult.Continue);
    }
}

class Pipe3 : IPipe
{
    public ValueTask<PipeResult> Execute(CancellationToken cancellationToken)
    {
        return new ValueTask<PipeResult>(PipeResult.Continue);
    }
}
```

### Consuming pipeline data 
Pipeline execution results with `IPipelineResult` which holds all exposed pipes data.
```csharp
var result = await pipeline.Execute(CancellationToken.None);
result.Get<Pipe1>().Should().NotBeNull();
result.Get<SomeData>().Should().NotBeNull();
```
#### Disposing pipeline result
Pipeline result inherited from `IAsyncDisposable` and can be disposed.
```csharp
await using var result = await pipeline.Execute(CancellationToken.None);
// or
var result = await pipeline.Execute(CancellationToken.None);
await result.DisposeAsync();
```
Below code presents implementation of `DisposeAsync` from `PipelineResult` class.
```csharp
public async ValueTask DisposeAsync()
{
    foreach (var pair in availableData)
    {
        if (pair.Value is IAsyncDisposable asyncDisposable)
            await asyncDisposable.DisposeAsync().ConfigureAwait(false);
        else if(pair.Value is IDisposable disposable)
        {
            disposable.Dispose();
        }
    }
    availableData.Clear();
}
```

### Breaking the pipeline
It's possible to break pipeline execution in each of pipe. 
```csharp

var pipeline = new Pipeline();
pipeline.Add<Pipe1>();
pipeline.Add<BreakPipe>();
pipeline.Add<Pipe2>();
var result = await pipeline.Execute(CancellationToken.None);

result.GetOptional<Pipe2>().Should().BeNull();


class BreakPipe : IPipe
{
    public ValueTask<PipeResult> Execute(CancellationToken cancellationToken)
    {
        return new ValueTask<PipeResult>(PipeResult.Break);
    }
}
(...)
```

### Examples
Examples can be found at [pipeline examples](https://github.com/adobrzyc/modulos/tree/master/examples/Examples.Modulos/Pipeline).



