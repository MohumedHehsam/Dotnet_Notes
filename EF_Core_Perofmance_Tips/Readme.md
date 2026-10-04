use DbContext.Entry(entry)

1-.State -> change state to (Added,Modified,Deleted,Detatched,Unchanged)

2-.property(x=>X...).CurrentValue/OriginalValue/IsModified -> used also to deal with shadow properties 

3-.Reference(x=>X.reference).Load()

4-.Reference(x=>X.reference).Query().Select/Load()

5-.Collection(x=>x.collection).Query().where(filteration).Select/Load();

6-.CurrentValues.SetValues(DTO/Dictionary);

7-PropertyValues dbValues = await context.Entry(product).GetDatabaseValuesAsync();
``` csharp
if (dbValues == null)
{
    // تم حذف السجل من قاعدة البيانات بواسطة مستخدم آخر!
}
else
{
    var currentDbPrice = dbValues.GetValue<decimal>(nameof(Product.Price));
}
```

8-.RelaodAsync() -> reload data with database

9-.Properties -> IEnumerable of properties of all fileds
## N + 1 Query Problem

**Example:** A `Department` has a collection of `Employees`. For each department processed, its employees must be loaded.

Executing any of the following inside a loop triggers a separate query per iteration ($N$ queries + 1 initial query):
- `dbContext.Entry(dept).Collection(d => d.Employees).Load();` *(Explicit Loading)*
- `dbContext.Employees.Where(x => x.DepartmentId == deptId).ToList();` *(Separate Query)*
- Accessing `dept.Employees` with **Lazy Loading** enabled

###  Select is better in memeory as it allows to filter data on database , while Eager Loading loads all data  EX. Select(x=>x.Employees.where(..))

### Solutions
1. **Projection (`.Select`):** Shape and project only the required department and employee fields in a single query.
2. **Eager Loading (`.Include`):** Load related entities upfront via `dbContext.Departments.Include(d => d.Employees)`.



## Using Lazy Loading 
it's not enabled by default , to enable it : 

- 1- install Microsoft.EntityFrameworkCore.Proxies
- 2- Configure the DbContext Call .UseLazyLoadingProxies() inside your DbContext.OnConfiguring method or within your dependency injection setup in Program.cs.
- 3- Entity classes must be public and not sealed & Make Navigation Properties virtual (both reference and collection types)
- 4: Usage : EF Core queries the related data automatically when the navigation property is accessed

#### so it makes 2 queries instead of one , if you have a collection N entities they take N + 1 queries , Select or .Include is better but again Select is even better as it allows database filteration inside it



## Select Specific Columns 
Any Query without `Select` retrieves all data from Database , which consumes more memeory,Netwok traffic and make query slower

## Execute Query as late as possible 

These Query methods enforce database to execute query first before continue : 
- Conversion Methods -> ToList() / ToDictionary ..
- Single-Value Methods -> Min() / Max() /First() / Last() / FirstOrDefult() ... 

it is better to call them as late as possible in query tree to allow database to execute as much query as possbile on database side not server side 

## Optimizing Delete 

#### ❌ WRONG : fetch data using first query , and uses another query to delete , not suitable for bulk delete (Ex. 1000 rows) as it consumes a lot of memeory
```CSharp
var user = await context.Users.FindAsync(id);
if (user != null)
{
    context.Users.Remove(user);
    await context.SaveChangesAsync();
}
```
#### ✅use `ExecuteDelete` : 1 query


```CSharp
await context.Users
    .Where(u => u.Id == id)
    .ExecuteDeleteAsync();
```


#### ✅use Stub Entity : 1 query

```CSharp
var user = new User { Id = id };
context.Users.Remove(user);
await context.SaveChangesAsync();
```

## Cold And Warm Queries
A delay happens at first query execution after app starts , the following same queries are warm queries

what happesn ?
when creating first instance of DbContext , it Read Models & Relations to create Mdoel `IModel`, compiling first query into expression tree and store it's compilation into cache so that first query is Cold and the following ones are warm and fast

Also JIT Have to compile IL Code of EF Core / ADO.NET first to machine code
The query might need server to generate a new execution plan for it 

so , the bigger your DbContext is and the more complex is first query , the slower the cold start

✅ optimize it using Compiled Models , Compiled Queries 

✅ there is also nuget packages (Ex. Ngen.exe & (R2R)) that reduce JIT Working by precompiled IL 

✅ you can also divide big DbContext into smaller ones to reduce time of Discovering `IModel`

## Compiled Queries 
EF Core analyze query each time before execting and store cached Expression tree but still analyze query each time , if Query is Complex it adds some delay to each query for anaylzing it 
Compiled Query Remove need for analyzing or expression tree compilation 

```CSharp
    // inside CompiledQueries.cs
    public static readonly Func<AppDbContext, int, Task<User?>> GetUserByIdCompiled =
        EF.CompileAsyncQuery((AppDbContext context, int id) =>
            context.Users
                   .FirstOrDefault(u => u.Id == id));

    // inside UserServices.cs
    public async Task<User?> GetUserAsync(int id)
    {
        return await GetUserByIdCompiled(dbContext, id);
    }

```
`Note` : They have only notable performance gain in Complex queries


Here is a clean, simple, and accurate rewrite:

## Change Tracking

By default, EF Core tracks changes for every queried entity by creating an entry to monitor its properties. When calling `SaveChanges()`, having fewer tracked entities makes the process faster.
For read-only queries where you do not need to update data, use `.AsNoTracking()` to avoid this tracking overhead, reduce memory allocations, and lower GC pressure.

```csharp
var users = await context.Users.AsNoTracking().ToListAsync();
```
`Note` : when AsNoTracking applied to root Query it extneds to `Include` entities too

####  EF Core don't track when
- projection with select and not adding the entity object to Projection result
- Keyless Entities 
- Scalar methods -> Sum()/Count()...
- Immediate Queries Executed using `ExecuteDelete` / `ExecuteUpdate`

## Client Evaluation
from EF Core 3.0 till now , you can use user-defined functions in only 
- final select statement
- after loading data into memory in query using `ToList`/`AsEnumerable`/....

#### `Note` : if user-defined method uses Entity Objects in `Select` it makes them tracked 

at any othe query statement (Ex. `where`,`OrderBy`) you have to tell Ef Core that your method name mapps to a method in database inside `OnModelCreating()


## clear ChangeTracker after bulck insert

after inserting many objects Ex. 5000 , call 
```CSharp
 context.ChangeTracker.Clear(); 
```
before continue working on same DbContext to avoid memory leak

## Enable MultipleActiveResultSets
adding `MultipleActiveResultSets=True;` to connection string enables multiple queries at same connection which means :
+ enhanced `SplitQuery` perfrmance and prevent Buffering reuslt of first table before sending first table 
+ reduce Connection Pool Exhaustion , as single operation previously required many connections 


