# Re: Connecting Claude to Bloomberg without the paid AI product

Thanks for the good question. Short answer: **you can't bypass Bloomberg's access controls, but you may not need the paid product to build a personal research tool.**

## 1. Two clarifications
- **The Claude API isn't something you download.** It's a cloud service your code calls over the internet. You can't run it on a local network.
- **MCP itself is free.** MCP (Model Context Protocol) is an open standard for connecting AI to tools. Bloomberg announced **Enterprise MCP** on 29 Sep 2026. It's part of its enterprise **Data License** product, so the cost is for licensed data, not for MCP. Bloomberg says a version for Terminal users is coming but hasn't given a date. [1][2]

## 2. What not to do
Don't try to get around Bloomberg's login or license, for example by pulling data "through the local network." That breaks Bloomberg's terms. On a university Terminal, it could also put the whole school's contract at risk.

## 3. A legitimate DIY route
- **Desktop API (BLPAPI).** On the Windows PC where the Terminal is running and logged in, Python can pull data through Bloomberg's official package (`blpapi`, or the wrapper `xbbg`). It connects to the Terminal on `localhost:8194`. [3][4]
- **Connect it to Claude.** Wrap your Python function as a tool with the Claude API's *tool use* feature. You can also build a small local MCP server with the open-source SDK. Claude calls your function when it needs data.
- **Simplest option.** Export data to CSV with the Excel add-in (BDP/BDH), then give the file to Claude for analysis.

```python
# Sketch only. Needs a logged-in Bloomberg Terminal on the same Windows PC.
from xbbg import blp

def get_price(ticker: str):
    return blp.bdp(ticker, "PX_LAST").to_dict()

tools = [{
    "name": "get_price",
    "description": "Last price for a Bloomberg ticker, e.g. 'AAPL US Equity'",
    "input_schema": {"type": "object",
                     "properties": {"ticker": {"type": "string"}},
                     "required": ["ticker"]},
}]
# Pass `tools` to client.messages.create(...). When Claude returns a tool_use
# block, run get_price and send the result back as a tool_result message.
```

## 4. Check before you build
University Bloomberg agreements often say data, raw or derived, must stay on the Terminal it came from. [5] Sending it to an outside cloud service like Claude may not be allowed. **Ask the library or our Bloomberg account manager first, in writing.** Don't store, share or redistribute the data. *(This is not legal advice.)*

## 5. A practical plan
1. Build the tool first with **fake (mock) data** or free public data (FRED, exchange data, company filings).
2. Add simple safety rules: a few tickers per request, short date ranges, and send Claude only summary numbers instead of raw tables.
3. Switch to real Bloomberg data only after you've confirmed what's allowed.

I've drafted a full design (PRD, screen map, database design) for this tool. Happy to go through it with you.

**Sources**
[1] https://cyprusshippingnews.com/2026/10/06/bloomberg-launches-enterprise-mcp-to-seamlessly-connect-bloomberg-data-with-clients-enterprise-ai-applications
[2] https://mcpplaygroundonline.com/blog/bloomberg-enterprise-mcp
[3] https://www.bloomberg.com/professional/support/api-library
[4] https://merton.readthedocs.io/en/latest/cookbook/bloomberg.html
[5] https://www.library.hbs.edu/Find/Databases/Bloomberg
