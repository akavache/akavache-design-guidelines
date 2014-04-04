RxUi Design Guidelines
======================

A set of practices to help maintain sanity when building an RxUI client 
application distilled from hard lessons learned.

## Best Practices

The following recommendations are intended to help keep ReactiveUI code 
predictable, understandable, and fast.

They are, however, only guidelines. Use best judgment when determining whether 
to apply the recommendations here to a given piece of code.

### Commands

Prefer binding user interactions to commands rather than methods.

__Do__

```csharp
// In XAML
<Button Command="{Binding DeleteCommand}" .../>

public class RepositoryViewModel : ViewModelBase 
{
  public RepositoryViewModel() 
  {
    DeleteCommand = new ReactiveAsyncCommand();
    DeleteCommand.RegisterAsyncObservable(
      x => Delete(),  
      e => /* Do something with error */)
    .Subscribe();
  }

  public ReactiveAsyncCommand DeleteCommand { get; private set; }

  public IObservable<Unit> Delete() {...}
}
```

__Don't__

Use the Caliburn.Micro conventions for associating buttons and commands:

```csharp
// In XAML
<Button x:Name="Delete" .../>

public class RepositoryViewModel : PropertyChangedBase
{
  public void Delete() {...}	
}
```

Why? 

1. ReactiveAsyncCommand exposes the `CanExecute` property of the command to 
enable applications to introduce additional behaviour.
2. It handles marshaling the result back to the UI thread.
3. It tracks in-flight items.


#### Command Names

Suffix `ReactiveCommand` properties' names with `Command`, e.g.:

```csharp
	
public IReactiveCommand SynchronizeCommand { get; private set; }

// and then in the ctor:

SynchronizeCommand = new ReactiveCommand();
SynchronizeCommand.RegisterAsync(_ => Synchronize(mergeInsteadOfRebase: !IsAhead))
    .Subscribe();

```

### UI Thread and Schedulers

Always make sure to update the UI on the `RxApp.DeferredScheduler` to ensure UI 
changes happen on the UI thread. In practice, this typically means making sure 
to update view models on the deferred scheduler.

__Do__

```csharp
.FetchStuffAsync()
.ObserveOn(RxApp.DeferredScheduler)
.Subscribe(x => this.SomeViewModelProperty = x);
```
__Don't__

```csharp
.FetchStuffAsync()
.Subscribe(x => this.SomeViewModelProperty = x);
```

Even better, pass in the scheduler to methods that take one in.

__Better__

```csharp
.FetchStuffAsync(RxApp.DeferredScheduler)
.Subscribe(x => this.SomeViewModelProperty = x);
```

### Prefer Observable Property Helpers to setting properties explicitly

When a property's value depends on another property, a set of properties, or an 
observable stream, rather than set the value explicitly, use 
`ObservableAsPropertyHelper` with `WhenAny` wherever possible.

__Do__

```csharp
public class RepositoryViewModel : ViewModelBase 
{
  ObservableAsPropertyHelper<bool> canDoIt;

  public RepositoryViewModel() 
  {
    someViewModelProperty = this.WhenAny(x => x.StuffFetched, 
									 y => y.OtherStuffNotBusy, 
									 (x, y) => x && y)
      .ToProperty(this, x => x.CanDoIt, setViaReflection: false);
  }

  public bool CanDoIt
  {
    get { return canDoIt.Value; }  
  }	
}
```

__Don't__

```csharp
.FetchStuffAsync()
.ObserveOn(RxApp.DeferredScheduler)
.Subscribe(x => this.SomeViewModelProperty = x);
```

## Akavache

Ok, so I lied. This isn't just about RxUI. Here are some guidelines for 
Akavache. The general rule of thumb with Akavache is to treat it like any other 
service boundary. Pretend it's like calling a Web API.

### Store dumb simple data objects in the cache

__Do__

```csharp
// Data Class 
public class RepositoryData
{
    public string Name { get; set; }
    public string Owner { get; set; }
}

// When saving data.

var repoViewModel = GetRepository(name);
var repositoryData = new RepositoryData { Name = repoViewModel.Name, Owner = 
repoViewModel.Owner };

IBlobCache cache = GetCache(...);
cache.InsertObject<RepositoryData>(repositoryData);

// When retrieving
var repoData = await cache.GetObjectAsync<RepositoryData>();
var repoViewModel = new RepositoryViewModel { Name = repoData.Name, 
repoData.Owner };
```

__Don't__

Store arbitrary objects graphs:

```csharp
var repoViewModel = GetRepository(name);
IBlobCache cache = GetCache(...);
cache.InsertObject<RepositoryData>(repoViewModel);

// When retrieving
var repoViewModel = await cache.GetObjectAsync<RepositoryViewModel>();
```

__Better__

Akavache should have helper methods for doing this:

```csharp
var repoViewModel = GetRepository(name);
var repositoryData = repoViewModel.ToCacheable<RepositoryData>();

IBlobCache cache = GetCache(...);
cache.InsertObject<RepositoryData>(repositoryData);

// When retrieving
var repoData = await cache.GetObjectAsync<RepositoryData>();
var repoViewModel = new RepositoryViewModel(repoData);
// OR
var repoViewModel = repoData.ToViewModel<IRepositoryViewModel>();
```
