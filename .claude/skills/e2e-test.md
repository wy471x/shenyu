---
name: e2e-test
description: >
  Generate or update ShenYu end-to-end tests following the project's e2e framework conventions.
  Use when writing new e2e test cases, adding test coverage for a plugin, creating scenario-based
  integration tests, or setting up docker-compose/k8s test infrastructure for shenyu-e2e.
---

# ShenYu E2E Test Authoring

## Overview

The ShenYu e2e test framework lives in `shenyu-e2e/` (standalone Maven project, Java 17, JUnit 5.8.2, RestAssured 5.2.0). It tests the full gateway flow: **shenyu-admin → data sync → shenyu-bootstrap → proxied service**, managed through docker-compose or k3s.

### Module structure

| Module | Purpose |
|---|---|
| `shenyu-e2e-common` | Annotations, data models (`SelectorData`, `RuleData`, `Condition`), `ResourceDataTemplate` builders |
| `shenyu-e2e-client` | HTTP clients: `AdminClient` (admin REST API), `GatewayClient` (gateway proxy + actuator) |
| `shenyu-e2e-engine` | JUnit5 extension, scenario DSL (`ScenarioSpec`/`CaseSpec`/etc.), checkers, wait helpers |
| `shenyu-e2e-case` | Actual test case modules: `case-http`, `case-grpc`, `case-apache-dubbo`, `case-websocket`, `case-spring-cloud`, `case-sofa`, `case-cluster`, `case-storage`, `case-logging-kafka`, `case-logging-rocketmq` |

### Test architecture

Every test module follows the **two-file pattern**:

1. **`XxxPluginTest.java`** — JUnit 5 test class annotated with `@ShenYuTest`, wires lifecycle (`@BeforeAll`/`@BeforeEach`/`@AfterEach`), runs scenarios
2. **`XxxPluginCases.java`** — implements `ShenYuScenarioProvider`, builds the DSL scenario specs

---

## Step 1 — Create the case module (if needed)

For a new plugin or protocol, create a new Maven module under `shenyu-e2e/shenyu-e2e-case/`:

```xml
<!-- shenyu-e2e/shenyu-e2e-case/shenyu-e2e-case-<name>/pom.xml -->
<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.apache.shenyu</groupId>
        <artifactId>shenyu-e2e-case</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>
    <artifactId>shenyu-e2e-case-<name></artifactId>
    <dependencies>
        <dependency>
            <groupId>org.apache.shenyu</groupId>
            <artifactId>shenyu-e2e-engine</artifactId>
            <version>${project.version}</version>
        </dependency>
    </dependencies>
</project>
```

Add the module to `shenyu-e2e/shenyu-e2e-case/pom.xml` `<modules>` list.

## Step 2 — Write the Cases class (scenario provider)

Create `src/test/java/org/apache/shenyu/e2e/testcase/<name>/XxxPluginCases.java`.

```java
package org.apache.shenyu.e2e.testcase.<name>;

import com.google.common.collect.Lists;
import org.apache.shenyu.e2e.client.admin.AdminClient;
import org.apache.shenyu.e2e.client.gateway.GatewayClient;
import org.apache.shenyu.e2e.engine.annotation.ShenYuScenario;
import org.apache.shenyu.e2e.engine.annotation.ShenYuTest;
import org.apache.shenyu.e2e.engine.scenario.function.HttpCheckers;
import org.apache.shenyu.e2e.engine.scenario.specification.ScenarioSpec;
import org.apache.shenyu.e2e.engine.scenario.specification.ShenYuBeforeEachSpec;
import org.apache.shenyu.e2e.engine.scenario.specification.ShenYuCaseSpec;
import org.apache.shenyu.e2e.engine.scenario.specification.ShenYuScenarioSpec;
import org.apache.shenyu.e2e.engine.scenario.specification.ShenYuAfterEachSpec;
import org.apache.shenyu.e2e.engine.service.ShenYuScenarioProvider;
import org.apache.shenyu.e2e.model.data.Condition;
import org.apache.shenyu.e2e.model.data.RuleData;
import org.apache.shenyu.e2e.model.data.SelectorData;
import org.apache.shenyu.e2e.model.handle.DivideRuleHandle;  // plugin-specific
import org.apache.shenyu.e2e.model.handle.DivideUpstream;   // plugin-specific
import org.apache.shenyu.e2e.model.handle.DivideUpstreamList; // plugin-specific

import java.util.List;

import static org.apache.shenyu.e2e.model.template.ResourceDataTemplate.newBindingData;
import static org.apache.shenyu.e2e.model.template.ResourceDataTemplate.newCondition;
import static org.apache.shenyu.e2e.model.template.ResourceDataTemplate.newConditions;
import static org.apache.shenyu.e2e.model.template.ResourceDataTemplate.newDivideRuleHandle;
import static org.apache.shenyu.e2e.model.template.ResourceDataTemplate.newRuleBuilder;
import static org.apache.shenyu.e2e.model.template.ResourceDataTemplate.newSelectorBuilder;
import static org.apache.shenyu.e2e.model.template.ResourceDataTemplate.newUpstreamsBuilder;

public class XxxPluginCases implements ShenYuScenarioProvider {

    @Override
    public List<ScenarioSpec> get() {
        return Lists.newArrayList(
                scenario1(),
                scenario2()
        );
    }

    private ScenarioSpec scenario1() {
        return ShenYuScenarioSpec.builder()
                .name("scenario name shown in test report")
                .beforeEachSpec(ShenYuBeforeEachSpec.builder()
                        // Checker: assert route does NOT exist before test
                        .checker(HttpCheckers.notExists("/http/test/endpoint"))
                        // Create selector + rule + binding
                        .addSelectorAndRule(
                                newSelectorBuilder("my-selector", Plugin.DIVIDE)
                                        .handle(newUpstreamsBuilder("example-host.com"))
                                        .conditionList(newConditions(
                                                Condition.ParamType.URI,
                                                Condition.Operator.EQUAL,
                                                "/test/**"))
                                        .build(),
                                newBindingData("my-selector", Plugin.DIVIDE.getAlias(), "example-host.com"),
                                newRuleBuilder("my-rule")
                                        .handle(newDivideRuleHandle())
                                        .conditionList(newConditions(
                                                Condition.ParamType.URI,
                                                Condition.Operator.EQUAL,
                                                "/test/**"))
                                        .build()
                        )
                        // Waiting: poll until route takes effect
                        .waiting(HttpCheckers.exists("/http/test/endpoint"))
                        .build())
                .caseSpec(ShenYuCaseSpec.builder()
                        // Add assertions (each runs as a separate verification step)
                        .addExists("/http/test/endpoint")
                        .addExists("GET", "/http/test/endpoint")  // explicit method
                        .addExists("POST", "/http/test/endpoint",
                                Map.of("key", "value"))  // POST with JSON body
                        .addNotExists("/http/wrong-path")
                        .addVerifier("/status/200", isEmptyOrNullString())  // Hamcrest body matcher
                        .build())
                .afterEachSpec(ShenYuAfterEachSpec.builder()
                        // Cleanup: poll until route is gone after deletion
                        .deleteWaiting(HttpCheckers.notExists("/http/test/endpoint"))
                        .build())
                .build();
    }

    // ... more scenario methods
}
```

### Plugin enum reference

The `Plugin` enum maps names to admin plugin IDs:

| Plugin | Enum value | Admin ID |
|---|---|---|
| divide | `Plugin.DIVIDE` | 5 |
| dubbo | `Plugin.DUBBO` | 6 |
| spring-cloud | `Plugin.SPRING_CLOUD` | 8 |
| grpc | `Plugin.GRPC` | 15 |
| websocket | `Plugin.WEBSOCKET` | 26 |
| sofa | `Plugin.SOFA` | — |

### Condition types

```java
Condition.ParamType.URI      // match request path
Condition.ParamType.METHOD   // match HTTP method (use Operator.EQUAL with "GET"/"POST"/etc.)
Condition.ParamType.HEADER   // match request header
Condition.ParamType.QUERY    // match query parameter
Condition.ParamType.HOST     // match Host header
Condition.ParamType.IP       // match client IP
Condition.ParamType.COOKIE   // match cookie

Condition.Operator.EQUAL         // "="
Condition.Operator.MATCH         // regex match
Condition.Operator.STARTS_WITH
Condition.Operator.ENDS_WITH
Condition.Operator.CONTAINS
Condition.Operator.PATH_PATTERN  // ant-style /** matching
Condition.Operator.REGEX
Condition.Operator.EXCLUDE       // negated match
```

### Template helpers (`ResourceDataTemplate`)

```java
// Selector builder — pre-sets: type=CUSTOM, matchMode=AND, continued/logged/enabled=true, sort=1
newSelectorBuilder("selectorName", Plugin.DIVIDE)
    .handle(newUpstreamsBuilder("host.com"))  // or custom handle object
    .conditionList(newConditions(ParamType.URI, Operator.EQUAL, "/path/**"))
    .build()

// Rule builder
newRuleBuilder("ruleName")
    .handle(newDivideRuleHandle())  // plugin-specific handle
    .conditionList(newConditions(ParamType.URI, Operator.EQUAL, "/path/**"))
    .build()

// Binding data (for proxy selector)
newBindingData("selectorName", Plugin.DIVIDE.getAlias(), "upstream-host.com")

// Convenience: single condition
newCondition(ParamType.METHOD, Operator.EQUAL, "GET")

// Convenience: list of conditions (all same paramType + operator, different values)
newConditions(ParamType.URI, Operator.EQUAL, "/foo/**", "/bar/**")
```

### Plugin-specific handles

```java
// Divide (HTTP proxy)
import org.apache.shenyu.e2e.model.handle.DivideRuleHandle;
import org.apache.shenyu.e2e.model.handle.DivideUpstream;
import org.apache.shenyu.e2e.model.handle.DivideUpstreamList;
newDivideRuleHandle()  // hash LB, retry=1, timeout=3000, max header/request=10240

// Dubbo
import org.apache.shenyu.e2e.model.handle.DubboHandler;
import org.apache.shenyu.e2e.model.handle.DubboRuleHandle;
DubboHandler.builder().protocol("dubbo://").upstreamUrl("dubbo:20888")
    .status(true).upstreamHost("dubbo").weight(50).warmup(600000).build()
DubboRuleHandle.builder().loadBalance("random").timeout(3000).retries(0).build()

// gRPC
import org.apache.shenyu.e2e.model.handle.GrpcSelectorHandle;
import org.apache.shenyu.e2e.model.handle.GrpcRuleHandle;
GrpcSelectorHandle.builder().upstreamUrl("localhost:8080").build()
GrpcRuleHandle.builder().build()

// Spring Cloud
import org.apache.shenyu.e2e.model.handle.SpringCloudSelectorHandle;
import org.apache.shenyu.e2e.model.handle.SpringCloudRuleHandle;
SpringCloudSelectorHandle.builder().serviceId("serviceName").build()
SpringCloudRuleHandle.builder().loadBalance("roundRobin").timeout(3000).build()

// SOFA
import org.apache.shenyu.e2e.model.handle.SofaSelectorHandle;
import org.apache.shenyu.e2e.model.handle.SofaRuleHandle;
SofaSelectorHandle.builder().serviceId("serviceName").build()
SofaRuleHandle.builder().loadBalance("random").timeout(3000).retries(0).build()
```

## Step 3 — Write the Test class (JUnit runner)

Create `src/test/java/org/apache/shenyu/e2e/testcase/<name>/XxxPluginTest.java`.

```java
package org.apache.shenyu.e2e.testcase.<name>;

import org.apache.shenyu.e2e.client.admin.AdminClient;
import org.apache.shenyu.e2e.client.gateway.GatewayClient;
import org.apache.shenyu.e2e.client.WaitDataSync;
import org.apache.shenyu.e2e.engine.annotation.ShenYuAdminClient;
import org.apache.shenyu.e2e.engine.annotation.ShenYuGatewayClient;
import org.apache.shenyu.e2e.engine.annotation.ShenYuScenario;
import org.apache.shenyu.e2e.engine.annotation.ShenYuTest;
import org.apache.shenyu.e2e.engine.scenario.specification.AfterEachSpec;
import org.apache.shenyu.e2e.engine.scenario.specification.BeforeEachSpec;
import org.apache.shenyu.e2e.engine.scenario.specification.CaseSpec;
import org.apache.shenyu.e2e.engine.service.ServiceConfigure;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

@ShenYuTest(environments = {
        @Environment(serviceName = "admin", configure = @ServiceConfigure(
                baseUrl = "http://localhost:31095",
                type = ServiceTypeEnum.SHENYU_ADMIN,
                parameters = {
                        @Parameter(key = "username", value = "admin"),
                        @Parameter(key = "password", value = "123456")
                })),
        @Environment(serviceName = "gateway", configure = @ServiceConfigure(
                baseUrl = "http://localhost:31195",
                type = ServiceTypeEnum.SHENYU_GATEWAY))
})
public class XxxPluginTest {

    // Track created selector IDs for cleanup
    private final List<String> selectorIds = new ArrayList<>();

    @BeforeAll
    void setup(
            @ShenYuAdminClient AdminClient adminClient,
            @ShenYuGatewayClient GatewayClient gatewayClient) {
        adminClient.login();

        // Wait for initial data sync (selectors, metadata, rules)
        WaitDataSync.waitAdmin2GatewayDataSyncEquals(
                () -> adminClient.listAllSelectors().size(),
                () -> gatewayClient.getSelectorCache().size(),
                adminClient);
        WaitDataSync.waitAdmin2GatewayDataSyncEquals(
                () -> adminClient.listAllMetaData().size(),
                () -> gatewayClient.getMetaDataCache().size(),
                adminClient);
        WaitDataSync.waitAdmin2GatewayDataSyncEquals(
                () -> adminClient.listAllRules().size(),
                () -> gatewayClient.getRuleCache().size(),
                adminClient);

        // For non-divide plugins: enable the plugin and wait for gateway to pick it up
        // adminClient.changePluginStatus("<namespace-plugin-id>",
        //     Map.of("pluginId", "6", "name", "dubbo", "enabled", "true",
        //            "role", "Proxy", "sort", "200", "namespaceId", Constants.SYS_DEFAULT_NAMESPACE_NAMESPACE_ID));
        // WaitDataSync.waitGatewayPluginUse(gatewayClient, "org.apache.shenyu.plugin.apache.dubbo.ApacheDubboPlugin");

        // Optional: clean up existing selectors for isolation
        // adminClient.deleteAllSelectors();
    }

    @BeforeEach
    void beforeEach(
            @ShenYuAdminClient AdminClient client,
            @ShenYuGatewayClient GatewayClient gateway,
            BeforeEachSpec spec) {
        // Pre-condition check
        spec.getChecker().check(gateway);

        // Create selectors and rules via admin API
        selectorIds.clear();
        spec.getResources().getResources().forEach(res -> {
            SelectorData selector = res.getSelector();
            client.create(selector);
            selectorIds.add(selector.getId());

            res.getRules().forEach(rule -> {
                rule.setSelectorId(selector.getId());
                client.create(rule);
            });

            // If binding data present (proxy-selector)
            if (res.getBindingData() != null) {
                res.getBindingData().setSelectorId(selector.getId());
                res.getBindingData().setNamespaceId(selector.getNamespaceId());
                client.bindingData(res.getBindingData());
            }
        });

        // Wait for data to propagate to gateway
        spec.getWaiting().waitFor(gateway);
    }

    @AfterEach
    void afterEach(
            @ShenYuAdminClient AdminClient client,
            @ShenYuGatewayClient GatewayClient gateway,
            AfterEachSpec spec) {
        // Delete created selectors (cascades to rules)
        spec.getDeleter().delete(client, selectorIds);
        // Wait for deletion to propagate
        spec.deleteWaiting().waitFor(gateway);
    }

    // The actual test — one invocation per ScenarioSpec
    @ShenYuScenario(provider = XxxPluginCases.class)
    void testScenario(GatewayClient gateway, CaseSpec spec) {
        spec.getVerifiers().forEach(v ->
                v.verify(gateway.getHttpRequesterSupplier().get()));
    }

    // Optional cleanup
    // @AfterAll
    // void tearDown(@ShenYuAdminClient AdminClient client) {
    //     client.deleteAllSelectors();
    // }
}
```

### Key annotations reference

- **`@ShenYuTest(environments = {...})`** — class-level. Declares admin + gateway + external service endpoints. Validates `baseUrl` reachability before tests run. Each `Environment` has:
  - `serviceName()` — unique name
  - `configure()` — `ServiceConfigure` with `moduleName`, `baseUrl`, `type` (SHENYU_ADMIN / SHENYU_GATEWAY / EXTERNAL_SERVICE), and `parameters[]`
  - Admin environments **must** include `username` and `password` parameters.

- **`@ShenYuScenario(provider = XxxPluginCases.class)`** — method-level test template. One invocation per `ScenarioSpec` returned by the provider. Injects `CaseSpec` + clients.

## Step 4 — WebSocket tests

For websocket tests, the test method uses `WebSocketVerifier` instead of `Verifier`:

```java
@ShenYuScenario(provider = WebSocketPluginCases.class)
void testWebSocket(GatewayClient gateway, CaseSpec spec) {
    spec.getWebSocketVerifiers().forEach(w ->
            w.verify(gateway.getWebSocketClientSupplier().get(), gateway));
}
```

WebSocket case specs use `WebSocketCheckers`:

```java
ShenYuCaseSpec.builder()
    .addExists("/ws-native/myWebSocket?token=Jack",
               "Hello ShenYu",                    // message to send
               "apache shenyu server reply")      // expected reply
    .addNotExists("/ws-annotation/myWs",
                  "Hello ShenYu")                 // message that should fail
    .build()
```

In `@BeforeAll`, enable the websocket plugin:
```java
adminClient.changePluginStatus("1801816010882822163",
    Map.of("pluginId", "26", "name", "websocket", "enabled", "true",
           "role", "Proxy", "sort", "200",
           "namespaceId", Constants.SYS_DEFAULT_NAMESPACE_NAMESPACE_ID));
WaitDataSync.waitGatewayPluginUse(gatewayClient,
    "org.apache.shenyu.plugin.websocket.WebSocketPlugin");
```

## Step 5 — Docker-compose setup

Each case module needs `compose/` and `k8s/` directories.

### Compose directory structure

```
shenyu-e2e-case/shenyu-e2e-case-<name>/
├── compose/
│   ├── script/
│   │   └── e2e-<name>-compose.sh   # CI entry point
│   ├── shenyu-examples-<name>-compose.yml
│   └── sync/                        # (or reuse shared sync configs)
│       └── shenyu-sync-websocket.yml
```

### Compose script template

```bash
#!/bin/bash
# compose/script/e2e-<name>-compose.sh

SHENYU_TESTCASE_DIR=$(cd "$(dirname "$0")"/../../../../.. && pwd)

# Optional: initialize storage
# bash "${SHENYU_TESTCASE_DIR}"/k8s/script/storage/storage_init_mysql.sh

# Create external network
docker network create shenyu --driver bridge 2>/dev/null || true

SYNC_ARRAY=("websocket")  # or "http", "zookeeper", "nacos"
for sync in "${SYNC_ARRAY[@]}"; do
    # Start middleware + admin + bootstrap
    docker compose -f "${SHENYU_TESTCASE_DIR}"/shenyu-e2e-case/compose/sync/shenyu-sync-${sync}.yml up -d --quiet-pull

    # Healthcheck: poll admin and bootstrap until healthy
    bash "${SHENYU_TESTCASE_DIR}"/k8s/script/healthcheck.sh http://localhost:31095/actuator/health 30 2
    bash "${SHENYU_TESTCASE_DIR}"/k8s/script/healthcheck.sh http://localhost:31195/actuator/health 30 2

    # Start example service
    docker compose -f compose/shenyu-examples-<name>-compose.yml up -d
    bash "${SHENYU_TESTCASE_DIR}"/k8s/script/healthcheck.sh http://localhost:31189/actuator/health 30 2

    # Run tests
    ./mvnw -B -f ./shenyu-e2e/pom.xml -pl shenyu-e2e-case/shenyu-e2e-case-<name> -am test \
        || { docker compose logs shenyu-admin shenyu-bootstrap; exit 1; }

    # Teardown
    docker compose -f compose/shenyu-examples-<name>-compose.yml down
    docker compose -f "${SHENYU_TESTCASE_DIR}"/shenyu-e2e-case/compose/sync/shenyu-sync-${sync}.yml down
done
```

### Reuse shared compose configs

Shared sync compose files are in `shenyu-e2e-case/compose/sync/`:
- `shenyu-sync-websocket.yml` — shenyu-mysql + shenyu-admin (port 31095) + shenyu-bootstrap (port 31195)
- `shenyu-sync-http.yml`
- `shenyu-sync-zookeeper.yml` — shenyu-zookeeper + mysql + admin + bootstrap

Shared storage compose files in `shenyu-e2e-case/compose/storage/`:
- `shenyu-storage-h2.yml`, `shenyu-storage-mysql.yml`, `shenyu-storage-postgres.yml`, `shenyu-storage-opengauss.yml`

The example service compose file is module-specific — it starts whatever backend service the tests proxy to.

## Step 6 — Register in CI

Add your test to `.github/workflows/e2e-k8s.yml` under the `e2e-case` job matrix:

```yaml
# In the matrix strategy section:
case:
  - { name: "http", script: "e2e-http-sync-compose.sh" }
  - { name: "grpc", script: "e2e-grpc-sync-compose.sh" }
  - { name: "<name>", script: "e2e-<name>-compose.sh" }   # add this
```

## DataSync tests

In addition to scenario tests, every plugin should have a `DataSynTest` class verifying that admin→gateway data sync works:

```java
@Test
void testDataSyn(
        @ShenYuAdminClient AdminClient adminClient,
        @ShenYuGatewayClient GatewayClient gatewayClient) {
    // Create selector + rule via admin API, then assert gateway cache reflects them
    SelectorData selector = newSelectorBuilder("syn-test", Plugin.DIVIDE)
            .handle(newUpstreamsBuilder("localhost"))
            .conditionList(newConditions(ParamType.URI, Operator.EQUAL, "/syn-test"))
            .build();
    adminClient.create(selector);
    selectorIds.add(selector.getId());

    // ... create rules, then verify gateway cache
    WaitDataSync.waitAdmin2GatewayDataSyncEquals(
            () -> adminClient.listAllSelectors().size(),
            () -> gatewayClient.getSelectorCache().size(),
            adminClient);
}
```

## Common pitfalls

1. **Pre-docker health checks**: `@ShenYuTest` validates all `baseUrl` values are reachable before any test runs. If this fails, the entire test class fails (not skipped).

2. **Selector ID tracking**: Always track created selector IDs in `@BeforeEach` and pass them to `spec.getDeleter().delete()` in `@AfterEach`. Failing to clean up leaves state that can break other scenarios.

3. **Plugin enablement**: Non-divide plugins (dubbo, grpc, websocket, etc.) need explicit `changePluginStatus()` + `waitGatewayPluginUse()` in `@BeforeAll`.

4. **Data sync delays**: Always use `WaitDataSync` or `WaitForHelper.waitForEffecting()` — never `Thread.sleep()`. The framework retries up to 30 times at 3s intervals.

5. **Namespace**: Use `Constants.SYS_DEFAULT_NAMESPACE_NAMESPACE_ID` (`649330b6-c2d7-4edc-be8e-8a54df9eb385`) as the default namespaceId.

6. **Name uniqueness**: The framework auto-wraps resource names with scenario IDs (`NameUtils.wrap(name, scenarioId)`) to prevent conflicts across test runs.

7. **Response format**: AdminClient expects exact response messages (e.g., `"create success"`, `"query success"`, `"delete success"`). The HTTP checker `exists()` asserts `code` is null and `message` does NOT contain "please check your configuration!", while `notExists()` asserts `code < 0` and message DOES contain "please check your configuration!".
