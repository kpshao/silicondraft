# SiliconDraft 对外契约 · AD5 输出规格（WAT）

> **契约性质**：AD5 是交付给 YMS 的**输出契约**。
> **⚠️ 规格未冻结**：AD5 属代工厂 / 客户侧的私有或行业约定 WAT 数据交换格式，公开渠道**未能检索到权威规范文档**。本文件定义**接口契约与占位规格**，具体字段规格冻结依赖用户提供样例文件（见 **`SD-Q-003`，唯一开工阻塞项**）。
> **强制要求**：**规格冻结前，AD5 插件在导出前置校验中必须返回错误阻止导出**，避免产出错误格式文件。
> **本文档不是需求**：需求见 `doc/requirements-spec.MD` §6.5。


> **⚠️ 重要前置**：AD5 属于**代工厂 / 客户侧的私有或行业约定 WAT 数据交换格式**，公开渠道**未能检索到权威的格式规范文档**。因此本节定义**接口契约与占位规格**，**具体字段规格冻结依赖用户提供样例文件**（见 **SD-Q-003，唯一开工阻塞项**）。
> **规格冻结前，AD5 插件的 `ValidateContext` 必须返回错误阻止导出**，避免产出错误格式。

**SD-FMT-030**（MUST）：AD5 是 WAT（PCM/ET）数据的**表格型交换格式**，典型组织维度为：

```
Lot → Wafer → Site（测试点） → Test Key（测试结构） → Parameter（参数）
```

**SD-FMT-031**（MUST）：必须支持的字段集（依据 WAT 数据通用字段约定，最终以样例为准）：

| 分组 | 字段 | 来源 |
| --- | --- | --- |
| 归属 | `LOT_ID`, `WAFER_ID`, `WAFER_SLOT` | `sd_lot` / `sd_wafer` |
| 位置 | `SITE_ID`, `X_COORD`, `Y_COORD`, `RADIUS` | `sd_wat_structure` |
| 结构 | `TEST_KEY_ID`, `STRUCTURE_TYPE`, `DUT_TYPE` | `sd_wat_structure` |
| 程序 | `TEST_PROGRAM_REV`, `TESTER_ID`, `PROBE_CARD_ID` | 元数据 |
| 条件 | `TEMPERATURE`, `BIAS_CONDITION` | 测试项参数（§7.2） |
| 参数 | `PARAMETER_NAME`, `MEASURED_VALUE`, `UNIT` | `sd_test_item` / 结果 |
| 判定 | `LOWER_SPEC_LIMIT`, `UPPER_SPEC_LIMIT`, `PASS_FAIL` | `sd_test_item_limit` |
| 其他 | `RETEST_FLAG`, `TIMESTAMP` | 派生 |

**SD-FMT-032**（MUST）：**排布形态** —— AD5 通常有两种排布，插件必须两种都支持（`options.layout`）：

| layout | 说明 | 典型列 |
| --- | --- | --- |
| `long`（默认） | 每行一个（Site, Parameter）测量值 | `LOT,WAFER,SITE,KEY,PARAM,VALUE,UNIT,LSL,USL,PF` |
| `wide` | 每行一个 Site，列为各 Params | `LOT,WAFER,SITE,X,Y,<Param1>,<Param2>,...` |

**SD-FMT-033**（MUST）：**文件组织**：

| 模式 | 说明 |
| --- | --- |
| `per_lot`（默认） | 一个 Lot 一个文件（WAT 数据惯例：一个文件含 25 片 Wafer） |
| `per_wafer` | 一片一个文件 |
| `per_lot_per_test` | 按参数拆文件（部分老工具要求） |

**SD-FMT-034**（MUST）：**编码与格式细节**（默认值，以样例为准）：

| 项 | 默认 |
| --- | --- |
| 编码 | `gbk`（国内代工厂常见）/ 可切 `utf-8` |
| 分隔符 | `\t`（可切 `,` / 空格） |
| 换行 | `\r\n`（Windows 兼容）/ 可切 `\n` |
| 小数位 | 4（可配） |
| 表头块 | 可选的文件级 Key-Value 头（`options.include_header_block`），含产品、节点、日期、程序版本 |
| 缺失值 | 空串或 `NA`（可配） |
| 科学计数 | 小值必须用科学计数（阈值可配，默认 `<1e-4`） |

**SD-FMT-035**（MUST）：**必须提供模板机制** —— `options.template_file` 指向外部模板文件（表头文本 + 列顺序声明），使同一插件可适配不同代工厂的 AD5 变体，无需改代码。

**SD-FMT-036**（MUST）：**AD5 自检**必须校验：

- 行数 = 结构数 × 启用参数数（long 布局）
- 每个 `(LOT, WAFER, SITE, KEY, PARAM)` 组合唯一
- 数值可被解析为浮点（或为约定的缺失标记）
- 编码与分隔符与声明一致（**重新读回验证**）
- `PASS_FAIL` 与 `LSL/USL` 判定一致

**SD-FMT-037**（MUST）：**与 STDF 的坐标一致性** —— AD5 的 Site 坐标必须与 `sd_wat_structure` 一致，且同 Wafer 的 CP STDF 与 WAT AD5 必须共享同一 Wafer 坐标系（保证 YMS 侧空间叠加正确）。

