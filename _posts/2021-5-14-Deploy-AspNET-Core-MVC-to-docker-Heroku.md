---
layout: post
title: Deploy ASP .NET Core MVC app with PostgreSQL to Heroku using GitHub Action
date:   2021-05-14 00:08:30 +0800
categories: .net core
---


## Getting Started
We need to have a few things ready before we start:

1.  Visual Studio 2019
    - I am using the [Community edition](https://visualstudio.microsoft.com/downloads/) of Visual Studio 2019.

2.  PostgreSQL

3.  Install docker by following the [steps here](https://docs.docker.com/get-docker/)

4.  Create a new account in [Heroku](https://signup.Heroku.com/)

5.  [Install Heroku cli](https://devcenter.Heroku.com/articles/Heroku-cli)

6.  A GitHub repository for the project that we want to deploy

## Setup a .NET Core MVC Project with PostgreSQL

1.  Create a new project using 'ASP.NET Core Web Application' template

    Follow the steps [here](https://docs.microsoft.com/en-us/aspnet/core/tutorials/first-mvc-app/start-mvc?view=aspnetcore-5.0&tabs=visual-studio) to setup the project for .NET Core 5.0

    We will use Model-View-Controller. Follow the steps in [Tutorial: Get started with EF Core in an ASP.NET MVC web app](https://docs.microsoft.com/en-us/aspnet/core/data/ef-mvc/intro?view=aspnetcore-5.0) to get started.

    After setting up the MVC project in Visual Studio, run the app to make sure it is working.


2.  We will setup a PostgreSQL database with Entity Framework Core for our web app.
    This step uses a local database. I will discuss how to get the app to work with PostgreSQL in Heroku later in this post.

    I followed the steps from [Getting Started with Entity Framework Core (PostgreSQL)](https://medium.com/@RobertKhou/getting-started-with-entity-framework-core-PostgreSQL-c6fa09681624).

    In pgAdmin, create a new database and a new table.

    In Visual Studio, install the following packages with Nuget package manager:
    1.  Npgsql

    2.  Npgsql.EntityFrameworkCore.PostgreSQL

    3.  Microsoft.EntityFrameworkCore.Design

3. Once we have a database and table, we will convert it to Models in C\#.

    In same directory as your project file, convert the database in PostgreSQL to Models. 
    
    Replace the fields in the following snippet with your own credentials.

    ``` bash
    dotnet ef dbcontext scaffold "Host=localhost;Database=demo\_db;Username=demo\_user;Password=demo\_password" Npgsql.EntityFrameworkCore.PostgreSQL -o Models
    ```

    [Install dotnet ef tool separately](https://stackoverflow.com/questions/57066856/dotnet-ef-not-found-in-net-core-3). If dotnet ef is not installed.

4.  After that, you should have two classes generated in the project:
    - Model class.
    - Context Class.
        - This class is used to perform CRUD operations to our database.

5.  Display the database items in your web page.

    I am creating a new view in a new `Demo()` method in the `HomeController`.

    You can choose to create create a new controller and a new view.

    In the `Demo()` method, add a new Razor View.
    -  Use the list template, and select your model class.

    -  The following snippet is my `Demo()` controller method in `HomeController.cs`

    ``` csharp
    public async Task<ActionResult> Demo()
    {
        IQueryable<DemoItem> items = from item in context.DemoItem 
        orderby item.Id select item;
        List<DemoItem> result = await items.ToListAsync();
        return View(result);
    }
    ```

    You should now have a new view generated in the Home folder under `Views/` folder

6.  Build and run your app locally to see the result. I use the following link to view my page.

    -  <https://localhost:44379/Home/Demo>

        -  `Home` is for `HomeController`

        -  `Demo` is my `Demo()` method in `HomeController`

7.  You should see a list view similar to (Picture From: [Microsoft docs](https://docs.microsoft.com/en-us/aspnet/mvc/overview/getting-started/getting-started-with-ef-using-mvc/creating-an-entity-framework-data-model-for-an-asp-net-mvc-application))

![_config.yml]({{ site.baseurl }}/images/post/may21/mvc-list-view.png)

## Setup Docker
1.  Create a Dockerfile project folder. The following is the a snippet of my Dockerfile.
    ``` docker
    FROM mcr.microsoft.com/dotnet/sdk:5.0-buster-slim AS build-env

    WORKDIR /app

    COPY DemoMVCApp.csproj ./

    RUN dotnet restore

    COPY . .

    RUN dotnet publish -c Release -o out

    FROM mcr.microsoft.com/dotnet/aspnet:5.0-buster-slim

    WORKDIR /app

    COPY --from=build-env /app/out .

    CMD ASPNETCORE\_URLS=http://\*:$PORT dotnet DemoMVCApp.dll
    ```

2. The Dockerfile will do the following steps:

    Download .NET core 5.0 sdk to build the project.

    Download all required Nuget packages and copy all required files to `/app` folder.

    Compiles the project and gives a `{project-name}.dll` file

    ``` Docker
    RUN dotnet publish -c Release -o out
    ```

    Download .NET core runtime to run the web app.

    We will have to supply `ASPNETCORE_URLS` variable for it to be run on Heroku.

3. The web app needs to be using the port number provided in Heroku. In `Program.cs` modify your `CreateHostBuilder()` method simillar to the following.
    ``` csharp
    public static IHostBuilder CreateHostBuilder(string[] args);

    Host.CreateDefaultBuilder(args)
    .ConfigureWebHostDefaults(webBuilder =>
    {

        var port = Environment.GetEnvironmentVariable("PORT");

        webBuilder.UseUrls($"http://+:{port}");

        webBuilder.UseStartup&lt;Startup&gt;();

    });
    ```
    This will get the `ASPNETCORE_URLS` variable passed in from the Dockerfile

    We will need to change the listening port for the app. Otherwise, the web app will not start properly in Heroku because Heroku will use a different port than the usual port 5000 and 5001. See [Practicalities of deploying dockerized ASP.NET Core application to Heroku](https://habr.com/en/post/450904/).

4.  In the project's directory, build the Dockerfile and make sure the it builds successfully.

    - `docker build -t {your-app-name} .`

    We will deploy this image to the Heroku registy in the next steps.

## Configure a new Heroku app

1.  Create a new Heroku app in Heroku website.

    After you have create a Heroku account, you will be able to create a new app in the dashboard.

    Your web app will be accessed through [https://{your-app-name}.Herokuapp.com](), after the your have deployed the app.

    Your app name should be the same as your Docker image name.

2.  Install PostgreSQL and create a table in Heroku.

    I followed the steps in [Getting Started with Heroku, PostgreSQL and PgAdmin by @vapurrmaid](https://medium.com/@vapurrmaid/getting-started-with-Heroku-postgres-and-pgadmin-run-on-part-2-90d9499ed8fb) to setup a PostgreSQL database on Heroku and use it with my project.

    Setup the Heroku database table in pgAdmin, so that its contents will be the same as the one in your local database.

3.  Now, you can change your database's connection string in your project to see if the web app works.

    Remember to change the connection string back when you are deploying to GitHub.

4.  In a command prompt, login to Heroku cli.

    ``` bash
    Heroku login
    Heroku container:login
    ```

5.  Configure Heroku to use containers

    ``` bash
    Heroku stack:set container
    ```

## Use GitHub Actions to deploy the app to Heroku

1.  In your GitHub repository, add three GitHub Secrets.

    We will use these values in GitHub actions workflow.

    - `Heroku_APP_NAME`
      - Name of your application

    - `DATABASE_URL`
      - Find this value is in your Heroku app -> Settings -> Config Vars

    - `Heroku_API_KEY`
      - Either reveal an existing api key or generate a new one in [Heroku Account page](https://dashboard.Heroku.com/account).
<br/><br/>
2.  Add a new GitHub workflow on the Github Actions tab in your GitHub repository. This will build and deploy the dockerized web app to Heroku whenever you push something new to GitHub.

    ```yml
    name: deploy-Heroku

    on:
        push:
            branches: [ master ]
        pull_request:
            branches: [ master ]
        workflow_dispatch:

    jobs:
        build:
            runs-on: ubuntu-latest
            steps:
            - uses: actions/checkout@v2
            - name: Build and deploy the Docker image
                env:
                Heroku_API_KEY: ${{ secrets.Heroku_API_KEY }}
                APP_NAME: ${{ secrets.Heroku_APP_NAME }}
                run: |
                docker login --username=_ --password=$Heroku_API_KEY registry.Heroku.com
                Heroku container:push web -a $APP_NAME
                Heroku container:release web -a $APP_NAME
    ```

3.  Configure Heroku connection string.

    Right now, if we deployed the dockerized web app, it will not connect to the Heroku database.

    We had supplied the enviroment variables in Dockerfile, now we need to modify the web app so that it will take in the variables in Heroku.

    Modify `ConfigureServices()` in `Startup.cs` to use Heroku's connection string when the web app is not running in development enviroment.  

    ``` csharp
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllersWithViews();

        var IsDevelopment = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") == "Development";

        var connectionString = IsDevelopment ? Configuration.GetConnectionString("DefaultConnection") : GetHerokuConnectionString();

        services.AddDbContext<TodoContext>(options => options.UseNpgsql(connectionString));
    }
    ```

    Add in the `GetHeroConnectionString()` method. When the web app is deployed in Heroku, this `ConfigureServices()` will use the this method to convert the `DATABASE_URL` variable from Heroku to the connection string that is expected by the .NET Core web app. 
    
    I modified the `GetHeroConnectionString()` function that I copied from this [post by @n1ghtmare](https://medium.com/@vapurrmaid/getting-started-with-Heroku-postgres-and-pgadmin-run-on-part-2-90d9499ed8fb).

    ``` csharp
    private static string GetHerokuConnectionString()
    {
        string connectionUrl = Environment.GetEnvironmentVariable("DATABASE_URL");

        var databaseUri = new Uri(connectionUrl);

        string db = databaseUri.LocalPath.TrimStart('/');
        string[] userInfo = databaseUri.UserInfo.Split(':', StringSplitOptions.RemoveEmptyEntries);

        return $"User ID={userInfo[0]};Password={userInfo[1]};Host={databaseUri.Host};Port={databaseUri.Port};Database={db};Pooling=true;SSL Mode=Require;Trust Server Certificate=True;";
    }
    ```

4.  Push to your GitHub repo, and the website is now live on [https://{your-app-name}.herokuapp.com]().

5.  Make sure the web app also runs locally wih local PostgreSQL database.
