# SQL Server Containers Lab

## Pre Lab
1. Install docker for Windows:

https://docs.docker.com/docker-for-windows/install/

2. Install Azure Data Studio

https://docs.microsoft.com/en-us/sql/azure-data-studio/download?view=sql-server-ver15

---

## Lab
### 1. Getting started with SQL Server in Containers

#### Introduction
In this section you will run SQL Server in a container and connect to it with SSMS/Azure Data Studio. This is the easiest way to get started with SQL Server in containers.  
 
#### Steps
1. Change the `SA_PASSWORD` in the command below and run it in your terminal:
``` 
docker run -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=YourStrong!Passw0rd' `
      -p 1500:1433 --name sql1 `
      -d mcr.microsoft.com/mssql/server:2019-CTP3.2-ubuntu      
 ```

> Tip: edit commands in a text editor prior to pasting in the terminal to easily edit the commands.
>
> Note: By default, the password must be at least 8 characters long and contain characters from three of the following four sets: Uppercase letters, Lowercase letters, Base 10 digits, and Symbols.

 
2. Check that SQL Server is running:
```
docker ps
```

![GettingStartedResults.PNG](/Media/Container-GettingStartedResults.png)

3. Connect to SQL Server in container using SSMS or Azure Data Studio.

Open SSMS or Azure Data Studio and connect to the SQL Server in container instance by connecting host:

```
localhost,1500
```
![GettingStartedOpsStudio.PNG](/Media/Container-GettingStartedDataStudio.png)

3. Run SQLCMD inside the container. First run bash interactively in the container with docker execute 'bash' inside 'sql1' container. 

```
docker exec -it sql1 bash
```
Use SQLCMD within the container to connect to SQL Server:
```
/opt/mssql-tools/bin/sqlcmd -U SA -P 'YourStrong!Passw0rd'
```
![sqlcmd.PNG](/Media/Container-ExecSQLCMD.png)

Exit SQLCMD and the container with exit:
```
exit
```
7. Clean up the container
```
docker stop sql1
docker container rm sql1
```

> **Key Takeaway**
> 
>SQL Server running in a container is the same SQL Server engine as it is on Linux OS or Windows.
 
---

### 2.  Build your own container 

#### Introduction:
In the past, if you were to set up a new SQL Server environment or dev test, your first order of business was to install a SQL Server onto your machine. But, that creates a situation where the environment on your machine might not match test/production.

With Docker, you can get SQL Server as an image, no installation necessary. Then, your build can include the base SQL Server image right alongside any additional environment needs, ensuring that your SQL Server instance, its dependencies, and the runtime, all travel together.

In this section you will build your a own container layered on top of the SQL Server image. 

Scenario: Let's say for testing purposes you want to start the container with the same state. We’ll copy a .bak file into the container which can be restored with T-SQL.  

 
#### Steps:

1. Change directory to the `mssql-custom-image-example folder`.
```
cd mssql-custom-image-example/
```

2. Create a Dockerfile with the following contents
```
FROM microsoft/mssql-server-linux:latest
COPY ./SampleDB.bak /var/opt/mssql/data/SampleDB.bak
CMD ["/opt/mssql/bin/sqlservr"]
```

3. Run the following to build your container
```
docker build . -t mssql-with-backup-example
```

4. Start the container by running the following command after replacing `SA_PASSWORD` with your password
```
docker run -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=YourStrong!Passw0rd' `
      -p 1500:1433 --name sql2 `
      -d mssql-with-backup-example
```

5. Edit the `-P` with the value used for `SA_PASSWORD` used in the previous command and view the contents of the backup file built in the image:

```
docker exec -it sql2 /bin/bash
```
```
/opt/mssql-tools/bin/sqlcmd -S localhost \
   -U SA -P 'YourStrong!Passw0rd' \
   -Q 'RESTORE FILELISTONLY FROM DISK = "/var/opt/mssql/data/SampleDB.bak"' \
   -W \
   | tr -s ' ' | cut -d ' ' -f 1-2
```

the output of this command should be similar to 
![RestoredDBList.PNG](/Media/RestoredDBList.png)

6. Edit the `-P` with the value of `SA_PASSWORD` used to start the container and restore the database in the container:
```
/opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'YourStrong!Passw0rd' -Q 'RESTORE DATABASE ProductCatalog FROM DISK = "/var/opt/mssql/data/SampleDB.bak" WITH MOVE "ProductCatalog" TO "/var/opt/mssql/data/ProductCatalog.mdf", MOVE "ProductCatalog_log" TO "/var/opt/mssql/data/ProductCatalog.ldf"'
```
the output of this command should be similar to 
![RestoredDB.PNG](/Media/RestoredDB.png)

If you connect to the instance, you should see that the database was restored.
 
![Container-RestoredDB.PNG](/Media/Container-RestoredDB.png)

7. Clean up the container
```
docker stop sql2
docker container rm sql2
```


> **Key Takeaway**
>
> A **Dockerfile** defines what goes on in the environment inside your container. Access to resources like networking interfaces and disk drives is virtualized inside this environment, which is isolated from the rest of your system, so you need to map ports to the outside world, and be specific about what files you want to “copy in” to that environment. However, after doing that, you can expect that the build of your app defined in this Dockerfile behaves exactly the same wherever it runs.
>
> -- https://docs.docker.com/get-started/part2/#your-new-development-environment