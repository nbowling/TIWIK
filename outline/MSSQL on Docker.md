# Nicks Notes
## Installing and configuring MSSQL 2025 on Docker


### If necessary create a volume to persist data

If need to create a docker volume to persist the data in: `$ docker volume create <volume name>`

If exists or is no longer required: `$ docker volume rm <volume name>`

List containers with: `$ docker ps -a`

### Create database container
This way maps the database container to the previously created volume on Docker

`$ docker run -e 'ACCEPT_EULA=Y' -e 'MSSQL_SA_PASSWORD=<Your Secure Password>' -e "MSSQL_PID=Evaluation" -p 1433:1433 --name sql1 --hostname sql1 -v <volume name>:/var/opt/mssql -d mcr.microsoft.com/mssql/server:2025-latest`

Can stop and remove the container
```
$ docker stop <container name or id>
$ docker rm <container name or id>
```

You can recreate it with the same volume and your data will persist.

### Check where data folder is
Open the container as root so you can access bash: `$ docker exec -it <container name> bash`

In the container shell: `mssql@sql1:/$ ls /var/opt/mssql`
The result is >

data&emsp;log&emsp;secrets

### Copying Northwind into a docker container
If necessary the next step is to copy a database into the server. This can be confusing because:
1. The permissions on the file will need adjusting so that MSSQL can access it
2. The diecctory mapping is confusing

The easiest way is to set up a disposable container that you can control and map it to the same volume as the MSSQL Server container
`$ docker run --name temp-copy -v sqldata:/data -d busybox sleep 3600`

Note data is the mount point this is slightly confusing this means that to access the MSSQL data directory where you want the database to be it is at; /data/data

From the host: `$ docker cp ./Northwind.mdf temp-copy:/data/data/Northwind.mdf`

Now we need to adjust permissions and ownership of the database file, in this case Northwind.mdf. Exit to the host then run an Ubuntu container as root: `$ docker run --rm -it -v sqldata:/data ubuntu bash`

cd to where mssql is hiding data file (try /data/data)
```
$ chmod 644 Northwind.mdf
$ chown 10001:10001 Northwind.mdf
```

From the host you can now access sqlcmd (ToDo how to install this)

`$ sqlcmd -S localhost -U SA -P '<YourSecurePassword>'`

```SQL
1> CREATE DATABASE Northwind
2> ON (FILENAME = N'/var/opt/mssql/data/Northwind.mdf')
3> FOR ATTACH;
4> GO
```
All being well the database will attach. If you have any problems it is highly likely to be permission related.

## Creating a backup of a Docker volume

If you need to backup the containers attached volume then that is possible in several ways. The following are the most straight forward. There is also a short script to automate the process and have it rin as a cron job.

First approach is to just use docker to do the job

`$ docker run --rm -v <volume name>:/source:ro -v /media/nickb/library/MSSQL_Backup:/backup ubuntu tar czf /backup/<volume name>-backup.tar.gz -C /source .`

**Don't forget the trailing period**


The second and better way is by using native SQL Server tools.

`$ docker exec <sql container name> /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 0klah0Ma9#? -C -Q "BACKUP DATABASE Northwind TO DISK = N'/var/opt/mssql/data/Northwind.bak' WITH NOFORMAT, NOINIT, NAME = 'Full Backup', SKIP, NOREWIND, NOUNLOAD, STATS = 10"`

The copy the backup file to host: `$ docker cp <sql container name>:/var/opt/mssql/data/YourDatabase.bak ~/<path to backup>`

Check out the documentation for what all the FLAGS do.

Can automate with the following script backup-sql-volumes.sh
```bash
#!/bin/bash

	BACKUP_DIR=/media/nickb/library/MSSQL_Backup
	TIMESTAMP=$(date +%Y%m%d-%H%M%S)

	# Create backup directory
	mkdir -p $BACKUP_DIR

	# Stop SQL Server for consistent backup
	docker stop sql1

	# Backup each volume
	docker run --rm -v <volume name>:/source:ro -v $BACKUP_DIR:/backup ubuntu tar czf /backup/<volume name>-$TIMESTAMP.tar.gz -C /source .

	# Restart SQL Server
	docker start sql1

	# Keep only last 7 days of backups
	find $BACKUP_DIR -name "<volume name>-*.tar.gz" -mtime +7 -delete

	echo "Backup completed: $TIMESTAMP"
```
Then to automate add script to crontab

`chmod +x ~/backup-sql-volumes.sh`

Edit the crontab: `crontab -e` by adding the following line:

0 2 * * * /home/<yourusername>/backup-sql-volumes.sh >> /home/<yourusername>/backup.log 2>&1

## For SQL Server 2022 Only
This works:

`docker run -e "ACCEPT_EULA=Y" -e "SA_PASSWORD=0klah0Ma9#?" -e "MSSQL_PID=Evaluation" -p 1433:1433 -v --name sqlserver -d mcr.microsoft.com/mssql/server:2022-latest`

However work will be lost on shutdown

Data can be persisted by mapping to a docker volume or you can map data directories to the host machine

`docker run -e "ACCEPT_EULA=Y" -e 'MSSQL_SA_PASSWORD=0klah0Ma9#?' -p 1433:1433 --name <container name> --hostname <container name> -v <directory path on home>:/var/opt/mssql/data -v <directory path on home>:/var/opt/mssql/log -v <directory path on home>:/var/opt/mssql secrets-d mcr.microsoft.com/mssql/server:2022-latest`

Best automated with a docker compose file (docker-compose.yaml)
```yaml
services:
  mssql-server:
    image: mcr.microsoft.com/mssql/server:2022-latest # Use the desired SQL Server image and tag
    container_name: sql2
    ports:
      - "1433:1433" # Maps host port 1433 to container port 1433
    environment:
      # Required EULA acceptance
      - ACCEPT_EULA=Y
      # Sets the strong SA password (replace "YourStrongPassword1" with your own secure password)
      - SA_PASSWORD=0klah0Ma9#?
      # Set SQL Server version
      - MSSQL_PID=Evaluation
      # Others
      - MSSQL_DATA_DIR=/var/opt/mssql/data
      - MSSQL_LOG_DIR=/var/opt/mssql/log
      - MSSQL_BACKUP_DIR=/var/opt/mssql/backup

    volumes:
      # Optional: Persist data outside the container
      - ~/mssql_data:/var/opt/mssql
      # - ~/mssql-data:/var/opt/mssql/data
    restart: unless-stopped # Automatically restart the container unless it is explicitly stopped

# Define the volume for persistent storage
volumes:
  mssql_data:
  ```
  
Run from yaml file directory with: `$ docker compose up -d`

The -d flag runs the database in detached mode which is probably what you want.

To run sqlcmd from host: `$ sqlcmd -S localhost -U SA -P <YourStrong!Passw0rd>`
(may need -C to bypass self certs)

Test with:
```SQL
SELECT name FROM sys.databases;
GO
```
## Restoring a backup
```bash
????
```

## MSSQL Extension on Visual Studio Code

Problem
Table designer won't start when server is running on Docker


