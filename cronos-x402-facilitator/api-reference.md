# API Reference

## Network Constants

<table><thead><tr><th width="165.54296875">Constant</th><th width="382.046875">Testnet Value</th><th>Mainnet Value</th></tr></thead><tbody><tr><td><strong>Network String</strong></td><td><code>cronos-testnet</code></td><td><code>cronos</code></td></tr><tr><td><strong>Chain ID</strong></td><td><code>338</code></td><td><code>25</code></td></tr><tr><td><strong>RPC URL</strong></td><td><code>https://evm-t3.cronos.org</code></td><td><code>https://evm.cronos.org</code></td></tr><tr><td><strong>USDX Contract</strong></td><td><code>0x149a72BCdFF5513F2866e9b6394edba2884dbA07</code></td><td>Coming soon</td></tr><tr><td><strong>Facilitator URL</strong></td><td><code>https://facilitator.cronoslabs.org/v2/x402</code></td><td>Same (single endpoint)</td></tr></tbody></table>

{% hint style="info" %}
**Note**: Use these contract addresses in the `asset` field of your payment requirements. Always verify contract addresses match your target network.
{% endhint %}

{% hint style="info" %}
**Note**: The facilitator uses one base URL for all networks. Switch between mainnet and testnet by setting the `network` field in your payment requirements (`"cronos-testnet"` or `"cronos"`). See [API Endpoints](api-reference.md#api-endpoints) below.
{% endhint %}

## API Endpoints

<table data-full-width="false"><thead><tr><th width="160.40625">Endpoint</th><th width="113.14453125">Method</th><th>URL</th></tr></thead><tbody><tr><td><strong>Health Check</strong></td><td>GET</td><td><code>https://facilitator.cronoslabs.org/healthcheck</code></td></tr><tr><td><strong>Discovery</strong></td><td>GET</td><td><code>https://facilitator.cronoslabs.org/v2/x402/supported</code></td></tr><tr><td><strong>Verify</strong></td><td>POST</td><td><code>https://facilitator.cronoslabs.org/v2/x402/verify</code></td></tr><tr><td><strong>Settle</strong></td><td>POST</td><td><code>https://facilitator.cronoslabs.org/v2/x402/settle</code></td></tr></tbody></table>

### Health Check Endpoint

**GET** `/healthcheck`

Returns the health status of the facilitator service, including uptime, response time, and timestamp. No authentication required.

**Response:**

```json
{
    "status": "success",
    "results": {
        "uptime": 167592.074548689,
        "responseTime": [
            13401697,
            473212559
        ],
        "message": "OK",
        "timestamp": 1762490327834
    }
}
```

### Discovery Endpoint <a href="#discovery-endpoint" id="discovery-endpoint"></a>

**GET** `/v2/x402/supported`

Retrieves the supported payment kinds from the facilitator service, including available networks, schemes, and X402 protocol versions. No authentication required.

```json
{
  "kinds": [
    {
      "x402Version": 1,
      "scheme": "exact",
      "network": "cronos-testnet"
    }
  ]
}
```

### Verify Endpoint

**POST** `/v2/x402/verify`

Validates a payment header and requirements without executing an on-chain transaction. Use this to verify payment validity before committing to settlement.

**Headers:**

```
Content-Type: application/json
X402-Version: 1
```

**Request Body:**

```json
{
  "x402Version": 1,
  "paymentHeader": "eyJ4NDAyVmVyc2lvbiI6MS4uLn0=",
  "paymentRequirements": {
    "scheme": "exact",
    "network": "cronos-testnet",
    "payTo": "0xSeller...",
    "asset": "0xUSDX...",
    "maxAmountRequired": "1000000",
    "maxTimeoutSeconds": 300
  }
}
```

**Payment Header (Base64 decoded structure):**

```json
{
  "x402Version": 1,
  "scheme": "exact",
  "network": "cronos-testnet",
  "payload": {
    "from": "0xFrom...",
    "to": "0xPayTo...",
    "value": "1000000",
    "validAfter": 0,
    "validBefore": 1735689551,
    "nonce": "0xNonce...",
    "signature": "0xSignature...",
    "asset": "0xUSDX..."
  }
}
```

**Success Response:**

```json
{
  "isValid": true,
  "invalidReason": null
}
```

**Failure Response:**

```json
{
  "isValid": false,
  "invalidReason": "Unsupported network: ethereum-mainnet"
}
```

**What This Endpoint Does:**

* Decodes Base64-encoded payment headers
* Verifies scheme matches supported types (`exact`)
* Confirms network is supported (`cronos-testnet`)
* Validates asset/token contracts and amounts
* Verifies EIP-3009 signatures using EIP-712 typed data
* Checks `validBefore` and `validAfter` timestamp windows
* Rate limited to 10 requests per minute per IP

**Common Invalid Reasons:**

* `Unsupported scheme: [scheme]`
* `Unsupported network: [network]`
* `Unsupported asset: [asset]`
* `Recipient does not match payTo address`
* `Invalid amount: [amount]`
* `Invalid EIP-3009 signature`
* `Invalid Base64 paymentHeader`

### Settle Endpoint <a href="#settle-endpoint" id="settle-endpoint"></a>

**POST** `/v2/x402/settle`

Executes a verified payment on-chain by submitting an EIP-3009 `transferWithAuthorization` transaction.

**Headers:**

```
Content-Type: application/json
X402-Version: 1
```

**Request Body:**

Same format as the [Verify Endpoint](introduction.md#verify-endpoint).

**Success Response:**

```json
{
  "x402Version": 1,
  "event": "payment.settled",
  "txHash": "0xTxHash...",
  "from": "0xFrom...",
  "to": "0xPayTo...",
  "value": "1000000",
  "blockNumber": 17352,
  "network": "cronos-testnet",
  "timestamp": "2025-11-04T20:19:11.000Z"
}
```

**Failure Response:**

```json
{
  "x402Version": 1,
  "event": "payment.failed",
  "network": "cronos-testnet",
  "timestamp": "2025-11-04T20:19:11.000Z",
  "error": "Authorization already used"
}
```

**What This Endpoint Does:**

* Calls `transferWithAuthorization` on USDX Token
* Waits for transaction confirmation on-chain (facilitator blocks until settled)
* Uses unique nonces to prevent duplicate transactions
* Returns transaction hash, block number, and timestamp
* Rate limited to 5 requests per minute per IP

**Common Errors:**

* `Authorization already used` (duplicate nonce - transaction already executed)
* `Authorization expired` (validBefore timestamp passed)
* `Authorization not yet valid` (validAfter not reached)
* `ERC20: transfer amount exceeds balance` (insufficient balance)
* `Invalid signature` (signature verification failed off-chain or on-chain)
* `Transaction failed on-chain` (generic failure)
