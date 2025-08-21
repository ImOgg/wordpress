# WordPress REST API 本地開發環境設定指南

## 前置條件
- 本地已安裝 PHP 8.2+ 環境
- 本地已安裝 MySQL 資料庫
- 已下載 WordPress 原始檔案

## 啟動 WordPress 本地環境

### 1. 啟動 PHP 內建伺服器
```bash
cd your-wordpress-folder
php -S localhost:8000
```

### 2. 設定永久連結（重要！）
1. 瀏覽器開啟 `http://localhost:8000/wp-admin`
2. 登入管理後台
3. 到 `設定` → `永久連結`
4. **選擇任何一個選項，不要選「樸素」**（建議選「文章名稱」）
5. 點擊 `儲存變更`

> ⚠️ **重要**：不設定永久連結的話，REST API 將無法正常工作！

### 3. 測試 REST API 是否正常
瀏覽器開啟或用 Postman 測試：
```
http://localhost:8000/wp-json/
```
應該回傳 JSON 格式的 WordPress API 資訊。

---

## 建立自定義 CRUD API

### 1. 建立外掛檔案
檔案路徑：`wp-content/plugins/my-api/my-api.php`

```php
<?php
/**
 * Plugin Name: My CRUD API
 * Description: 完整的 CRUD API 範例
 * Version: 1.0
 */

if (!defined('ABSPATH')) {
    exit;
}

// 外掛啟用時建立資料表
register_activation_hook(__FILE__, 'create_my_table');

function create_my_table() {
    global $wpdb;
    
    $table_name = $wpdb->prefix . 'my_items';
    
    $charset_collate = $wpdb->get_charset_collate();
    
    $sql = "CREATE TABLE $table_name (
        id mediumint(9) NOT NULL AUTO_INCREMENT,
        name tinytext NOT NULL,
        description text,
        price decimal(10,2),
        created_at datetime DEFAULT CURRENT_TIMESTAMP,
        updated_at datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
        PRIMARY KEY (id)
    ) $charset_collate;";
    
    require_once(ABSPATH . 'wp-admin/includes/upgrade.php');
    dbDelta($sql);
}

// 註冊 API 端點
add_action('rest_api_init', function () {
    
    // 讀取所有項目 (READ)
    register_rest_route('myapi/v1', '/items', array(
        'methods' => 'GET',
        'callback' => 'get_all_items',
        'permission_callback' => '__return_true'
    ));
    
    // 讀取單個項目 (READ)
    register_rest_route('myapi/v1', '/items/(?P<id>\d+)', array(
        'methods' => 'GET',
        'callback' => 'get_single_item',
        'permission_callback' => '__return_true'
    ));
    
    // 新增項目 (CREATE)
    register_rest_route('myapi/v1', '/items', array(
        'methods' => 'POST',
        'callback' => 'create_item',
        'permission_callback' => '__return_true'
    ));
    
    // 更新項目 (UPDATE)
    register_rest_route('myapi/v1', '/items/(?P<id>\d+)', array(
        'methods' => 'PUT',
        'callback' => 'update_item',
        'permission_callback' => '__return_true'
    ));
    
    // 刪除項目 (DELETE)
    register_rest_route('myapi/v1', '/items/(?P<id>\d+)', array(
        'methods' => 'DELETE',
        'callback' => 'delete_item',
        'permission_callback' => '__return_true'
    ));
});

// API 回調函數
function get_all_items() {
    global $wpdb;
    $table_name = $wpdb->prefix . 'my_items';
    
    $results = $wpdb->get_results("SELECT * FROM $table_name ORDER BY id DESC");
    
    return new WP_REST_Response($results, 200);
}

function get_single_item($request) {
    global $wpdb;
    $table_name = $wpdb->prefix . 'my_items';
    $id = $request['id'];
    
    $result = $wpdb->get_row($wpdb->prepare("SELECT * FROM $table_name WHERE id = %d", $id));
    
    if (!$result) {
        return new WP_REST_Response(array('error' => 'Item not found'), 404);
    }
    
    return new WP_REST_Response($result, 200);
}

function create_item($request) {
    global $wpdb;
    $table_name = $wpdb->prefix . 'my_items';
    
    $data = $request->get_json_params();
    
    // 如果 get_json_params() 失敗，手動解析
    if (!$data) {
        $body = $request->get_body();
        $data = json_decode($body, true);
    }
    
    if ($data === null || (is_array($data) && empty($data))) {
        return new WP_REST_Response(array('error' => 'No data provided'), 400);
    }
    
    $name = isset($data['name']) ? sanitize_text_field($data['name']) : '';
    $description = isset($data['description']) ? sanitize_textarea_field($data['description']) : '';
    $price = isset($data['price']) ? floatval($data['price']) : 0;
    
    if (empty($name)) {
        return new WP_REST_Response(array('error' => 'Name is required'), 400);
    }
    
    $result = $wpdb->insert(
        $table_name,
        array(
            'name' => $name,
            'description' => $description,
            'price' => $price
        ),
        array('%s', '%s', '%f')
    );
    
    if ($result === false) {
        return new WP_REST_Response(array('error' => 'Failed to create item'), 500);
    }
    
    return new WP_REST_Response(array(
        'success' => true,
        'id' => $wpdb->insert_id,
        'message' => 'Item created successfully'
    ), 201);
}

function update_item($request) {
    global $wpdb;
    $table_name = $wpdb->prefix . 'my_items';
    $id = $request['id'];
    
    $data = $request->get_json_params();
    
    // 如果 get_json_params() 失敗，手動解析
    if (!$data) {
        $body = $request->get_body();
        $data = json_decode($body, true);
    }
    
    if ($data === null || (is_array($data) && empty($data))) {
        return new WP_REST_Response(array('error' => 'No data provided'), 400);
    }
    
    // 檢查項目是否存在
    $existing = $wpdb->get_row($wpdb->prepare("SELECT * FROM $table_name WHERE id = %d", $id));
    if (!$existing) {
        return new WP_REST_Response(array('error' => 'Item not found'), 404);
    }
    
    // 部分更新：只更新有提供的欄位
    $update_data = array();
    $update_format = array();
    
    if (isset($data['name'])) {
        $update_data['name'] = sanitize_text_field($data['name']);
        $update_format[] = '%s';
    }
    
    if (isset($data['description'])) {
        $update_data['description'] = sanitize_textarea_field($data['description']);
        $update_format[] = '%s';
    }
    
    if (isset($data['price'])) {
        $update_data['price'] = floatval($data['price']);
        $update_format[] = '%f';
    }
    
    if (empty($update_data)) {
        return new WP_REST_Response(array('error' => 'No valid fields to update'), 400);
    }
    
    $result = $wpdb->update(
        $table_name,
        $update_data,
        array('id' => $id),
        $update_format,
        array('%d')
    );
    
    if ($result === false) {
        return new WP_REST_Response(array('error' => 'Failed to update item'), 500);
    }
    
    return new WP_REST_Response(array(
        'success' => true,
        'message' => 'Item updated successfully',
        'updated_fields' => array_keys($update_data)
    ), 200);
}

function delete_item($request) {
    global $wpdb;
    $table_name = $wpdb->prefix . 'my_items';
    $id = $request['id'];
    
    $result = $wpdb->delete(
        $table_name,
        array('id' => $id),
        array('%d')
    );
    
    if ($result === false) {
        return new WP_REST_Response(array('error' => 'Failed to delete item'), 500);
    }
    
    return new WP_REST_Response(array(
        'success' => true,
        'message' => 'Item deleted successfully'
    ), 200);
}
?>
```

### 2. 啟用外掛
1. 到 WordPress 管理後台：`外掛` → `已安裝的外掛`
2. 找到 "My CRUD API" 並點擊 `啟用`
3. 啟用時會自動建立資料表 `wp_my_items`

---

## Postman API 測試設定

### 通用設定
每個 API 請求都需要設定：

**Headers:**
- Key: `Content-Type`
- Value: `application/json`

**Body 設定（POST/PUT 請求）:**
- 選擇 `raw`
- 右側下拉選單選擇 `JSON`

### CRUD 操作範例

#### 1. CREATE - 新增項目
```
Method: POST
URL: http://localhost:8000/wp-json/myapi/v1/items
Headers: Content-Type: application/json
Body (raw JSON):
{
    "name": "iPhone 15",
    "description": "最新款 iPhone",
    "price": 30000
}
```

#### 2. READ - 取得所有項目
```
Method: GET
URL: http://localhost:8000/wp-json/myapi/v1/items
```

#### 3. READ - 取得單筆項目
```
Method: GET
URL: http://localhost:8000/wp-json/myapi/v1/items/1
```

#### 4. UPDATE - 更新項目（支援部分更新）
```
Method: PUT
URL: http://localhost:8000/wp-json/myapi/v1/items/1
Headers: Content-Type: application/json
Body (raw JSON):
{
    "name": "iPhone 15 Pro",
    "price": 35000
}
```

#### 5. DELETE - 刪除項目
```
Method: DELETE
URL: http://localhost:8000/wp-json/myapi/v1/items/1
```

---

## 常見問題排解

### 1. API 回傳 HTML 頁面而非 JSON
- **原因**：永久連結設定問題
- **解決**：到 `設定` → `永久連結`，選擇任何非「樸素」選項並儲存

### 2. 新增/更新時回傳 "No data provided"
- **原因**：Content-Type header 未設定或 Body 格式錯誤
- **解決**：確保 Headers 有 `Content-Type: application/json`，Body 選擇 `raw` + `JSON`

### 3. 外掛啟用後找不到資料表
- **原因**：外掛可能未正確啟用
- **解決**：停用外掛後重新啟用，確保 `register_activation_hook` 執行

### 4. PHP 警告錯誤
- **解決**：在 `wp-config.php` 中加入除錯設定：
```php
define('WP_DEBUG', true);
define('WP_DEBUG_LOG', true);
define('WP_DEBUG_DISPLAY', true);
```

---

## API 端點總覽

| 操作 | Method | URL | 說明 |
|------|--------|-----|------|
| 取得所有項目 | GET | `/wp-json/myapi/v1/items` | 回傳所有項目 |
| 取得單筆項目 | GET | `/wp-json/myapi/v1/items/{id}` | 回傳指定 ID 的項目 |
| 新增項目 | POST | `/wp-json/myapi/v1/items` | 新增一筆項目 |
| 更新項目 | PUT | `/wp-json/myapi/v1/items/{id}` | 更新指定 ID 的項目 |
| 刪除項目 | DELETE | `/wp-json/myapi/v1/items/{id}` | 刪除指定 ID 的項目 |

---

## 進階功能擴展

### 1. 新增認證機制
### 2. 新增分頁功能
### 3. 新增搜尋和排序
### 4. 新增資料驗證
### 5. 新增 API 文件化

現在你已經有了一個完整的 WordPress REST API 開發環境！