

* Building image example
 ```bash
  docker build -t myip -f Dockerfile_ENTRYPOINT .
 ```


## Recap and cheat sheet 
List Docker CLI commands
```bash
docker
docker container --help
```

## Display Docker version and info
```bash
docker --version
docker version
docker info
```
## Execute Docker image
```bash
docker run hello-world
```
## List Docker images
```docker image ls```
## List Docker containers (running, all, all in quiet mode)

```bash
docker container ls
docker container ls --all
docker container ls -aq
```


Here is a list of the basic Docker commands from this page, and some related ones if you’d like to explore a bit before moving on.


```bash
docker build -t friendlyhello . # Create image using this directory's Dockerfile
docker run -p 4000:80 friendlyhello # Run "friendlyname" mapping port 4000 to 80
docker run -d -p 4000:80 friendlyhello # Same thing, but in detached mode
docker container ls # List all running containers
docker container ls -a # List all containers, even those not running
docker container stop <hash> # Gracefully stop the specified container
docker container kill <hash> # Force shutdown of the specified container
docker container rm <hash> # Remove specified container from this machine
docker container rm $(docker container ls -a -q) # Remove all containers
docker image ls -a # List all images on this machine
docker image rm <image id> # Remove specified image from this machine
docker image rm $(docker image ls -a -q) # Remove all images from this machine
docker login # Log in this CLI session using your Docker credentials
docker tag <image> username/repository:tag # Tag <image> for upload to registry
docker push username/repository:tag # Upload tagged image to registry
docker run username/repository:tag # Run image from a registry
```


## ENTRYPOINT
* Entrypoint sets the command and parameters that will be executed first when a container is run.
Any command line arguments passed to docker run <image> will be appended to the entrypoint command,
and will override all elements specified using CMD. 
For example, docker run <image> bash will add the command argument bash to the end of the entrypoint.
```ENTRYPOINT ["echo"]```

```bash
# Working with ENTRYPOINT AND CMD - CMD is a parameter to ENTRYPOINT
# ENTRYPOINT ["wget", "-O-", "-q"]
# CMD http://ifconfig.me/ip

# Overriding ENTRYPOINT from commandline `--entrypoint`
```

### RUN vs CMD vs ENTRYPOINT

In a nutshell
* RUN executes command(s) in a new layer and creates a new image. E.g., it is often used for installing software packages.
* CMD sets default command and/or parameters, which can be overwritten from command line when docker container runs.
* ENTRYPOINT configures a container that will run as an executable.

## Getting Host IP

Mac:
`docker.for.mac.localhost` no longer works, at least Docker for Mac Edge Version 18.03.0-ce-rc4-mac57 (23360). Use `docker.for.mac.host.internal` instead! [link](https://forums.docker.com/t/understanding-the-docker-for-mac-localhost-behavior/41921/2)
however should be `host.docker.internal` for Windows and Mac, but due to https://github.com/docker/for-mac/issues/2965 doesn`t work on Mac


## Examples
### Running MySQL DB
Based on [Customize your MySQL database in Docker](https://medium.com/@lvthillo/customize-your-mysql-database-in-docker-723ffd59d8fb)

1. Create SQL scripts for table creation and data insertion
Write a `CreateTable.sql`. This file contains the SQL statement to create a table called employees. 
```sql
CREATE TABLE employees (
first_name varchar(25),
last_name  varchar(25),
department varchar(15),
email  varchar(50)
);
```
Write a `InsertData.sql`. This file contains our statement to insert data in the table ‘employees’.
```sql
INSERT INTO employees (first_name, last_name, department, email) 
VALUES ('Lorenz', 'Vanthillo', 'IT', 'lvthillo@mail.com');
```

2. Create **Dockerfile**
```
# Derived from official mysql image (our base image)
FROM mysql
# Add a database
ENV MYSQL_DATABASE company
# Add the content of the sql-scripts/ directory to your image
# All scripts in docker-entrypoint-initdb.d/ are automatically
# executed during container startup
COPY ./sql-scripts/ /docker-entrypoint-initdb.d/
```
**NOTE** - All scripts in `docker-entrypoint-initdb.d/` are automatically executed during container startup


3. Create Docker **image**:

```bash
docker build -t my-mysql .
```

4. Start your MySQL container from the image:
```bash
docker run -d -p 3306:3306 --name my-mysql \
-e MYSQL_ROOT_PASSWORD=supersecret my-mysql
```
`-d` - run container in detached mode, in a background, as a service
`-p` - expose port outside the docker network (`--publish , -p`		Publish a container’s port(s) to the host)
`--name` - name container so that it would be easier to manage it
`-e`- set environment variables
`my-mysql` - name of the **image**

5. Verify - `exec` inside the container:
```bash
docker exec -it my-mysql bash


root@c86ff80d7524:/# mysql -uroot -p
Enter password: (supersecret)
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| company            |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)
mysql> use company;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A
Database changed
mysql> show tables;
+-------------------+
| Tables_in_company |
+-------------------+
| employees         |
+-------------------+
1 row in set (0.00 sec)
mysql> show columns from employees;
+------------+-------------+------+-----+---------+-------+
| Field      | Type        | Null | Key | Default | Extra |
+------------+-------------+------+-----+---------+-------+
| first_name | varchar(25) | YES  |     | NULL    |       |
| last_name  | varchar(25) | YES  |     | NULL    |       |
| department | varchar(15) | YES  |     | NULL    |       |
| email      | varchar(50) | YES  |     | NULL    |       |
+------------+-------------+------+-----+---------+-------+
4 rows in set (0.00 sec)
mysql> select * from employees;
+------------+-----------+------------+-------------------+
| first_name | last_name | department | email             |
+------------+-----------+------------+-------------------+
| Lorenz     | Vanthillo | IT         | lvthillo@mail.com |
+------------+-----------+------------+-------------------+
1 row in set (0.01 sec)
```

But in some cases this is not the best solution:

* If you insert a lot of data your image size will grow significantly
* Need to build a new image when you want to update the data
That’s why there is another way to customize your Docker MySQL.

#### Use bind mounts to customize your MySQL database in Docker

```bash
docker run -d -p 3306:3306 --name my-mysql \
-v ~/my-mysql/sql-scripts:/docker-entrypoint-initdb.d/ \
-e MYSQL_ROOT_PASSWORD=supersecret \
-e MYSQL_DATABASE=company \
mysql
```
Using this method is already more flexible but it will be a little bit harder to distribute among the developers. They all need to store the scripts in a certain directory on their local machine and they need to point to that directory when they execute the docker run command.
