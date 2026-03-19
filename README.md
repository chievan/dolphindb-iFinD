# iFinD (同花顺 HTTP API) DolphinDB 模块使用说明

## 1. 模块介绍

`iFinD` 模块是对同花顺 iFinD HTTP API 的深度封装，旨在为 DolphinDB 用户提供原生、高速、稳定的金融数据接入方案。通过该模块，用户可以在 DolphinDB 脚本中直接获取 A 股、港股、美股、基金、债券、指数等全品种行情及各种财务、估值指标，并自动解析为结构化的 `Table` 对象。

**核心优势**:

- **强类型校验**: 针对日期格式、参数类型进行全量拦截，避免无效 API 请求。
- **自动 Token 转换**: 支持长效 `refreshToken` 自动置换为临时 `accessToken`。
- **高性能解析**: 针对 API 返回的 JSON 结构进行深度优化，直接转换为 DolphinDB 表。
- **安全拦截**: 实时监测 `dataVol` (数据消耗量)，在无数据或权限不足时提供清晰反馈。

---

## 2. 快速开始

### 2.1 依赖安装与加载

本模块依赖 DolphinDB 官方的 `httpClient` 网络插件。

注意：在使用本模块前请按照 [httpClient 插件说明](https://docs.dolphindb.cn/zh/plugins/httpClient/httpclient.html)进行安装。例如可执行下列命令安装、加载，或在节点配置文件中配置  `preloadModules=plugins::httpClient`：
```
installPlugin("httpClient");
loadPlugin("httpClient");
```

### 2.2 安装模块

#### 版本要求
- 支持 DolphinDB Server 2.00.17 及以上版本。
- 支持 Linux x64，Linux ARM ，Linux ABI，Windows x64。

#### 安装步骤

1. 在 DolphinDB 客户端中使用 `listRemoteModules` 函数查询可用的模块。

   ```
   login("admin", "123456")
   listRemoteModules()
   ```

2. 使用 `installModule` 函数安装模块。

   ```
   installModule("iFinD")
   ```

3. 使用 use 关键字加载模块。

   ```
   use iFinD
   ```
### 2.3 凭证获取

要使用 iFinD 数据服务，您需要首先在同花顺 iFinD 终端获取长效凭证，并通过模块置换为访问令牌：

1. **获取 Refresh Token**：

   - 登录 **同花顺 超级命令SuperCommand**。
   - 在工具中点击“`refresh_token`查询”，然后复制即可（通常是以 `eyJ` 开头的长字符串）。
2. **获取 Access Token**：

   - 使用模块提供的 `getAccessToken` 函数，将长效 `refreshToken` 转换为有效期内的 `accessToken`。后续的所有数据查询接口都建议使用 `accessToken` 以获得最佳安全性。

```dolphindb
// 设置您从终端获取的长效凭证
refreshToken = "eyJ..." 

// 调用接口获取临时访问令牌
accToken = iFinD::getAccessToken(refreshToken)
```

---

## 3. 基础 API

### thsBasicData

获取证券的基础快照、财务快照或特定日期的指标值。

**语法**

```dolphindb
iFinD::thsBasicData(token, code, indicator, [indicatorParam])
```

**详情**

该函数用于获取非时间序列的基础数据，如公司名称、上市日期，或某一特定日期的财务指标快照。支持批量证券查询。

**参数**

**token** STRING 类型标量，支持 refreshToken 或 accessToken。

**code** STRING 类型标量，证券代码，多个代码用半角逗号分隔。如 "600519.SH,000001.SZ"。

**indicator** STRING 类型标量，指标英文名，逗号分隔。

**indicatorParam** STRING 类型标量，可选。指标参数，不同指标用分号分隔。

**返回值**

返回一个 Table。包含证券代码列 ths_code 以及各请求指标对应的列。

**示例**

```dolphindb
accToken = iFinD::getAccessToken(refreshToken)
tb = iFinD::thsBasicData(accToken, "600519.SH,600518.SH", "ths_stock_short_name_stock")
```

**返回样例**

| ths_code  | ths_stock_short_name_stock |
| :-------- | :------------------------- |
| 600519.SH | 贵州茅台                   |
| 600518.SH | 康美药业                   |

---

### thsDateSequence

获取历史财务、估值等具有时间戳的序列数据。

**语法**

```dolphindb
iFinD::thsDateSequence(token, code, indicator, indicatorParam, config, startDate, endDate)
```

**详情**

针对具有时间序列特征的指标进行查询，如历史市盈率、每日换手率等。

**参数**

**token** STRING 类型标量。

**code** STRING 类型标量，多个代码用半角逗号分隔。如 "600519.SH,600030.SH"。

**indicator** STRING 类型标量。

**indicatorParam** STRING 类型标量，指标参数。

**config** STRING 类型标量，配置项。如 "Interval:D" 表示日频。详见 [字典参数常用配置](#5-字典参数常用配置)。

**startDate** STRING 类型标量，格式为 "YYYY-MM-DD"。表示开始日期。

**endDate** STRING 类型标量，格式为 "YYYY-MM-DD"。必须晚于或等于 startDate。

**返回值**

返回一个 Table。包含日期列 time、证券代码列 ths_code 及对应指标列。

**示例**

```dolphindb
tb = iFinD::thsDateSequence(accToken, "600519.SH,600518.SH", "ths_pe_ttm_stock,ths_pb_latest_stock", "100;100", "", "2026-02-01", "2026-03-01")
```

**返回样例**

| time       | ths_code  | ths_pe_ttm_stock | ths_pb_latest_stock |
| :--------- | :-------- | :--------------- | :------------------ |
| 2026-02-02 | 600519.SH | 31.42            | 8.12                |
| ...        | ...       | ...              | ...                 |
| 2026-02-02 | 600518.SH | NULL             | 1.85                |

---

### thsHistoryQuotes

获取标准的日、周、月等历史 K 线行情数据。

**语法**

```dolphindb
iFinD::thsHistoryQuotes(token, code, indicator, config, startDate, endDate)
```

**详情**

用于提取 OHLCV 等标准行情指标，支持前复权、后复权等进阶设置。

**参数**

**token** STRING 类型标量。

**code** STRING 类型标量，多个代码用半角逗号分隔。

**indicator** STRING 类型标量，常用 "open,high,low,close,volume,amount"。

**config** STRING 类型标量，行情配置。如 "CPS:2" 表示前复权。详见 [字典参数常用配置](#5-字典参数常用配置)。

**startDate** STRING 类型标量，格式为 "YYYY-MM-DD"。

**endDate** STRING 类型标量，格式为 "YYYY-MM-DD"。

**返回值**

返回 Table，包含行情序列。

**示例**

```dolphindb
tb = iFinD::thsHistoryQuotes(accToken, "600519.SH,600518.SH", "open,high,low,close,volume", "Fill:Original", "2026-03-01", "2026-03-05")
```

**返回样例**

| time       | ths_code  | open    | high    | low     | close   | volume |
| :--------- | :-------- | :------ | :------ | :------ | :------ | :----- |
| 2026-03-02 | 600519.SH | 1850.10 | 1870.00 | 1845.00 | 1865.00 | 250000 |
| 2026-03-03 | 600519.SH | 1865.00 | 1868.00 | 1852.00 | 1858.50 | 210000 |

---

### thsHighFrequency

获取日内分钟级的明细或 K 线数据。

**语法**

```dolphindb
iFinD::thsHighFrequency(token, code, indicator, config, startTime, endTime)
```

**详情**

提供高精度的日内行情查询，最小颗粒度支持到 1 分钟线。

**参数**

**token** STRING 类型标量。

**code** STRING 类型标量，多个代码用半角逗号分隔。

**indicator** STRING 类型标量。

**config** STRING 类型标量。常用 "Interval:1"。详见 [字典参数常用配置](#5-字典参数常用配置)。

**startTime** STRING 类型标量，格式为 "YYYY-MM-DD HH:mm:ss"。须含秒。

**endTime** STRING 类型标量，格式为 "YYYY-MM-DD HH:mm:ss"。晚于 startTime。

**返回值**

返回 Table。

**示例**

```dolphindb
tb = iFinD::thsHighFrequency(accToken, "600519.SH,600518.SH", "open,high,low,close,volume", "Interval:1,Fill:Original", "2026-03-11 09:30:00", "2026-03-11 15:00:00")
```

**返回样例**

| time                | ths_code  | open    | high    | low     | close   | volume |
| :------------------ | :-------- | :------ | :------ | :------ | :------ | :----- |
| 2026-03-11 09:30:00 | 600519.SH | 1860.00 | 1862.50 | 1859.00 | 1861.20 | 5000   |
| 2026-03-11 09:31:00 | 600519.SH | 1861.20 | 1861.50 | 1858.00 | 1858.50 | 4200   |

---

### thsRealTime

获取最及时的实时行情快照（盘中刷新）。

**语法**

```dolphindb
iFinD::thsRealTime(token, code, indicator)
```

**详情**

获取当前盘中的实时买卖价、最新成交价、涨跌幅等。

**参数**

**token** STRING 类型标量。

**code** STRING 类型标量，多个代码用半角逗号分隔。

**indicator** STRING 类型标量。

**返回值**

返回包含最新行情信息的单行或多行 Table。

**示例**

```dolphindb
tb = iFinD::thsRealTime(accToken, "600519.SH,601318.SH", "latest")
```

**返回样例**

| ths_code  | latest  |
| :-------- | :------ |
| 600519.SH | 1875.40 |
| 600518.SH | 3.25    |

---

## 4. 管理与辅助函数

### getAccessToken

使用 `refreshToken` 获取短期有效的 `accessToken`。

**语法**

```dolphindb
iFinD::getAccessToken(refreshToken)
```

**参数**

**refreshToken** STRING 类型标量，登录同花顺 iFinD 超级命令通过相应工具获取的长效凭证。

**返回值**

返回一个新的 `accessToken` (STRING)。

---

## 5. 字典参数常用配置

| 参数类 (Key)       | 说明     | 可选值 (Value)                                                   |
| :----------------- | :------- | :--------------------------------------------------------------- |
| **Interval** | 周期     | `D` (日), `W` (周), `M` (月), `1` (1分钟), `5` (5分钟) |
| **Days**     | 日历模式 | `Tradedays` (交易日), `Alldays` (自然日)                     |
| **Fill**     | 填充逻辑 | `Previous` (向前填充), `Blank` (空值), `Original` (原始)   |
| **CPS**      | 复权逻辑 | `1` (不复权), `2` (前复权), `3` (后复权)                   |

---

## 6. 技术特性与校验规则

### 6.1 严苛的参数校验

模块内部会对输入进行前置过滤，不合法的输入会直接触发异常提示而不消耗 API 流量：

- **格式校验**：startDate 必须符合 "YYYY-MM-DD"；startTime 必须符合 "YYYY-MM-DD HH:mm:ss"。
- **类型校验**：所有参数必须为 STRING scalar。
- **报错效果**：
  - `code must be a STRING scalar`
  - `startDate must follow 'YYYY-MM-DD'`

### 6.2 dataVol 监控

当查询的代码不存在、指标拼写错误或日期区间无数据时，iFinD 会返回 `dataVol: 0`。模块会拦截此类响应并提示：

> `iFinD Error: No data found for the specified parameters (dataVol=0).`

---

## 7. 进阶用法 (Best Practices)

### 7.1 并发调用示例

利用 DolphinDB 的 `submitJob` 进行多品种并行数据加载：

```dolphindb
def parallelLoad(tk, sym){
    return iFinD::thsRealTime(tk, sym, "latest")
}
submitJob("job1", "IFIND", parallelLoad, accToken, "600519.SH")
```

### 7.2 批量化请求

尽可能在一个 code 字符串中包含多个代码（用逗号分隔）。一次请求多个股票的性能远高于单只股票重复请求。

---

## 8. 常见错误代码参考

| 状态码                   | 含义              | 建议                                                              |
| :----------------------- | :---------------- | :---------------------------------------------------------------- |
| **HTTP 401**       | Token 失效 / 非法 | 请重新调用 `getAccessToken` 或检查 `refreshToken`             |
| **-1302**          | Token 已过期      | 同上                                                              |
| **dataVol: 0**     | 查无数据          | 检查代码后缀 (如 .SH) 或指标名拼写                                |
| **Invalid Format** | 格式错误          | 检查日期/时间字符串是否完全符合 YYYY-MM-DD 或 YYYY-MM-DD HH:mm:ss |

---

*Last modified: 2026-03-18 (v1.1.9)*
