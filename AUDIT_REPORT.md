# Engineering Audit Report

## 1. Private Key Leak in `wallet.js`

**Finding:**
A critical security vulnerability was identified in `base_wallet_backend/core/wallet.js`. The `CreateKeys` class contains a `console.log` statement within the `createPublicKey` method that prints the raw private key to the standard output.

```javascript
    console.log(
      `Private Key ${this.privateKey} \nPublic Key Hash :${PubKeyHash.toString(
        "hex"
      )}\nPublic Address: ${Publicaddress}`
    );
```

**Impact:**
This exposure allows anyone with access to the application logs to obtain the private key, compromising the funds controlled by that key. This is a severe violation of security best practices (CWE-532: Insertion of Sensitive Information into Log File).

**Remediation:**
The `console.log` statement exposing the private key must be removed immediately.

## 2. Lack of Atomic Database Locking in `prepareTx.js`

**Finding:**
The `createTransaction` process in `base_wallet_backend/core/prepareTx.js` fetches spent transactions via `fetchSpentTxs` to identify available UTXOs. However, there is no mechanism to lock these UTXOs in the database during the transaction creation process.

**Impact:**
This creates a race condition known as a "double-spend" vulnerability. If two transaction requests are processed concurrently, they may both select the same UTXOs as inputs. Since the database does not enforce atomic locking or mark UTXOs as "pending spent," both transactions could be successfully created and broadcast. While the blockchain network would eventually reject the second transaction, the application state may become inconsistent, leading to potential financial loss or service disruption.

**Remediation:**
Implement atomic database locking or a "pending" state for UTXOs. When a UTXO is selected for a transaction, it should be marked as locked/pending in the database within a transaction. Subsequent requests should ignore locked UTXOs.

## 3. UTXO Model vs. HSM/Secure Enclaves & Idempotent Ledgers

**Finding:**
The current UTXO implementation is fundamentally incompatible with the mandate for HSM/Secure Enclaves and Idempotent Ledgers.

**Analysis:**

*   **HSM/Secure Enclaves:**
    The codebase (e.g., `wallet.js`, `prepareTx.js`) handles raw private keys in application memory. The `CreateKeys` and `createTx` classes accept `privateKey` as a constructor argument and use it for signing operations in software.
    *   **Violation:** Secure Enclaves and HSMs require that private keys *never* leave the secure hardware boundary. Signing operations must be offloaded to the hardware.
    *   **Recommendation:** Refactor the wallet architecture to store keys in an HSM/Secure Enclave. The application should only hold references (key handles) and request signatures from the HSM for transaction hashes.

*   **Idempotent Ledgers:**
    The UTXO model consumes specific, unique outputs (Unspent Transaction Outputs). Once a UTXO is spent, it cannot be spent again.
    *   **Violation:** This model makes it difficult to implement idempotent ledger operations at the application level. If a transaction submission is retried (e.g., due to a network timeout), the same request might fail if the underlying UTXOs were effectively "consumed" by the first attempt (even if the application didn't confirm it yet) or if the retry logic attempts to use different UTXOs for the same logical operation.
    *   **Recommendation:** An idempotent ledger typically implies an account-based model or a request-id based system where the same request (same ID) produces the same result (or no-op if already applied). To support this with UTXOs, the system needs a complex management layer that tracks "intent" and maps it to UTXO selection, ensuring that retries don't create conflicting transactions or double-spends. The current implementation lacks this abstraction.
