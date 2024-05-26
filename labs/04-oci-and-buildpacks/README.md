# OCI Images and Cloud Native Buildpacks

## Learning Goals

* Understanding OCI Images and the role of Cloud Native Buildpacks
* Using and configuring the Buildpacks integration in Spring Boot
* SBOM management with Buildpacks and Spring Boot

## Overview

In this exercise, you will work with Cloud Native Buildpacks to containerize your Spring Boot applications. The result will be secure and performant OCI images which also include SBOMs.

## Details

### Containerizing Spring Boot Applications

Many options exist for packaging Spring Boot applications as OCI images. Spring Boot provides a native integration with [Cloud Native Buildpacks](https://buildpacks.io), a tool to transform application source code into production-ready images without the need for a Dockerfile. This strategy provides a great developer experience and clear separation of concerns between application teams and platform teams.

Cloud Native Buildpacks is a specification and multiple implementations exist. By default, Spring Boot uses the [Paketo Buildpacks](https://paketo.io) implementation.

Let's try that out!

For starters, containerize the `buildpacks-gradle-demo` application with the `bootBuildImage` task (in Maven, you would run the `./mvnw spring-boot:build-image` command).

```shell
cd buildpacks-gradle-demo
./gradlew bootBuildImage
```

Inspect the logs from the process. You can see how the Buildpacks process is composed of multiple steps, each contributing a certain feature to the final build, such as the Java runtime, the application executable JAR, and the Spring Boot-specific configuration. Let's run it!

```shell
docker run --rm -p 8080:8080 buildpacks-gradle-demo
```

One of the component included in the final image is a convenient utility to configure the memory for the Java Virtual Machine based on a smart algorithm battle-tested over many years of production deployments with Cloud Foundry and Kubernetes. You can check the logs at the start of the process to find out the computed memory configuration.

```shell
Calculating JVM memory based on 2107780K available memory
Calculated JVM Memory Configuration: -XX:MaxDirectMemorySize=10M -Xmx1512853K -XX:MaxMetaspaceSize=82926K -XX:ReservedCodeCacheSize=240M -Xss1M (Total Memory: 2107780K, Thread Count: 250, Loaded Class Count: 12227, Headroom: 0%)
```

For more information, check the [documentation](https://paketo.io/docs/reference/java-reference/#memory-calculator).

Before moving on, make sure you stop the process running the container.

The Buildpacks projects comes with its own CLI: `pack`. You don't need it for containerizing Spring Boot applications thanks to the native Buildpacks integration, but you might find it useful to inspect the generated images.

First, ensure you have pack installed (`pack version`) or follow the [instructions](https://buildpacks.io/docs/for-platform-operators/how-to/integrate-ci/pack/) to install it.

Then, run the following command to inspect the image you have just built.

```shell
pack inspect buildpacks-gradle-demo
```

You'll see all the different Buildpacks components that contributed to the containerization process.

```shell
Stack: io.buildpacks.stacks.jammy.tiny

Base Image:
  Reference: 9110449875b05c50e7ec31d76cf6e366a919201eced17b8aa1e6d5997e4f6530
  Top Layer: sha256:848aafec928d16405590003187b91e76233406a81b9dd2fee0dd90d0c70fd2c2

Run Images:
  index.docker.io/paketobuildpacks/run-jammy-tiny:latest

Rebasable: true

Buildpacks:
  ID                                         VERSION        HOMEPAGE
  paketo-buildpacks/ca-certificates          3.8.0          https://github.com/paketo-buildpacks/ca-certificates
  paketo-buildpacks/bellsoft-liberica        10.8.0         https://github.com/paketo-buildpacks/bellsoft-liberica
  paketo-buildpacks/syft                     1.47.0         https://github.com/paketo-buildpacks/syft
  paketo-buildpacks/executable-jar           6.10.0         https://github.com/paketo-buildpacks/executable-jar
  paketo-buildpacks/dist-zip                 5.8.0          https://github.com/paketo-buildpacks/dist-zip
  paketo-buildpacks/spring-boot              5.30.0         https://github.com/paketo-buildpacks/spring-boot

Processes:
  TYPE                  SHELL        COMMAND        ARGS                                                      WORK DIR
  web (default)                      java           org.springframework.boot.loader.launch.JarLauncher        /workspace
  executable-jar                     java           org.springframework.boot.loader.launch.JarLauncher        /workspace
  task                               java           org.springframework.boot.loader.launch.JarLauncher        /workspace
```

### Generating SBOMs for container images

Generating an SBOM for a Java application packaged as a container image requires a tool that supports finding and extracting information from JARs and recognizes the OS packages and the Java runtime.

In the previous section, you used the `bootBuildImage` Gradle task from Spring Boot to containerize a Java application. Under the hood, Spring Boot uses Cloud Native Buildpacks, a specification to transform application source code into a container image without needing a Dockerfile. 

More specifically, Spring Boot uses the Paketo Buildpacks implementation, which generates an SBOM for each layer of the built image using Syft. When you need to create an SBOM for a container image built using Buildpacks, you can extract the autogenerated SBOMs from the image rather than generating them as a separate task.

You can use the pack CLI to extract the SBOMs from each layer of the image.

```shell
pack sbom download buildpacks-gradle-demo -o sbom-layers
```

You'll find the SBOMs in the `sbom-layers` directory.

```shell
sbom-layers
└── layers
    └── sbom
        └── launch
            ├── buildpacksio_lifecycle
            │   └── launcher
            │       ├── sbom.cdx.json
            │       ├── sbom.spdx.json
            │       └── sbom.syft.json
            ├── paketo-buildpacks_bellsoft-liberica
            │   ├── helper
            │   │   └── sbom.syft.json
            │   └── jre
            │       └── sbom.syft.json
            ├── paketo-buildpacks_ca-certificates
            │   └── helper
            │       └── sbom.syft.json
            ├── paketo-buildpacks_executable-jar
            │   ├── sbom.cdx.json
            │   └── sbom.syft.json
            ├── paketo-buildpacks_spring-boot
            │   ├── helper
            │   │   └── sbom.syft.json
            │   └── spring-cloud-bindings
            │       └── sbom.syft.json
            └── sbom.legacy.json
```

That is a convenient feature of Buildpacks that should lead to higher-quality SBOMs than those generated after the image build. However, extracting the SBOMs from the image is cumbersome and a bit challenging to integrate with other tools in the supply chain security ecosystem. Furthermore, its behavior could be more consistent. Different buildpacks produce SBOMs in different formats. The only constant in the example above is the Syft format.

### SBOM Management with Spring Boot 3.3

In the previous lab, you worked with the new feature in Spring Boot 3.3 to serve the auto-generated application SBOM via a dedicated Actuator endpoint. On top of that, you can include additional SBOMs to the endpoint. In this exercise, you'll extend the endpoint to show all the SBOMs generated as part of the Buildpacks process for each layer of the image.

Go to the `sbom-gradle-demo` project and add the following configuration to the `application.yml` file to include the additional SBOMs part of the container image. Currently, an explicit list must be included. In the future, Spring Boot might provide some automated detection of such SBOMs when using Buildpacks.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: sbom
  endpoint:
    sbom:
      additional:
        buildpacks-lifecycle:
          location: "optional:file:/layers/sbom/launch/buildpacksio_lifecycle/launcher/sbom.cdx.json"
        buildpacks-liberica-helper:
          location: "optional:file:/layers/sbom/launch/paketo-buildpacks_bellsoft-liberica/helper/sbom.syft.json"
        buildpacks-liberica-jre:
          location: "optional:file:/layers/sbom/launch/paketo-buildpacks_bellsoft-liberica/jre/sbom.syft.json"
        buildpacks-ca-certificates:
          location: "optional:file:/layers/sbom/launch/paketo-buildpacks_ca-certificates/helper/sbom.syft.json"
        buildpacks-executable-jar:
          location: "optional:file:/layers/sbom/launch/paketo-buildpacks_executable-jar/sbom.cdx.json"
        buildpacks-spring-boot-helper:
          location: "optional:file:/layers/sbom/launch/paketo-buildpacks_spring-boot/helper/sbom.syft.json"
        buildpacks-spring-boot-spring-cloud-bindings:
          location: "optional:file:/layers/sbom/launch/paketo-buildpacks_spring-boot/spring-cloud-bindings/sbom.syft.json"
```

Next, build again the Spring Boot application as a container image.

```shell
./gradlew bootBuildImage
```

Then, run the application container.

```shell
docker run --rm -p 8080:8080 buildpacks-gradle-demo
```

Open a browser window an visit `http://localhost:8080/actuator/sbom`. You'll find a list of all the available SBOMs. In this case, you have both the one built via CycloneDX when building the JAR file and the ones generated as part of the Buildpacks process for each layer of the image.

You can visualize each SBOM file using its identifier in the URL. For example, you can get insights into the Java runtime installed inside the image: `http://localhost:8080/actuator/sbom/buildpacks-liberica-jre`.

Finally, you can use Trivy to scan the image for vulnerabilities. Trivy will be able to detect the SBOM files included in the image and use them to gather inputs on which components to scan.

```shell
trivy image buildpacks-gradle-demo
```

Before moving on, make sure you stop the process running the container.
