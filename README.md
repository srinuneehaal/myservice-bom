# myservice-bom

`myservice-parent` is a standalone Maven `pom` artifact for centralized version management across Spring Boot microservices that live in other repositories.

It is intentionally focused on shared platform dependencies:

- Spring Boot dependency alignment through `spring-boot-dependencies`
- explicit management for `org.json:json`
- explicit management for `commons-io:commons-io`

## Managed Versions

The root [`pom.xml`](pom.xml) keeps versions in one place as properties.

| Property | Version |
| --- | --- |
| `spring-boot.version` | `3.4.13` |
| `org-json.version` | `20251224` |
| `commons-io.version` | `2.21.0` |
| `java.version` | `21` |

Use `java.version` as the platform baseline for all consuming microservices.

## Consumption Options

Use it as a parent POM when you want shared dependency versions and common plugin defaults:

```xml
<parent>
    <groupId>com.myservice.platform</groupId>
    <artifactId>myservice-parent</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</parent>
```

Use it as an imported BOM when the microservice must keep its own parent:

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

After either setup, declare common dependencies without versions:

```xml
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
```

## Repository Access

When your microservice project lives in another repository, add the GitHub Packages Maven repository:

```xml
<repositories>
    <repository>
        <id>github</id>
        <url>https://maven.pkg.github.com/srinuneehaal/myservice-bom</url>
    </repository>
</repositories>
```

That consumer repository must authenticate to GitHub Packages. In GitHub Actions, use `actions/setup-java` with `server-id: github`. For local development, configure matching credentials in `~/.m2/settings.xml`.

## Repository Layout

- [`pom.xml`](pom.xml): publishable BOM and lightweight parent POM
- [`.github/workflows/bom-ci.yml`](.github/workflows/bom-ci.yml): CI validation for the BOM
- [`CHANGELOG.md`](CHANGELOG.md): release history

## Release and Maintenance

- Version the BOM independently using semantic versioning.
- Update dependency versions in root properties only.
- Record each release in [`CHANGELOG.md`](CHANGELOG.md).
- GitHub Packages publish target: `https://maven.pkg.github.com/srinuneehaal/myservice-bom`
- Publishing runs from GitHub Actions on every push to `main` and on manual workflow dispatches. Pull requests validate only and do not publish.
- Validate changes before release:

```bash
mvn -B -ntp clean install
```

This repo does not include any microservice implementation. Create and consume this BOM from a separate microservice repository.
