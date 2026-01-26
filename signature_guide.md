# Merchant Signature Generation Guide

## Overview

All API requests to Swich payment endpoints must include a valid signature for security verification. This guide explains how to generate signatures correctly.

---

## Quick Reference

| Property | Value |
|----------|-------|
| **Algorithm** | MD5 |
| **Output Format** | Lowercase hexadecimal (32 characters) |
| **Secret Key Format** | `mer_sk_xxxxxxxxxxxxx` (20 characters) |
| **Key Normalization** | All keys converted to **lowercase** |
| **Parameter Sorting** | Alphabetical (ASCII order) on **lowercase keys** |

---

## Signature Generation Algorithm

### Step-by-Step Process

```
1. Collect all request parameters (excluding 'signature')
2. Remove null values, empty strings, objects, and arrays
3. Convert all parameter keys to LOWERCASE
4. Sort parameters alphabetically by lowercase key name (ASCII order)
5. Build query string: key1=value1&key2=value2&...
6. Append your secret key directly (no & prefix)
7. Compute MD5 hash of the combined string
8. Convert to lowercase hexadecimal
```

### Visual Example

```
Request:
{
  "amount": 1000,
  "currency": "PKR",
  "orderId": "ORD-12345",
  "description": "Payment for order",
  "customerRef": { "name": "Ayesha" },  ✗ SKIPPED (object)
  "items": ["item1", "item2"],           ✗ SKIPPED (array)
  "signature": "..."                     ✗ SKIPPED (signature field)
}

Secret Key: mer_sk_abc123def456

Step 1-2: Filter parameters
  ✓ amount, currency, description, orderId

Step 3: Normalize keys to lowercase
  ✓ amount → amount
  ✓ currency → currency
  ✓ description → description
  ✓ orderId → orderid (lowercase!)

Step 4: Sort alphabetically (lowercase keys)
  ✓ amount, currency, description, orderid

Step 5: Build query string
  ✓ amount=1000&currency=PKR&description=Payment for order&orderid=ORD-12345

Step 6: Append secret key
  ✓ amount=1000&currency=PKR&description=Payment for order&orderid=ORD-12345mer_sk_abc123def456

Step 7-8: MD5 hash (lowercase)
  ✓ a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
```

---

## Value Formatting Rules

### ⚠️ Critical: Format Values Correctly

| Type | Format | Example Input | Signature Value |
|------|--------|---------------|-----------------|
| **String** | As-is | `"PKR"` | `PKR` |
| **Integer** | As-is (plain number) | `100` | `100` |
| **Long** | As-is (plain number) | `1000` | `1000` |
| **Decimal** | .NET ToString() | `1000.50` | `1000.50` |
| **Double** | .NET ToString() | `99.99` | `99.99` |
| **Float** | .NET ToString() | `50.5` | `50.5` |
| **Boolean** | Lowercase | `true` | `true` |
| **Enum** | Integer value | `WalletProvider.JazzCash` (value: 2) | `2` |
| **DateTime** | .NET ToString() | `2024-01-15T10:30:00` | `1/15/2024 10:30:00 AM` |
| **DateTimeOffset** | .NET ToString() | `2024-01-15T10:30:00+05:00` | `1/15/2024 10:30:00 AM +05:00` |
| **Null** | **SKIP** | `null` | *(not included)* |
| **Empty String** | **SKIP** | `""` | *(not included)* |
| **Object** | **SKIP** | `{ "name": "..." }` | *(not included)* |
| **Array** | **SKIP** | `["a", "b"]` | *(not included)* |

### Important Notes

1. **🔑 ALL Keys Must Be Lowercase**
   - `orderId` → `orderid`
   - `callbackUrl` → `callbackurl`
   - `paymentMethod` → `paymentmethod`
   - This prevents signature mismatches due to casing differences

2. **Numbers**: Format depends on type
   - **Integers** (int, long): Plain number → `100`, `1000`, `20`
   - **Decimals** (decimal, double, float): Use .NET ToString() → `1000.50`, `99.99`
   - **Enums**: Convert to integer value → `WalletProvider.JazzCash` (2) → `"2"`

3. **Booleans**: Must be lowercase
   - ✅ `true`, `false`
   - ❌ `True`, `FALSE`, `TRUE`

4. **Objects & Arrays**: Completely excluded from signature
   - Complex nested objects are NOT included
   - Arrays of any type are NOT included

5. **DateTime Values**: Use .NET default ToString()
   - Will vary based on culture settings
   - Recommended: Use ISO 8601 strings for consistency

---

## Code Examples

### C# (.NET)

```csharp
using System.Security.Cryptography;
using System.Text;

public static string GenerateSignature(Dictionary<string, object> parameters, string secretKey)
{
    // Filter, normalize keys to lowercase, and sort parameters
    var sortedParams = new SortedDictionary<string, string>(StringComparer.Ordinal);
    
    foreach (var kvp in parameters)
    {
        // Skip signature field
        if (kvp.Key.Equals("signature", StringComparison.OrdinalIgnoreCase))
            continue;
            
        // Skip null values
        if (kvp.Value == null)
            continue;
            
        // Skip complex types (objects, arrays)
        if (IsComplexType(kvp.Value))
            continue;
            
        // Normalize key to lowercase and format value
        var normalizedKey = kvp.Key.ToLowerInvariant();
        sortedParams[normalizedKey] = FormatValue(kvp.Value);
    }

    // Build query string
    var queryString = string.Join("&", 
        sortedParams.Select(kvp => $"{kvp.Key}={kvp.Value}"));

    // Append secret key and compute MD5
    var signatureInput = queryString + secretKey;
    
    using var md5 = MD5.Create();
    var hashBytes = md5.ComputeHash(Encoding.UTF8.GetBytes(signatureInput));
    
    return BitConverter.ToString(hashBytes).Replace("-", "").ToLowerInvariant();
}

private static bool IsComplexType(object value)
{
    if (value == null) return false;
    var type = value.GetType();
    
    // Check for arrays
    if (type.IsArray) return true;
    
    // Check for collections (but not string)
    if (typeof(System.Collections.IEnumerable).IsAssignableFrom(type) && type != typeof(string))
        return true;
    
    // Check for complex objects (classes, but not string or decimal)
    return type.IsClass && type != typeof(string) && !type.IsPrimitive && type != typeof(decimal);
}

private static string FormatValue(object value)
{
    return value switch
    {
        // Integers: plain number
        int i => i.ToString(),
        long l => l.ToString(),
        
        // Decimals: use ToString() (preserves precision)
        decimal d => d.ToString(),
        double dbl => dbl.ToString(),
        float f => f.ToString(),
        
        // Booleans: lowercase
        bool b => b.ToString().ToLowerInvariant(),
        
        // Enums: convert to integer
        Enum e => Convert.ToInt32(e).ToString(),
        
        // DateTime: default ToString()
        DateTime dt => dt.ToString(),
        DateTimeOffset dto => dto.ToString(),
        
        // Default: ToString()
        _ => value.ToString()
    };
}
```

### JavaScript (Node.js)

```javascript
const crypto = require('crypto');

function generateSignature(parameters, secretKey) {
    // Filter and normalize parameters
    const filtered = Object.entries(parameters)
        .filter(([key, value]) => {
            if (key.toLowerCase() === 'signature') return false;
            if (value === null || value === undefined) return false;
            if (typeof value === 'object') return false; // Skip objects & arrays
            return true;
        })
        // Normalize keys to lowercase
        .map(([key, value]) => [key.toLowerCase(), value]);

    // Sort alphabetically by lowercase key
    filtered.sort((a, b) => a[0].localeCompare(b[0]));

    // Format values and build query string
    const queryString = filtered
        .map(([key, value]) => `${key}=${formatValue(value)}`)
        .join('&');

    // Append secret key and compute MD5
    const signatureInput = queryString + secretKey;
    return crypto.createHash('md5').update(signatureInput).digest('hex');
}

function formatValue(value) {
    if (typeof value === 'number') {
        // Check if it's an integer or has decimal places
        if (Number.isInteger(value)) {
            return value.toString(); // Plain number for integers
        }
        return value.toString(); // Keep decimal precision
    }
    if (typeof value === 'boolean') {
        return value.toString().toLowerCase();
    }
    return String(value);
}

// Usage
const params = {
    amount: 1000,        // Integer: will be "1000"
    currency: 'PKR',
    orderId: 'ORD-12345' // Note: key becomes "orderid" (lowercase)
};
const signature = generateSignature(params, 'mer_sk_abc123def456');
console.log(signature); // MD5 hash based on: amount=1000&currency=PKR&orderid=ORD-12345mer_sk_abc123def456
```

### Python

```python
import hashlib
from decimal import Decimal

def generate_signature(parameters: dict, secret_key: str) -> str:
    # Filter and normalize parameters
    filtered = {}
    for key, value in parameters.items():
        if key.lower() == 'signature':
            continue
        if value is None:
            continue
        if isinstance(value, (dict, list)):  # Skip objects & arrays
            continue
        
        # Normalize key to lowercase
        normalized_key = key.lower()
        filtered[normalized_key] = format_value(value)
    
    # Sort alphabetically by lowercase key
    sorted_params = dict(sorted(filtered.items()))
    
    # Build query string
    query_string = '&'.join([f'{k}={v}' for k, v in sorted_params.items()])
    
    # Append secret key and compute MD5
    signature_input = query_string + secret_key
    return hashlib.md5(signature_input.encode('utf-8')).hexdigest()

def format_value(value):
    if isinstance(value, bool):
        # Handle bool before int (since bool is subclass of int in Python)
        return str(value).lower()
    if isinstance(value, int):
        # Integers: plain number
        return str(value)
    if isinstance(value, (float, Decimal)):
        # Decimals: use str() to preserve precision
        return str(value)
    return str(value)

# Usage
params = {
    'amount': 1000.00,
    'currency': 'PKR',
    'orderId': 'ORD-12345'
}
signature = generate_signature(params, 'mer_sk_abc123def456')
print(signature)
```

### PHP

```php
<?php

function generateSignature(array $parameters, string $secretKey): string {
    // Filter and normalize parameters
    $filtered = [];
    foreach ($parameters as $key => $value) {
        if (strtolower($key) === 'signature') continue;
        if ($value === null) continue;
        if (is_array($value) || is_object($value)) continue; // Skip objects & arrays
        
        // Normalize key to lowercase
        $normalizedKey = strtolower($key);
        $filtered[$normalizedKey] = formatValue($value);
    }
    
    // Sort alphabetically by lowercase key (ASCII order)
    ksort($filtered, SORT_STRING);
    
    // Build query string
    $parts = [];
    foreach ($filtered as $key => $value) {
        $parts[] = "$key=$value";
    }
    $queryString = implode('&', $parts);
    
    // Append secret key and compute MD5
    $signatureInput = $queryString . $secretKey;
    return strtolower(md5($signatureInput));
}

function formatValue($value): string {
    if (is_bool($value)) {
        // Handle bool before int check
        return $value ? 'true' : 'false';
    }
    if (is_int($value)) {
        // Integers: plain number
        return (string)$value;
    }
    if (is_float($value)) {
        // Floats: use string conversion to preserve precision
        return (string)$value;
    }
    return (string)$value;
}

// Usage
$params = [
    'amount' => 1000.00,
    'currency' => 'PKR',
    'orderId' => 'ORD-12345'
];
$signature = generateSignature($params, 'mer_sk_abc123def456');
echo $signature;
?>
```

### Java

```java
import java.security.MessageDigest;
import java.util.*;
import java.text.DecimalFormat;

public class SignatureGenerator {
    
    public static String generateSignature(Map<String, Object> parameters, String secretKey) 
            throws Exception {
        
        // Filter, normalize keys, and sort parameters
        TreeMap<String, String> sorted = new TreeMap<>();
        
        for (Map.Entry<String, Object> entry : parameters.entrySet()) {
            String key = entry.getKey();
            Object value = entry.getValue();
            
            if (key.equalsIgnoreCase("signature")) continue;
            if (value == null) continue;
            if (value instanceof Map || value instanceof List) continue; // Skip objects & arrays
            
            // Normalize key to lowercase
            String normalizedKey = key.toLowerCase();
            sorted.put(normalizedKey, formatValue(value));
        }
        
        // Build query string
        StringBuilder queryString = new StringBuilder();
        for (Map.Entry<String, String> entry : sorted.entrySet()) {
            if (queryString.length() > 0) queryString.append("&");
            queryString.append(entry.getKey()).append("=").append(entry.getValue());
        }
        
        // Append secret key and compute MD5
        String signatureInput = queryString.toString() + secretKey;
        
        MessageDigest md = MessageDigest.getInstance("MD5");
        byte[] hashBytes = md.digest(signatureInput.getBytes("UTF-8"));
        
        StringBuilder hexString = new StringBuilder();
        for (byte b : hashBytes) {
            hexString.append(String.format("%02x", b));
        }
        
        return hexString.toString();
    }
    
    private static String formatValue(Object value) {
        if (value instanceof Boolean) {
            // Handle boolean before Integer check
            return value.toString().toLowerCase();
        }
        if (value instanceof Integer || value instanceof Long) {
            // Integers: plain number
            return value.toString();
        }
        if (value instanceof Double || value instanceof Float) {
            // Decimals: use toString() to preserve precision
            return value.toString();
        }
        if (value instanceof Enum) {
            // Enums: convert to integer value
            return String.valueOf(((Enum<?>) value).ordinal());
        }
        return value.toString();
    }
}
```

---

## Testing Your Implementation

### Test Endpoint

Use our signature testing API to verify your implementation:

```
POST /api/v1/signature/generate
```

**Request:**
```json
{
  "parameters": {
    "amount": 1000,
    "currency": "PKR",
    "orderId": "ORD-12345"
  },
  "secretKey": "mer_sk_abc123def456"
}
```

**Response:**
```json
{
  "signature": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
  "queryString": "amount=1000&currency=PKR&orderid=ORD-12345",
  "sortedParameters": {
    "amount": "1000",
    "currency": "PKR",
    "orderid": "ORD-12345"
  },
  "signatureInput": "amount=1000&currency=PKR&orderid=ORD-12345mer_sk_abc...456"
}
```

### Verify Endpoint

```
POST /api/v1/signature/verify
```

**Request:**
```json
{
  "parameters": {
    "amount": 1000,
    "currency": "PKR",
    "orderId": "ORD-12345"
  },
  "secretKey": "mer_sk_abc123def456",
  "providedSignature": "your_generated_signature"
}
```

**Response (Success):**
```json
{
  "isValid": true,
  "providedSignature": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
  "expectedSignature": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
  "queryString": "amount=1000&currency=PKR&orderid=ORD-12345",
  "issues": []
}
```

**Response (Failure):**
```json
{
  "isValid": false,
  "providedSignature": "wrong_signature",
  "expectedSignature": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
  "queryString": "amount=1000&currency=PKR&orderid=ORD-12345",
  "sortedParameters": {
    "amount": "1000",
    "currency": "PKR",
    "orderid": "ORD-12345"
  },
  "issues": [
    "Signature length is 15, expected 32 characters for MD5",
    "Signature contains uppercase letters - should be lowercase"
  ]
}
```

---

## Common Mistakes & Troubleshooting

### ? Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| Signature length ? 32 | Wrong hash algorithm or encoding | Use MD5, output as hex |
| Uppercase letters in signature | Not converting to lowercase | Call `.toLowerCase()` on result |
| Wrong parameter order | Not sorting alphabetically | Use ASCII/lexicographic sort |
| Including signature field | Signature included in params | Exclude 'signature' key |
| Wrong number format | Not using 2 decimal places | Format as `"0.00"` |
| Including nested objects | Objects/arrays in signature | Skip complex types |

### ?? Debugging Checklist

1. **Is your signature 32 characters?**
   - MD5 produces 32 hex characters
   - If shorter/longer, check your hash function

2. **Is your signature lowercase?**
   - Must be `abcdef123456...` not `ABCDEF123456...`

3. **Are all parameter keys converted to lowercase?**
   - `orderId` → `orderid`
   - `callbackUrl` → `callbackurl`
   - `Amount` → `amount`

4. **Are parameters sorted correctly?**
   - Alphabetical (ASCII) order on **lowercase keys**
   - All keys must be lowercase before sorting

5. **Are numbers formatted correctly?**
   - Integers: Plain number → `1000`, `20`, `5`
   - Decimals: Use ToString() → `1000.50`, `99.99`
   - Enums: Integer value → `2`, `5`

6. **Are you skipping objects and arrays?**
   - `customerRef: { name: "..." }` ? SKIP
   - `items: [...]` ? SKIP

7. **Is the secret key appended correctly?**
   - NO `&` before secret key
   - Directly append: `...orderId=123mer_sk_xxx`

---

## API Request Example

### Complete Payment Request

```bash
curl -X POST "https://api.swich.com/api/v1/payments/initiate" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -d '{
    "amount": 1500,
    "currency": "PKR",
    "orderId": "ORD-2024-001",
    "description": "Payment for Order #2024-001",
    "callbackUrl": "https://yoursite.com/callback",
    "customerRef": {
      "name": "Ayesha Khan",
      "email": "ayesha@example.com"
    },
    "signature": "a1b2c3d4e5f6789012345678abcdef12"
  }'
```

**Signature Calculation:**
```
Parameters (after filtering, customerRef is skipped as it's an object):
  amount=1500
  callbackUrl=https://yoursite.com/callback
  currency=PKR
  description=Payment for Order #2024-001
  orderId=ORD-2024-001

Keys normalized to lowercase:
  amount → amount
  callbackUrl → callbackurl
  currency → currency
  description → description
  orderId → orderid

Query String (sorted by lowercase keys):
  amount=1500&callbackurl=https://yoursite.com/callback&currency=PKR&description=Payment for Order #2024-001&orderid=ORD-2024-001

Signature Input:
  amount=1500&callbackurl=https://yoursite.com/callback&currency=PKR&description=Payment for Order #2024-001&orderid=ORD-2024-001mer_sk_your_secret_key

MD5 Hash:
  a1b2c3d4e5f6789012345678abcdef12
```

---

## Security Best Practices

### ?? Protect Your Secret Key

1. **Never expose in client-side code**
   - Don't include in JavaScript, mobile apps, or browser code
   - Generate signatures on your server only

2. **Store securely**
   - Use environment variables
   - Use secret management services (Azure Key Vault, AWS Secrets Manager)
   - Never commit to version control

3. **Rotate if compromised**
   - Contact admin to regenerate your secret key
   - Update all integrations with new key

4. **Use HTTPS only**
   - All API calls must use HTTPS
   - Never send signatures over HTTP

### ?? Secret Key Format

```
Format: mer_sk_xxxxxxxxxxxxx
Length: 20 characters
Prefix: mer_sk_ (7 characters)
Random: 13 alphanumeric characters (lowercase)

Example: mer_sk_a1b2c3d4e5f6g
```

---

## Getting Help

### Test Your Implementation

1. Use `/api/v1/signature/example` to see a complete example
2. Use `/api/v1/signature/generate` to generate signatures for testing
3. Use `/api/v1/signature/verify` to debug signature mismatches

### Contact Support

If you continue to have issues:
- Email: support@swich.com
- Include: Your merchant ID, request payload (without signature), and the signature you generated

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2024-01-15 | Initial release |
| 1.1 | 2024-01-20 | Added: Skip objects and arrays from signature calculation |
| 2.0 | 2026-01-26 | **BREAKING**: Keys normalized to lowercase; Integer formatting changed (plain numbers); Enum handling added; DateTime uses default ToString() |

---

## Migration from v1.1 to v2.0

### ⚠️ Breaking Changes

1. **All keys are now lowercase in signature calculation**
   - Old: `orderId`, `callbackUrl`, `paymentMethod`
   - New: `orderid`, `callbackurl`, `paymentmethod`

2. **Integer formatting changed**
   - Old: `1000` → `"1000.00"`
   - New: `1000` → `"1000"`

3. **Enum handling**
   - New: Enums are converted to integer values
   - Example: `WalletProvider.JazzCash` (value: 2) → `"2"`

4. **DateTime formatting**
   - Old: ISO 8601 format `"2024-01-15T10:30:00"`
   - New: .NET default ToString() (culture-dependent)

### Migration Steps

If you have existing integrations:
1. Update your signature generation code to convert all keys to lowercase
2. Remove forced 2-decimal formatting for integers
3. Test your signatures using the `/api/v1/signature/generate` endpoint
4. Verify with `/api/v1/signature/verify` before deploying to production

---

**Document Version:** 2.0  
**Last Updated:** 2026-01-26  
**API Version:** v1
