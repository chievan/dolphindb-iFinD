# iFinD (同花顺 HTTP API) DolphinDB 模块使用说明

## 1. 模块介绍

`ifind` 模块是对同花顺 iFinD HTTP API 的深度封装，旨在为 DolphinDB 用户提供原生、高速、稳定的金融数据接入方案。通过该模块，用户可以在 DolphinDB 脚本中直接获取 A 股、港股、美股、基金、债券、指数等全品种行情及各种财务、估值指标，并自动解析为结构化的 `Table` 对象，极大地简化了量化研究的数据预处理流程。

---

## 2. 快速开始

### 2.1 依赖安装

本模块依赖 DolphinDB 内置的 `httpClient` 网络插件：

```dolphindb
try { loadPlugin("httpClient") } catch(ex) {}
use ifind
```

### 2.2 凭证准备 (Refresh Token)

1.  登录同花顺 iFinD 终端。
2.  点击 **工具 -> refresh_token 查询/更新**。
3.  获取属于您的长效 `refresh_token`（建议在 DolphinDB 中使用单引号 `'...'` 包裹）。

---

## 3. 核心 API 参考 (API Reference)

### 3.1 基础数据接口 (THS_BD)

获取证券的基础资料、实时快照或特定日期的指标值。

**语法**: `ifind::THS_BD(token, codes, indicators, [indipara])`
- **参数**:
    - `token`: `refreshToken` (自动登录) 或 `accessToken` (高效复用)。
    - `codes`: 证券代码。支持 `STRING` (如 `"600519.SH"`) 或 `STRING VECTOR` (如 `["600519.SH", "000001.SZ"]`)。
    - `indicators`: 指标名称。多个指标用逗号分隔，如 `"ths_stock_short_name_stock,ths_open_price_stock"`。
    - `indipara`: **(可选)** 指标参数。如 `"date:20231231;type:1"`。
- **示例**: `ifind::THS_BD(token, "600519.SH", "ths_stock_short_name_stock")`

**返回值示例**:

| ths_code | ths_stock_short_name_stock |
| :--- | :--- |
| 600519.SH | 贵州茅台 |

---

### 3.2 日期序列接口 (THS_DS)

获取历史财务数据、估值指标等具有时间序列特征的数据。

**语法**: `ifind::THS_DS(token, codes, indicators, indipara, functionpara, startdate, enddate)`
- **参数**:
    - `token/codes/indicators`: 同上。
    - `indipara`: 指标特定参数，通常传空字符串 `""` 即可。
    - `functionpara`: 功能参数。控制周期和填充方式，如 `"Interval:D,Days:Tradedays"`。
    - `startdate`: 开始日期。支持 `"YYYY-MM-DD"`。
    - `enddate`: 结束日期。支持 `"YYYY-MM-DD"`。
- **示例**: `ifind::THS_DS(token, "600519.SH", "ths_pe_ttm_stock", "", "", "2024-01-01", "2024-01-03")`

**返回值示例**:

| ths_code | time | ths_pe_ttm_stock |
| :--- | :--- | :--- |
| 600519.SH | 2024-01-02 | 30.25 |
| 600519.SH | 2024-01-03 | 29.88 |

---

### 3.3 历史行情接口 (THS_HQ)

获取标准的历史日/周/月 K 线行情（OHLCV）。

**语法**: `ifind::THS_HQ(token, codes, indicators, functionpara, startdate, enddate)`
- **参数**:
    - `indicators`: 常用指标如 `"open,high,low,close,volume,amount"`。
    - `functionpara`: 控制复权和周期。如 `"Interval:D,CPS:2,Fill:Original"` (CPS:2 为前复权)。
    - `startdate/enddate`: 历史行情起止日期。
- **示例**: `ifind::THS_HQ(token, "600519.SH", "open,close", "Interval:D", "2024-01-01", "2024-01-03")`

**返回值示例**:

| ths_code | time | open | close |
| :--- | :--- | :--- | :--- |
| 600519.SH | 2024-01-02 | 1700.0 | 1720.5 |
| 600519.SH | 2024-01-03 | 1721.0 | 1715.0 |

---

### 3.4 高频序列接口 (THS_HF)

获取日内高频 K 线数据（如 1 分钟线、5 分钟线）。

**语法**: `ifind::THS_HF(token, codes, indicators, functionpara, starttime, endtime)`
- **参数**:
    - `starttime`: 开始时间。格式 `"YYYY-MM-DD HH:mm:ss"`。
    - `endtime`: 结束时间。格式 `"YYYY-MM-DD HH:mm:ss"`。
    - `functionpara`: 常用周期 `"Interval:1"`。
- **示例**: `ifind::THS_HF(token, "600519.SH", "close", "Interval:1", "2024-03-11 09:30:00", "2024-03-11 09:32:00")`

**返回值示例**:

| ths_code | time | close |
| :--- | :--- | :--- |
| 600519.SH | 2024-03-11 09:31:00 | 1705.5 |
| 600519.SH | 2024-03-11 09:32:00 | 1706.2 |

---

### 3.5 实时行情接口 (THS_RQ)

获取最实时的盘中指标快照。

**语法**: `ifind::THS_RQ(token, codes, indicators)`
- **参数**:
    - `indicators`: 常用 `"latest,changeRatio,bid1,ask1"`。
- **示例**: `ifind::THS_RQ(token, "600519.SH", "latest,changeRatio")`

**返回值示例**:

| ths_code | latest | changeRatio |
| :--- | :--- | :--- |
| 600519.SH | 1710.2 | 0.45 |

---

## 4. 管理与辅助函数

### 4.1 模块信息 (module_info)

查询当前安装的模块版本及元数据。
- **调用**: `ifind::module_info()`

### 4.2 认证登录 (getAccessToken)

使用 `refreshToken` 获取短期有效的 `accessToken`。
- **语法**: `accToken = ifind::getAccessToken(refreshToken)`

### 4.3 底层调用 (getApiData)

直接向底层 API 发送请求（常用于调用暂未封装的特殊接口）。
- **语法**: `ifind::getApiData(token, apiName, params)`

---

## 5. 参数字典常用配置

| 参数类 (Key) | 说明 | 可选值 (Value) |
| :--- | :--- | :--- |
| **Interval** | 周期 | `D` (日), `W` (周), `M` (月), `1` (1分钟), `5` (5分钟) |
| **Days** | 日历模式 | `Tradedays` (交易日), `Alldays` (自然日) |
| **Fill** | 填充逻辑 | `Previous` (向前填充), `Blank` (空值), `Original` (原始) |
| **CPS** | 复权逻辑 | `1` (不复权), `2` (前复权), `3` (后复权) |

---

## 6. 参数校验与错误排查

从 **v1.0.8** 开始，模块引入了强类型校验：
- **数据类型错误**: 若将 `codes` 传为数字而非字符串，将抛出 `codes must be a STRING or STRING VECTOR`。
- **空值拦截**: `NULL` 值入口会被自动拦截并显示具体参数名。
- **认证错误**:
  - `HTTP 401`: Token 损坏或已失效。
  - `-1301`: Refresh Token 填写错误。

---

## 7. 性能优化建议 (Best Practices)

1.  **Token 复用**：在一个长脚本或循环中，先调用 `getAccessToken` 获得临时 Token，并将其传入后续业务接口。这可以避免每次调用也要请求鉴权服务器，性能提升显著。
2.  **批量请求**：`codes` 参数支持向量。一次请求 50 只股票的性能远高于由于 50 次单请求。
3.  **并发执行**：本模块在 DolphinDB 环境下是线程安全的，可通过 `submitJob` 并发加载不同品种的数据。
