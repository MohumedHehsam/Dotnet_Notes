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


