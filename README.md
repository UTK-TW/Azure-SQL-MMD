# Azure SQL 動態資料遮罩快速實作指南

## 概述
本指南提供 Azure SQL Database 動態資料遮罩功能的完整實作流程，包含表格建立、遮罩設定、權限管理等相關指令。

## 🚀 快速開始

### 1. 建立測試表格
```sql
-- 建立用戶基本資料表 (適用於動態資料遮罩測試)
CREATE TABLE UserProfileTest (
    -- 主鍵 - 不需要遮罩
    UserID INT IDENTITY(1,1) PRIMARY KEY,
    
    -- 一般資訊 - 不需要遮罩
    UserName NVARCHAR(50) NOT NULL,
    Age INT,
    Gender NVARCHAR(10),
    Country NVARCHAR(50),
    City NVARCHAR(50),
    
    -- 需要遮罩的敏感資訊
    Email NVARCHAR(100) MASKED WITH (FUNCTION = 'email()'),
    Phone NVARCHAR(20) MASKED WITH (FUNCTION = 'partial(1,"XXX-XXX-",4)'),
    IDNumber NVARCHAR(20) MASKED WITH (FUNCTION = 'partial(2,"XXXXXXXX",2)'),
    CreditCardNumber NVARCHAR(20) MASKED WITH (FUNCTION = 'default()'),
    Salary DECIMAL(10,2) MASKED WITH (FUNCTION = 'random(10000, 100000)'),
    BankAccount NVARCHAR(20) MASKED WITH (FUNCTION = 'default()'),
    Address NVARCHAR(200) MASKED WITH (FUNCTION = 'partial(5,"XXXXXXXXXXXXX",0)'),
    
    -- 建立時間 - 不需要遮罩
    CreatedDate DATETIME2 DEFAULT GETDATE(),
    LastLoginDate DATETIME2
);
```

### 2. 插入測試資料
```sql
INSERT INTO UserProfileTest (UserName, Age, Gender, Country, City, Email, Phone, IDNumber, CreditCardNumber, Salary, BankAccount, Address, LastLoginDate)
VALUES 
('張小明', 28, '男', '台灣', '台北', 'zhang.ming@example.com', '0912-345-678', 'A123456789', '1234-5678-9012-3456', 75000.00, '1234567890123456', '台北市信義區信義路100號', '2024-01-15 10:30:00'),
('李小華', 32, '女', '台灣', '高雄', 'lee.hua@example.com', '0987-654-321', 'B987654321', '9876-5432-1098-7654', 85000.00, '9876543210987654', '高雄市前鎮區中華五路200號', '2024-01-16 14:20:00'),
('王大偉', 45, '男', '台灣', '台中', 'wang.wei@example.com', '0923-456-789', 'C456789123', '4567-8901-2345-6789', 95000.00, '4567890123456789', '台中市西屯區台灣大道300號', '2024-01-17 09:45:00');
```

### 3. 設定使用者權限
```sql
-- 授予使用者 UTK_GM_MEMBER 查詢權限 (會看到遮罩資料)
GRANT SELECT ON UserProfileTest TO [UTK_GM_MEMBER];

-- 如果需要建立使用者
IF NOT EXISTS (SELECT * FROM sys.database_principals WHERE name = 'UTK_GM_MEMBER')
BEGIN
    CREATE USER [UTK_GM_MEMBER] WITHOUT LOGIN;
END
```

## 🔧 現有表格遮罩設定

### 為現有表格新增遮罩
```sql
-- MEMBER 表格
ALTER TABLE MEMBER ALTER COLUMN ID_NBR ADD MASKED WITH (FUNCTION = 'partial(2,"XXXXXXXX",2)');
ALTER TABLE MEMBER ALTER COLUMN HOME_TEL_NO ADD MASKED WITH (FUNCTION = 'partial(1,"XXX-XXX-",4)');
ALTER TABLE MEMBER ALTER COLUMN HOME_TEL_NO2 ADD MASKED WITH (FUNCTION = 'partial(1,"XXX-XXX-",4)');
ALTER TABLE MEMBER ALTER COLUMN EMAIL ADD MASKED WITH (FUNCTION = 'email()');
ALTER TABLE MEMBER ALTER COLUMN ADDRESS ADD MASKED WITH (FUNCTION = 'partial(5,"XXXXXXXXXXXXX",0)');
ALTER TABLE MEMBER ALTER COLUMN MOBILE ADD MASKED WITH (FUNCTION = 'partial(1,"XXX-XXX-",4)');

-- CUSTOMER 表格
ALTER TABLE CUSTOMER ALTER COLUMN ID_NBR ADD MASKED WITH (FUNCTION = 'partial(2,"XXXXXXXX",2)');
ALTER TABLE CUSTOMER ALTER COLUMN MOBILE ADD MASKED WITH (FUNCTION = 'partial(1,"XXX-XXX-",4)');
ALTER TABLE CUSTOMER ALTER COLUMN EMAIL ADD MASKED WITH (FUNCTION = 'email()');
ALTER TABLE CUSTOMER ALTER COLUMN ADDRESS ADD MASKED WITH (FUNCTION = 'partial(5,"XXXXXXXXXXXXX",0)');
ALTER TABLE CUSTOMER ALTER COLUMN CONTACT_PHONE ADD MASKED WITH (FUNCTION = 'partial(1,"XXX-XXX-",4)');
ALTER TABLE CUSTOMER ALTER COLUMN CONTACT_EMAIL ADD MASKED WITH (FUNCTION = 'email()');

-- USERS 表格
ALTER TABLE USERS ALTER COLUMN EMAIL ADD MASKED WITH (FUNCTION = 'email()');
```

## 🔑 權限管理

### 增加 UNMASK 權限 (可看到原始資料)
```sql
-- 單一使用者
GRANT UNMASK TO [admin_user];

-- 批量使用者
GRANT UNMASK TO [user1];
GRANT UNMASK TO [user2];
GRANT UNMASK TO [user3];
```

### 移除 UNMASK 權限 (只能看到遮罩資料)
```sql
-- 單一使用者
REVOKE UNMASK FROM [regular_user];

-- 批量使用者
REVOKE UNMASK FROM [user1];
REVOKE UNMASK FROM [user2];
```

## 🧪 測試與驗證

### 測試遮罩效果
```sql
-- 以受限使用者身分查詢 (看到遮罩資料)
EXECUTE AS USER = 'UTK_GM_MEMBER';
SELECT * FROM UserProfileTest;
REVERT;

-- 查看目前使用者權限
SELECT 
    CASE 
        WHEN HAS_PERMS_BY_NAME(NULL, NULL, 'UNMASK') = 1 
        THEN '有 UNMASK 權限 - 可以看到原始資料'
        ELSE '沒有 UNMASK 權限 - 只能看到遮罩資料'
    END AS permission_status;
```

### 查看遮罩設定
```sql
-- 查看所有遮罩欄位
SELECT 
    t.name AS table_name,
    c.name AS column_name,
    c.masking_function
FROM sys.masked_columns c
JOIN sys.tables t ON c.object_id = t.object_id
ORDER BY t.name, c.name;

-- 查看 UNMASK 權限
SELECT 
    p.permission_name,
    p.state_desc,
    pr.name AS principal_name,
    pr.type_desc AS principal_type
FROM sys.database_permissions p
JOIN sys.database_principals pr ON p.grantee_principal_id = pr.principal_id
WHERE p.permission_name = 'UNMASK'
ORDER BY pr.name;
```

## 🔄 緊急還原指令

### 移除所有遮罩設定
```sql
-- UserProfileTest 表格還原
ALTER TABLE UserProfileTest ALTER COLUMN Email DROP MASKED;
ALTER TABLE UserProfileTest ALTER COLUMN Phone DROP MASKED;
ALTER TABLE UserProfileTest ALTER COLUMN IDNumber DROP MASKED;
ALTER TABLE UserProfileTest ALTER COLUMN CreditCardNumber DROP MASKED;
ALTER TABLE UserProfileTest ALTER COLUMN Salary DROP MASKED;
ALTER TABLE UserProfileTest ALTER COLUMN BankAccount DROP MASKED;
ALTER TABLE UserProfileTest ALTER COLUMN Address DROP MASKED;

-- 現有表格還原
-- MEMBER 表格
ALTER TABLE MEMBER ALTER COLUMN ID_NBR DROP MASKED;
ALTER TABLE MEMBER ALTER COLUMN HOME_TEL_NO DROP MASKED;
ALTER TABLE MEMBER ALTER COLUMN HOME_TEL_NO2 DROP MASKED;
ALTER TABLE MEMBER ALTER COLUMN EMAIL DROP MASKED;
ALTER TABLE MEMBER ALTER COLUMN ADDRESS DROP MASKED;
ALTER TABLE MEMBER ALTER COLUMN MOBILE DROP MASKED;

-- CUSTOMER 表格
ALTER TABLE CUSTOMER ALTER COLUMN ID_NBR DROP MASKED;
ALTER TABLE CUSTOMER ALTER COLUMN MOBILE DROP MASKED;
ALTER TABLE CUSTOMER ALTER COLUMN EMAIL DROP MASKED;
ALTER TABLE CUSTOMER ALTER COLUMN ADDRESS DROP MASKED;
ALTER TABLE CUSTOMER ALTER COLUMN CONTACT_PHONE DROP MASKED;
ALTER TABLE CUSTOMER ALTER COLUMN CONTACT_EMAIL DROP MASKED;

-- USERS 表格
ALTER TABLE USERS ALTER COLUMN EMAIL DROP MASKED;
```

### 自動化還原
```sql
-- 一鍵還原所有遮罩
DECLARE @sql NVARCHAR(MAX) = '';
SELECT @sql = @sql + 'ALTER TABLE ' + t.name + ' ALTER COLUMN ' + c.name + ' DROP MASKED;' + CHAR(13)
FROM sys.masked_columns c
JOIN sys.tables t ON c.object_id = t.object_id;

PRINT @sql;
-- 檢查生成的指令無誤後，取消註解執行
-- EXEC sp_executesql @sql;
```

## 📋 遮罩效果預覽

| 欄位類型 | 原始資料 | 遮罩後顯示 |
|---------|---------|-----------|
| Email | zhang.ming@example.com | aXXX@XXXX.com |
| Phone | 0912-345-678 | 0XXX-XXX-678 |
| IDNumber | A123456789 | A1XXXXXXXX89 |
| CreditCard | 1234-5678-9012-3456 | XXXX |
| Salary | 75000.00 | 隨機數值 (10000-100000) |
| BankAccount | 1234567890123456 | XXXX |
| Address | 台北市信義區信義路100號 | 台北市XXXXXXXXXXXXX |

## ⚠️ 注意事項

1. **備份**: 執行前務必備份資料庫
2. **測試**: 先在測試環境中完整測試
3. **權限**: 確保執行用戶有 `ALTER` 權限
4. **監控**: 執行後監控應用程式運作狀況
5. **文件**: 記錄執行時間和負責人員

## 🛠️ 故障排除

- **遮罩未生效**: 檢查使用者是否有 UNMASK 權限
- **權限錯誤**: 確認執行用戶有足夠權限
- **應用程式錯誤**: 檢查應用程式是否適應遮罩後的資料格式
