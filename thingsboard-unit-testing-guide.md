# Unit Testing ThingsBoard Code Snippets

## Overview

ThingsBoard uses two scripting languages in rule nodes and integrations:
- **TBEL (ThingsBoard Expression Language)** - Java-like syntax, super fast (recommended)
- **JavaScript** - Traditional JS, uses Nashorn/GraalVM (slower but more familiar)

This guide covers practical approaches to unit test these code snippets **outside** of ThingsBoard UI.

---

## Table of Contents

1. [Built-in Testing (ThingsBoard UI)](#built-in-testing-thingsboard-ui)
2. [Extracting Code for Unit Testing](#extracting-code-for-unit-testing)
3. [Testing JavaScript Snippets](#testing-javascript-snippets)
4. [Testing TBEL Snippets](#testing-tbel-snippets)
5. [Testing Decoder Functions](#testing-decoder-functions)
6. [Testing Script Transform Nodes](#testing-script-transform-nodes)
7. [CI/CD Integration](#cicd-integration)
8. [Best Practices](#best-practices)

---

## Built-in Testing (ThingsBoard UI)

### Using the Test Function in UI

ThingsBoard provides built-in testing for many script nodes:

**Steps:**
1. Open your rule chain in ThingsBoard UI
2. Edit a Script Filter, Script Transform, or similar node
3. Click "Test Filter Function" / "Test Transform Function"
4. Fill in test inputs:
   - **Message Type**: POST_TELEMETRY_REQUEST, etc.
   - **Message Payload**: JSON data
   - **Metadata**: Key-value pairs
5. Click "Test" to see output

**Limitations:**
- Manual testing only
- No automation
- No version control integration
- No CI/CD support
- Testing one scenario at a time

**Best for:** Quick validation during development

---

## Extracting Code for Unit Testing

### Where ThingsBoard Stores Scripts

Scripts are stored in JSON configuration within:
- Rule chain definitions (JSON export)
- Integration configurations (database)
- Dashboard widgets (JSON)

### Extraction Strategy

**Option 1: Export from UI**
```bash
# Export rule chain via UI
# Save as: rule-chains/my-rule-chain.json
```

**Option 2: Version Control Integration**
- Use ThingsBoard PE's VCS Auto-Commit feature
- Automatically commits rule chains to Git
- Enables automated testing in CI/CD

### Example: Script Stored in Rule Chain JSON

```json
{
  "name": "Temperature Filter",
  "type": "org.thingsboard.rule.engine.filter.TbJsFilterNode",
  "configuration": {
    "jsScript": "return msg.temperature > 25 && msg.temperature < 100;"
  }
}
```

---

## Testing JavaScript Snippets

### Approach 1: Node.js Unit Tests

ThingsBoard JavaScript snippets can be tested with standard Node.js test frameworks.

**Setup:**
```bash
npm install --save-dev jest
```

**Directory Structure:**
```
thingsboard-scripts/
├── scripts/
│   ├── temperature-filter.js
│   └── telemetry-transform.js
├── tests/
│   ├── temperature-filter.test.js
│   └── telemetry-transform.test.js
└── package.json
```

### Example: Temperature Filter

**scripts/temperature-filter.js**
```javascript
/**
 * ThingsBoard Filter Node Script
 * Returns true if temperature is valid
 */
function temperatureFilter(msg, metadata, msgType) {
  // Extract temperature from message
  const temp = msg.temperature;

  // Validate range
  if (temp === undefined || temp === null) {
    return false;
  }

  // Check valid range (25-100°C)
  return temp > 25 && temp < 100;
}

// Export for testing
if (typeof module !== 'undefined' && module.exports) {
  module.exports = temperatureFilter;
}
```

**tests/temperature-filter.test.js**
```javascript
const temperatureFilter = require('../scripts/temperature-filter');

describe('Temperature Filter', () => {

  test('should return true for valid temperature', () => {
    const msg = { temperature: 30 };
    const metadata = {};
    const msgType = 'POST_TELEMETRY_REQUEST';

    expect(temperatureFilter(msg, metadata, msgType)).toBe(true);
  });

  test('should return false for temperature below range', () => {
    const msg = { temperature: 20 };
    expect(temperatureFilter(msg, {}, 'POST_TELEMETRY_REQUEST')).toBe(false);
  });

  test('should return false for temperature above range', () => {
    const msg = { temperature: 105 };
    expect(temperatureFilter(msg, {}, 'POST_TELEMETRY_REQUEST')).toBe(false);
  });

  test('should return false for undefined temperature', () => {
    const msg = {};
    expect(temperatureFilter(msg, {}, 'POST_TELEMETRY_REQUEST')).toBe(false);
  });

  test('should return false for null temperature', () => {
    const msg = { temperature: null };
    expect(temperatureFilter(msg, {}, 'POST_TELEMETRY_REQUEST')).toBe(false);
  });

  test('should handle edge cases', () => {
    expect(temperatureFilter({ temperature: 25 }, {}, 'POST_TELEMETRY_REQUEST')).toBe(false);
    expect(temperatureFilter({ temperature: 25.1 }, {}, 'POST_TELEMETRY_REQUEST')).toBe(true);
    expect(temperatureFilter({ temperature: 99.9 }, {}, 'POST_TELEMETRY_REQUEST')).toBe(true);
    expect(temperatureFilter({ temperature: 100 }, {}, 'POST_TELEMETRY_REQUEST')).toBe(false);
  });
});
```

**Run tests:**
```bash
npm test
```

### Example: Transform Script

**scripts/telemetry-transform.js**
```javascript
/**
 * ThingsBoard Transform Node Script
 * Enriches telemetry with calculated fields
 */
function telemetryTransform(msg, metadata, msgType) {
  // Parse if string
  if (typeof msg === 'string') {
    msg = JSON.parse(msg);
  }

  // Add timestamp if missing
  if (!msg.ts) {
    msg.ts = Date.now();
  }

  // Calculate heat index if temperature and humidity present
  if (msg.temperature !== undefined && msg.humidity !== undefined) {
    msg.heatIndex = calculateHeatIndex(msg.temperature, msg.humidity);
  }

  // Add device type from metadata
  if (metadata.deviceType) {
    msg.deviceType = metadata.deviceType;
  }

  return { msg: msg, metadata: metadata, msgType: msgType };
}

function calculateHeatIndex(temp, humidity) {
  // Simplified heat index formula
  return temp + (0.5 * humidity / 100 * temp);
}

// Export for testing
if (typeof module !== 'undefined' && module.exports) {
  module.exports = { telemetryTransform, calculateHeatIndex };
}
```

**tests/telemetry-transform.test.js**
```javascript
const { telemetryTransform, calculateHeatIndex } = require('../scripts/telemetry-transform');

describe('Telemetry Transform', () => {

  test('should add timestamp if missing', () => {
    const msg = { temperature: 25 };
    const result = telemetryTransform(msg, {}, 'POST_TELEMETRY_REQUEST');

    expect(result.msg.ts).toBeDefined();
    expect(typeof result.msg.ts).toBe('number');
  });

  test('should calculate heat index', () => {
    const msg = { temperature: 30, humidity: 60 };
    const result = telemetryTransform(msg, {}, 'POST_TELEMETRY_REQUEST');

    expect(result.msg.heatIndex).toBeDefined();
    expect(result.msg.heatIndex).toBe(39); // 30 + (0.5 * 60/100 * 30)
  });

  test('should add device type from metadata', () => {
    const msg = { temperature: 25 };
    const metadata = { deviceType: 'temperature-sensor' };
    const result = telemetryTransform(msg, metadata, 'POST_TELEMETRY_REQUEST');

    expect(result.msg.deviceType).toBe('temperature-sensor');
  });

  test('should handle string input', () => {
    const msgString = '{"temperature": 25}';
    const result = telemetryTransform(msgString, {}, 'POST_TELEMETRY_REQUEST');

    expect(result.msg.temperature).toBe(25);
  });

  test('should preserve original fields', () => {
    const msg = { temperature: 25, humidity: 60, pressure: 1013 };
    const result = telemetryTransform(msg, {}, 'POST_TELEMETRY_REQUEST');

    expect(result.msg.temperature).toBe(25);
    expect(result.msg.humidity).toBe(60);
    expect(result.msg.pressure).toBe(1013);
  });
});

describe('Heat Index Calculation', () => {

  test('should calculate correctly for typical values', () => {
    expect(calculateHeatIndex(30, 60)).toBe(39);
    expect(calculateHeatIndex(25, 80)).toBe(35);
  });

  test('should handle edge cases', () => {
    expect(calculateHeatIndex(0, 0)).toBe(0);
    expect(calculateHeatIndex(100, 100)).toBe(150);
  });
});
```

### Approach 2: Extracting Inline Scripts

For scripts embedded in rule chain JSON:

**extract-scripts.js**
```javascript
const fs = require('fs');

/**
 * Extract all JavaScript snippets from rule chain JSON
 */
function extractScriptsFromRuleChain(ruleChainPath, outputDir) {
  const ruleChain = JSON.parse(fs.readFileSync(ruleChainPath, 'utf8'));
  const scripts = [];

  // Iterate through nodes
  ruleChain.nodes.forEach((node, index) => {
    const config = node.configuration;

    // Check for jsScript field
    if (config.jsScript) {
      const scriptName = node.name.replace(/\s+/g, '-').toLowerCase();
      const scriptPath = `${outputDir}/${scriptName}.js`;

      // Wrap script in testable function
      const wrappedScript = wrapScript(config.jsScript, node.type);

      fs.writeFileSync(scriptPath, wrappedScript);

      scripts.push({
        name: node.name,
        type: node.type,
        path: scriptPath
      });

      console.log(`Extracted: ${node.name} -> ${scriptPath}`);
    }
  });

  return scripts;
}

function wrapScript(script, nodeType) {
  let wrapper = '';

  if (nodeType.includes('Filter')) {
    wrapper = `
/**
 * Auto-extracted filter script
 * @param {Object} msg - Message payload
 * @param {Object} metadata - Message metadata
 * @param {string} msgType - Message type
 * @returns {boolean} - Filter result
 */
function filter(msg, metadata, msgType) {
  ${script}
}

if (typeof module !== 'undefined' && module.exports) {
  module.exports = filter;
}
`;
  } else if (nodeType.includes('Transform')) {
    wrapper = `
/**
 * Auto-extracted transform script
 * @param {Object} msg - Message payload
 * @param {Object} metadata - Message metadata
 * @param {string} msgType - Message type
 * @returns {Object} - Transformed result
 */
function transform(msg, metadata, msgType) {
  ${script}
  return { msg: msg, metadata: metadata, msgType: msgType };
}

if (typeof module !== 'undefined' && module.exports) {
  module.exports = transform;
}
`;
  } else {
    wrapper = `
/**
 * Auto-extracted script
 */
${script}

if (typeof module !== 'undefined' && module.exports) {
  module.exports = { msg, metadata, msgType };
}
`;
  }

  return wrapper;
}

// Usage
const scripts = extractScriptsFromRuleChain(
  './rule-chains/main-telemetry-chain.json',
  './scripts'
);

console.log(`\nExtracted ${scripts.length} scripts`);
```

**Run extraction:**
```bash
node extract-scripts.js
```

---

## Testing TBEL Snippets

TBEL is more challenging to test because it's a custom language. Two approaches:

### Approach 1: Use ThingsBoard TBEL Library

ThingsBoard provides a standalone TBEL library you can use in Java tests.

**Maven dependency:**
```xml
<dependency>
    <groupId>org.thingsboard</groupId>
    <artifactId>tbel</artifactId>
    <version>1.0.0</version>
    <scope>test</scope>
</dependency>
```

**Java Unit Test:**
```java
package com.yourcompany.thingsboard.tests;

import org.junit.jupiter.api.Test;
import org.mvel2.MVEL;
import java.util.HashMap;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

public class TBELScriptTest {

    @Test
    public void testTemperatureFilter() {
        // TBEL script from ThingsBoard
        String tbelScript = "return msg.temperature > 25 && msg.temperature < 100;";

        // Prepare message context
        Map<String, Object> context = new HashMap<>();
        Map<String, Object> msg = new HashMap<>();
        msg.put("temperature", 30);
        context.put("msg", msg);

        // Execute TBEL
        Object result = MVEL.eval(tbelScript, context);

        assertTrue((Boolean) result, "Should return true for valid temperature");
    }

    @Test
    public void testTemperatureFilterEdgeCases() {
        String tbelScript = "return msg.temperature > 25 && msg.temperature < 100;";

        // Test below range
        Map<String, Object> context1 = createContext(20);
        assertFalse((Boolean) MVEL.eval(tbelScript, context1));

        // Test above range
        Map<String, Object> context2 = createContext(105);
        assertFalse((Boolean) MVEL.eval(tbelScript, context2));

        // Test boundary
        Map<String, Object> context3 = createContext(25);
        assertFalse((Boolean) MVEL.eval(tbelScript, context3));

        Map<String, Object> context4 = createContext(25.1);
        assertTrue((Boolean) MVEL.eval(tbelScript, context4));
    }

    @Test
    public void testTransformScript() {
        String tbelScript =
            "msg.fahrenheit = msg.temperature * 9/5 + 32;" +
            "return {msg: msg, metadata: metadata, msgType: msgType};";

        Map<String, Object> context = new HashMap<>();
        Map<String, Object> msg = new HashMap<>();
        msg.put("temperature", 25);
        context.put("msg", msg);
        context.put("metadata", new HashMap<String, String>());
        context.put("msgType", "POST_TELEMETRY_REQUEST");

        Map<String, Object> result = (Map<String, Object>) MVEL.eval(tbelScript, context);

        Map<String, Object> resultMsg = (Map<String, Object>) result.get("msg");
        assertEquals(77.0, (Double) resultMsg.get("fahrenheit"), 0.1);
    }

    private Map<String, Object> createContext(double temperature) {
        Map<String, Object> context = new HashMap<>();
        Map<String, Object> msg = new HashMap<>();
        msg.put("temperature", temperature);
        context.put("msg", msg);
        return context;
    }
}
```

### Approach 2: Convert TBEL to JavaScript for Testing

For simpler TBEL scripts that use basic Java-like syntax:

**tbel-to-js-converter.js**
```javascript
/**
 * Simple TBEL to JavaScript converter for testing
 * Handles basic cases - won't work for all TBEL features
 */
function convertTBELtoJS(tbelScript) {
  let jsScript = tbelScript;

  // Convert foreach to JavaScript for...of
  jsScript = jsScript.replace(
    /foreach\s*\(\s*(\w+)\s*:\s*(\w+)\s*\)/g,
    'for (const $1 of $2)'
  );

  // Handle null-safe operator (simplified)
  jsScript = jsScript.replace(/(\w+)\?\.(\w+)/g, '($1 && $1.$2)');

  return jsScript;
}

// Example usage
const tbelScript = `
  var total = 0;
  foreach (value : msg.readings) {
    total = total + value;
  }
  return total > 100;
`;

const jsScript = convertTBELtoJS(tbelScript);
console.log(jsScript);
```

**Note:** This approach has limitations and won't work for all TBEL features. Best for simple scripts.

---

## Testing Decoder Functions

Decoders are used in integrations to convert incoming payloads.

### Example: LoRaWAN Decoder

**decoders/lora-temperature-decoder.js**
```javascript
/**
 * Decode LoRaWAN payload to ThingsBoard telemetry
 * @param {Array} bytes - Payload bytes
 * @returns {Object} - Decoded telemetry
 */
function decodeUplink(bytes) {
  // Example: 2 bytes for temperature (signed int16)
  // 2 bytes for humidity (unsigned int16)

  if (bytes.length < 4) {
    throw new Error('Invalid payload length');
  }

  // Temperature: signed int16, big-endian
  const tempRaw = (bytes[0] << 8) | bytes[1];
  const temperature = tempRaw > 32767 ? tempRaw - 65536 : tempRaw;
  const tempCelsius = temperature / 100.0;

  // Humidity: unsigned int16, big-endian
  const humidityRaw = (bytes[2] << 8) | bytes[3];
  const humidity = humidityRaw / 100.0;

  return {
    temperature: tempCelsius,
    humidity: humidity
  };
}

// For ThingsBoard integration, return in expected format
function decode(payload) {
  const bytes = hexToBytes(payload.data);
  const decoded = decodeUplink(bytes);

  return {
    deviceName: payload.deviceInfo.deviceEUI,
    deviceType: 'lora-sensor',
    telemetry: decoded,
    attributes: {
      lastSeen: new Date().toISOString()
    }
  };
}

function hexToBytes(hex) {
  const bytes = [];
  for (let i = 0; i < hex.length; i += 2) {
    bytes.push(parseInt(hex.substr(i, 2), 16));
  }
  return bytes;
}

if (typeof module !== 'undefined' && module.exports) {
  module.exports = { decodeUplink, decode, hexToBytes };
}
```

**tests/lora-temperature-decoder.test.js**
```javascript
const { decodeUplink, decode, hexToBytes } = require('../decoders/lora-temperature-decoder');

describe('LoRaWAN Temperature Decoder', () => {

  describe('decodeUplink', () => {

    test('should decode valid temperature and humidity', () => {
      // Temperature: 25.50°C = 2550 = 0x09F6
      // Humidity: 60.00% = 6000 = 0x1770
      const bytes = [0x09, 0xF6, 0x17, 0x70];

      const result = decodeUplink(bytes);

      expect(result.temperature).toBeCloseTo(25.5, 2);
      expect(result.humidity).toBeCloseTo(60.0, 2);
    });

    test('should handle negative temperatures', () => {
      // Temperature: -10.00°C = -1000 = 0xFC18 (two's complement)
      // Humidity: 50.00% = 5000 = 0x1388
      const bytes = [0xFC, 0x18, 0x13, 0x88];

      const result = decodeUplink(bytes);

      expect(result.temperature).toBeCloseTo(-10.0, 2);
      expect(result.humidity).toBeCloseTo(50.0, 2);
    });

    test('should throw error for invalid payload length', () => {
      const bytes = [0x09, 0xF6]; // Only 2 bytes

      expect(() => decodeUplink(bytes)).toThrow('Invalid payload length');
    });

    test('should handle zero values', () => {
      const bytes = [0x00, 0x00, 0x00, 0x00];

      const result = decodeUplink(bytes);

      expect(result.temperature).toBe(0);
      expect(result.humidity).toBe(0);
    });
  });

  describe('decode (ThingsBoard format)', () => {

    test('should return ThingsBoard-compatible format', () => {
      const payload = {
        data: '09F61770', // Hex string
        deviceInfo: {
          deviceEUI: 'test-device-001'
        }
      };

      const result = decode(payload);

      expect(result.deviceName).toBe('test-device-001');
      expect(result.deviceType).toBe('lora-sensor');
      expect(result.telemetry.temperature).toBeCloseTo(25.5, 2);
      expect(result.telemetry.humidity).toBeCloseTo(60.0, 2);
      expect(result.attributes.lastSeen).toBeDefined();
    });
  });

  describe('hexToBytes', () => {

    test('should convert hex string to byte array', () => {
      const hex = '09F61770';
      const bytes = hexToBytes(hex);

      expect(bytes).toEqual([0x09, 0xF6, 0x17, 0x70]);
    });

    test('should handle lowercase hex', () => {
      const hex = '09f61770';
      const bytes = hexToBytes(hex);

      expect(bytes).toEqual([0x09, 0xF6, 0x17, 0x70]);
    });
  });
});
```

**Run tests with coverage:**
```bash
npm test -- --coverage
```

---

## Testing Script Transform Nodes

### Example: Enrichment Transform

**scripts/device-enrichment.js**
```javascript
/**
 * Enrich device telemetry with additional data
 */
function enrichTelemetry(msg, metadata, msgType) {
  // Add processing timestamp
  msg.processedAt = Date.now();

  // Add device location from metadata
  if (metadata.latitude && metadata.longitude) {
    msg.location = {
      lat: parseFloat(metadata.latitude),
      lon: parseFloat(metadata.longitude)
    };
  }

  // Calculate derived metrics
  if (msg.voltage && msg.current) {
    msg.power = msg.voltage * msg.current;
  }

  // Add quality indicators
  msg.quality = calculateQuality(msg);

  // Add metadata flags
  metadata.enriched = 'true';
  metadata.enrichmentVersion = '1.0';

  return { msg: msg, metadata: metadata, msgType: msgType };
}

function calculateQuality(msg) {
  let score = 100;

  // Deduct points for missing fields
  if (!msg.temperature) score -= 20;
  if (!msg.humidity) score -= 20;
  if (!msg.pressure) score -= 10;

  // Deduct for out-of-range values
  if (msg.temperature < -50 || msg.temperature > 100) score -= 30;
  if (msg.humidity < 0 || msg.humidity > 100) score -= 30;

  return Math.max(0, score);
}

if (typeof module !== 'undefined' && module.exports) {
  module.exports = { enrichTelemetry, calculateQuality };
}
```

**tests/device-enrichment.test.js**
```javascript
const { enrichTelemetry, calculateQuality } = require('../scripts/device-enrichment');

describe('Device Enrichment Transform', () => {

  test('should add processing timestamp', () => {
    const msg = { temperature: 25 };
    const before = Date.now();
    const result = enrichTelemetry(msg, {}, 'POST_TELEMETRY_REQUEST');
    const after = Date.now();

    expect(result.msg.processedAt).toBeGreaterThanOrEqual(before);
    expect(result.msg.processedAt).toBeLessThanOrEqual(after);
  });

  test('should add location from metadata', () => {
    const msg = { temperature: 25 };
    const metadata = { latitude: '40.7128', longitude: '-74.0060' };
    const result = enrichTelemetry(msg, metadata, 'POST_TELEMETRY_REQUEST');

    expect(result.msg.location).toEqual({
      lat: 40.7128,
      lon: -74.0060
    });
  });

  test('should calculate power from voltage and current', () => {
    const msg = { voltage: 12, current: 2.5 };
    const result = enrichTelemetry(msg, {}, 'POST_TELEMETRY_REQUEST');

    expect(result.msg.power).toBe(30);
  });

  test('should calculate quality score', () => {
    const msg = { temperature: 25, humidity: 60, pressure: 1013 };
    const result = enrichTelemetry(msg, {}, 'POST_TELEMETRY_REQUEST');

    expect(result.msg.quality).toBe(100);
  });

  test('should add metadata flags', () => {
    const msg = { temperature: 25 };
    const metadata = {};
    const result = enrichTelemetry(msg, metadata, 'POST_TELEMETRY_REQUEST');

    expect(result.metadata.enriched).toBe('true');
    expect(result.metadata.enrichmentVersion).toBe('1.0');
  });

  test('should handle missing optional fields gracefully', () => {
    const msg = { temperature: 25 };
    const metadata = {};

    expect(() => {
      enrichTelemetry(msg, metadata, 'POST_TELEMETRY_REQUEST');
    }).not.toThrow();
  });
});

describe('Quality Calculation', () => {

  test('should return 100 for complete, valid data', () => {
    const msg = { temperature: 25, humidity: 60, pressure: 1013 };
    expect(calculateQuality(msg)).toBe(100);
  });

  test('should deduct for missing temperature', () => {
    const msg = { humidity: 60, pressure: 1013 };
    expect(calculateQuality(msg)).toBe(80);
  });

  test('should deduct for missing humidity', () => {
    const msg = { temperature: 25, pressure: 1013 };
    expect(calculateQuality(msg)).toBe(80);
  });

  test('should deduct for out-of-range temperature', () => {
    const msg = { temperature: 150, humidity: 60, pressure: 1013 };
    expect(calculateQuality(msg)).toBe(70);
  });

  test('should never return negative score', () => {
    const msg = { temperature: 150, humidity: -10 };
    const quality = calculateQuality(msg);
    expect(quality).toBeGreaterThanOrEqual(0);
  });
});
```

---

## CI/CD Integration

### GitHub Actions Workflow

**.github/workflows/test-thingsboard-scripts.yml**
```yaml
name: Test ThingsBoard Scripts

on:
  push:
    branches: [ main, develop ]
    paths:
      - 'scripts/**'
      - 'decoders/**'
      - 'tests/**'
  pull_request:
    branches: [ main ]

jobs:
  test-javascript:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint

      - name: Run tests
        run: npm test -- --coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
          flags: javascript-scripts

      - name: Check coverage threshold
        run: |
          COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage $COVERAGE% is below 80% threshold"
            exit 1
          fi

  test-tbel:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Setup Java
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Run TBEL tests
        run: mvn test -Dtest=TBELScriptTest

      - name: Publish test results
        if: always()
        uses: dorny/test-reporter@v1
        with:
          name: TBEL Test Results
          path: target/surefire-reports/*.xml
          reporter: java-junit

  validate-rule-chains:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Validate rule chain JSON
        run: |
          for file in rule-chains/*.json; do
            echo "Validating $file"
            jq empty "$file" || exit 1
          done

      - name: Check for deprecated nodes
        run: |
          python3 scripts/check-deprecated-nodes.py rule-chains/
```

### Pre-commit Hook

**.git/hooks/pre-commit**
```bash
#!/bin/bash

echo "Running pre-commit checks for ThingsBoard scripts..."

# Run JavaScript tests
echo "Testing JavaScript scripts..."
npm test
if [ $? -ne 0 ]; then
    echo "❌ JavaScript tests failed"
    exit 1
fi

# Run linter
echo "Running ESLint..."
npm run lint
if [ $? -ne 0 ]; then
    echo "❌ Linting failed"
    exit 1
fi

# Validate rule chain JSON
echo "Validating rule chain JSON files..."
for file in rule-chains/*.json; do
    if [ -f "$file" ]; then
        jq empty "$file" 2>/dev/null
        if [ $? -ne 0 ]; then
            echo "❌ Invalid JSON: $file"
            exit 1
        fi
    fi
done

echo "✅ All pre-commit checks passed"
exit 0
```

---

## Best Practices

### 1. Separate Business Logic from ThingsBoard Context

**Bad:**
```javascript
// Tightly coupled to ThingsBoard
return msg.temperature > 25 && msg.temperature < 100;
```

**Good:**
```javascript
// Testable business logic
function isValidTemperature(temp) {
  return temp !== undefined &&
         temp !== null &&
         temp > 25 &&
         temp < 100;
}

// ThingsBoard wrapper
function filter(msg, metadata, msgType) {
  return isValidTemperature(msg.temperature);
}
```

### 2. Use Descriptive Test Names

**Bad:**
```javascript
test('test1', () => {
  expect(filter({temperature: 30})).toBe(true);
});
```

**Good:**
```javascript
test('should return true when temperature is within valid range (25-100°C)', () => {
  const msg = { temperature: 30 };
  expect(filter(msg, {}, 'POST_TELEMETRY_REQUEST')).toBe(true);
});
```

### 3. Test Edge Cases

Always test:
- Null/undefined values
- Empty objects/arrays
- Boundary values
- Invalid data types
- Missing required fields

### 4. Mock External Dependencies

If your script calls external services:

```javascript
// Mock HTTP calls
jest.mock('axios');
const axios = require('axios');

test('should call external API', async () => {
  axios.get.mockResolvedValue({ data: { result: 'success' } });

  const result = await enrichWithExternalData(msg);

  expect(axios.get).toHaveBeenCalledWith('https://api.example.com/data');
  expect(result.externalData).toBe('success');
});
```

### 5. Version Control Your Scripts

```
thingsboard-scripts/
├── .git/
├── scripts/
│   └── filters/
│   └── transforms/
│   └── decoders/
├── rule-chains/
│   └── main-telemetry.json
│   └── alarm-processing.json
├── tests/
├── package.json
└── README.md
```

### 6. Document Expected Input/Output

```javascript
/**
 * Temperature validation filter
 *
 * @param {Object} msg - Message payload
 * @param {number} msg.temperature - Temperature in Celsius
 * @param {Object} metadata - Message metadata
 * @param {string} msgType - Message type
 * @returns {boolean} - True if temperature is valid (25-100°C)
 *
 * @example
 * filter({ temperature: 30 }, {}, 'POST_TELEMETRY_REQUEST') // returns true
 * filter({ temperature: 20 }, {}, 'POST_TELEMETRY_REQUEST') // returns false
 */
```

### 7. Measure Code Coverage

Aim for:
- **80%+ line coverage** for critical scripts
- **100% branch coverage** for filter logic
- **All edge cases covered**

```bash
npm test -- --coverage --coverageThreshold='{"global":{"lines":80}}'
```

---

## Summary

| Approach | Best For | Pros | Cons |
|----------|----------|------|------|
| **ThingsBoard UI Testing** | Quick validation | Easy, no setup | Manual, no automation |
| **JavaScript + Jest** | JS rule nodes | Full ecosystem, easy | Not for TBEL |
| **Java + MVEL** | TBEL scripts | Accurate execution | Requires Java setup |
| **Extraction Scripts** | Existing scripts | Automates testing | Initial setup effort |
| **CI/CD Integration** | Production systems | Automated, reliable | Requires infrastructure |

**Recommended Workflow:**
1. Write scripts with testability in mind (separate business logic)
2. Create unit tests alongside scripts
3. Use extraction tools for existing scripts
4. Integrate into CI/CD pipeline
5. Use ThingsBoard UI for final validation
6. Monitor in production

This approach ensures your ThingsBoard scripts are:
- ✅ Tested before deployment
- ✅ Maintainable over time
- ✅ Compatible with upgrades
- ✅ Reliable in production
