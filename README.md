# cockroachDB
https://www.cockroachlabs.com/docs/stable/start-a-local-cluster-in-docker-linux.html

# start docker sample
## start cluster with docker

```sh
# pull the docker image
sudo docker pull cockroachdb/cockroach:v24.1.3
sudo docker images

# create a bridge network
sudo docker network create -d bridge roachnet
sudo docker network ls

# create docker volumes for each cluster node
sudo docker volume create roach1
sudo docker volume create roach2
sudo docker volume create roach3
sudo docker volume ls

# start the cluster of cockroach dbs
sudo docker run -d \
  --name=roach1 \
  --hostname=roach1 \
  --net=roachnet \
  -p 26257:26257 \
  -p 8080:8080 \
  -v "roach1:/cockroach/cockroach-data" \
  cockroachdb/cockroach:v24.1.3 start \
    --advertise-addr=roach1:26357 \
    --http-addr=roach1:8080 \
    --listen-addr=roach1:26357 \
    --sql-addr=roach1:26257 \
    --insecure \
    --join=roach1:26357,roach2:26357,roach3:26357

sudo docker run -d \
  --name=roach2 \
  --hostname=roach2 \
  --net=roachnet \
  -p 26258:26258 \
  -p 8081:8081 \
  -v "roach2:/cockroach/cockroach-data" \
  cockroachdb/cockroach:v24.1.3 start \
    --advertise-addr=roach2:26357 \
    --http-addr=roach2:8081 \
    --listen-addr=roach2:26357 \
    --sql-addr=roach2:26258 \
    --insecure \
    --join=roach1:26357,roach2:26357,roach3:26357

sudo docker run -d \
  --name=roach3 \
  --hostname=roach3 \
  --net=roachnet \
  -p 26259:26259 \
  -p 8082:8082 \
  -v "roach3:/cockroach/cockroach-data" \
  cockroachdb/cockroach:v24.1.3 start \
    --advertise-addr=roach3:26357 \
    --http-addr=roach3:8082 \
    --listen-addr=roach3:26357 \
    --sql-addr=roach3:26259 \
    --insecure \
    --join=roach1:26357,roach2:26357,roach3:26357
```

### options for docker run
* advertise-addr:inter-node traffic
* http-addr:DB console traffic
* listen-addr
* sql-addr:SQL traffic
* join


## initialize cluster

```sh
# Perform one-time initialization of the cluster
# run cockroach init command
sudo docker exec -it roach1 ./cockroach --host=roach1:26357 init --insecure

# show starting logs of nodes
sudo docker exec -it roach1 grep 'node starting' /cockroach/cockroach-data/logs/cockroach.log -A ll
```

## connecting cluster

```sh
# start the SQL shell
sudo docker exec -it roach1 ./cockroach sql --host=roach2:26258 --insecure

> CREATE DATABASE bank;
> CREATE TABLE bank.accounts (id INT PRIMARY KEY, balance DECIMAL);
> INSERT INTO bank.accounts VALUES (1, 100.50);
> SELECT * FROM bank.accounts;
> \q

# open new shell roach2
sudo docker exec -it roach2 ./cockroach --host=roach2:26258 sql --insecure

> SELECT * FROM bank.accounts;
> \q
```

## run a sample workload
cockroach workload command does not support connecting or security flags like other cockroach commands.  
instead, you must use a connection string at the end of the command.  

```sh
# load the initial dataset on roach1:26257
sudo docker exec -it roach1 ./cockroach workload init movr 'postgresql://root@roach1:26257?sslmode=disable'

# run the workload for five minutes
sudo docker exec -it roach1 ./cockroach workload run movr --duration=5m 'postgresql://root@roach1:26257?sslmode=disable'

```

## access the DB console
go to http://localhost:8080  
roach2 and roach3 are reachable on ports 8081 and 8082  

## stop the cluster

```sh
# use docker stop and rm command
sudo docker stop -t 300 roach1 roach2 roach3

sudo docker rm roach1 roach2 roach3

sudo docker volume rm roach1 roach2 roach3

sudo docker network rm roachnet

```

# start single-node cluster
## available docker environments
* COCKROACH_DATABASE
* COCKROACH_USER
* COCKROACH_PASSWORD

## docker start
```sh
# create docker volume
sudo docker volume create roach-single

# start cluster
sudo docker run -d \
  --env COCKROACH_DATABASE={DATABASE_NAME} \
  --env COCKROACH_USER={USER_NAME} \
  --env COCKROACH_PASSWORD={PASSWORD} \
  --name=roach-single \
  --hostname=roach-single \
  -p 26257:26257 \
  -p 8080:8080 \
  -v "roach-single:/cockroach/cockroach-data" \
  cockroachdb/cockroach:v24.1.3 start-single-node \
  --http-addr=roach-single:8080

# to mounts /docker-entrypoint-initdb.d to ~/init-scripts to run the sql files

# check starting logs
sudo docker exec -it roach-single grep 'node starting' /cockroach/cockroach-data/logs/cockroach.log -A 11

# connect to cluster
sudo docker logs --follow roach-single
sudo docker exec -it roach-single ./cockroach sql --url="postgresql://root@127.0.0.1:26257/defaultdb?sslcert=certs%2Fclient.root.crt&sslkey=certs%2Fclient.root.key&sslmode=verify-full&sslrootcert=certs%2Fca.crt"
```

## access the db console
http://localhost:8080  

## stop cluster

```sh
sudo docker stop -t 300 roach-single
sudo docker rm roach-single
sudo docker volume rm roach-single
```
