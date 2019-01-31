---
title:  "A bounded in-memory cache for async operations written in F#"
---

# A bounded in-memory cache for async operations written in F#
I am currently working on an optimization algorithm that gathers information from map routing services and linear solvers. In order to reduce latency and unnecessary computations to a minimum I decided to add a transparent in-memory cache.

The [code of the in-memory cache is available as a Gist](https://gist.github.com/gubser/0888d947a701ad11b4b8fdccacdf1ed3) for now. I plan to put it into a proper repository later. It's a single file with less than 200 lines of code (including comments). Built upon [ConcurrentDictionary](https://docs.microsoft.com/en-us/dotnet/api/system.collections.concurrent.concurrentdictionary-2?view=netstandard-2.0), which is suitable for highly concurrent applications.

## Design Goals

**Avoid duplicate requests:**
In a traditional cache setting, an algorithm (T1, see diagram below) that wants to fetch a value would first check the cache. If it's a cache miss, it would request the value from the webservice, wait for the answer to arrive and store the value in the cache. After that any later access (T2) would result in a cache hit.

![Traditional cache method](http://www.plantuml.com/plantuml/png/PP0n2iCm34LtdSBGkOCzPYY17A7UIWTZYqRGg45MSlwwRMfmCdt-u_45whC6qMLwck75SH51LfWBeaXpO3NUjjL1quSGHsp85MMbY03UclFj99Zkbqs3RrJwLpipKSuGej8Q5El2blkLX0UpWf-nE-CjW7UV_X14BgJLAlQkCoCfG8-Soa_U)

However, in a multithreaded or asynchronous environment there is a major drawback in this concept: If multiple threads (T1, T2, T3) want to fetch the same uncached value, multiple identical requests are sent to the webservice. For example, this situation is common in optimization algoritms where you use some sort of local search heuristic: Many solutions are generated that differ only in a few parts. To evaluate them you need mostly the same values over and over.

![Problem with multiple consumers](http://www.plantuml.com/plantuml/png/XP0_YuGm4CNx-HI1gw_mJsLn2FQ7SEcEBR9nq42C4XC__vh54MPJMCdxpVl98-qMb0znjg9Rd8xUemk_IuzkC6w4zRWPRLRbWf05ZoMF5OyriDmfFI4ZV-Xten505kBx_ylZyFWvQ_3-N7IXRYDciss7KQRRqqOaXGoYkLAbOn_zQdE9UAwTyNEWqi7iAY3_3tLaSObtEuKiMVTkO7fc05adDdf4brS9oxeHama0BReXpXPU)

It would be better if all threads would just wait for a single request to avoid unnecessary communication. Effectively this means that we also need to cache pending requests:

![Cache with pending requests](http://www.plantuml.com/plantuml/png/XP0nJyGm38Lt_uf8h30qwTG17Re_S0AB1J64nhf6IjEIEkNlerQ1u3BSKkczFdzstcbXcpYFGPsdsEUKAFA5elFn2hDDx7i_syWA6ocrb4RA5eG-stuWuRnGMdrF0DYeXxUxHExziSJ0zknNorJq_gsXCjcfqIzBpHRxcCQcKuFdUrNUz4oVcHQkSzZ042ScDQsKzlZJb_KCW7gZV8HCvR8VTdLHtu9h0TSLRZRC9QSv7F32HtDhWH4BpE-2KiUXsqwvAH8u-bVRNmyReRHG1W2mtRZNH1IFnrSRstBx_iUzsf09u4JHvJ5y0m00)

**Do not consider stale entries:**
Cache entries should not be used when they get over a certain age.

**Do not cache errors:**
We only want to cache values and pending requests. No errors, because we don't want to end up with a [negative cache](https://en.wikipedia.org/wiki/Negative_cache).
If a pending request yields an error, all waiting consumers are notified and the pending request is evicted from the cache.

**Limited number of entries:**
The cache should only store a limited amount of entries. To satisfy this limit, entries are evicted from the cache in least recently used (LRU) order before adding new entries.

**Non-functional requirements:** The algorithm should have a low memory and read overhead as well as being easy to read and maintain.

## Data Structure
I started with the data structures holding the values. A `CacheEntry` is a discriminated union of either
- a `CachedValue` record containing the value itself, last access time and retrieval time or
- a `PendingRequest` of type `Task`.

```fsharp
type CachedValue<'value> = {
    Value: 'value
    RetrievedAt: int
    mutable LastAccessedAt: int
}

type CacheEntry<'value> =
| CachedValue of CachedValue<'value>
| PendingRequest of Task<'value>
```

The additional values in `CachedValue` mean:
- `RetrievedAt`: Used to determine if the entry is out-of-date (invalid).
- `LastAccessedAt`: Used to sort entries to evict from the cache in LRU order.

## Cache Logic
The cache is implemented as a class and holds a reference to a `ConcurrentDictionary`. `request` is the function we want to cache. The other parameters `maxCount`, `removeBulkSize`, `maxRetrievalAge` are discussed in the following section.
```fsharp
type AsyncBoundedCache<'key,'value>
        (maxCount: int, removeBulkSize: int, maxRetrievalAge: int,
         request: 'key -> Async<'value>) =
    let cache = ConcurrentDictionary<'key, CacheEntry<'value>>()

    ...
```

The cache can be used like this:
```
// the function
let echo key = async {
    Async.Sleep 100
    return sprintf "Hello '%s'" key
}

// cached version with the same signature as `echo`
let cachedEcho = AsyncBoundedCache(100, 10, 100, echo).Get
```

### Lock-free cache read
Any consumer of this cache will call the `Get` member function in the following code section. If the cache hit a valid value or a pending request, return it via `accessValue`. If the cache missed (invalid or missing cache entry) we go into `getOrRequest`. Note how `accessValue` either waits for the task to yield a result if it's a `PendingRequest` or updates the `LastAccessedAt` time if it's a `CachedValue`.
I'm still amazed how concise and readable F# allows us to express this.

```fsharp
// Check validity
let isValid cachedValue =
    (now() - cachedValue.RetrievedAt) < maxRetrievalAge

let isValidOrPending entry =
    match entry with
    | CachedValue cachedValue -> isValid cachedValue
    | PendingRequest _ -> true

// Get the value from a CacheEntry and update its access time
let accessValue entry = async {
    match entry with
    | CachedValue cachedValue ->
        cachedValue.LastAccessedAt <- now()
        return cachedValue.Value
    | PendingRequest task ->
        return! Async.AwaitTask task
}

// perform a lock-free read.
// if the cached value is too old or missing, it starts a new request operation.
member __.Get key = async {
    match cache.TryGetValue(key) with
    | true, entry when isValidOrPending entry ->
        return! accessValue entry
    | _ ->
        return! getOrRequest key
}
```

### On cache miss
Cache read was the easy part. Now it gets messy because of this [property of ConcurrentDictionary described in the Docs](https://docs.microsoft.com/en-us/dotnet/api/system.collections.concurrent.concurrentdictionary-2?view=netstandard-2.0#remarks):
> For modifications and write operations to the dictionary, ConcurrentDictionary<TKey,TValue>  uses fine-grained locking to ensure thread safety. (Read operations on the dictionary are performed in a lock-free manner.) However, delegates for these methods are called outside the locks to avoid the problems that can arise from executing unknown code under a lock. Therefore, the code executed by these delegates is not subject to the atomicity of the operation.

In order to guarantee that nobody else has started a request in the meantime, we need to ensure atomicity ourselves. That's why I added a write lock. In `getOrRequest` we do almost the same thing as `Get`. But in addition, we acquire the lock and add a `PendingRequest` using `startRequestTask`. Note that we hold the lock only for a very brief period of time between reading the cache and creating the PendingRequest.

Within the Task created by `startRequestTask` we can finally call `request`, create a `CachedValue` and store it in the cache (which will replace the existing `PendingRequest` there).
If there is an exception, the `PendingRequest` is removed and anyone that waits on this task (remember `accessValue` from above?) gets that exception. Neat.

```fsharp
let writeLock = new SemaphoreSlim(1, 1)

let startRequestTask key = Async.StartAsTask(async {
    try
        let! value = request key
        let entry = CachedValue {
            Value = value;
            RetrievedAt = now();
            LastAccessedAt = now()
        }

        // we've got the value. now replace cached entry
        do! Async.AwaitTask (writeLock.WaitAsync())
        try
            havingLockEnsureBound ()
            cache.[key] <- entry
        finally
            writeLock.Release() |> ignore
        
        return value
    with
    | e ->
        // something failed. remove cached entry and propagate exception
        // to anyone that waits on this task
        do! Async.AwaitTask (writeLock.WaitAsync())
        try
            cache.TryRemove key |> ignore
        finally
            writeLock.Release() |> ignore

        return raise e
})

// Acquire the writeLock, try to get the cached value or pending request.
// If its a miss, start a new request task.
// The lock is necessary to avoid two threads to start a new task simultaneously.
let getOrRequest key = async {
    do! Async.AwaitTask (writeLock.WaitAsync())
    let entry =
        try
            // Try again to get the value, now having lock
            match cache.TryGetValue(key) with
            | true, existingEntry when isValidOrPending existingEntry ->
                // A valid or pending cache entry found, use that.
                existingEntry
            | _ ->
                // Start a new request task because only an invalid entry was
                // found or none at all.
                havingLockEnsureBound ()
                let newEntry = PendingRequest (startRequestTask key)
                cache.[key] <- newEntry
                newEntry
        finally
            writeLock.Release() |> ignore
    return! accessValue entry
}
```

If you want to get more functional, `startRequestTask` would be a good place to extend the code to react to `Result.Ok` and `Result.Error` when doing [railway oriented programming](https://fsharpforfunandprofit.com/rop/)

The last function `havingLockEnsureBound` removes excess cache elements.
First it evicts `CachedValue`s in LRU order and if that doesn't suffice it starts to evict `PendingRequest`s.
This is computationally expensive because we're sorting all cache elements by access time.
To make this a seldom used operation, we remove a few extra elements (`removeBulkSize`) as well.
Before optimizing this further, I need to run some performance experiments.
Note that any consumer that still holds a reference to `Task` will get the result. It just won't be cached.

```fsharp
let havingLockEnsureBound () =
    if cache.Count >= maxCount then
        // remove excess cached values, and some more while we're at it
        let countToRemove = (cache.Count - maxCount) + removeBulkSize
        let toRemoveCached =
            cache
            |> Seq.choose (fun kv ->
                match kv.Value with
                | CachedValue cachedValue -> Some (kv.Key, cachedValue)
                | _ -> None
            )
            |> Seq.sortBy (fun (_, cachedValue) -> cachedValue.LastAccessedAt)
            |> Seq.map fst
            |> Seq.truncate countToRemove
            |> Seq.toList
        
        toRemoveCached |> List.iter (fun e -> cache.TryRemove e |> ignore)

        // check if we satisfy the bound now
        if cache.Count >= maxCount then
            // there are still too many items. above we have only removed
            // CachedValues. Apparently we have deleted them all and we are
            // now left with PendingRequests only.
            let countToRemove = (cache.Count - maxCount) + removeBulkSize

            // I don't care which PendingRequest was accessed least recently
            // because I assume every PendingRequest is recent enough.
            // 
            // any consumer will still hold a reference to the task so they will
            // receive their answer. it just won't be cached here.
            let toRemove =
                cache.Keys
                |> Seq.truncate countToRemove
                |> Seq.toList
            
            toRemove |> List.iter (fun e -> cache.TryRemove e |> ignore)
```

## Unit tests and project files
I'm currently working on some unit tests and plan to move the code into a proper repository.
[See this Gist for unit tests](https://gist.github.com/gubser/f527e1fdbb060a77be340442b15fcc55).
