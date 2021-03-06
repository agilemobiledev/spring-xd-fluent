Spring XD Java DSL
==================

This project represent some experiments with fluent API programming of XD.

## Build it

After cloning, simply run `mvn install` to create a `spring-xd-java-dsl 0.0.1.BUILD-SNAPSHOT` and install it in your local repo.

## Using it

Create a simple project with pom, add a dependency on `spring-xd-fluent`. Here is a simple pom:

    <?xml version="1.0" encoding="UTF-8"?>
    <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
        <modelVersion>4.0.0</modelVersion>

      <groupId>foo</groupId>
      <artifactId>bar</artifactId>
      <version>0.0.1</version>

      <properties>
        <java.version>1.8</java.version>
        <maven.compiler.source>${java.version}</maven.compiler.source>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.target>${java.version}</maven.compiler.target>
      </properties>

      <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-xd-java-dsl</artifactId>
            <version>0.0.1.BUILD-SNAPSHOT</version>
        </dependency>
      </dependencies>
    </project>

The only requirement for running any code you develop is that you have XD running. The easiest way to do that is run XD single node. Spring-XD-Fluent uses the XD REST API to communicate with that running XD.

Here is the first basic program:

    import org.springframework.xd.fluent.XD;
    import org.springframework.xd.fluent.domain.standard.Sinks;
    import org.springframework.xd.fluent.domain.standard.Sources;

    public class Demo {
      public static void main(String[] args) {
        XD.source(Sources.time()).sink(Sinks.log()).deploy("teststream",true);
      }
    }

The boolean parameter to `deploy()` specifies that you want this new stream
to replace any currently defined with that name.

This will deploy a stream that does `'time | log'`, producing this kind of output in XD:

  `13:02:38,506  INFO task-scheduler-6 sink.code-stream-1 - 2015-04-14 13:02:38`

Something more sophisticated:

    DeployableStream s = 
      XD.source(Sources.time("HH:mm:ss")).
      process(Processors.transform("payload.substring(6)")).
      sink(Sinks.log());
    s.deploy("teststream",true);

Here we are passing a format option to the `time` source and an expression to the `transform` processor.

## Now for the cool stuff

Let's rewrite the previous example, using a Java lambda construct:

    DeployableStream s = 
      XD.source(Sources.time("HH:mm:ss")).
      process(payload -> payload.substring(6)).
      sink(Sinks.log());
    s.deploy("mystream",true);

Alternatively let's use some RxJava:

    DeployableStream s = XD.source(Sources.time("HH:MM:ss")).
      process(time -> "{\"time\":\"" + time + "\"}"). // make it json
      process(Processors.jsonToTuple()).
      processRx(inputStream -> 
        inputStream.map(tuple -> {
          return tuple.getValue("time").toString();
        }).
        buffer(5).
        map(data -> tuple().of("time", data.get(0)))).
        sink(Sinks.log());
    s.deploy("mystream",true);

Surely that is the most efficient way to print the time out every 5 seconds...

## State of the project

It is early (early!) days. Only a small number of sources/sinks/processors are included in the proof of concept. The examples above do work but if you vary things too much you may get into problems with incompatible message formats when you deploy it.

## How does it work

The interesting part is when using lambdas or Rx flows. Basically the code will generate XD modules on the fly that embed the code expressed as a lambda or Rx flow, it then registers these modules dynamically as part of the `deploy()` operation before it deploys the stream.

## FAQ

