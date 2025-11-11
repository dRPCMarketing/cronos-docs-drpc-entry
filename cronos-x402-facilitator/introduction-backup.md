---
hidden: true
---

# Introduction-backup

## **What is the Cronos X402 Facilitator?**

The Cronos x402 Facilitator is a third-party service that enables sellers to verify and settle blockchain payments without running blockchain infrastructure. Acting as the core engine for x402 transactions on Cronos, it manages verification, execution, and settlement of gas-efficient stablecoin payments.

Built on the [x402 protocol specification](https://github.com/coinbase/x402), the facilitator provides a stateless, production-ready API that connects sellers, APIs, and AI agents to on-chain commerce. It uses EIP-3009 (`transferWithAuthorization`) to enable gasless token transfers. The facilitator submits transactions and pays gas fees on behalf of authorized payers. See the [Facilitator API Reference](introduction.md#api-reference) for endpoint details.

## Why Sellers Choose the Facilitator <a href="#why-sellers-choose-the-facilitator" id="why-sellers-choose-the-facilitator"></a>

* **Eliminate Technical Complexity:** No need for blockchain expertise. Integrate payments using familiar HTTP APIs instead of complex wallet management, transaction signing, and gas optimization.
* **Zero Infrastructure Overhead:** No nodes to run, no transaction monitoring, no gas price management. The facilitator provides battle-tested, managed infrastructure.
* **Security Without Expertise:** Cryptographic best practices, EIP-3009 signature verification, and replay protection are handled correctly out of the box.
* **Gasless Payments for Users:** Through EIP-3009, payers send stablecoin payments without holding native tokens for gas fees.
* **Predictable Economics:** Cronos offers extremely low transaction fees. Combined with the facilitator's efficient architecture, this enables micropayments and high-volume use cases.
* **Production-Ready Reliability:** Stateless design with no database, comprehensive error handling, rate limiting, and idempotent operations ensure safe retries.
* **Standards-Based:** Implements x402 protocol v1 for compatibility with emerging ecosystem of x402-enabled wallets, AI agents, and tooling.

## How It Works

here is the high-level workflow diagram:

<figure><img src="../.gitbook/assets/paymentflow.png" alt=""><figcaption></figcaption></figure>

<details>

<summary>Note: unfold to see the sequence diagram code here</summary>

```javascript
sequenceDiagram
    participant PA as Buyer<br/>(Wallet/AI)
    participant RA as Seller<br/>(Resource Server)
    participant FS as Facilitator<br/>Service
    participant CB as Cronos<br/>Blockchain
    
    Note over PA,CB: 1. Request Resource
    PA->>RA: GET /api/premium-data
    RA->>PA: 402 Payment Required<br/>{paymentRequirements}
    
    Note over PA,CB: 2. Sign Authorization
    PA->>PA: Generate nonce<br/>Sign EIP-712 typed data<br/>Create payment header
    
    Note over PA,CB: 3. Retry with Payment
    PA->>RA: GET /api/premium-data<br/>X-PAYMENT
    
    Note over PA,CB: 4. Verify Payment
    RA->>FS: POST /verify<br/>{paymentHeader, paymentRequirements}
    FS->>FS: Decode Base64<br/>Verify EIP-3009 signature<br/>Check amount/network
    FS->>RA: {isValid: true}
    
    Note over PA,CB: 5. Settle On-Chain
    RA->>FS: POST /settle<br/>{paymentHeader, paymentRequirements}
    FS->>CB: transferWithAuthorization()<br/>{from, to, value, nonce, signature}
    CB->>CB: Validate signature<br/>Check nonce uniqueness<br/>Transfer USDX tokens
    FS->>FS: Wait for blockchain confirmation
    CB->>FS: Transaction receipt<br/>{txHash, blockNumber}
    FS->>RA: {event: "payment.settled", txHash, ...}
    
    Note over PA,CB: 6. Deliver Content
    RA->>PA: 200 OK<br/>{data, payment: {txHash, value, ...}}
```

</details>

The x402 payment flow involves four components: **Buyer** (wallet/AI agent), **Seller** (your API/resource server), **Facilitator** (this service), and **Cronos Blockchain**. For implementation guides, see [Quick Start for Buyers](quick-start-for-buyers.md) and [Quick Start for Sellers](quick-start-for-sellers.md).

### Payment Flow

**1. Request Resource**\
The buyer attempts to access a payment-gated resource:

```http
GET /api/premium-data HTTP/1.1
Host: seller-api.example.com
```

The seller's server responds with an HTTP 402 Payment Required status code and payment requirements. The x402 protocol extends the HTTP 402 status code to enable machine-readable payment flows.

```http
HTTP/1.1 402 Payment Required
Content-Type: application/json
```

Response body containing payment requirements:

```json
{
  "error": "Payment Required",
  "x402Version": 1,
  "paymentRequirements": {
    "scheme": "exact",
    "network": "cronos-testnet", // or "cronos" for Cronos Mainnet
    "payTo": "0xSeller...",
    "asset": "0xUSDX...",
    "maxAmountRequired": "1000000",
    "maxTimeoutSeconds": 300
  }
}
```

**2. Sign Authorization**

The buyer's wallet signs an EIP-3009 authorization using EIP-712 typed data, creating a signature that allows the facilitator to transfer tokens on their behalf without needing gas. The signature includes a unique nonce to prevent replay attacks. This authorization is encoded as a Base64 payment header. See [implementation details](https://file+.vscode-resource.vscode-cdn.net/Users/danielluo/Downloads/x402-archive/facilitator_docs.md#generating-payment-header) for the complete signing flow.

**3. Retry with Payment**

The buyer retries the request with the signed payment header:

```http
GET /api/premium-data HTTP/1.1
Host: seller-api.example.com
X-PAYMENT: eyJ4NDAyVmVyc2lvbiI6MSwic2NoZW1lIjoiZXhhY3QiLC4uLn0=
```

**4. Verify Payment**

The seller forwards the payment header to the facilitator's `/verify` endpoint. The facilitator validates the header structure, decodes Base64, verifies the EIP-3009 signature cryptographically, and checks network/asset/amount requirements, all without an on-chain transaction. For more information, see the [Verify Endpoint](introduction-backup.md#verify-endpoint) documentation.

**5. Settle On-Chain**

Once verified, the seller requests settlement by calling the facilitator's [Settle Endpoint](introduction-backup.md#settle-endpoint). The facilitator submits a `transferWithAuthorization` transaction to the Cronos blockchain, paying gas fees on behalf of the buyer. The USDX smart contract validates the signature, checks nonce uniqueness to prevent replay attacks, and transfers tokens from the buyer to the seller. The facilitator waits for blockchain confirmation before returning the transaction receipt to the seller.

**6. Deliver Content**

With payment confirmed on-chain, the seller delivers the protected resource along with transaction details (txHash, blockNumber, timestamp) to the buyer.

## API Reference

### Network Constants

| Constant            | Testnet Value                                | Mainnet Value            |
| ------------------- | -------------------------------------------- | ------------------------ |
| **Network String**  | `cronos-testnet`                             | `cronos`                 |
| **Chain ID**        | `338`                                        | `25`                     |
| **RPC URL**         | `https://evm-t3.cronos.org`                  | `https://evm.cronos.org` |
| **USDX Contract**   | `0x149a72BCdFF5513F2866e9b6394edba2884dbA07` | Coming soon              |
| **Facilitator URL** | `https://facilitator.cronoslabs.org/v2/x402` | Same (single endpoint)   |

{% hint style="info" %}
**Note**

Use these contract addresses in the `asset` field of your payment requirements. Always verify contract addresses match your target network.
{% endhint %}

{% hint style="info" %}
**Note**

The facilitator uses one base URL for all networks. Switch between mainnet and testnet by setting the `network` field in your payment requirements (`"cronos-testnet"` or `"cronos"`). See [API Endpoints ](introduction-backup.md#api-endpoints)below.
{% endhint %}

### API Endpoints

<table data-full-width="false"><thead><tr><th width="162.51953125">Endpoint</th><th width="116.76953125">Method</th><th>URL</th></tr></thead><tbody><tr><td><strong>Health Check</strong></td><td>GET</td><td><code>https://facilitator.cronoslabs.org/healthcheck</code></td></tr><tr><td><strong>Discovery</strong></td><td>GET</td><td><code>https://facilitator.cronoslabs.org/v2/x402/supported</code></td></tr><tr><td><strong>Verify</strong></td><td>POST</td><td><code>https://facilitator.cronoslabs.org/v2/x402/verify</code></td></tr><tr><td><strong>Settle</strong></td><td>POST</td><td><code>https://facilitator.cronoslabs.org/v2/x402/settle</code></td></tr></tbody></table>

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

```json
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



## Quick Start for Buyers <a href="#quick-start-for-buyers" id="quick-start-for-buyers"></a>

This section explains how buyers (wallets, AI agents, or applications) can generate valid payment headers that authorize token transfers and authenticate payment when accessing protected resources. For the seller-side implementation, see [Quick Start for Sellers](introduction.md#quick-start-for-sellers).

### Prerequisites <a href="#prerequisites" id="prerequisites"></a>

* Cronos-compatible wallet with private key or browser wallet extension
* USDX token balance (see [Network Constants](introduction.md#network-constants) for contract addresses)

### Integration Steps <a href="#integration-steps" id="integration-steps"></a>

1. **Receive 402 Response** - Buyer requests resource and receives payment requirements
2. **Generate Payment Header** - Create EIP-3009 signature authorizing token transfer
3. **Retry with Payment** - Send request with `X-PAYMENT` containing authorization
4. **Receive Content** - Seller settles payment and returns protected resource



**Installation**

```bash
npm install ethers
```

**Complete Payment Flow**

```javascript
const axios = require('axios');
const { ethers } = require('ethers');
const { createPaymentHeader } = require('./payment-header-generator');

async function payForResource(resourceUrl, wallet) {
  try {
    // Step 1: Initial request to resource (expect 402)
    const initialResponse = await axios.get(resourceUrl).catch(err => err.response);

    // Step 2: Check for 402 Payment Required
    if (initialResponse.status !== 402) {
      return initialResponse.data;
    }

    const { paymentRequirements } = initialResponse.data;

    // Step 3: Generate payment header (sign EIP-3009 authorization)
    const paymentHeaderBase64 = await createPaymentHeader({
      wallet: wallet,
      paymentRequirements: paymentRequirements,
      network: paymentRequirements.network,
    });

    // Step 4: Retry request with payment header
    const paidResponse = await axios.get(resourceUrl, {
      headers: { 'X-PAYMENT': paymentHeaderBase64 },
    });
    
    return paidResponse.data;

  } catch (error) {
    throw error;
  }
}

// Example usage
async function main() {
  const privateKey = process.env.PRIVATE_KEY;
  const provider = new ethers.JsonRpcProvider('https://evm-t3.cronos.org');
  const wallet = new ethers.Wallet(privateKey, provider);

  const resourceUrl = 'https://seller-api.example.com/api/premium-data';
  const result = await payForResource(resourceUrl, wallet);

  return result;
}

main();
```

**Generating Payment Header**

```javascript
const { ethers } = require('ethers');

/**
 * Generates a random 32-byte nonce for EIP-3009 authorization
 */
function generateNonce() {
  return ethers.hexlify(ethers.randomBytes(32));
}

/**
 * Creates a signed payment header for x402 payments
 */
async function createPaymentHeader({ wallet, paymentRequirements, network }) {
  const { payTo, asset, maxAmountRequired, maxTimeoutSeconds, scheme } = paymentRequirements;

  // Generate unique nonce
  const nonce = generateNonce();
  
  // Calculate validity window
  const validAfter = 0; // Valid immediately
  const validBefore = Math.floor(Date.now() / 1000) + maxTimeoutSeconds;

  // Set up EIP-712 domain
  const domain = {
    name: "USDX Coin",
    version: "1",
    chainId: "338",
    verifyingContract: asset,
  };

  // Define EIP-712 typed data structure
  const types = {
    TransferWithAuthorization: [
      { name: 'from', type: 'address' },
      { name: 'to', type: 'address' },
      { name: 'value', type: 'uint256' },
      { name: 'validAfter', type: 'uint256' },
      { name: 'validBefore', type: 'uint256' },
      { name: 'nonce', type: 'bytes32' },
    ],
  };

  // Create the message to sign
  const message = {
    from: wallet.address,
    to: payTo,
    value: maxAmountRequired,
    validAfter: validAfter,
    validBefore: validBefore,
    nonce: nonce,
  };

  // Sign using EIP-712
  const signature = await wallet.signTypedData(domain, types, message);

  // Construct payment header
  const paymentHeader = {
    x402Version: 1,
    scheme: scheme,
    network: network,
    payload: {
      from: wallet.address,
      to: payTo,
      value: maxAmountRequired,
      validAfter: validAfter,
      validBefore: validBefore,
      nonce: nonce,
      signature: signature,
      asset: asset,
    },
  };

  // Base64-encode
  return Buffer.from(JSON.stringify(paymentHeader)).toString('base64');
}

module.exports = { createPaymentHeader, generateNonce, NETWORKS, TOKEN_NAME, TOKEN_VERSION };
```

**Common Pitfalls**

Common mistakes when implementing the payment header generation:

**Wrong Token Metadata**

```javascript
// WRONG - will fail signature verification
const domain = { name: 'USDX', version: '2' };

// CORRECT - must be exact
const domain = { name: 'USDX Coin', version: '1' };
```

**Invalid Nonce Format**

```javascript
// WRONG
const nonce = ethers.hexlify(ethers.randomBytes(16)); // Too short
const nonce = '1234567890'; // Not hex

// CORRECT
const nonce = ethers.hexlify(ethers.randomBytes(32)); // 32 bytes
```

**Decimal in Amount**

```javascript
// WRONG
const value = '1.5'; // Has decimal
const value = 1500000; // Number type (precision loss)

// CORRECT
const value = '1500000'; // Base units as string
```

**Timestamp in Milliseconds**

```javascript
// WRONG
const validBefore = Date.now() + 300000; // Milliseconds

// CORRECT
const validBefore = Math.floor(Date.now() / 1000) + 300; // Seconds
```



## Quick Start for Sellers

This section explains how sellers (API providers, resource servers) can integrate x402 payment verification and settlement into their services. For the buyer-side implementation, see [Quick Start for Buyers](quick-start-for-buyers.md).

### Prerequisites

* Cronos-compatible wallet for receiving payments

### Integration Steps

1. **Detect Payment Requirement:** Check if the requested resource requires payment
2. **Respond with 402:** Return payment requirements to the buyer
3. **Extract Payment Header:** Buyer retries with signed authorization in `X-PAYMENT`
4. **Verify Payment:** Forward to facilitator's [Verify Endpoint](introduction-backup.md#verify-endpoint)
5. **Settle Payment:** If valid, call the [Settle Endpoint](introduction-backup.md#settle-endpoint)&#x20;
6. **Deliver Resource:** Return protected content with 200 OK

### Complete Payment Flow

```javascript
const express = require('express');
const axios = require('axios');
const app = express();
app.use(express.json());

// Configuration
const FACILITATOR_URL = 'https://facilitator.cronoslabs.org/v2/x402';
const SELLER_WALLET = process.env.SELLER_WALLET;
const USDX_CONTRACT = '0x149a72BCdFF5513F2866e9b6394edba2884dbA07'; // Cronos testnet - see Network Constants

// Protected API endpoint
app.get('/api/premium-data', async (req, res) => {
  const paymentHeader = req.headers['X-PAYMENT'] || req.body?.paymentHeader;

  // Step 1: Check if payment is provided
  if (!paymentHeader) {
    return res.status(402).json({
      error: 'Payment Required',
      x402Version: 1,
      paymentRequirements: {
        scheme: 'exact',
        network: 'cronos-testnet', // Switch to 'cronos' for Cronos mainnet
        payTo: SELLER_WALLET,
        asset: USDX_CONTRACT,
        description: 'Premium API data access',
        mimeType: 'application/json',
        maxAmountRequired: '1000000', // 1 USDX (6 decimals)
        maxTimeoutSeconds: 300
      }
    });
  }

  try {
    const requestBody = {
      x402Version: 1,
      paymentHeader: paymentHeader,
      paymentRequirements: { 
        scheme: 'exact',
        network: 'cronos-testnet', // Same network as in 402 response
        payTo: SELLER_WALLET,
        asset: USDX_CONTRACT,
        description: 'Premium API data access',
        mimeType: 'application/json',
        maxAmountRequired: '1000000',
        maxTimeoutSeconds: 300
      }
    };

    // Step 2: Verify payment
    const verifyRes = await axios.post(`${FACILITATOR_URL}/verify`, requestBody, {
      headers: { 'Content-Type': 'application/json', 'X402-Version': '1' }
    });

    if (!verifyRes.data.isValid) {
      return res.status(402).json({
        error: 'Invalid payment',
        reason: verifyRes.data.invalidReason
      });
    }

    // Step 3: Settle payment
    const settleRes = await axios.post(`${FACILITATOR_URL}/settle`, requestBody, {
      headers: { 'Content-Type': 'application/json', 'X402-Version': '1' }
    });

    // Step 4: Check settlement and return content
    if (settleRes.data.event === 'payment.settled') {
      return res.status(200).json({
        data: {
          premiumContent: 'This is your premium data',
        },
        payment: {
          txHash: settleRes.data.txHash,
          from: settleRes.data.from,
          to: settleRes.data.to,
          value: settleRes.data.value,
          blockNumber: settleRes.data.blockNumber,
          timestamp: settleRes.data.timestamp
        }
      });
    } else {
      return res.status(402).json({
        error: 'Payment settlement failed',
        reason: settleRes.data.error
      });
    }
  } catch (error) {
    return res.status(500).json({
      error: 'Server error processing payment',
      details: error.response?.data || error.message
    });
  }
});

app.listen(3000);
```

> **Note:** For Python, Go, or other language examples, the integration pattern is identical: make HTTP POST requests to `/verify` and `/settle` with the same JSON structure.

### Testing Your Integration

**1. Test Environment**

The facilitator uses a single base URL for both networks:

* **Facilitator URL**: `https://facilitator.cronoslabs.org/v2/x402`
* **Network Selection**: Specify via `"network"` field in payment requirements
  * Testnet: `"network": "cronos-testnet"` (Chain ID: `338`)
  * Mainnet: `"network": "cronos"` (Chain ID: `25`)

**2. Test Health Check**

See the [Health Check Endpoint](introduction-backup.md#health-check-endpoint) for details.

```bash
curl -X GET https://facilitator.cronoslabs.org/healthcheck
```

**3. Test Discovery**

See the [Discovery Endpoint](introduction-backup.md#discovery-endpoint) for details.

```bash
curl -X GET https://facilitator.cronoslabs.org/v2/x402/supported
```

**4. Simulate Payment Flow**

```bash
# Request without payment (should return 402)
curl -X GET http://localhost:3000/api/premium-data

# Request with payment header (after signing)
curl -X GET http://localhost:3000/api/premium-data \
  -H "X-PAYMENT: eyJ4NDAyVmVyc2lvbiI6MS4uLn0="
```

**5. Test Edge Cases**

See [Verify Endpoint](introduction-backup.md#verify-endpoint) and [Settle Endpoint](introduction-backup.md#settle-endpoint) for complete error documentation.

| Scenario          | Expected Response                                                              |
| ----------------- | ------------------------------------------------------------------------------ |
| Invalid signature | `{"isValid": false, "invalidReason": "Invalid EIP-3009 signature"}`            |
| Wrong network     | `{"isValid": false, "invalidReason": "Unsupported network: ethereum-mainnet"}` |
| Duplicate nonce   | `{"event": "payment.failed", "error": "Authorization already used"}`           |
| Expired auth      | `{"event": "payment.failed", "error": "Authorization expired"}`                |

**6. Monitor Transactions**

* Check facilitator response for `txHash` and `blockNumber`
* View on Cronos Block Explorer: `https://explorer.cronos.org/testnet/tx/[txHash]`

***

## Resources & Next Steps

The Cronos x402 Facilitator enables sellers to accept on-chain stablecoin payments through a simple HTTP API, eliminating blockchain infrastructure complexity while leveraging Cronos' low fees and high throughput.

### Useful Links

**Facilitator Service:**

* **Base URL**: [https://facilitator.cronoslabs.org](https://facilitator.cronoslabs.org/)
* **Health Check**: [https://facilitator.cronoslabs.org/healthcheck](https://facilitator.cronoslabs.org/healthcheck)
* **API Endpoints**: [https://facilitator.cronoslabs.org/v2/x402/](https://facilitator.cronoslabs.org/v2/x402/)

**Documentation & Resources:**

* **Developer Dashboard**: [https://developer.crypto.com](https://developer.crypto.com/)
* **API Documentation**: TBD
* **Cronos Block Explorer**: [https://explorer.cronos.org](https://explorer.cronos.org/)
* **GitHub Repository**: [https://github.com/crypto-com/x402-facilitator-service](https://github.com/crypto-com/x402-facilitator-service)
* **x402 Protocol Spec**: [https://github.com/coinbase/x402](https://github.com/coinbase/x402)
* **Support**: TBD

