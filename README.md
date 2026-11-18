# LiBenchmark AI-Subset（D1 / D2）v1.0-draft

基于两个公开番茄检测数据集的 **AI 自动生成属性标注 + 可查询目标检测评测用例集**。
全流程人工标注投入为 0：逐框属性（成熟度 / 遮挡 / 尺寸）由脚本从原始检测框确定性导出，
查询用例由大模型（GLM-5.3-flash）按冻结模板生成、gold 答案 100% 由脚本求解并通过
oracle 自洽校验（D1 1967/1967、D2 1024/1024，Stage D 集合精确相等）。

> **English abstract**: This release provides auto-generated per-box attribute annotations
> (maturity / occlusion / size) and a query-based object-detection evaluation suite (Li0–Li3,
> 1,967 Chinese cases on D1, 1,004 migrated cases on D2, with English translations), derived
> from two public tomato datasets (D1: ScienceDB 10.57760/sciencedb.j00001.00946, CC BY 4.0;
> D2: tomatOD, CC BY-NC-SA 4.0). All gold answers are script-solved and oracle-verified.
> Images are NOT redistributed; see "原始数据获取".

## 版本状态（务必阅读）

- 版本：`LiBench-v1.0-draft`（与数据内 `benchmark_ver` 字段一致）
- **AI 子集，未经人工逐条审校**（每条用例的 `provenance.human_verified` 均为 `false`）
- 人工审校完成后将发布 `v1.0` 正式冻结版（含审校标记），届时本包归档不再更新
- 数据按「冻结事实源」管理：发布后不做原地修改，修正以新版本发布

## 目录结构

```
├─ cases/D1/GLMGenerated.jsonl            # D1 查询用例 1,967 条（中文，事实源）
├─ cases/D1/GLMGenerated.bilingual.jsonl  # 同上 + query_en 英文翻译（供开放词表 baseline）
├─ cases/D2/GLMGenerated.jsonl            # D2 查询用例 1,004 条（D1 冻结池逐字迁移 + 重算 gold）
├─ derived/attributes_D1.jsonl            # D1 逐框属性标注（36,426 框）
├─ derived/attributes_D2.jsonl            # D2 逐框属性标注（2,418 框）
├─ derived/attr_thresholds.json           # size 分位数 / occlusion 阈值（冻结值）
├─ ontology/onto-v1.0.yaml                # 属性本体（域、取值、有序性、类名映射）
├─ splits/D1_val_by_lighting.json         # D1 val 跨光照划分
├─ scripts/                               # 生成/求解/校验脚本（provenance，见下）
├─ licenses/                              # 两份许可全文
└─ checksums.sha256                       # 全文件 sha256 清单
```

## 原始数据获取（本包不含图像）

属性与 gold 均以 `image_id` + `box_id` 引用原始数据集，请自行下载对齐：

| 数据集                           | 获取                                                         | 许可            | 引用要求                                                     |
| :------------------------------- | :----------------------------------------------------------- | :-------------- | :----------------------------------------------------------- |
| D1（安农大番茄多模态数据集）     | ScienceDB: [10.57760/sciencedb.j00001.00946](https://doi.org/10.57760/sciencedb.j00001.00946) | CC BY 4.0       | 张雨、饶元、陈文骏等（2023）；数据论文 *China Scientific Data* 10.1:1-16 |
| D2（tomatOD，277 图 / 2,418 框） | GitHub: [up2metric/tomatOD](https://github.com/up2metric/tomatOD) | CC BY-NC-SA 4.0 | Tsironis V., Bourou S., Stentoumis C. (2020). ISPRS Archives XLIII-B3-2020:1077 |

对齐说明：`image_id` 与原始图像文件名（去扩展名）一致；`box_id`（`b01`…）为图像内
检测框的 1-based 序号，顺序与原始标注文件一致。

## 生成方法（全程零人工标注）

**逐框属性**（`attributes_*.jsonl`，脚本确定性导出，`scripts/export_attributes*.py`）：

- `maturity`：原始检测类别查表映射（D1 `immature/half/ripe`、D2 `unripe/semi-ripe/fully-ripe`
  → 本体三档 `unripe/half-ripe/ripe`，映射表见 `ontology/onto-v1.0.yaml`）
- `occlusion`：框最大重叠覆盖率 `r_i = max_{j≠i} |b_i ∩ b_j| / |b_i|`，0 → 无遮挡，
  0 < r ≤ 0.15 → 轻度，r > 0.15 → 中重度（阈值 0.15 为经验值冻结，未做人工校准）
- `size`：框像素面积 1/3、2/3 分位数三档（D1 train 32,829 框标定：≤1927 / ≤3782 px²；
  D2 为数据集级 N/A，阈值跨集未验证，默认不导出——见用例 `na_domains_by_M`）

**查询用例**（`cases/*/GLMGenerated*.jsonl`，`scripts/gen_c_cases.py`）：

- 生成模型 GLM-5.3-flash，prompt 模板 `c-cases-v1`，val 400 图（官方划分，与训练 disjoint）
- 防污染：与后续评测模型换模型族（GLM vs DeepSeek）+ 换 prompt + val 图隔离
- gold（`gold_flat_slots` + `gold_box_ids`）由脚本按本体对属性表确定性求解，非模型产出
- 英文翻译：DeepSeek-V4.1-Flash-1017，temperature 0.1，硬规则 0 违例
- D2 用例：D1 冻结查询池**逐字复用**（零新 query），D2 属性表重算 gold 后过可实现性闸门

**质量校验**（`scripts/run_stage_d.py`）：Stage D oracle 检查——按 gold 槽位对每条用例
重新求值，与标注 `gold_box_ids` 集合精确相等才通过。D1 1,967/1,967、D2 1,024/1,024
（1024 为过闸门数）全过，含 19 档 τ 扫描与文件自洽检查。

## 用例字段说明（`cases/*/GLMGenerated*.jsonl`）

| 字段                                                         | 含义                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `case_id`                                                    | 唯一编号（D1 `LiB-*` / D2 `LiB-D2-*`）                       |
| `level`                                                      | Li0（纯基础）/ Li1（合取）/ Li2（范围 GTE/LTE）/ Li3（否定） |
| `subset`                                                     | Li0-Base / Li1-Conj / Li2-Range / Li3-Bool / Li3-Neg（最小对）/ Empty-Answer / Distract-Neg |
| `image_id` / `lighting`                                      | 原始图像 id 与光照元数据（D2 无光照标注，为 null）           |
| `query_zh` / `query_en`                                      | 中文查询句 / 英文翻译（仅 bilingual 文件有 `query_en`）      |
| `query_style`                                                | 句式（如 distract / plain / minimal-pair A/B）               |
| `gold_flat_slots`                                            | 四域（base/maturity/occlusion/size）的算子+值，gold 槽位     |
| `gold_box_ids` / `all_box_ids`                               | 应检出框集合 / 图内全部框（Empty-Answer 时 gold 为空数组）   |
| `contrast_ratio` / `boundary_flag` / `empty_answer` / `ambiguity_flag` | 造题闸门统计量                                               |
| `min_pair_id`                                                | Li3-Neg 最小对编号（同图同风格，A/B 两句仅否定差）           |
| `provenance`                                                 | 生成模型、prompt 版本、gold 求解来源、本体版本、人工审校状态 |
| `d2_split` / `provenance.migrated_from`                      | 仅 D2：原始 split 与迁移来源                                 |

属性文件字段（`attributes_*.jsonl`）：`image_id / split / lighting / box_id /
class_id / class_name / cx / cy / w / h`（YOLO 归一化框，D1 为 1280×720 基准）/
`area_px / r_occl / occlusion(+label) / size(+label)`。

## 规模统计

D1 用例（1,967 条，val 400 图）：

| 子集                       | 条数 |
| :------------------------- | ---: |
| Li0-Base                   |  300 |
| Li1-Conj                   |  423 |
| Li2-Range                  |  306 |
| Li3-Bool                   |  423 |
| Li3-Neg（60 对 ×2 风格行） |  120 |
| Empty-Answer               |  105 |
| Distract-Neg               |  150 |

D2 用例 1,004 条（分布与 D1 冻结池对应，过可实现性闸门后的子集）；
属性标注 D1 36,426 框 / D2 2,418 框。

已知局限：Li3-Neg 最小对 60 对 < 协议目标 150 对（val 规模所限）；
occlusion 阈值为未经人工验证的经验值（操作化定义）。

## 许可（重要：分目录双许可）

本包为两个原始数据集的衍生数据库，按来源分别授权，**不设统一许可**：

- `cases/D1/`、`derived/attributes_D1.jsonl` 等基于 D1 的内容 → **CC BY 4.0**
  （[全文](licenses/CC-BY-4.0.txt)｜[deed](https://creativecommons.org/licenses/by/4.0/)）
- `cases/D2/`、`derived/attributes_D2.jsonl` 等基于 D2 的内容 → **CC BY-NC-SA 4.0**
  （[全文](licenses/CC-BY-NC-SA-4.0.txt)｜[deed](https://creativecommons.org/licenses/by-nc-sa/4.0/)；
  SA 条款要求衍生作品以相同许可发布，NC 限非商业）
- 公共部分（`README.md`、`scripts/`、`checksums.sha256`）随所在目录许可一起使用；
  如需整体再分发，请按两部分分别注明许可
- 使用时须同时引用本数据集与对应原始数据集（见下）

## 引用

审校冻结后正式 DOI 将补充在此。当前请引用本仓库：

```bibtex
@dataset{libenchmark_ai_v1draft,
  title  = {LiBenchmark AI-Subset (D1/D2), v1.0-draft},
  year   = {2026},
  note   = {Auto-generated attribute annotations and query-based detection
            evaluation cases; oracle-verified; not yet human-reviewed},
  howpublished = {Gitee repository}
}
```

原始数据集引用：张雨、饶元、陈文骏等（2023），ScienceDB [10.57760/sciencedb.j00001.00946](https://doi.org/10.57760/sciencedb.j00001.00946)；
Tsironis V., Bourou S., Stentoumis C. (2020), ISPRS Archives XLIII-B3-2020:1077。

## 完整性校验

```powershell
# Windows（PowerShell）
Get-Content checksums.sha256 | ForEach-Object { $h,$p = $_ -split '  '; if ((Get-FileHash $p -Algorithm SHA256).Hash.ToLower() -eq $h) { "OK  $p" } else { "FAIL $p" } }
# Linux
sha256sum -c checksums.sha256
```

## scripts/ 说明（provenance 脚本，按原样提供）

`export_attributes.py` / `export_attributes_D2.py`（属性导出）、`gen_c_cases.py`
（D1 用例生成 + gold 求解）、`transfer_to_D2.py`（D2 迁移 + 重算 gold）、
`translate_cases.py`（英文化）、`run_stage_d.py`（oracle 校验）。
脚本内路径为项目内相对引用，复现需自行调整数据根目录；仅作为生成过程溯源与可复现凭据提供。

## 联系

问题反馈请提 issue；使用本数据集的论文请引用（引用条目见上，正式版发布后替换为 DOI）。
