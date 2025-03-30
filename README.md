# sql_test

## Task

- 回傳以下未開立的資料欄位結果: id，invoice_number，track，year，month，begin_number，end_number
## Final result
線上模擬query: https://onecompiler.com/mysql/43cqycdhz
![final_result](https://github.com/Shih-Lun-Huang/sql_test/blob/main/img/%E6%88%AA%E5%9C%96%202025-03-30%2017.08.32.png)

## 作法

1. 建立所需資料表:發票本(invoice_books)、每張發票開立的紀錄表(invoices)。
```SQL
-- 建立資料表
CREATE TABLE invoice_books (
    id INT AUTO_INCREMENT PRIMARY KEY,
    track VARCHAR(2) NOT NULL,              -- 發票字軌（AA、AB、AC）
    begin_number INT NOT NULL,              -- 發票起始號碼
    end_number INT NOT NULL,                -- 發票結束號碼
    year INT NOT NULL,                      -- 發票年份（民國113年 > 2024）
    month INT NOT NULL,                     -- 發票月份（03）
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,  
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE(track, begin_number, end_number)  -- 確保發票區間不重複
);

CREATE TABLE invoices (
    id INT AUTO_INCREMENT PRIMARY KEY,
    invoice_number VARCHAR(13) NOT NULL UNIQUE,  -- 發票號碼（例如：AA-12345678）
    invoice_date DATE NOT NULL,                  -- 開立日期
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,  
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```
2. 新增各資料表模擬資料共168筆，並刪除部分invoices資料模擬發票跳號: 
   - AA發票本刪除尾號5-8。
   - AB發票本刪除尾號8-13。
   - AC發票本刪除尾號67-77。
```SQL
-- 新增模擬資料

INSERT INTO invoice_books (track, begin_number, end_number, year, month)
VALUES 
('AA', 12345600, 12345649, 113, 03),
('AB', 98765400, 98765449, 113, 03),
('AC', 45678900, 45678999, 113, 03);

INSERT INTO invoices (invoice_number, invoice_date)
VALUES 
('AA-12345600', '2024-03-01'),
('AA-12345601', '2024-03-01'),
('AA-12345602', '2024-03-01'),
('AA-12345603', '2024-03-01'),
('AA-12345604', '2024-03-02'), -- 跳過5-8
('AA-12345609', '2024-03-02'),
('AA-12345610', '2024-03-02'),
('AA-12345611', '2024-03-02'),
...
```

3. 透過`SQL`找出哪些發票跳號:

```SQL
WITH RECURSIVE invoice_series AS (  -- 遞迴 CTE invoice_series，產生所有可能的發票號碼：
    SELECT id, track, begin_number AS num, end_number, year, month FROM invoice_books
    UNION ALL
    SELECT id, track, num + 1, end_number, year, month FROM invoice_series WHERE num < end_number
)

SELECT 
    isr.id AS id, 
    CONCAT(isr.track, '-', LPAD(isr.num, 8, '0')) AS invoice_number,
    isr.track, 
    isr.year, 
    isr.month, 
    (SELECT begin_number FROM invoice_books WHERE track = isr.track LIMIT 1) AS begin_number, 
    (SELECT end_number FROM invoice_books WHERE track = isr.track LIMIT 1) AS end_number
FROM invoice_series isr
LEFT JOIN invoices inv ON CONCAT(isr.track, '-', LPAD(isr.num, 8, '0')) = inv.invoice_number  -- 過濾出沒有出現在 invoices 表中的號碼
WHERE inv.invoice_number IS NULL AND NOT (isr.track = 'AC' AND isr.num >= 45678988)  -- 排除 AC 軌跡號 >= 45678988 的發票
ORDER BY isr.track, isr.num;
```

p.s. 有考慮直接從 `invoices` 表中抓取最後一筆發票號碼，並將其之後的所有發票號碼視為空號。但這種方法可能會 **錯誤地將實際為跳號的發票視為正常的空號**。

例如，假設題目規定 **45678990 之後的發票才算空號**，但 `invoices` 表中最後一筆記錄的發票號碼是 `45678988`，那麼該方法會將 `45678989` 和 `45678990` 都視為正常的空號。然而，若 `45678989` 和 `45678990` 只是因為發票號碼跳號而缺失，這種處理方式將導致誤判，影響數據的準確性。
