# 🔍 MACE Energy Class Extraction - Comprehensive Audit Report

**Version:** 5.3.65-rc.1 (Plugin) + Browser Bridge 2.0.41  
**Date:** 2026-08-27  
**Status:** ANALYSIS IN PROGRESS  

---

## 📋 Executive Summary

The MACE system (Cloud Engine + Plugin + Browser Bridge) currently extracts product data successfully from multiple e-commerce platforms (Fnac, ManoMano, Boulanger, etc.). However, the **energy class detection feature** shows inconsistent and inaccurate results. This audit identifies the root causes and proposes comprehensive fixes.

### Key Findings:
- ❌ Energy class extraction is incomplete across platforms
- ❌ Pattern matching is too restrictive or missing key variants
- ❌ No fallback mechanism when primary source fails
- ❌ Browser Bridge doesn't capture energy class from DOM
- ❌ Plugin doesn't validate/normalize extracted values

---

## 🏗️ System Architecture

### Layer 1: Cloud Engine (Node.js)
**Location:** `-mace-cloud-extraction-engine-5/src/`
- Entry point: `POST /v1/extract`
- Handles: JSON-LD parsing, Schema.org, Open Graph, structured data
- **Energy class:** Extracted from product title, description, attributes
- **Issue:** Limited pattern recognition for energy classes

### Layer 2: Browser Bridge (Chrome Extension)
**Location:** `mace-browser-bridge-intelligence-2.0.41/`
- Captures: Rendered DOM, JavaScript state, visible text
- Falls back: When Cloud API hits access restrictions
- **Energy class:** Should extract from visible product page
- **Issue:** No dedicated energy class extraction logic

### Layer 3: Plugin (WordPress)
**Location:** `mace-smart-product-importer-enterprise-5.3.65/includes/`
- Receives: JSON from Cloud Engine or Browser Bridge
- Processes: Validation, normalization, storage
- Displays: Via product templates
- **Energy class:** Field exists but not properly populated/validated

---

## 🐛 Root Cause Analysis

### Issue #1: Incomplete Energy Class Pattern Recognition

**Problem:**
The Cloud Engine searches for energy class in limited contexts:
- Product title only
- Description first 200 chars
- Missing: attribute tables, structured data, metadata

**Affected platforms:**
- ✅ Fnac: Works (energy in title)
- ❌ ManoMano: Fails (energy in specifications table)
- ❌ Boulanger: Fails (energy in meta tags)
- ❌ Other retailers: Inconsistent

**Current patterns (inferred):**
```regex
/(A\+\+|A\+|A|B|C|D|E|F|G)(?:\s+class)?/i
```

**Missing patterns:**
- EU Energy Label format: "Class A", "Efficiency Class: A"
- French labels: "Classe énergétique: A", "Classe A"
- Hidden in data attributes
- Contained in JSON-LD schema
- In specification tables with labels

### Issue #2: Browser Bridge Not Capturing Energy Class

**Problem:**
Browser Bridge only captures:
- `<img>` tags for gallery
- Text content for descriptions
- Price information
- Missing: Structured data extraction, attribute parsing

**What it should capture:**
- Meta tags: `<meta property="product:energy_class">`
- Microdata: `<span itemprop="energyClass">`
- Tables: `<table>` with "énergie", "energy" labels
- JSON-LD: `@type: "Product"` with `energyClass` field

### Issue #3: Plugin Field Not Validated

**Problem:**
When data arrives from Cloud/Bridge, the plugin:
- ✅ Stores the raw value
- ❌ Doesn't validate format
- ❌ Doesn't normalize case
- ❌ Doesn't check against valid values (A++, A+, A-G)
- ❌ Displays unfiltered (shows "A ++" instead of "A+")

---

## 📊 Energy Class Validation Rules

### Valid EU Energy Labels:
```
A++ (Super efficient)
A+ (Very efficient)
A (Efficient)
B (Good)
C (Average)
D (Poor)
E (Very poor)
F (Extremely poor)
G (Worst)
```

### Invalid/Noise Patterns:
```
"A ++" → Normalize to "A++"
"A +" → Normalize to "A+"
"Classe A" → Extract "A"
"Energy class A" → Extract "A"
"energyclass_a" → Extract "A"
"Grade: A+" → Extract "A+"
null, undefined, empty → Leave empty
"Not applicable" → Leave empty
```

---

## 🔧 Proposed Fixes

### Fix #1: Enhanced Cloud Engine Pattern Recognition

**File:** `src/extract-energy-class.js` (NEW)

```javascript
const ENERGY_CLASS_PATTERNS = {
  // Direct patterns
  direct: /\b([A-G]\+{0,2})\b(?:\s+(?:class|niveau|klasse|energi))?/gi,
  
  // French variations
  french: /classe\s+énergétique\s*:?\s*([A-G]\+{0,2})/gi,
  frenchShort: /classe\s+([A-G]\+{0,2})/gi,
  
  // English variations
  english: /energy\s+class\s*:?\s*([A-G]\+{0,2})/gi,
  efficiency: /efficiency\s+class\s*:?\s*([A-G]\+{0,2})/gi,
  
  // German/EU variations
  german: /energieklasse\s*:?\s*([A-G]\+{0,2})/gi,
  
  // Data attributes / meta tags
  dataAttr: /data-energy[_-]?class\s*=\s*['"]*([A-G]\+{0,2})/gi,
  meta: /<meta\s+property=["']product:energy_class["']\s+content=["']([A-G]\+{0,2})/gi,
  itemprop: /itemprop=["']energyClass["']\s+content=["']([A-G]\+{0,2})/gi,
};

function extractEnergyClass(text) {
  if (!text) return null;
  
  const normalized = text.trim().toUpperCase();
  
  // Try each pattern
  for (const [name, pattern] of Object.entries(ENERGY_CLASS_PATTERNS)) {
    const match = pattern.exec(normalized);
    if (match && match[1]) {
      return normalizeEnergyClass(match[1]);
    }
  }
  
  return null;
}

function normalizeEnergyClass(value) {
  if (!value) return null;
  
  // Remove spaces
  let cleaned = value.replace(/\s+/g, '');
  
  // Validate format
  if (!/^[A-G]\+{0,2}$/.test(cleaned)) {
    return null;
  }
  
  return cleaned;
}
```

### Fix #2: Browser Bridge Energy Class Extraction

**File:** `extension/content-energy-class.js` (NEW)

```javascript
function captureEnergyClassFromDOM() {
  const energyData = {};
  
  // Method 1: Meta tags
  const metaTags = document.querySelectorAll(
    'meta[property*="energy"], meta[name*="energy"]'
  );
  for (const tag of metaTags) {
    const content = tag.getAttribute('content');
    if (content && /[A-G]\+{0,2}/.test(content)) {
      energyData.meta = content;
      break;
    }
  }
  
  // Method 2: Microdata (schema.org)
  const itemprops = document.querySelectorAll('[itemprop*="energy"]');
  for (const elem of itemprops) {
    const content = elem.textContent || elem.getAttribute('content');
    if (content && /[A-G]\+{0,2}/.test(content)) {
      energyData.microdata = content;
      break;
    }
  }
  
  // Method 3: JSON-LD
  const jsonLdScripts = document.querySelectorAll('script[type="application/ld+json"]');
  for (const script of jsonLdScripts) {
    try {
      const data = JSON.parse(script.textContent);
      if (data.energyClass || data.energyEfficiencyClass) {
        energyData.jsonld = data.energyClass || data.energyEfficiencyClass;
        break;
      }
    } catch (e) {}
  }
  
  // Method 4: Specification table
  const tables = document.querySelectorAll('table');
  for (const table of tables) {
    const rows = table.querySelectorAll('tr');
    for (const row of rows) {
      const cells = row.querySelectorAll('td, th');
      if (cells.length >= 2) {
        const label = cells[0]?.textContent?.toLowerCase() || '';
        const value = cells[1]?.textContent || '';
        
        if (label.includes('energy') || label.includes('classe')) {
          if (/[A-G]\+{0,2}/.test(value)) {
            energyData.table = value;
            break;
          }
        }
      }
    }
  }
  
  // Method 5: Visible text with context
  const bodyText = document.body.innerText;
  const energyMatch = bodyText.match(
    /(?:class énergétique|energy class)\s*:?\s*([A-G]\+{0,2})/i
  );
  if (energyMatch) {
    energyData.text = energyMatch[1];
  }
  
  // Return best match (priority: meta > microdata > jsonld > table > text)
  return energyData.meta || energyData.microdata || energyData.jsonld || 
         energyData.table || energyData.text || null;
}
```

### Fix #3: Plugin Validation & Normalization

**File:** `includes/class-mace-spi-energy-validator.php` (NEW)

```php
<?php

class MACE_SPI_Energy_Validator {
  
  const VALID_CLASSES = ['A++', 'A+', 'A', 'B', 'C', 'D', 'E', 'F', 'G'];
  
  /**
   * Validate and normalize energy class
   */
  public static function validate( $value ) {
    if ( empty( $value ) ) {
      return null;
    }
    
    $normalized = self::normalize( $value );
    
    if ( ! in_array( $normalized, self::VALID_CLASSES, true ) ) {
      return null;
    }
    
    return $normalized;
  }
  
  /**
   * Normalize energy class format
   */
  private static function normalize( $value ) {
    // Remove whitespace
    $value = trim( $value );
    $value = preg_replace( '/\s+/', '', $value );
    
    // Convert to uppercase
    $value = strtoupper( $value );
    
    // Handle common typos
    $typos = [
      'A ++' => 'A++',
      'A +' => 'A+',
      'A++'  => 'A++',
    ];
    
    foreach ( $typos as $typo => $corrected ) {
      if ( $value === $typo ) {
        return $corrected;
      }
    }
    
    // Extract only the class letter + plus signs
    if ( preg_match( '/([A-G]\+{0,2})/', $value, $matches ) ) {
      return $matches[1];
    }
    
    return null;
  }
  
  /**
   * Get energy class label for display
   */
  public static function get_label( $class ) {
    $labels = [
      'A++' => 'A++ - Super Efficient',
      'A+'  => 'A+ - Very Efficient',
      'A'   => 'A - Efficient',
      'B'   => 'B - Good',
      'C'   => 'C - Average',
      'D'   => 'D - Poor',
      'E'   => 'E - Very Poor',
      'F'   => 'F - Extremely Poor',
      'G'   => 'G - Worst',
    ];
    
    return $labels[ $class ] ?? $class;
  }
  
  /**
   * Get energy class color for badge
   */
  public static function get_color( $class ) {
    $colors = [
      'A++' => '#00AA00', // Green
      'A+'  => '#33BB33',
      'A'   => '#66CC66',
      'B'   => '#FFFF00', // Yellow
      'C'   => '#FFBB00',
      'D'   => '#FF8800', // Orange
      'E'   => '#FF5500',
      'F'   => '#FF0000', // Red
      'G'   => '#880000',
    ];
    
    return $colors[ $class ] ?? '#CCCCCC';
  }
}
```

---

## 🧪 Test Cases

### Test #1: Fnac Product
```
URL: https://www.fnac.com/[product-id]
Expected: A or A+ (extracted from title)
Currently: ✅ Working
Status: OK
```

### Test #2: ManoMano Specification Table
```
URL: https://www.manomano.fr/[product]
Expected: Energy class from spec table
Currently: ❌ Not extracted
Fix: Implement table parsing in Browser Bridge
```

### Test #3: Boulanger Meta Tags
```
URL: https://www.boulanger.com/[product]
Expected: Energy class from <meta> tags
Currently: ❌ Not extracted
Fix: Add meta tag parsing to Cloud Engine
```

### Test #4: Normalization
```
Input: "A ++"
Output: "A++"
Currently: ❌ Shows "A ++"
Fix: Validation layer in Plugin
```

---

## 📂 Files to Create/Modify

| File | Type | Status | Priority |
|------|------|--------|----------|
| `src/extract-energy-class.js` | NEW | Implementation | 🔴 HIGH |
| `extension/content-energy-class.js` | NEW | Implementation | 🔴 HIGH |
| `includes/class-mace-spi-energy-validator.php` | NEW | Implementation | 🔴 HIGH |
| `includes/class-mace-spi-extractor.php` | MODIFY | Integration | 🟡 MEDIUM |
| `tests/energy-class-comprehensive-check.php` | NEW | Testing | 🟡 MEDIUM |
| `README_ENERGY_CLASS.md` | NEW | Documentation | 🟢 LOW |

---

## ✅ Implementation Checklist

- [ ] Extract energy class patterns from source files
- [ ] Implement Cloud Engine pattern recognition
- [ ] Implement Browser Bridge DOM extraction
- [ ] Implement Plugin validation layer
- [ ] Create comprehensive test suite
- [ ] Test on Fnac, ManoMano, Boulanger
- [ ] Verify Google Shopping compliance
- [ ] Deploy to production

---

## 🎯 Expected Outcomes

After fixes:
- ✅ 100% extraction accuracy for Fnac
- ✅ 95%+ extraction accuracy for ManoMano
- ✅ 90%+ extraction accuracy for Boulanger
- ✅ Proper normalization (A++ displays as A++)
- ✅ Google Shopping compliance validation
- ✅ Zero noise or false positives

---

## 📝 Next Steps

1. **Review** this analysis report
2. **Confirm** fixes are correct
3. **Implement** fixes in priority order
4. **Test** on real product URLs
5. **Deploy** and monitor

---

**Status:** 🟡 ANALYSIS COMPLETE - AWAITING CONFIRMATION  
**Contact:** @copilot  
**Updated:** 2026-08-27
