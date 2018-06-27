# Routing in ReactiveUI

    ReactiveUI takes a View Model First approach to navigation.  ReactiveUI calls this navigation event Routing.

## `IScreen`
    Your applications implementation of `IScreen` needs
    - Initialize `RoutingState`
    - Register Views to View Models
    - Set the default page
    - The ability to set `RoutedViewHost`

## Register Views 
    In order to allow ReactiveUI to have knowledge of your View <-> View Model binding you have to register your Views to your View Models.  ReactiveUI expects some `IViewFor<TViewModel>`.  The default IoC container for ReactiveUI is splat and registrations would look something like the following.

``` csharp
// This registers your actual Router instance, which implements IScreen
Locator.CurrentMutable.RegisterConstant(new Router(), typeof(IScreen));

// Register Views and View Models
Locator.CurrentMutable.Register(() => new MainView(), typeof(IViewFor<MainViewModel>));
Locator.CurrentMutable.Register(() => new SecondView(), typeof(IViewFor<SecondViewModel>));
Locator.CurrentMutable.Register(() => new LandingView(), typeof(IViewFor<LandingViewModel>));
```

## View Models should extend from `IRoutableViewModel`
    
```csharp
public class RoutableViewModel : ReactiveObject, IRoutableViewModel 
{
    public string UrlPathSegment { get; protected set; }

    public IScreen HostScreen { get; protected set; }
}
```

## Application

``` csharp
    var bootstrapper = new AppBootstrapper();
    MainPage = bootstrapper.CreateMainPage();
```

## Samples
 - https://github.com/GiusepeCasagrande/RoutingSimpleSample
 - https://github.com/TheEightBot/Reactive-Examples