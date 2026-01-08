# Onchain RPCs Configuration - Usage Guide

## Overview

**Purpose**: The onchain RPCs configuration provides a decentralized, chain-agnostic solution for HTML inscriptions and client-side applications to dynamically fetch reliable RPC endpoints for Ethereum-compatible networks. Instead of hardcoding RPC URLs (which can break, become rate-limited, or require updates), your inscriptions can automatically discover and use the best available RPC endpoints for any supported chain.

**How It Works**: The `rpcs.json` file is hosted on IPFS and accessible via ENS through `rpcs.calldata.eth`. This means the configuration is:
- **Decentralized**: No single point of failure - hosted on IPFS
- **Censorship-resistant**: Accessible via ENS, independent of traditional DNS
- **Always up-to-date**: Updates propagate automatically through IPFS
- **Chain-agnostic**: Single endpoint works for all supported networks
- **Failover-ready**: Multiple RPC endpoints per chain for reliability

## Endpoint

The onchain RPCs configuration is available at:
- **Primary**: `https://rpcs.calldata.eth.limo`
- **Fallback**: `https://rpcs.calldata.eth.link`

Both endpoints resolve to the same IPFS-hosted JSON file via ENS, providing redundancy and ensuring availability even if one gateway is down.

## Currently Supported Chains

The current configuration supports the following chains:

| Chain ID | Network | RPC Endpoints |
|----------|---------|---------------|
| `1` | Ethereum Mainnet | eth.llamarpc.com, eth.drpc.org |
| `11155111` | Sepolia Testnet | sepolia.drpc.org, 0xrpc.io/sep |
| `8453` | Base Mainnet | mainnet.base.org, base.drpc.org |
| `42161` | Arbitrum One | arbitrum.drpc.org, public-arb-mainnet.fastnode.io |
| `137` | Polygon | polygon-rpc.com, rpc-mainnet.matic.quiknode.pro |

## JSON Structure

The file structure is simple - chain IDs as string keys, with arrays of RPC endpoint URLs:

```json
{
  "1": [
    "https://eth.llamarpc.com",
    "https://eth.drpc.org"
  ],
  "11155111": [
    "https://sepolia.drpc.org",
    "https://0xrpc.io/sep"
  ]
}
```

## Modular RPC Module (Drop-in Ready)

### Complete Module

Copy and paste this code into your inscription. It's completely self-contained and modular:

```javascript
(function(CHAIN_ID){
  const E='rpcs.calldata.eth';
  let RPCs=[],loaded=false;
  
  async function loadRPCs(){
    if(loaded)return RPCs;
    const s=String(CHAIN_ID);
    try{
      let r=await fetch('https://'+E+'.limo');
      if(r.ok){
        let d=await r.json();
        if(d[s]&&Array.isArray(d[s])){RPCs=d[s];loaded=true;return RPCs;}
      }
    }catch{}
    try{
      let r=await fetch('https://'+E+'.link');
      if(r.ok){
        let d=await r.json();
        if(d[s]&&Array.isArray(d[s])){RPCs=d[s];loaded=true;return RPCs;}
      }
    }catch{}
    return RPCs;
  }
  
  async function call(method,params){
    if(!loaded)await loadRPCs();
    for(let rpc of RPCs){
      try{
        let r=await fetch(rpc,{
          method:'POST',
          headers:{'Content-Type':'application/json'},
          body:JSON.stringify({jsonrpc:'2.0',id:1,method:method,params:params})
        });
        if(r.ok){
          let d=await r.json();
          if(!d.error)return d.result;
        }
      }catch{}
    }
    return null;
  }
  
  // Export: RPC is the const you use, call() is the function
  window.RPC={endpoints:()=>loadRPCs(),call:call};
})(1); // <-- Set your CHAIN_ID here

// Usage:
// RPC.call('eth_blockNumber',[]).then(console.log);
// RPC.endpoints().then(rpcs=>console.log('RPCs:',rpcs));
```

### Minified Version (Smallest Size)

For maximum size savings, use this minified version:

```javascript
(function(C){const E='rpcs.calldata.eth';let R=[],L=0;async function load(){if(L)return R;const s=String(C);try{let r=await fetch('https://'+E+'.limo');if(r.ok){let d=await r.json();if(d[s]&&Array.isArray(d[s])){R=d[s];L=1;return R;}}}catch{}try{let r=await fetch('https://'+E+'.link');if(r.ok){let d=await r.json();if(d[s]&&Array.isArray(d[s])){R=d[s];L=1;return R;}}}catch{}return R;}async function call(m,p){if(!L)await load();for(let r of R){try{let x=await fetch(r,{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({jsonrpc:'2.0',id:1,method:m,params:p})});if(x.ok){let d=await x.json();if(!d.error)return d.result;}}catch{}}return null;}window.RPC={endpoints:load,call:call};})(1);
```

**Size**: ~580 bytes (minified)

### Usage Examples

Once the module is loaded, use it like this:

```javascript
// Get current block number
RPC.call('eth_blockNumber', []).then(block => console.log('Block:', block));

// Call a contract function
const contractAddress = '0x...';
const functionData = '0x...'; // ABI-encoded function call
RPC.call('eth_call', [{to: contractAddress, data: functionData}, 'latest'])
  .then(result => console.log('Result:', result));

// Get RPC endpoints array (useful for direct fetch)
RPC.endpoints().then(rpcs => {
  console.log('Available RPCs:', rpcs);
  // rpcs is an array: ['https://eth.llamarpc.com', 'https://eth.drpc.org']
});

// Get balance
RPC.call('eth_getBalance', ['0x...', 'latest']).then(balance => {
  console.log('Balance:', balance);
});
```

### Features

- ✅ **Modular**: Drop-in code, no dependencies
- ✅ **Cached**: RPCs loaded once and reused
- ✅ **Failover**: Tries multiple endpoints automatically
- ✅ **Simple API**: `RPC.call(method, params)` and `RPC.endpoints()`
- ✅ **Minimal**: Under 600 bytes minified
- ✅ **Global**: Available via `window.RPC` after initialization

### The RPC Const

After initialization, `window.RPC` (or just `RPC`) provides:

- **`RPC.call(method, params)`**: Make an RPC call with automatic failover
  - Returns a Promise that resolves to the RPC result or `null` if all fail
  - Example: `RPC.call('eth_blockNumber', [])`
  
- **`RPC.endpoints()`**: Get the array of RPC URLs for your chain
  - Returns a Promise that resolves to an array of RPC endpoint strings
  - Example: `RPC.endpoints().then(rpcs => console.log(rpcs))`
  - Useful if you want to use the RPC URLs directly with `fetch()`

The RPC endpoints are automatically fetched and cached on first use, so subsequent calls are instant.

### Example: ETH Price Oracle inscription

Here's an example that fetches and displays the current ETH price in USD using Chainlink's price oracle:

```html
<!DOCTYPE html>
<html>
<head>
<meta charset=utf-8>
<title>ETH Price</title>
<style>
body{margin:0;padding:20px;font-family:system-ui;background:#000;color:#fff;display:flex;flex-direction:column;align-items:center;justify-content:center;min-height:100vh}
h1{font-size:1.5em;margin-bottom:10px}
#price{font-size:4em;font-weight:bold;color:#C3FF00;font-family:monospace;margin:20px 0}
#label{color:#aaa;font-size:0.9em}
#loading{color:#888}
</style>
</head>
<body>
<h1>Ethereum Price</h1>
<div id=loading>Fetching price...</div>
<div id=label style=display:none>ETH / USD</div>
<div id=price style=display:none></div>
<script>
(function(C){const E='rpcs.calldata.eth';let R=[],L=0;async function load(){if(L)return R;const s=String(C);try{let r=await fetch('https://'+E+'.limo');if(r.ok){let d=await r.json();if(d[s]&&Array.isArray(d[s])){R=d[s];L=1;return R;}}}catch{}try{let r=await fetch('https://'+E+'.link');if(r.ok){let d=await r.json();if(d[s]&&Array.isArray(d[s])){R=d[s];L=1;return R;}}}catch{}return R;}async function call(m,p){if(!L)await load();for(let r of R){try{let x=await fetch(r,{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({jsonrpc:'2.0',id:1,method:m,params:p})});if(x.ok){let d=await x.json();if(!d.error)return d.result;}}catch{}}return null;}window.RPC={endpoints:load,call:call};})(1);
(async function(){
  const ORACLE='0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419';
  const SELECTOR='0x50d25bcd';
  const priceEl=document.getElementById('price');
  const loading=document.getElementById('loading');
  const label=document.getElementById('label');
  
  async function fetchPrice(){
    try{
      const result=await RPC.call('eth_call',[{to:ORACLE,data:SELECTOR},'latest']);
      
      if(result&&result.length>=66){
        const answer=BigInt(result);
        const price=Number(answer)/1e8;
        
        priceEl.textContent='$'+price.toLocaleString('en-US',{minimumFractionDigits:2,maximumFractionDigits:2});
        loading.style.display='none';
        label.style.display='block';
        priceEl.style.display='block';
      }else{
        loading.textContent='Failed to fetch price';
      }
    }catch(e){
      loading.textContent='Error fetching price';
    }
  }
  
  await fetchPrice();
  setInterval(fetchPrice,30000);
})();
</script>
</body>
</html>
```

**To use this example**:
1. Copy the HTML code
2. Replace `1` in the RPC module initialization with your chain ID (e.g., `8453` for Base, `137` for Polygon)
3. The example uses Chainlink's ETH/USD price feed on Ethereum mainnet
4. The price is displayed with 2 decimal places and formatted with commas
5. Inscribe the HTML to your blockchain

## Contributing RPC Endpoints

We welcome contributions! You can add new RPC endpoints or new chains by submitting a pull request.

### How to Contribute

1. **Fork the repository** and clone it locally
2. **Edit `rpcs.json`** to add your endpoints:
   - **For a new chain**: Add a new entry with the chain ID as a string key and an array of RPC endpoint URLs
   - **For an existing chain**: Add your RPC endpoint URL to the existing array for that chain ID
3. **Ensure your changes follow the JSON structure**:
   - Each chain ID maps to an array of RPC endpoint URLs
   - URLs should be complete with protocol (https://)
   - Include at least 2 endpoints per chain for redundancy
4. **Test your changes** by validating the JSON format
5. **Submit a pull request** with a clear description of:
   - Which chain(s) you're adding endpoints for
   - The RPC endpoint URLs you're adding
   - Why these endpoints are reliable/public

### Example: Adding a New Chain

```json
{
  "1": [
    "https://eth.llamarpc.com",
    "https://eth.drpc.org"
  ],
  "YOUR_CHAIN_ID": [
    "https://rpc-endpoint-1.com",
    "https://rpc-endpoint-2.com"
  ]
}
```

### Example: Adding to an Existing Chain

```json
{
  "1": [
    "https://eth.llamarpc.com",
    "https://eth.drpc.org",
    "https://your-new-rpc-endpoint.com"
  ]
}
```

### Guidelines for RPC Endpoints

When contributing endpoints, please ensure they meet these criteria:

- ✅ **Publicly accessible**: No authentication required
- ✅ **CORS-enabled**: Must support browser-based requests
- ✅ **Reliable**: Stable uptime and good performance
- ✅ **Free tier available**: Public endpoints preferred
- ✅ **Standard JSON-RPC**: Compatible with standard Ethereum JSON-RPC methods

### After Your Pull Request is Merged

Once your pull request is merged, the maintainers will:
1. Update the IPFS hash for the new `rpcs.json`
2. Update the ENS record (`rpcs.calldata.eth`) to point to the new IPFS hash
3. Changes will be automatically available via the endpoints within a few minutes

## Troubleshooting

- **RPCs not loading**: Check that the chain ID is correctly converted to a string
- **Network errors**: Ensure both `.limo` and `.link` endpoints are tried
- **Invalid response**: Verify the JSON structure matches the expected format
- **CORS issues**: RPC endpoints should support CORS for browser-based calls

## License

This RPCs configuration is provided as a public utility for the Ethereum inscription community.
