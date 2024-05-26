# Dependency Management Management

## Learning Goals

* Managing dependencies with Gradle and Maven
* Locking dependencies with Gradle and Maven
* Verifying digests and signatures for Java dependencies

## Overview

In this exercise, you will see some of the main features in Gradle and Maven to manage dependencies in a secure way, including locks, digests, and signatures verification.

## Details

### Managing dependencies with Gradle

#### Resolving dependencies

Gradle manages two types of dependencies: the build dependencies used in your Java applications and the plugin dependencies. The build dependencies can be configured to be fetched from the `repositories` section of the `build.gradle` file. There is no implicit value applied.

```groovy
repositories {
    mavenCentral()
}
```

Plugin dependencies can be configured to be fetched from the `pluginRepositories` section of the `settings.gradle` file. By default, the [Gradle Plugin Portal](https://plugins.gradle.org) is used.

Explore all the build dependencies (direct and transitive) used in the `dependencies-gradle-demo` Spring Boot application.

```shell
cd dependencies-gradle-demo
./gradlew dependencies
```

Notice how some libraries are included as transitive dependencies from other libraries, but with different version numbers. Gradle will resolve the conflict by using the highest version number (e.g. `io.micrometer:micrometer-observation:1.12.6 -> 1.13.0`).

Open the `build.gradle` file inside the `dependencies-gradle-demo` folder and declare an explicit version for the Micrometer Observation dependency.

```groovy
dependencies {
	...
	implementation 'io.micrometer:micrometer-observation:1.12.6'
}
```

Gradle will now resolve the conflict by using the explicit version you have just declared in the `build.gradle` file. Verify that by checking the dependency list again (`io.micrometer:micrometer-observation:1.12.6`).

```shell
./gradlew dependencies
```

The project is configured to enforce a reproducible dependency resolution strategy. Use of dynamic or changing versions will fail the build. Try replacing the version number for the Micrometer Observation dependency to a dynamic one.

```groovy
dependencies {
	...
	implementation 'io.micrometer:micrometer-observation:1.+'
}
```

If you run a build (`./gradlew build`), it will fail because of the following configuration (`build.gradle`).

```groovy
configurations.all {
    resolutionStrategy {
        failOnNonReproducibleResolution()
    }
}
```

### Managing dependencies with Maven

Maven manages two types of dependencies: the build dependencies used in your Java applications and the plugin dependencies. The build dependencies can be configured to be fetched from the `<repositories>` section of the `pom.xml` file. Maven Central is always included as a repository, unless explicitly disabled.

Plugin dependencies can be configured to be fetched from the `<pluginRepositories>` section of the `pom.xml` file. By default, Maven Central is always included as a repository, unless explicitly disabled.

In the `dependencies-maven-demo` folder, you can see that the `pom.xml` file doesn't specify any repository.

You can view all the settings applied by Maven implicitly by running the following command that generates the effective POM. In particular, notice the content of the `<repositories>` and `<pluginRepositories>`.

```shell
cd ../dependencies-maven-demo
./mvnw help:effective-pom
```

Explored all the build dependencies (direct and transitive) used in the `dependencies-maven-demo` Spring Boot application.

```shell
./mvnw dependency:tree -Dverbose
```

Notice how some libraries are included as transitive dependencies from other libraries, but with different version numbers. Maven will resolve the conflict by using a nearest-node strategy (e.g. `io.micrometer:micrometer-observation:jar:1.13.0:compile (version managed from 1.12.6)`).

Open the `pom.xml` file inside the `dependencies-maven-demo` folder and declare an explicit version for the Micrometer Observation dependency.

```maven
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-observation</artifactId>
    <version>1.12.6</version>
</dependency>
```

Maven will now resolve the conflict by using the explicit version you have just declared in the `build.gradle` file because it's the nearest node in the dependency graph. Verify that by checking the dependency list again (`io.micrometer:micrometer-observation:jar:1.12.6:compile`).

```shell
./mvnw dependency:tree -Dverbose
```
