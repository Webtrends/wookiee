wookiee
================
* [wookiee-grpc](#wookiee-grpc)

# wookiee-grpc
## Install
wookiee-grpc is available for Scala 2.12 and 2.13. There are no plans to support scala 2.11 or lower.
```sbt
libraryDependencies += "com.oracle.infy.wookiee" %% "wookiee-grpc" % "0.2.0"
```

## Setup ScalaPB
We use [ScalaPB](https://github.com/scalapb/ScalaPB) to generate source code from a `.proto` file. You can use
other plugins/code generators if you wish. wookiee-grpc will work as long as you have `io.grpc.ServerServiceDefinition`
for the server and something that accept `io.grpc.ManagedChannel` for the client.

Declare your gRPC service using proto3 syntax and save it in `src/main/protobuf/myService.proto`
```proto
syntax = "proto3";

package com.oracle.infy.wookiee;

message HelloRequest {
  string name = 1;
}

message HelloResponse {
  string resp = 1;
}

service MyService {
  rpc greet(HelloRequest) returns (HelloResponse) {}
}

```

Add ScalaPB plugin to `plugin.sbt` file
```sbt
addSbtPlugin("com.thesamet" % "sbt-protoc" % "0.99.34")
libraryDependencies += "com.thesamet.scalapb" %% "compilerplugin" % "0.10.8"

```

Configure the project in `build.sbt` so that ScalaPB can generate code
```sbt
    mdocIn := file("wookiee-docs/docs"),
    mdocOut := file("."),
    mdocVariables := Map(
      "VERSION" -> version.value.split("-").headOption.getOrElse("error-in-build-sbt"),
      "PROTO_FILE" -> protoFile,
      "PROTO_DEF" -> readF(s"wookiee-proto/"++protoFile, _.mkString),
      "PLUGIN_DEF" -> readSection("project/plugins.sbt", "scalaPB"),
      "PROJECT_DEF" -> readSection("build.sbt", "scalaPB"),
      "EXAMPLE" -> readF("wookiee-docs/src/main/scala/com/oracle/infy/wookiee/Example.scala", _.drop(2).mkString)
    )
  )
  .settings(
    libraryDependencies ++= Seq(
      Deps.test.curatorTest,
      Deps.test.slf4jLog4jImpl
    )
  )
  .dependsOn(root, `wookiee-proto`)
  .enablePlugins(MdocPlugin)

lazy val `wookiee-proto` = project
  .in(file("wookiee-proto"))
  .settings(commonSettings)
  .settings(

```

In the sbt shell, type `protocGenerate` to generate scala code based on the `.proto` file. ScalaPB will generate
code and put it under `target/scala-2.13/src_managed/main`.

## Using wookiee-grpc
After the code has been generated by ScalaPB, you can use wookiee-grpc for service discoverability and load balancing.

```scala
import java.lang.Thread.UncaughtExceptionHandler
import java.util.concurrent.{Executors, ForkJoinPool, ThreadFactory}

import cats.effect.IO
import com.oracle.infy.wookiee.grpc.{WookieeGrpcChannel, WookieeGrpcServer}
import com.oracle.infy.wookiee.model.Host
// This is from ScalaPB generated code
import com.oracle.infy.wookiee.myService.MyServiceGrpc.MyService
import com.oracle.infy.wookiee.myService.{HelloRequest, HelloResponse, MyServiceGrpc}
import io.chrisdavenport.log4cats.slf4j.Slf4jLogger
import io.grpc.ServerServiceDefinition
import org.apache.curator.test.TestingServer

import scala.concurrent.duration._
import scala.concurrent.{Await, ExecutionContext, Future}

object Example {

  def main(args: Array[String]): Unit = {
    val bossThreads = 10
    val mainECParallelism = 10

    // wookiee-grpc is written using functional concepts. One key concept is side-effect management/referential transparency
    // We use cats-effect (https://typelevel.org/cats-effect/) internally.
    // If you want to use cats-effect, you can use the methods that return IO[_]. Otherwise, use the methods prefixed with `unsafe`.
    // When using `unsafe` methods, you are expected to handle any exceptions
    val logger = Slf4jLogger.create[IO].unsafeRunSync()

    val uncaughtExceptionHandler = new UncaughtExceptionHandler {
      override def uncaughtException(t: Thread, e: Throwable): Unit = {
        logger.error(e)("Got an uncaught exception on thread " ++ t.getName).unsafeRunSync()
      }
    }

    val tf = new ThreadFactory {
      override def newThread(r: Runnable): Thread = {
        val t = new Thread(r)
        t.setName("blocking-" ++ t.getId.toString)
        t.setUncaughtExceptionHandler(uncaughtExceptionHandler)
        t.setDaemon(true)
        t
      }
    }

    // The blocking execution context must create daemon threads if you want your app to shutdown
    val blockingEC = ExecutionContext.fromExecutorService(Executors.newCachedThreadPool(tf))
    // This is the execution context used to execute your application specific code
    implicit val mainEC: ExecutionContext = ExecutionContext.fromExecutor(
      new ForkJoinPool(
        mainECParallelism,
        ForkJoinPool.defaultForkJoinWorkerThreadFactory,
        uncaughtExceptionHandler,
        true
      )
    )

    val zookeeperDiscoveryPath = "/discovery"

    // This is just to demo, use an actual Zookeeper quorum.
    val zkFake = new TestingServer()
    val connStr = zkFake.getConnectString

    val ssd: ServerServiceDefinition = MyService.bindService(
      new MyService {
        override def greet(request: HelloRequest): Future[HelloResponse] = {
          println("received request")
          Future.successful(HelloResponse("Hello " ++ request.name))
        }
      },
      mainEC
    )

    val serverF: Future[WookieeGrpcServer] = WookieeGrpcServer.startUnsafe(
      zookeeperQuorum = connStr,
      discoveryPath = zookeeperDiscoveryPath,
      zookeeperRetryInterval = 3.seconds,
      zookeeperMaxRetries = 20,
      serverServiceDefinition = ssd,
      port = 9091,
      // This is an optional arg. wookiee-grpc will try to resolve the address automatically.
      // If you are running this locally, its better to explicitly set the hostname
      localhost = Host(0, "localhost", 9091, Map.empty),
      mainExecutionContext = mainEC,
      blockingExecutionContext = blockingEC,
      bossThreads = bossThreads,
      mainExecutionContextThreads = mainECParallelism
    )

    val wookieeGrpcChannel: WookieeGrpcChannel = WookieeGrpcChannel.unsafeOf(
      zookeeperQuorum = connStr,
      serviceDiscoveryPath = zookeeperDiscoveryPath,
      zookeeperRetryInterval = 3.seconds,
      zookeeperMaxRetries = 20,
      grpcChannelThreadLimit = bossThreads,
      mainExecutionContext = mainEC,
      blockingExecutionContext = blockingEC
    )

    val stub: MyServiceGrpc.MyServiceStub = MyServiceGrpc.stub(wookieeGrpcChannel.managedChannel)

    val gRPCResponseF: Future[HelloResponse] = for {
      server <- serverF
      resp <- stub.greet(HelloRequest("world!"))
      _ <- wookieeGrpcChannel.shutdownUnsafe()
      _ <- server.shutdownUnsafe()
    } yield resp

    println(Await.result(gRPCResponseF, Duration.Inf))
    zkFake.close()
    ()
  }
}

Example.main(Array.empty[String])
// received request
// HelloResponse(Hello world!,UnknownFieldSet(Map()))
```


