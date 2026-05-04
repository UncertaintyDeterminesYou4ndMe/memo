

## 1. 启动 DuckDB

你可以用 Python、命令行（duckdb shell）或者其他支持 DuckDB 的工具。这里以 Python 为例，命令行类似。

```python
import duckdb
con = duckdb.connect('qa_pairs.duckdb')
```

---

## 2. 读取 JSON 文件

DuckDB 支持直接读取目录下的所有 JSON 文件（需要 DuckDB 0.7.0+）。假设每个 JSON 文件是一条记录（即每个文件是一个对象），你可以这样做：

```sql
CREATE TABLE qa_pairs AS
SELECT * FROM read_json_auto('./qa_pair/*.json');
```

**说明：**
- `read_json_auto` 会自动推断 JSON 结构并合并所有文件。
- `*.json` 会匹配目录下所有 JSON 文件。

如果你用 Python，可以这样：

```python
con.execute("""
    CREATE TABLE qa_pairs AS
    SELECT * FROM read_json_auto('./qa_pair/*.json')
""")
```

---

## 3. 检查表结构

```sql
DESCRIBE qa_pairs;
SELECT * FROM qa_pairs LIMIT 5;
```

---

## 4. 查询分析

现在你可以像普通表一样分析这些数据了：

```sql
SELECT question, answer FROM qa_pairs WHERE question LIKE '%duckdb%';
```

---

## 5. 进阶：JSON 文件结构复杂怎么办？

- 如果每个 JSON 文件是一个数组（比如每个文件是一个 list），用 `read_json_auto` 也能自动展开。
- 如果字段类型不一致，可以用 `read_json` 并手动指定 schema。

---

## 6. 其他注意事项

- DuckDB 读取 JSON 时性能很高，适合批量导入。
- 如果 JSON 文件特别大，建议分批导入或先用 Python 预处理。

---

## 总结

**最简单的命令行方案：**

```sql
CREATE TABLE qa_pairs AS
SELECT * FROM read_json_auto('./qa_pair/*.json');
```

**Python 方案：**

```python
import duckdb
con = duckdb.connect('qa_pairs.duckdb')
con.execute("""
    CREATE TABLE qa_pairs AS
    SELECT * FROM read_json_auto('./qa_pair/*.json')
""")
```

---

### 1. 展示更多行

默认只显示前200行，可以用 `LIMIT` 控制：

```sql
SELECT * FROM qa_pairs LIMIT 1000;
```

### 2. 展示更宽的内容（不省略长字符串）

DuckDB shell 默认会截断每列的显示宽度。你可以用 `.mode` 和 `.maxcolwidth` 命令调整[有误，与实际不符]：

#### 查看帮助

```sql
.help
```

#### 设置最大列宽

```sql
.maxcolwidth 1000 [有误，与实际不符]
```
这会把每列最大显示宽度设为1000字符（你可以根据需要调整）。

#### 关闭省略（显示完整内容）

目前 DuckDB shell 没有“完全不省略”的选项，但把 `.maxcolwidth` 设得足够大就能显示绝大多数内容。

### 3. 导出为文件查看

如果你想**完全不省略**，可以把结果导出为 CSV 或 Parquet 文件，然后用文本编辑器或其他工具查看：

```sql
COPY (SELECT * FROM qa_pairs) TO 'qa_pairs.csv' (HEADER, DELIMITER ',');
```

或者

```sql
COPY (SELECT * FROM qa_pairs) TO 'qa_pairs.parquet';
```

---

## 总结

- 用 `.maxcolwidth 1000` 增大每列显示宽度，减少省略。[有误，与实际不符]
- 用 `LIMIT` 控制显示行数。
- 用 `COPY` 导出完整数据到文件查看。

---

**示例：**

```sql
.maxcolwidth 1000 [有误，与实际不符]
SELECT * FROM qa_pairs LIMIT 10;
```
