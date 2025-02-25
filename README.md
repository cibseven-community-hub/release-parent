CIB seven Community Release Parent
===================

[![License](https://img.shields.io/github/license/cibseven/cibseven?color=blue&logo=apache)](https://github.com/cibseven-community-hub/release-parent/blob/master/LICENSE)
[![Discussions](https://img.shields.io/badge/discussions-cibseven-green)](https://github.com/orgs/cibseven-community-hub/discussions)
 
## cibseven-community-release-parent

The CIB seven Community Release Parent POM provides common settings for all CIB seven Community projects and components.

## Getting the latest release

You can download source and binaries from our [repository](https://artifacts.cibseven.org/) .

Alternatively, you can pull it from the central Maven repositories:

```xml
<dependency>
  <groupId>org.cibseven.community</groupId>
  <artifactId>release-parent</artifactId>
  <version>1.0.0</version>
</dependency>
```

## Building

Building requires a Java JDK and [Apache Maven](https://maven.apache.org/).
The required Java version is found in the `pom.xml` as the `maven.compiler.source` property.

From a CL, run `mvn` without arguments to invoke the default Maven goal to run all tests and checks.

## Contributing

Please see our [contribution guidelines](https://github.com/cibseven/cibseven/blob/main/CONTRIBUTING.md) for how to raise issues and how to contribute code to our project.

## License

This code is licensed under the [Apache License v2](https://www.apache.org/licenses/LICENSE-2.0).

