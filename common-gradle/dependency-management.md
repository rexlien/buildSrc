# Shared dependency management

`dependency-management.gradle` owns the canonical Spring Boot platform and the
approved security exceptions. The plain xln library applies it before
`common.gradle`, without requesting the Boot plugin. Applications apply it from
`boot.gradle` immediately after the Boot plugin imports its canonical BOM.
`common.gradle` and the policy coordinate one synchronous protobuf/gRPC generator
configuration once imports and repositories are ready, in either convention
order. This must precede protobuf's evaluation finalization, while
dependency-management 1.1.7 caches the first managed-version lookup. Do not import
another Boot BOM after this policy or mutate generator locators in `afterEvaluate`.
The application plugin owns each task's Java gRPC plugin entry and its
`@generated=omit` option. The plain library adds that entry itself; shared
conventions add the Kotlin gRPC plugin in both paths. Do not pre-create the
application task's `grpc` entry before Boot's protobuf integration runs.

The shared buildSrc maintainers own these exceptions. Keep application dependency
declarations versionless, and keep ordinary versions outside Boot's platform in
the existing `versions` map. This policy uses explicit BOM imports and managed
dependencies, including in generated Maven dependency management; it does not
use BOM property substitutions, resolution forces, advisory ignores or scan
filters. Both Jackson BOMs intentionally specify the shared annotations artifact
at **2.21**, rather than either family's patch version.

| Dependency | Boot 4.1.2 / observed baseline | Selected fix |
| --- | --- | --- |
| Jackson 2 BOM | 2.21.7 | 2.21.8 |
| Jackson 3 BOM | 3.1.7 | 3.1.8 |
| `com.fasterxml.jackson.core:jackson-annotations` | 2.21 | 2.21, agreed by both fixed BOMs |
| `at.yawk.lz4:lz4-java` | 1.11.0 / 1.11.2 from Spark/Kafka graphs | 1.11.4 |
| `com.github.luben:zstd-jni` | 1.5.7-7 / 1.5.6-10 from Spark/Kafka graphs | 1.5.7-14 |
| `org.xerial.snappy:snappy-java` | 1.1.10.8 / 1.1.10.7 from Spark/Kafka graphs | 1.1.10.10 |
| Logback `logback-classic` / `logback-core` | 1.5.38 | 1.6.6, aligned |
| `net.minidev:json-smart` / `accessors-smart` | 2.6.0 | 2.6.1, aligned |
| ASM `asm`, `asm-analysis`, `asm-commons`, `asm-tree`, `asm-util` | 9.7.1 (`asm` in test graphs) | 9.11.1, aligned when present |
| `org.apache.kafka:kafka-clients` | 4.2.2, Boot-managed | 4.2.2, unchanged |
| `jakarta.validation:jakarta.validation-api` | 3.1.1, Boot-managed | 3.1.1 retained; no published stable fix |

Published Maven Central POMs verified on 2026-11-03:

- [Boot 4.1.2](https://repo.maven.apache.org/maven2/org/springframework/boot/spring-boot-dependencies/4.1.2/spring-boot-dependencies-4.1.2.pom)
- [Jackson 2 BOM 2.21.8](https://repo.maven.apache.org/maven2/com/fasterxml/jackson/jackson-bom/2.21.8/jackson-bom-2.21.8.pom)
- [Jackson 3 BOM 3.1.8](https://repo.maven.apache.org/maven2/tools/jackson/jackson-bom/3.1.8/jackson-bom-3.1.8.pom)
- [LZ4 1.11.4](https://repo.maven.apache.org/maven2/at/yawk/lz4/lz4-java/1.11.4/lz4-java-1.11.4.pom)
- [Zstandard JNI 1.5.7-14](https://repo.maven.apache.org/maven2/com/github/luben/zstd-jni/1.5.7-14/zstd-jni-1.5.7-14.pom)
- [Snappy 1.1.10.10](https://repo.maven.apache.org/maven2/org/xerial/snappy/snappy-java/1.1.10.10/snappy-java-1.1.10.10.pom)

Additional published POMs and upstream release evidence verified on 2026-11-04:

- [Logback classic 1.6.6](https://repo.maven.apache.org/maven2/ch/qos/logback/logback-classic/1.6.6/logback-classic-1.6.6.pom)
  depends on core from the same parent; manage both artifacts together.
  [Release notes](https://logback.qos.ch/news.html#1.6.5) identify the successive
  MDC discriminator path traversal fixes in 1.6.3 and 1.6.5. Version 1.6.6 contains
  those fixes and further rolling-file corrections. Its parent targets Java 11
  and SLF4J 2.0.19; the existing Java 17 / Boot-managed SLF4J 2.0.20 stack remains.
- [json-smart 2.6.1](https://repo.maven.apache.org/maven2/net/minidev/json-smart/2.6.1/json-smart-2.6.1.pom)
  depends on [accessors-smart 2.6.1](https://repo.maven.apache.org/maven2/net/minidev/accessors-smart/2.6.1/accessors-smart-2.6.1.pom),
  which depends on ASM 9.11.1. The
  [released v2.6.1 changelog](https://github.com/netplex/json-smart-v2/blob/v2.6.1/README.md)
  explicitly fixes CVE-2026-113602 via PR 305. The released tag resolves to commit
  `8cb39c2b01b8cffb6441a86a0e9fc229fb84d870`; its strict parser checks real
  end-of-input rather than accepting the literal 0x1A sentinel as EOF. Snyk's
  advisory still lists all versions as affected with no fixed release on this
  date. Treat source/behavior verification and scanner disposition separately;
  never ignore the advisory or report a clean scan based on this version alone.
- [ASM 9.11.1 POM](https://repo.maven.apache.org/maven2/org/ow2/asm/asm/9.11.1/asm-9.11.1.pom)
  and the published `asm-analysis`, `asm-commons`, `asm-tree`, `asm-util` POMs
  consistently reference 9.11.1 siblings. The
  [official release notes](https://asm.ow2.io/versions.html) list the security
  fixes in 9.11 and subsequent regression fixes in 9.11.1. Management aligns the
  family without adding unused modules to consumer classpaths.
- [Jakarta Validation metadata](https://repo.maven.apache.org/maven2/jakarta/validation/jakarta.validation-api/maven-metadata.xml)
  lists 3.1.1 as the latest stable version and only 4.0.0-M1 above it. The recorded
  Low advisory affects `[2.0.1,4.0.0-M1)` and requires a pre-JDK-24 SecurityManager
  with unequal protection-domain permissions. Keep 3.1.1 and existing validation
  contracts; the Low finding remains unresolved until a stable compatible fix
  exists. No prerelease adoption or dependency removal is part of this policy.

Advisories motivating the exceptions (all IDs have the `SNYK-JAVA-` prefix):

- Jackson: `COMFASTERXMLJACKSONCORE-21761837`,
  `TOOLSJACKSONCORE-21761838` (CVE-2026-110879).
- LZ4: `ATYAWKLZ4-20570516`, `ATYAWKLZ4-20570526`.
- Zstandard: `COMGITHUBLUBEN-19653585`, `COMGITHUBLUBEN-19659571`,
  `COMGITHUBLUBEN-19659584`, `COMGITHUBLUBEN-19659603`,
  `COMGITHUBLUBEN-19659617`, `COMGITHUBLUBEN-19778368`.
- Snappy: `ORGXERIALSNAPPY-19778376`, `ORGXERIALSNAPPY-19964087`,
  `ORGXERIALSNAPPY-20825055`, `ORGXERIALSNAPPY-21755437`.
- Logback: `CHQOSLOGBACK-19882330` (`[0.9.14,1.6.3)`),
  `CHQOSLOGBACK-20824362` (`[0.9.14,1.6.5)`).
- json-smart: `NETMINIDEV-21715438` (scanner currently lists `[0,]`; upstream
  released fix in 2.6.1, fresh scanner disposition required).
- ASM: `ORGOW2ASM-21715439`, `ORGOW2ASM-21715445`, `ORGOW2ASM-21715448`,
  `ORGOW2ASM-21715452` (`[3.0,9.11)`), and `ORGOW2ASM-21715442` (`[5.0,9.11)`).
- Jakarta Validation: `JAKARTAVALIDATION-20936881` (`[2.0.1,4.0.0-M1)`;
  unresolved, no stable fix).

The current milestone qualifies only xln-common and its nested buildSrc working
copy. Later applications adopt the policy by updating their shared buildSrc
revision and using its application convention; their graphs and behavior need
separate verification. This document does not imply those consumers have adopted
or passed the change.

Remove an exception when the canonical platform and supported Spark/Kafka
transitives select a fixed version without it. First remove it in a candidate
change, verify all nine xln dependency graphs (including plugin and fixture loader
graphs), coherent Jackson families/annotations, Logback/minidev/ASM families,
one LZ4 provider, Scala 2.13/plain JAR compatibility and relevant behavior tests,
then rerun unfiltered Snyk scans. Retain Boot and starter-test 4.1.2 while qualifying
these exceptions. Logging checks must load the actual included appender and
encoder; JSON checks must exercise JSONPath and generated bean accessors as well
as strict trailing-input rejection. Existing validation error mapping remains.
Update the independent graph/configuration acceptance values and provenance with
that evidence. Publication and graph alignment alone do not establish security
or runtime compatibility.
