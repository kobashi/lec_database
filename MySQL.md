---
layout: home
permalink: /
---
# MySQL集計関数の使用方法

## 1. 集計関数の基本（式を与えずに使用）
集計関数（`COUNT`、`SUM`、`AVG`、`MAX`、`MIN`など）は複数の行を1つの結果に集約します。引数に列名を直接指定するのが基本です。

### 例: 社員テーブルの給料を集計
```sql
SELECT 
    COUNT(*) AS employee_count, -- 全行数
    SUM(salary) AS total_salary, -- 給料合計
    AVG(salary) AS avg_salary, -- 給料平均
    MAX(salary) AS max_salary, -- 最高給料
    MIN(salary) AS min_salary -- 最低給料
FROM employees;
```

- **特徴**: 列名（例: `salary`）や`*`を直接指定。
- **注意**: `NULL`は集計対象外（`COUNT(*)`は全行をカウント）。

## 2. 集計関数に式を指定
引数に計算式や関数を指定し、加工した値を集計できます。

### 例: ボーナスや税金を考慮した集計
```sql
SELECT 
    SUM(salary * 1.1) AS total_with_bonus, -- 10%ボーナス加算
    AVG(salary * 0.8) AS avg_after_tax -- 20%税金控除後の平均
FROM employees;
```

- **用途**: 割引、税計算、単位変換後の集計。

## 3. 比較演算子を使用した集計とCASEの省略
MySQLでは、比較演算子（`>`、`<`、`=`など）は真なら`1`、偽なら`0`に評価されます。この特性を利用すると、`CASE`式を省略して条件付き集計を簡潔に記述できます。

### 例1: CASEを使用した条件集計
```sql
SELECT 
    COUNT(CASE WHEN salary > 50000 THEN 1 END) AS high_salary_count, -- 給料50,000超の人数
    SUM(CASE WHEN department = 'IT' THEN salary ELSE 0 END) AS it_salary_total -- IT部門の給料合計
FROM employees;
```

### 例2: 比較演算子でCASEを省略
```sql
SELECT 
    SUM(salary > 50000) AS high_salary_count, -- 給料50,000超の人数（1 or 0でカウント）
    SUM(salary * (department = 'IT')) AS it_salary_total -- IT部門の給料合計
FROM employees;
```

- **仕組み**:
  - `salary > 50000`: 真なら`1`、偽なら`0`を返し、`SUM`で`1`の総和を計算（人数カウント）。
  - `department = 'IT'`: 真なら`1`、偽なら`0`を返し、`salary * 1`で該当行の給料を加算。
- **利点**: `CASE`を省略することでクエリが簡潔。
- **注意**: 可読性が低下する場合があるため、複雑な条件では`CASE`推奨。

## 4. CASE式を用いた集計
`CASE`式は複雑な条件分岐に適しており、複数の条件を柔軟に集計可能。

### 例: 給料範囲ごとの人数
```sql
SELECT 
    COUNT(CASE WHEN salary < 30000 THEN 1 END) AS low_salary_count,
    COUNT(CASE WHEN salary BETWEEN 30000 AND 60000 THEN 1 END) AS mid_salary_count,
    COUNT(CASE WHEN salary > 60000 THEN 1 END) AS high_salary_count
FROM employees;
```

- **特徴**: 範囲や複数条件を明示的に指定。
- **省略可能性**: 単純な比較なら以下のように簡略化可能。
  ```sql
  SELECT 
      SUM(salary < 30000) AS low_salary_count,
      SUM(salary BETWEEN 30000 AND 60000) AS mid_salary_count,
      SUM(salary > 60000) AS high_salary_count
  FROM employees;
  ```

## 5. WINDOW関数としてOVER節を指定
集計関数を`OVER`節と組み合わせると、行ごとに集計結果を保持しつつ元データを維持できます。

### 例1: 部署ごとの給料合計
```sql
SELECT 
    employee_id,
    department,
    salary,
    SUM(salary) OVER (PARTITION BY department) AS dept_salary_total
FROM employees;
```

- **説明**: `PARTITION BY department`で部署ごとに集計し、各行に合計を付加。

### 例2: 累積給料
```sql
SELECT 
    employee_id,
    hire_date,
    salary,
    SUM(salary) OVER (ORDER BY hire_date) AS cumulative_salary
FROM employees;
```

- **説明**: `ORDER BY hire_date`で入社日順に累積計算。

### 例3: 部署内ランキング
```sql
SELECT 
    employee_id,
    department,
    salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS salary_rank
FROM employees;
```

- **説明**: 部署内で給料順にランキング。

## サンプルデータでの動作確認
```sql
CREATE TABLE employees (
    employee_id INT,
    department VARCHAR(50),
    salary DECIMAL(10, 2),
    hire_date DATE
);
INSERT INTO employees VALUES
(1, 'IT', 60000, '2023-01-01'),
(2, 'IT', 55000, '2023-02-01'),
(3, 'HR', 45000, '2023-01-15'),
(4, 'HR', 70000, '2023-03-01');
```

### 比較演算子省略の例
```sql
SELECT 
    SUM(salary > 50000) AS high_salary_count, -- 給料50,000超の人数
    SUM(salary * (department = 'IT')) AS it_salary_total -- IT部門の給料合計
FROM employees;
```
**結果**:
```
high_salary_count | it_salary_total
2                 | 115000
```

## まとめ
- **式なし集計**: 列名を直接指定。
- **式あり集計**: 計算式でデータ加工。
- **比較演算子とCASE**:
  - 比較演算子は`0`/`1`に評価され、`CASE`を省略可能（例: `SUM(salary > 50000)`）。
  - 複雑な条件や可読性重視なら`CASE`使用。
- **WINDOW関数**: `OVER`節でグループ集計やランキングを各行に付加。
- **省略の利点**: クエリが簡潔になるが、複雑な場合は`CASE`で明確化。
