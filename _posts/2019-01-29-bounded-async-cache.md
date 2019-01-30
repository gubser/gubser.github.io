---
title:  "A bounded in-memory cache for async operations written in F#"
---

# A bounded in-memory cache for async operations written in F#
I am currently working on an algorithm for solving small but highly constrained vehicle routing problems (VRP). The distance and duration is calculated via a routing webservice. In order to reduce latency and unnecessary computation time to a minimum I decided to add an in-memory cache to the VRP algorithm.

The [solution available as a Gist](https://gist.github.com/gubser/0888d947a701ad11b4b8fdccacdf1ed3) is a single file with less than 200 lines of code (including comments) built upon [ConcurrentDictionary](https://docs.microsoft.com/en-us/dotnet/api/system.collections.concurrent.concurrentdictionary-2?view=netstandard-2.0), which is suitable for highly concurrent applications.

## Design Goals

**Avoid duplicate requests:**
In a traditional cache setting, an algorithm that wants to fetch a value would first check the cache. If it's a cache miss, it would request the value from the webservice, wait for the answer to arrive and store the value in the cache.
However, in a multithreaded or asynchronous environment there is a major drawback in this concept: If multiple threads want to fetch the same uncached value, multiple identical requests are sent to the webservice. It would be better if all threads would just wait for a single request to avoid unnecessary communication. Effectively this means that we also need to cache pending requests.

**Do not consider stale entries:**
Cache entries should not be used when they get over a certain age.

**Do not cache errors:**
We only want to cache values and pending requests. No errors, because we don't want to end up with a [negative cache](https://en.wikipedia.org/wiki/Negative_cache).
If a pending request yields an error, all waiting consumers are notified and the pending request is evicted from the cache.

**Limited number of entries:**
The cache should only store a limited amount of entries. To satisfy this limit, entries are evicted from the cache in least recently used (LRU) order before adding new entries.

**Non-functional requirements:** The algorithm should have a low memory overhead and should be elegant, easy to read and maintain.

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
The cache is implemented as a class and holds a reference to a `ConcurrentDictionary`. (The cache parameters `maxCount`, `removeBulkSize`, ... are discussed later in this post.)
```fsharp
type AsyncBoundedCache<'key,'value> (maxCount: int, removeBulkSize: int, maxRetrievalAge: int, request: 'key -> Async<'value>) =
    let cache = ConcurrentDictionary<'key, CacheEntry<'value>>()

    ...
```

Any consumer of this cache will call the `Get` member function in the following code section. If the cache hit (valid or pending cache entry), `Get` returns the value by calling the `accessValue` function. If the cache missed (invalid or missing cache entry), the value is retrieved by calling the `getOrRequest` function. Note how `accessValue` either waits for the task to yield a result or updates the `LastAccessedAt` time.
I'm still amazed how concise and readable F# allows us to express this.

```fsharp
    // Check validity
    let isValid cachedValue = (now() - cachedValue.RetrievedAt) < maxRetrievalAge
    
    let isValidOrPending entry =
        match entry with
        | CachedValue value -> isValid value
        | PendingRequest _ -> true
    
    // Get the value from a CacheEntry and update its access time
    let accessValue key entry = async {
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
            return! accessValue key entry
        | _ ->
            return! acquire key
    }
```

Cache read was the easy part. Now it gets messy because of this [property of ConcurrentDictionary described in the Docs](https://docs.microsoft.com/en-us/dotnet/api/system.collections.concurrent.concurrentdictionary-2?view=netstandard-2.0#remarks):
> For modifications and write operations to the dictionary, ConcurrentDictionary<TKey,TValue>  uses fine-grained locking to ensure thread safety. (Read operations on the dictionary are performed in a lock-free manner.) However, delegates for these methods are called outside the locks to avoid the problems that can arise from executing unknown code under a lock. Therefore, the code executed by these delegates is not subject to the atomicity of the operation.

So, in order to guarantee that nobody else has started a request in the meantime, we need to ensure atomicity ourselves by introducing a write lock. So what `getOrRequest` does is: acquire the lock, check the cache again and return if the value is found. Otherwise create a task by calling `startRequestTask`.
Inside `startRequestTask` we can finally call `request`, create a `CachedValue` and store it in the cache (which will replace the existing `PendingRequest` there).
If there is an exception, the `PendingRequest` is removed and anyone that waits on this task (remember `accessValue` from above?) gets that exception. Neat.

```fsharp
    let writeLock = new SemaphoreSlim(1, 1)

    let startAcquireTask key = Async.StartAsTask(async {
        try
            let! value = acquire key
            let entry = CachedValue { Value = value; RetrievedAt = now(); LastAccessedAt = now() }

            // we've got the value. now replace cached entry
            do! Async.AwaitTask (writeLock.WaitAsync())
            try
                havingLockEnsureBound ()
                havingLockSetEntry key entry
            finally
                writeLock.Release() |> ignore
            
            return value
        with
        | e ->
            // something failed. remove cached entry and propagate exception
            // to anyone that waits on this task
            do! Async.AwaitTask (writeLock.WaitAsync())
            try
                havingLockUnsetEntry key
            finally
                writeLock.Release() |> ignore

            return raise e
    })

    // Acquire the writeLock, try to get the cached value or pending request.
    // If its a miss, start a new acquire task.
    // The lock is necessary to avoid two threads to start a new task simultaneously.
    let acquire key = async {
        do! Async.AwaitTask (writeLock.WaitAsync())
        let entry =
            try
                match cache.TryGetValue(key) with
                | true, existingEntry when isValidOrPending existingEntry ->
                    // A valid or pending cache entry found, use that.
                    existingEntry
                | _ ->
                    havingLockEnsureBound ()

                    // No cache entry found or invalid cache entry -> start a new acquire task
                    let newEntry = PendingRequest (startAcquireTask key)
                    havingLockSetEntry key newEntry
                    newEntry
            finally
                writeLock.Release() |> ignore
        return! accessValue key entry
    }
```

If you want to get more functional, `startRequestTask` would be a good place to extend the code to react to `Result.Ok` and `Result.Error` when doing [railway oriented programming](https://fsharpforfunandprofit.com/rop/)

Now there are only three interesting functions left to explore: `havingLockSetEntry`, `havingLockEnsureBound` and `havingLockUnsetEntry`.

The functions `havingLockSetEntry` and `havingLockUnsetEntry` are just wrappers around `ConcurrentDictionary.AddOrUpdate` and `ConcurrentDictionary.TryRemove` in order to track the total count of cache entries. I added these because I thought it would [lock the whole dictionary](https://github.com/dotnet/corefx/issues/3357) but apparently [reads are unaffected](https://github.com/dotnet/corefx/blob/a10890f4ffe0fadf090c922578ba0e606ebdd16c/src/System.Collections.Concurrent/src/System/Collections/Concurrent/ConcurrentDictionary.cs#L441). Still, accessing the `ConcurrentDictionary.Count` property is a [surprisingly "expensive" operation](https://github.com/dotnet/corefx/blob/a10890f4ffe0fadf090c922578ba0e606ebdd16c/src/System.Collections.Concurrent/src/System/Collections/Concurrent/ConcurrentDictionary.cs#L939-L982) using a for loop. Still, this is a case of premature optimization and I should remove this unnecessary code.

The job of `havingLockEnsureBound` is to make sure that we never exceed `maxCount` elements in the cache. First it evicts `CachedValue`s in LRU order and if that doesn't suffice it starts to evict `PendingRequest`s. Note that any consumer that still holds a reference to `Task` will get the result. It just won't be cached here.

```fsharp
    let mutable count = 0

    // Remove a CacheEntry and update count. Needs writeLock.
    let havingLockUnsetEntry key =
        match cache.TryRemove(key) with
        | true, _ -> count <- count - 1
        | false, _ -> ()

    // Remove entries from the cache until bounds are satisfied. Needs writeLock.
    let havingLockEnsureBound () =
        if count >= maxCount then
            // remove excess cached values, and some more (=removeBulkSize) while we're at it
            let countToRemove = (count - maxCount) + removeBulkSize
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
            
            toRemoveCached |> List.iter havingLockUnsetEntry

            // check if we satisfy the bound now
            if count >= maxCount then
                // there are still too many items. above we have only removed
                // CachedValues. Apparently we have deleted them all and we are now left with
                // PendingRequests only.
                let countToRemove = (count - maxCount) + removeBulkSize

                // I don't care which PendingRequest was accessed least recently
                // because I assume every PendingRequest is recent enough.
                // 
                // any consumer will still hold a reference to the task so they will
                // receive their answer. it just won't be cached here.
                let toRemove =
                    cache.Keys
                    |> Seq.truncate countToRemove
                    |> Seq.toList
                
                toRemove |> List.iter havingLockUnsetEntry

    // Add an entry to the cache or updates an entry in the cache.
    // Will evict other cache entries if bound is hit. Needs writeLock.
    let havingLockSetEntry key entry =
        havingLockEnsureBound ()

        let addFunc _ = count <- count + 1; entry
        let updateFunc _ _ = entry

        cache.AddOrUpdate(key, addFunc, updateFunc) |> ignore
```

## Unit tests and project files
I'm currently working on some unit tests and plan to move the code into a proper repository.
[See this Gist for unit tests](https://gist.github.com/gubser/f527e1fdbb060a77be340442b15fcc55).
