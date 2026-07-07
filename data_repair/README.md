# 数据修复工作流（分析后人工处理）

`qun2_main.ipynb` 大模型跑完后会输出按意图分 sheet 的 Excel。由于合并单元格、模型输出等原因，常出现 **空值**、**「极」字污染** 等问题；多批次结果还需 **合并为版本汇总表**。本目录提供分析完成后的四步人工/半自动处理脚本。

## 处理顺序

```
qun2_main 输出 Excel
       │
       ▼
┌──────────────────┐
│ ① none_update.py │  空值补齐（对照 QQ 群 txt 回填发言时间/玩家ID/消息）
└────────┬─────────┘
         ▼
┌──────────────────┐
│ ② jiyouhua.py    │  「极」字修正（话题簇/时间/ID/消息字段纠错）
└────────┬─────────┘
         ▼
┌──────────────────┐
│ ③ merge.py       │  单文件内话题簇列（A列）连续相同值合并单元格
└────────┬─────────┘
         ▼
┌──────────────────┐
│ ④ excelmerg.py   │  多份 Excel 按 sheet 合并为最终版本文件（最后一步）
└──────────────────┘
```

## 依赖

```bash
pip install pandas openpyxl
```

（与 `qun2_main_workflow/requirements.txt` 一致。）

## 各脚本说明

### ① none_update.py — 空值补齐

**作用**：读取分析产出的 Excel 与同期 QQ 群 txt，对 `发言时间`、`玩家ID`、`玩家消息` 中的空单元格，用 txt 原始记录回填。

**匹配规则**：
- 有时间 → 按时间匹配，辅以 ID 或消息
- 无时间 → 按 ID+消息、仅 ID、或消息模糊匹配反查

**配置**（修改文件底部 `if __name__ == "__main__"`）：

```python
excel_path  = r"路径\02232群.xlsx"              # qun2_main 输出
txt_path    = r"路径\《欢迎来到地球》测试2群.txt"  # 同期群聊导出
output_path = excel_path.replace(".xlsx", "_空值补齐V2.xlsx")
```

**运行**：

```bash
cd data_repair
python none_update.py
```

**输出**：`原文件名_空值补齐V2.xlsx`

---

### ② jiyouhua.py — 「极」字修正

**作用**：模型或合并单元格偶发将内容污染为含「极」「极海听雷」等，本脚本对照 txt 修正 `话题簇`、`发言时间`、`玩家ID`、`玩家消息`，并生成修正日志。

**配置**：

```python
excel_path  = r"路径\xxx_空值补齐V2.xlsx"   # 上一步输出
txt_path    = r"路径\群聊.txt"
output_path = excel_path.replace(".xlsx", "修正.xlsx")
log_path    = excel_path.replace(".xlsx", "_含极三态修正.xlsx")  # 修正日志
```

**运行**：

```bash
python jiyouhua.py
```

**输出**：
- `xxx修正.xlsx` — 修正后的主文件
- `xxx_含极三态修正.xlsx` — 修正记录（sheet/行号/原值/新值）

---

### ③ merge.py — 单文件话题簇合并

**作用**：对单个 Excel 每个 sheet 的 **A 列（话题簇）**，将连续相同话题名的行合并为一个单元格并居中。便于人工审阅单批结果。

**配置**：

```python
INPUT_FILE  = Path(r"路径\xxx修正.xlsx")
OUTPUT_FILE = INPUT_FILE.parent / "xxx_已合并.xlsx"
```

**运行**：

```bash
python merge.py
```

**输出**：`xxx_已合并.xlsx`

---

### ④ excelmerg.py — 多文件最终合并（最后一步）

**作用**：将多份已处理 Excel（如不同日期批次）按五个意图 sheet 合并为 **一份版本汇总表**：
- `体验反馈`、`疑惑询问`、`建议灵感`、`情绪输出`、`问题反馈`
- 保留各文件内话题簇块的合并单元格结构
- 按话题簇名称排序后写入新文件

**配置**：

```python
excel_files = [
    r"路径\批次1_已合并.xlsx",
    r"路径\批次2_已合并.xlsx",
    # ... 可追加更多批次
]
output_path = r"路径\【地球】_版本名_玩家发言分类「2群」_日期范围.xlsx"
```

**运行**：

```bash
python excelmerg.py
```

**输出**：最终交付用汇总 Excel。

## 完整操作示例（测试2群）

假设 `qun2_main` 已输出 `02232群.xlsx`：

| 步骤 | 输入 | 输出 |
|------|------|------|
| ① none_update | `02232群.xlsx` + 群 txt | `02232群_空值补齐V2.xlsx` |
| ② jiyouhua | 上一步输出 + 群 txt | `02232群_空值补齐V2修正.xlsx` |
| ③ merge | 上一步输出 | `02232群_空值补齐V2修正_已合并.xlsx` |
| ④ excelmerg | 多个 `_已合并.xlsx` | 版本汇总最终 xlsx |

> 若只有单批次、不需汇总，可跳过 ④，以 ③ 的输出作为终稿。

## 注意事项

- 各脚本底部路径为 **硬编码**，每次运行前请按实际文件修改。
- txt 须与 Excel 分析时段对应的 **同一群** 导出文件。
- Excel 表头须包含：`话题簇`、`发言时间`、`玩家ID`、`玩家消息`（与 qun2_main 输出一致）。
- `merge.py` 与 `excelmerg.py` 都会处理合并单元格，建议按顺序执行，避免重复合并导致格式错乱。
