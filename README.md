## RBC GraalVM Demo

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
$ pmap <PID> | tail -n 1 | awk '/[0-9]K/{print $2}'
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
$ pmap <PID> | tail -n 1 | awk '/[0-9]K/{print $2}'
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

As you can see, we realized significant imrpovement in performance by simply replacing the standard Hotspot VM with GraalVM.

### Running in Docker/Podman


