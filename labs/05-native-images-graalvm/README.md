# Native Executables with GraalVM

## Learning Goals

* Understanding GraalVM native compilation
* Compiling Spring Boot applications into native executables
* Configuring the GraalVM compiler to support dynamic behavior
* Generating SBOMs for native executables

## Overview

In this exercise, you will work with Spring Boot applications compiled to native executables using GraalVM. You'll see how to configure a project to support GraalVM, how the compilation works, and how to generate SBOMs for the native artifacts.

## Details

Before moving on to the example, ensure you have installed Oracle GraalVM 21+ and set it as your default Java version and distribution within your project. Otherwise, you can do so with [SDKMAN](https://sdkman.io).

```shell
java -version

java version "22" 2024-03-19
Java(TM) SE Runtime Environment Oracle GraalVM 22+36.1 (build 22+36-jvmci-b02)
Java HotSpot(TM) 64-Bit Server VM Oracle GraalVM 22+36.1 (build 22+36-jvmci-b02, mixed mode, sharing)
```

### Native Image Support in Spring Boot

Spring Boot provides out-of-the-box support for GraalVM native images. It provides all the necessary configuration for the GraalVM compiler to work correctly with dynamic behavior like proxies, reflection, and resources whenever they are used through the Spring APIs. Let's test that out.

Go to the `native-image-gradle-demo` project and add a `DemoController` class and a `Book` record at the bottom of the `Application.java` file.

```java
@RestController
class DemoController {

	@GetMapping("/books")
	public List<Book> getBooks() {
		return List.of(
			new Book("The Hobbit"),
			new Book("The Lord of the Rings")
		);
	}
	
}

record Book(String title) {}
```

By default, Spring Boot relies on the Jackson library to convert Java objects to JSON and the other way around. Jackson relies on reflective behavior for the conversion to work, a dynamic behavior that the GraalVM compiler needs to be informed about for the native compilation to work successfully. Since you are using the `Book` record as a return type of one of the Spring APIs (`RestController`), Spring will automatically include the necessary configuration to enable reflection on the `Book` class, ensuring Jackson to work correctly.

Go ahead and package the Spring Boot application as a native executable (in Maven you would use the command `./mvnw -Pnative native:compile`).

```shell
cd native-image-gradle-demo
./gradlew nativeCompile
```

The process can take up to a few minutes, depending on how much memory and CPU you have available in your environment.

Next, you can launch the applicaton as a native executable, running directly on your operating system without any intermediate Java runtime.

```shell
./build/native/nativeCompile/native-image-gradle-demo
```

You can now verify if the application is working as expected. Open another Terminal window and run the following command.

```shell
http :8080/books
```

You should get back a list of books in JSON format.

Before moving on, remember to stop the process running the native executable.

### Configuring GraalVM with Spring Boot Native Hints

In some cases, you might use dynamic Java features outside the Spring APIs. In that scenario, you'll need to configure the GraalVM compiler yourself to make them work when the application runs as a native executable.

You could provide the configuration via the GraalVM specific JSON format, but Spring Boot provide a more convenient way through annotations and programmatic APIs.

Let's continue the work on the previous application. Delete the `DemoController` class you added previously to the `Application.java` file. This time, you will define the same endpoint relying on a `RouterFunction`. It's a compact and performant way to define an HTTP API. 

```java
@SpringBootApplication
public class Application {

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}

	@Bean
	RouterFunction<ServerResponse> routerFunction() {
		return RouterFunctions.route()
			.GET("/", request -> ServerResponse.ok().body("Welcome to your development environment!"))
			.GET("/books", request -> ServerResponse.ok().body(List.of(
				new Book("The Hobbit"),
				new Book("The Lord of the Rings")
			)))
			.build();
	}

}

record Book(String title) {}
```

The downside is that Spring doesn't pick up on the types that need custom GraalVM configuration. Let's see that in action.

Compile the application again to a native executable.

```shell
./gradlew nativeCompile
```

Next, run the application.

```shell
./build/native/nativeCompile/native-image-gradle-demo
```

You can now test that the Jackson serialization will fail. Open another Terminal window and run the following command.

```shell
http :8080/books
```

Before moving on, remember to stop the process running the native executable.

How to solve this problem?

Spring Boot provides two main ways to define "hints" for the GraalVM compiler. One way is by implementing the `RuntimeHintsRegistrar` interface, especially useful for libraries and if you need to provide a long configuration to the GraalVM compiler. The other way is via annotations, more suitable for applications where you don't need to provide a long configuration to the GraalVM compiler. You'll use the latter approach.

Open the `Application.java` file again and use the `@RegisterReflectionForBinding` annotation to list what classes should be registered for reflection with the GraalVM compiler.

```java
@SpringBootApplication
@RegisterReflectionForBinding(Book.class)
public class Application {
	...
}
```

Compile the application again to a native executable.

```shell
./gradlew nativeCompile
```

Next, run the application.

```shell
./build/native/nativeCompile/native-image-gradle-demo
```

You can now test that the Jackson serialization will succeed. Open another Terminal window and run the following command.

```shell
http :8080/books
```

Before moving on, remember to stop the process running the native executable.

### SBOM Management with GraalVM

When working with native executables, we don't have the option to scan the final artifact for SBOM generation directly. Instead, we need to move the generation step to an earlier stage of the software development lifecycle.

The Oracle GraalVM distribution offers an experimental SBOM generation feature when compiling the application. This feature uses Syft under the hood and only supports the CycloneDX format.

The quality of the SBOM generated as part of the GraalVM compilation is usually comparable to generating an SBOM with Syft out of a JAR artifact. For example, it's suitable for vulnerability management scenarios but not for license compliance use cases because the necessary information is likely missing from the final result.

First, configure the GraalVM Gradle plugin for the `native-image-gradle-demo` project, inside the `build.gradle` file.

```groovy
graalvmNative {
    binaries {
        configureEach {
            buildArgs.add("--enable-sbom=cyclonedx,export")
        }
    }
}
```

Next, compile the application to a native executable.

```shell
./gradlew nativeCompile
```

As part of the native compilation process, an SBOM is generated using Syft and following the CycloneDX format. A copy of the SBOM is included in the artifact itself, and because you added the export flag in the configuration, it's also exported to `build/native/nativeCompile/native-image-gradle-demo.sbom.json`. You can inspect the exported SBOM or even extract the one embedded in the native executable using the [Native Image Inspection Tool](https://www.graalvm.org/latest/reference-manual/native-image/debugging-and-diagnostics/InspectTool).

To achieve the same result in a Maven project (`pom.xml`), you can configure the build as follows.

```xml
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
    <configuration>
        <buildArgs combine.children="append">
            <buildArg>--enable-sbom=cyclonedx,export</buildArg>
        </buildArgs>
    </configuration>
</plugin>
```
