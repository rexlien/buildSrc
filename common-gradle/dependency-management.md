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

Each consumer adopts this policy through its shared buildSrc working copy and
application convention. Its actual graphs and behavior require separate
verification; this document alone does not establish adoption or a passing run.

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

## Application and early plugin security correction — 2026-11-05

`plugin-dependency-management.gradle` is a separate canonical policy applied to
each root **ScriptHandler before the plugins DSL**. Its native Jackson 3 BOM
platform aligns the actual plugin core/databind family at 3.1.8 and annotations
at 2.21. A native classpath constraint selects Commons Lang 3.20.0, matching the
current application platform and exceeding the advisory's 3.18.0 fixed floor.
Application BOM imports happen too late to secure this graph. Do not replace
this policy with application pins, a resolution strategy or forced versions.
The plain xln build retains its no-Boot-plugin boundary; constraints do not add
unused Lang or Jackson implementation modules. Consumers must apply the policy
at each entry point that resolves a plugin classpath.

Boot Gradle plugin 4.1.2 owns buildpack-platform 4.1.2, which owns Jackson
databind 3.1.7; Boot plugin/buildpack/loader tooling owns Commons Compress
1.27.1, which owns Commons Lang 3.16.0. Published POMs verified on 2026-11-05:

- [Boot Gradle plugin](https://repo.maven.apache.org/maven2/org/springframework/boot/spring-boot-gradle-plugin/4.1.2/spring-boot-gradle-plugin-4.1.2.pom),
  [buildpack platform](https://repo.maven.apache.org/maven2/org/springframework/boot/spring-boot-buildpack-platform/4.1.2/spring-boot-buildpack-platform-4.1.2.pom),
  [loader tools](https://repo.maven.apache.org/maven2/org/springframework/boot/spring-boot-loader-tools/4.1.2/spring-boot-loader-tools-4.1.2.pom).
- [Commons Compress 1.27.1](https://repo.maven.apache.org/maven2/org/apache/commons/commons-compress/1.27.1/commons-compress-1.27.1.pom).
  `SNYK-JAVA-ORGAPACHECOMMONS-10734078` is fixed in Lang 3.18.0 and above;
  `SNYK-JAVA-TOOLSJACKSONCORE-21761838` is fixed in Jackson 3.1.8 or 3.2.4.

The application policy manages the Apache HTTP Components 4 family together:
httpclient/httpmime 4.5.14 and httpcore 4.4.16. This also aligns older transitive
requests from consumers. Consumers must verify their actual HTTP behavior,
including request encoding, payload bytes, response handling and failures.

- [HttpClient 4.5.14](https://repo.maven.apache.org/maven2/org/apache/httpcomponents/httpclient/4.5.14/httpclient-4.5.14.pom),
  [HttpMime 4.5.14](https://repo.maven.apache.org/maven2/org/apache/httpcomponents/httpmime/4.5.14/httpmime-4.5.14.pom),
  [shared parent](https://repo.maven.apache.org/maven2/org/apache/httpcomponents/httpcomponents-client/4.5.14/httpcomponents-client-4.5.14.pom)
  align the family and httpcore 4.4.16; the parent targets Java 6, compatible
  with the existing Java 17 runner.
- `SNYK-JAVA-ORGAPACHEHTTPCOMPONENTS-1048058`, `30647`, `31517`
  have fixed floors 4.5.13, 4.3.6 and 4.5.3 respectively.

Remove HTTP4 management or early plugin exceptions only when their owning
upstream graphs select the compatible fixed family without them. Verify actual
plugin, application and test dependency graphs, generator conventions and
packaged JAR entries, plus affected consumer behavior, before fresh unfiltered
security scans. The json-smart scanner discrepancy and unresolved Jakarta
Validation advisory remain separate; these changes do not imply a clean scan.

## Provided Spark dependency security correction — 2026-11-16

The compile-only Spark 4.2.1 graph also needs security management. Its dependencies
are supplied by a Spark host, so absence from the library runtime graph does not
make their findings irrelevant. Spark 4.3.0's published parent still declares the
same affected versions; changing the Spark version alone does not fix them.
Keep the upstream Spark artifacts and existing Scala/public API boundaries.

| Family | Spark baseline | Managed fix |
| --- | --- | --- |
| Vert.x core, web-client, web-common, auth-common, uri-template | 4.5.26 | 4.5.36 |
| Ivy | 2.5.3 | 2.6.0 |
| Parquet column, common, encoding, format-structures, hadoop, jackson, variant | 1.17.0 | 1.18.1 |
| XMLSchema core | 2.3.1 | 2.3.3 |
| ZooKeeper / jute | 3.9.5 | 3.9.6 |

Published Maven Central POMs checked on 2026-11-16:

- [Spark 4.3.0 parent](https://repo.maven.apache.org/maven2/org/apache/spark/spark-parent_2.13/4.3.0/spark-parent_2.13-4.3.0.pom)
  declares the affected versions above and Java 17 / Scala 2.13.18.
- [Vert.x 4.5.36 dependency management](https://repo.maven.apache.org/maven2/io/vertx/vertx-dependencies/4.5.36/vertx-dependencies-4.5.36.pom)
  aligns these Vert.x modules. Manage only the used family here, preserving
  Boot's Netty ownership rather than importing Vert.x's Netty 4.1 BOM.
- [Ivy 2.6.0](https://repo.maven.apache.org/maven2/org/apache/ivy/ivy/2.6.0/ivy-2.6.0.pom).
- [Parquet 1.18.1 parent](https://repo.maven.apache.org/maven2/org/apache/parquet/parquet/1.18.1/parquet-1.18.1.pom)
  lists the aligned Parquet modules; management does not add unused modules.
- [XMLSchema 2.3.3](https://repo.maven.apache.org/maven2/org/apache/ws/xmlschema/xmlschema-core/2.3.3/xmlschema-core-2.3.3.pom).
- [ZooKeeper 3.9.6](https://repo.maven.apache.org/maven2/org/apache/zookeeper/zookeeper/3.9.6/zookeeper-3.9.6.pom)
  requires jute from the same release.

Snyk advisory fixed floors motivating these selections (all IDs have the
`SNYK-JAVA-` prefix): Vert.x `IOVERTX-16433278`, `19433234`, `20954498`,
`22190589`; Ivy `ORGAPACHEIVY-19432141`; Parquet `ORGAPACHEPARQUET-19666552`;
XMLSchema `ORGAPACHEWSXMLSCHEMA-22151046`, `22283204`; ZooKeeper
`ORGAPACHEZOOKEEPER-20158985`, `20158990`, `20159004`, `20159005`.

Declaration and resolved-graph guards independently check these managed versions,
including provided compile graphs. Qualify them with the full library check,
the separate Spark API consumer linkage check and fresh unfiltered Snyk results.
The actual Spark host still needs its own runtime qualification and dependency
adoption; library management does not patch an external Spark distribution.
Remove each exception when the supported upstream Spark graph supplies its fixed
family without the exception, then repeat those checks.
