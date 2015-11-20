# RxKitchenSink
Dumping ground for useful Reactive Extensions code snippets

### Pausable

I first noticed this at [rxmarbles.com](http://rxmarbles.com), but that is for the JavaScript libraries. The `Pausable` filter wasn't available in C#. So I went searching and found [this post on Stack Overflow](http://stackoverflow.com/questions/7620182/pause-and-resume-subscription-on-cold-iobservable). It does the trick.

```c#
public static IObservable<T> Pausable<T>(
    this IObservable<T> source,
    IObservable<bool> pauser)
{
    return Observable.Create<T>(o =>
    {
        var paused = new SerialDisposable();
        var subscription = Observable.Publish(source, ps =>
        {
            var values = new ReplaySubject<T>();
            Func<bool, IObservable<T>> switcher = b =>
            {
                if (b)
                {
                    values.Dispose();
                    values = new ReplaySubject<T>();
                    paused.Disposable = ps.Subscribe(values);
                    return Observable.Empty<T>();
                }
                else
                {
                    return values.Concat(ps);
                }
            };

            return pauser.StartWith(false).DistinctUntilChanged()
                .Select(p => switcher(p))
                .Switch();
        }).Subscribe(o);
        return new CompositeDisposable(subscription, paused);
    });
}
```
