Title: [Obsolete] ReactiveList
---

One of the built-in classes that ships with ReactiveUI is an improved version of .NET's `ObservableCollection` (which is ironically, *not* an Observable). `ReactiveList` should be used in any place that you would normally use a List or ObservableCollection, as it has additional useful Rx features.

# Subscribing to Changes

`ReactiveList` provides several useful Observables that can be subscribed to in order to inform you about changes in the list, as well as providing you with notifications that happen before a list is about to change:

* **(Before)ItemsAdded** - signals when items are added
* **(Before)ItemsRemoved** - signals when items are removed
* **(Before)ItemsMoved** - signals when items are moved
* **CountChang(ing/ed)** - signals when the number of items in the list
  changes for any reason
* **Changed** - passes along the `NotifyCollectionChangedEventArgs` from all
  changes (i.e. is an Observable version of `NotifyCollectionChanged`).
* **ShouldReset** - signals that the observer should reread the entire
  collection, as it has changed significantly

# Semantics of Reset

One thing that is particularly important to understand is the meaning of the ShouldReset Observable. The meaning of this event is, "This collection has changed drastically, you should reread the contents". Many people conflate *Reset* and *Clear*, thinking that this means the collection is now empty.

This is important, because if you only Subscribe to `ItemsAdded` and `ItemsRemoved`, you will not be correctly tracking every item in the list. ReactiveList will detect this scenario and attempt to warn you about it. For example, here is an example of maintaining a "running count" of the number of
items in the list:

```cs
// Note that this code is hacky and for illustrative purposes.
// CountChanged would suffice for this.
TweetList = new ReactiveList<Tweet>();

var count = 0;
var addedOrRemoved = Observable.Merge(
    TweetList.ItemsAdded.Select(_ => 1),
    TweetList.ItemsRemoved.Select(_ => -1));

addedOrRemoved.Subscribe(x => count += x);
TweetList.ShouldReset.Subscribe(_ => count = TweetList.Count);
```

Should you want to execute code on every object in a collection as they are added or removed, the `ActOnEveryObject` method documented in this guide will handle many edge cases around this task automatically.

# Using Change Tracking

Not only can ReactiveList watch changes to the list, it can optionally tell you about changes to any item *in* the list. One practical example of this use-case is, we'd like to be notified when any document in the document list becomes dirty. Here's one way to do that:

```cs
DocumentList = new ReactiveList<Document>() {
    ChangeTrackingEnabled = true,
};

DocumentList.ItemChanged
    .Where(x => x.PropertyName == "IsDirty" && x.Sender.IsDirty)
    .Select(x => x.Sender)
    .Subscribe(x => {
        Console.WriteLine("Make sure to save {0}!", x.DocumentName);
    });
```

Note that we had to set `ChangeTrackingEnabled` here to `true`, since change tracking is disabled by default for performance reasons. Note that one thing we *didn't* have to do in this code is, attempt to watch for changing elements in the collection - ReactiveList does that all for us.

ReactiveList only tracks changes to its immediate objects, it won't track an entire object hierarchy (i.e. `listOfItems[0].Foo.Bar = true` won't trigger a change notification, but `listOfItems[0].Foo = Bar` will).

# Suppressing Notifications

Since ReactiveLists are often bound to UI elements like ListBoxes, making many small changes to a list all at once can have a significant impact in performance - changes to the list result in UI elements being created or destroyed as well as relayout and rerendering, which are quite expensive
operations.

Instead, when you want to make several changes to a list at the same time, use `SuppressChangeNotifications` - this will disable change notifications for the duration of the operation, then send a **Reset** notification which will trigger the UI to reload itself.

```cs
// Without this using statement, the ListBox associated with this List would
// refresh **four** separate times!
using (TweetsList.SuppressChangeNotifications()) {
    TweetsList.Clear();
    for (var page=0; page < 3; page++) {
        var tweets = await GetTweets(page);
        TweetsList.AddRange(tweets);
    }
}
```

Range methods such as `AddRange`, `InsertRange`, etc automatically will suppress change notifications if the percentage of the list being changed is above a certain level (i.e. if you're changing 90% of the list, it makes sense to signal a Reset, but if you're changing 5% of the list, it's better to signal Adds/Deletes)

# CreateCollection

One method that is often useful for testing, is to create a list whose contents automatically get populated by an Observable. While in this case, it's probably easier to test the object state directly, you can also check correctness via counting changes:

```cs
[Fact]
public void MakeSureTweetListGetsUpdatedOnRefresh()
{
    var fixture = new TweetsListViewModel();
    var output = fixture.TweetList.ItemsAdded.CreateCollection();
    Assert.Equal(0, output.Count);

    fixture.Refresh.Execute();
    Assert.Equal(1, output.Count);
}
```

# CreateDerivedCollection and Derived Lists

`ReactiveList.CreateDerivedCollection` is an extremely powerful method in MVVM programming, which allows you to create a projection of a ReactiveList as another list. `CreateDerivedCollection` allows you to do projection, ordering, and filtering of a source ReactiveList.

Once a derived list has been created, this list will update dynamically based on the source collection - as the source list has items added and removed, the derived list will also update its contents. If `ChangeTrackingEnabled` is enabled on the source list, changes to the source list will also cause updates to the derived list.

For example, if you have a source list of MenuItems and you have a `filter` method of `x => x.IsEnabled`, changing `IsEnabled` on any of the menu items will hide or show the items in the list.  Note that the derived list is read-only - you cannot manually modify the derived list.

```cs
public class TweetsListViewModel : ReactiveObject
{
    private readonly ReactiveList<Tweet> _tweets;
    public IReactiveDerivedList<TweetTileViewModel> TweetTiles { get; }

    public TweetsListViewModel()
    {
        _tweets = new ReactiveList<Tweet>();
        TweetTiles = Tweets.CreateDerivedCollection(
            x => new TweetTileViewModel(x));

        // Adding a new item to Tweets results in a 
        // new ViewModel showing up in TweetTiles.
        Tweets.Add(new Tweet { Title = "Hello!" });
    }
}
```

# Derived Lists filtering and ordering
`CreateDerivedCollection` can do some more interesting things, but to illustrate it, we need a more interesting set of classes.

```cs
public class Tweet 
{
    public DateTime CreatedAt { get; set; }
}

public class TweetTileViewModel : ReactiveObject
{
    private bool isHidden;
    public bool IsHidden 
    {
        get => isHidden;
        set => this.RaiseAndSetIfChanged(ref isHidden, value);
    }

    public Tweet Model { get; set; }
}
```

Now, let's connect up the filtering and ordering. It's important to note, that since want to refilter the list based on the IsHidden value, it must be a ReactiveObject and fire change notifications, or else CreateDerivedList has no way to know when it should refilter.

Because we had to put `IsHidden` on the ViewModel, we need to set up two levels of filtering. Models that are ReactiveObject would make this easier, but is often not possible. Let's take a look:

```cs
public class TweetsListViewModel : ReactiveObject
{
    ReactiveList<Tweet> Tweets = new ReactiveList<Tweet>();

    IReactiveDerivedList<TweetTileViewModel> TweetTiles;
    IReactiveDerivedList<TweetTileViewModel> VisibleTiles;

    public TweetsListViewModel()
    {
        TweetTiles = Tweets.CreateDerivedCollection(
            x => new TweetTileViewModel(x), x => true,
            (x, y) => x.CreatedAt.CompareTo(y.CreatedAt));

        VisibleTiles = TweetTiles.CreateDerivedCollection(
            x => x, x => !x.IsHidden);
    }
}
```

CreateDerivedCollection allows us to declare how a list should be transformed and keep up with the original list.