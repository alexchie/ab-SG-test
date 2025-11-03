# 綠界 ECPay 物流整合說明

## 目前狀態

✅ **已實作：模擬版本（可立即使用）**
- 7-11 和全家超商選擇
- 互動式門市選擇地圖（模擬）
- 16 家測試門市（8家7-11 + 8家全家）
- 完整的 UI/UX 流程

## 🚀 升級到真實綠界 API

### 步驟 1：申請綠界帳號

1. 前往 [綠界科技](https://www.ecpay.com.tw/)
2. 申請「特店會員」
3. 取得測試環境帳號（MerchantID）
4. 取得 HashKey 和 HashIV

### 步驟 2：設定後端伺服器

綠界 CVS 物流需要後端支援，因為需要：
- 加密參數（使用 HashKey 和 HashIV）
- 產生檢查碼（CheckMacValue）
- 處理綠界回傳資料

**推薦後端語言：**
- PHP（綠界提供官方 SDK）
- Node.js
- Python
- C# ASP.NET

### 步驟 3：整合綠界 CVS 地圖

#### 前端呼叫流程：

```javascript
function openECPayStoreMap() {
    const storeType = document.getElementById('selectedStoreType').value;
    
    // 呼叫後端 API 產生綠界表單
    fetch('/api/ecpay/create-cvs-form', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
            storeType: storeType, // UNIMART (7-11) 或 FAMI (全家)
            serverReplyURL: 'https://your-domain.com/api/ecpay/cvs-callback'
        })
    })
    .then(response => response.json())
    .then(data => {
        // 開啟綠界地圖（會彈出新視窗）
        const form = document.createElement('form');
        form.method = 'POST';
        form.action = 'https://logistics-stage.ecpay.com.tw/Express/map'; // 測試環境
        form.target = '_blank';
        
        // 添加綠界要求的參數
        Object.keys(data.params).forEach(key => {
            const input = document.createElement('input');
            input.type = 'hidden';
            input.name = key;
            input.value = data.params[key];
            form.appendChild(input);
        });
        
        document.body.appendChild(form);
        form.submit();
        document.body.removeChild(form);
    });
}
```

#### 後端範例（Node.js + Express）：

```javascript
const express = require('express');
const crypto = require('crypto');

const MERCHANT_ID = 'YOUR_MERCHANT_ID';
const HASH_KEY = 'YOUR_HASH_KEY';
const HASH_IV = 'YOUR_HASH_IV';

app.post('/api/ecpay/create-cvs-form', (req, res) => {
    const { storeType, serverReplyURL } = req.body;
    
    const params = {
        MerchantID: MERCHANT_ID,
        MerchantTradeNo: generateTradeNo(), // 訂單編號
        LogisticsType: 'CVS',
        LogisticsSubType: storeType, // UNIMART 或 FAMI
        IsCollection: 'N',
        ServerReplyURL: serverReplyURL
    };
    
    // 產生檢查碼
    params.CheckMacValue = generateCheckMacValue(params, HASH_KEY, HASH_IV);
    
    res.json({ params });
});

// 接收綠界回傳的門市資料
app.post('/api/ecpay/cvs-callback', (req, res) => {
    const { CVSStoreID, CVSStoreName, CVSAddress, CVSTelephone } = req.body;
    
    // 驗證檢查碼
    if (verifyCheckMacValue(req.body, HASH_KEY, HASH_IV)) {
        // 儲存門市資訊到 session 或資料庫
        // 返回前端更新 UI
        
        res.send('1|OK'); // 告訴綠界接收成功
    } else {
        res.send('0|ERROR');
    }
});

function generateCheckMacValue(params, hashKey, hashIV) {
    // 綠界檢查碼產生邏輯
    // 1. 參數按照 A-Z 排序
    // 2. 組成 key1=value1&key2=value2 格式
    // 3. 前後加上 HashKey 和 HashIV
    // 4. URL encode
    // 5. 轉小寫
    // 6. SHA256 加密
    // 7. 轉大寫
    
    // ... 實作省略，請參考綠界官方文件
}
```

#### 後端範例（PHP）：

```php
<?php
// 使用綠界官方 PHP SDK
require_once('ECPay.Payment.Integration.php');

$MerchantID = 'YOUR_MERCHANT_ID';
$HashKey = 'YOUR_HASH_KEY';
$HashIV = 'YOUR_HASH_IV';

$AL = new ECPayLogistics();
$AL->Send = array(
    'MerchantID' => $MerchantID,
    'MerchantTradeNo' => 'YOUR_TRADE_NO',
    'LogisticsType' => 'CVS',
    'LogisticsSubType' => 'UNIMART', // 或 FAMI
    'IsCollection' => 'N',
    'ServerReplyURL' => 'https://your-domain.com/cvs-callback.php'
);

// 產生表單 HTML
$html = $AL->CvsMap();
echo $html;
?>
```

### 步驟 4：測試流程

1. **測試環境地圖網址：**
   - https://logistics-stage.ecpay.com.tw/Express/map

2. **正式環境地圖網址：**
   - https://logistics.ecpay.com.tw/Express/map

3. **測試帳號：**
   - 向綠界申請測試環境帳號

### 支援的超商類型

| 代碼 | 超商名稱 | LogisticsSubType |
|------|----------|------------------|
| UNIMART | 7-ELEVEN | UNIMART |
| FAMI | 全家 | FAMI |
| HILIFE | 萊爾富 | HILIFE |
| OKMART | OK超商 | OKMART |

## 📚 參考資源

1. [綠界物流 API 文件](https://www.ecpay.com.tw/Service/API_Dwnld)
2. [綠界技術支援](https://www.ecpay.com.tw/Service/API_Contact)
3. [綠界測試環境](https://developers.ecpay.com.tw/)

## 💡 建議

1. **先在測試環境完整測試**
2. **注意檢查碼計算的正確性**（最常見的錯誤來源）
3. **ServerReplyURL 必須是公開可存取的網址**
4. **門市選擇完成後會關閉地圖視窗並回傳資料**
5. **建議使用 HTTPS**

## 🔧 目前模擬版本 vs 真實 API

| 功能 | 模擬版本 | 真實 ECPay |
|------|----------|-----------|
| 超商選擇 | ✅ 7-11/全家 | ✅ 7-11/全家/萊爾富/OK |
| 門市選擇 | ✅ 16家模擬門市 | ✅ 全台所有門市 |
| 互動地圖 | ✅ 模擬彈出視窗 | ✅ 真實綠界地圖 |
| 需要後端 | ❌ 純前端 | ✅ 需要後端 |
| 需要申請 | ❌ 不需要 | ✅ 需要綠界帳號 |
| 立即使用 | ✅ 是 | ❌ 需開發後端 |

## 🎯 下一步

1. 先使用目前的模擬版本測試 UI/UX
2. 確認流程符合需求後
3. 申請綠界帳號
4. 開發後端 API
5. 整合真實綠界地圖
6. 測試環境完整測試
7. 上線到正式環境

如有問題，請聯繫綠界技術支援或參考官方文件。
