# ThingsBoard Upgrade Testing Strategy
## Comprehensive Guide for Testing Custom Rule Nodes and Extensions

> **Document Version:** 1.0
> **Last Updated:** 2025-11-18
> **Purpose:** Deep analysis and testing strategy for ThingsBoard upgrades with custom code

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Understanding the Upgrade Landscape](#understanding-the-upgrade-landscape)
3. [Risk Assessment Framework](#risk-assessment-framework)
4. [Pre-Upgrade Preparation](#pre-upgrade-preparation)
5. [Testing Strategy Overview](#testing-strategy-overview)
6. [Custom Rule Node Testing](#custom-rule-node-testing)
7. [Rule Chain Testing](#rule-chain-testing)
8. [Integration Testing](#integration-testing)
9. [Performance and Load Testing](#performance-and-load-testing)
10. [Rollback and Recovery Planning](#rollback-and-recovery-planning)
11. [Post-Upgrade Validation](#post-upgrade-validation)
12. [Automation and CI/CD](#automation-and-cicd)
13. [Appendix: Checklists and Templates](#appendix-checklists-and-templates)

---

## Executive Summary

ThingsBoard upgrades, especially when custom rule nodes and extensions are involved, require a methodical, multi-layered testing approach. This document provides a comprehensive strategy based on:

- **Sequential upgrade requirements** (must upgrade step-by-step through versions)
- **API breaking changes** between versions (Java 17, Angular 15, removed libraries)
- **Database migration complexities** (automatic schema updates, Redis flushing)
- **Custom code compatibility** (rule nodes, widgets, integrations)

### Key Principles

1. **Never skip versions** - Always upgrade sequentially
2. **Test in isolation** - Separate environments for each testing phase
3. **Automate everything** - CI/CD pipelines for continuous validation
4. **Document all customizations** - Maintain inventory of custom code
5. **Plan for rollback** - Always have a recovery path

---

## Understanding the Upgrade Landscape

### ThingsBoard Version Upgrade Path

ThingsBoard requires **sequential upgrades**. You cannot jump from version 3.3 to 3.7 directly.

**Example Upgrade Path:**
```
3.3.x → 3.4.4 → 3.5.x → 3.6.4 → 3.7.x → 4.0.x → 4.2.x
```

Each upgrade includes:
- Database schema updates (automatic via upgrade scripts)
- Configuration file changes (require manual merge)
- Dependency updates (Java, Angular, libraries)
- API changes (potential breaking changes)

### Critical Breaking Changes by Version

#### Version 3.7
- **Java 17 migration** - Must install JDK 17, set as default
- **Impact:** All custom Java code must be compatible with Java 17
- **Testing focus:** Compilation, runtime compatibility, deprecated API usage

#### Version 3.8.1
- **Angular 15 migration** - Complete UI framework upgrade
- **Removed flex-layout library** - Breaking change for custom widgets
- **Impact:** Custom widgets and rule node UIs must be rebuilt on Angular 15
- **Testing focus:** UI rendering, custom widget functionality, responsive layouts

#### Version 4.x
- **REST API changes** - Removed deprecated parameters (e.g., 'useRedisQueueForMessagePersistence')
- **Impact:** API clients, integrations, custom nodes using REST API
- **Testing focus:** API integration tests, external system connectivity

### Database Migration Considerations

#### PostgreSQL-Only Since 3.0
- **Cassandra deprecated** - Must migrate to PostgreSQL
- **Migration path:** Use official migration scripts
- **Testing focus:** Data integrity, query performance, index effectiveness

#### Redis Cache Management
- **Requirement:** Flush all Redis keys before upgrade
- **Command:** `FLUSHALL` on Redis instance
- **Impact:** Temporary cache miss, increased DB load post-upgrade
- **Testing focus:** Cache rebuild performance, application behavior during cache warmup

---

## Risk Assessment Framework

### Custom Code Risk Matrix

| Component Type | Risk Level | Why | Testing Priority |
|---------------|-----------|-----|------------------|
| **Custom Rule Nodes** | HIGH | Direct dependency on ThingsBoard APIs, potential breaking changes | P0 - Critical |
| **Custom Widgets** | HIGH | Angular version dependencies, UI library changes | P0 - Critical |
| **Rule Chains** | MEDIUM | Configuration-based, but may reference deprecated nodes | P1 - High |
| **Integrations** | MEDIUM | External dependencies, API compatibility | P1 - High |
| **Database Customizations** | HIGH | Schema changes may conflict | P0 - Critical |
| **Custom Plugins** | HIGH | Deep system integration | P0 - Critical |
| **Configuration Files** | LOW | Manual merge supported | P2 - Medium |

### Identifying Custom Code

Before upgrading, create a **comprehensive inventory**:

```bash
# Custom Rule Nodes (Java)
find . -name "*.java" -path "*/rule/nodes/*" -type f

# Custom Widgets (TypeScript/Angular)
find . -name "*.ts" -path "*/widget/*" -type f
find . -name "*.component.*" -type f

# Rule Chain Definitions (JSON)
find . -name "*rule-chain*.json" -type f

# Custom Integrations
grep -r "extends.*Integration" --include="*.java"

# Custom Plugins
grep -r "extends.*Plugin" --include="*.java"
```

### Dependency Analysis

**For each custom component, document:**

1. **ThingsBoard APIs used**
   - Which interfaces/classes are extended?
   - Which internal APIs are called?
   - Are any APIs marked as deprecated?

2. **External dependencies**
   - Third-party libraries
   - Version compatibility with new Java/Angular
   - Security vulnerabilities in dependencies

3. **Configuration requirements**
   - Custom configuration parameters
   - Environment variables
   - External service connections

4. **Data dependencies**
   - Database tables accessed
   - Custom schema elements
   - Attribute/telemetry structures

---

## Pre-Upgrade Preparation

### 1. Environment Setup

Create **multiple isolated environments**:

```
┌─────────────────┐
│   PRODUCTION    │  ← Never test here first
└─────────────────┘
        ↑
        │ (Final deployment)
        │
┌─────────────────┐
│   STAGING       │  ← Pre-production validation
└─────────────────┘
        ↑
        │
┌─────────────────┐
│   TEST/QA       │  ← Functional testing
└─────────────────┘
        ↑
        │
┌─────────────────┐
│   DEV/UPGRADE   │  ← Initial upgrade testing
└─────────────────┘
```

### 2. Data Backup Strategy

**Complete backup checklist:**

- [ ] PostgreSQL database dump
  ```bash
  pg_dump -U postgres thingsboard > thingsboard_backup_$(date +%Y%m%d).sql
  ```

- [ ] Configuration files
  ```bash
  tar -czf config_backup_$(date +%Y%m%d).tar.gz \
    /etc/thingsboard/ \
    /usr/share/thingsboard/conf/
  ```

- [ ] Custom code repository
  ```bash
  git tag pre-upgrade-$(date +%Y%m%d)
  git push --tags
  ```

- [ ] Rule chain exports (JSON)
  ```bash
  # Via UI: Export each rule chain to JSON
  # Store in version control
  ```

- [ ] Device/Asset/Dashboard exports
  ```bash
  # Use ThingsBoard REST API to export entities
  # Maintain in separate backup directory
  ```

- [ ] Redis snapshot (if used for caching)
  ```bash
  redis-cli BGSAVE
  ```

### 3. Documentation Requirements

Create these documents **before starting**:

#### Custom Code Inventory
```markdown
# Custom Components Inventory

## Custom Rule Nodes
| Name | File | Purpose | TB APIs Used | Last Modified | Owner |
|------|------|---------|--------------|---------------|-------|
| CustomAggregationNode | CustomAggregationNode.java | Aggregate sensor data | TbNode, TbContext | 2024-10-15 | John Doe |

## Custom Widgets
| Name | File | Purpose | Dependencies | Last Modified | Owner |
|------|------|---------|--------------|---------------|-------|
| CustomGaugeWidget | custom-gauge.component.ts | Display custom metrics | flex-layout | 2024-09-20 | Jane Smith |

## Rule Chains
| Name | Custom Nodes Used | External Integrations | Complexity |
|------|-------------------|----------------------|------------|
| Main Device Telemetry | CustomAggregationNode | Kafka, MQTT | High |
```

#### API Usage Map
```markdown
# ThingsBoard API Usage

## Custom Rule Nodes

### CustomAggregationNode
- **Extends:** org.thingsboard.rule.engine.api.TbNode
- **Uses:**
  - TbContext.tellNext()
  - TbContext.getAttributeService()
  - TbMsg.getData()
- **Potential Issues:**
  - Check if TbContext API changed in target version
  - Verify attribute service method signatures
```

### 4. Version Compatibility Check

**Research target version for:**

1. **Breaking changes**
   - Review release notes: https://thingsboard.io/docs/reference/releases/
   - Check GitHub issues for breaking changes
   - Search for migration guides

2. **Deprecated API usage**
   ```bash
   # Search your custom code for deprecated annotations
   grep -r "@Deprecated" --include="*.java" /path/to/thingsboard/source

   # Check your code against deprecated APIs
   grep -r "YourCustomClass" /path/to/custom/code |
     grep -f deprecated_apis.txt
   ```

3. **Dependency compatibility**
   ```xml
   <!-- Example: Check if your custom rule node dependencies are compatible -->
   <dependencies>
     <dependency>
       <groupId>org.thingsboard</groupId>
       <artifactId>rule-engine-api</artifactId>
       <version>${target.version}</version> <!-- Test compilation -->
     </dependency>
   </dependencies>
   ```

---

## Testing Strategy Overview

### Multi-Layer Testing Pyramid

```
                    ┌─────────────┐
                    │  Manual     │ ← Exploratory testing
                    │  Testing    │
                    └─────────────┘
              ┌──────────────────────┐
              │   E2E Testing        │ ← Complete workflow validation
              └──────────────────────┘
        ┌────────────────────────────────┐
        │   Integration Testing          │ ← Rule chains, APIs, DB
        └────────────────────────────────┘
    ┌────────────────────────────────────────┐
    │   Component Testing                    │ ← Individual rule nodes
    └────────────────────────────────────────┘
┌──────────────────────────────────────────────┐
│   Unit Testing                               │ ← Business logic
└──────────────────────────────────────────────┘
```

### Testing Phases

#### Phase 1: Pre-Upgrade Validation
- **Goal:** Establish baseline
- **Duration:** 1-2 weeks
- **Activities:**
  - Document current system behavior
  - Create test cases for all custom functionality
  - Establish performance baselines
  - Export all configurations

#### Phase 2: Upgrade in Dev Environment
- **Goal:** Identify issues early
- **Duration:** 1-2 weeks
- **Activities:**
  - Perform upgrade on dev environment
  - Run automated test suite
  - Fix compilation/runtime errors
  - Update custom code as needed

#### Phase 3: Comprehensive Testing
- **Goal:** Validate all functionality
- **Duration:** 2-4 weeks
- **Activities:**
  - Unit testing of custom components
  - Integration testing of rule chains
  - Performance testing under load
  - Security testing

#### Phase 4: Staging Validation
- **Goal:** Production-like validation
- **Duration:** 1-2 weeks
- **Activities:**
  - Deploy to staging with production data copy
  - Run full regression suite
  - User acceptance testing
  - Performance validation

#### Phase 5: Production Deployment
- **Goal:** Safe production upgrade
- **Duration:** 1 day + monitoring period
- **Activities:**
  - Scheduled maintenance window
  - Upgrade execution
  - Smoke testing
  - Monitoring and validation

---

## Custom Rule Node Testing

### Understanding Custom Rule Nodes

Custom rule nodes extend ThingsBoard's processing capabilities. They:
- Implement `TbNode` interface
- Process messages asynchronously
- Can call external services
- May use ThingsBoard internal APIs
- Have configuration UIs (Angular components)

### Unit Testing Approach

#### Test Structure

```java
package com.yourcompany.rule.nodes;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;
import org.thingsboard.rule.engine.api.*;
import org.thingsboard.server.common.data.msg.*;

import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

public class CustomAggregationNodeTest {

    @Mock
    private TbContext ctx;

    @Mock
    private TbNodeConfiguration config;

    private CustomAggregationNode node;

    @BeforeEach
    public void setUp() throws TbNodeException {
        MockitoAnnotations.openMocks(this);
        node = new CustomAggregationNode();

        // Mock configuration
        when(config.getData()).thenReturn(createConfig());
        node.init(ctx, config);
    }

    @Test
    public void testMessageProcessing() throws Exception {
        // Arrange
        TbMsg msg = createTestMessage();

        // Act
        node.onMsg(ctx, msg);

        // Assert
        verify(ctx, times(1)).tellNext(any(TbMsg.class), eq("Success"));
    }

    @Test
    public void testErrorHandling() throws Exception {
        // Arrange
        TbMsg invalidMsg = createInvalidMessage();

        // Act
        node.onMsg(ctx, invalidMsg);

        // Assert
        verify(ctx, times(1)).tellFailure(any(TbMsg.class), any(Throwable.class));
    }

    @Test
    public void testConfigurationValidation() {
        // Test that node validates configuration correctly
        assertThrows(TbNodeException.class, () -> {
            node.init(ctx, createInvalidConfig());
        });
    }

    private TbNodeConfiguration createConfig() {
        // Create valid configuration JSON
        String configJson = "{\"aggregationType\":\"AVG\",\"windowSize\":60}";
        return new TbNodeConfiguration(new ObjectMapper().readTree(configJson));
    }

    private TbMsg createTestMessage() {
        return TbMsg.newMsg(
            "POST_TELEMETRY_REQUEST",
            DeviceId.fromString("12345"),
            TbMsgMetaData.EMPTY,
            "{\"temperature\":25.5}"
        );
    }
}
```

#### Key Testing Scenarios

1. **Message Processing**
   - Valid message handling
   - Invalid message handling
   - Edge cases (null values, empty data, malformed JSON)

2. **Configuration**
   - Valid configuration initialization
   - Invalid configuration rejection
   - Configuration changes at runtime

3. **Context Interaction**
   - Correct use of `ctx.tellNext()`
   - Proper error handling with `ctx.tellFailure()`
   - Attribute service calls
   - Telemetry service calls

4. **External Dependencies**
   - Mock external services
   - Test timeout handling
   - Test connection failure scenarios

5. **State Management**
   - Thread safety
   - Resource cleanup (`destroy()` method)
   - Memory leaks

### Integration Testing with Generator Node

ThingsBoard provides a **Generator Node** for testing:

```
┌──────────────┐         ┌─────────────────┐         ┌──────────────┐
│  Generator   │────────▶│  Your Custom    │────────▶│   Debug      │
│  Node        │         │  Rule Node      │         │   Terminal   │
└──────────────┘         └─────────────────┘         └──────────────┘
```

**Setup Steps:**

1. Create test rule chain in ThingsBoard UI
2. Add Generator Node (configure message rate, payload)
3. Add your custom rule node
4. Add Debug Terminal Node (to inspect output)
5. Enable "Debug mode" for your custom node
6. Start generator and observe:
   - Input messages
   - Processing behavior
   - Output messages
   - Error messages
   - Performance metrics

**Generator Node Configuration Example:**
```json
{
  "msgCount": 100,
  "periodInSeconds": 1,
  "originatorType": "DEVICE",
  "originatorId": "test-device-id",
  "msgType": "POST_TELEMETRY_REQUEST",
  "data": "{\"temperature\": ${random(20, 30)}, \"humidity\": ${random(40, 60)}}"
}
```

### Testing Custom Node UI Components

If your rule node has Angular configuration UI:

```typescript
// custom-node-config.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { CustomNodeConfigComponent } from './custom-node-config.component';

describe('CustomNodeConfigComponent', () => {
  let component: CustomNodeConfigComponent;
  let fixture: ComponentFixture<CustomNodeConfigComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [ CustomNodeConfigComponent ],
      imports: [ /* Angular Material modules, etc. */ ]
    }).compileComponents();
  });

  beforeEach(() => {
    fixture = TestBed.createComponent(CustomNodeConfigComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });

  it('should validate configuration', () => {
    component.configuration = { aggregationType: '', windowSize: -1 };
    expect(component.validate()).toBe(false);

    component.configuration = { aggregationType: 'AVG', windowSize: 60 };
    expect(component.validate()).toBe(true);
  });

  it('should emit configuration changes', (done) => {
    component.configurationChange.subscribe(config => {
      expect(config.aggregationType).toBe('SUM');
      done();
    });

    component.onAggregationTypeChange('SUM');
  });
});
```

### Upgrade-Specific Testing

**After upgrading ThingsBoard version:**

1. **Compilation Test**
   ```bash
   mvn clean compile
   # Should complete without errors
   # Check for deprecation warnings
   ```

2. **API Compatibility Test**
   ```bash
   mvn test
   # All existing unit tests should pass
   # Fix any failures due to API changes
   ```

3. **Runtime Loading Test**
   - Deploy custom rule node JAR to upgraded ThingsBoard
   - Verify it appears in Rule Node library
   - Check for ClassNotFoundException or LinkageError in logs

4. **Functional Regression Test**
   - Create test rule chain with custom node
   - Process test messages
   - Verify output matches expected behavior from pre-upgrade

5. **Performance Regression Test**
   - Process same message volume as baseline
   - Compare processing time, memory usage, CPU usage
   - Identify any performance degradation

### Common Upgrade Issues for Custom Rule Nodes

| Issue | Symptom | Solution |
|-------|---------|----------|
| **API signature changed** | Compilation error | Update method signatures to match new API |
| **Deprecated API removed** | NoSuchMethodError at runtime | Migrate to replacement API |
| **Class moved/renamed** | ClassNotFoundException | Update imports, rebuild with new dependencies |
| **Serialization format changed** | Deserialization errors | Update JSON parsing logic |
| **Context API changed** | Runtime errors in ctx calls | Review TbContext API changes, update usage |
| **Angular version mismatch** | UI component doesn't load | Rebuild UI on new Angular version |

---

## Rule Chain Testing

### Rule Chain Complexity Assessment

Categorize your rule chains by complexity:

#### Simple Rule Chain
- Linear flow
- 3-5 nodes
- No external integrations
- **Testing effort:** Low
- **Test approach:** Manual smoke testing

#### Medium Rule Chain
- Some branching logic
- 6-15 nodes
- 1-2 external integrations
- **Testing effort:** Medium
- **Test approach:** Automated integration tests

#### Complex Rule Chain
- Multiple branches, loops
- 15+ nodes
- Multiple external integrations
- Custom rule nodes
- **Testing effort:** High
- **Test approach:** Comprehensive automated testing + manual validation

### Rule Chain Export and Version Control

**Best practice:** Store rule chain definitions in version control

```bash
# Export rule chain via ThingsBoard UI
# Save to: /rule-chains/main-device-telemetry-chain.json

# Commit to Git
git add rule-chains/
git commit -m "Snapshot rule chains before upgrade"
git tag pre-upgrade-rule-chains-$(date +%Y%m%d)
```

### Rule Chain Testing Strategy

#### 1. Static Analysis

Analyze exported rule chain JSON:

```python
import json

def analyze_rule_chain(json_file):
    with open(json_file) as f:
        chain = json.load(f)

    # Extract nodes
    nodes = chain.get('nodes', [])

    # Identify custom nodes
    custom_nodes = [n for n in nodes if 'com.yourcompany' in n.get('type', '')]

    # Identify deprecated nodes (maintain a list of deprecated types)
    deprecated = ['org.thingsboard.rule.engine.OldNode']
    deprecated_nodes = [n for n in nodes if n.get('type') in deprecated]

    # Check for external integrations
    integration_nodes = [n for n in nodes if 'rest' in n.get('type', '').lower()
                         or 'kafka' in n.get('type', '').lower()
                         or 'mqtt' in n.get('type', '').lower()]

    print(f"Total nodes: {len(nodes)}")
    print(f"Custom nodes: {len(custom_nodes)}")
    print(f"Deprecated nodes: {len(deprecated_nodes)}")
    print(f"Integration nodes: {len(integration_nodes)}")

    if deprecated_nodes:
        print("\n⚠️  WARNING: Deprecated nodes found:")
        for node in deprecated_nodes:
            print(f"  - {node.get('name')} ({node.get('type')})")

    return {
        'total': len(nodes),
        'custom': custom_nodes,
        'deprecated': deprecated_nodes,
        'integrations': integration_nodes
    }

# Run analysis
analyze_rule_chain('rule-chains/main-device-telemetry-chain.json')
```

#### 2. Functional Testing

**Test each rule chain path:**

```python
# Test case structure for rule chain testing

test_cases = [
    {
        'name': 'Valid telemetry processing',
        'input': {
            'type': 'POST_TELEMETRY_REQUEST',
            'deviceId': 'test-device-1',
            'data': {'temperature': 25.5, 'humidity': 60}
        },
        'expected_output': {
            'relation': 'Success',
            'processed': True,
            'attributes_saved': ['temperature', 'humidity']
        }
    },
    {
        'name': 'Invalid telemetry - missing fields',
        'input': {
            'type': 'POST_TELEMETRY_REQUEST',
            'deviceId': 'test-device-1',
            'data': {'temperature': 25.5}  # humidity missing
        },
        'expected_output': {
            'relation': 'Failure',
            'error_logged': True
        }
    },
    {
        'name': 'Alarm threshold exceeded',
        'input': {
            'type': 'POST_TELEMETRY_REQUEST',
            'deviceId': 'test-device-1',
            'data': {'temperature': 85.0}  # Over threshold
        },
        'expected_output': {
            'relation': 'Alarm',
            'alarm_created': True,
            'alarm_severity': 'CRITICAL'
        }
    }
]
```

#### 3. Integration Testing with REST API

Use ThingsBoard REST API to test rule chains programmatically:

```python
import requests
import time

class ThingsBoardTester:
    def __init__(self, base_url, username, password):
        self.base_url = base_url
        self.token = self._login(username, password)

    def _login(self, username, password):
        response = requests.post(
            f"{self.base_url}/api/auth/login",
            json={"username": username, "password": password}
        )
        return response.json()['token']

    def send_telemetry(self, device_token, telemetry):
        """Send telemetry to device"""
        response = requests.post(
            f"{self.base_url}/api/v1/{device_token}/telemetry",
            json=telemetry
        )
        return response.status_code == 200

    def get_latest_telemetry(self, device_id, keys):
        """Get latest telemetry for device"""
        headers = {"X-Authorization": f"Bearer {self.token}"}
        response = requests.get(
            f"{self.base_url}/api/plugins/telemetry/DEVICE/{device_id}/values/timeseries",
            params={"keys": ",".join(keys)},
            headers=headers
        )
        return response.json()

    def check_alarm(self, device_id):
        """Check if alarm exists for device"""
        headers = {"X-Authorization": f"Bearer {self.token}"}
        response = requests.get(
            f"{self.base_url}/api/alarm/DEVICE/{device_id}",
            headers=headers
        )
        return response.json()

# Usage
tester = ThingsBoardTester("http://localhost:8080", "tenant@thingsboard.org", "tenant")

# Test 1: Send telemetry
device_token = "YOUR_DEVICE_TOKEN"
tester.send_telemetry(device_token, {"temperature": 25.5, "humidity": 60})

# Wait for rule chain processing
time.sleep(2)

# Verify telemetry was saved
telemetry = tester.get_latest_telemetry("device-id", ["temperature", "humidity"])
assert "temperature" in telemetry
assert telemetry["temperature"][0]["value"] == "25.5"

# Test 2: Trigger alarm
tester.send_telemetry(device_token, {"temperature": 85.0})
time.sleep(2)

# Verify alarm was created
alarms = tester.check_alarm("device-id")
assert len(alarms['data']) > 0
assert alarms['data'][0]['severity'] == 'CRITICAL'

print("✅ All rule chain tests passed")
```

#### 4. Debug Mode Testing

Enable debug mode in ThingsBoard UI:

1. Open Rule Chain editor
2. Click Debug Mode toggle
3. Select specific nodes to debug
4. Send test messages
5. Observe message flow through chain
6. Verify each node's input/output
7. Check for errors or unexpected routing

**What to verify in Debug Mode:**

- [ ] Messages reach all expected nodes
- [ ] Transformations produce correct output
- [ ] Branching logic routes messages correctly
- [ ] Error messages are handled appropriately
- [ ] No messages are lost or stuck
- [ ] Performance is acceptable (check processing time per node)

### Rule Chain Migration Testing

**When upgrading, test:**

1. **Import compatibility**
   - Export rule chain from old version
   - Import to new version
   - Verify all nodes appear correctly
   - Check for "unknown node type" errors

2. **Reference integrity**
   - Rule chains may reference devices, assets, dashboards
   - Verify all references still resolve
   - Check externalId mapping

3. **Deprecated node replacement**
   - If rule chain uses deprecated nodes, replace them
   - Test equivalent functionality with new nodes
   - Validate output matches original behavior

### Automated Rule Chain Testing Framework

```bash
#!/bin/bash
# rule-chain-test.sh - Automated rule chain testing

# Configuration
TB_URL="http://localhost:8080"
TB_USER="tenant@thingsboard.org"
TB_PASS="tenant"
DEVICE_TOKEN="YOUR_DEVICE_TOKEN"

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Test counter
TESTS_RUN=0
TESTS_PASSED=0

# Login and get token
TOKEN=$(curl -s -X POST "$TB_URL/api/auth/login" \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"$TB_USER\",\"password\":\"$TB_PASS\"}" | jq -r '.token')

echo "🔐 Logged in to ThingsBoard"

# Test function
test_rule_chain() {
    local test_name=$1
    local telemetry=$2
    local expected_result=$3

    echo -n "Testing: $test_name... "
    TESTS_RUN=$((TESTS_RUN + 1))

    # Send telemetry
    response=$(curl -s -X POST "$TB_URL/api/v1/$DEVICE_TOKEN/telemetry" \
      -H "Content-Type: application/json" \
      -d "$telemetry")

    # Wait for processing
    sleep 2

    # Check result (this is simplified - you'd implement actual verification)
    # For example, check if alarm was created, attribute was updated, etc.

    if [ "$response" == "200" ]; then
        echo -e "${GREEN}✅ PASSED${NC}"
        TESTS_PASSED=$((TESTS_PASSED + 1))
    else
        echo -e "${RED}❌ FAILED${NC}"
    fi
}

# Run tests
echo "🧪 Starting rule chain tests..."

test_rule_chain "Normal telemetry" \
  '{"temperature": 25.5, "humidity": 60}' \
  "Success"

test_rule_chain "High temperature alarm" \
  '{"temperature": 85.0}' \
  "Alarm created"

test_rule_chain "Invalid data format" \
  '{"temp": "not_a_number"}' \
  "Error handled"

# Summary
echo ""
echo "📊 Test Summary:"
echo "   Tests run: $TESTS_RUN"
echo "   Tests passed: $TESTS_PASSED"
echo "   Tests failed: $((TESTS_RUN - TESTS_PASSED))"

if [ $TESTS_PASSED -eq $TESTS_RUN ]; then
    echo -e "${GREEN}✅ All tests passed!${NC}"
    exit 0
else
    echo -e "${RED}❌ Some tests failed${NC}"
    exit 1
fi
```

---

## Integration Testing

### External System Integration Testing

Many ThingsBoard deployments integrate with external systems. Test each integration point:

#### HTTP/REST Integrations

```python
# Mock external REST API for testing
from flask import Flask, request, jsonify
import json

app = Flask(__name__)

# Track calls for verification
calls_received = []

@app.route('/webhook/telemetry', methods=['POST'])
def receive_telemetry():
    data = request.json
    calls_received.append({
        'timestamp': time.time(),
        'data': data
    })
    return jsonify({'status': 'received'}), 200

@app.route('/test/verify', methods=['GET'])
def verify_calls():
    """Endpoint for test verification"""
    return jsonify({'calls': calls_received}), 200

@app.route('/test/reset', methods=['POST'])
def reset():
    """Reset call tracking"""
    calls_received.clear()
    return jsonify({'status': 'reset'}), 200

if __name__ == '__main__':
    app.run(port=5000)
```

**Test script:**

```python
import requests
import time

# Start mock server (in separate process)
# Then run tests:

def test_http_integration():
    # Reset mock server
    requests.post('http://localhost:5000/test/reset')

    # Send telemetry through ThingsBoard (triggers rule chain → HTTP node → mock server)
    tb = ThingsBoardTester("http://localhost:8080", "tenant@thingsboard.org", "tenant")
    tb.send_telemetry(DEVICE_TOKEN, {"temperature": 25.5})

    # Wait for rule chain processing
    time.sleep(3)

    # Verify mock server received the call
    response = requests.get('http://localhost:5000/test/verify')
    calls = response.json()['calls']

    assert len(calls) == 1, "Expected 1 call to webhook"
    assert calls[0]['data']['temperature'] == 25.5, "Temperature value mismatch"

    print("✅ HTTP integration test passed")
```

#### MQTT Integration Testing

```python
import paho.mqtt.client as mqtt
import json
import time

class MQTTIntegrationTester:
    def __init__(self, broker_host, broker_port=1883):
        self.client = mqtt.Client()
        self.messages_received = []

        self.client.on_connect = self._on_connect
        self.client.on_message = self._on_message

        self.client.connect(broker_host, broker_port, 60)
        self.client.loop_start()

    def _on_connect(self, client, userdata, flags, rc):
        print(f"Connected to MQTT broker with result code {rc}")
        # Subscribe to topic where ThingsBoard publishes
        client.subscribe("thingsboard/output/#")

    def _on_message(self, client, userdata, msg):
        payload = json.loads(msg.payload.decode())
        self.messages_received.append({
            'topic': msg.topic,
            'payload': payload,
            'timestamp': time.time()
        })

    def verify_message_sent(self, expected_topic, expected_data, timeout=10):
        """Verify a message was published to MQTT"""
        start = time.time()
        while time.time() - start < timeout:
            for msg in self.messages_received:
                if msg['topic'] == expected_topic:
                    if all(msg['payload'].get(k) == v for k, v in expected_data.items()):
                        return True
            time.sleep(0.5)
        return False

# Usage
mqtt_tester = MQTTIntegrationTester("localhost", 1883)

# Trigger ThingsBoard rule chain that publishes to MQTT
tb = ThingsBoardTester("http://localhost:8080", "tenant@thingsboard.org", "tenant")
tb.send_telemetry(DEVICE_TOKEN, {"temperature": 85.0, "humidity": 60})

# Verify MQTT message was published
time.sleep(2)
assert mqtt_tester.verify_message_sent(
    "thingsboard/output/alarms",
    {"deviceId": "test-device-1", "alarmType": "HighTemperature"}
), "MQTT alarm message not received"

print("✅ MQTT integration test passed")
```

#### Kafka Integration Testing

```python
from kafka import KafkaConsumer, KafkaProducer
import json
import time

class KafkaIntegrationTester:
    def __init__(self, bootstrap_servers):
        self.consumer = KafkaConsumer(
            'thingsboard.telemetry.output',
            bootstrap_servers=bootstrap_servers,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            auto_offset_reset='latest',
            consumer_timeout_ms=10000
        )

        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )

    def send_to_thingsboard(self, topic, message):
        """Send message to Kafka topic that ThingsBoard consumes"""
        self.producer.send(topic, message)
        self.producer.flush()

    def verify_output(self, expected_messages, timeout=10):
        """Verify ThingsBoard published expected messages to Kafka"""
        messages = []
        start = time.time()

        for message in self.consumer:
            messages.append(message.value)
            if len(messages) >= len(expected_messages):
                break
            if time.time() - start > timeout:
                break

        return messages

# Usage
kafka_tester = KafkaIntegrationTester(['localhost:9092'])

# Send telemetry that triggers rule chain → Kafka output
tb = ThingsBoardTester("http://localhost:8080", "tenant@thingsboard.org", "tenant")
tb.send_telemetry(DEVICE_TOKEN, {"temperature": 25.5})

# Verify Kafka received the processed message
time.sleep(3)
messages = kafka_tester.verify_output([{'temperature': 25.5}], timeout=5)

assert len(messages) > 0, "No messages received on Kafka"
assert messages[0]['temperature'] == 25.5, "Temperature value mismatch"

print("✅ Kafka integration test passed")
```

### Database Integration Testing

Test custom database queries and stored procedures:

```python
import psycopg2
import json

class DatabaseIntegrationTester:
    def __init__(self, host, database, user, password):
        self.conn = psycopg2.connect(
            host=host,
            database=database,
            user=user,
            password=password
        )
        self.cursor = self.conn.cursor()

    def verify_telemetry_saved(self, device_id, key, expected_value):
        """Verify telemetry was saved to database"""
        query = """
        SELECT ts_kv.str_v, ts_kv.long_v, ts_kv.dbl_v
        FROM ts_kv
        JOIN ts_kv_dictionary ON ts_kv.key = ts_kv_dictionary.key_id
        WHERE ts_kv.entity_id = %s
          AND ts_kv_dictionary.key = %s
        ORDER BY ts_kv.ts DESC
        LIMIT 1
        """
        self.cursor.execute(query, (device_id, key))
        result = self.cursor.fetchone()

        if result is None:
            return False

        # Check appropriate column based on value type
        str_v, long_v, dbl_v = result
        if str_v is not None:
            return str_v == str(expected_value)
        elif long_v is not None:
            return long_v == int(expected_value)
        elif dbl_v is not None:
            return abs(dbl_v - float(expected_value)) < 0.001

        return False

    def verify_alarm_created(self, device_id, alarm_type):
        """Verify alarm was created in database"""
        query = """
        SELECT id, severity, status
        FROM alarm
        WHERE originator_id = %s
          AND type = %s
          AND status != 'CLEARED_UNACK'
        ORDER BY created_time DESC
        LIMIT 1
        """
        self.cursor.execute(query, (device_id, alarm_type))
        result = self.cursor.fetchone()
        return result is not None

    def cleanup_test_data(self, device_id):
        """Clean up test data after tests"""
        # Delete test telemetry
        self.cursor.execute("DELETE FROM ts_kv WHERE entity_id = %s", (device_id,))
        # Delete test alarms
        self.cursor.execute("DELETE FROM alarm WHERE originator_id = %s", (device_id,))
        self.conn.commit()

# Usage
db_tester = DatabaseIntegrationTester(
    host="localhost",
    database="thingsboard",
    user="postgres",
    password="password"
)

# Send telemetry
tb = ThingsBoardTester("http://localhost:8080", "tenant@thingsboard.org", "tenant")
tb.send_telemetry(DEVICE_TOKEN, {"temperature": 25.5})

time.sleep(2)

# Verify in database
assert db_tester.verify_telemetry_saved(
    device_id="test-device-uuid",
    key="temperature",
    expected_value=25.5
), "Telemetry not found in database"

print("✅ Database integration test passed")

# Cleanup
db_tester.cleanup_test_data("test-device-uuid")
```

---

## Performance and Load Testing

### Performance Baseline Establishment

**Before upgrade, establish baselines:**

```python
import requests
import time
import statistics
from concurrent.futures import ThreadPoolExecutor, as_completed

class PerformanceTester:
    def __init__(self, tb_url, device_token):
        self.tb_url = tb_url
        self.device_token = device_token
        self.results = []

    def send_single_telemetry(self, payload):
        """Send single telemetry message and measure latency"""
        start = time.time()
        try:
            response = requests.post(
                f"{self.tb_url}/api/v1/{self.device_token}/telemetry",
                json=payload,
                timeout=10
            )
            latency = (time.time() - start) * 1000  # ms
            success = response.status_code == 200
        except Exception as e:
            latency = None
            success = False

        return {
            'success': success,
            'latency_ms': latency,
            'timestamp': time.time()
        }

    def load_test(self, num_messages, concurrent_threads=10):
        """Run load test with multiple concurrent requests"""
        print(f"🚀 Starting load test: {num_messages} messages, {concurrent_threads} threads")

        start_time = time.time()

        with ThreadPoolExecutor(max_workers=concurrent_threads) as executor:
            futures = []
            for i in range(num_messages):
                payload = {
                    "temperature": 20 + (i % 20),
                    "humidity": 40 + (i % 30),
                    "timestamp": int(time.time() * 1000)
                }
                futures.append(executor.submit(self.send_single_telemetry, payload))

            for future in as_completed(futures):
                result = future.result()
                self.results.append(result)

        end_time = time.time()
        duration = end_time - start_time

        return self.analyze_results(duration)

    def analyze_results(self, duration):
        """Analyze performance test results"""
        successful = [r for r in self.results if r['success']]
        failed = [r for r in self.results if not r['success']]
        latencies = [r['latency_ms'] for r in successful]

        if not latencies:
            return {
                'total_messages': len(self.results),
                'successful': 0,
                'failed': len(failed),
                'error': 'All requests failed'
            }

        return {
            'total_messages': len(self.results),
            'successful': len(successful),
            'failed': len(failed),
            'duration_seconds': duration,
            'throughput_msg_per_sec': len(successful) / duration,
            'latency_min_ms': min(latencies),
            'latency_max_ms': max(latencies),
            'latency_avg_ms': statistics.mean(latencies),
            'latency_median_ms': statistics.median(latencies),
            'latency_p95_ms': self.percentile(latencies, 95),
            'latency_p99_ms': self.percentile(latencies, 99)
        }

    @staticmethod
    def percentile(data, p):
        """Calculate percentile"""
        sorted_data = sorted(data)
        index = int(len(sorted_data) * p / 100)
        return sorted_data[index]

    def print_results(self, results):
        """Print formatted results"""
        print("\n📊 Performance Test Results:")
        print("=" * 60)
        print(f"Total Messages:        {results['total_messages']}")
        print(f"Successful:            {results['successful']}")
        print(f"Failed:                {results['failed']}")
        print(f"Duration:              {results['duration_seconds']:.2f}s")
        print(f"Throughput:            {results['throughput_msg_per_sec']:.2f} msg/s")
        print(f"\nLatency Statistics:")
        print(f"  Min:                 {results['latency_min_ms']:.2f}ms")
        print(f"  Max:                 {results['latency_max_ms']:.2f}ms")
        print(f"  Average:             {results['latency_avg_ms']:.2f}ms")
        print(f"  Median:              {results['latency_median_ms']:.2f}ms")
        print(f"  95th percentile:     {results['latency_p95_ms']:.2f}ms")
        print(f"  99th percentile:     {results['latency_p99_ms']:.2f}ms")
        print("=" * 60)

# Run baseline test before upgrade
print("📈 Pre-Upgrade Baseline Test")
pre_tester = PerformanceTester("http://localhost:8080", DEVICE_TOKEN)
pre_results = pre_tester.load_test(num_messages=1000, concurrent_threads=20)
pre_tester.print_results(pre_results)

# Save baseline
import json
with open('performance_baseline_pre_upgrade.json', 'w') as f:
    json.dump(pre_results, f, indent=2)

# After upgrade, run same test
print("\n📈 Post-Upgrade Performance Test")
post_tester = PerformanceTester("http://localhost:8080", DEVICE_TOKEN)
post_results = post_tester.load_test(num_messages=1000, concurrent_threads=20)
post_tester.print_results(post_results)

# Compare results
print("\n📊 Performance Comparison:")
print(f"Throughput change: {((post_results['throughput_msg_per_sec'] / pre_results['throughput_msg_per_sec']) - 1) * 100:+.2f}%")
print(f"Avg latency change: {((post_results['latency_avg_ms'] / pre_results['latency_avg_ms']) - 1) * 100:+.2f}%")
print(f"P95 latency change: {((post_results['latency_p95_ms'] / pre_results['latency_p95_ms']) - 1) * 100:+.2f}%")

# Alert if degradation exceeds threshold
DEGRADATION_THRESHOLD = 20  # 20% degradation is concerning
latency_degradation = ((post_results['latency_avg_ms'] / pre_results['latency_avg_ms']) - 1) * 100

if latency_degradation > DEGRADATION_THRESHOLD:
    print(f"\n⚠️  WARNING: Latency degradation of {latency_degradation:.2f}% exceeds threshold!")
else:
    print(f"\n✅ Performance within acceptable range")
```

### Resource Monitoring During Tests

```bash
#!/bin/bash
# monitor-resources.sh - Monitor system resources during testing

# Configuration
INTERVAL=5  # seconds
OUTPUT_FILE="resource_usage_$(date +%Y%m%d_%H%M%S).csv"

# Header
echo "timestamp,cpu_percent,memory_mb,disk_io_read_mb,disk_io_write_mb,network_rx_mb,network_tx_mb" > "$OUTPUT_FILE"

# Get initial values for delta calculations
PREV_DISK_READ=$(cat /proc/diskstats | awk '{sum+=$6} END {print sum}')
PREV_DISK_WRITE=$(cat /proc/diskstats | awk '{sum+=$10} END {print sum}')
PREV_NET_RX=$(cat /proc/net/dev | tail -n +3 | awk '{sum+=$2} END {print sum}')
PREV_NET_TX=$(cat /proc/net/dev | tail -n +3 | awk '{sum+=$10} END {print sum}')

echo "📊 Monitoring system resources... (Ctrl+C to stop)"
echo "Output: $OUTPUT_FILE"

while true; do
    # Timestamp
    TS=$(date +%s)

    # CPU usage (overall)
    CPU=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)

    # Memory usage (MB)
    MEM=$(free -m | awk 'NR==2{print $3}')

    # Disk I/O (MB)
    CURR_DISK_READ=$(cat /proc/diskstats | awk '{sum+=$6} END {print sum}')
    CURR_DISK_WRITE=$(cat /proc/diskstats | awk '{sum+=$10} END {print sum}')
    DISK_READ=$(echo "scale=2; ($CURR_DISK_READ - $PREV_DISK_READ) * 512 / 1024 / 1024 / $INTERVAL" | bc)
    DISK_WRITE=$(echo "scale=2; ($CURR_DISK_WRITE - $PREV_DISK_WRITE) * 512 / 1024 / 1024 / $INTERVAL" | bc)
    PREV_DISK_READ=$CURR_DISK_READ
    PREV_DISK_WRITE=$CURR_DISK_WRITE

    # Network I/O (MB)
    CURR_NET_RX=$(cat /proc/net/dev | tail -n +3 | awk '{sum+=$2} END {print sum}')
    CURR_NET_TX=$(cat /proc/net/dev | tail -n +3 | awk '{sum+=$10} END {print sum}')
    NET_RX=$(echo "scale=2; ($CURR_NET_RX - $PREV_NET_RX) / 1024 / 1024 / $INTERVAL" | bc)
    NET_TX=$(echo "scale=2; ($CURR_NET_TX - $PREV_NET_TX) / 1024 / 1024 / $INTERVAL" | bc)
    PREV_NET_RX=$CURR_NET_RX
    PREV_NET_TX=$CURR_NET_TX

    # Write to CSV
    echo "$TS,$CPU,$MEM,$DISK_READ,$DISK_WRITE,$NET_RX,$NET_TX" >> "$OUTPUT_FILE"

    # Display
    echo "$(date +'%H:%M:%S') | CPU: ${CPU}% | Mem: ${MEM}MB | Disk R/W: ${DISK_READ}/${DISK_WRITE} MB/s | Net R/W: ${NET_RX}/${NET_TX} MB/s"

    sleep $INTERVAL
done
```

### Rule Engine Performance Testing

Test rule engine processing performance:

```python
import time
import psycopg2

class RuleEnginePerformanceTester:
    def __init__(self, db_config):
        self.conn = psycopg2.connect(**db_config)
        self.cursor = self.conn.cursor()

    def get_rule_engine_stats(self):
        """Get rule engine processing statistics from database"""
        query = """
        SELECT
            COUNT(*) as total_messages,
            AVG(EXTRACT(EPOCH FROM (updated_time - created_time))) * 1000 as avg_processing_time_ms
        FROM rule_node_debug_event
        WHERE created_time > NOW() - INTERVAL '1 hour'
        """
        self.cursor.execute(query)
        result = self.cursor.fetchone()
        return {
            'total_messages': result[0],
            'avg_processing_time_ms': result[1]
        }

    def get_failed_messages(self):
        """Get count of failed messages"""
        query = """
        SELECT COUNT(*)
        FROM event
        WHERE event_type = 'ERROR'
          AND created_time > NOW() - INTERVAL '1 hour'
        """
        self.cursor.execute(query)
        return self.cursor.fetchone()[0]

    def get_queue_sizes(self):
        """Get current queue sizes (if using queue-based processing)"""
        # This depends on your ThingsBoard configuration
        # Example for checking queue stats
        stats_query = """
        SELECT
            queue_name,
            msg_count,
            avg_processing_time
        FROM tb_queue_stats
        WHERE updated_time > NOW() - INTERVAL '5 minutes'
        """
        self.cursor.execute(stats_query)
        return self.cursor.fetchall()

# Usage
db_config = {
    'host': 'localhost',
    'database': 'thingsboard',
    'user': 'postgres',
    'password': 'password'
}

re_tester = RuleEnginePerformanceTester(db_config)

# Before load test
print("📊 Rule Engine Stats Before Load Test:")
before_stats = re_tester.get_rule_engine_stats()
print(f"  Total messages processed: {before_stats['total_messages']}")
print(f"  Avg processing time: {before_stats['avg_processing_time_ms']:.2f}ms")

# Run load test (using PerformanceTester from above)
# ... load test code ...

# After load test
time.sleep(10)  # Wait for processing to complete
print("\n📊 Rule Engine Stats After Load Test:")
after_stats = re_tester.get_rule_engine_stats()
print(f"  Total messages processed: {after_stats['total_messages']}")
print(f"  Avg processing time: {after_stats['avg_processing_time_ms']:.2f}ms")

failed = re_tester.get_failed_messages()
print(f"  Failed messages: {failed}")

if failed > 0:
    print(f"\n⚠️  WARNING: {failed} messages failed processing")
```

---

## Rollback and Recovery Planning

### Pre-Upgrade Snapshot

Create complete system snapshot before upgrade:

```bash
#!/bin/bash
# create-upgrade-snapshot.sh

BACKUP_DIR="/backup/thingsboard-upgrade-$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

echo "📦 Creating ThingsBoard upgrade snapshot..."
echo "Backup directory: $BACKUP_DIR"

# 1. Stop ThingsBoard
echo "⏸  Stopping ThingsBoard service..."
sudo systemctl stop thingsboard

# 2. PostgreSQL backup
echo "💾 Backing up PostgreSQL database..."
sudo -u postgres pg_dump thingsboard | gzip > "$BACKUP_DIR/thingsboard_db.sql.gz"

# 3. Configuration files
echo "⚙️  Backing up configuration files..."
sudo tar -czf "$BACKUP_DIR/config.tar.gz" \
    /etc/thingsboard/ \
    /usr/share/thingsboard/conf/

# 4. Custom code / plugins
echo "🔧 Backing up custom code..."
if [ -d "/usr/share/thingsboard/extensions" ]; then
    sudo tar -czf "$BACKUP_DIR/extensions.tar.gz" \
        /usr/share/thingsboard/extensions/
fi

# 5. Logs (for reference)
echo "📝 Backing up logs..."
sudo tar -czf "$BACKUP_DIR/logs.tar.gz" \
    /var/log/thingsboard/

# 6. Redis snapshot (if using Redis)
echo "🗄  Backing up Redis..."
if systemctl is-active --quiet redis; then
    sudo redis-cli BGSAVE
    sleep 5
    sudo cp /var/lib/redis/dump.rdb "$BACKUP_DIR/redis_dump.rdb"
fi

# 7. System information
echo "ℹ️  Saving system information..."
cat > "$BACKUP_DIR/system_info.txt" <<EOF
ThingsBoard Upgrade Snapshot
Created: $(date)
Hostname: $(hostname)
OS: $(cat /etc/os-release | grep PRETTY_NAME | cut -d'"' -f2)
ThingsBoard Version: $(cat /usr/share/thingsboard/conf/thingsboard.yml | grep -A1 "application:" | grep "version:" | awk '{print $2}')
Java Version: $(java -version 2>&1 | head -n 1)
PostgreSQL Version: $(sudo -u postgres psql --version)
Redis Version: $(redis-server --version)
Disk Usage:
$(df -h)
Memory:
$(free -h)
EOF

# 8. Create restore script
echo "📜 Creating restore script..."
cat > "$BACKUP_DIR/restore.sh" <<'EOF'
#!/bin/bash
# Auto-generated restore script

BACKUP_DIR="$(dirname "$0")"

echo "🔄 Restoring ThingsBoard from backup..."
echo "⚠️  WARNING: This will overwrite current ThingsBoard installation!"
read -p "Are you sure you want to continue? (yes/no): " confirm

if [ "$confirm" != "yes" ]; then
    echo "Restore cancelled"
    exit 1
fi

# Stop ThingsBoard
echo "⏸  Stopping ThingsBoard..."
sudo systemctl stop thingsboard

# Restore database
echo "💾 Restoring database..."
sudo -u postgres dropdb thingsboard
sudo -u postgres createdb thingsboard
gunzip -c "$BACKUP_DIR/thingsboard_db.sql.gz" | sudo -u postgres psql thingsboard

# Restore configuration
echo "⚙️  Restoring configuration..."
sudo tar -xzf "$BACKUP_DIR/config.tar.gz" -C /

# Restore extensions
if [ -f "$BACKUP_DIR/extensions.tar.gz" ]; then
    echo "🔧 Restoring extensions..."
    sudo tar -xzf "$BACKUP_DIR/extensions.tar.gz" -C /
fi

# Restore Redis
if [ -f "$BACKUP_DIR/redis_dump.rdb" ]; then
    echo "🗄  Restoring Redis..."
    sudo systemctl stop redis
    sudo cp "$BACKUP_DIR/redis_dump.rdb" /var/lib/redis/dump.rdb
    sudo chown redis:redis /var/lib/redis/dump.rdb
    sudo systemctl start redis
fi

echo "✅ Restore complete!"
echo "Starting ThingsBoard..."
sudo systemctl start thingsboard

echo "Checking status..."
sleep 5
sudo systemctl status thingsboard
EOF

chmod +x "$BACKUP_DIR/restore.sh"

# 9. Verify backup
echo "✅ Verifying backup..."
BACKUP_SIZE=$(du -sh "$BACKUP_DIR" | cut -f1)
echo "Backup size: $BACKUP_SIZE"

# List files
echo "Backup contents:"
ls -lh "$BACKUP_DIR"

# Start ThingsBoard again (we stopped it for consistent backup)
echo "▶️  Starting ThingsBoard service..."
sudo systemctl start thingsboard

echo ""
echo "✅ Backup complete!"
echo "📁 Backup location: $BACKUP_DIR"
echo "📜 To restore: $BACKUP_DIR/restore.sh"
echo ""
echo "⚠️  IMPORTANT: Test the restore procedure in a dev environment before relying on it!"
```

### Rollback Decision Matrix

| Scenario | Severity | Action | Rollback? |
|----------|----------|--------|-----------|
| **Custom rule node fails to load** | HIGH | Check logs, fix ClassLoader issues | Maybe - can hot-fix |
| **Database migration fails** | CRITICAL | DO NOT PROCEED | YES - immediate rollback |
| **Performance degradation >50%** | HIGH | Investigate, check query plans | YES - unless quick fix identified |
| **Data loss detected** | CRITICAL | STOP IMMEDIATELY | YES - restore from backup |
| **External integrations broken** | MEDIUM | Fix integration configs | NO - fix forward |
| **UI widgets not rendering** | MEDIUM | Rebuild widgets on new Angular | NO - fix forward |
| **Alarms not triggering** | HIGH | Check rule chains, test | Maybe - depends on criticality |
| **Some devices cannot connect** | HIGH | Check protocol compatibility | Maybe - isolate affected devices |
| **Memory leak detected** | HIGH | Monitor, check for resource cleanup | YES - if OOM risk |

### Rollback Procedure

```bash
#!/bin/bash
# rollback-upgrade.sh - Rollback failed upgrade

BACKUP_DIR="/backup/thingsboard-upgrade-YYYYMMDD"  # Update with actual backup dir

echo "🔙 Starting ThingsBoard rollback..."
echo "⚠️  WARNING: This will restore ThingsBoard to pre-upgrade state"
read -p "Backup directory: " BACKUP_DIR

if [ ! -d "$BACKUP_DIR" ]; then
    echo "❌ Backup directory not found: $BACKUP_DIR"
    exit 1
fi

# Run the restore script from backup
if [ -f "$BACKUP_DIR/restore.sh" ]; then
    "$BACKUP_DIR/restore.sh"
else
    echo "❌ Restore script not found in backup"
    exit 1
fi

# Post-rollback validation
echo ""
echo "🔍 Post-rollback validation..."

# Check service status
if systemctl is-active --quiet thingsboard; then
    echo "✅ ThingsBoard service is running"
else
    echo "❌ ThingsBoard service is not running"
    sudo journalctl -u thingsboard -n 50
    exit 1
fi

# Check database connectivity
sudo -u postgres psql -d thingsboard -c "SELECT COUNT(*) FROM device;" > /dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "✅ Database is accessible"
else
    echo "❌ Database is not accessible"
    exit 1
fi

# Check API endpoint
sleep 10  # Wait for startup
curl -f http://localhost:8080/api/noauth/serverTime > /dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "✅ API is responding"
else
    echo "❌ API is not responding"
    exit 1
fi

echo ""
echo "✅ Rollback complete and validated"
echo "📊 Next steps:"
echo "   1. Verify device connectivity"
echo "   2. Check rule chains are processing"
echo "   3. Review what caused upgrade failure"
echo "   4. Plan remediation before re-attempting upgrade"
```

---

## Post-Upgrade Validation

### Comprehensive Smoke Test Suite

```python
# smoke_test.py - Post-upgrade smoke tests

import requests
import time
import sys

class SmokeTest:
    def __init__(self, base_url, credentials):
        self.base_url = base_url
        self.credentials = credentials
        self.token = None
        self.test_results = []

    def run_all_tests(self):
        """Run all smoke tests"""
        tests = [
            self.test_login,
            self.test_api_endpoints,
            self.test_device_telemetry,
            self.test_rule_chains,
            self.test_alarms,
            self.test_dashboards,
            self.test_widgets,
            self.test_integrations
        ]

        print("🔥 Starting Post-Upgrade Smoke Tests\n")

        for test in tests:
            try:
                result = test()
                self.test_results.append(result)
            except Exception as e:
                self.test_results.append({
                    'test': test.__name__,
                    'status': 'FAILED',
                    'error': str(e)
                })

        self.print_summary()

    def test_login(self):
        """Test authentication"""
        print("🔐 Testing login...")
        response = requests.post(
            f"{self.base_url}/api/auth/login",
            json=self.credentials
        )

        if response.status_code == 200:
            self.token = response.json()['token']
            print("   ✅ Login successful\n")
            return {'test': 'login', 'status': 'PASSED'}
        else:
            print(f"   ❌ Login failed: {response.status_code}\n")
            return {'test': 'login', 'status': 'FAILED', 'error': response.text}

    def test_api_endpoints(self):
        """Test critical API endpoints"""
        print("🌐 Testing API endpoints...")

        endpoints = [
            '/api/user',
            '/api/tenant',
            '/api/devices',
            '/api/ruleChains',
            '/api/alarms'
        ]

        headers = {"X-Authorization": f"Bearer {self.token}"}

        for endpoint in endpoints:
            response = requests.get(f"{self.base_url}{endpoint}", headers=headers)
            if response.status_code != 200:
                print(f"   ❌ {endpoint} failed: {response.status_code}")
                return {'test': 'api_endpoints', 'status': 'FAILED', 'endpoint': endpoint}
            print(f"   ✅ {endpoint}")

        print()
        return {'test': 'api_endpoints', 'status': 'PASSED'}

    def test_device_telemetry(self):
        """Test device telemetry ingestion"""
        print("📊 Testing device telemetry...")

        # This assumes you have a test device token
        device_token = "TEST_DEVICE_TOKEN"

        telemetry = {
            "temperature": 25.5,
            "humidity": 60,
            "test_timestamp": int(time.time())
        }

        response = requests.post(
            f"{self.base_url}/api/v1/{device_token}/telemetry",
            json=telemetry
        )

        if response.status_code == 200:
            print("   ✅ Telemetry ingestion working\n")
            return {'test': 'device_telemetry', 'status': 'PASSED'}
        else:
            print(f"   ❌ Telemetry failed: {response.status_code}\n")
            return {'test': 'device_telemetry', 'status': 'FAILED'}

    def test_rule_chains(self):
        """Test that rule chains are active"""
        print("⚙️  Testing rule chains...")

        headers = {"X-Authorization": f"Bearer {self.token}"}
        response = requests.get(f"{self.base_url}/api/ruleChains", headers=headers)

        if response.status_code == 200:
            chains = response.json().get('data', [])
            active_chains = [c for c in chains if not c.get('debugMode', False)]

            print(f"   ✅ Found {len(active_chains)} active rule chains")

            # Check if any custom rule nodes exist
            # (This is simplified - you'd check specific chains)
            print()
            return {'test': 'rule_chains', 'status': 'PASSED', 'count': len(active_chains)}
        else:
            print(f"   ❌ Failed to fetch rule chains\n")
            return {'test': 'rule_chains', 'status': 'FAILED'}

    def test_alarms(self):
        """Test alarm system"""
        print("🚨 Testing alarms...")

        headers = {"X-Authorization": f"Bearer {self.token}"}

        # Get recent alarms
        response = requests.get(
            f"{self.base_url}/api/alarms",
            headers=headers,
            params={'pageSize': 10, 'page': 0}
        )

        if response.status_code == 200:
            print("   ✅ Alarm API accessible\n")
            return {'test': 'alarms', 'status': 'PASSED'}
        else:
            print(f"   ❌ Alarm API failed: {response.status_code}\n")
            return {'test': 'alarms', 'status': 'FAILED'}

    def test_dashboards(self):
        """Test dashboards load"""
        print("📈 Testing dashboards...")

        headers = {"X-Authorization": f"Bearer {self.token}"}
        response = requests.get(f"{self.base_url}/api/tenant/dashboards", headers=headers)

        if response.status_code == 200:
            dashboards = response.json().get('data', [])
            print(f"   ✅ Found {len(dashboards)} dashboards\n")
            return {'test': 'dashboards', 'status': 'PASSED', 'count': len(dashboards)}
        else:
            print(f"   ❌ Dashboard API failed\n")
            return {'test': 'dashboards', 'status': 'FAILED'}

    def test_widgets(self):
        """Test custom widgets"""
        print("🎨 Testing widgets...")

        headers = {"X-Authorization": f"Bearer {self.token}"}
        response = requests.get(f"{self.base_url}/api/widgetsBundles", headers=headers)

        if response.status_code == 200:
            bundles = response.json()
            print(f"   ✅ Found {len(bundles)} widget bundles\n")
            return {'test': 'widgets', 'status': 'PASSED'}
        else:
            print(f"   ❌ Widget API failed\n")
            return {'test': 'widgets', 'status': 'FAILED'}

    def test_integrations(self):
        """Test integrations are configured"""
        print("🔗 Testing integrations...")

        headers = {"X-Authorization": f"Bearer {self.token}"}
        response = requests.get(f"{self.base_url}/api/integrations", headers=headers)

        if response.status_code == 200:
            integrations = response.json().get('data', [])
            print(f"   ✅ Found {len(integrations)} integrations\n")
            return {'test': 'integrations', 'status': 'PASSED', 'count': len(integrations)}
        else:
            print(f"   ❌ Integration API failed\n")
            return {'test': 'integrations', 'status': 'FAILED'}

    def print_summary(self):
        """Print test summary"""
        print("\n" + "="*60)
        print("SMOKE TEST SUMMARY")
        print("="*60)

        passed = sum(1 for r in self.test_results if r['status'] == 'PASSED')
        failed = sum(1 for r in self.test_results if r['status'] == 'FAILED')

        for result in self.test_results:
            status_icon = "✅" if result['status'] == 'PASSED' else "❌"
            print(f"{status_icon} {result['test']}: {result['status']}")
            if 'error' in result:
                print(f"   Error: {result['error']}")

        print("="*60)
        print(f"Total: {len(self.test_results)} | Passed: {passed} | Failed: {failed}")
        print("="*60)

        if failed > 0:
            print("\n⚠️  SOME TESTS FAILED - Review before production use")
            sys.exit(1)
        else:
            print("\n✅ ALL SMOKE TESTS PASSED")
            sys.exit(0)

# Run smoke tests
if __name__ == "__main__":
    tester = SmokeTest(
        base_url="http://localhost:8080",
        credentials={
            "username": "tenant@thingsboard.org",
            "password": "tenant"
        }
    )

    tester.run_all_tests()
```

### Manual Validation Checklist

```markdown
# Post-Upgrade Manual Validation Checklist

## System Health
- [ ] ThingsBoard service is running (`systemctl status thingsboard`)
- [ ] No errors in application logs (`/var/log/thingsboard/thingsboard.log`)
- [ ] Database queries are responding normally
- [ ] Redis cache is functioning (if used)
- [ ] Disk space is sufficient
- [ ] Memory usage is normal
- [ ] CPU usage is normal

## Authentication & Authorization
- [ ] Can log in as System Administrator
- [ ] Can log in as Tenant Administrator
- [ ] Can log in as Customer User
- [ ] User permissions are correct
- [ ] OAuth/LDAP integration works (if configured)

## Device Connectivity
- [ ] Sample device can connect via MQTT
- [ ] Sample device can connect via HTTP
- [ ] Sample device can connect via CoAP
- [ ] Device credentials are valid
- [ ] Device provisioning works

## Data Ingestion
- [ ] Telemetry data is being received
- [ ] Telemetry is saved to database
- [ ] Latest values are displayed correctly
- [ ] Historical data is intact
- [ ] Attributes can be updated

## Rule Engine
- [ ] All rule chains are listed
- [ ] Custom rule nodes appear in library
- [ ] Rule chains are processing messages
- [ ] Debug mode works
- [ ] No errors in rule node execution
- [ ] External integrations are working

## Alarms
- [ ] Can create alarms manually
- [ ] Rule engine generates alarms correctly
- [ ] Alarm notifications are sent
- [ ] Alarm history is preserved
- [ ] Can acknowledge/clear alarms

## Dashboards & Widgets
- [ ] All dashboards load correctly
- [ ] Custom widgets render properly
- [ ] Real-time updates work
- [ ] Data visualization is accurate
- [ ] Interactive controls function

## Integrations
- [ ] HTTP integrations work
- [ ] MQTT integrations work
- [ ] Kafka integrations work (if used)
- [ ] Webhook callbacks are received
- [ ] External API calls succeed

## Custom Components
- [ ] All custom rule nodes are functional
- [ ] Custom widgets display correctly
- [ ] Custom plugins are loaded
- [ ] Custom integrations work
- [ ] No ClassNotFoundExceptions in logs

## Performance
- [ ] API response times are acceptable
- [ ] Dashboard load times are reasonable
- [ ] Rule chain processing is not delayed
- [ ] No memory leaks detected
- [ ] Database queries are optimized

## Data Integrity
- [ ] Device count matches pre-upgrade
- [ ] Asset count matches pre-upgrade
- [ ] Dashboard count matches pre-upgrade
- [ ] Rule chain count matches pre-upgrade
- [ ] Historical telemetry is intact
- [ ] No data loss detected
```

---

## Automation and CI/CD

### GitHub Actions Workflow for Testing

```yaml
# .github/workflows/thingsboard-upgrade-test.yml

name: ThingsBoard Upgrade Testing

on:
  workflow_dispatch:
    inputs:
      target_version:
        description: 'Target ThingsBoard version'
        required: true
        default: '3.7.0'
  push:
    branches:
      - main
      - develop
    paths:
      - 'custom-rule-nodes/**'
      - 'custom-widgets/**'

jobs:
  compile-custom-nodes:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Compile custom rule nodes
        working-directory: ./custom-rule-nodes
        run: |
          mvn clean compile
          mvn test

      - name: Build JAR
        working-directory: ./custom-rule-nodes
        run: mvn package

      - name: Upload artifacts
        uses: actions/upload-artifact@v3
        with:
          name: custom-rule-nodes
          path: custom-rule-nodes/target/*.jar

  test-custom-widgets:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        working-directory: ./custom-widgets
        run: npm ci

      - name: Run linter
        working-directory: ./custom-widgets
        run: npm run lint

      - name: Run tests
        working-directory: ./custom-widgets
        run: npm test

      - name: Build widgets
        working-directory: ./custom-widgets
        run: npm run build

  integration-test:
    runs-on: ubuntu-latest
    needs: [compile-custom-nodes, test-custom-widgets]

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: thingsboard
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Download ThingsBoard
        run: |
          wget https://github.com/thingsboard/thingsboard/releases/download/v${{ github.event.inputs.target_version }}/thingsboard-${{ github.event.inputs.target_version }}.deb

      - name: Install ThingsBoard
        run: |
          sudo dpkg -i thingsboard-${{ github.event.inputs.target_version }}.deb

      - name: Configure ThingsBoard
        run: |
          sudo sed -i 's/localhost/postgres/g' /etc/thingsboard/conf/thingsboard.conf

      - name: Download custom rule nodes
        uses: actions/download-artifact@v3
        with:
          name: custom-rule-nodes
          path: /tmp/custom-nodes

      - name: Install custom rule nodes
        run: |
          sudo cp /tmp/custom-nodes/*.jar /usr/share/thingsboard/extensions/

      - name: Start ThingsBoard
        run: |
          sudo systemctl start thingsboard
          # Wait for startup
          timeout 120 bash -c 'until curl -f http://localhost:8080/api/noauth/serverTime; do sleep 5; done'

      - name: Run integration tests
        run: |
          python3 tests/integration_test.py

      - name: Check logs for errors
        if: failure()
        run: |
          sudo cat /var/log/thingsboard/thingsboard.log | grep ERROR

  performance-test:
    runs-on: ubuntu-latest
    needs: integration-test

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Run performance tests
        run: |
          python3 tests/performance_test.py

      - name: Upload performance results
        uses: actions/upload-artifact@v3
        with:
          name: performance-results
          path: performance_*.json
```

### Pre-Commit Hooks for Quality Checks

```bash
# .git/hooks/pre-commit

#!/bin/bash
# Pre-commit hook for ThingsBoard custom code

echo "🔍 Running pre-commit checks..."

# Check Java files
JAVA_FILES=$(git diff --cached --name-only --diff-filter=ACM | grep '\.java$')

if [ -n "$JAVA_FILES" ]; then
    echo "📝 Checking Java files..."

    # Run checkstyle
    mvn checkstyle:check
    if [ $? -ne 0 ]; then
        echo "❌ Checkstyle failed"
        exit 1
    fi

    # Ensure tests pass
    mvn test
    if [ $? -ne 0 ]; then
        echo "❌ Tests failed"
        exit 1
    fi
fi

# Check TypeScript files
TS_FILES=$(git diff --cached --name-only --diff-filter=ACM | grep '\.ts$')

if [ -n "$TS_FILES" ]; then
    echo "📝 Checking TypeScript files..."

    # Run linter
    npm run lint
    if [ $? -ne 0 ]; then
        echo "❌ Linter failed"
        exit 1
    fi

    # Run tests
    npm test
    if [ $? -ne 0 ]; then
        echo "❌ Tests failed"
        exit 1
    fi
fi

echo "✅ Pre-commit checks passed"
exit 0
```

---

## Appendix: Checklists and Templates

### Pre-Upgrade Checklist

```markdown
# ThingsBoard Upgrade Pre-Flight Checklist

## Planning Phase
- [ ] Identify target ThingsBoard version
- [ ] Review release notes for target version and all intermediate versions
- [ ] Document all breaking changes
- [ ] Create inventory of custom code
- [ ] Identify deprecated APIs in use
- [ ] Plan upgrade path (sequential versions)
- [ ] Schedule maintenance window
- [ ] Notify stakeholders

## Environment Preparation
- [ ] Set up dev/test environment
- [ ] Set up staging environment
- [ ] Verify environments match production
- [ ] Install required dependencies (Java 17, etc.)
- [ ] Configure monitoring tools

## Backup Preparation
- [ ] Create PostgreSQL database backup
- [ ] Export all rule chains
- [ ] Export all dashboards
- [ ] Export device/asset configurations
- [ ] Backup configuration files
- [ ] Backup custom code repositories
- [ ] Create Redis snapshot (if applicable)
- [ ] Verify backup integrity
- [ ] Test restore procedure in dev

## Testing Preparation
- [ ] Create test device accounts
- [ ] Prepare test data sets
- [ ] Document baseline performance metrics
- [ ] Create automated test scripts
- [ ] Set up monitoring during tests
- [ ] Prepare rollback procedure

## Custom Code Audit
- [ ] List all custom rule nodes
- [ ] List all custom widgets
- [ ] List all custom integrations
- [ ] List all custom plugins
- [ ] Document API dependencies
- [ ] Check compatibility with target version
- [ ] Update unit tests

## Sign-off
- [ ] Technical lead approval
- [ ] Stakeholder approval
- [ ] Maintenance window confirmed
- [ ] Communication plan ready
- [ ] Rollback plan documented
```

### Upgrade Execution Checklist

```markdown
# ThingsBoard Upgrade Execution Checklist

## Pre-Upgrade (T-1 hour)
- [ ] Final backup completed
- [ ] Backup verified
- [ ] Rollback procedure tested
- [ ] Monitoring dashboards ready
- [ ] Team assembled
- [ ] Communication sent to users

## Upgrade Execution (T-0)
- [ ] Stop ThingsBoard service
- [ ] Verify service stopped
- [ ] Flush Redis cache (if applicable)
- [ ] Run upgrade script
- [ ] Monitor upgrade logs
- [ ] Check for errors
- [ ] Verify database schema updated

## Custom Code Deployment
- [ ] Deploy updated custom rule nodes
- [ ] Deploy updated custom widgets
- [ ] Deploy updated integrations
- [ ] Verify files in correct locations
- [ ] Check file permissions

## Service Startup
- [ ] Start ThingsBoard service
- [ ] Monitor startup logs
- [ ] Wait for full initialization
- [ ] Check service status
- [ ] Verify no errors in logs

## Smoke Testing (T+10 min)
- [ ] Login test
- [ ] API endpoint test
- [ ] Device telemetry test
- [ ] Rule chain test
- [ ] Dashboard load test
- [ ] Widget rendering test
- [ ] Integration test

## Validation (T+30 min)
- [ ] Run automated test suite
- [ ] Check custom rule nodes loaded
- [ ] Verify rule chains processing
- [ ] Test alarm generation
- [ ] Verify data persistence
- [ ] Check external integrations

## Performance Validation (T+1 hour)
- [ ] Run performance tests
- [ ] Compare with baseline
- [ ] Check resource usage
- [ ] Monitor for memory leaks
- [ ] Verify no degradation

## Go/No-Go Decision
- [ ] All smoke tests passed
- [ ] No critical errors
- [ ] Performance acceptable
- [ ] Data integrity confirmed
- [ ] **Decision:** GO / NO-GO (rollback)

## Post-Upgrade (if GO)
- [ ] Enable monitoring alerts
- [ ] Update documentation
- [ ] Notify users of completion
- [ ] Schedule follow-up review
- [ ] Document lessons learned

## Rollback (if NO-GO)
- [ ] Execute rollback procedure
- [ ] Verify rollback successful
- [ ] Document failure reasons
- [ ] Schedule post-mortem
- [ ] Plan remediation
```

### Issue Tracking Template

```markdown
# Upgrade Issue: [Brief Description]

## Issue Details
- **Severity:** Critical / High / Medium / Low
- **Category:** Custom Rule Node / Widget / Integration / Database / Performance
- **Detected:** [Date/Time]
- **Environment:** Dev / Test / Staging / Production

## Description
[Detailed description of the issue]

## Steps to Reproduce
1.
2.
3.

## Expected Behavior
[What should happen]

## Actual Behavior
[What actually happens]

## Error Messages / Logs
```
[Paste relevant error messages]
```

## Impact
- **Users Affected:** [Number/Percentage]
- **Functionality Lost:** [Description]
- **Workaround Available:** Yes / No
- **Workaround:** [If yes, describe]

## Root Cause
[Analysis of why this occurred]

## Resolution
- **Action Taken:** [What was done to fix]
- **Code Changes:** [Link to commits/PRs]
- **Testing:** [How fix was validated]
- **Deployed:** [Date/Time]

## Prevention
[How to prevent this in future upgrades]

## Related Issues
- #[issue number]
```

---

## Final Recommendations

### Critical Success Factors

1. **Never rush an upgrade** - Allocate sufficient time for testing
2. **Test everything twice** - Dev environment, then staging
3. **Automate relentlessly** - Manual testing is error-prone
4. **Monitor continuously** - Before, during, and after upgrade
5. **Document thoroughly** - Future you will thank present you
6. **Plan for failure** - Rollback should be tested and ready
7. **Communicate clearly** - Keep stakeholders informed
8. **Validate data integrity** - Data loss is unacceptable

### Red Flags to Watch For

🚩 **Database migration takes longer than expected** - May indicate data corruption
🚩 **Custom rule nodes fail to load** - Likely API compatibility issue
🚩 **Performance degradation >20%** - Something changed significantly
🚩 **Increasing error rate in logs** - System instability
🚩 **Memory usage growing continuously** - Memory leak
🚩 **Devices disconnecting** - Protocol or authentication issue
🚩 **Rule chains stuck in "processing"** - Deadlock or infinite loop

### When to Seek Help

- Upgrade fails repeatedly in dev environment
- Unclear error messages in logs
- Performance issues without obvious cause
- Data integrity concerns
- Custom code compatibility issues

**Resources:**
- ThingsBoard Community Forum: https://groups.google.com/forum/#!forum/thingsboard
- ThingsBoard GitHub Issues: https://github.com/thingsboard/thingsboard/issues
- ThingsBoard Documentation: https://thingsboard.io/docs/

---

## Document Change Log

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2025-11-18 | Initial comprehensive testing strategy | System |

---

**END OF DOCUMENT**
