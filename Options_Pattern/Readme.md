Old Way to access settings 
```csharp
public class EmailService
{
    private readonly IConfiguration _configuration;

    public EmailService(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public void SendEmail()
    {
        var host = _configuration["Smtp:Host"];
        var port = _configuration["Smtp:Port"];
    }
}
```
### The options pattern uses classes to provide strongly typed access to groups of related settings, using it achive 2 design patterns 
+ Encapsulation : Service only see configurations that it needs
+ Seperation Of Conecern : service don't need to know the key name of configurations
+ Can validate settings before throwning exception

## How to use the options pattern
- store settings int appsettings.json 
```json
"Position": {
  "Name": "Joe Smith",
  "Title": "Editor"
}
```
- Create Options Class with public read-write properties that match corresponding entries , The Position field is used to avoid hardcoding the string "Position" , only public 

```csharp
public class PositionOptions
{
    public const string Position = "Position";

    public string? Name { get; set; }
    public string? Title { get; set; }
}
```


##  First Method : Using IConfiguration.Bind 
same as old method but more automatic 
```csharp
positionOptions = new PositionOptions();
InjectedIConfiguration.GetSection(PositionOptions.Position).Bind(positionOptions);
//OR simply
positionOptions = Config.GetSection(PositionOptions.Position)
            .Get<PositionOptions>();
```
* ### changes to the JSON configuration in the app settings file are read without app restart

## Second Method : Options added to service container using Configure
in Program.cs
```csharp
builder.Services.Configure<PositionOptions>(builder.Configuration.GetSection(PositionOptions.Position));
```
then inject either of these types into your service 
1. `IOptions<PositionOptions>`
2. `IOptionsSnapshot<PositionOptions>` 
3. `IOptionsMonitor<PositionOptions>` 

### 📊 Options Interfaces Comparison

| Interface | Service Lifetime | Can Inject Into Singleton? | Reads Updated Config (Reloadable) | Named Options | Change Notifications | Primary Use Case |
| :--- | :---: | :---: | :---: | :---: | :---: | :--- |
| **`IOptions<TOptions>`** | **Singleton** | ✅ Yes | ❌ No *(Read-once at start)* | ❌ No | ❌ No | Fixed settings that don't change during app runtime (e.g., core infrastructure). |
| **`IOptionsSnapshot<TOptions>`** | **Scoped** | ❌ No *(Captive Dependency)* | ✅ Yes *(Recomputed per request)* | ✅ Yes | ❌ No | Per-request dynamic settings in web apps (controllers, scoped services). |
| **`IOptionsMonitor<TOptions>`** | **Singleton** | ✅ Yes | ✅ Yes *(Real-time updates)* | ✅ Yes | ✅ Yes (`OnChange`) | Singletons or background services needing real-time reloaded settings. |

---

### ⚙️ Supporting Infrastructure Interfaces

| Interface | Lifetime / Role | Key Responsibilities & Behavior |
| :--- | :--- | :--- |
| **`IOptionsFactory<TOptions>`** | Transient / Factory Service | • Responsible for instantiating new options instances via its `Create(name)` method.<br>• Executes all registered `IConfigureOptions<T>` and `IConfigureNamedOptions<T>` first.<br>• Runs all `IPostConfigureOptions<T>` afterward to finalize configuration. |
| **`IOptionsMonitorCache<TOptions>`** | Cache Provider | • Provides selective options invalidation and caching specifically for `IOptionsMonitor<T>`. |
| **`IPostConfigureOptions<TOptions>`** | Post-processing | • Allows modifying or validating options **after** all initial configurations have been applied. |

### NOTE : Avoid `IOptionsSnapshot` and use Other 2 interfaces, as in high throughput apps many instances of `IOptionsSnapshot` are created for each request and have perofrmance penalty 

+ ### `IOptionsSnapshot` uses `OptionsCache<TOptions>` to cahce `TOptions` to provide same value during single request lifetime only 
+ ### `IOptionsMonitor` uses `OptionsMonitorCache<TOptions>` to cahce `TOptions`, allows cache (Clear/Invalidation/Manal Adding) and cache updates when value change automatically
+ ### `IOptions` only makes lazy initialization for each TOptions as it doesn't have named Options 

## Named Options 
#### are case-sensitive names to different configurations but same properties 

Consider the following JSON configuration:

```JSON
"TopItem": {
  "Month": {
    "Name": "Green Widget",
    "Model": "GW46"
  },
  "Year": {
    "Name": "Orange Gadget",
    "Model": "OG35"
  }
}
```


Rather than creating two classes to bind TopItem:Month and TopItem:Year, the following class is used for each section.


```csharp
public class TopItemSettings
{
    public const string Month = "Month";
    public const string Year = "Year";

    public string? Name { get; set; }
    public string? Model { get; set; }
}
```
configures the named options:

```csharp
builder.Services.Configure<TopItemSettings>(TopItemSettings.Month,
    builder.Configuration.GetSection("TopItem:Month"));
builder.Services.Configure<TopItemSettings>(TopItemSettings.Year,
    builder.Configuration.GetSection("TopItem:Year"));
```
use injected options 
```csharp
class MyService(IOptionsSnapshot<TopItemSettings> Options)

//Somewhere in service 
monthTopItem = Options.Get(TopItemSettings.Month);
yearTopItem = Options.Get(TopItemSettings.Year);

```


## How Options Naming works internally 
all options are name instances 
+ `services.Configure<TOptions>(...)` add configuration to TOptions with name `String.Empty`
+ `services.Configure<TOptions>(string Name,...)` add Configuration to TOptions with name `Name` 
+ `services.ConfigureAll<TOptions>(...)`add Configuration to TOptions with name `Null` , also implemented to all named and non-named instances too 
+ `services.PostConfigureAll<TOptions>(...)`add Configuration to TOptions with name `Null` , also implemented to all named and non-named instances too 


both `IOptionsSnapshot<TOptions>` , `IOptionsMonitor<TOptions>` support these methods 
```csharp
var defaultOpt = snapshot.Value;               // Default instance (string.Empty)
var specificOpt = snapshot.Get("FeatureAdmin"); // Named instance "FeatureAdmin"
```

## `OptionsBuilder` Api
it is a more better option to configuring Options in Program.cs than `builder.Services.Configure<TOptions>(....)` as :
+ it makes configuring named options easier
+ allows using DI Services to configure an option
+ allows validation at startup to avoid runtime exceptions , and manual validation as well 
you need to call `AddOptions<TOptions>()` first 
```csharp
builder.Services.AddOptions<SmtpOptions>()
    .Bind(builder.Configuration.GetSection(SmtpOptions.SectionName)) //no need to recall option name each time in subsequent configurations as before
    .Configure<IWebHostEnvironment>((options, env) =>       //`env` is an injected service
    {
        if (env.IsDevelopment())
        {
            options.Port = 1025; 
        }
    }).PostConfigure<IWebHostEnvironment>((options, env) =>
    {
        if (env.IsProduction())
        {
            options.Port = 2525;
        }
    });
    .ValidateDataAnnotations()
    .Validate(opt => opt.Host.EndsWith(".com") || opt.Host == "localhost", 
              "Host must be a valid domain ending with .com or localhost")  //manual validation
    .ValidateOnStart(); // validate at project startup
```

if you don't use `ValidateOnStart` Exception might be thrown when accessing value 
```csharp
IOptionsSnapshot<KeyOptions> Options; //Injected options
try
{
    var keyOptions = Options.Value;
}
catch (OptionsValidationException ex)
{
    ...
}
```
## Post Configuration 
Options Creation Pipeline is as follows 

1. Configure
2. PostConfigure
3. Validation

Post Configuration is used to Add Configuration to Option after it is Configured , for example to add smart 'Defaults' or Override 3rd-party library configurations 

```csharp
builder.Services.AddSomeThirdPartyAuthentication();

builder.Services.PostConfigure<AuthenticationOptions>(options =>
{
    options.DefaultScheme = "CustomScheme";
});
```
* ###  Note : there is also `PostConfigureAll` to post-configre all named options 

## Class-level validation with `IValidatableObject`
used to make complex validation or validation that requires the value of multiple properties , you need to :
+ Option Class need to implement `IValidatableObject` 
+ Call `ValidateDataAnnotations()` in the Program file.

