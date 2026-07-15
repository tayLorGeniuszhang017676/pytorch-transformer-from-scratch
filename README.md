# step 1: Amazon 竞品监控（Keepa API + Streamlit 看板）

监控 DE / US 两站共 6 个竞品 y ASIN 的价格、Buy Box、BSR 排名、评分与评论、Coupon、跟卖数和断货状态。架构：`keepa_fetch.py` 定时拉取 Keepa 数据写入 SQLite → 与上一轮快照 diff，命中阈值推送飞书/钉钉/Slack → `dashboard.py` 提供 Tableau 风格的交互式看板。

```
config.yaml        ASIN 清单、阈值、告警通道（改这里即可）
keepa_fetch.py     抓取 + 入库 + diff 告警（定时跑这个）
dashboard.py       Streamlit 看板（streamlit run dashboard.py）
alerts.py          告警规则与 webhook 推送
db.py              SQLite 存储层（monitor.db）
demo_seed.py       生成 90 天演示数据，无需 API key 先看效果
.github/workflows/monitor.yml   GitHub Actions 每小时定时任务
```

## Step 2:ch快速开始（先看效果，不需要 Keepa key）

```bash
pip install -r requirements.txt
python demo_seed.py
streamlit run dashboard.py
```

看板功能：站点 / ASIN / 时间范围筛选，价格与 BSR 趋势（同色系跨图表一致，DE 冷色、US 暖色），KPI 卡片带 7 天涨跌幅，最新快照表和告警流水。

## 接入真实数据

1. 在 keepa.com 注册并订阅 API，拿到 access key（token 按订阅档位每分钟发放；本配置 6 个 ASIN 含 Buy Box 历史，每轮消耗约十几个 token，小时级轮询占用很小，具体配额见 keepa.com/#!api）。
2. 删除演示库并抓取：

```bash
rm -f monitor.db
export KEEPA_API_KEY=你的key
python keepa_fetch.py     # 首轮会回填约 90 天历史，之后每轮增量更新
streamlit run dashboard.py
```

## 告警配置

在 `config.yaml` 的 `alerts` 里选通道（`feishu` / `dingtalk` / `slack` / `console`）并填机器人 webhook；也可以用环境变量 `ALERT_WEBHOOK_URL` 传入，避免地址进仓库。钉钉机器人需在安全设置中添加自定义关键词「竞品监控」。默认阈值：价格变动 ≥5%、BSR 变动 ≥30%、评分下降 ≥0.1、单轮新增评论 ≥5 条，另有 Coupon 上下线、疑似断货/恢复、Listing 标题变更三类事件告警，均可在 `thresholds` 中调整。

## 定时运行

服务器上用 cron：

```
17 * * * * cd /path/to/amazon-competitor-monitor && KEEPA_API_KEY=xxx python keepa_fetch.py >> monitor.log 2>&1
```

或者直接推到 GitHub 私有仓库，用自带的 Actions workflow：在仓库 Settings → Secrets 添加 `KEEPA_API_KEY` 和（可选）`ALERT_WEBHOOK_URL`，workflow 每小时运行一次并把 `monitor.db` 提交回仓库，看板在任何机器上 clone 下来就能打开。

## 加入我方 ASIN 对比

在 `config.yaml` 的 `own_asins` 里填入你们自己的 ASIN，会一并抓取，看板和告警中会标记「★我方」，方便直接对比定价和排名走势。

## 指标口径与已知局限

Keepa 的数据更新频率随商品热度浮动，热销品接近小时级，冷门品可能数小时更新一次，所以"实时"实际是小时级轮询，这对竞品监控足够。断货判断基于在售 offer 数，是近似口径。评分与评论只能拿到总量曲线，拿不到单条差评内容——「评分下降 + 评论激增」的组合告警就是差评流入的信号，收到后去 listing 页人工确认。关键词自然排名和广告位不在 Keepa 数据范围内，需要卖家精灵 / Helium 10 补充，可作为下一步扩展；此外主图和五点变更检测、汇率归一对比也是容易加的方向。
