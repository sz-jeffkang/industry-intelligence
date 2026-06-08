# 空间经济分析方法

> 从企业点位数据到空间经济规律的系统化分析方法。覆盖沿路产业带 buffer 分析、创新载体耦合度诊断、企业生命周期空间行为三大模块。核心发现：**"近路孵化，远路建园"**是头部企业的普遍空间规律。

---

## 目录

1. [沿路产业带 Buffer 分析](#沿路产业带-buffer-分析)
2. [创新载体耦合度分析](#创新载体耦合度分析)
3. [企业生命周期空间行为](#企业生命周期空间行为)
4. [空间带假说验证方法](#空间带假说验证方法)
5. [已知反直觉发现](#已知反直觉发现)
6. [常见陷阱与避坑清单](#常见陷阱与避坑清单)

---

## 沿路产业带 Buffer 分析

### 1.1 分析目的

回答以下问题：
- 企业是否沿着交通要道形成了带状集聚？
- 距道路不同距离的企业在规模、产业类型、创新能力上有何差异？
- 哪些产业有"沿路偏好"？

### 1.2 三级空间分区

| 分区 | 距离范围 | 经济含义 | 典型特征 |
|------|:--------:|----------|----------|
| **带内 (Core Band)** | ≤500m | 中小企业孵化器 | 企业密度最高、平均规模偏小、战新渗透率中等 |
| **过渡带 (Transition Band)** | 500m-2km | 最优区位 | 平均产值最高、"近而不贴"的黄金距离 |
| **远离带 (Outer Band)** | >2km | 头部企业独立园区 | 企业数少但单企规模大、小巨人/上市企业集中 |

> **核心发现**：过渡带（500m-2km）平均产值通常高于带内，说明"近而不贴"是最优产业区位。

### 1.3 数据准备

```sql
-- 路网分级统计（只取高速/主干/一级）
SELECT road_class, COUNT(*) AS segment_count,
       ROUND(SUM(road_length_m)::numeric / 1000, 1) AS total_km
FROM road_network r
JOIN region_boundary b ON ST_Intersects(r.geom, b.geometry)
WHERE road_class IN ('motorway', 'trunk', 'primary')
GROUP BY road_class
ORDER BY total_km DESC;
```

⚠️ 注意事项：
- `road_length_m` 字段必须带表前缀（如 `r."Shape_Leng"`），大写的 `L` 是 PostgreSQL 区分大小写的陷阱
- 路网表 `geom` 列类型为 `MULTILINESTRING`

### 1.4 多级 Buffer 企业统计

```sql
-- 关键道路500m/1km/2km/5km缓冲区内企业统计
WITH key_roads AS (
    SELECT ST_Union(r.geom) AS geom
    FROM road_network r
    JOIN region_boundary b ON ST_Intersects(r.geom, b.geometry)
    WHERE road_class IN ('motorway', 'trunk', 'primary')
)
SELECT 
    CASE 
        WHEN ST_DWithin(e.geom::geography, kr.geom::geography, 500)  THEN '带内(≤500m)'
        WHEN ST_DWithin(e.geom::geography, kr.geom::geography, 2000) THEN '过渡带(500m-2km)'
        ELSE '远离(>2km)'
    END AS zone,
    COUNT(*) AS enterprise_count,
    COUNT(*) FILTER (WHERE output_clean > 0) AS with_output,
    ROUND(AVG(output_clean) FILTER (WHERE output_clean > 0), 2) AS avg_output_yi,
    COUNT(*) FILTER (WHERE xiaojuren IS NOT NULL) AS xiaojuren_count,
    COUNT(*) FILTER (WHERE gaoxin = '1') AS gaoxin_count
FROM enterprises e, key_roads kr
WHERE e.geom IS NOT NULL
GROUP BY zone
ORDER BY zone;
```

⚠️ 关键陷阱：**必须用 `::geography` 转换**。4326 坐标系下 `ST_DWithin` 单位为度数，不加 `::geography` 时 500 度 ≈ 半个地球，结果完全错误。

### 1.5 逐路产业集聚分析

```sql
-- 每条要道500m内的TOP4产业
SELECT 
    rd.road_name,
    rd.road_class,
    COALESCE(e.industry_big, '未分类') AS industry,
    COUNT(*) AS enterprise_count,
    ROUND(SUM(e.output_clean), 1) AS total_output_yi
FROM road_network rd
JOIN enterprises e ON e.geom IS NOT NULL 
    AND ST_DWithin(e.geom::geography, rd.geom::geography, 500)
JOIN region_boundary b ON ST_Intersects(rd.geom, b.geometry)
WHERE e.output_clean > 0
  AND rd.road_class IN ('motorway', 'trunk', 'primary')
GROUP BY rd.road_name, rd.road_class, industry
ORDER BY rd.road_name, total_output_yi DESC;
```

### 1.6 战新产业沿路偏好度

```python
# 战新产业沿路偏好度 = 带内占比 / 全区占比
zhanxin_industries = [
    '网络与通信', '半导体与集成电路', '超高清视频显示',
    '智能终端', '智能传感器', '高端装备与仪器', '机器人',
    '新能源', '智能网联汽车', '高性能材料', '低空经济与空天'
]

for industry in zhanxin_industries:
    near_road_pct = count_near_road / total_near_road  # 带内该产业占比
    region_pct = count_region / total_region            # 全区该产业占比
    preference = near_road_pct / region_pct
    
    # preference > 1.2 → 该产业有明显沿路偏好
    # preference ≈ 1.0 → 无明显偏好
    # preference < 0.8 → 该产业倾向"离路"
```

### 1.7 街道沿路集聚度

```sql
-- 各街道企业在要道500m内的比例
WITH key_roads AS (
    SELECT ST_Union(geom) AS geom
    FROM road_network r
    JOIN region_boundary b ON ST_Intersects(r.geom, b.geometry)
    WHERE road_class IN ('motorway', 'trunk', 'primary')
)
SELECT 
    e.street,
    COUNT(*) AS total_enterprises,
    COUNT(*) FILTER (
        WHERE ST_DWithin(e.geom::geography, kr.geom::geography, 500)
    ) AS near_road_count,
    ROUND(
        COUNT(*) FILTER (WHERE ST_DWithin(e.geom::geography, kr.geom::geography, 500))::numeric 
        / COUNT(*) * 100, 1
    ) AS near_road_pct
FROM enterprises e, key_roads kr
WHERE e.geom IS NOT NULL AND e.street IS NOT NULL
GROUP BY street
ORDER BY near_road_pct DESC;
```

---

## 创新载体耦合度分析

### 2.1 分析目的

衡量"创新空间"与"产业空间"的空间耦合程度：
- 产业园区离最近的创新载体有多远？
- 不同类型创新载体对周边产业产出率的带动效应差异？
- 是否存在"500m 圈层效应"？

### 2.2 数据源

| 表 | 关键字段 | 说明 |
|----|----------|------|
| `innovation_carriers` | `carrier_name`, `carrier_type`, `level`, `geom` | 创新载体点位 |
| `parks` | `park_name`, `park_geom`, `output_per_ha` | 产业园区面状数据 |

**载体类型分类**：
- 重点实验室 — 学术导向，产业溢出弱
- 工程研究中心 — 产学研结合，产业溢出最强
- 工程技术研究中心 — 企业主导，应用导向
- 工程实验室 — 中试验证
- 企业技术中心 — 企业内部研发
- 公共技术服务平台 — 中小企业服务

### 2.3 载体类型 × 产出率交叉分析

```sql
-- 各类型创新载体500m内园区的平均产出率
SELECT 
    ic.carrier_type,
    COUNT(DISTINCT p.park_id) AS covered_parks,
    ROUND(AVG(p.output_per_ha)::numeric, 1) AS avg_output_per_ha
FROM parks p
JOIN innovation_carriers ic ON ic.geom IS NOT NULL 
    AND ST_DWithin(p.park_geom::geography, ic.geom::geography, 500)
WHERE p.total_output_yi > 0 
  AND ic.carrier_type IS NOT NULL
GROUP BY ic.carrier_type
ORDER BY avg_output_per_ha DESC;
```

### 2.4 500m 圈层效应分析

```sql
-- 园区到最近创新载体的距离 → 产出率圈层效应
WITH park_nearest_carrier AS (
    SELECT 
        p.park_id,
        p.output_per_ha,
        (
            SELECT MIN(ST_Distance(p.park_geom::geography, ic.geom::geography))
            FROM innovation_carriers ic
            WHERE ic.geom IS NOT NULL
        ) AS min_distance_m
    FROM parks p
    WHERE p.total_output_yi > 0
)
SELECT 
    CASE 
        WHEN min_distance_m <= 500  THEN '≤500m'
        WHEN min_distance_m <= 1000 THEN '500m-1km'
        WHEN min_distance_m <= 2000 THEN '1-2km'
        ELSE '>2km'
    END AS distance_band,
    COUNT(*) AS park_count,
    ROUND(AVG(output_per_ha)::numeric, 1) AS avg_output_per_ha
FROM park_nearest_carrier
GROUP BY distance_band
ORDER BY MIN(min_distance_m);
```

**预期发现**：
- **500m 圈层效应**：500m 内园区产出率显著高于 500m-1km 带（通常高 30%-50%）
- **1km 外衰减**：超过 1km 后，创新溢出效应快速衰减
- **"质 > 量"规律**：创新载体覆盖的园区数量多不等于产出高——载体质量（类型/级别）比数量更重要

### 2.5 创新覆盖 vs 无覆盖对比

```sql
-- 有创新载体覆盖 (≤1km) vs 无覆盖的园区产出率
SELECT 
    p.street,
    COUNT(*) FILTER (WHERE min_dist <= 1000) AS with_innovation,
    ROUND(AVG(p.output_per_ha) FILTER (WHERE min_dist <= 1000)::numeric, 1) AS avg_with,
    COUNT(*) FILTER (WHERE min_dist > 1000) AS without_innovation,
    ROUND(AVG(p.output_per_ha) FILTER (WHERE min_dist > 1000)::numeric, 1) AS avg_without
FROM (
    SELECT p.*, (
        SELECT MIN(ST_Distance(p.park_geom::geography, ic.geom::geography))
        FROM innovation_carriers ic WHERE ic.geom IS NOT NULL
    ) AS min_dist
    FROM parks p WHERE p.total_output_yi > 0
) sub
GROUP BY p.street
ORDER BY p.street;
```

---

## 企业生命周期空间行为

### 3.1 核心命题

> **"头部企业离路建园"** — 产值 TOP N 企业中，大多数不在交通要道 500m 带内，而是拥有独立园区或地块。

这揭示了企业生命周期中的空间行为规律：

```
路边500m → 中小企业孵化器（交通便利、低门槛、配套密集）
    ↓ 成长
过渡带 → 扩张期企业（需要更大空间，但仍需交通可达）
    ↓ 成熟
独立园区 → 头部企业"毕业"（自建园区、产业链自成体系）
```

### 3.2 TOP 企业离路验证 SQL

```sql
-- TOP N 企业的沿路分布
WITH key_roads AS (
    SELECT ST_Union(geom) AS geom
    FROM road_network r
    JOIN region_boundary b ON ST_Intersects(r.geom, b.geometry)
    WHERE road_class IN ('motorway', 'trunk', 'primary')
),
top_enterprises AS (
    SELECT enterprise_name, street, output_clean, geom,
           ROW_NUMBER() OVER (ORDER BY output_clean DESC) AS rank
    FROM enterprises
    WHERE output_clean > 0 AND geom IS NOT NULL
    ORDER BY output_clean DESC
    LIMIT 20
)
SELECT 
    rank,
    enterprise_name,
    output_clean,
    street,
    CASE 
        WHEN ST_DWithin(te.geom::geography, kr.geom::geography, 500)  THEN '带内≤500m'
        WHEN ST_DWithin(te.geom::geography, kr.geom::geography, 2000) THEN '过渡带500m-2km'
        ELSE '远离>2km'
    END AS road_zone,
    -- 是否在园区内
    (SELECT COUNT(*) > 0 FROM parks p 
     WHERE ST_Within(te.geom, p.park_geom)) AS in_park
FROM top_enterprises te, key_roads kr
ORDER BY rank;
```

### 3.3 系别空间聚合分析

企业集团（系别）的空间分布反映产业组织模式：

```sql
-- 各集团/系别的空间分布
SELECT 
    group_name,
    COUNT(*) AS subsidiary_count,
    COUNT(DISTINCT street) AS street_coverage,
    ROUND(SUM(output_clean), 1) AS total_output_yi,
    -- 空间集中度：子公司是否集中在同一街道
    ROUND(MAX(street_share), 1) AS max_street_concentration_pct
FROM (
    SELECT group_name, street, output_clean,
           COUNT(*)::numeric / SUM(COUNT(*)) OVER (PARTITION BY group_name) * 100 AS street_share
    FROM enterprises
    WHERE group_name IS NOT NULL
    GROUP BY group_name, street, output_clean
) sub
GROUP BY group_name
HAVING COUNT(*) >= 2
ORDER BY total_output_yi DESC;
```

### 3.4 生命周期阶段的空间特征

| 生命周期阶段 | 企业规模 | 空间偏好 | 园区类型 | 政策启示 |
|:------------:|:--------:|----------|----------|----------|
| 初创期 | <0.5亿 | 路边/产业园孵化器 | 商务办公园、科创园 | 降低租金门槛 |
| 成长期 | 0.5-5亿 | 过渡带/标准厂房 | 工业生产园 | 弹性空间供应 |
| 成熟期 | 5-50亿 | 独立楼层/专属园区 | 大型产业园 | 产业链配套招商 |
| 头部期 | >50亿 | 独立园区/地块 | 自建园区 | 供地+政策包 |

---

## 空间带假说验证方法

### 4.1 适用范围

当规划提出"X 轴 Y 带 Z 片区"等空间分区框架时，需先用企业点位数据检验该分区在统计上是否成立。

### 4.2 四步验证流水线

**Step 1: 经度/纬度分箱核密度** — 检测方向性梯度

```python
import numpy as np

# 按 0.01°（≈1km）分箱，统计各箱制造/服务占比
bins = np.arange(min_lng, max_lng + 0.01, 0.01)

for lo, hi in zip(bins[:-1], bins[1:]):
    in_bin = [e for e in enterprises if lo <= e['lng'] < hi]
    manufacturing = sum(1 for e in in_bin if e['type'] == '制造业')
    service = sum(1 for e in in_bin if e['type'] == '服务业')
    ratio = manufacturing / service if service > 0 else float('inf')
    
# 判断逻辑：
# - 制造业占比随经度单调变化 → 东西分异成立
# - 呈 U 型（两端制造高、中间服务高）→ 不支持简单东西二分
```

**Step 2: 关键道路分界检验** — 测试"路 = 带边界"假说

```python
# 以某条主要道路经度为界，统计两侧制造业占比
divide_lng = <道路平均经度>

cur.execute("""
    SELECT 
        CASE WHEN lng < %s THEN '西' ELSE '东' END AS side,
        COUNT(*),
        COUNT(CASE WHEN industry_category = '制造业' THEN 1 END) AS mfg_count
    FROM enterprises
    WHERE street IN (<区域街道列表>)
    GROUP BY side
""", (divide_lng,))

# 若东西两侧制造占比差 < 5pp → 该路不是功能分界
```

**Step 3: 街道级加权梯度** — 找真正的功能分异方向

```python
# 按街道计算制造业占比，排名后聚类
streets_ranked = sorted(streets, key=lambda s: s['mfg_pct'], reverse=True)

# 按占比聚类为 N 带
north_zone = [s for s in streets if s['mfg_pct'] > 85]   # 制造带
mid_zone   = [s for s in streets if 70 < s['mfg_pct'] <= 85]  # 混合带
south_zone = [s for s in streets if s['mfg_pct'] <= 70]  # 服务带

# 加权平均各带制造占比
for zone_name, zone_streets in [('北带', north_zone), ('中带', mid_zone), ('南带', south_zone)]:
    weighted_mfg = sum(s['mfg'] for s in zone_streets) / sum(s['total'] for s in zone_streets)
    print(f"{zone_name}: 制造占比 {weighted_mfg:.1%}")
```

**Step 4: 输出结论 + 可视化**

- 输出各假说的检验结论（✅成立 / ❌不成立）与统计差异
- 给出推荐的带结构（附梯度值）
- 标注对规划方案的影响

---

## 已知反直觉发现

### 发现1: 头部企业"离路建园"

> 产值 TOP10 企业中，大部分不在交通要道 500m 带内，而是拥有独立园区/地块。

**解释**：路边 500m 是中小企业的孵化器，头部企业"毕业"后独立建园。这是产业发展的自然规律，不应解读为"头部企业没有沿路偏好"。

**政策启示**：
- 路边空间：优先保障中小创新企业，做"孵化-加速"功能
- 独立园区：为成长到一定规模的企业提供供地通道
- 不要因为沿路带内无头部企业就否定沿路产业带的价值

### 发现2: 过渡带产值最高

> 500m-2km 过渡带平均产值通常高于带内和远离带。

**解释**：过渡带"近而不贴"——享受交通可达性但不受路侧噪音/拥堵影响，是成熟企业的理想区位。

**政策启示**：
- 工业用地出让优先考虑过渡带区位
- 过渡带做"标准厂房 + 公共配套"组合
- 带内做轻量化的商务办公/科创空间

### 发现3: 战新产业无明显沿路偏好

> 大部分战新产业在带内占比与全区均值一致，仅部分产业（如低空经济）略有偏好。

**解释**：战新产业的企业选址由创新生态（靠近高校/研究院/创新载体）驱动，而非交通可达性。传统制造业的沿路偏好更显著。

### 发现4: 交通干线不一定是功能分界

> 纵向交通干线穿越多个功能带，不是带的边界。产业功能梯度可能是横向而非纵向。

**验证方法**：经线分界统计 → 若两侧产业占比差异 < 5pp → 该道路不是功能分界。

---

## 常见陷阱与避坑清单

### ST_DWithin 坐标系陷阱

```sql
-- ❌ 4326坐标系中500单位=500度≈半个地球
ST_DWithin(geom, road_geom, 500)

-- ✅ 加 ::geography 做米级距离
ST_DWithin(geom::geography, road_geom::geography, 500)
```

### 空间关联方法选择

| 方法 | 适用场景 | 注意事项 |
|------|----------|----------|
| `ST_Within` | 点在多边形内 | 点在边界上不会被重复计数 |
| `ST_Intersects` | 面面/线面相交 | 点在边界上会被多个面同时计数 → 重复 |
| `ST_DWithin` | 距离筛选 | 必须用 `::geography` 做米级单位 |

### road_length 字段

- `"Shape_Leng"` 字段**大写 L**，必须带表前缀 `rd."Shape_Leng"`
- 若报 `column does not exist`，检查大写和表前缀

### 产值字段清洗

```sql
-- ⚠️ "24年产值"含中文数字混合 → 正则清洗
CASE WHEN output_raw ~ '^[0-9.]+$' THEN output_raw::numeric
     ELSE NULL END

-- 区间格式 "0.03-0.05" → 取中值
CASE WHEN output_raw ~ '^[0-9.]+-[0-9.]+$' THEN
    (SPLIT_PART(output_raw, '-', 1)::numeric + 
     SPLIT_PART(output_raw, '-', 2)::numeric) / 2.0
END
```

### 统计口径陷阱

- 企业底表通常为四上企业口径，非全量企业
- 路边带内"企业少"可能因为小微企业未纳入统计
- 空间分析报告必须注明统计口径和数据来源

### 创新载体编码问题

- 部分数据源的中文列名可能存在编码问题（如 UTF-8 → Latin-1 → UTF-8 双编码乱码）
- 修复方法：`val.encode('latin-1').decode('utf-8')`
- 建索引前用 `cursor.description` 确认列名实际编码
