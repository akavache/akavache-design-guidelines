Akavache Design Guidelines
======================

A set of practices to help maintain sanity when building an application that uses Akavache 
application distilled from hard lessons learned.

The general rule of thumb with Akavache is to treat it like any other 
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
