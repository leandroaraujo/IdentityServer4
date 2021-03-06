.. _refEntityFrameworkQuickstart:
Using EntityFramework Core for configuration data
=================================================

IdentityServer is designed for extensibility, and one of the extensibility points is the storage mechanism used for data that IdentityServer needs.
This quickstart shows to how configure IdentityServer to use EntityFramework (EF) as the storage mechanism for this data (rather than using the in-memory implementations we had been using up until now).

IdentityServer4.EntityFramework
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

There are two types of data that we are moving to the database. 
The first is the configuration data (scopes and clients).
The second is operational data that IdentityServer produces as it's being used.
These stores are modeled with interfaces, and we provide an EF implementation of these interfaces in the `IdentityServer4.EntityFramework` Nuget package.

Get started by adding a reference to the `IdentityServer4.EntityFramework` Nuget package in `project.json` in the IdentityServer project:: 

    "IdentityServer4.EntityFramework": "1.0.0-rc2"

Adding SqlServer
^^^^^^^^^^^^^^^^

Given EF's flexibility, you can then use any EF-supported database.
For this quickstart we will use the LocalDb version of SqlServer that comes with Visual Studio.

To add SqlServer, we need several more dependencies. 
In the `"dependencies"` section in `project.json` add these packages::

  "Microsoft.EntityFrameworkCore.SqlServer": "1.0.0",
  "Microsoft.EntityFrameworkCore.Tools": "1.0.0-preview2-final"

And then in the `"tools"` section add this configuration::

    "Microsoft.EntityFrameworkCore.Tools": "1.0.0-preview2-final"

Configuring the stores
^^^^^^^^^^^^^^^^^^^^^^

The next step is to replace the current calls to ``AddInMemoryClients`` and ``AddInMemoryScopes`` in the ``Configure`` method in `Startup.cs`.
We will replace them with this code::

  using Microsoft.EntityFrameworkCore;
  using System.Reflection;

  public void ConfigureServices(IServiceCollection services)
  {
      services.AddMvc();

      var connectionString = @"server=(localdb)\mssqllocaldb;database=IdentityServer4;trusted_connection=yes";
      var migrationsAssembly = typeof(Startup).GetTypeInfo().Assembly.GetName().Name;
            
      // configure identity server with in-memory users, but EF stores for clients and scopes
      services.AddDeveloperIdentityServer()
          .AddInMemoryUsers(Config.GetUsers())
          .AddConfigurationStore(builder =>
              builder.UseSqlServer(connectionString, options =>
                  options.MigrationsAssembly(migrationsAssembly)))
          .AddOperationalStore(builder =>
              builder.UseSqlServer(connectionString, options =>
                  options.MigrationsAssembly(migrationsAssembly)));
  }

The above code is hard-coding a connection string, which you should feel free to change if you wish.
Also, the calls to ``AddConfigurationStore`` and ``AddOperationalStore`` are registering the EF-backed store implementations.

The "builder" callback function passed to these APIs is the EF mechanism to allow you to configure the ``DbContextOptionsBuilder`` for the ``DbContext`` for each of these two stores.
This is how our ``DbContext`` classes can be configured with the database provider you want to use.
In this case by calling ``UseSqlServer`` we are using SqlServer.
As you can also tell, this is where the connection string is provided.

The "options" callback function in ``UseSqlServer`` is what configures the assembly where the EF migrations are defined.
EF requires the use of migrations to define the schema for the database. 

.. Note:: It is the responsibility of your hosting application to define these migrations, as they are specific to your database and provider.

We'll add the migrations next.

Adding migrations
^^^^^^^^^^^^^^^^^

To create the migrations, open a command prompt in the IdentityServer project directory.
In the command prompt run these two commands::

   dotnet ef migrations add InitialIdentityServerMigration -c PersistedGrantDbContext 
   dotnet ef migrations add InitialIdentityServerMigration -c ConfigurationDbContext

It should look something like this:

.. image:: images/8_add_migrations.png

You should now see a `~/Migrations` folder in the project. 
This contains the code for the newly created migrations.

Initialize the database
^^^^^^^^^^^^^^^^^^^^^^^

Now that we have the migrations, we can write code to create the database from the migrations.
We will also seed the database with the in-memory configuration data that we defined in the previous quickstarts.

In `Startup.cs` add this method to help initialize the database::

    private void InitializeDatabase(IApplicationBuilder app)
    {
        using (var scope = app.ApplicationServices.GetService<IServiceScopeFactory>().CreateScope())
        {
            scope.ServiceProvider.GetRequiredService<PersistedGrantDbContext>().Database.Migrate();

            var context = scope.ServiceProvider.GetRequiredService<ConfigurationDbContext>();
            context.Database.Migrate();
            if (!context.Clients.Any())
            {
                foreach (var client in Config.GetClients())
                {
                    context.Clients.Add(client.ToEntity());
                }
                context.SaveChanges();
            }

            if (!context.Scopes.Any())
            {
                foreach (var client in Config.GetScopes())
                {
                    context.Scopes.Add(client.ToEntity());
                }
                context.SaveChanges();
            }
        }
    }

And then we can invoke this from the ``Configure`` method::

    public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
    {
        // this will do the initial DB population
        InitializeDatabase(app);

        // the rest of the code that was already here
        // ...
    }

Now if you run the IdentityServer project, the database should be created and seeded with the quickstart configuration data.
You should be able to use SqlServer Management Studio or Visual Studio to connect and inspect the data.

.. image:: images/8_database.png


Run the client applications
^^^^^^^^^^^^^^^^^^^^^^^^^^^

You should now be able to run any of the existing client applications and sign-in, get tokens, and call the API -- all based upon the database configuration.


