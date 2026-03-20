# Hive Payments — QR Codes, Keychain & Payment Verification

## HBD/HIVE Transfers

Hive supports trustless payments using HBD (Hive-Backed Dollars) or HIVE tokens. Two primary methods for triggering payments from a web app:

1. **Hive Keychain browser extension** — `requestTransfer()` for desktop users
2. **QR code scanning** — `hive://sign/op/` URI for mobile Keychain app

Both result in a standard `transfer` operation on the Hive blockchain.

---

## Method 1: Hive Keychain Browser Extension

The Keychain browser extension injects `window.hive_keychain` into the page.

### requestTransfer

```javascript
function payWithKeychain(recipient, amount, memo, currency = 'HBD') {
  if (!window.hive_keychain) {
    alert('Hive Keychain extension not found');
    return;
  }

  // CRITICAL: amount must be a STRING with exactly 3 decimal places, NO currency suffix
  // "1.000" ✅  |  "1.0" ❌  |  "1.000 HBD" ❌  |  1.0 ❌
  const amountStr = parseFloat(amount).toFixed(3);

  window.hive_keychain.requestTransfer(
    null,        // from: null = active Keychain user
    recipient,   // to: receiving Hive account
    amountStr,   // amount: "1.000" (string, 3 decimals, NO currency)
    memo,        // memo: any string
    currency,    // currency: 'HBD' or 'HIVE' (separate param!)
    (response) => {
      if (response.success) {
        console.log('Transfer broadcast:', response.result);
        // response.result contains the transaction ID
      } else {
        console.error('Transfer failed:', response.message);
      }
    }
  );
}
```

### Common Mistakes

| Mistake | Error | Fix |
|---------|-------|-----|
| `"1.000 HBD"` as amount | "Amount requires a string with 3 decimals" | Remove currency suffix — it's the 5th parameter |
| `1.0` (number) | Same error | Must be a string: `"1.000"` |
| `"1.0"` (wrong decimals) | Same error | Always use `.toFixed(3)` |
| Passing username as 1st arg | Wrong user prompted | Use `null` to let Keychain pick active user |

---

## Method 2: QR Code for Mobile Keychain

The Hive Keychain mobile app scans QR codes containing `hive://sign/op/` URIs.

### URI Format

**CRITICAL:** The URI is NOT a simple query string. It encodes the full operation as URL-safe base64:

```
hive://sign/op/{url-safe-base64-encoded-operation}
```

### Building the URI

```javascript
function buildHivePaymentURI(recipient, amount, memo) {
  // 1. Build the transfer operation array
  const op = [
    "transfer",
    {
      to: recipient,
      amount: parseFloat(amount).toFixed(3) + ' HBD',  // HERE amount includes currency
      memo: memo
    }
  ];

  // 2. JSON stringify
  const json = JSON.stringify(op);

  // 3. Base64 encode with URL-safe characters
  const base64 = btoa(json)
    .replace(/\+/g, '-')    // + → -
    .replace(/\//g, '_')    // / → _
    .replace(/=+$/, '');    // strip trailing =

  // 4. Build URI
  return `hive://sign/op/${base64}`;
}
```

### WRONG vs RIGHT URI format

```
WRONG: hive://sign/transfer?to=shop&amount=1.000%20HBD&memo=test
       ↑ Keychain will NOT recognize this format

RIGHT: hive://sign/op/WyJ0cmFuc2ZlciIseyJ0byI6InNob3AiLCJhbW91bnQiOiIxLjAwMCBIQkQiLCJtZW1vIjoidGVzdCJ9XQ
       ↑ Base64-encoded operation — Keychain scans and parses this
```

### Important difference from requestTransfer

| | requestTransfer (browser) | QR URI (mobile) |
|---|---|---|
| Amount format | `"1.000"` (no currency) | `"1.000 HBD"` (WITH currency) |
| Currency | Separate 5th parameter | Part of amount string |
| From user | `null` (auto) or specified | Determined by scanner |

---

## QR Code Generation

Use **qrcodejs** (not the `qrcode` npm package) — it's simpler and works reliably in all environments including Shadow DOM.

### CDN

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js"></script>
```

### Usage

```javascript
// Clear previous QR code first
const container = document.getElementById('qrBox');
container.innerHTML = '';

new QRCode(container, {
  text: uri,                         // the hive://sign/op/... URI
  width: 240,
  height: 240,
  colorDark: '#000000',
  colorLight: '#ffffff',
  correctLevel: QRCode.CorrectLevel.H  // highest error correction
});
```

### Styling the QR Code

Always wrap in a white padded container (quiet zone required for scanning):

```css
.qr-box {
  background: #fff;
  padding: 16px;
  border-radius: 12px;
  box-shadow: 0 4px 18px rgba(0,0,0,0.15);
  display: inline-block;
}

/* Crisp rendering */
.qr-box canvas {
  image-rendering: pixelated;
  display: block;
}
```

### Shadow DOM (Lit/Web Components)

qrcodejs works inside Shadow DOM — pass a real DOM element from `this.shadowRoot`:

```javascript
async _payQR() {
  this._state = 'awaiting-payment';
  await this.updateComplete; // wait for Lit to render the container

  const uri = this._buildHiveUri();
  const qrBox = this.shadowRoot.querySelector('#qr-box');
  if (qrBox && typeof QRCode !== 'undefined') {
    qrBox.innerHTML = '';
    new QRCode(qrBox, {
      text: uri, width: 240, height: 240,
      colorDark: '#000000', colorLight: '#ffffff',
      correctLevel: QRCode.CorrectLevel.H
    });
  }
}
```

### Why NOT the `qrcode` npm package

The `qrcode` npm package (`QRCode.toCanvas()` / `QRCode.toDataURL()`) has issues:
- `toCanvas` fails inside Shadow DOM — can't find the canvas after Lit re-renders
- `toDataURL` is async and the data URL can be lost on re-renders
- The qrcodejs constructor approach is synchronous and renders directly into the DOM

---

## Payment Verification (Polling)

After showing the QR code, poll the recipient's transaction history to detect the incoming payment.

### Polling Pattern

```javascript
async function pollForPayment(recipient, expectedAmount, expectedMemo, onSuccess, onTimeout) {
  const HIVE_APIS = [
    'https://api.hive.blog',
    'https://api.openhive.network',
    'https://anyx.io'
  ];

  for (let attempt = 0; attempt < 100; attempt++) {  // 100 × 3s = 5 min max
    for (const api of HIVE_APIS) {
      try {
        const resp = await fetch(api, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            jsonrpc: '2.0',
            method: 'condenser_api.get_account_history',
            params: [recipient, -1, 20],  // last 20 operations
            id: 1
          })
        });
        const data = await resp.json();
        if (data.result) {
          const found = data.result.find(tx => {
            const op = tx[1]?.op;
            if (!op || op[0] !== 'transfer') return false;
            // Match recipient, amount (with currency), and memo
            return op[1].to === recipient
              && op[1].amount === expectedAmount  // "1.000 HBD"
              && op[1].memo === expectedMemo;
          });
          if (found) {
            onSuccess(found);
            return;
          }
        }
        break; // one API responded, skip fallbacks
      } catch {
        continue; // try next API node
      }
    }
    await new Promise(r => setTimeout(r, 3000)); // wait 3 seconds
  }

  onTimeout();
}
```

### Key Details

- **`get_account_history` params:** `[account, -1, 20]` — `-1` means most recent, `20` is how many ops to fetch
- **Amount comparison:** On-chain amount includes currency: `"1.000 HBD"` — always compare the full string
- **Memo matching:** Use unique memos (timestamp + random hash) to avoid false positives from unrelated transfers
- **Poll interval:** 3 seconds matches Hive's block time — polling faster is wasteful
- **Timeout:** 100 attempts = ~5 minutes, reasonable for manual QR scanning + confirming
- **Node failover:** Try multiple API nodes but break after first success — don't query all nodes every time

### Unique Memo Generation

```javascript
function generatePaymentMemo(prefix, shop) {
  const ts = new Date().toISOString().replace(/[-:T.Z]/g, '').slice(0, 14);
  const rand = Array.from(crypto.getRandomValues(new Uint8Array(4)))
    .map(b => b.toString(16).padStart(2, '0')).join('');
  return `${prefix}:${shop}:${ts}:${rand}`;
  // e.g. "snapie:myshop:20260320142139:a1b2c3d4"
}
```

---

## Complete Payment Flow (Both Methods)

```javascript
// 1. Generate unique memo
const memo = generatePaymentMemo('myapp', recipientAccount);
const amount = total.toFixed(3);

// 2. Option A: Keychain browser extension
function payKeychain() {
  window.hive_keychain.requestTransfer(
    null, recipientAccount, amount, memo, 'HBD',
    (resp) => {
      if (resp.success) showSuccess();
      else showError(resp.message);
    }
  );
}

// 3. Option B: QR code for mobile
function payQR() {
  const op = ["transfer", { to: recipientAccount, amount: amount + ' HBD', memo }];
  const base64 = btoa(JSON.stringify(op)).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
  const uri = `hive://sign/op/${base64}`;

  // Render QR code
  new QRCode(qrContainer, { text: uri, width: 240, height: 240, ... });

  // Start polling for confirmation
  pollForPayment(recipientAccount, amount + ' HBD', memo, showSuccess, showTimeout);
}
```
