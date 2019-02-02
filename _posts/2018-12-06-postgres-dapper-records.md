---
title:  "Lightweight persistence layer with PostgreSQL, Dapper and F# records"
comments: true
---
## Lightweight persistence layer with PostgreSQL, Dapper and F# records

I needed a simple way to store F# records into a postgres database.

Using [Npgsql](http://www.npgsql.org/) and [Dapper](https://github.com/StackExchange/Dapper)
querying the database is quite easy:
```fsharp
let connectionString = Environment.GetEnvironmentVariable "CONNECTION_STRING"

let querySingleAsync<'a> command param = async {
    use conn = new NpgsqlConnection(connectionString)

    let! result = conn.QuerySingleOrDefaultAsync<'a>(command, param) |> Async.AwaitTask
    if isNull (box result) then
        return None
    else
        return Some result
}
```

```fsharp
let fruitId = Guid.Parse("6f9b206f-eed9-434a-a235-7ff0e6565b09")
let command = "SELECT * FROM fruit WHERE fruit = @fruit"
let param = dict [ "fruit_id", box fruitId ]
let! result = querySingleAsync command param
```

For adding and modifying records, you need to write lengthy INSERT and UPDATE statements because you have to name the column for each record label.
There are a number of Dapper extensions to assist you, such as [Dapper.Contrib](https://github.com/StackExchange/Dapper/tree/master/Dapper.Contrib), [Dapper.Rainbow](https://github.com/StackExchange/Dapper), etc.

After looking at the [options available](https://dapper-tutorial.net/third-party-library)
I chose Dapper.Contrib but wasn't happy with it: Because PostgreSQL has [interesting behaviour with non-lower case identifiers](https://stackoverflow.com/questions/2878248/postgresql-naming-conventions) and [Dapper.Contrib treats the Key-column differently than others](https://github.com/StackExchange/Dapper/issues/1008) I ended up having all property names of my persistence types match my snake_case database column names exactly. Yikes!

Then I decided to roll my own in F#:

The main goal is to avoid having to write boring `INSERT` and `UPDATE` SQL statements.
Let's start by building a list of column names for our SQL statement using .NET type reflection:
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
// needs CLIMutableAttribute so Dapper can use property setters to create the object
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
    let columns = String.Join(", ", propertyNames<'a>)
    let placeholders = String.Join(", ", propertyNames<'a> |> Array.map (fun c -> "@" + c))

    sprintf "INSERT INTO %s (%s) VALUES (%s)" tableName columns placeholders

let getUpdateCommand<'a> tableName keyName =
    let assignments =
        propertyNames<'a>
        |> Array.map (fun n -> sprintf "%s = @%s" n n)
    let assignmentsString = String.Join(", ", assignments)

    sprintf "UPDATE %s SET %s WHERE %s = @%s" tableName assignmentsString keyName keyName
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

let loadAsync<'a> tableName keyName (id: obj) = async {
    use conn = new NpgsqlConnection(connectionString)

    let command = sprintf "SELECT * FROM %s WHERE %s = @%s" tableName keyName keyName
    let param = readOnlyDict [ keyName, id ]

    return! querySingleAsync<'a> command param
}
```

## Using pascal case
In case you don't want to mix pascal and snake case in your code we can add some case conversion logic into `getInsertCommand` and `getUpdateCommand` like this:

```fsharp
let pascalToSnake pascalCaseName =
    pascalCaseName
    // any upper case character gets replaced by an underscore plus the lower case character
    |> Seq.map (fun c -> if Char.IsUpper(c) then ['_'; Char.ToLower(c)] else [c])
    |> Seq.concat
    |> Seq.skip 1   // skip the first underscore
    |> Seq.toArray
    |> String

let columnNames<'a> = 
    propertyNames<'a>
    |> Seq.map pascalToSnake
    |> Seq.sort
    |> Seq.toArray


// we can now rewrite our two SQL statement builder functions
let getInsertCommand<'a> tableName = 
    let columns = String.Join(", ", columnNames<'a>)
    let placeholders = String.Join(", ", propertyNames<'a> |> Array.map (fun c -> "@" + c))

    sprintf "INSERT INTO %s (%s) VALUES (%s)" tableName columns placeholders

let getUpdateCommand<'a> tableName idPropertyName =
    let assignments =
        (columnNames<'a>, propertyNames<'a>)
        ||> Array.map2 (fun c p -> sprintf "%s = @%s" c p)

    let assignmentsString = String.Join(", ", assignments)

    let idColumnName = pascalToSnake idPropertyName

    sprintf "UPDATE %s SET %s WHERE %s = @%s" tableName assignmentsString idColumnName idPropertyName
```

Example:
```fsharp

type UpperFruit = {
    FruitId: Guid
    Name: string
    WeightKg: int
}

getInsertCommand<UpperFruit> "fruit"
// INSERT INTO fruit (fruit_id, name, weight_kg) VALUES (@FruitId, @Name, @WeightKg)

getUpdateCommand<UpperFruit> "fruit" "FruitId"
// UPDATE fruit SET fruit_id = @FruitId, name = @Name, weight_kg = @WeightKg WHERE fruit_id = @FruitId
```

## Source Code
[https://github.com/gubser/dapper-npgsql-persistence](https://github.com/gubser/dapper-npgsql-persistence)
