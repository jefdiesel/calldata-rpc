# ChainHost RPC - On-Chain RPC Proxies

Free JSON-RPC proxies for EVM chains, powered by [ChainHost](https://chainhost.online) and Cloudflare Workers. Each endpoint is an [Ethscription](https://ethscriptions.com) name serving as a decentralized RPC proxy with automatic failover.

No API keys. No IPFS. No ENS gateways. Just POST your JSON-RPC request.

## Endpoints

| Chain | Endpoint | Chain ID |
|-------|----------|----------|
| Ethereum Mainnet | `https://mainnetrpc.chainhost.online` | 1 |
| Polygon | `https://137polygon.chainhost.online` | 137 |
| Base | `https://8453base.chainhost.online` | 8453 |
| Arbitrum One | `https://42161.chainhost.online` | 42161 |
| Sepolia Testnet | `https://11155111.chainhost.online` | 11155111 |

Each endpoint has a landing page at its URL with docs, usage examples, and the drop-in module.

## Usage

### Direct fetch

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

### Drop-in Module (~250 bytes)

Paste this into any HTML inscription or client-side app:

```js
(function(){const B='https://mainnetrpc.chainhost.online';window.RPC={call:async(m,p)=>{let r=await fetch(B,{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({jsonrpc:'2.0',id:1,method:m,params:p})});let d=await r.json();return d.error?null:d.result;}};})();
```

Change the URL to target a different chain:

```js
// Base
const B='https://8453base.chainhost.online';

// Polygon
const B='https://137polygon.chainhost.online';

// Arbitrum
const B='https://42161.chainhost.online';
```

Then use it:

```js
// Get block number
RPC.call('eth_blockNumber', []).then(console.log);

// Call a contract
RPC.call('eth_call', [{ to: '0x...', data: '0x...' }, 'latest']).then(console.log);

// Get balance
RPC.call('eth_getBalance', ['0x...', 'latest']).then(console.log);
```

### Get Raw Endpoint List

```
GET https://mainnetrpc.chainhost.online/endpoints
→ ["https://eth.llamarpc.com", "https://eth.drpc.org"]
```

## How It Works

Each RPC proxy is a [ChainHost](https://chainhost.online) subdomain backed by a Cloudflare Worker. The subdomain names (`mainnetrpc`, `137polygon`, `8453base`, `42161`, `11155111`) are Ethscriptions — on-chain names claimed on Ethereum.

When you POST a JSON-RPC request:

1. The CF Worker receives it at the edge (global, fast)
2. Proxies to the first backend RPC endpoint
3. If it fails, automatically tries the next endpoint
4. Returns the result with CORS headers

No client-side failover logic needed. The worker handles it.

### vs. IPFS + ENS approach

| | ChainHost RPC | IPFS/ENS RPCs |
|---|---|---|
| Client module size | ~250 bytes | ~580 bytes |
| Failover | Server-side (invisible) | Client-side (each try = network request) |
| CORS | Handled by worker | Depends on each RPC endpoint |
| Update endpoints | Edit worker config, deploy | Re-pin IPFS, update ENS record |
| Dependencies | Cloudflare Worker | IPFS gateway + ENS gateway |
| Gateway risk | CF edge network | `.limo` / `.link` gateways can go down |
| Identity | Ethscription names | ENS name |

## Currently Supported Chains

| Chain ID | Network | Backend RPCs |
|----------|---------|-------------|
| 1 | Ethereum Mainnet | eth.llamarpc.com, eth.drpc.org |
| 137 | Polygon | polygon-rpc.com, rpc-mainnet.matic.quiknode.pro |
| 8453 | Base | mainnet.base.org, base.drpc.org |
| 42161 | Arbitrum One | arbitrum.drpc.org, public-arb-mainnet.fastnode.io |
| 11155111 | Sepolia | sepolia.drpc.org, 0xrpc.io/sep |

## Example: ETH Price Oracle

See [example.html](example.html) — fetches live ETH/USD price from Chainlink's oracle using `mainnetrpc.chainhost.online`. Under 250 bytes of RPC module code.

## Self-Host

The RPC proxy is part of the [ChainHost Cloudflare Worker](https://github.com/jefdiesel/chainhost). To run your own:

1. Clone: `git clone https://github.com/jefdiesel/chainhost`
2. Edit `cloudflare-worker/subdomain-router.js` — modify `RPC_PROXY_MAP` with your own endpoints
3. Deploy: `cd cloudflare-worker && npx wrangler deploy`
4. Point `*.yourdomain.com` to the worker

## License

Public utility for the Ethereum inscription community.
