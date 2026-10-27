# oracle-on-docker

## Create compatible docker/podman machine

An Oracle Database 23.26.3 and APEX 26.2 container needs at least:
- 3 CPU's
- 8 Gig on memory!!
- a shitload of fysical storage 

Bash:

alias docker=podman
alias docker-compose=podman-compose

docker machine stop                           # Stop currently running docker/podman VM machine
podman machine set --cpus 3 --memory 8192     # Upgrade VM machine to 3 CPU's and 8GB memory
docker machine start                          # Start docker/podman VM machine again

podman-compose up -d               # Starts all services in compose.yaml
podman-compose up -d oraclefree    # Starts Oracle Database 26ai service only
podman-compose up -d ords          # Starts Oracle ORDS service only

podman-compose stop           # Stop the pod - 
podman-compose start          # Start the pod




Run `setup_env.sh`


For conn_string.txt file, execute bash command below to create the file, open the file and replace the placeholders with their corresponding value as described below.

`echo CONN_STRING=username/password@hostname:port/service > conn_string.txt`

|Placeholder|Details|
|-----------|-------|
|Username|Enter an account that can connect AS SYSDBA, like SYS for example|
|Password|Enter the corresponding password|
|Hostname|Enter the hostname as specified in the docker-compose.yaml, no forwarding slashes //|
|Port    |Enter the port number used to connect inside the container/docker network, default is 1521|
|Service |Enter the PDB service name to install ORDS/APEX into the PDB|

```yaml
  database:  # This is the name of the service, you can rename this
    image: container-registry.oracle.com/database/free:23.7.0.0  # The docker image for this service
    hostname: ORADB23
    network: my-oracle-network  # Connect the service to a docker network
    # The port mapping of the host port to the container port.
    # For connecting from outside the container
    ports:
      - 1522:1521
    environment:
      # SYS, SYSTEM, PDBADMIN
      - ORACLE_PWD=
      - ORACLE_CHARACTERSET=AL32UTF8
      # - ORACLE_SID=ORCLCDB  # Oracle 23ai Free Edition always uses FREE as CDB name and service name
      # - ORACLE_PDB=ORCLPDB1  # Oracle 23ai Free Edition always uses FREEPDB1 as PDB name and service name
```
### Run podman-compose for the first time

To build all containers, volumes and networks (without (re)building the image (--build)) as specified in the `./docker-compose.yaml` file.

```bash
podman-compose -p apex-dev up -d
```

### Run podman-compose to start containers

To start all containers as specified in the `./docker-compose.yaml` file, but all objects and services must already exists.

```bash
podman-compose -p apex-dev start
```

### Run podman-compose to stop running containers

This command stops all containers as specified in the `./docker-compose.yaml` file, without removing them

```bash
podman-compose -p apex-dev stop
```

### Run podman-compose to bring down complete setup

This command stops and removes all containers, volumes and networks as specified in the `./docker-compose.yaml` file.

```bash
podman-compose -p apex-dev down
```

# Install APEX 24.2 on top

This section describes how to install APEX 24.2 (or any other higher version) on top of the already installed APEX 24.1.

## Create .env.ords

The `developer-ords` image validates if the Oracle APEX version installed on the database is equal to the Oracle APEX version in the image (24.1.0). By creating the file below, we instruct the ORDS container to **ignore** the installed Oracle APEX version, as we may have patched and/or upgraded it to a higher version manually.

Create a file called `.env.ords` in the `ORDS` root directory, with the following content:

```
IGNORE_APEX=TRUE
```

Do that by executing script below

```bash
echo IGNORE_APEX=TRUE > /Users/gbreems/oradata/ORADB23/ords/config/.env.ords
```

As ORDS does not ship with APEX 24.2 yet, we tell ORDS to ignore APEX.

## Upgrade to APEX 24.2

```bash
wget https://download.oracle.com/otn_software/apex/apex-latest.zip
unzip apex-latest.zip
rm apex-latest.zip
rm -rf ./META-INF || true

INSTALL_LOGS=/tmp/install_container.log

cd apex
sql "sys@//localhost:1522/freepdb1 as sysdba" @apexins.sql SYSAUX SYSAUX TEMP /i/
exit
cd ..
```

## Copy images into container

Copy the `./images` folder into the container to update all necessary files.

```bash
#Copies ./images folder on the host system into the container at specified location <container_name:location> (in archive mode) 
cd apex
docker cp --archive ./images apex-dev-ords:/opt/oracle/apex/24.1.0/
cd ..
```

>*You need to re-apply this step if you've brought the docker-compose setup **down** completely.*

## Patch APEX with latest Patch Set Bundle (24.2.x)

> ***Attention:** You need to be a registered Oracle Support customer to be able to download the patch!*

Execute step below to download the PSE Bundle for APEX 24.2 (Patch 37366599).  

Enter your Oracle Support SSO credentials when asked.

```bash
./wget_p37366566.sh

unzip -q -o p37366599_242_GENERIC.zip -d patch_bundle
rm p37366599_242_GENERIC.zip

cd patch_bundle/37366599
sql "sys@//localhost:1522/freepdb1 as sysdba" @catpatch.sql
exit
cd ../..
```

## Copy patched images into container

Copy the `./images` folder into the container to update all necessary files.

```bash
#Copies ./images folder on the host system into the container at specified location <container_name:location> (in archive mode) 
cd patch_bundle/37366599
docker cp --archive ./images apex-dev-ords:/opt/oracle/apex/24.1.0/
cd ..
```

>*You need to re-apply this step if you've brought the docker-compose setup **down** completely.*







## Connect to Docker Container

```bash
docker exec -it <container_id> bash
```

