---
name: futu-api-options
description: "登录 Futu OpenD，通过 Futu API 查询美股期权链、快照报价和 Greeks；连接失败时指导用户登录 Futu。"
---

# Futu API Options

Use this skill when connecting to Futu OpenD to read U.S. options chains, contract quotes, or Greeks. Keep requests read-only; do not place, modify, or cancel orders.

## Connection and login

- Futu API requires the local OpenD gateway. Default to `127.0.0.1:11111`; keep the listener on localhost and do not expose it to the LAN or internet.
- Connect with the Futu SDK (`futu-api`) using `OpenQuoteContext(host="127.0.0.1", port=11111)`. The Futu ID, registered email, or phone and password are entered by the user in OpenD. Never request or store passwords or verification codes in chat or scripts.
- If OpenD is not running, the port is closed, or the SDK cannot connect, stop retrying and tell the user: **“Futu API 还没连通，请先打开并登录 Futu OpenD（富途牛牛），保持它运行后告诉我，我再重试。”** Do not fall back to IBKR unless the user asks.
- If the API responds that the questionnaire or agreement is incomplete, distinguish this from a gateway connection failure. Direct the user to the “合规确认” section of [Futu API 权限与额度](https://openapi.futunn.com/futu-api-doc/intro/authority.html), where Futu users can open the official questionnaire/agreement flow. The user must answer and submit it. Afterward, have the user restart or log in to OpenD again, then retry once. Never answer the questionnaire, accept agreements, or echo user-specific redirect URLs or `user_id` values.
- If OpenD connects but the API reports missing market-data permission, report the permission shown in OpenD and the returned error. Do not purchase a quote card or change account permissions without the user's explicit request.

## Retrieve an option chain

1. Resolve the underlying and expiration from the user's request. Use a future or current expiration; expired chains are unsupported. For one expiration, pass it as both `start` and `end`. The API allows at most a 30-day expiration range and 10 option-chain requests per 30 seconds.
2. Create an `OpenQuoteContext` for `127.0.0.1:11111`, then call `get_option_chain(code=underlying, start=expiration, end=expiration)`. For SPX index options, use the underlying code `US..SPX`; the returned contracts use codes such as `US.SPXW...`. Always use the contract codes returned by the chain rather than constructing them manually.
3. Treat the chain response as contract metadata. For live or last-session quotes, pass returned option codes to `get_market_snapshot(code_list)`. Each snapshot has its own `update_time`; report timestamps per contract and do not imply the entire chain shares one synchronized timestamp. Batch requests at no more than 400 codes each.
4. Include only useful fields in output: contract code/name, expiration, call/put, strike, update time, bid/ask, last price, implied volatility, and available Greeks (`option_delta`, `option_gamma`, `option_vega`, `option_theta`, `option_rho`). Close the quote context in a `finally` block.

Example for one expiration:

```python
from futu import OpenQuoteContext, RET_OK

ctx = OpenQuoteContext(host="127.0.0.1", port=11111)
try:
    ret, chain = ctx.get_option_chain(
        code="US..SPX", start=expiration, end=expiration
    )
    if ret != RET_OK:
        print("Option-chain request failed:", chain)
    else:
        codes = chain["code"].tolist()
        ret, snapshot = ctx.get_market_snapshot(codes[:400])
        if ret == RET_OK:
            print(snapshot[[
                "code", "update_time", "bid_price", "ask_price", "last_price",
                "option_implied_volatility", "option_delta", "option_gamma",
                "option_vega", "option_theta", "option_rho",
            ]])
finally:
    ctx.close()
```

For chains longer than 400 contracts, request snapshots in batches and obey the documented request limits. If `from futu import ...` fails, use the official `futu-api` Python SDK in an isolated environment rather than modifying system Python unnecessarily.

## Data and permission limits

- Futu's option-chain call returns static contract information; use returned codes to request dynamic quote data. See [Get Option Chain](https://openapi.futunn.com/futu-api-doc/quote/get-option-chain.html) and [Get Market Snapshot](https://openapi.futunn.com/futu-api-doc/quote/get-market-snapshot.html).
- U.S. index underlying quotes may be unavailable through Futu API even when index-option contracts can be queried. Do not interpret an unsupported SPX index snapshot as a failure of the SPXW option chain; report the limitation and use a separately authorized underlying-data source if needed.
- Read the quote entitlement in OpenD for the current account. U.S. option quote access can depend on account eligibility or a market-data card, and terms may change. Do not assume every account has real-time OPRA data.
- When the market is closed, snapshots may show the prior session's `update_time`. State that clearly; do not claim a live quote-latency test from weekend or closed-market data.

## Official references

- [OpenD overview](https://openapi.futunn.com/futu-api-doc/opend/opend-intro.html)
- [Visual OpenD setup and first-login requirements](https://openapi.futunn.com/futu-api-doc/quick/opend-base.html)
- [Authorities, quota, and compliance confirmation](https://openapi.futunn.com/futu-api-doc/intro/authority.html)
- [U.S. option-code format and index-quote availability](https://openapi.futunn.com/futu-api-doc/en/qa/quote.html)
