---
title:  "Lightweight persistence layer with PostgreSQL, Dapper and F# records"
---
## Lightweight persistence layer with PostgreSQL, Dapper and F# records

I needed a simple way to store F# records into a postgres database.


Using [Npgsql](http://www.npgsql.org/) and [Dapper](https://github.com/StackExchange/Dapper)
querying the database is quite easy:
```fsharp
let connectionString = Environment.GetEnvironmentVariable "CONNECTION_STRING"

//
// query helpers
let querySingleAsync<'a> command param = async {
    use conn = new NpgsqlConnection(connectionString)

    let! result = conn.QuerySingleOrDefaultAsync<'a>(command, param) |> Async.AwaitTask
    if isNull (box result) then
        return None
    else
        return Some result
}

let queryAsync<'a> command param = async {
    use conn = new NpgsqlConnection(connectionString)

    return! conn.QueryAsync<'a>(command, param) |> Async.AwaitTask
}
```

```fsharp
let fruitId = 55
let command = "SELECT * FROM fruit WHERE fruit = @fruit"
let param = dict [ "fruit_id", box fruitId ]
let! result = querySingleAsync command param
```



Instead of writing INSERT and UPDATE statements, Dapper comes with a bunch of extensions, such as [Dapper.Contrib](https://github.com/StackExchange/Dapper/tree/master/Dapper.Contrib), [Dapper.Rainbow](https://github.com/StackExchange/Dapper), etc.
Each of them is a bit different, for example Dapper.Rainbow requires the identity column name to called `Id`.
I chose Dapper.Contrib and ended up having all property names of my persistence types match the database columns exactly, in snake_case, because PostgreSQL has [interesting behaviour with non-lower case identifiers](https://stackoverflow.com/questions/2878248/postgresql-naming-conventions) and [Dapper.Contrib treats the Key-column differently than others](https://github.com/StackExchange/Dapper/issues/1008).

Then I thought, how hard can it be to achieve this in F#? 
The main goal is to avoid having to write boring `INSERT` and `UPDATE` SQL statements.
Let's start by getting the column names for our SQL statement utilizing .NET type reflection:
```fsharp
// a list of properties of type 'a
let propertyNames<'a> =
    typedefof<'a>.GetProperties()
    |> Array.map (fun p -> p.Name)
    |> Array.sort

// creates a dictionary of (property name, property value)-pairs that can be passed to Dapper
let getRecordParams a =
    a.GetType().GetProperties() 
    |> Array.map (fun p -> p.Name, p.GetValue(a)) 
    |> dict
```

Example:
```fsharp
[<CLIMutable>]
type Fruit = {
    fruit_id: Guid
    name: string
    weight_kg: int
}

let names = propertyNames<Fruit>
printf "%A" names
// [|"fruit_id"; "name"; "weight_kg"|]

let v = {
    fruit_id = Guid.NewGuid()
    name = "Orange"
    weight_kg = 1000
}

let parameters = getRecordParams v
printf "%A" parameters
// seq [[fruit_id, 6f9b206f-eed9-434a-a235-7ff0e6565b09]; [name, Orange]; [weight_kg, 1000]]
```

With these two helpers `propertyNames` and `getRecordParams` we can now build SQL statements.
```fsharp
let getInsertCommand<'a> tableName = 
    let names = propertyNames<'a>
    let columns = String.Join(", ", names)
    let placeholders = String.Join(", ", names |> Array.map (fun c -> "@" + c))

    sprintf "INSERT INTO %s (%s) VALUES (%s)" tableName columns placeholders

let getUpdateCommand<'a> tableName keyName =
    let names = propertyNames<'a>
    let assignments = String.Join(", ", names |> Array.map (fun n -> sprintf "%s = @%s" n n))

    sprintf "UPDATE %s SET %s WHERE %s = @%s" tableName assignments keyName keyName
```

Example:
```fsharp
getInsertCommand<Fruit> "fruit"
// "INSERT INTO fruit (fruit_id, name, weight_kg) VALUES (@fruit_id, @name, @weight_kg)"

getUpdateCommand<Fruit> "fruit" "fruit_id"
// "UPDATE fruit SET fruit_id = @fruit_id, name = @name, weight_kg = @weight_kg WHERE fruit_id = @fruit_id"
```

And now for the actual convenience functions:
(Note that we never embed data in our SQL command string, all data gets passed safely via `param`)
```fsharp
let insertAsync<'a> tableName a = async {
    use conn = new NpgsqlConnection(connectionString)

    let command = getInsertCommand<'a> tableName
    let param = getRecordParams a

    do! conn.ExecuteAsync(command, param) |> Async.AwaitTask |> Async.Ignore
}

let updateAsync<'a> tableName keyName a = async {
    use conn = new NpgsqlConnection(connectionString)

    let command = getUpdateCommand<'a> tableName keyName
    let param = getRecordParams a

    do! conn.ExecuteAsync(command, param) |> Async.AwaitTask |> Async.Ignore
}

let deleteAsync tableName keyName (id: obj) = async {
    use conn = new NpgsqlConnection(connectionString)

    let command = sprintf "DELETE FROM %s WHERE %s = @%s" tableName keyName keyName
    let param = dict [ keyName, box id ]

    do! conn.ExecuteAsync(command, param) |> Async.AwaitTask |> Async.Ignore
}
```

## Persistence types to domain types
Agreed, the snake_case properties in the `Fruit` type do look ugly but they are confined to your
infrastructure layer if you transform them into domain objects later.

## Mapping optional values
From https://stackoverflow.com/questions/42797288/dapper-column-to-f-option-property
```fsharp
type OptionHandler<'T>() =
    inherit SqlMapper.TypeHandler<'T option>()

    override __.SetValue(param, value) = 
        let valueOrNull = 
            match value with
            | Some x -> box x
            | None -> null

        param.Value <- valueOrNull    

    override __.Parse value =
        if isNull value || value = box DBNull.Value 
        then None
        else Some (value :?> 'T)

// call on startup
let setup () =
    Dapper.DefaultTypeMap.MatchNamesWithUnderscores <- true
    SqlMapper.AddTypeHandler (OptionHandler<DateTime>())
    SqlMapper.AddTypeHandler (OptionHandler<string>())
    SqlMapper.AddTypeHandler (OptionHandler<byte[]>())

    ()
```
