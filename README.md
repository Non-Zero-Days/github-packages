# GitHub Packages

**Watch the video [here]()**

## Prerequisites

- Install [Docker Desktop](https://hub.docker.com/editions/community/docker-ce-desktop-windows)
- [.NET 6](https://dotnet.microsoft.com/download/dotnet/6.0)

## Loose Agenda

- Build a container image
- Store our built container image in GitHub Packages

## Step by Step

### Setup Playground

Create a new directory for this exercise and navigate a terminal instance to the new directory.

### Create a Web API

From the terminal instance run `dotnet new webapi -n non-zero-packages -o .`

Open `Program.cs`

- Remove `app.UseHttpsRedirection();` from line 19.
- Remove `if (app.Environment.IsDevelopment())` surrounding to enable swagger on all builds.

Your `Program.cs` should now look something like:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add services to the container.

builder.Services.AddControllers();
// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configure the HTTP request pipeline.
app.UseSwagger();
app.UseSwaggerUI();

app.UseHttpsRedirection();

app.UseAuthorization();

app.MapControllers();

app.Run();

```

This will create a .NET WebAPI in the current directory. You can run `dotnet build` to verify it produced a viable application.

### Add a New Endpoint

Let's create a new endpoint that prints out a configuration value. 

Open `Controllers\WeatherForecastController.cs` and add `IConfiguration` to the injected dependencies.

```csharp
    private readonly IConfiguration _configuration;

    public WeatherForecastController(ILogger<WeatherForecastController> logger, IConfiguration configuration)
    {
        _logger = logger;
        _configuration = configuration;
    }
```

Add a new `GET` endpoint which returns the value of the `NON_ZERO_VALUE` configuration.

```csharp
    [HttpGet("configuration")]
    public string NonZeroConfiguration()
    {
        return _configuration.GetValue<string>("NON_ZERO_VALUE");
    }
```

### Add a Dockerfile

In the root directory, add a new file named `Dockerfile`.

Add the following content to the `Dockerfile`

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build-env
WORKDIR /app

COPY *.csproj ./
RUN dotnet restore

COPY . ./
RUN dotnet publish -c Release -o out

FROM mcr.microsoft.com/dotnet/aspnet:6.0.0 as final
WORKDIR /app
COPY --from=build-env /app/out .

EXPOSE 80
ENTRYPOINT ["dotnet", "non-zero-packages.dll"]

```

### Build and Run the Image

From the terminal in the root directory, run `docker build -t webapi .`

Now let's run the produced image while setting our `NON_ZERO_VALUE` configuration.

`docker run -it -e NON_ZERO_VALUE=day -p 803:80 webapi`

Open a browser to [http://localhost:803/swagger](http://localhost:803/swagger). 

Call the `/WeatherForecast/configuration` endpoint and note the response is our configured value `day`.

### Push to GitHub Packages

For this step, you'll need to generate a Personal Access Token (PAT) via the [tokens page on GitHub](https://github.com/settings/tokens). Generate a new token with the `write:packages` permission.

**NOTE - For the purpose of this exercise I'm going to store a PAT in a session variable `$CR_PAT`.**

Log in to the GitHub Packages container registry `ghcr.io` by running `echo $CR_PAT | docker login ghcr.io -u USERNAME --password-stdin`

Tag the webapi image by running `docker tag webapi ghcr.io/YOUR_GITHUB_USERNAME/non-zero-packages:latest`

Push the image to GitHub Packages by running `docker push ghcr.io/YOUR_GITHUB_USERNAME/non-zero-packages:latest`

Navigate to GitHub and check out the `Packages` tab on the associated profile. You should see your image!

On the right of the image's page you should see `Package settings` wherein you can change the visibility or delete your image. 

Congratulations on a non-zero day!

## Additional Resources

- [GitHub Packages Documentation](https://docs.github.com/en/packages)
