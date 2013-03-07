RxUi Design Guidelines
======================

A set of practices to help maintain sanity when building an RxUI client application distilled from hard lessons learned.

## Best Practices

The following recommendations are intended to help keep ReactiveUI code predictable, understandable, and fast.

They are, however, only guidelines. Use best judgment when determining whether to apply the recommendations here to a given piece of code.

### Commands

Prefer binding user interactions to commands rather than methods.

__Do__

    // In Xaml
    <Button Command="{Binding DeleteCommand}" .../>

    // In ctor
    DeleteCommand = new ReactiveAsyncCommand();
    DeleteCommand.RegisterAsyncObservable(
            x => Delete(),  
            e => /* Do something with error */)
        .Subscribe();
    
    // In class
    public IObservable<Unit> Delete() {...}

__Don't__

    // In Xaml (using Caliburn Micro based convention)
    <Button x:Name="Delete" .../>

    // In class
    public void Delete() {...}

This provides multiple benefits.

1. We can bind to the `CanExecute` property of the command to disable/enable buttons.
2. It handles marshaling the result back to the UI thread.
3. It tracks in-flight items.


### UI Thread and Schedulers

Always make sure to update the UI on the `RxApp.DeferredScheduler` to ensure UI changes happen on the UI thread. In practice, this typically means making sure to update view models on the deferred scheduler.

__Do__

    .FetchStuffAsync()
    .ObserveOn(RxApp.DeferredScheduler)
    .Subscribe(x => this.SomeViewModelProperty = x);

__Don't__

    .FetchStuffAsync()
    .Subscribe(x => this.SomeViewModelProperty = x);


Even better, pass in the scheduler to methods that take one in.

__Better__

    .FetchStuffAsync(RxApp.DeferredScheduler)
    .Subscribe(x => this.SomeViewModelProperty = x);


### Prefer Observable Properties Helpers to setting properties explicitly
When a property's value depends on another property or observable sequence, rather than set the value explicitly, use `ObservableAsPropertyHelper` with `ObservableForProperty` or `WhenAny` wherever possible.

__Do__

     // Class member
     ObservableAsPropertyHelper<string> someViewModelProperty;
 
    // In ctor
    someViewModelProperty = this.ObservableForProperty(x => x.StuffFetched)
        .Where(x != null)
        .ToProperty(this, x => x.SomeViewModelProperty, setViaReflection: false);


    // Property definition
    public string SomeViewModelProperty
    {
        get
        {
            return someViewModelProperty.Value;
        }
    }

__Don't__

    .FetchStuffAsync()
    .ObserveOn(RxApp.DeferredScheduler)
    .Subscribe(x => this.SomeViewModelProperty = x);


Use `WhenAny` when creating a property dependent on more than one property.

__Do__

     // Class member
     ObservableAsPropertyHelper<bool> canDoIt;
 
    // In ctor
    someViewModelProperty = this.WhenAny(x => x.StuffFetched, y => y.OtherStuffNotBusy, (x, y) => x && y)
        .ToProperty(this, x => x.CanDoIt, setViaReflection: false);


    // Property definition
    public bool CanDoIt
    {
        get
        {
            return canDoIt.Value;
        }
    }


## Akavache

Ok, so I lied. This isn't just about RxUI. Here are some guidelines for Akavache. The general rule of thumb with Akavache is to treat it like any other service boundary. Pretend it's like calling a Web API.

### Store dumb simple data objects in the cache

__Do__

    // Data Class 
    public class RepositoryData
    {
        public string Name { get; set; }
        public string Owner { get; set; }
    }

    // When saving data.

    var repoViewModel = GetRepository(name);
    var repositoryData = new RepositoryData { Name = repoViewModel.Name, Owner = repoViewModel };

    IBlobCache cache = GetCache(...);
    cache.InsertObject<RepositoryData>(repositoryData);

    // When retrieving
    var repoData = await cache.GetObjectAsync<RepositoryData>();
    var repoViewModel = new RepositoryViewModel { Name = repoData.Name, repoData.Owner };

__Don't__

    var repoViewModel = GetRepository(name);
    IBlobCache cache = GetCache(...);
    cache.InsertObject<RepositoryData>(repoViewModel);

    // When retrieving
    var repoViewModel = await cache.GetObjectAsync<RepositoryViewModel>();


Even better, we should have helper methods for doing this.

__Better__

    var repoViewModel = GetRepository(name);
    var repositoryData = repoViewModel.ToCacheable<RepositoryData>();

    IBlobCache cache = GetCache(...);
    cache.InsertObject<RepositoryData>(repositoryData);

    // When retrieving
    var repoData = await cache.GetObjectAsync<RepositoryData>();
    var repoViewModel = new RepositoryViewModel(repoData);
    // OR
    var repoViewModel = repoData.ToViewModel<IRepositoryViewModel>();
