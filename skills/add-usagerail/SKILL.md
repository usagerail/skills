---
name: add-usagerail
description: Add API keys, per-customer rate limits, quotas and usage metering to a Node.js or Python API server with the UsageRail SDK. Use when someone wants to sell access to their API, charge by usage through their own Stripe account, or keep one customer from using up their API.
tags: [api, api-keys, rate-limiting, usage-billing, stripe, fastapi, express, django, flask, hono]
---

# Add UsageRail to an API

UsageRail's SDK checks API keys, applies rate limits and quotas, and counts usage per customer inside the user's own API server. Requests never pass through UsageRail. Counts go to UsageRail in the background every 60 seconds.

## Before you change anything

1. Find the framework. The SDKs cover Express, Hono, Fastify and Next.js on Node.js 22+, and FastAPI, Starlette, Flask and Django on Python 3.10+. For other languages, use the HTTP check endpoint described at https://usagerail.com/docs. It adds one network call per request.
2. Ask how callers authenticate today. With no keys yet, use `auth: "api-key"` and UsageRail issues the keys. With existing keys or logins, use `auth: "custom"` with a function that returns the customer's ID. The docs for each framework show how.
3. The user needs an app key from https://app.usagerail.com. Read it from the `USAGERAIL_APP_KEY` environment variable. Never write the key into source code, and never ask the user to paste it into the chat.

## Wire it in

### FastAPI or Starlette

```bash
pip install usagerail
```

```python
import os

from fastapi import FastAPI, Request
from usagerail.fastapi import UsageRailMiddleware

app = FastAPI()
app.add_middleware(UsageRailMiddleware, app_key=os.environ["USAGERAIL_APP_KEY"], auth="api-key")


@app.post("/v1/generate")
async def generate(request: Request):
    result = await run_model(await request.json())
    request.state.usagerail.track(result.tokens)  # optional: count tokens instead of requests
    return result.body
```

### Flask

```python
import os

from flask import Flask
from usagerail.flask import UsageRail

app = Flask(__name__)
UsageRail(app, app_key=os.environ["USAGERAIL_APP_KEY"], auth="api-key")
```

Inside a route, `g.usagerail.track(n)` counts custom units.

### Django

```python
# settings.py
MIDDLEWARE = [
    # ...
    "usagerail.django.UsageRailMiddleware",
]
USAGERAIL = {"app_key": os.environ["USAGERAIL_APP_KEY"], "auth": "api-key"}
```

Inside a view, `request.usagerail.track(n)` counts custom units.

### Express

```bash
npm install @usagerail/node
```

```js
import express from "express";
import { usageRail } from "@usagerail/node/express";

const app = express();

app.use(usageRail({
  appKey: process.env.USAGERAIL_APP_KEY,
  auth: "api-key",
}));

app.post("/v1/generate", express.json(), async (req, res) => {
  const result = await generate(req.body);
  req.usageRail.track(result.tokens); // optional: count tokens
  res.json(result);
});
```

Hono and Fastify have their own entry points: `@usagerail/node/hono` and `@usagerail/node/fastify`.

## How callers send their key

In the `Authorization: Bearer <key>` header, or in `x-api-key` when there's no bearer token.

## What rejected requests look like

| Status | Meaning |
|---|---|
| 401 | Unknown or revoked key |
| 402 | Prepaid credit used up |
| 403 | Suspended customer |
| 429 | Rate limit or quota reached |
| 503 | UsageRail unreachable, and the app is set to refuse |

Update the API's own docs and client error handling for these.

## Tell the user before they deploy

- Rate limits are counted per server instance by default.
- The Express middleware and the Python SDK don't check or count WebSocket connections. Hono counts a successful handshake once, and so does Fastify with `@fastify/websocket`.
- Counts are sent every 60 seconds. A process killed with `kill -9` or by running out of memory loses the counts it hadn't sent, and they're never billed. A normal shutdown sends them first.
- To charge customers, the user connects their own Stripe account. Money goes straight there, and UsageRail is not a merchant of record.

Setup for each framework, options and known limitations: https://usagerail.com/docs
