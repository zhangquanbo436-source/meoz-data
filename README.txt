MeoZ 沪深主板 A 股历史数据
请求范围：2023-01-01 至 2026-10-26（北京时间，已收盘）；首个交易日 2023-01-03。
范围：沪深主板 A 股，代码前缀 000/001/002/003/600/601/603/605；含 ST 和已退市股票。
不含 B 股、创业板、科创板、北交所。股票列表不按当前存续状态排除已退市股。
股票列表 3486 只：上市 3194，退市 292。
个股日线合计 2,927,642 行，覆盖 921 个交易日。

重要缺口
1. sw_l1（申万一级行业）缺失 3486 只。已尝试申万官网分类页面及官方 XLS 下载，连接失败，故留空。
   industry_provider 是猫爪原始行业，不是申万行业，不能当成 sw_l1。
   申万官方来源：https://www.swsresearch.com/institute_sw/allIndex/downloadCenter/industryType
   XLS：https://www.swsresearch.com/swindex/pdf/SwClass2021/StockClassifyUse_stock.xls
2. 中证全指 000985.SH 实际 848 行，沪深300 000300.SH 实际 921 行。
   共 73 个指数/日期缺口见 index_missing_dates.parquet。缺失未填充或插值。
   已分别按带后缀区间、单日重新查询中证全指近期缺口，API仍返回无数据。
3. 每字段空值与源数据异常见 quality_report.json 和 daily_coverage.parquet。未知状态保留 null。

文件组织
stock_daily_YYYY[_partNN][_subNN].parquet：按年拆分，所有实际压缩文件不超过20,000,000字节。
受上传通道300秒超时影响，个别未完成分片进一步拆为较小的sub分片；内容和总行数不变。
每个 Parquet 独立可读。小表和每年尾部文件自然小于 20MB，不人为填充。
stock_list.parquet：每只股票一行。退市日期来自沪深交易所；未退市为空。
index_daily_000985.SH.parquet、index_daily_000300.SH.parquet：指数 OHLC。
daily_coverage.parquet：逐日原始与输出行数、空值计数、ST来源日期覆盖。
stock_coverage.parquet：逐股票逐年的记录数、首末日期。
manifest.json：每文件字节数、行数（Parquet）、SHA256 校验值。

个股日线字段及单位
code：字符串，带 .SH/.SZ 后缀；date：YYYY-MM-DD 字符串。
open/high/low/close：元/股，不复权，使用 stk_factor_pro 的同名未复权字段，不保存 qfq/hfq 价格。
pre_close：元/股，保留猫爪“可比昨收/涨跌幅基准价”原值；除权日不保证等于上一日原始 close。
adj_factor：无量纲复权因子，原始值。前复权通常为原价×当日因子/锚定日因子，后复权为原价×当日因子。
volume：手，1手=100股；来自 vol。若要股数，乘100。
amount：人民币元；来自 amount。
turnover：百分数，1表示1%；来自 turnover_rate。
float_mv：人民币元；来自 circ_mv（非万元）。
up_limit/down_limit：元/股，来自 pricelimit；按人民币分规范为2位小数，消除API的float32浮点尾差；无数据保留 null。
is_st：可空布尔值，逐日对照 st 接口标记；当日ST源有覆盖时，命中True、未命中False；整日无覆盖则null。
因此无需另存推算的ST起止表。原始历史名称不用于判断 ST，避免当前名称污染历史状态。

股票列表字段
code/name/list_date/delist_date/sw_l1/industry_provider/list_status/list_date_source/delist_date_source。
list_status：L上市、D退市。sw_l1为要求的申万一级行业列，目前因源不可用留空。
industry_provider仅供原始信息参考，非申万分类，也不是历史时点行业。
delist_date为摘牌/终止上市生效日，不以最后一个有行情的日期冒充；292只退市A股全部有交易所日期。
列表保留2023年前已退市证券，因此部分股票在本数据时间范围无行情是正常的。

清洗和完整性规则
仅保留有正成交量、正成交额和正OHLC的交易记录，不生成停牌日。
删除上市日前、退市生效日起的越界记录；明细数量见quality_report.json。
逐交易日请求，避免猫爪未明确提示的6000行上限；ST按月获取，达到6000行时递归拆日期。
检查 code/date 唯一性、主表关联、OHLC范围、文件可读性及实际字节数。
源数据异常不会擅自纠正，异常计数为 0。

数据来源及获取方式
猫爪真实API，经用户本机 meoz Skill 脚本 execute_requests 调用，API Key未写入文件。
交易日历：https://meoz.cn/docs/query/calendar/start-to-latest
日线及复权因子/换手率/流通市值：https://meoz.cn/docs/query/stock/stk-factor-pro
股票列表：https://meoz.cn/docs/query/stock/basic （分别请求L和D，P返回未找到记录）
ST：https://meoz.cn/docs/query/stock/st
涨跌停价：https://meoz.cn/docs/query/stock/pricelimit
指数：https://meoz.cn/docs/query/index/indexdaily
这些接口的文档服务为live；其日期区间支持历史日线，未擅自切换历史服务同名接口。
上海交易所退市资料：https://www.sse.com.cn/assortment/stock/list/delisting/
深圳交易所退市资料：https://www.szse.cn/market/stock/suspend/index.html
官方退市名单按分页总记录数核对，A/B共用公司代码时筛出A股记录；退市股上市日期使用交易所A股日期。
猫爪与交易所的上市日期差异记录在official_date_corrections.parquet。
文档目录版本1.0.51，所用接口文档版本见quality_report.json。
