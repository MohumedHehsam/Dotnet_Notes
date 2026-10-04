## All Ways to access a confgiuration 
to start using Configurations , Inject `IConfiguration` instance 

```Csharp
Configuration["myKey"];
Configuration.GetValue<string>("MyKey","Fall back value");
Configuration["Api:Uri"] ?? "Fall back value"; // Hierarchy 
Configuration.GetSection("Api").Bind(new ApiOptions());
var apiOptions = Configuration.GetSection("Api").Get<ApiOptions>();

//Options Pattern
// in program.cs
builder.Configure<ApiOptions>(Configuration.GetSection("Api"));
//inject into consumer class's Constructor
class MyClass (IOptions<ApiOptions> options)

```
#### Configuration Keys are key-insensetive

Default Configuration Soruces (the latter override earlier sources)

1. `appsettings.json` (Optional)
2. `appsettings.{Environment}.json` (Optional)
3. User Secrets (Only Works in `Developement`)
4. Environment Variables (Ex. Container/OS variables)
5. Command-Line Arguments (Ex. `dotnet run --Logging:LogLevel:Default=Debug`)

