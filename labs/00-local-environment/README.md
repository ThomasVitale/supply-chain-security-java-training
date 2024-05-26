# Development Environment

## Learning Goals

* Working in a containerized development environment

## Overview

After following the instructions in the `README.md` file available in the root folder of this repository, it's time to verify that everything is working correctly.

## Details

### Working with Java Applications

Let's first try out working with Java applications.

Run the `dev-environment-demo` Spring Boot application:

```shell
cd dev-environment-demo
./gradlew bootRun
```

Open another Terminal window and verify the application is running correctly:

```shell
http :8080
```

The result should be: "Welcome to your development environment!".

Notice how you didn't have to install any Java distribution nor the [httpie](https://httpie.io) utility used in the previous command.

Before moving on, remember to stop the process running the Spring Boot application.

### Working with Containers

Next, let's try working with containers.

Containerize the `dev-environment-demo` Spring Boot application.

```shell
./gradlew bootBuildImage
```

Then, run the newly created container image.

```shell
docker run --rm -p 8080:8080 dev-environment-demo
```

Open another Terminal window and verify the container is running correctly:

```shell
http :8080
```

Before moving on, remember to stop the process running the Spring Boot container.
