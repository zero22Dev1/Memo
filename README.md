#📘 SQL & PL/SQL メモ集

業務や学習で役立つ　SQL および PL/SQL の基本構文・Tips をまとめたものです。

---

## 🔹 SQL 基本文法

```bat
wmic process where "CommandLine like '%metabase%'" call terminate
```

```sql
WHERE createdAt = COALESCE(
  [[ {{dateOfCreation}} ]],
  DATE_FORMAT(
    CASE 
      WHEN DAYOFWEEK(CURDATE()) = 2   -- 月曜なら金曜
        THEN DATE_SUB(CURDATE(), INTERVAL 3 DAY)
      ELSE DATE_SUB(CURDATE(), INTERVAL 1 DAY)  -- それ以外は前日
    END,
  '%Y%m%d')
)
```

```sql

-- 上位20テーブルの物理サイズ
SELECT table_name,
       ROUND((data_length+index_length)/1024/1024,1) AS MB
FROM information_schema.tables
WHERE table_schema = '<metabase_db_name>'
ORDER BY (data_length+index_length) DESC
LIMIT 20;

-- 日付・時刻系カラムのあるテーブル（削除/アーカイブ候補の探索）
SELECT table_name, column_name, data_type
FROM information_schema.columns
WHERE table_schema = '<metabase_db_name>'
  AND data_type IN ('datetime','timestamp','date')
ORDER BY table_name, column_name;


WITH vars AS (
  SELECT COALESCE({{date}}, DATE_FORMAT(CURDATE() - INTERVAL 1 DAY, '%Y%m%d')) AS target_date
)
SELECT *
FROM your_table, vars
WHERE your_table.target_date = vars.target_date;

-- 昨日をデフォルト値として使用するクエリ
WITH vars AS (
  SELECT DATE_FORMAT(CURDATE() - INTERVAL 1 DAY, '%Y%m%d') AS yesterday
)
SELECT *
FROM your_table, vars
WHERE target_date = COALESCE({{date}}, vars.yesterday);
```

```sql
-- 今日をデフォルト値として使用するクエリ
WITH vars AS (
  SELECT DATE_FORMAT(CURDATE(), '%Y%m%d') AS today
)
SELECT *
FROM your_table, vars
WHERE target_date = COALESCE({{date}}, vars.today);
```




### SELECT文
```sql
SELECT REPLACE(
         REPLACE(
           REPLACE(列名, CHR(13), ''), -- CR削除
         CHR(10), ''),                 -- LF削除
       CHR(9), ''                      -- TAB削除
       ) AS clean_text
FROM テーブル名;



SELECT column1, column2
FROM table_name
WHERE 条件;
```
### INSERT文 
```sql
INSERT INTO table_name (col1, col2)
VALUES ('value1', 'value2');
```
### UPDATE文

```sql
UPDATE table_name
SET col1 = 'value1'
WHERE 条件;
```

### DELETE文

```sql
DELETE FROM table_name
WHERE 条件
```

### 🔹 条件・関数の使い方
```sql
-- NULLなら空文字を返す
NVL(column, '')
--LIKE（部分一致）
SELECT * FROM table_name
WHERE column LIKE '%検索文字%';

-- 文字列の整形
TRIM(column)                -- 前後の空白除去
REPLACE(column, '-', '')   -- ハイフン削除
LOWER(column)              -- 小文字化
UPPER(column)              -- 大文字化

```

### 🔹 型変換（CAST / TO_CHAR / TO_NUMBER / TO_DATE）

```sql
--- 日付 → 文字列
TO_CHAR(date, 'YYYY-MM-DD')

--- 数値 → 文字列
TO_CHAR(number)

--- 文字列 → 数値
TO_NUMBER('123')

-- 文字列 → 日付
TO_DATE('2025-04-01', 'YYYY-MM-DD')

--- 明示的な型変換
CAST(column AS VARCHAR2(10))
```

### 🔹 条件式と論理
```sql

---  IN 句（複数条件）
SELECT *
FROM employees
WHERE department_id IN (10, 20, 30);

--- count関数について
--- COUNT(*)：NULL含む「全行」カウント
--- COUNT(col)：NULL以外の「値あり」だけをカウント



--- NOT IN

SELECT *
FROM employees
WHERE department_id NOT IN (40, 50);

--- CASE
SELECT employee_id,
       salary,
       CASE 
         WHEN salary >= 10000 THEN '高給'
         WHEN salary >= 5000 THEN '中給'
         ELSE '低給'
       END AS salary_rank
FROM employees;

```

```sql

--- •	SUBSTR(..., 開始位置, 長さ) で 長さ を指定しなかった場合：
--- → その開始位置から文字列の最後までをすべて取ってしまう
DECLARE
  original_str VARCHAR2(4000) := '...';  -- 312文字以上の文字列を指定
  part1 VARCHAR2(125);
  part2 VARCHAR2(125);
  part3 VARCHAR2(125);
BEGIN
  IF LENGTH(original_str) >= 1 THEN
    part1 := SUBSTR(original_str, 1, 125);
  END IF;

  IF LENGTH(original_str) >= 126 THEN
    part2 := SUBSTR(original_str, 126, 125);
  END IF;

  IF LENGTH(original_str) >= 251 THEN
    part3 := SUBSTR(original_str, 251, 125);
  END IF;

  DBMS_OUTPUT.PUT_LINE('part1: ' || part1);
  DBMS_OUTPUT.PUT_LINE('part2: ' || part2);
  DBMS_OUTPUT.PUT_LINE('part3: ' || part3);
END;

```

## 🔹 副問い合わせ（サブクエリ）

副問い合わせ（subquery）は、あるSQL文の中で別のSELECT文を使う構文です。  
結果を元に条件判定したり、値を取得したりできます。

---

### 📌 種類別サブクエリの例

---

### ✅ スカラーサブクエリ（単一行・単一列）

```sql
SELECT employee_id, 
       (SELECT department_name 
        FROM departments 
        WHERE departments.department_id = employees.department_id) AS dept_name
FROM employees;
```
### ✅ WHERE句でのサブクエリ（単一行）
外側のクエリの 条件 に利用

結果は1行1列に限定される


```sql
SELECT employee_id, salary
FROM employees
WHERE salary > (
  SELECT AVG(salary)
  FROM employees
);

```
### ✅ IN句を使う複数行サブクエリ
複数行が返るサブクエリ用に IN, EXISTS を使う
```sql
SELECT employee_id, department_id
FROM employees
WHERE department_id IN (
  SELECT department_id
  FROM departments
  WHERE location_id = 1700
);

```
### ✅ EXISTS（存在チェック）を使った相関サブクエリ
相関サブクエリ（外側の値を参照する）
あるデータの「存在有無」をチェック
```sql
SELECT d.department_name
FROM departments d
WHERE EXISTS (
  SELECT 1
  FROM employees e
  WHERE e.department_id = d.department_id
);
```
###✅ FROM句で使うサブクエリ（インラインビュー）
集計・加工を事前に済ませてメインクエリで利用
```sql

SELECT department_id, avg_salary
FROM (
  SELECT department_id, AVG(salary) AS avg_salary
  FROM employees
  GROUP BY department_id
)
WHERE avg_salary > 10000;
```

###✅ WITH句（共通表式、サブクエリの再利用）

複数回使うサブクエリを名前付きで定義・再利用できる
ネストが深くなりにくく、読みやすい

```sql
WITH dept_avg AS (
  SELECT department_id, AVG(salary) AS avg_salary
  FROM employees
  GROUP BY department_id
)
SELECT e.employee_id, e.salary, d.avg_salary
FROM employees e
JOIN dept_avg d
  ON e.department_id = d.department_id;

```
###特定の文字を含む

```sql

SELECT
  SUM(CASE WHEN name LIKE 'A%' THEN 1 ELSE 0 END) AS Aで始まる件数,
  SUM(CASE WHEN name LIKE '%B%' THEN 1 ELSE 0 END) AS Bを含む件数
FROM users;

SELECT
  COUNT(CASE WHEN name LIKE 'A%' THEN 1 END) AS Aで始まる件数,
  COUNT(CASE WHEN name LIKE '%B%' THEN 1 END) AS Bを含む件数
FROM users;

```

``` sql
-- 全ロックを表示: ロックしているもの
SELECT DO.OWNER || '.' || DO.OBJECT_NAME AS OBJECT,
  DECODE(LO.LOCKED_MODE,
     1, 'NULL',
     2, '行共有(SS)',
     3, '行排他(SX)',
     4, '共有(S)',
     5, '共有行排他(SRX)',
     6, '排他(X)',
     '???' ) AS LOCKED_MODE,
  TO_CHAR(L.CTIME / 60,'99990.9') AS MIN,
  LO.ORACLE_USERNAME,
  LO.OS_USER_NAME,
  LO.PROCESS,
  S.SID,
  S.SERIAL#
FROM DBA_OBJECTS DO, V$LOCKED_OBJECT LO, V$LOCK L, V$SESSION S
WHERE DO.OBJECT_ID = LO.OBJECT_ID AND
  LO.SESSION_ID = L.SID AND
  LO.XIDSQN = L.ID2 AND
  LO.PROCESS = S.PROCESS AND
  LO.XIDUSN > 0;
```



-- SESSIONをKILLする: [例] SID = 177, SERIA#L = 4335 の場合

``` sql
ALTER SYSTEM KILL SESSION '177,4335';

-- アカウントの状態を確認
SELECT username, account_status
FROM dba_users
WHERE username = 'ユーザー名';

-- アカウントのロック解除
ALTER USER ユーザー名 ACCOUNT UNLOCK;
```


```bat
@echo off
REM ===============================
REM MySQL80 サービス再起動バッチ
REM ===============================

echo MySQL80 サービスを停止します...
net stop MySQL80

echo.
echo サービスが完全に停止するまで待機中...
timeout /t 5 > nul

echo.
echo MySQL80 サービスを起動します...
net start MySQL80

pause
```

```bat
@echo off
REM ===============================
REM MySQL80 サービス停止バッチ
REM ===============================

sc query MySQL80 | findstr /i "STOPPED" > nul
if %errorlevel%==0 (
    echo MySQL80 サービスはすでに停止しています。
) else (
    echo MySQL80 サービスを停止します...
    net stop MySQL80
)

pause
```


```bat
@echo off
REM ===============================
REM MySQL80 サービス開始バッチ
REM ===============================

sc query MySQL80 | findstr /i "RUNNING" > nul
if %errorlevel%==0 (
    echo MySQL80 サービスはすでに起動しています。
) else (
    echo MySQL80 サービスを起動します...
    net start MySQL80
)

pause
```
## MySQL 関連サービスの状態を確認するコマンド
```bat
sc query state= all | findstr /i "mysql"
```
```bat
@echo off
REM ===============================
REM MySQL84 サービス開始バッチ
REM ===============================

sc query MySQL84 | findstr /i "RUNNING" > nul
if %errorlevel%==0 (
    echo MySQL84 サービスはすでに起動しています。
) else (
    echo MySQL84 サービスを起動します...
    net start MySQL84
)

pause
```
```bat
@echo off
REM ===============================
REM MySQL84 サービス停止バッチ
REM ===============================

sc query MySQL84 | findstr /i "STOPPED" > nul
if %errorlevel%==0 (
    echo MySQL84 サービスはすでに停止しています。
) else (
    echo MySQL84 サービスを停止します...
    net stop MySQL84
)

pause
```

```bat
@echo off
REM ===============================
REM MySQL84 サービス再起動バッチ
REM ===============================

echo MySQL84 サービスを停止します...
net stop MySQL84

echo.
echo サービスが完全に停止するまで待機中...
timeout /t 5 > nul

echo.
echo MySQL84 サービスを起動します...
net start MySQL84

pause
```


