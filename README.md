## GraalVM & Liberty Demo

Agenda:

* Walk through the GraalVM Overview slides
* Discuss Migrating from Java 8 to Java 11
* Demos
	* Spring Pet Clinic (JAR vs Native Image)
	* Comparing Liberty on Hotspot and GraalVM
	* Running Native Image in Docker/Podman
	* (Optional) Demonstrate on Kubernetes


### Comparing Spring Pet Clinic Performance Using Hotspot and Native Image

If you haven't already, package the application:

```
$ cd graalvm-native-image-spring-petclinic
$ mvn package
```

Make certain you're using Java 8 Hotspot VM:

```
$ sdkman use java 8.0.271-oracle
$ java -version
java version "1.8.0_271"
Java(TM) SE Runtime Environment (build 1.8.0_271-b09)
Java HotSpot(TM) 64-Bit Server VM (build 25.271-b09, mixed mode)
```

Run the 'fat jar' app and note the startup time and memory utilization.

```
$ java -jar target/petclinic-jpa-0.0.1-SNAPSHOT.jar
```

Linux:

```
$ pmap <PID> | tail -n 1 | gawk '/total/ { a=strtonum($2); b=int(a/1024); printf "Total memory used is: "; printf b" MB\n"};'
```

Mac:

Use Activity Monitor or:

```
$ vmmap <PID> | grep -e '\[<PID>]' -e 'Physical footprint'
```

If you haven't already, compile the native image version of the application (will take several minutes):

```
$ ./compile.sh
```
Run the native image version and note the startup time and memory utilization.

```
$ target/petclinic-jpa
```
Check memory usage:

Linux:

```
$ pmap <PID> | tail -n 1 | gawk '/total/ { a=strtonum($2); b=int(a/1024); printf "Total memory used is: "; printf b" MB\n"};'
```

Mac:

Use Activity Monitor or:

```
$ vmmap <PID> | grep -e '\[<PID>]' -e 'Physical footprint'
```

### Liberty Performance Comparison (Hotspot vs GraalVM)


```
$ cd graalvm-liberty-jaxrs-demo
```

**This particluar example requires JDK 8, so make certain you're using both Java 8 Hotspot and Java 8 GraalVM versions for testing.**


```
$ sdkman use java 8.0.271-oracle
$ java -version
java version "1.8.0_271"
Java(TM) SE Runtime Environment (build 1.8.0_271-b09)
Java HotSpot(TM) 64-Bit Server VM (build 25.271-b09, mixed mode)
```
```
$ mvn package
```

Start the application:

```
$ mvn liberty:dev
```

Next, execute the run-test.sh script in the project directory to begin the benchmark tests. Run the script several times until the VM stablizes.

Example results using Java 8 (8u271) Hotspot VM:

```
$ ./run-test.sh
Testing with 20,000,000 (20 million) iterations
Time taken to complete in milliseconds: 3978 ; and result is: 3000000000
real    0m3.984s
user    0m0.004s
sys     0m0.000s
```

Stop the application (CTRL-C) and change JDK 8 Hotspot VM to the GraalVM version:

```
$ sdkman use java 20.3.0-r8-grlee
$ java -version
java version "1.8.0_271"
Java(TM) SE Runtime Environment (build 1.8.0_271-b09)
Java HotSpot(TM) 64-Bit Server VM GraalVM EE 20.3.0 (build 25.271-b09-jvmci-20.3-b06, mixed mode))
```

Start the application:

```
$ mvn liberty:dev
```
Run the run-test.sh script again. Run the script several times until the VM stablizes.

Example results using GraalVM 8 (20.3.0):

```
$ ./run-test.sh
Testing with 20,000,000 (20 million) iterations
Time taken to complete in milliseconds: 140 ; and result is: 3000000000
real	0m0.151s
user	0m0.003s
sys		0m0.005s
```

As you can see, we realized significant improvement in performance by simply replacing the standard Hotspot VM with GraalVM.

### Running with Docker/Podman

```
$ cd graalvm-native-image-spring-petclinic
```
Create a new file called `Dockerfile-jar` and add the following:

```
FROM oracle/graalvm-ce:20.3.0-java11ENV MYSQL_HOST=mysql \    DO_NOT_INITIALIZE=never \    ORACLE_HOST=oracle \    VERSION=v0.11COPY target/petclinic-jpa-0.0.1-SNAPSHOT.jar .EXPOSE 8080CMD ["java","-jar","petclinic-jpa-0.0.1-SNAPSHOT.jar"]
```
Build the image:

```
$ docker build -f Dockerfile-jar -t petclinic-mysql-jar:1.0 .
```

```
$ docker imagesREPOSITORY                       TAG          IMAGE ID       CREATED          SIZEpetclinic-mysql-jar              1.0          0e991e1ed702   20 minutes ago   1.44GB
```

Make certain MySQL is up and running:

```
$ sudo systemctl status mysqld
```

Run the jar docker image:

```
$ docker run --net="host" -i --rm -p 8080:8080 petclinic-mysql-jar:1.0
```
```
Started PetClinicApplication in 5.561 seconds (JVM running for 6.261)
```

In a seperate terminal window, get the container ID and then display the container stats:

```
$ docker ps
CONTAINER ID  IMAGE                      COMMAND  CREATED        STATUS            PORTS   NAMES
e5c825a82215  petclinic-mysql-jar:1.0             6 seconds ago  Up 6 seconds ago          dmiring_lichterman

$ docker stats 86f6525e4ccd
CONTAINER ID   NAME                  CPU %     MEM USAGE / LIMIT     MEM %     NET I/O   BLOCK I/O   PIDS86f6525e4ccd   admiring_lichterman   0.13%     943.8MiB / 25.02GiB   3.68%     0B / 0B   0B / 0B     56
```

Next, edit `Dockerfile-multistage` and change the GraalVM version on the first line to 20.3.0 (Java 8 or 11):

```
FROM oracle/graalvm-ce:20.3.0-java8 as graalvm
```

Create the docker image (takes several minutes so build it in advance):

```
$ ./build-on-docker.sh
```

```
$ docker images
REPOSITORY                                       TAG            IMAGE ID      CREATED       SIZE
localhost/marthenl/petclinic-mysql-native-image  0.12           3499c3bb8dc1  10 hrs ago   245 MB

```
Run the native image docker image:

```
$ docker run --net="host" -i --rm -p 8080:8080 localhost/marthenl/petclinic-mysql-native-image:0.12
```

```
Started PetClinicApplication in 0.571 seconds (JVM running for 0.592)
```

In a seperate terminal window, get the container ID and then display the container stats:

```
$ docker ps
CONTAINER ID  IMAGE                                                 COMMAND  CREATED        STATUS            PORTS   NAMES
e5c825a82215  localhost/marthenl/petclinic-mysql-native-image:0.12           6 seconds ago  Up 6 seconds ago          friendly_neumann

$ docker stats e5c825a82215
CONTAINER ID   NAME               CPU %     MEM USAGE / LIMIT     MEM %     NET I/O   BLOCK I/O   PIDSe5c825a82215   friendly_neumann   0.00%     119.1MiB / 25.02GiB   0.46%     0B / 0B   0B / 0B     31
```

As you can see, the native image docker container is much smaller and starts much faster than the 'fat-jar' version.