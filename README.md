# ChainHost RPC - On-Chain RPC Proxies

Free JSON-RPC proxies for EVM chains, powered by [ChainHost](https://chainhost.online) and Cloudflare Workers.

Each endpoint is an [Ethscription](https://ethscriptions.com) — a name inscribed permanently on Ethereum mainnet (`data:,mainnetrpc`, `data:,8453base`, etc.). The on-chain name IS the RPC identity. No API keys. No IPFS. No ENS gateways. Just POST your JSON-RPC request.

## Endpoints

| Chain | Endpoint | Chain ID |
|-------|----------|----------|
| Ethereum Mainnet | `https://mainnetrpc.chainhost.online` | 1 |
| Polygon | `https://137polygon.chainhost.online` | 137 |
| Base | `https://8453base.chainhost.online` | 8453 |
| Arbitrum One | `https://42161.chainhost.online` | 42161 |
| Sepolia Testnet | `https://11155111.chainhost.online` | 11155111 |

Visit any endpoint in a browser to see its landing page with docs and the drop-in module.

## How It Works

```
Client → Cloudflare Edge → Backend RPC (with failover)
```

1. You POST a JSON-RPC request to `mainnetrpc.chainhost.online`
2. Cloudflare's wildcard route (`*.chainhost.online`) sends it to the ChainHost worker
3. The worker sees subdomain `mainnetrpc`, looks it up in `RPC_PROXY_MAP`
4. Proxies the request to the first backend RPC (`eth.llamarpc.com`)
5. If that fails, tries the next (`eth.drpc.org`), then the next (`rpc.ankr.com/eth`), etc.
6. Returns the result with CORS headers

All failover happens server-side. The client sends one request and gets one response.

### Caching

Read-only methods are cached in Cloudflare KV to reduce backend load and speed up responses:

- **12 second TTL**: `eth_blockNumber`, `eth_gasPrice`, `eth_getBlockByNumber` — changes every block
- **5 minute TTL**: `eth_call`, `eth_getBalance`, `eth_getCode`, `eth_getTransactionCount`, `eth_getStorageAt`, `eth_chainId`, `net_version` — changes less frequently
- **Not cached**: `eth_sendRawTransaction`, `eth_estimateGas`, and everything else — always proxied fresh

Cached responses return an `X-Cache: HIT` header. Misses return `X-Cache: MISS`.

### Rate Limiting

60 requests per minute per IP per chain. If you hit the limit you'll get a `429` with a `Retry-After: 60` header. This protects the backend RPCs and keeps the free tier sustainable.

### Why On-Chain Names?

Each subdomain (`mainnetrpc`, `8453base`, `137polygon`, `42161`, `11155111`) is an Ethscription — inscribed as `data:,mainnetrpc` (etc.) on Ethereum mainnet. The name is permanently recorded on-chain and the current owner controls what it serves through ChainHost.

This means:
- The RPC identity is verifiable on-chain
- Ownership is transferable (send the ethscription to a new wallet)
- Anyone can check who controls an endpoint by looking up the name on [ethscriptions.com](https://ethscriptions.com)

## Drop-in Module

Paste this into any HTML inscription or client-side app. ~250 bytes, no dependencies:

```js
(function(){const B='https://mainnetrpc.chainhost.online';window.RPC={call:async(m,p)=>{let r=await fetch(B,{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({jsonrpc:'2.0',id:1,method:m,params:p})});let d=await r.json();return d.error?null:d.result;}};})();
```

Swap the URL for a different chain:

```js
// Ethereum
const B='https://mainnetrpc.chainhost.online';

// Base
const B='https://8453base.chainhost.online';

// Polygon
const B='https://137polygon.chainhost.online';

// Arbitrum
const B='https://42161.chainhost.online';

// Sepolia
const B='https://11155111.chainhost.online';
```

### Usage

```js
// Get block number
RPC.call('eth_blockNumber', []).then(console.log);

// Call a contract (e.g. Chainlink ETH/USD oracle)
RPC.call('eth_call', [{
  to: '0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419',
  data: '0x50d25bcd'
}, 'latest']).then(result => {
  const price = Number(BigInt(result)) / 1e8;
  console.log('ETH price: $' + price.toFixed(2));
});

// Get balance
RPC.call('eth_getBalance', ['0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045', 'latest'])
  .then(console.log);

// Get gas price
RPC.call('eth_gasPrice', []).then(console.log);
```

### Direct fetch (no module needed)

```js
fetch('https://mainnetrpc.chainhost.online', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    jsonrpc: '2.0', id: 1,
    method: 'eth_blockNumber', params: []
  })
}).then(r => r.json()).then(console.log);
```

### Get raw endpoint list

```
GET https://mainnetrpc.chainhost.online/endpoints
→ ["https://eth.llamarpc.com","https://eth.drpc.org","https://rpc.ankr.com/eth","https://ethereum-rpc.publicnode.com"]
```

## vs. IPFS + ENS Approach

This project is inspired by [calldata-rpc](https://github.com/chopperdaddy/calldata-rpc) by [@chopperdaddy](https://github.com/chopperdaddy), which hosts RPC endpoint lists on IPFS via ENS (`rpcs.calldata.eth`). Both approaches serve the same community — here's how they differ:

| | ChainHost RPC (proxy) | calldata-rpc (IPFS config) |
|---|---|---|
| **Architecture** | Client → CF Worker → RPC | Client → IPFS gateway → JSON config, then Client → RPC directly |
| **Client module** | ~250 bytes | ~580 bytes |
| **Failover** | Server-side (invisible to client) | Client-side (each retry = visible network request) |
| **CORS** | Handled by the worker | Depends on each individual RPC endpoint |
| **Updating endpoints** | Edit worker config, `wrangler deploy`, instant | Re-pin to IPFS, update ENS record, wait for propagation |
| **Dependencies** | Cloudflare Workers | IPFS gateway (`.limo`/`.link`) + ENS |
| **Identity** | Ethscription names on mainnet | ENS name (`rpcs.calldata.eth`) |
| **Rate limit distribution** | Concentrated (all traffic from CF datacenter IPs) | Distributed (each user hits RPCs from their own IP) |

**The tradeoff**: ChainHost RPC gives you simpler client code, zero CORS headaches, and instant endpoint updates. The IPFS approach distributes rate limits better because each user hits backend RPCs from their own IP rather than through a shared proxy.

**They complement each other.** Use ChainHost endpoints as primary (fast, simple, server-side failover). Use the IPFS config as a fallback list if the proxy is down. Both are public utilities for the same community.

### Why mirrors matter

ChainHost is open source and supports [mirrors](https://chost.app) — anyone can deploy the same worker on their own domain. Each mirror (`*.chainhost.online`, `*.chost.app`, `*.immutable.church`, or your own) proxies RPC traffic from different Cloudflare datacenter IPs. More mirrors = more distributed egress = harder for backend RPCs to rate limit any single source. The ethscription names resolve on every mirror that runs the worker.

## Backend RPC Endpoints

| Chain ID | Network | Backend RPCs |
|----------|---------|-------------|
| 1 | Ethereum Mainnet | eth.llamarpc.com, eth.drpc.org, rpc.ankr.com/eth, ethereum-rpc.publicnode.com |
| 137 | Polygon | polygon-rpc.com, rpc-mainnet.matic.quiknode.pro |
| 8453 | Base | mainnet.base.org, base.drpc.org |
| 42161 | Arbitrum One | arbitrum.drpc.org, public-arb-mainnet.fastnode.io |
| 11155111 | Sepolia | sepolia.drpc.org, 0xrpc.io/sep |

## Example

See [example.html](example.html) — a complete HTML inscription that fetches live ETH/USD price from Chainlink's price oracle through `mainnetrpc.chainhost.online`. The entire RPC module is ~250 bytes.

## Self-Host

The RPC proxy is part of the [ChainHost Cloudflare Worker](https://github.com/jefdiesel/chainhost). To run your own mirror:

1. Fork & clone: `git clone https://github.com/jefdiesel/chainhost`
2. Edit `cloudflare-worker/subdomain-router.js` — modify `RPC_PROXY_MAP` with your own backend endpoints
3. Deploy: `cd cloudflare-worker && npx wrangler deploy`
4. Point `*.yourdomain.com` to the worker in your Cloudflare dashboard

Your mirror will serve ChainHost sites AND proxy RPC requests for every chain in the map.

## License

Public utility for the Ethereum inscription community.
