Title: Routing State
Order: 10
---

# Routing State

The `RoutingState` object is implemented in the `ReactiveUI` core package.  This object provides access to an underlying Navigation Stack.  The navigation stack is a collection of `IRoutableViewModel`.  `IRoutableViewModel` is an `IReactiveObject` that exposes an `IScreen` which should be registered during application startup.

```csharp
    public interface IRoutableViewModel : IReactiveObject
    {
        string UrlPathSegment { get; }

        IScreen HostScreen { get; }
    }
```

The Router exposes several functions that allow interaction with the Navigation Stack.  They are all `Reactive Command` objects that return an observable sequence.

- `Navigate` - Pushes a new `IRoutableViewModel` onto the navigation stack.

- `NavigateBack` - Pops the top `IRoutableViewModel` from the stack and returns a signal.

- `NavigatAndReset` - Resets the navigation stack and navigates to the specified `IRoutableViewModel`.

# Routable View Model Extensions

- `WhenNavigatedTo`

- `WhenNavigatedToObservable`

- `WhenNavigatingFromObservable`