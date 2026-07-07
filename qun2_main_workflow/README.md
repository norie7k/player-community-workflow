# qun2_main 玩家发言分类工作流（测试2群）

本目录从 `玩家发言分类（供研发侧）` 中单独抽取 `qun2_main.ipynb` 及其依赖文件，按用途分类整理，便于研发侧独立运行与交付。

## 目录结构

```
qun2_main_工作流打包/
├── qun2_main.ipynb          # 主流程 Notebook（数据处理 + 大模型四阶段分类）
├── requirements.txt         # Python 依赖
├── README.md                # 本说明
├── scripts/                 # 核心 Python 模块
│   ├── data_processing.py   # QQ 群聊 txt 解析、客服映射、JSONL 构建
│   └── model_classifyV1_Copy1.py  # 模型调用、Excel 写入、话题簇白名单、后处理
├── prompts/                 # 大模型 System 提示词
│   ├── 提示词1.md           # 模型#1：筛除非游戏相关发言
│   ├── 提示词2.md           # 模型#2：五类意图分类
│   ├── 提示词5.md           # 模型#3：话题簇标注
│   └── 话提簇校正提示词.md   # 模型#4：话题簇命名校正（对照白名单）
├── data/                    # 输入数据与运行时状态
│   ├── 《欢迎来到地球》测试2群.txt  # QQ 群原始导出聊天记录
│   ├── mapping地球2.xlsx      # 客服昵称映射表（sheet: 昵称映射）
│   └── 话提簇白名单q2.jsonl   # 话题簇白名单（运行中自动追加）
└── output/                  # 分类结果输出
    └── 02232群.xlsx         # 示例输出（按意图分 sheet 的 Excel）
```

## 工作流说明

`qun2_main.ipynb` 针对 **测试2群** 在指定时间窗口内的玩家发言，执行以下流水线：

| 阶段 | 模块/提示词 | 作用 |
|------|-------------|------|
| 0. 数据预处理 | `data_processing.py` | 解析 txt → 过滤无效消息 → 映射客服/研发 → 输出 JSONL |
| 1. 相关性筛选 | `提示词1.md` | 保留与游戏相关的发言 |
| 2. 意图分类 | `提示词2.md` | 分为体感反馈、疑惑询问、玩家建议、情绪输出、问题反馈等 |
| 3. 话题簇 | `提示词5.md` | 为每条发言生成话题簇与描述 |
| 4. 话题簇校正 | `话提簇校正提示词.md` + `话提簇白名单q2.jsonl` | 统一命名，新簇写入白名单 |
| 5. 后处理 | `postprocess_excel_by_topic` | 按时间间隔合并同话题发言，写入 Excel |

默认分析时间范围（可在 Notebook 中修改）：

- 开始：`2026-02-14 00:00:00`
- 结束：`2026-02-24 00:00:00`

默认研发人员 ID 映射（`speaker_map`）：

| QQ ID | 备注 |
|-------|------|
| 16186514 | peter本尊 |
| 1655611808 | 运营绾绾 |
| 2073820674 | 沙利文老师 |
| 2726067525 | milissa |

## 环境准备

```bash
cd qun2_main_工作流打包
pip install -r requirements.txt
```

在 Jupyter / VS Code 中打开 `qun2_main.ipynb`，**工作目录需为本打包根目录**（包含 `scripts/`、`data/`、`prompts/` 的文件夹）。

## 运行前配置

在 Notebook「设置参数」单元格中修改：

1. **API 配置**（火山引擎 Ark）
   - `API_URL`
   - `API_KEY`（建议使用环境变量，勿提交到版本库）
   - `V3_MODEL_ID` / `V3_1_MODEL_ID` / `R1_MODEL_ID`（各阶段接入点）

2. **批处理参数**
   - `BATCH_SIZE`：每批条数（默认 200）
   - `SLEEP_BETWEEN`：批次间隔秒数（防限流）
   - `TEMPERATURE` / `MAX_TOKENS` / `TIMEOUT_SEC`

3. **数据范围**
   - `start_time` / `end_time`
   - 替换 `data/` 下 txt 或 mapping 文件时，同步修改对应路径变量

## 输出说明

- 主输出：`output/02232群.xlsx`
- 按 **意图分类** 分 sheet 写入，含发言时间、玩家 ID、消息、分类标签、话题簇等字段
- `data/话提簇白名单q2.jsonl` 会在运行过程中增量更新，下次运行可复用已有话题簇命名

## 文件来源对照

| 本包路径 | 原始位置 |
|----------|----------|
| `qun2_main.ipynb` | `玩家发言分类（供研发侧）/qun2_main.ipynb` |
| `scripts/*.py` | 同目录下 `data_processing.py`、`model_classifyV1_Copy1.py` |
| `prompts/*.md` | 同目录下对应提示词文件 |
| `data/《欢迎来到地球》测试2群.txt` | 同目录 |
| `data/mapping地球2.xlsx` | `玩家发言整理（供运营侧）/社群数据/mapping地球2.xlsx` |
| `data/话提簇白名单q2.jsonl` | 同目录 |
| `output/02232群.xlsx` | 同目录（历史运行产物） |

## 注意事项

- 完整跑完约 8000+ 条、42 批，耗时长且消耗 API 额度，建议先用小时间窗口试跑。
- 若出现 `白名单更新出错：'话题簇名称'`，多为模型输出字段与预期不一致，分类结果仍会写入 Excel，可检查模型#4 输出格式。
- 本包 **不包含** 其他 Notebook（如 `qun1.ipynb`、`main.ipynb`）及 `peter发言/`、`test/` 等子目录。

## 下游：分析后数据修复（人工处理）

`qun2_main` 输出的 Excel 需经四步后处理，脚本在仓库 `data_repair/` 目录：

| 顺序 | 脚本 | 作用 |
|------|------|------|
| ① | `none_update.py` | 对照群 txt 补齐空单元格 |
| ② | `jiyouhua.py` | 修正「极」字污染字段 |
| ③ | `merge.py` | 单文件话题簇列合并 |
| ④ | `excelmerg.py` | 多批次 Excel 最终汇总 |

详见仓库根目录 [data_repair/README.md](../data_repair/README.md)
