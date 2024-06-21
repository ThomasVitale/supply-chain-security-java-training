# SBOMs, Vulnerability Scanning, and Compliance

## Learning Goals

* Understanding the use cases for SBOMs
* Generating SBOMs for Java source code, artifacts, and builds
* Configuring the SBOM support in Spring Boot
* Vulnerability scanning and compliance verification with SBOMs

## Overview

In this exercise, you will explore different options to generate SBOMs for Java applications and strategies to manage supply chain security risks, including vulnerability scanning and license checks.

## Details

### Managing SBOMs for Java Artifacts

Let's start by investigating how to generate SBOMs for Spring Boot applications packaged as JAR artifacts.
Creating an SBOM at this stage of the software development lifecycle should guarantee visibility into all the components of the final application artifact. The result heavily depends on the tool used to generate the SBOM. How many components can it find?

Even though a good tool might identify (almost) all the software components included in the application artifact, the depth of information for each might be lacking. Is that bad? It depends. For example, SBOMs generated at this stage could be a good fit for vulnerability scanning purposes. In contrast, they might miss the necessary information for verifying license compliance or tracking the relationships among dependencies. Based on your needs, consider if it makes sense to move (or pair this with) the SBOM generation to an earlier step in the software lifecycle to address use cases not possible at this stage.

Generating an SBOM for a Java application packaged as a JAR artifact requires a tool that supports finding and extracting information from JARs. [Syft](https://github.com/anchore/syft) is a popular open-source tool for generating SBOMs and works with JAR artifacts.

First, ensure you have Syft installed (`syft version`) or follow the [instructions](https://github.com/anchore/syft?tab=readme-ov-file#installation) to install it.

Then, package the `sbom-gradle-demo` application as a JAR artifact.

```shell
cd sbom-gradle-demo
./gradlew bootJar
```

Finally, use Syft to scan the JAR file and generate an SBOM for it.

```shell
syft build/libs/sbom-gradle-demo-1.0.jar \
  --output cyclonedx-json=bom-syft.cdx.json
```

Open the generated SBOM (`bom-syft.cdx.json`) and familiarize with its structure. You can now use it as a basis for supply chain risk management. As a bonus step, you can follow the instructions for the `bonus-dependency-track` lab to get an instance of Dependency Track up and running that you can use to visualize the SBOM content.

### Managing SBOMs for Java Builds

Let's now investigate how to make the generation process part of the build lifecycle. For Java applications, you need a tool to hook into the build process and extract information about every component used to assemble the software.

Creating an SBOM at this stage of the software development lifecycle should guarantee visibility into all the components used to build the application. Since the generation process hooks into the build lifecycle, you can get complete control over which components to include and a more in-depth result than any other strategy.

SBOMs generated at this stage can be a good fit for most use cases, such as vulnerability scanning, license compliance, and procurement. You typically can tailor the operation to include only certain types of dependencies in the final SBOM (useful, for example, if you want to exclude the test dependencies). In contrast, these SBOMs might miss the dependencies needed at runtime, which is usually the case for applications distributed as container images.

The CycloneDX project offers two convenient plugins for [Gradle](https://github.com/CycloneDX/cyclonedx-gradle-plugin) and [Maven](https://github.com/CycloneDX/cyclonedx-maven-plugin) to make the SBOM generation process part of the build lifecycle. That's what we're going to use.

First, add the CycloneDX Gradle plugin to the `sbom-gradle-demo` project, inside the `build.gradle` file.

```groovy
plugins {
	...
  id 'org.cyclonedx.bom' version '1.8.2'
}
```

The plugin adds a `cyclonedxBom` task to generate an SBOM for the application. You can also make it part of the build step so that every time you build the application, an SBOM is generated automatically.

Since we're using Spring Boot 3.3.1, we don't need any extra configuration because Spring Boot has introduced native support for the CycloneDX Gradle and Maven plugins. Whenever you package your application as a JAR file, Spring Boot will take care of generating an SBOM for you and include it in the final JAR artifact.

```shell
./gradlew bootJar
```

You can find the generated SBOM in `build/reports/application.cdx.json`. You can now use it as a basis for supply chain risk management. As a bonus step, you can follow the instructions for the `bonus-dependency-track` lab to get an instance of Dependency Track up and running that you can use to visualize the SBOM content.

Let's now try the same for Maven. First, add the CycloneDX Maven plugin to the `sbom-maven-demo` project, inside the `pom.xml` file.

```xml
<plugin>
    <groupId>org.cyclonedx</groupId>
    <artifactId>cyclonedx-maven-plugin</artifactId>
</plugin>
```

The plugin adds a `makeAggregateBom` target to generate an SBOM for the application. You can also make it part of the build step so that every time you build the application, an SBOM is generated automatically.

Since we're using Spring Boot 3.3.1, we don't need any extra configuration because Spring Boot has introduced native support for the CycloneDX Gradle and Maven plugins. Whenever you package your application as a JAR file, Spring Boot will take care of generating an SBOM for you and include it in the final JAR artifact.

```shell
cd ../sbom-maven-demo
./mvnw package
```

You can find the generated SBOM in `target/classes/META-INF/sbom/application.cdx.json`.

### Vulnerability Scanning with Trivy

Let's now explore some options for performing vulnerability scanning for Spring Boot applications using SBOMs. We'll use the SBOM you have previously generated for the `sbom-gradle-demo` application as an example, but the same will work for Maven.

```shell
cd ../sbom-gradle-demo
```

Ensure you have trivy installed (`trivy version`) or follow the [instructions](https://aquasecurity.github.io/trivy/latest/getting-started/installation/) to install it.

Use Trivy to scan the SBOM for vulnerabilities.

```shell
trivy sbom build/reports/application.cdx.json
```

You can also use it to check if there is any license issue.

```shell
trivy sbom build/reports/application.cdx.json --scanners license
```

### SBOM Management with Spring Boot 3.3

You have already tried out the new feature in Spring Boot 3.3 supporting the CycloneDX plugins for generating SBOMs when packaging a Spring Boot application.

Another feature introduced in Spring Boot 3.3 is a brand new Actuator endpoint (`/actuator/sbom`) that you can enable to serve the application SBOM via an HTTP endpoint. During the build process, Spring Boot will include the SBOM in the final JAR artifact, so that it can be served from the classpath. You can also include additional SBOMs to the endpoint. You'll see that in another lab.

If your project uses Spring Boot Actuator, you can enable the dedicated SBOM endpoint. Go to the `sbom-gradle-demo` project and add the following configuration to the `application.yml` file to enable the endpoint.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: sbom
```

Let's test it out. First, package the application as a JAR artifact.

```shell
./gradlew bootJar
```

Then, run the application on the Java runtime.

```shell
java -jar build/libs/sbom-gradle-demo-1.0.jar
```

Open a browser window an visit `http://localhost:8080/actuator/sbom`. You'll find a list of all the available SBOMs. In this case, you have only one: the one generated by CycloneDX as part of the application packaging operation.

You can visualize the full SBOM file using its identifier in the URL: `http://localhost:8080/actuator/sbom/application`.

Before moving on, remember to stop the process running the application.
