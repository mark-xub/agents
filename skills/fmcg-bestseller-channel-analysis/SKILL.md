---
name: fmcg-bestseller-channel-analysis
description: 快消爆品「商家 × 商品」粒度的渠道曝光/点击/成交分析，以及 Push 任务的强意图成交分析。以 gift_item_pool 生效在投清单为准圈定商家商品，用事件流水表拆分 Banner/Push/SMS/美妆pop(hotpage) 四渠道点击与成交，用 push 明细表做强意图归因。适用于用户提到"爆品渠道分析""商家商品渠道数据""各渠道曝光点击成交""banner push sms hotpage 美妆pop""push 强意图成交""渠道成交拆分""在投商品渠道效果"等场景时触发。
dependencies:
  skill:
    - odps-skill
---

# 快消爆品「商家 × 商品」渠道分析

## 目标

本 Skill 服务三类分析目标：

1. **商家 × 商品各渠道曝光/点击/成交**：以 `gift_item_pool` 圈定"截至目标日已开始投放"的商家商品，逐商品拆分 Banner / Push / SMS / 美妆pop(hotpage) 四渠道的曝光、点击、成交。
2. **Push 任务强意图成交分析**：用 push 明细表解析出 item_id / 任务，结合强意图触发口径，分析强意图人群的 push 触达→点击→成交链路。
3. **大盘成交买家漏斗分析（渠道归因桑基图）**：以大盘成交用户为起点，追踪「大盘成交 → 切端触发 → 强意图识别 → 未购 → Push触发」多层漏斗，用 ECharts Sankey 可视化。

---

## 执行方式（已从 MCP 切换到 odps-skill）

⚠️ **重要变更**：读取 ODPS 的通道已由原来的 MCP `maxcompute-mcp-server` 换成 **odps-skill**。所有 SQL 都通过 odps-skill 的本地脚本执行，不再走 MCP 工具。

skill 脚本路径：`/Users/mark/.agents/skills/odps-skill/scripts/pyodps.py`

### 三种执行方式

1. **单条短 SQL** → 用 `-e`：
   ```bash
   uv run /Users/mark/.agents/skills/odps-skill/scripts/pyodps.py -e "select ... limit 20;"
   ```

2. **多行 / 复杂 SQL** → 写入当前工作目录的 `.sql` 文件，用 `-f`：
   ```bash
   uv run /Users/mark/.agents/skills/odps-skill/scripts/pyodps.py -f ./query.sql
   ```

3. **需要 SET 与 SELECT 同批提交（push 明细表解析专用）** → 必须写 python 脚本用 `hints` co-submit，见下文「Push 明细表解析」章节。

### ⚠️ 关键限制：`-e` / `-f` 不能让 SET 生效于后续 SELECT

`pyodps.py` CLI 按 `;` 拆分逐条提交，每条 SQL 是**独立 instance**。因此 `set xxx=true; select ...;` 里的 SET flag **不会**作用到后面的 SELECT——SET 在它自己的 instance 里就丢了。

- 对**不需要 SET** 的普通查询（gift_item_pool、事件流水表），`-e` / `-f` 完全够用。
- 对**必须带 SET** 的 push 明细表解析（`odps.sql.udf.getjsonobj.new=true` 等），**只能**走 python `hints` 脚本，不能用 `-e`/`-f` 拼 `set; select;`（会得到假的空结果）。

### 大结果落盘

宽查询或导出场景，第一次运行就直接重定向到当前工作目录文件，再抽查前几行；不要整块贴回：
```bash
uv run /Users/mark/.agents/skills/odps-skill/scripts/pyodps.py -f ./query.sql > ./query.out 2>&1
```
需要易解析表格时，SQL 里加 `set odps.sql.select.output.format=csv;`（此 SET 与本条 SELECT 同批，属于 CLI 支持范围）。

### PyODPS 长查询：异步提交 + 轮询模式

跨项目 JOIN 或大表扫描可能跑数分钟甚至超时。此时 **不能用 `instance.wait_for_success()`**（阻塞且受 read_timeout 影响），必须写独立 Python 脚本做异步提交+轮询。

**⚠️ 关键陷阱：read_timeout 必须设短（30s），不能设长（600s）**

`instance.reload()` 是一次 HTTP 调用，受 `options.read_timeout` 控制。如果设 `read_timeout=600`，每次 `reload()` 会阻塞最多 600s，轮询循环形同虚设。正确做法是 `read_timeout=30`，让每次状态检查快速返回，用 `time.sleep(10)` 控制轮询间隔。

```python
options.connect_timeout = 30
options.read_timeout = 30    # ← 必须短，让 reload() 快速返回
options.retry_times = 3
```

**提交阶段不受 read_timeout 影响**：`odps.execute_sql()` 提交请求只返回 instance ID，HTTP 请求很小，即使 read_timeout=30 也够用。

**轮询模板**：

```python
instance = odps.execute_sql(sql, project=project)
print(f"Instance ID: {instance.id}")

start = time.time()
max_wait = 600
elapsed = 0
while elapsed < max_wait:
    elapsed = time.time() - start
    try:
        instance.reload()
        if instance.is_terminated():
            break
    except Exception as e:
        print(f"  [{elapsed:.0f}s] Poll error (will retry): {e}")
    time.sleep(10)

if not instance.is_successful():
    print("Query FAILED")
    sys.exit(1)

# 读取结果
with instance.open_reader() as reader:
    for record in reader:
        print("\t".join(str(v) for v in record.values))
```

参考实现：`outputs/run_maogeping_async.py`。

### ⚡ 跨项目 JOIN 优化三板斧

跨项目 JOIN（如 `amp_im` × `wlods`，后者 778M 行）如果不优化会跑 15+ 分钟甚至超时。三个关键优化手段：

1. **VALUES CTE 替代 IN 列表**：用小表 `FROM VALUES (...) AS t(col1, col2)` 做品牌字典，避免在 SQL 里写巨型 `CASE WHEN ... IN (...)` 列表。单源维护，不易出错。

   ```sql
   WITH brand_map AS (
       SELECT item_id, brand
       FROM VALUES
           ('566446518116', '娇兰'), ('651588553248', '卡姿兰'), ...
           AS t(item_id, brand)
   )
   ```

2. **MAPJOIN 广播小表**：`/*+ MAPJOIN(bm) */` 让 brand_map（~19 行）广播到所有 worker，避免 shuffle。

   ```sql
   SELECT /*+ MAPJOIN(bm) */ DISTINCT r.user_id, bm.brand
   FROM big_table r JOIN brand_map bm ON r.itemid = bm.item_id
   ```

3. **预聚合后再 JOIN**：在事件表侧先用 `MAX(CASE WHEN ... THEN 1 ELSE 0 END)` 把事件级行折叠为用户级 0/1 标志，再做跨项目 JOIN。这样 shuffle 数据量从事件行数降为用户行数（量级差 10~100x）。

   ```sql
   user_intent AS (
       SELECT e.user_id,
              MAX(CASE WHEN e.tppid = '32404' THEN 1 ELSE 0 END) AS strong_flag,
              MAX(CASE WHEN ... AND e.result = 'succ' THEN 1 ELSE 0 END) AS push_flag
       FROM wlods.rt_intent_event_tt_send_log e
       JOIN brand_map bm ON ...
       WHERE e.ds = '...'
       GROUP BY e.user_id     -- 预聚合到用户级
   )
   -- 跨项目 JOIN 时只有用户行数，不是事件行数
   SELECT b.brand, SUM(u.strong_flag), SUM(u.push_flag)
   FROM market_buyers b JOIN user_intent u ON b.user_id = u.user_id
   ```

**效果实测**：7品牌全量 JOIN 从 15+ 分钟超时 → 优化后 ~10 秒完成。

### 🗂 大表查询标准模式：中间表拆分

**原则：凡是扫描超过 1 亿行的大表查询，一律拆成中间表，每步独立提交。**

原因：单条 SQL 里多步 JOIN/聚合如果卡住，无法判断瓶颈在哪一步。拆成中间表后：
- 每步有独立的 instance ID 和执行时间，**可以精确定位慢在哪**
- 中间结果可复用（换口径、换日期、换品牌不用重扫大表）
- 失败重试成本从"重跑整条链"降到"重跑某一步"

**标准拆分模式**：

```
Step 1: 大表扫描 + 预聚合 → 写中间表（重活，只做一次）
Step 2: 中间表 JOIN 业务表 → 出结果（轻量，秒级）
Step 3: （可选）结果表 JOIN 维度表 → 最终输出
```

**实现方式**：用 `CREATE TABLE ... AS SELECT` 写中间表，生命周期设短（如 `lifecycle 7`）：

```sql
-- Step 1: wlods 多日扫描 → 预聚合到用户级（重活）
CREATE TABLE IF NOT EXISTS ${project}.tmp_user_intent_7d LIFECYCLE 7 AS
SELECT  /*+ MAPJOIN(bm) */
        e.user_id,
        MAX(CASE WHEN e.tppid = '32404' THEN 1 ELSE 0 END) AS strong_flag,
        MAX(CASE WHEN e.tppid = '32404' AND e.result = 'succ' THEN 1 ELSE 0 END) AS push_flag
FROM    wlods.rt_intent_event_tt_send_log e
LATERAL VIEW EXPLODE(SPLIT(REGEXP_REPLACE(e.item_list, '\[|\]|"', ''), ',')) t AS item_raw
JOIN    brand_map bm ON TRIM(t.item_raw) = bm.item_id
WHERE   e.ds BETWEEN '${start_ds}' AND '${end_ds}'
AND     e.intent_code IN ('strongPurchase', 'comparePrices_v1')
GROUP BY e.user_id;

-- Step 2: 轻量 JOIN（秒级完成）
SELECT  b.brand,
        COUNT(DISTINCT b.user_id) AS buyer_uv,
        SUM(COALESCE(u.strong_flag, 0)) AS strong_intent_uv,
        SUM(COALESCE(u.push_flag, 0)) AS push_covered_uv
FROM    market_buyers b
LEFT JOIN tmp_user_intent_7d u ON b.user_id = u.user_id
GROUP BY b.brand;
```

**适用判断**：

| 场景 | 是否需要中间表 |
|------|-------------|
| 单分区扫描 + 简单聚合 | 不需要，直接 `-f` |
| 单分区跨项目 JOIN（优化三板斧后 ~10s） | 不需要，一条 SQL 搞定 |
| **多分区扫描 + 跨项目 JOIN**（如 7 天 wlods） | **需要，拆中间表** |
| 多步 JOIN + 多张维度表 | 需要，每步一个中间表 |

**实测对比**：7 天 wlods 扫描（54 亿行），单条 SQL 跑 ~16 分钟（含排队），拆中间表后 Step 1 约 2~3 分钟、Step 2 秒级完成，且 Step 1 结果可多次复用。

---

## 数据表总览

| 表名 | 分区 | 角色 | 关键说明 |
|------|------|------|---------|
| `amp_im.gift_item_pool` | `ds` | **圈定在投商家商品**（目标 1 起点） | attribute 单层非转义 JSON；startTime 毫秒 epoch；env 仅 online/prepare |
| `amp_im.s_tt_gift_benefit_record_tt4_parsed` | **`dt`** | **各渠道点击/成交 + 大盘买家**（目标 1/3 核心） | 分区是 dt 不是 ds；产出滞后 1~2 天；source='-' 无归因须排除 |
| `wlods.rt_intent_event_tt_send_log` | `ds` | **意图事件漏斗**（目标 3 核心） | 778M+ 行；intent_code 过滤意图类型；tppid='32404' 为强意图；item_list 需 EXPLODE+REGEXP_REPLACE 解析；p8_items 记录已购；result='succ' 为 Push 成功 |
| `tbcdm.dwd_tb_zyy_log_msg_push_di` | `ds` | **Push 明细/强意图归因**（目标 2 核心） | tag 多层转义 JSON；解析必须 SET 三个 flag 且 hints 同批提交 |
| `amp_im.ads_fmcg_brand_channel_report_di2` | `ds` | 品牌级渠道漏斗聚合（辅助/曝光补充） | 品牌×日，含 banner 曝光、强意图触发、大盘成交 |
| `tbbi.tmp_ads_tb_udec_msg_banner_expos_activity_hh` | `ds`+`hh` | **Banner 曝光/点击明细（商品级曝光，首选）** | task_id 固定 '601'；**business_id = itemId**；arg1 区分曝光/点击；UV 用 long_login_user_id；全天取 hh<='23'；列级 LABEL 权限已开通（2026-08-10） |
| `tbbi.ads_tb_udec_msg_log_banner_hh` | `ds` | Banner 曝光/点击明细（历史表，business_id=merchant_id 口径） | 用 business_id(=merchant_id) 关联；需 SELECT+LABEL2 敏感字段权限；已被上面 tmp 表口径替代 |

---

## 目标 1：商家 × 商品各渠道曝光/点击/成交

分三步：先圈定在投清单 → 再拆渠道点击/成交 → 曝光补充。

### 步骤 1：圈定"截至目标日已开始投放"的商家商品（gift_item_pool）

#### 取数规则

1. **生效过滤**：`status='1'`（ACTIVE）+ `env='online'`。本表 `env` 实际取值只有 `online` / `prepare`，**没有 `prod`**，卡 `env='prod'` 会查空。
2. **attribute 解析**：`attribute` 是**单层非转义 JSON**（与 push 明细表那种嵌套转义不同），`GET_JSON_OBJECT` 可直接取值，**无需任何 SET flag**，因此可直接用 `-e`/`-f`。常用字段：`itemTitle`、`startTime`（**毫秒级 epoch 字符串**，如 `"1785772800000"`）、`endTime`、`startPrice`。
3. **已开始过滤（务必卡准边界）**：只保留 `startTime <= 目标时点` 的商品。
   - 取"当前实时"：`CAST(start_time_ms AS BIGINT) <= UNIX_TIMESTAMP(GETDATE())*1000`
   - 取"截至历史某日 D"（含 D 当天已起投）：用毫秒常量 `CAST(start_time_ms AS BIGINT) < [D 次日 00:00 毫秒值]`，即边界 = **D+1 天 00:00:00（东八区）**，切勿多算一天。
   - ⚠️ 反例：想卡"截至 8/3"却误用 8/5 的毫秒值 `1785859200000`，会把 8/4 才起投的自由点错误纳入。正确边界应是 8/4 00:00 = `1785772800000`。实测：娇兰(8/1)、卡姿兰(8/3) 入选，自由点(8/4) 剔除，故"截至 8/3 在投"仅娇兰+卡姿兰 2 个商品。
4. **品牌名**：本表**无独立品牌字段**，品牌信息混在 `itemTitle` 长文本里。用 `merchant_id → brand` 的 CASE 映射固化（内容从各商家 itemTitle 归纳）；**新商家进来时需在 CASE 里补一行**。同一商家名下所有 item 品牌一致。
5. **banner 关联键**：商品级 Banner 曝光用 `tbbi.tmp_ads_tb_udec_msg_banner_expos_activity_hh`，**`business_id` = itemId**（CAST 成 STRING 与 item_id 关联），task_id 固定 '601'。历史表 `tbbi.ads_tb_udec_msg_log_banner_hh` 的 business_id 是 merchant_id，口径不同，别混用。

#### 生效商家 × 商品清单 SQL

```sql
SELECT  merchant_id,
        CASE merchant_id
          WHEN '2943025980' THEN '娇兰'
          WHEN '755579902'  THEN '卡姿兰'
          WHEN '1785158051' THEN '自由点'
          WHEN '66472471'   THEN '洁婷'
          WHEN '2081848447' THEN '可丽金'
          WHEN '2424298091' THEN '海蓝之谜'
          WHEN '839895996'  THEN '毛戈平'
          WHEN '2220413020394' THEN '薇尔'
          WHEN '1768562735' THEN '娇韵诗'
          ELSE NULL
        END AS brand,
        item_id,
        item_title,
        start_time_ms,
        FROM_UNIXTIME(CAST(start_time_ms AS BIGINT)/1000) AS start_time,
        benefit_cnt,
        max_ds
FROM (
    SELECT  merchant_id, item_id,
            GET_JSON_OBJECT(MAX(attribute),'$.itemTitle')   AS item_title,
            GET_JSON_OBJECT(MAX(attribute),'$.startTime')   AS start_time_ms,
            COUNT(DISTINCT benefit_item_id)                 AS benefit_cnt,
            MAX(ds)                                         AS max_ds
    FROM    amp_im.gift_item_pool
    WHERE   ds='${bizdate}'
      AND   status='1'
      AND   env='online'
    GROUP BY merchant_id, item_id
) t
WHERE start_time_ms IS NOT NULL
  AND CAST(start_time_ms AS BIGINT) <= UNIX_TIMESTAMP(GETDATE())*1000
ORDER BY merchant_id, item_id;
```

> 卡历史日（如"截至 8/3"）时，把最后过滤换成毫秒常量比较 `CAST(start_time_ms AS BIGINT) < 1785772800000`（= 8/4 00:00:00），避免 `UNIX_TIMESTAMP(DATE)` / 两参数 `UNIX_TIMESTAMP` 的类型与 hive 兼容报错。

执行：
```bash
uv run /Users/mark/.agents/skills/odps-skill/scripts/pyodps.py -f ./scope.sql
```

从结果取得 `item_id` 清单，作为步骤 2 的 `${in_scope_item_ids}` 与 `merchant_id` 清单（步骤 3 曝光）。

#### 已知 merchant_id → brand 映射

| merchant_id | brand | 依据 itemTitle |
|-------------|-------|---------------|
| 2943025980 | 娇兰 | 娇兰帝皇蜂姿复原蜜精华… |
| 755579902 | 卡姿兰 | 卡姿兰黑磁散粉定妆粉… |
| 1785158051 | 自由点 | 自由点益生菌卫生巾… |
| 66472471 | 洁婷 | 洁婷樱花卫生巾… |
| 2081848447 | 可丽金 | 可丽金胶原大膜王涂抹面膜… |
| 2424298091 | 海蓝之谜 | 海蓝之谜奇迹面霜/精萃水礼盒… |
| 839895996 | 毛戈平 | 毛戈平鱼子酱面膜/气垫/粉饼… |
| 2220413020394 | 薇尔 | 薇尔纯棉卫生巾/安睡裤… |
| 1768562735 | 娇韵诗 | 娇韵诗双萃精华液/眼精华… |

#### 已知 item_id → brand 映射（目标 3 买家漏斗专用）

| brand | item_id 列表 |
|-------|-------------|
| 娇兰 | 566446518116 |
| 卡姿兰 | 651588553248 |
| 可丽金 | 638131550168 |
| 海蓝之谜 | 43984164814, 649652424111, 650605693497, 805251110686 |
| 毛戈平 | 14021638771, 537219932625, 679396365892, 740202067434, 781646174250 |
| 娇韵诗 | 558744903450, 649916182962, 707440056381 |
| 兰蔻 | 1058038587039, 1060753340704, 1060755092498, 1064360618638 |

> 新品牌/新商品入场时在上表补行，同时更新 VALUES CTE。

### 步骤 2：各渠道点击/成交（s_tt_gift_benefit_record_tt4_parsed）

事件流水表，字段：`action`（CLICK/ORDER/ISSUE_BENEFIT）、`source`、`itemid`、`user_id`、`event_time`。

**source 业务映射（重要）**：
- `push` = Push
- `banner` = 资源位
- `default` = 短信
- `hotpage` = 美妆pop
- `-` = 未打渠道归因

**大盘 vs 分渠道口径（重要）**：
- **大盘成交 = 不限定 source（含 `-`）的全部 ORDER**。不要把 `-` 单独叫"自然成交"当子集看，也不要用它替代大盘。
- **分渠道 = 限定 source 对应值**（push/banner/default/hotpage）。四渠道成交合计 ≤ 大盘，差额即未归因（`-`）部分。

#### 三条必踩坑（务必先读）

1. **分区字段是 `dt`，不是 `ds`**。误用 `ds` 过滤会得到假的空结果——最容易踩的坑。
2. **产出有延迟**，最新可用分区常滞后当前日期 1~2 天（如 2026-08-04 查询时最新分区仅到 `dt='20260802'`）。跑数前先探最新分区，不要硬套"今天/昨天"：
   ```sql
   SELECT dt, COUNT(*) cnt FROM amp_im.s_tt_gift_benefit_record_tt4_parsed
   WHERE dt BETWEEN '${start_dt}' AND '${end_dt}' GROUP BY dt ORDER BY dt;
   ```
3. **大盘成交别漏掉 `source='-'`**。做分渠道拆分时限定 push/banner/default，但算大盘时**不能过滤 source**（含 `-` 才是全量成交）。若某商品成交几乎全落 `-`，说明其渠道触达尚未启用或未打通归因。

#### 渠道点击/成交 + 大盘汇总 SQL（一次出全）

item 清单取自步骤 1（不要写死白名单）。本表不需要 SET，可直接 `-e`/`-f`。用 `GROUPING SETS` 同时出「分渠道」和「大盘（不限 source）」两行口径，`src='ALL'` 即大盘：

```sql
SELECT  itemid,
        COALESCE(source,'ALL')                                       AS src,
        SUM(CASE WHEN action='CLICK' THEN 1 ELSE 0 END)              AS click_pv,
        COUNT(DISTINCT CASE WHEN action='CLICK' THEN user_id END)    AS click_uv,
        SUM(CASE WHEN action='ORDER' THEN 1 ELSE 0 END)              AS order_pv,
        COUNT(DISTINCT CASE WHEN action='ORDER' THEN user_id END)    AS order_uv
FROM    amp_im.s_tt_gift_benefit_record_tt4_parsed
WHERE   dt='${latest_dt}'                        -- 用探到的最新分区
  AND   itemid IN (${in_scope_item_ids})         -- 来自步骤 1；注意不过滤 source
GROUP BY itemid, GROUPING SETS((source),())      -- source 各值 + 大盘(不限)
ORDER BY itemid, src;
```

> 若只要分渠道、不要大盘，去掉 GROUPING SETS 改 `GROUP BY itemid, source` 并加 `AND source IN ('push','banner','default','hotpage')` 即可。PV=事件数，UV=去重人数。

**实测证据（dt=20260802）**：娇兰 566446518116 → 资源位(banner) 点击 55UV/58PV、成交 1UV，push 点击 22UV、成交 0，大盘(ALL) 成交 40UV/42PV（= 资源位归因 1 + 未归因 39）；卡姿兰 651588553248 → 三渠道全 0，大盘成交 974UV/1017PV 全落未归因 `-`（渠道未打通归因）。

### 步骤 3：曝光补充

事件流水表主要承接点击/成交，**曝光**需从曝光侧表补：

- **Banner 曝光/点击（商品级，首选）**：`tbbi.tmp_ads_tb_udec_msg_banner_expos_activity_hh`，分区 `ds`+`hh`（全天取 `hh<='23'`）。口径：**`task_id` 固定 '601'**（Banner 任务）；**`business_id` = itemId**（CAST STRING 后与在投清单 item_id 关联）；`arg1='Page_MsgCenter_PrimeActivityNotice_Show'` = 曝光，`arg1='Page_MsgCenter_PrimeActivityNotice_ActPoint-Click'` = 点击；UV 用 `long_login_user_id` 去重。**列级 LABEL 权限已开通（2026-08-10，此前报错 CheckLabelSecurity：`long_login_user_id` 需 LABEL3，`arg1`/`task_id`/`business_id` 需 LABEL2）**；表内另有独立 `item_id` 列，可与 business_id 交叉核对。交叉校验：曝光表 click_uv 与事件流水表资源位 click_uv 量级一致（如娇兰 5,219 vs 5,115），两套埋点口径吻合。完整模板见 `outputs/banner_expos.sql`：

**模式 A：固定一天**（ds = 目标日）：

```sql
-- Banner 曝光/点击（商品级）：task_id='601'，business_id=itemId
SELECT  c.brand, c.item_id,
        COUNT(DISTINCT CASE WHEN t.arg1='Page_MsgCenter_PrimeActivityNotice_Show'
              THEN t.long_login_user_id END)          AS banner_imp_uv,
        SUM(CASE WHEN t.arg1='Page_MsgCenter_PrimeActivityNotice_Show'
              THEN 1 ELSE 0 END)                      AS banner_imp_pv,
        COUNT(DISTINCT CASE WHEN t.arg1='Page_MsgCenter_PrimeActivityNotice_ActPoint-Click'
              THEN t.long_login_user_id END)          AS banner_click_uv,
        SUM(CASE WHEN t.arg1='Page_MsgCenter_PrimeActivityNotice_ActPoint-Click'
              THEN 1 ELSE 0 END)                      AS banner_click_pv
FROM    tbbi.tmp_ads_tb_udec_msg_banner_expos_activity_hh t
JOIN    item_scope c                       -- 步骤 1 的在投清单（item_id + brand）
  ON    CAST(t.business_id AS STRING) = c.item_id
WHERE   t.ds = '${bizdate}'
  AND   t.hh <= '23'
  AND   CAST(t.task_id AS STRING) = '601'
GROUP BY c.brand, c.item_id;               -- 品牌级则只 GROUP BY c.brand
```

**模式 B：商品上线之后到目标日的所有天**（每个商品按自己起投日 start_ds 起算，ds ∈ [start_ds, ${end_date}]）：

```sql
-- item_scope 在步骤 1 的基础上多带一列起投日 start_ds：
--   TO_CHAR(FROM_UNIXTIME(CAST(start_time_ms AS BIGINT)/1000),'yyyymmdd') AS start_ds
SELECT  c.brand, c.item_id,
        MIN(t.ds) AS first_ds, MAX(t.ds) AS last_ds,   -- 实际有数据的起止日（可选）
        COUNT(DISTINCT CASE WHEN t.arg1='Page_MsgCenter_PrimeActivityNotice_Show'
              THEN t.long_login_user_id END)          AS banner_imp_uv,
        SUM(CASE WHEN t.arg1='Page_MsgCenter_PrimeActivityNotice_Show'
              THEN 1 ELSE 0 END)                      AS banner_imp_pv,
        COUNT(DISTINCT CASE WHEN t.arg1='Page_MsgCenter_PrimeActivityNotice_ActPoint-Click'
              THEN t.long_login_user_id END)          AS banner_click_uv,
        SUM(CASE WHEN t.arg1='Page_MsgCenter_PrimeActivityNotice_ActPoint-Click'
              THEN 1 ELSE 0 END)                      AS banner_click_pv
FROM    tbbi.tmp_ads_tb_udec_msg_banner_expos_activity_hh t
JOIN    item_scope c
  ON    CAST(t.business_id AS STRING) = c.item_id
WHERE   t.ds >= c.start_ds                 -- 每个商品从自己上线日开始
  AND   t.ds <= '${end_date}'
  AND   t.hh <= '23'
  AND   CAST(t.task_id AS STRING) = '601'
GROUP BY c.brand, c.item_id;
-- 起投日晚于 ${end_date} 的商品 start_ds > end_date，天然无行，无需额外过滤；
-- 若要严格卡"截至 end_date 在投"，仍按步骤 1 的毫秒常量边界过滤。
-- 跨天 UV 为整个区间去重人数（≠ 各天 UV 之和）。
```

- **权限容错（务必）**：banner 曝光查询**单独一步执行**，若偶发 `NoPermission`（列级 LABEL 权限已于 2026-08-10 开通，正常可读），**不要中断整体流程**——该表各列按空处理（0 或「—」），报告注明"banner 曝光表暂无权限，数据暂为空"；事件流水表的点击/成交照常输出。
- **历史口径（已替代）**：`tbbi.ads_tb_udec_msg_log_banner_hh`，business_id = merchant_id，需 SELECT + LABEL2 权限；仅在 tmp 表不可用时参考。
- **品牌级曝光/强意图（辅助）**：`amp_im.ads_fmcg_brand_channel_report_di2`（品牌×日，分区 `ds`），字段 `banner_imp_uv`、`intent_trigger_uv`、`push_send_uv`、`push_arrive_uv` 等，用于在商品级曝光缺权限时做品牌级兜底。

### 目标 1 输出格式

先按分析窗口过滤商品：**只输出起投时间 ≤ 窗口截止日的商品**（如窗口=8/3，则剔除 8/4 才起投的商品），不在窗口内的商品一律不输出。

指标口径：**点击、成交统一用 UV（去重人数）**，各渠道按「曝光 → 点击 → 成交」顺序排列并计算 **uCTR / uCVR**。列布局（资源位曝光与点击成交同组）：

```
| 品牌 | item_id | 资源位曝光UV | 资源位点击UV | 资源位成交UV | 资源位uCTR | 资源位uCVR | Push送达UV | Push点击UV | Push成交UV | Push uCTR | Push uCVR | 短信点击UV | 短信成交UV | 短信uCVR | 美妆pop点击UV | 美妆pop成交UV | 美妆popuCVR | 渠道成交UV | 大盘成交UV | 渠道渗透率 |
```

- **渠道成交UV** = 资源位 + Push + 短信 + 美妆pop 四渠道归因成交之和（放在表尾、大盘成交之前）。
- **渠道渗透率** = 渠道成交UV ÷ 大盘成交UV，反映渠道归因对大盘成交的渗透/打通程度（越低说明越多成交未归因）。大盘为 0 记「—」。

- **资源位（Banner）曝光** = `tmp_ads_tb_udec_msg_banner_expos_activity_hh` 的 `banner_imp_uv`（task_id='601'、business_id=itemId）；点击/成交 = 事件流水表 `source='banner'`。曝光与点击成交分属两套埋点，量级一致即可。
- **Push 无曝光数据**：用 **Push 送达UV** 代替曝光列（push 明细表 arrive_uv，默认限爆品任务 `mp_task_id='47939001'`）。Push uCTR 分母 = 送达UV。
- **短信无曝光数据**：仅列点击/成交，不算 uCTR（无分母），只算 uCVR。
- **美妆pop（source='hotpage'）无曝光数据**：与短信类似，仅列点击/成交，不算 uCTR，只算 uCVR。美妆pop是美妆行业 pop 弹窗渠道，事件流水表中 `source='hotpage'`。
- **uCTR** = 点击UV ÷ 曝光UV（Push 用送达UV作分母）；**uCVR** = 成交UV ÷ 点击UV。**分母为 0 记「—」**，不要用 0 或报错填充。
- **大盘成交** = 不限 source 的全量 ORDER（含未归因 `-`）。
- 渠道成交合计（Push+资源位+短信+美妆pop）与大盘对比能看出归因链路是否打通（渠道 ≪ 大盘 → 归因未接上）；备注可标注口径异常（如"渠道归因成交为 0，大盘全落未归因"）。

### 目标 1 日度/区间模式（起投日 → 目标日，每日明细 + 总计）

用户要"从商品上线之后每天的数据和总计"时，按各商品自己的起投日 `start_ds` 起算、到目标日，输出每日汇总 + 区间总计。要点：

1. **先取各商品起投日**（`scope_start.sql`）：gift_item_pool 里 `TO_CHAR(FROM_UNIXTIME(CAST(startTime AS BIGINT)/1000),'yyyymmdd') AS start_ds`，作为三张表的裁剪边界。
2. **事件流水表必须按起投日裁剪**（⚠️ 最易错）：事件流水表记录商品**全量**成交（含起投前的自然单），若不卡 `dt >= start_ds`，大盘会被起投前自然成交严重灌水（实测卡姿兰 8/2、毛戈平/娇韵诗/海蓝之谜等起投前都有大量自然单）。做法：JOIN 在投清单取 `start_ds`，`WHERE e.dt >= c.start_ds AND e.dt <= '${end_dt}'`。
3. **一条 SQL 同时出四套口径**：`GROUP BY itemid, GROUPING SETS ((dt,source),(dt),(source),())` → 每日×分渠道、每日×大盘、区间×分渠道、区间×大盘；用 `COALESCE(dt,'TOTAL')`、`COALESCE(source,'ALL')` 标记汇总行。
4. **Banner** 用模式 B（`ds >= c.start_ds AND ds <= '${end_date}'`），同样 `GROUPING SETS ((t.ds),())` 出每日+区间。
5. **Push** 明细同样加 `GROUP BY item_id, GROUPING SETS ((ds),())`（实测爆品任务无起投前数据，可不裁剪）。
6. **总计行口径**：为各商品「起投日→目标日」**区间去重** UV 的加总（跨商品未去重），≠ 每日行求和；区间 uCTR/uCVR 与单日口径不同，报告须注明不可直接对比。
7. **每日行按起投日裁剪**：商品未起投的日期在附录宽表记「·」；每日汇总只加总当天已在投的商品。
8. 输出结构：① 每日汇总表（日期 × 各渠道曝光/点击/成交/渗透率）② **品牌维度**：品牌区间漏斗表（商品数/曝光/点击/uCTR/成交/uCVR/Push/大盘/渗透率）+ 品牌×每日宽表（资源位曝光UV、资源位点击UV、资源位uCVR 各一张）③ 商品区间汇总表（沿用目标 1 二十一列布局 + 起投日列）④ 附录分商品每日宽表（资源位曝光/渠道成交/大盘成交/Push送达 各一张，行=商品，列=日期，末列=区间总计）。品牌 UV 为品牌内商品加总（跨商品未去重）；品牌尚未起投的日期记「·」；小样本 Push uCTR >100% 注明为埋点口径差异。
9. 参考实现：`outputs/daily_channel_clip.sql`（裁剪+GROUPING SETS）、`outputs/daily_banner.sql`（模式B）、`outputs/push_daily.py`、`outputs/build_daily.py`（合并）、`outputs/make_report_daily.py`（成报）。
10. **新起投商品/新品牌识别**：每次跑数先用目标日（或最近可用 ds）的 gift_item_pool 全量清单与已有品牌映射对比，`startTime` 晚于上次窗口的是新入场商品；品牌归属从 `attribute` JSON 的 `itemTitle` / `benefitItemTitle` / 店铺判断（如 8/10 新增：兰蔻×4 = 1060755092498/1060753340704/1064360618638/1058038587039，洁婷×1 = 40601814277，为首个个护品牌）。若曝光表查询出现映射外的 business_id，先查 pool attribute 补齐品牌映射再出品牌汇总；当日 ds 分区未产出时回退最近可用 ds。

---

## 目标 2：Push 任务强意图成交分析

### 数据来源

- **Push 明细底表**：`tbcdm.dwd_tb_zyy_log_msg_push_di`（分区 `ds`，二级渠道 `channel_level2_name`，如 `活动优惠`）。`tag` 字段是**多层转义 JSON**（`bizUtParam` / `tppTrackInfo` 皆为转义字符串），需解析出 item_id / mp_task_id。
- **强意图口径**：`intent_trigger`（强意图触发人群，push 前置环节）。品牌级触发量可从 `ads_fmcg_brand_channel_report_di2.intent_trigger_uv` 取；实时事件可下钻 `wlods.rt_intent_event_tt_send_log`（按 intent_code/hit_type）。

### ⚠️ 解析 push 明细表必须用 hints 同批提交（不能用 -e/-f）

解析 `dwd_tb_zyy_log_msg_push_di` 的嵌套转义 tag，必须启用三个 SET：

```sql
set odps.sql.udf.getjsonobj.new = true;   -- 新版 UDF，正确解析嵌套转义 JSON
set odps.sql.type.system.odps2  = true
set odps.sql.mapper.split.size  = 4096;
```

**这三个 SET 必须与 SELECT 在同一个 ODPS instance 内提交才生效**。而 `pyodps.py` CLI 按 `;` 拆分逐条独立提交，SET 会丢失 → 解析全 NULL → 过滤后 0 行（假空）。所以**必须写 python 脚本用 `odps.execute_sql(sql, hints={...})` 同批提交**，不能用 `-e`/`-f`。

**验证证据**：娇兰复原蜜 566446518116，ds=20260802，活动优惠，按 mp_task_id=47939001 过滤 → hints 同批提交得 send 1,287PV/1,190UV、arrive 501PV/486UV、click 20PV/20UV（与品牌 ADS 报表 1190/486/20 完全一致）；SET 未同批（分开提交）→ 解析全 NULL，过滤后 0 行（假空）。

### hints 脚本模板

复制到当前工作目录（如 `./push_intent.py`），改 SQL 后 `uv run ./push_intent.py`：

```python
# /// script
# requires-python = ">=3.9"
# dependencies = [
#   "pyodps",
#   "alibabacloud-sts20150401>=1,<2"
# ]
# ///
import sys
sys.path.insert(0, "/Users/mark/.agents/skills/odps-skill/scripts")
import pyodps

config = pyodps.read_config()
odps = pyodps.get_odps_client(config)
project = config.get("project")

sql = """
SELECT  item_id,
        COUNT(*)                AS rows_cnt,
        COUNT(DISTINCT user_id) AS send_uv,
        COUNT(send_time)        AS send_cnt,
        COUNT(arrive_time)      AS arrive_cnt,
        COUNT(DISTINCT CASE WHEN arrive_time IS NOT NULL THEN user_id END) AS arrive_uv,
        COUNT(click_time)       AS click_cnt,
        COUNT(DISTINCT CASE WHEN click_time IS NOT NULL THEN user_id END)  AS click_uv
FROM (
    SELECT  utdid, user_id, send_time, arrive_time, click_time,
    CASE
    WHEN GET_JSON_OBJECT(GET_JSON_OBJECT(GET_JSON_OBJECT(tag,'$.bizUtParam'),'$.tppTrackInfo'),'$.itemType')='1'
    AND GET_JSON_OBJECT(GET_JSON_OBJECT(GET_JSON_OBJECT(tag,'$.bizUtParam'),'$.tppTrackInfo'),'$.id') IS NOT NULL
    THEN GET_JSON_OBJECT(GET_JSON_OBJECT(GET_JSON_OBJECT(tag,'$.bizUtParam'),'$.tppTrackInfo'),'$.id')
    WHEN GET_JSON_OBJECT(GET_JSON_OBJECT(GET_JSON_OBJECT(tag,'$.bizUtParam'),'$.tppTrackInfo'),'$.eventId') IS NOT NULL
    THEN GET_JSON_OBJECT(GET_JSON_OBJECT(GET_JSON_OBJECT(tag,'$.bizUtParam'),'$.tppTrackInfo'),'$.eventId')
    WHEN GET_JSON_OBJECT(GET_JSON_OBJECT(tag,'$.bizUtParam'),'$.itemId') IS NOT NULL
    THEN GET_JSON_OBJECT(GET_JSON_OBJECT(tag,'$.bizUtParam'),'$.itemId')
    ELSE NULL
    END AS item_id,
    CASE
    WHEN GET_JSON_OBJECT(GET_JSON_OBJECT(tag,'$.bizUtParam'),'$.mpTaskId') IS NOT NULL
    THEN GET_JSON_OBJECT(GET_JSON_OBJECT(tag,'$.bizUtParam'),'$.mpTaskId')
    WHEN tag NOT LIKE '%marketing.agoo%'
    AND GET_JSON_OBJECT(GET_JSON_OBJECT(tag,'$.bizUtParam'),'$.mpTaskId') IS NULL
    THEN SPLIT_PART(task_id,'-',4)
    WHEN tag LIKE '%marketing.agoo%' THEN task_id
    END AS mp_task_id
    FROM    tbcdm.dwd_tb_zyy_log_msg_push_di
    WHERE   ds='${bizdate}'
    AND   channel_level2_name IN ('活动优惠')
) t
WHERE mp_task_id='${push_mp_task_id}'      -- 按爆品 push 任务过滤（如 47939001）
  AND item_id IN (${in_scope_item_ids})
GROUP BY item_id
"""

hints = {
    "odps.sql.udf.getjsonobj.new": "true",
    "odps.sql.type.system.odps2": "true",
    "odps.sql.mapper.split.size": "4096",
}

inst = odps.execute_sql(sql, project=project, hints=hints)
print("INSTANCE_ID:", inst.id)
inst.wait_for_success()
with inst.open_reader() as reader:
    header = [c.name for c in reader.schema.columns] if getattr(reader, "schema", None) else None
    print("HEADER:", header)
    n = 0
    for rec in reader:
        print("ROW:", list(rec.values))
        n += 1
    print("ROW_COUNT:", n)
```

### item_id / mp_task_id 解析逻辑（放入上面 SQL）

item_id 解析优先级（tppTrackInfo.id > tppTrackInfo.eventId > bizUtParam.itemId）已在模板中。

**mp_task_id 是解析字段，不是表列**：它从 `tag.bizUtParam.mpTaskId` 取，取不到时按是否 agoo 兜底（非 agoo 取 `SPLIT_PART(task_id,'-',4)`，agoo 直接取 `task_id`）。表里的 `task_id` / `ucp_task_id` 直接列**不能**当爆品任务 ID 用。

**分析指定 push 任务时必须用 mp_task_id 过滤**（如爆品任务 `47939001`），否则会混入其它任务的 push 量。实测：`mp_task_id='47939001'` + ds=20260802 只命中娇兰 566446518116（send 1190UV/arrive 486UV/click 20UV），与品牌 ADS 报表 `push_send_uv/push_arrive_uv/push_click_uv`(1190/486/20) 完全一致——卡姿兰的 push 来自其它任务，不属于 47939001。

```sql
CASE
  WHEN GET_JSON_OBJECT(GET_JSON_OBJECT(tag,'$.bizUtParam'),'$.mpTaskId') IS NOT NULL
    THEN GET_JSON_OBJECT(GET_JSON_OBJECT(tag,'$.bizUtParam'),'$.mpTaskId')
  WHEN tag NOT LIKE '%marketing.agoo%'
       AND GET_JSON_OBJECT(GET_JSON_OBJECT(tag,'$.bizUtParam'),'$.mpTaskId') IS NULL
    THEN SPLIT_PART(task_id,'-',4)
  WHEN tag LIKE '%marketing.agoo%' THEN task_id
END AS mp_task_id
```

### 强意图成交链路

将 push 明细解析出的 item×任务的 send/arrive/click，与步骤 2 事件流水表里 `source='push'` 的 ORDER 关联，构成强意图 push 全链路：

```
强意图触发(intent_trigger) → push 发送(send) → 送达(arrive) → 点击(click) → 成交(order)
```

- 触发→发送→送达→点击：来自 push 明细表（hints 解析）。
- 点击→成交：来自事件流水表 `source='push'` 的 CLICK/ORDER（步骤 2）。
- 品牌级强意图触发量可用 `ads_fmcg_brand_channel_report_di2.intent_trigger_uv` 交叉核对。

### 目标 2 输出格式

**发送/送达/点击用 PV；成交统一用 UV**，并补上「渠道成交(合计)」与「大盘成交」两列：

```
| 商品 | Push发送PV | Push送达PV | 送达率 | Push点击PV | PCTR | Push成交UV | 渠道成交UV(合计) | 大盘成交UV |
```

- Push成交UV = 事件流水表 `source='push'` 的 ORDER（去重）；渠道成交(合计) = push+资源位+短信+美妆pop归因之和；大盘成交 = 不限 source。
- 送达率 = 送达/发送；PCTR = 点击/送达（均 PV）。

**⚠️ 本漏斗 Push 点击 ≠ 目标 1 渠道表的 Push 点击（口径本就不同）**：目标 1 渠道表的 Push 点击是**事件流水表** `source='push'`、**UV**、**不限任务**；本漏斗 Push 点击是 **push 明细表** `click_time`、**PV**、**只限指定 mp_task_id**。差异来自三点：来源表/埋点不同、UV vs PV、任务范围不同——属预期，报告须注明，别当成数据错误。

关注：送达率异常（系统/时段）、PCTR 高但成交低（文案与落地不一致）、强意图触发体量极小（模型未覆盖该品牌人群）。

---

## 目标 3：大盘成交买家漏斗分析（渠道归因桑基图）

### 分析目标

以大盘成交用户为起点，追踪买家在意图漏斗中的覆盖情况：

```
大盘成交UV → 切端触发UV → 强意图识别UV → 强意图未购UV → Push触发UV
```

核心问题：大盘成交买家中，有多少人经过了意图识别链路？有多少人被 Push 触达？未覆盖的用户走了什么转化路径？

### 数据源

- **大盘成交用户**：`amp_im.s_tt_gift_benefit_record_tt4_parsed`，`action='ORDER'`，分区 `dt`
- **意图事件**：`wlods.rt_intent_event_tt_send_log`，分区 `ds`，778M+ 行大表
- **品牌商品字典**：`brand_map` VALUES CTE（见步骤 1 的 item_id → brand 映射）

### 关键字段说明（wlods.rt_intent_event_tt_send_log）

| 字段 | 说明 |
|------|------|
| `user_id` | 用户 ID |
| `intent_code` | 意图类型，过滤 `'strongPurchase'` 和 `'comparePrices_v1'` |
| `tppid` | TPP 场景 ID，`'32404'` 表示强意图场景 |
| `item_list` | 商品列表，JSON 数组字符串，需 `REGEXP_REPLACE` + `EXPLODE` 解析 |
| `p8_items` | 已购商品列表（JSON），用于判断"强意图但未购" |
| `result` | 执行结果，`'succ'` 表示 Push 成功触发 |

### ⚡ SQL 优化：跨项目 JOIN 必用三板斧

`amp_im`（买家表）和 `wlods`（意图表 778M 行）的跨项目 JOIN **必须**用以下三个优化，否则 15+ 分钟超时：

#### 优化版 SQL 模板（实测 7 品牌 ~10s 完成）

```sql
WITH brand_map AS (
    SELECT  item_id, brand
    FROM    VALUES
            ('566446518116', '娇兰'), ('651588553248', '卡姿兰'), ('638131550168', '可丽金'),
            ('43984164814', '海蓝之谜'), ('649652424111', '海蓝之谜'), ('650605693497', '海蓝之谜'), ('805251110686', '海蓝之谜'),
            ('14021638771', '毛戈平'), ('537219932625', '毛戈平'), ('679396365892', '毛戈平'), ('740202067434', '毛戈平'), ('781646174250', '毛戈平'),
            ('558744903450', '娇韵诗'), ('649916182962', '娇韵诗'), ('707440056381', '娇韵诗'),
            ('1058038587039', '兰蔻'), ('1060753340704', '兰蔻'), ('1060755092498', '兰蔻'), ('1064360618638', '兰蔻')
            AS t(item_id, brand)
),
market_buyers AS (
    SELECT  /*+ MAPJOIN(bm) */
            DISTINCT r.user_id, bm.brand
    FROM    amp_im.s_tt_gift_benefit_record_tt4_parsed r
    JOIN    brand_map bm
    ON      r.itemid = bm.item_id
    WHERE   r.dt = '${bizdate}'
    AND     r.action = 'ORDER'
),
user_intent AS (
    SELECT  /*+ MAPJOIN(bm) */
            e.user_id,
            MAX(CASE WHEN e.tppid = '32404' THEN 1 ELSE 0 END) AS strong_flag,
            MAX(CASE WHEN e.tppid = '32404'
                      AND (e.p8_items IS NULL OR INSTR(e.p8_items, CONCAT('"', TRIM(t.item_raw), '"')) = 0)
                     THEN 1 ELSE 0 END) AS not_purchased_flag,
            MAX(CASE WHEN e.tppid = '32404'
                      AND (e.p8_items IS NULL OR INSTR(e.p8_items, CONCAT('"', TRIM(t.item_raw), '"')) = 0)
                      AND e.result = 'succ'
                     THEN 1 ELSE 0 END) AS push_flag
    FROM    wlods.rt_intent_event_tt_send_log e
    LATERAL VIEW EXPLODE(SPLIT(REGEXP_REPLACE(e.item_list, '\[|\]|"', ''), ',')) t AS item_raw
    JOIN    brand_map bm
    ON      TRIM(t.item_raw) = bm.item_id
    WHERE   e.ds = '${bizdate}'
    AND     e.intent_code IN ('strongPurchase', 'comparePrices_v1')
    GROUP BY e.user_id
)
SELECT  b.brand,
        COUNT(u.user_id)          AS intent_trigger_uv,
        SUM(u.strong_flag)        AS strong_intent_uv,
        SUM(u.not_purchased_flag) AS not_purchased_uv,
        SUM(u.push_flag)          AS push_triggered_uv
FROM    market_buyers b
JOIN    user_intent u
ON      b.user_id = u.user_id
GROUP BY b.brand
ORDER BY intent_trigger_uv DESC;
```

#### 为什么这三个优化缺一不可

| 优化 | 作用 | 不做的后果 |
|------|------|-----------|
| VALUES CTE | 品牌字典单源维护，JOIN 替代 IN | SQL 膨胀到 47KB+，HTTP 超时 |
| MAPJOIN | 小表广播，免 shuffle | 跨项目 shuffle 巨慢 |
| 预聚合 (MAX CASE) | 事件级→用户级，shuffle 数据量降 10~100x | 778M 事件行全量 shuffle → 15+ min |

### 漏斗口径说明

| 漏斗层 | 含义 | SQL 对应 |
|--------|------|---------|
| 大盘成交UV | 当天购买该品牌商品的用户数 | `market_buyers` COUNT |
| 切端触发UV | 买家中在意图表有事件记录的用户 | `COUNT(u.user_id)` |
| 强意图识别UV | 买家中 tppid='32404'（强意图场景）的用户 | `SUM(strong_flag)` |
| 强意图未购UV | 强意图但未在 p8_items 中已购的用户 | `SUM(not_purchased_flag)` |
| Push触发UV | 未购用户中 result='succ'（Push 成功）的用户 | `SUM(push_flag)` |

> **JOIN 口径 vs IN 子句口径**：JOIN 捕获所有购买了品牌商品的用户的意图行为（即使意图触发在不同商品上），IN 子句只计算意图匹配同品牌商品的用户。JOIN 更准确，两者数值会有小幅差异。

### 全量意图漏斗（参考基线）

除买家漏斗外，还需跑出全量意图漏斗作为对比基线（不限大盘成交用户）。数据来源为 `amp_im.ads_fmcg_brand_channel_report_di2` 或 `wlods.rt_intent_event_tt_send_log` 全量聚合。

### ECharts Sankey 可视化模板

输出为自包含 HTML 文件（CDN 加载 ECharts 5.5.0），支持多品牌切换：

1. **顶部卡片**：品牌名 + 大盘成交UV + 覆盖率，点击切换
2. **Tab 栏**：品牌快速切换
3. **每个品牌面板**：
   - 4 个核心指标卡片（大盘UV / 覆盖率 / 未购比例 / Push触发率）
   - 左侧：大盘成交买家漏斗 Sankey（大盘成交UV → 切端触发 → 强意图 → 未购/Push）
   - 右侧：全量意图漏斗 Sankey（参考基线）
4. **7品牌全量对比表**：买家漏斗 + 全量漏斗 + 占比，高亮最优/最差值
5. **洞察区**：P0/P1/P2 级洞察，根据数据动态生成

**Sankey 节点配色规范**：

| 节点 | 颜色 |
|------|------|
| 大盘成交UV | 品牌色（动态） |
| 切端触发 | #5b8ff9 |
| 强意图 | #5ad8a6 |
| 未购 | #f6bd16 |
| Push触发 | #6dc8ec |
| 已购用户 | #e8684a |
| 未在切端/未Push | #ccc/#d9d9d9 |

参考实现：`outputs/sankey_market_funnel.html`。

### 实测数据（ds=dt=20260818，7品牌）

| 品牌 | 大盘成交UV | 切端触发UV | 强意图UV | 未购UV | Push触发UV | 覆盖率 |
|------|-----------|-----------|---------|--------|-----------|--------|
| 毛戈平 | 3485 | 1412 | 1412 | 953 | 305 | 40.5% |
| 卡姿兰 | 1978 | 506 | 506 | 324 | 66 | 25.6% |
| 可丽金 | 1455 | 741 | 741 | 566 | 95 | 51.0% |
| 娇韵诗 | 1295 | 640 | 640 | 495 | 149 | 49.4% |
| 海蓝之谜 | 1025 | 442 | 442 | 319 | 105 | 43.1% |
| 娇兰 | 390 | 130 | 130 | 91 | 22 | 33.3% |
| 兰蔻 | 178 | 74 | 74 | 52 | 10 | 41.6% |

### 典型洞察（自动生成参考）

- **P0：买家切端覆盖率差异巨大**（25.6%~51.0%）：大量成交用户未在意图漏斗中被检测到
- **P0：切端→强意图转化率全品牌100%**：瓶颈在"能否被切端捕获"，不在模型
- **P1：买家Push触发率低于全量（~48%）**：已购用户push触达效率反而更低
- **P2：买家在全量漏斗中占比极低**（0.06%~5.37%）：Push主要对象是"有意图但未购"的潜在用户

---

## 报告输出结构

1. **数据窗口与口径说明**（目标日、实际用到的分区、在投清单来源）
2. **在投商家 × 商品清单**（步骤 1 结果，标注起投时间）
3. **各渠道曝光/点击/成交汇总表**（目标 1）
4. **Push 强意图成交链路表**（目标 2）
5. **大盘成交买家漏斗 Sankey 可视化**（目标 3，HTML 交付）
6. **关键洞察与优化建议**（按 P0/P1/P2）
7. **查询 SQL / 脚本附录**（便于复用）

末尾必须用 present 工具把输出文件交付给用户。

---

## 注意事项

1. **执行通道**：一律走 odps-skill 的 `uv run .../pyodps.py`；不再用 MCP。普通查询 `-e`/`-f`，push 明细解析走 hints python 脚本。
2. **在投清单为准**：分析口径以 `gift_item_pool` 生效清单为准，不要直接套固定品牌白名单——实际在投商家常与历史白名单不一致。
3. **时间边界**：卡"截至日 D"用 D+1 天 00:00 的毫秒常量做上界，切勿多算一天。
4. **分区字段易错**：事件流水表是 `dt`，gift_item_pool / push 明细 / ADS 表是 `ds`；不要混用。
5. **source='-' 排除**：渠道点击/成交只取 push/banner/default/hotpage。
6. **push 解析 SET 必须 hints 同批**：`-e`/`-f` 拼 `set; select;` 会假空。
7. **分析指定 push 任务须按 mp_task_id 过滤**：mp_task_id 是从 `tag.bizUtParam.mpTaskId` 解析的字段（非表列，不能直接用 `task_id`/`ucp_task_id` 列），按爆品任务 ID（如 47939001）过滤，否则混入其它任务的 push 量。
8. **banner 商品级曝光**：首选 `tmp_ads_tb_udec_msg_banner_expos_activity_hh`（task_id 固定 '601'、business_id=itemId、arg1 区分曝光/点击、UV=long_login_user_id、全天 hh<='23'）；列级 LABEL 权限已于 2026-08-10 开通，正常可读；若偶发无权限则退回品牌级 ADS `banner_imp_uv`。
9. **空保护**：转化率分母用 `NULLIF(x,0)`。
10. **PV/UV 按来源表定**：事件流水表的点击、成交一律用 **UV**；push 明细表的发送/送达/点击用 **PV**。因此渠道表的 Push 点击(UV、不限任务) 与漏斗的 Push 点击(PV、限 task) 天然不同，报告须注明口径，别当成 bug。
11. **新商家接入**：在 `merchant_id → brand` CASE 映射里补一行。
12. **banner 曝光时间范围与容错**：支持两种模式——模式 A 固定一天（ds='${bizdate}'），模式 B 商品上线日→目标日（ds ∈ [start_ds, '${end_date}']，按各商品起投日动态起算；跨天 UV 为区间去重）。无权限（NoPermission）时该表各列按空处理并注明"数据暂为空"，不阻塞其余查询。
13. **跨项目 JOIN 必须用优化三板斧**（目标 3）：VALUES CTE + MAPJOIN + 预聚合，否则 778M 行意图表 JOIN 会 15+ 分钟超时。实测优化后 7 品牌 ~10s 完成。
14. **PyODPS read_timeout 必须设短（30s）**：长查询用异步提交+轮询，read_timeout 设 600s 会导致 `instance.reload()` 每次阻塞 600s，轮询失效。
15. **item_list 解析**：`wlods.rt_intent_event_tt_send_log` 的 `item_list` 是 JSON 数组字符串（如 `["123","456"]`），需 `REGEXP_REPLACE` 去括号和引号后 `SPLIT` + `EXPLODE`，每个 item 要 `TRIM`。
16. **p8_items 判断未购**：`INSTR(e.p8_items, CONCAT('"', TRIM(item_raw), '"')) = 0` 判断商品不在已购列表中。p8_items 为 NULL 也视为未购。
17. **美妆pop（hotpage）渠道**：`source='hotpage'`，美妆行业 pop 弹窗，无曝光数据（与短信类似），仅出点击/成交/uCVR。列布局放在短信之后、渠道成交合计之前。

---

## 示例触发语

- "统计在投商家商品的各渠道点击和成交"
- "娇兰复原蜜 banner / push / sms 数据"
- "跑一下 push 和 banner 的成交数据"
- "在投商品四渠道（banner/push/短信/美妆pop）拆分"
- "美妆pop hotpage 渠道成交数据"
- "push 强意图成交分析"
- "截至 8/3 在投商品的渠道曝光点击成交"
- "商家商品渠道效果拆分"
- "大盘成交买家漏斗桑基图"
- "品牌覆盖率分析"
- "大盘成交用户的意图覆盖情况"
- "各品牌买家 push 触达率"
