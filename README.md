# docker-geoserver

A simple docker container that runs Geoserver influenced by this docker
recipe: https://github.com/kartoza/docker-geoserver by Tim Sutton and this recipe https://github.com/thinkWhere/GeoServer-Docker.

This container is configured to build with:
* Tomcat8.5
* Openjdk 8 
* GeoServer 2.8.x / 2.9.x / 2.10.x / 2.11.x / 2.12.x / 2.13x
* GeoServer Plugins: Any plugins downloaded to /resources/plugins
* Truetype fonts: Any .ttf fonts copied to the /resources/fonts folder will be included in the container
* Native JAI / JAI ImageIO for better raster data processing 
* Clustering of Geoserver by hazelcast and jdbconfig
* Postgresql/Postgis - 10.0-2.4


**Note:** We recommend using ``apt-cacher-ng`` to speed up package fetching -
you should configure the host for it in the provided 71-apt-cacher-ng file.

## Getting the image

The image cannot be downloaded from dockerhub.

To build with apt-cacher-ng you need to
clone this repo locally first and modify the contents of 71-apt-cacher-ng to
match your cacher host. Then build using a local url instead of directly from
github.

```shell
git clone git://github.com/simonelanucara/geoserver-docker/
```
Now edit ``71-apt-cacher-ng``

And build:
```shell
docker build -t simonelanucara/geoserver-docker .
```
## Options

### Geoserver Plugins

To build the GeoServer image with plugins (Control Flow, Monitor, Inspire, etc), 
download the plugin zip files from the GeoServer download page and put them in 
`resources/plugins` before building.  You should make sure these match the version of
GeoServer you are installing.
GeoServer version is controlled by the variable in Dockerfile, or download the WAR bundle
for the version you want to `resources/geoserver.zip` before building.
The 2.13 version contain jdbcconfig and hazelcast plugin.

### Custom Fonts

To include any .ttf fonts with symbols in your container, copy them into the `resources/fonts` folder
before building.

### Tomcat Extras

Tomcat is bundled with extras including docs, examples, etc.  If you don't need these, set
the `TOMCAT_EXTRAS` build-arg to `false` when building the image.  (This is the default in 
build.sh.)

```shell
docker build --build-arg TOMCAT_EXTRAS=false -t simonelanucara/geoserver-docker .
```

### GeoWebCache

GeoServer is installed by default with the integrated GeoWebCache functionality.  If you are using
the stand-alone GeoWebCache, or another caching engine such as MapProxy, you can remove the built-in GWC
by setting the `DISABLE_GWC` build-arg to `true` when building the image.

```shell
docker build --build-arg DISABLE_GWC=true -t simonelanucara/geoserver-docker .
```

**Note:** this removes all *gwc* jar files from the installation. If you are building with plugins that have 
dependencies on the gwc classes, using this option could prevent geoserver from initializing.  
(examples include:  INSPIRE plugin v2.9.2+; control-flow plugin v2.9.2+)

### Native JAI / JAI ImageIO

Native JAI and JAI ImageIO are included in the final image by default providing better
performance for raster data processing. Unfortunately they native JAI is not under active
development anymore. In the event that you face issues with raster data processing,
they can remove them from the final image by setting the `JAI_IMAGEIO` build-arg to `false`
when building the image.

```shell
docker build --build-arg JAI_IMAGEIO=false -t simonelanucara/geoserver-docker .
```

### GDAL Image Formats support

You can optionally include native GDAL libraries and GDAL extension in the image to enable
support for GDAL image formats.

To include native GDAL libraries in the image, set the `GDAL_NATIVE` build-arg to `true`
when building the image.

```shell
docker build --build-arg GDAL_NATIVE=true -t simonelanucara/geoserver-docker .
```

To include the GDAL extension in the final image download the extension and place the zip
file in the `resources/plugins` folder before building the image. If you use the build.sh
script to build the image simply uncomment the relevant part of the script.

```shell
#if [ ! -f resources/plugins/geoserver-gdal-plugin.zip ]
#then
#    wget -c http://netix.dl.sourceforge.net/project/geoserver/GeoServer/2.8.3/extensions/geoserver-2.8.3-gdal-plugin.zip -O resources/plugins/geoserver-gdal-plugin.zip
#fi
```

## Run

### External geoserver_data directory
You probably want to run Geoserver with an external geoserver_data directory mapped as a docker volume.
This allows the configuration to be persisted or shared with other instances. To create a running container 
with an external volume do:

```shell
mkdir -p ~/geoserver_data
docker run \
	--name=geoserver \
	-p 8080:8080 \
	-d \
	-v $HOME/geoserver_data:/opt/geoserver/data_dir \
	-t simonelanucara/geoserver-docker
```

### Running multiple instances on the same machine
For installations with multiple containers on the same host, map port 8080 to a different port for each
instance.  It is recommended that the instance name contains the mapped port no for ease of reference.
Each instance should also have its own log file, specified by passing in the `GEOSERVER_LOG_LOCATION'
variable as illustrated in the example below.

```shell
mkdir -p ~/geoserver_data
docker run \
	--name=geoserver \
	-p 8085:8080 \
	-d \
	-v $HOME/geoserver_data:/opt/geoserver/data_dir \
	-e "GEOSERVER_LOG_LOCATION=/opt/geoserver/data_dir/logs/geoserver_8085.log" \
	-t simonelanucara/geoserver-docker
```

### Setting Tomcat properties

To set Tomcat properties such as maximum heap memory size, create a `setenv.sh` 
file such as:

```shell
JAVA_OPTS="$JAVA_OPTS -Xmx1024M"
JAVA_OPTS="$JAVA_OPTS -Djava.awt.headless=true -XX:+UseConcMarkSweepGC -XX:+CMSClassUnloadingEnabled"
```

Then pass the `setenv.sh` file as a volume at `/usr/local/tomcat/bin/setenv.sh` when running:

```shell
docker run -d \
    -v $HOME/tomcat/setenv.sh:/usr/local/tomcat/bin/setenv.sh \
    simonelanucara/geoserver-docker
```

This repository contains a ``run.sh`` script for your convenience.

### Using docker-compose

Docker-compose allows you to deploy a load-balanced cluster of geoserver containers with a single command.  A sample docker-compose.yml configuration file is included in this repository, along with a sample nginx configuration file.

To deploy using docker-compose:

1. copy nginx folder from this repository to your machine and edit the nginx.conf file.
2. copy tomcat_settings folder from this repository to your machine.
3. copy docker-compose.yml to your machine.  Edit the volume entries to reflect the correct location of your geoserver_data, nginx and tomcat_settings folders on your machine.
4. type `docker build -t simonelanucara/geoserver-docker:2.13 .` to build the image locally
5. type `docker-compose up --build -d`  to start up a cluster of 2x geoserver containers + 1 posgresql/postgis container +1 nginx load balancer.
6. configure jdbcconfig and cluster properties
7. access geoserver services at  http://YOURIP/geoserver/wms?
8. add postgis store

**Note:** The default geoserver user is 'admin' and the password is 'geoserver'.
It is recommended that these are changed for production systems.
The postgis geoserver store is avaliable on host 172.17.0.1, port 25433 dabase gis.
The postgresql connection is avaliable on your external IP on port 24533.
