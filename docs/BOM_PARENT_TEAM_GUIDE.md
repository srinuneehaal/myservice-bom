# Maven BOM vs Parent POM Guide

This page is written for team sharing in Confluence. You can paste it as-is or adapt section titles to your Confluence space.

## Purpose

This repository publishes `com.myservice.platform:myservice-parent:1.0.0-SNAPSHOT`, which is a Maven `pom` artifact used to centralize platform standards for our microservices.

It currently serves two roles:

- Parent POM for shared build configuration and properties
- BOM-like dependency source for shared dependency versions

## Quick Summary

- A regular `pom.xml` is the Maven project descriptor
- A parent POM is inherited through `<parent>`
- A BOM is imported through `<dependencyManagement>` with `type=pom` and `scope=import`
- `dependencyManagement` manages dependency versions, but does not add dependencies
- `pluginManagement` manages plugin versions/configuration, but does not run plugins
- `<plugins>` applies plugins to the build

## Diagram 1: Big Picture

```text
                        +----------------------------------+
                        |  myservice-parent (this repo)    |
                        |----------------------------------|
                        | properties                       |
                        | dependencyManagement             |
                        | pluginManagement                 |
                        | build/plugins                    |
                        | distributionManagement           |
                        +----------------------------------+
                                  /               \
                                 /                 \
                                /                   \
                               v                     v
                +---------------------------+   +---------------------------+
                | Service uses <parent>     |   | Service imports as BOM    |
                +---------------------------+   +---------------------------+
                | gets properties           |   | gets managed dep versions |
                | gets dep mgmt             |   | only                      |
                | gets plugin mgmt          |   |                           |
                | gets build plugins        |   | does NOT get properties   |
                | gets build rules          |   | does NOT get plugins      |
                +---------------------------+   +---------------------------+
```

## Why We Use This Repo

We want one platform artifact to standardize:

- Java version
- encoding
- managed dependency versions
- shared Maven plugin versions
- build rules such as Java and Maven minimum versions

Without this shared POM, every microservice would repeat version numbers and plugin configuration, which leads to drift.

## Key Concepts

### 1. POM

Every Maven project has a `pom.xml`. It describes:

- coordinates such as `groupId`, `artifactId`, `version`
- dependencies
- plugins
- packaging
- publishing rules

### 2. Parent POM

A parent POM is consumed like this:

```xml
<parent>
    <groupId>com.myservice.platform</groupId>
    <artifactId>myservice-parent</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</parent>
```

When a project uses a parent POM, it inherits:

- properties
- dependency management
- plugin management
- build plugins
- distribution settings, if relevant

This is the right choice when the service should follow the platform build standard.

#### Diagram 2: Parent Inheritance

```text
+-----------------------------+
| myservice-parent            |
|-----------------------------|
| properties                  |
| dependencyManagement        |
| pluginManagement            |
| plugins                     |
| enforcer rules              |
+-------------+---------------+
              |
              | inherited via <parent>
              v
+-----------------------------+
| orders-service              |
|-----------------------------|
| can use ${java.version}     |
| can omit dep versions       |
| gets compiler plugin config |
| gets enforcer checks        |
+-----------------------------+
```

### 3. BOM

A BOM is consumed like this:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.myservice.platform</groupId>
            <artifactId>myservice-parent</artifactId>
            <version>1.0.0-SNAPSHOT</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

When a project imports a BOM, Maven imports only managed dependency versions.

It does not inherit:

- properties
- plugin definitions
- plugin executions
- distribution management

This is the right choice when a service must keep a different parent POM but still wants our dependency version alignment.

#### Diagram 3: BOM Import

```text
+-----------------------------+
| myservice-parent            |
|-----------------------------|
| dependencyManagement        |
| properties                  |
| pluginManagement            |
| plugins                     |
+-------------+---------------+
              |
              | imported through
              | <dependencyManagement>
              v
+-----------------------------+
| orders-service              |
|-----------------------------|
| gets managed dep versions   |
| does NOT get properties     |
| does NOT get plugin config  |
| does NOT get build rules    |
+-----------------------------+
```

## What Our Current POM Provides

The current root `pom.xml` centralizes:

- Java version: `21`
- Spring Boot dependency alignment through `spring-boot-dependencies`
- explicit versions for `org.json:json` and `commons-io:commons-io`
- plugin versions for compiler, surefire, resources, and enforcer
- enforcement rules for Maven `3.8.6+` and Java `21+`

Important: because the artifact is named `myservice-parent`, teams may assume everything is inherited in all usage modes. That is not true. If it is imported as a BOM, only dependency management is imported.

## Parent vs BOM Behavior

| Capability | Use as Parent | Import as BOM |
| --- | --- | --- |
| Inherit `${java.version}` | Yes | No |
| Inherit `${maven.compiler.release}` | Yes | No |
| Use managed dependency versions without specifying versions | Yes | Yes |
| Inherit `maven-compiler-plugin` config | Yes | No |
| Inherit `maven-enforcer-plugin` rules | Yes | No |
| Inherit GitHub package publishing config | Yes | No |

#### Diagram 4: What Flows to the Consumer

```text
Legend:
  [YES] = available in consumer project
  [NO ] = not available in consumer project

+-----------------------------------+---------+---------+
| Capability                        | Parent  | BOM     |
+-----------------------------------+---------+---------+
| Dependency versions               | [YES]   | [YES]   |
| Properties                        | [YES]   | [NO ]   |
| Plugin default versions/config    | [YES]   | [NO ]   |
| Plugin executions                 | [YES]   | [NO ]   |
| Java/Maven enforcement rules      | [YES]   | [NO ]   |
+-----------------------------------+---------+---------+
```

## Demo 1: Microservice Using This Artifact as Parent

Use this when the microservice should adopt our platform defaults.

```xml
<project>
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.myservice.platform</groupId>
        <artifactId>myservice-parent</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>

    <artifactId>orders-service</artifactId>
    <packaging>jar</packaging>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.json</groupId>
            <artifactId>json</artifactId>
        </dependency>

        <dependency>
            <groupId>commons-io</groupId>
            <artifactId>commons-io</artifactId>
        </dependency>
    </dependencies>
</project>
```

### Result

- no need to specify versions for managed dependencies
- Java version and compiler release are inherited
- Maven compiler, resources, surefire, and enforcer plugins are inherited

## Demo 2: Microservice Keeping Its Own Parent and Importing Our BOM

Use this when the microservice already has another parent and only needs dependency version alignment.

```xml
<project>
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.4.13</version>
        <relativePath/>
    </parent>

    <groupId>com.myservice.orders</groupId>
    <artifactId>orders-service</artifactId>
    <version>1.0.0-SNAPSHOT</version>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.myservice.platform</groupId>
                <artifactId>myservice-parent</artifactId>
                <version>1.0.0-SNAPSHOT</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.json</groupId>
            <artifactId>json</artifactId>
        </dependency>
    </dependencies>
</project>
```

### Result

- dependency versions from our platform artifact are available
- plugin configuration from our POM is not inherited
- platform properties such as `${maven.compiler.release}` are not inherited

## Demo 3: Common Mistake

### Incorrect expectation

Developers sometimes import the BOM and expect platform properties to be available:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <release>${maven.compiler.release}</release>
            </configuration>
        </plugin>
    </plugins>
</build>
```

If the project only imported our artifact as a BOM, `${maven.compiler.release}` is not inherited from our POM.

### Correct understanding

- imported BOM: only managed dependencies come in
- parent POM: properties and build config come in

#### Diagram 5: Why the Common Mistake Happens

```text
Developer imports BOM
        |
        v
Expects ${maven.compiler.release}
        |
        v
Maven looks in current project / inherited parent
        |
        v
Property not found from BOM import
        |
        v
Build config fails or behaves unexpectedly
```

## Recommended Team Standard

Use this repo as a parent POM when:

- the service is a standard internal microservice
- the service should follow our Java and Maven build conventions
- the service should inherit plugin defaults and enforcement rules

Use this repo as a BOM import when:

- the service already has a required parent POM
- the service only needs dependency version alignment

## Suggested Team Rule

For most internal services, prefer:

```xml
<parent>
    <groupId>com.myservice.platform</groupId>
    <artifactId>myservice-parent</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</parent>
```

This gives the team a single place to manage:

- dependency versions
- Java version
- plugin versions
- build enforcement

## Diagram 6: Team Decision Flow

```text
Start
  |
  v
Does the service need to keep another parent POM?
  |\
  | Yes
  |  \
  |   v
  |  Import myservice-parent as BOM
  |  - use for dependency version alignment only
  |  - define service-specific properties/plugins locally
  |
  | No
  v
Use myservice-parent as <parent>
  - inherit platform properties
  - inherit plugin defaults
  - inherit build rules
```

## How to Add or Update a Managed Dependency

Update the root `pom.xml` in this repository:

1. Add or update the version property
2. Add or update the dependency in `<dependencyManagement>`
3. Release the new platform POM version
4. Upgrade consuming services to the new version

## Diagram 7: Where To Make Changes

```text
Need to update a library version?
  -> change <properties> and/or <dependencyManagement> in platform POM

Need to update Java version or compiler settings?
  -> change <properties> and build plugin configuration in platform POM

Need to add a plugin default for all standard services?
  -> change <pluginManagement> or <plugins> in platform POM

Need a service-only dependency?
  -> add it in that service's <dependencies>

Need a service-only plugin behavior?
  -> add it in that service's <build><plugins>
```

Example:

```xml
<properties>
    <mapstruct.version>1.6.3</mapstruct.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct</artifactId>
            <version>${mapstruct.version}</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

Consumers can then declare:

```xml
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
</dependency>
```

## FAQ

### Why not put every dependency directly under `<dependencies>` in the platform POM?

Because that would force all child projects to carry dependencies they may not need. `dependencyManagement` gives version alignment without forcing usage.

### Why not rely only on a BOM?

Because a BOM does not carry shared properties or build plugin configuration.

### Why is this artifact both a parent and a BOM-like source today?

Because the current `pom.xml` contains both parent-style sections and `dependencyManagement`. Maven allows this, but teams need to understand which parts are inherited in each consumption mode.

### Should we split this later?

Possibly. A cleaner long-term model is:

- `myservice-bom` for dependency management only
- `myservice-parent` for shared build defaults and properties, importing `myservice-bom`

That naming is clearer for new developers and reduces confusion.

## Copy-Paste Summary for Confluence Callout

Use `myservice-parent` as a parent when the service should inherit platform build standards. Use it as a BOM import only when the service must keep a different parent and only needs dependency version alignment. BOM import does not expose platform properties or plugin configuration.
