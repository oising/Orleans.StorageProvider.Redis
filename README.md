<p align="center">
  <img src="https://github.com/dotnet/orleans/blob/gh-pages/assets/logo.png" alt="Orleans.Redis" width="300px"> 
  <h1>Orleans Redis Providers</h1>
</p>

1.5.x branch 
[![Build status](https://ci.appveyor.com/api/projects/status/6xxnvi7rh131c9f1?svg=true)](https://ci.appveyor.com/project/OrleansContrib/orleans-storageprovider-redis)
2.x.x branch
[![Build status](https://ci.appveyor.com/api/projects/status/6xxnvi7rh131c9f1/branch/dev?svg=true)](https://ci.appveyor.com/project/OrleansContrib/orleans-storageprovider-redis/branch/dev)

A Redis implementation of the Orleans v3 Storage Provider model. Uses the Azure Redis Cache to persist grain states.

[Orleans](https://github.com/dotnet/orleans) is a framework that provides a straight-forward approach to building distributed high-scale computing applications, without the need to learn and apply complex concurrency or other scaling patterns. 

[Redis](https://redis.io/) is an open source (BSD licensed), in-memory data structure store, used as a database, cache and message broker.

**Orleans.Redis** is a package that use Redis as a backend for Orleans providers. It uses the great [StackExchange.Redis](https://stackexchange.github.io/StackExchange.Redis/) library underneath.


## Installation

> PS> Install-Package Orleans.Persistence.Redis -prerelease

## Usage

Configure your Orleans-cluster.

```cs
var silo = new SiloHostBuilder()
    .AddRedisGrainStorage("Redis", optionsBuilder => optionsBuilder.Configure(options =>
    {
        options.DataConnectionString = "localhost:6379"; // This is the deafult
        options.UseJson = true;
        options.DatabaseNumber = 1;
    }))
    .Build();
await silo.StartAsync();
```

Decorate your grain classes with the `StorageProvider` attribute.

 ```cs
[StorageProvider(ProviderName = "RedisGrainStorage")]
public class SomeGrain : Grain<SomeGrainState>, ISomeGrain
 ```

and update your appsettings.json configuration file as described below.

These settings will enable the redis cache to act as the store for grains that have 

* State
* Need to persist their state

## Configuration

Example Redis configuration (appsettings.json):
```json
    {
        "Logging": {
            ...
        },
        "Orleans" : {
            "Storage": {
            "Provider": "Redis",
            "Redis": {
                "DataConnectionString": "your-redis-server:6379",
                "DatabaseNumber": 1,
                "UseJson": false,
                "NumberOfConnectionRetries": 30,
                "DeltaBackOff": "00:00:10"
            }
        }
    }
```

Where:

* __UseJson=true/false__ (optional) Defaults to `true`, if set to `false` the Orleans binary serializer is used (this is recommended, as the JSON serializer is unable to serialize certain types).
* __DataConnectionString="..."__ (required) the connection string to your redis database (i.e. `<youraccount>.redis.cache.windows.net,abortConnect=false,ssl=true,password=<yourkey>`)
* __DatabaseNumber="1"__ (optional) the number of the redis database to connect to

Example initialization code:

```cs
    public static class Program
    {
        public static async Task Main(string[] args)
        {
            await CreateHostBuilder(args).Build().RunAsync();
        }

        private static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            ...
            .UseOrleans(config =>
            {
                var webHostBuilderContext = (WebHostBuilderContext)config.Properties.Values.First(v => v.GetType() == typeof(WebHostBuilderContext));
                var options = webHostBuilderContext.Configuration.GetSection("Orleans").Get<OrleansOptions>();

                config = options.Storage.Provider switch
                {
                    "Memory" => config.AddMemoryGrainStorageAsDefault(),
                    "Redis" => config.AddRedisGrainStorageAsDefault(storageOptions =>
                    {
                        storageOptions.DataConnectionString = options.Storage.Redis.DataConnectionString;
                        storageOptions.DatabaseNumber = options.Storage.Redis.DatabaseNumber;
                        storageOptions.UseJson = options.Storage.Redis.UseJson;
                        storageOptions.NumberOfConnectionRetries = options.Storage.Redis.NumberOfConnectionRetries;
                        storageOptions.DeltaBackOffMilliseconds = (int)options.Storage.Redis.DeltaBackOff.TotalMilliseconds;
                    }),
                    _ => config
                };

                ConfigureSiloBuilder(config, options);
            })
            ...
            .UseConsoleLifetime();

    }
```

## License

MIT
