# docker-vertica

Vertica docker deployment (single container version).

In the beginning it was inspired by [jbfavre docker-vertica repo](https://github.com/jbfavre/docker-vertica).

It provides:
- Dockerfiles per OS version
- entrypoint script running under database OS user (default dbadmin)
- environment setting scripts
- Scripts for loading of [VMART schema](https://www.vertica.com/docs/10.0.x/HTML/Content/Authoring/GettingStartedGuide/IntroducingVMart/IntroducingVMart.htm).

## Owners of trademarks

Docker and the Docker logo are trademarks or registered trademarks of Docker, Inc. in the United States and/or other countries.
Docker, Inc. and other parties may also have trademark rights in other terms used herein.

Vertica™, the Vertica Analytics Platform™, and FlexStore™ are trademarks of Micro Focus International plc.

## Supported platforms

Vertica:
- 9.x
- 10.x

CentOS
- 7.9
- 8.3

Ubuntu
- 16.04
- 18.04

Debian
- Jessie
- Stretch

Most likely the deployment will work for future versions of Vertica and CentOS / Ubuntu.

It is necessary to prepare separate Dockerfile for other OS families.

## How to build image from Dockerfile

First you have to download Vertica RPM/DEB package, either [Community Edition](https://www.vertica.com/try/)(registration required)
or you can download Enterprise edition, if you are Vertica customer.

Store RPM/DEB into packages folder and use build.sh script:
```
Usage: ./build.sh -v <vertica version> -f <OS family> -o <OS version> [-r <docker repository>]
Options are:
  -v - Vertica version, e.g. 10.0.1-5
  -f - OS family, e.g. CentOS
  -o - OS version of base image, e.g. 7.9.2009, 8.3.2011, ubuntu:18.04
  -r - Image name prefix, a path to repository, e.g. 123456789012.dkr.ecr.eu-central-1.amazonaws.com/databases
  -p - Push built image into repository defined by the image name prefix (-r)
  -h - show help
```

Example:
```
# Build Vertica 10.0.1-5 with CentOS 7.9 and push the image into an AWS repository
./build.sh -v 10.0.1-5 -f CentOS -o 8.3.2011 -r 123456789012.dkr.ecr.eu-central-1.amazonaws.com -p

# Full image path will be:
# 123456789012.dkr.ecr.eu-central-1.amazonaws.com/vertica:10.0.1-5.CentOS_8.3.2011
```

You may need to customize other parameters, here is the full list:
```
docker build -f Dockerfile_CentOS \
             --build-arg vertica_version=10.0.1-5 \
             --build-arg os_version=8.3.2011 \
             --build-arg vertica_db_user=mycustomuser \
             --build-arg vertica_db_group=mycustomgroup \
             --build-arg vertica_db_name=mycustomname \
             # Start and end years populated into VMART date_dimension table
             --build-arg vmart_start_year=2000 \
             --build-arg vmart_end_year=2100 \
             # Remove packages, which you do not need, to significantly reduce size of the image (separated by space)
             --build-arg remove_packages='voltagesecure MachineLearning' \
             -t 123456789012.dkr.ecr.eu-central-1.amazonaws.com/vertica:10.0.1-5.CentOS_8.3.2011 .
```

## How to run Vertica docker container

We recommend to run containers with persisted volumes.

### Docker run

```
docker run -p 5433:5433 \
           -v /path/to/vertica_data:/data \
           --name vertica \
           123456789012.dkr.ecr.eu-central-1.amazonaws.com/vertica:10.0.1-5.CentOS_8.3.2011
```

### Docker-compose

Example YAML file:
```
version: '3.7'

services:
  vertica:
    image: "123456789012.dkr.ecr.eu-central-1.amazonaws.com/vertica:10.0.1-5.CentOS_8.3.2011"
    ports:
      - "5433:5433"
    volumes:
      - vertica-data:/data
      - ./.docker-entrypoint-initdb.d:/docker-entrypoint-initdb.d/

volumes:
  vertica-data:
```

After you store it into docker-compose.yaml file, you can simply run:
```
docker-compose up -d
```

## Integration tests

There is a naive skeleton of integration tests for validation of the current state and for inspiration.
It can be configured in tests/config.yaml (config_full.yaml).
For each required combination of OS / Vertica version the image is build, container is started and tests are executed.
All available customizations of build / run are applied and tested.

Run tests:
```
pip3 install requirements_tests.txt
./tests.py
# Optionally it is possible to test loading of VMART schema:
./tests.py -l
# Use different config file
./tests.py -l -c tests/config_full.yaml
```

## How to configure docker container

It is possible to configure various aspects of Vertica.
To pass a variable to container you must:
```
# Docker run
docker run -p 5433:5433 -d \
  -e TZ='Europe/Prague' \
  123456789012.dkr.ecr.eu-central-1.amazonaws.com/vertica:10.0.1-5.CentOS_8.3.2011

# Docker-compose
  vertica:
    image: "123456789012.dkr.ecr.eu-central-1.amazonaws.com/vertica:10.0.1-5.CentOS_8.3.2011"
    ports:
      - "5433:5433"
    volumes:
      - vertica-data:/data
      - ./.docker-entrypoint-initdb.d:/docker-entrypoint-initdb.d/
    environment:
      TZ: "${VERTICA_CUSTOM_TZ:-Europe/Prague}"
```

List of available configuration parameters:

1. VMART_LOAD_DATA
  Set value to "y" to enable generating data and loading them into VMART schema
  More info about the schema can be found on [official web site](https://www.vertica.com/docs/10.0.x/HTML/Content/Authoring/GettingStartedGuide/IntroducingVMart/IntroducingVMart.htm).
2. APP_DB_USER
  Name of additional database user, who should be created.
  Pseudosuperuser role is granted and enabled to the user.
3. APP_DB_PASSWORD
  Password of APP_DB_USER.
4. TZ: "${VERTICA_CUSTOM_TZ:-Europe/Prague}"
  Customize timezone of database.
  Vertica does not contain all time zones - uncomment a workaround solution in Dockerfile (linking system time zones), if you want to set such time zone.
5. DEBUG_FAILING_STARTUP
  For development purposes. When you set the value to "y", entrypoint script does not end in case of failure, so you can investigate those failures.

## How to log in docker container

```
# Pure docker (we named the container in docker run statement above)
docker exec -it vertica bash -l
# docker compose
docker-compose exec vertica bash -l
```

Environment of dbadmin user is extended to be user-friendly, see /etc/profile.d/vertica_env.sh and $HOME/.vsqlrc for more details.

## How to execute scripts during container startup

Place scripts to be executed from entry point script during startup of container into folder "".docker-entrypoint-initdb.d".

Scripts are executed in lexicographical order.

Supported extensions are:
- sql - SQL commands executed through vsql
- sh - shell scripts

You have to mount .docker-entrypoint-initdb.d local folder into docker container /docker-entrypoint-initdb.d/ folder.
Check examples in docker-compose sections here.
