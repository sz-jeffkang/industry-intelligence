# 产业集群分析五步法

> 从零散企业数据到产业集群全景画像的系统化方法论。覆盖梯队分层、区位熵计算、集群健康度诊断、交叉融合识别、候选集群模板——5步完成从"有什么企业"到"产业怎么发展"的跃迁。

---

## 目录

1. [Step 1: 企业梯队分层](#step-1-企业梯队分层)
2. [Step 2: 产业集群识别](#step-2-产业集群识别)
3. [Step 3: 区位熵 (LQ) 空间分析](#step-3-区位熵-lq-空间分析)
4. [Step 4: 集群健康度诊断](#step-4-集群健康度诊断)
5. [Step 5: 集群交叉融合与选址匹配](#step-5-集群交叉融合与选址匹配)
6. [附录: 六大候选集群模板](#附录-六大候选集群模板)

---

## Step 1: 企业梯队分层

### 1.1 目的

将区域内的企业按产值规模划分为清晰梯队，识别"谁在支撑经济总量""谁在拉动增长""谁在填补生态"，为后续的集群分析和政策靶向提供基础分层。

### 1.2 分层标准

| 层级 | 阈值 | 标记 | 特征描述 |
|------|:----:|:----:|----------|
| 巨头 | ≥100亿 | 🔴 | 独立园区、产业链链主、区域经济压舱石 |
| 龙头 | 10-100亿 | 🟠 | 细分领域冠军、带动配套企业集聚 |
| 骨干 | 1-10亿 | 🟡 | 产业中坚、高成长潜力 |
| 中小 | <1亿(有产值) | 🟢 | 配套生态、创新种子 |
| 无数据 | 产值缺失 | ⬜ | 需核实统计口径 |

### 1.3 梯队分层 SQL

```sql
-- 产值梯队分层（含脏数据清洗）
WITH cleaned AS (
    SELECT 
        enterprise_name,
        street,
        industry_code,
        -- 产值字段清洗：去除非数字字符后转数值
        NULLIF(
            REGEXP_REPLACE(COALESCE(output_raw, '0'), '[^0-9.]', '', 'g'),
            ''
        )::numeric AS output_clean
    FROM enterprises
    WHERE enterprise_name IS NOT NULL
)
SELECT 
    CASE 
        WHEN output_clean >= 100 THEN '巨头≥100亿'
        WHEN output_clean >= 10  THEN '龙头10-100亿'
        WHEN output_clean >= 1   THEN '骨干1-10亿'
        WHEN output_clean > 0    THEN '中小<1亿'
        ELSE '无数据'
    END AS tier,
    COUNT(*) AS enterprise_count,
    ROUND(COALESCE(SUM(output_clean), 0), 1) AS total_output_yi,
    ROUND(COALESCE(AVG(output_clean), 0), 2) AS avg_output_yi
FROM cleaned
GROUP BY tier
ORDER BY MIN(output_clean) DESC NULLS LAST;
```

### 1.4 产值字段清洗陷阱

⚠️ 原始产值字段常见问题：
- 含中文字符（如 "约1.2亿"、"暂缺"）
- 含 `-`、空格等非数字字符
- 含区间格式（如 "0.03-0.05"）

**清洗公式**：
```sql
-- 标准清洗
REGEXP_REPLACE(COALESCE(output_raw, '0'), '[^0-9.]', '', 'g')

-- 区间格式→中值
CASE WHEN output_raw ~ '^[0-9.]+-[0-9.]+$' THEN
    (SPLIT_PART(output_raw, '-', 1)::numeric + 
     SPLIT_PART(output_raw, '-', 2)::numeric) / 2.0
WHEN output_raw ~ '^[0-9.]+$' THEN output_raw::numeric
ELSE NULL END
```

### 1.5 统计口径注意事项

- **四上企业口径**：企业底表通常为四上企业（规上工业 + 资质建筑业 + 限额以上批零住餐 + 规模以上服务业），不是全量企业
- **"无数据"≠"零产值"**：报告中使用"无四上企业数据"而非"零产值"
- **园区无四上企业**：可能以小微企业/配套服务/孵化器为主，不能直接判定为低效

---

## Step 2: 产业集群识别

### 2.1 集群定义方法

产业集群不是简单的"同一行业代码"，而是基于以下多维规则的综合判定：

| 识别维度 | 方法 | 示例 |
|----------|------|------|
| 行业代码前缀 | `LEFT(industry_code, 2) = 'XX'` | 电子元器件 = 行业代码以39开头 |
| 关键词匹配 | 企业名称/主营产品模糊匹配 | PCB = 名称含'电路'/'线路板' |
| 行业分类体系 | wind四级行业 / 国民经济行业大类 | 消费电子 = wind分类 + 关键词 |
| 战新产业标签 | 战略性新兴产业标签 | 低空经济 = 名称含'无人机'/'航空' |

### 2.2 集群企业提取 SQL

```sql
-- 通用集群企业提取模板
CREATE TEMP VIEW cluster_<集群名> AS
SELECT enterprise_name, street, industry_code, output_clean,
       xiaojuren, gaoxin, listed
FROM enterprises
WHERE 
    -- 规则1: 行业代码（多条OR）
    LEFT(industry_code, 2) IN ('<行业代码1>', '<行业代码2>')
    -- 规则2: 名称关键词（可选叠加）
    OR enterprise_name ~ '<关键词正则>'
    -- 规则3: 战新标签（如有）
    OR <战新标签列> >= 0.5;

-- 集群汇总
SELECT '<集群名>' AS cluster_name,
       COUNT(*) AS enterprise_count,
       ROUND(SUM(output_clean), 1) AS total_output_yi,
       COUNT(*) FILTER (WHERE xiaojuren IS NOT NULL) AS xiaojuren_count,
       COUNT(*) FILTER (WHERE gaoxin = '1') AS gaoxin_count,
       COUNT(*) FILTER (WHERE listed = '1') AS listed_count
FROM cluster_<集群名>;
```

### 2.3 集群交叉融合识别

识别集群间的交叉融合（同一企业属于多个集群），可发现产业链耦合机会：

```sql
-- 集群交叉矩阵
WITH cluster_tags AS (
    SELECT enterprise_name,
           CASE WHEN <集群A条件> THEN 1 ELSE 0 END AS is_cluster_a,
           CASE WHEN <集群B条件> THEN 1 ELSE 0 END AS is_cluster_b
    FROM enterprises
)
SELECT 
    SUM(CASE WHEN is_cluster_a=1 AND is_cluster_b=1 THEN 1 ELSE 0 END) AS cross_count,
    ROUND(SUM(CASE WHEN is_cluster_a=1 AND is_cluster_b=1 THEN output_clean ELSE 0 END), 1) AS cross_output
FROM cluster_tags;
```

---

## Step 3: 区位熵 (LQ) 空间分析

### 3.1 LQ 公式与含义

区位熵（Location Quotient）衡量某一产业在特定空间单元的专业化程度：

```
LQ_ij = (e_ij / E_i) / (E_j / E)

其中：
  e_ij = 空间单元i中产业j的企业数
  E_i  = 空间单元i中所有产业的企业总数
  E_j  = 全区域内产业j的企业总数
  E    = 全区域所有产业的企业总数
```

### 3.2 判定标准

| LQ 值 | 判定 | 标记 | 含义 |
|:-----:|------|:----:|------|
| > 2.0 | 高度专业化 | ★★★ | 显著集聚，产业地标 |
| 1.5 ~ 2.0 | 专业化 | ★★ | 明显集聚，竞争优势 |
| 1.0 ~ 1.5 | 初步专业化 | ★ | 有一定集聚趋势 |
| 0.5 ~ 1.0 | 非专业化 | — | 分布与全区平均水平一致 |
| < 0.5 | 显著不足 | ▼ | 该产业在本单元缺乏基础 |

### 3.3 LQ 计算 SQL

```sql
-- 街道×产业的区位熵全量计算
WITH street_industry_count AS (
    -- 每个街道每个产业的企业数
    SELECT street, industry_big, COUNT(*) AS ent_cnt
    FROM enterprises
    WHERE street IS NOT NULL AND industry_big IS NOT NULL
    GROUP BY street, industry_big
),
street_total AS (
    -- 每个街道总企业数
    SELECT street, SUM(ent_cnt) AS street_total
    FROM street_industry_count
    GROUP BY street
),
industry_total AS (
    -- 每个产业全区总企业数
    SELECT industry_big, SUM(ent_cnt) AS industry_total
    FROM street_industry_count
    GROUP BY industry_big
),
region_total AS (
    -- 全区总企业数
    SELECT SUM(ent_cnt) AS region_total
    FROM street_industry_count
)
SELECT 
    sic.street,
    sic.industry_big,
    sic.ent_cnt,
    ROUND(
        (sic.ent_cnt::numeric / st.street_total) / 
        (it.industry_total::numeric / rt.region_total),
        3
    ) AS lq_score,
    CASE 
        WHEN (sic.ent_cnt::numeric / st.street_total) / 
             (it.industry_total::numeric / rt.region_total) > 2.0 THEN '★★★'
        WHEN (sic.ent_cnt::numeric / st.street_total) / 
             (it.industry_total::numeric / rt.region_total) > 1.5 THEN '★★'
        WHEN (sic.ent_cnt::numeric / st.street_total) / 
             (it.industry_total::numeric / rt.region_total) > 1.0 THEN '★'
        ELSE ''
    END AS specialization_level
FROM street_industry_count sic
JOIN street_total st ON sic.street = st.street
JOIN industry_total it ON sic.industry_big = it.industry_big
CROSS JOIN region_total rt
WHERE it.industry_total >= 5  -- 过滤小样本
ORDER BY lq_score DESC;
```

### 3.4 LQ 的局限性

- LQ 基于企业数量，未加权产值——高产值集中但企业数少的产业可能被低估
- 小样本（企业总数 < 20）时 LQ 不稳定，需设最小阈值
- 建议配合**产值加权 LQ** 做交叉验证：用产值替代企业数重算

---

## Step 4: 集群健康度诊断

### 4.1 健康度三维公式

```
集群健康度 = 龙头力 × 配套力 × 创新力
```

| 维度 | 定义 | 计算方法 | 权重 |
|------|------|----------|:----:|
| **龙头力** | 头部企业引领能力 | (TOP3 企业产值 / 集群总产值) × CR5 集中度 | 0.4 |
| **配套力** | 企业密度与生态完整性 | (企业数 / 街道覆盖数) × 中小企占比 | 0.3 |
| **创新力** | 创新资源密度 | (小巨人数 + 国高数 + 上市数) / 企业总数 | 0.3 |

### 4.2 健康度计算 SQL

```sql
-- 集群健康度综合评分
WITH cluster_base AS (
    SELECT cluster_name,
           COUNT(*) AS total_ent,
           SUM(output_clean) AS total_output,
           COUNT(*) FILTER (WHERE xiaojuren IS NOT NULL) AS xiaojuren_cnt,
           COUNT(*) FILTER (WHERE gaoxin = '1') AS gaoxin_cnt,
           COUNT(*) FILTER (WHERE listed = '1') AS listed_cnt,
           COUNT(DISTINCT street) AS street_cnt,
           -- TOP3 产值占比
           SUM(CASE WHEN output_rank <= 3 THEN output_clean ELSE 0 END) / 
               NULLIF(SUM(output_clean), 0) AS top3_share,
           -- 中小企占比
           COUNT(*) FILTER (WHERE output_clean < 1 AND output_clean > 0)::numeric / 
               NULLIF(COUNT(*), 0) AS sme_ratio
    FROM (
        SELECT *, ROW_NUMBER() OVER (PARTITION BY cluster_name ORDER BY output_clean DESC) AS output_rank
        FROM cluster_enterprises
    ) sub
    GROUP BY cluster_name
)
SELECT cluster_name,
       total_ent,
       ROUND(total_output, 1) AS output_yi,
       -- 龙头力 (0-100)
       ROUND((top3_share * 100)::numeric, 1) AS leadership_score,
       -- 配套力 (0-100)
       ROUND(((total_ent::numeric / NULLIF(street_cnt, 0)) / 100 * 50 + 
              sme_ratio * 50)::numeric, 1) AS density_score,
       -- 创新力 (0-100)
       ROUND(((xiaojuren_cnt * 5 + gaoxin_cnt * 1 + listed_cnt * 3)::numeric / 
              NULLIF(total_ent, 0) * 100)::numeric, 1) AS innovation_score,
       -- 综合健康度
       ROUND((top3_share * 0.4 + 
              ((total_ent::numeric / NULLIF(street_cnt, 0)) / 100 * 0.15 + sme_ratio * 0.15) +
              ((xiaojuren_cnt * 5 + gaoxin_cnt * 1 + listed_cnt * 3)::numeric / NULLIF(total_ent, 0) * 0.3)
             ) * 100, 1) AS health_score
FROM cluster_base
ORDER BY health_score DESC;
```

### 4.3 健康度评级

| 评分 | 等级 | 含义 | 政策建议 |
|:----:|:----:|------|----------|
| ≥80 | S | 世界级集群 | 全球对标、品牌输出 |
| 60-80 | A | 成熟集群 | 巩固优势、补链强链 |
| 40-60 | B | 成长集群 | 引入龙头、培育配套 |
| 20-40 | C | 萌芽集群 | 重点扶持、孵化加速 |
| <20 | D | 散点分布 | 需重新评估集群定义 |

---

## Step 5: 集群交叉融合与选址匹配

### 5.1 集群交叉分析

识别两个集群间的企业重叠，发现产业链耦合机会：

```sql
-- 集群交叉融合矩阵
SELECT 
    c1.cluster_name AS cluster_a,
    c2.cluster_name AS cluster_b,
    COUNT(*) AS cross_enterprises,
    ROUND(SUM(e.output_clean), 1) AS cross_output
FROM cluster_tag_table c1
JOIN cluster_tag_table c2 ON c1.enterprise_name = c2.enterprise_name
JOIN enterprises e ON c1.enterprise_name = e.enterprise_name
WHERE c1.cluster_name < c2.cluster_name
  AND c1.is_member = 1 AND c2.is_member = 1
GROUP BY c1.cluster_name, c2.cluster_name
HAVING COUNT(*) >= 5
ORDER BY cross_enterprises DESC;
```

### 5.2 产业圈层选址速查

基于集群空间分布，为企业引进提供快速园区匹配：

```sql
-- 产业×街道集聚度（用于选址推荐）
SELECT 
    industry_big,
    street,
    COUNT(*) AS enterprise_count,
    ROUND(COUNT(*)::numeric / SUM(COUNT(*)) OVER (PARTITION BY industry_big) * 100, 1) AS concentration_pct
FROM enterprises
WHERE industry_big IS NOT NULL AND street IS NOT NULL
GROUP BY industry_big, street
ORDER BY industry_big, enterprise_count DESC;
```

**选址速查表模板**：

| 拟引进企业类型 | 首选街道 | 次选街道 | 推荐园区 | 选址理由 |
|---------------|:--------:|:--------:|---------|----------|
| 电子元器件制造 | `<街道A>` | `<街道B>` | `<园区名>` | 产业生态成熟、配套企业密集 |
| 智能装备 | `<街道C>` | `<街道D>` | `<园区名>` | 链主企业带动、供应链完备 |
| 新能源零部件 | `<街道E>` | `<街道F>` | `<园区名>` | 新兴产业集聚区、政策倾斜 |

---

## 附录: 六大候选集群模板

以下为常用的六类产业集群识别规则模板，可直接套用：

### 集群1: C39 电子元器件

```sql
-- 识别规则
LEFT(industry_code, 2) = '39'
-- 或企业名称含：电子/元件/半导体/集成电路
enterprise_name ~ '电子|元件|半导体|集成电路'
```

| 维度 | 典型值 |
|------|--------|
| 行业代码 | 39xx（计算机、通信和其他电子设备制造业） |
| 典型子类 | 3982 电子元件、3911 计算机整机、3921 通信设备 |
| 预期规模 | 企业数最多、产值占比最高 |

### 集群2: 智能装备

```sql
-- 识别规则
LEFT(industry_code, 2) IN ('34', '35')
-- 或名称含：装备/智能/自动化/机器人
enterprise_name ~ '装备|智能|自动化|机器人|CNC|数控'
```

| 维度 | 典型值 |
|------|--------|
| 行业代码 | 34xx 通用设备、35xx 专用设备 |
| 典型子类 | 3425 模具制造、3491 工业机器人、3562 半导体器件专用设备 |

### 集群3: 新能源零部件

```sql
-- 识别规则
LEFT(industry_code, 2) = '38'
AND enterprise_name ~ '新能源|储能|电池|光伏|逆变器|充电|锂电'
```

| 维度 | 典型值 |
|------|--------|
| 行业代码 | 38xx 电气机械和器材制造业 |
| 典型子类 | 3841 锂离子电池、3842 镍氢电池、3821 变压器 |

### 集群4: 消费电子

```sql
-- 识别规则（组合判断）
(
    LEFT(industry_code, 2) = '39'
    AND enterprise_name ~ '手机|平板|笔记本|可穿戴|智能终端|VR|AR|耳机|音箱'
)
OR main_products ~ '手机|平板|笔记本|耳机|音箱|可穿戴'
```

### 集群5: PCB/电路板

```sql
-- 识别规则
enterprise_name ~ '电路|线路板|PCB|FPC|柔性电路|IC载板|HDI'
-- 或 main_products 含
main_products ~ '印制电路板|电路板|PCB|FPC|HDI|IC载板'
```

| 维度 | 典型值 |
|------|--------|
| 天然特征 | 高耗能行业（不可替代，推绿电方案） |
| 产业地位 | 电子信息基础支撑 |

### 集群6: LED/光电

```sql
-- 识别规则
enterprise_name ~ '光电|光电子|LED|显示|背光|照明'
-- 或行业代码
LEFT(industry_code, 2) = '39'
AND main_products ~ 'LED|显示|背光|光源|Mini LED|Micro LED'
```

### 集群模板使用说明

1. **按需激活**：根据区域产业基础选择2-6个候选集群
2. **规则调优**：先用 `GROUP BY industry_code` 看实际分布，再微调识别规则
3. **交叉验证**：用 LQ > 1.0 验证集群是否为"真集聚"（非随机分布）
4. **阈值调整**：战新标签列通常为 0-1 连续得分（非布尔值），需先 `SELECT DISTINCT` 看分布后设阈值（如 ≥0.5）

---

## 常见陷阱与最佳实践

### PG SQL 类型转换

```sql
-- ❌ ROUND(double precision, int) 不存在
ROUND(park_area / 10000.0, 3)  -- ERROR

-- ✅ 显式 ::numeric 转换
ROUND((park_area / 10000.0)::numeric, 3)  -- OK
```

### 空间查询单位

```sql
-- ❌ 4326坐标系下 ST_DWithin 单位是度数，500度≈半个地球
ST_DWithin(geom, road_geom, 500)  -- 错误

-- ✅ 用 ::geography 做米级距离
ST_DWithin(geom::geography, road_geom::geography, 500)  -- 正确：500米
```

### 分类标签陷阱

- 部分数据源的战新标签列是 **0-1 连续匹配度得分**，不是 0/1 布尔值
- 必须先用 `SELECT DISTINCT <标签列> FROM table` 看分布，再设阈值
- 不要假设 `WHERE <标签列> = 1` 能正确筛选

### 零值结论验证

- "零产值园区" 可能因统计口径（四上企业）导致，非真正零值
- 报告用语：`无四上企业入园` 而非 `零产值`
- 低效判定需多维度交叉：产出率 + 容积率 + 建筑密度
