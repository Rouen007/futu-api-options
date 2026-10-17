# Futu API Options

通过本机运行的 Futu OpenD 和 Python SDK 查询美股期权合约、快照报价及 Greeks。此 skill 只读市场数据，不下单。详细边界和 API 示例见 [SKILL.md](SKILL.md)。

## 能做什么

- 查询指定美股标的和到期日的期权链。
- 用期权链返回的合约代码查询快照、买卖价、最新价、隐含波动率和可用 Greeks。
- 报告各合约更新时间、权限限制和数据缺口。

## 安装

此仓库是一个 agent skill，不包含独立服务。把它安装到 agent 平台支持的 skills 目录，保持 SKILL.md 在目录根部。若要在自己的 Python 程序中调用 Futu API，可在虚拟环境安装 SDK：

~~~bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install futu-api
~~~

还需要安装并启动 Futu OpenD，并在 OpenD 内由用户登录。SDK 默认连接地址应保持本机回环：127.0.0.1:11111。不要把 OpenD 端口暴露给局域网或互联网；不要把账号密码或验证码写入脚本。

## 最小查询流程

1. 在 OpenD 中确认已登录，并确认当前账号有相应行情权限。
2. 连接本机 OpenD，用 get_option_chain 查询一个到期日。
3. 使用返回的合约代码调用 get_market_snapshot。
4. 按每个合约的 update_time 标注时间，并在 finally 中关闭 quote context。

示例（查询链与首批最多 400 个合约的快照）：

~~~python
from futu import OpenQuoteContext, RET_OK

ctx = OpenQuoteContext(host="127.0.0.1", port=11111)
try:
    ret, chain = ctx.get_option_chain(
        code="US..SPX",
        start=expiration,
        end=expiration,
    )
    if ret != RET_OK:
        raise RuntimeError(chain)

    codes = chain["code"].tolist()
    ret, snapshot = ctx.get_market_snapshot(codes[:400])
    if ret != RET_OK:
        raise RuntimeError(snapshot)

    columns = [
        "code", "update_time", "bid_price", "ask_price", "last_price",
        "option_implied_volatility", "option_delta", "option_gamma",
        "option_vega", "option_theta", "option_rho",
    ]
    print(snapshot[[c for c in columns if c in snapshot.columns]])
finally:
    ctx.close()
~~~

将 expiration 替换成尚未到期的日期字符串。超过 400 个合约时分批请求；官方文档说明期权链每 30 秒最多请求 10 次，快照每 30 秒最多请求 60 次。SPX 指数期权链使用 US..SPX 作为 underlying，具体合约代码必须使用 API 返回值，不要自行拼接。

## 连接和数据问题

- **连接失败或端口关闭**：确认 OpenD 已启动并登录；停止无意义的自动重试。
- **合规确认未完成**：由账号持有人按 Futu 官方流程完成问卷/协议，然后重新登录 OpenD。
- **行情权限不足**：报告 OpenD 返回的权限错误，不要替用户购买行情卡或更改权限。
- **市场已收盘**：快照可能是上一交易时段数据，按返回时间说明，不要称作实时。
- **SPX 指数快照不可用**：指数行情限制不代表 SPXW 期权链也不可用。

## 官方文档

- [OpenD 概览](https://openapi.futunn.com/futu-api-doc/opend/opend-intro.html)
- [首次登录与 OpenD 设置](https://openapi.futunn.com/futu-api-doc/quick/opend-base.html)
- [权限、额度和合规确认](https://openapi.futunn.com/futu-api-doc/intro/authority.html)
- [期权链接口](https://openapi.futunn.com/futu-api-doc/quote/get-option-chain.html)
- [快照接口](https://openapi.futunn.com/futu-api-doc/quote/get-market-snapshot.html)

## 文件结构

- [SKILL.md](SKILL.md)：完整连接规则、查询步骤、限制和示例。
- **agents/openai.yaml**：Agent 发现元数据。

## 许可证

当前仓库未附 LICENSE 文件。公开可见不等于授予复制、修改或再分发许可。
