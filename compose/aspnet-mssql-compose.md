---
description: Create a Docker Compose application using ASP.NET Core and SQL Server on Linux in Docker.
keywords: dotnet, .NET, Core, example, ASP.NET Core, SQL Server, mssql
title: "Quickstart: Compose and ASP.NET Core with SQL Server"
---

This quick-start guide demonstrates how to use Docker Engine on Linux and Docker
Compose to set up and run the sample ASP.NET Core application using the
[ASP.NET Core Build image](https://hub.docker.com/r/microsoft/aspnetcore-build/)
with the
[SQL Server on Linux image](https://hub.docker.com/r/microsoft/mssql-server-linux/).
You just need to have [Docker Engine](https://docs.docker.com/engine/installation/)
and [Docker Compose](https://docs.docker.com/compose/install/) installed on your
platform of choice: Linux, Mac or Windows.

For this sample, we will create a sample .NET Core Web Application using the
`aspnetcore-build` Docker image. After that, we will create a `Dockerfile`,
configure this app to use our SQL Server database, and then create a
`docker-compose.yml` that will define the behavior of all of these components.

> **Note**: This sample is made for Docker Engine on Linux. For Windows
> Containers, visit
> [Docker Labs for Windows Containers](https://github.com/docker/labs/tree/master/windows).

1.  Create a new directory for your application.

    This directory will be the context of your docker-compose project. For
    [Docker for Windows](https://docs.docker.com/docker-for-windows/#/shared-drives) and
    [Docker for Mac](https://docs.docker.com/docker-for-mac/#/file-sharing), you
    have to set up file sharing for the volume that you need to map.

1.  Within your directory, use the `aspnetcore-build` Docker image to generate a
    sample web application within the container under the `/app` directory and
    into your host machine in the working directory:

    ```bash
    $ docker run -v ${PWD}:/app --workdir /app microsoft/aspnetcore-build:lts dotnet new mvc --auth Individual
    ```

    > **Note**: If running in Docker for Windows, make sure to use Powershell
    or specify the absolute path of your app directory.

1.  Create a `Dockerfile` within your app directory and add the following content:

    ```conf
    FROM microsoft/aspnetcore-build:lts
    COPY . /app
    WORKDIR /app
    RUN ["dotnet", "restore"]
    RUN ["dotnet", "build"]
    EXPOSE 80/tcp
    RUN chmod +x ./entrypoint.sh
    CMD /bin/bash ./entrypoint.sh
    ```

    This file defines how to build the web app image. It will use the
    [microsoft/aspnetcore-build](https://hub.docker.com/r/microsoft/aspnetcore-build/),
    map the volume with the generated code, restore the dependencies, build the
    project and expose port 80. After that, it will call an `entrypoint` script
    that we will create in the next step.

1.  The `Dockerfile` makes use of an entrypoint to your webapp Docker
    image. Create this script in a file called `entrypoint.sh` and paste the
    contents below.

    > **Note**: Make sure to use UNIX line delimiters. The script won't work if
    > you use Windows-based delimiters (Carriage return and line feed).

    ```bash
    #!/bin/bash

    set -e
    run_cmd="dotnet run --server.urls http://*:80"

    until dotnet ef database update; do
    >&2 echo "SQL Server is starting up"
    sleep 1
    done

    >&2 echo "SQL Server is up - executing command"
    exec $run_cmd
    ```

    This script will restore the database after it starts up, and then will run
    the application. This allows some time for the SQL Server database image to
    start up.

1.  Create a `docker-compose.yml` file. Write the following in the file, and
    make sure to replace the password in the `SA_PASSWORD` environment variable
    under `db` below. This file will define the way the images will interact as
    independent services.

    > **Note**: The SQL Server container requires a secure password to startup:
    > Minimum length 8 characters, including uppercase and lowercase letters,
    > base 10 digits and/or non-alphanumeric symbols.

    ```yaml
    version: "3"
    services:
        web:
            build: .
            ports:
                - "8000:80"
            depends_on:
                - db
        db:
            image: "microsoft/mssql-server-linux"
            environment:
                SA_PASSWORD: "your_password"
                ACCEPT_EULA: "Y"
    ```

    This file defines the `web` and `db` micro-services, their relationship, the
    ports they are using, and their specific environment variables.

1.  Go to `Startup.cs` and locate the function called `ConfigureServices` (Hint:
    it should be under line 42). Replace the entire function to use the following
    code (watch out for the brackets!).

    > **Note**: Make sure to update the `Password` field in the `connection`
    > variable below to the one you defined in the `docker-compose.yml` file.

    ```csharp
    [...]
    public void ConfigureServices(IServiceCollection services)
    {
        // Database connection string.
        // Make sure to update the Password value below from "your_password" to your actual password.
        var connection = @"Server=db;Database=master;User=sa;Password=your_password;";

        // This line uses 'UseSqlServer' in the 'options' parameter
        // with the connection string defined above.
        services.AddDbContext<ApplicationDbContext>(
            options => options.UseSqlServer(connection));

        services.AddIdentity<ApplicationUser, IdentityRole>()
            .AddEntityFrameworkStores<ApplicationDbContext>()
            .AddDefaultTokenProviders();

        services.AddMvc();

        // Add application services.
        services.AddTransient<IEmailSender, AuthMessageSender>();
        services.AddTransient<ISmsSender, AuthMessageSender>();
    }
    [...]
    ```

1.  Ready! You can now run the `docker-compose build` command.

    ```bash
    $ docker-compose build
    ```

1.  Make sure you allocate at least 4GB of memory to Docker Engine. Here is how
    to do it on
    [Docker for Mac](https://docs.docker.com/docker-for-mac/#/advanced) and
    [Docker for Windows](https://docs.docker.com/docker-for-windows/#/advanced).
    This is necessary to run the SQL Server on Linux container.

1.  Run the `docker-compose up` command. After a few seconds, you should be able
    to open [localhost:8000](http://localhost:8000) and see the ASP.NET core
    sample website. The application is listening on port 80 by default, but we
    mapped it to port 8000 in the `docker-compose.yml`.

    ```bash
    $ docker-compose up
    ```

    Go ahead and try out the website! This sample will use the SQL Server
    database image in the back-end for authentication.

Ready! You now have a ASP.NET Core application running against SQL Server in
Docker Compose! This sample made use of some of the most popular Microsoft
products for Linux. To learn more about Windows Containers, check out
[Docker Labs for Windows Containers](https://github.com/docker/labs/tree/master/windows)
to try out .NET Framework and more SQL Server tutorials.

## Next steps

- [Build your app using SQL Server](https://www.microsoft.com/en-us/sql-server/developer-get-started/?utm_medium=Referral&utm_source=docs.docker.com)
- [SQL Server on Docker Hub](https://hub.docker.com/r/microsoft/mssql-server-linux/)
- [ASP.NET Core](https://www.asp.net/core)
- [ASP.NET Core Docker image](https://hub.docker.com/r/microsoft/aspnetcore/) on DockerHub
