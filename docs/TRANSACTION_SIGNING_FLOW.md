# Transaction Signing Flow in Personal Mode

This document explains the complete flow of what happens when a transaction is signed in **personal mode** in the Hedera Transaction Tool.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Key Components](#key-components)
3. [Complete Flow Diagram](#complete-flow-diagram)
4. [Step-by-Step Walkthrough](#step-by-step-walkthrough)
5. [Frontend Implementation](#frontend-implementation)
6. [IPC Communication](#ipc-communication)
7. [Main Process Signing](#main-process-signing)
8. [Transaction Execution](#transaction-execution)
9. [Data Structures](#data-structures)
10. [Security Mechanisms](#security-mechanisms)

---

## Architecture Overview

The Hedera Transaction Tool uses a **three-tier architecture** within the Electron app:

```
┌─────────────────────────────────────────────────────────────────┐
│                     RENDERER PROCESS (Vue 3)                     │
│  ┌─────────────┐  ┌──────────────────┐  ┌───────────────────┐  │
│  │ Vue         │  │ Transaction      │  │ Stores            │  │
│  │ Components  │→ │ Processor Chain  │→ │ (Pinia)           │  │
│  └─────────────┘  └──────────────────┘  └───────────────────┘  │
│                              │                                   │
│                    ┌─────────▼─────────┐                        │
│                    │ IPC Services      │                        │
│                    │ (transactionService.ts)                    │
│                    └─────────┬─────────┘                        │
└──────────────────────────────┼──────────────────────────────────┘
                               │ ipcRenderer.invoke()
                    ┌──────────▼──────────┐
                    │    PRELOAD BRIDGE   │
                    │    (Secure IPC)     │
                    └──────────┬──────────┘
                               │ ipcMain.handle()
┌──────────────────────────────┼──────────────────────────────────┐
│                     MAIN PROCESS (Node.js)                       │
│                    ┌─────────▼─────────┐                        │
│                    │ IPC Handlers      │                        │
│                    │ (transactions.ts) │                        │
│                    └─────────┬─────────┘                        │
│                              │                                   │
│  ┌─────────────┐  ┌─────────▼─────────┐  ┌───────────────────┐  │
│  │ Prisma DB   │← │ Transaction       │→ │ Hedera SDK        │  │
│  │ (SQLite)    │  │ Service           │  │ (signing/execute) │  │
│  └─────────────┘  └───────────────────┘  └───────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Personal Mode vs Organization Mode

| Aspect | Personal Mode | Organization Mode |
|--------|---------------|-------------------|
| Signing | Local (Electron main process) | Local signing + backend coordination |
| Storage | Local SQLite via Prisma | PostgreSQL on backend server |
| Multi-user | Single user | Multiple signers via backend API |
| Execution | Direct to Hedera network | Backend coordinates execution |

---

## Key Components

### Frontend (Renderer Process)

| File | Purpose |
|------|---------|
| `front-end/src/renderer/stores/storeUser.ts` | User state, key pairs, password caching |
| `front-end/src/renderer/components/Transaction/TransactionProcessor/` | Chain of responsibility for processing |
| `front-end/src/renderer/components/Transaction/TransactionProcessor/components/SignPersonalRequestHandler.vue` | Personal mode signing handler |
| `front-end/src/renderer/components/Transaction/TransactionProcessor/components/ExecutePersonalRequestHandler.vue` | Network execution handler |
| `front-end/src/renderer/services/transactionService.ts` | IPC wrapper for transaction operations |

### IPC Layer

| File | Purpose |
|------|---------|
| `front-end/src/preload/localUser/transactions.ts` | Preload script exposing IPC methods |
| `front-end/src/main/modules/ipcHandlers/localUser/transactions.ts` | IPC handler registration |

### Main Process

| File | Purpose |
|------|---------|
| `front-end/src/main/services/localUser/transactions.ts` | Core signing and execution logic |
| `front-end/src/main/services/localUser/keyPairs.ts` | Key pair management and decryption |

---

## Complete Flow Diagram

```
┌────────────────────────────────────────────────────────────────────────┐
│                         USER CLICKS "SIGN"                              │
└───────────────────────────────────┬────────────────────────────────────┘
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│  SignSingleButton.vue / TransactionProcessor                           │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Build TransactionRequest:                                         │  │
│  │   - transactionBytes (serialized Hedera SDK transaction)          │  │
│  │   - transactionKey (required signing keys)                        │  │
│  │   - name, description, submitManually                             │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────┬────────────────────────────────────┘
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                    CHAIN OF RESPONSIBILITY                              │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ 1. ValidateRequestHandler    → Validates transaction format     │    │
│  │ 2. ConfirmTransactionHandler → Shows confirmation modal         │    │
│  │ 3. MultipleAccountUpdateRequestHandler                          │    │
│  │ 4. BigFileOrganizationRequestHandler                            │    │
│  │ 5. BigFilePersonalRequestHandler                                │    │
│  │ 6. OrganizationRequestHandler → Skipped (personal mode)         │    │
│  │ 7. SignPersonalRequestHandler → ★ SIGNING HAPPENS HERE          │    │
│  │ 8. ExecutePersonalRequestHandler → Executes on network          │    │
│  └────────────────────────────────────────────────────────────────┘    │
└───────────────────────────────────┬────────────────────────────────────┘
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│  SignPersonalRequestHandler.vue                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ 1. Flatten transaction key → Get required public keys             │  │
│  │ 2. Filter to keys user owns (user.publicKeys)                     │  │
│  │ 3. Get password (cached or prompt user)                           │  │
│  │ 4. Call IPC: signTransaction(bytes, keys, userId, password)       │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                          IPC BOUNDARY (ipcRenderer.invoke)
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│  MAIN PROCESS - transactions.ts                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ signTransaction(transactionBytes, publicKeys, userId, password)   │  │
│  │                                                                    │  │
│  │ 1. Parse transaction: Transaction.fromBytes(transactionBytes)     │  │
│  │ 2. Freeze transaction: tx.freezeWith(client)                      │  │
│  │ 3. Load key pairs from Prisma DB                                  │  │
│  │ 4. For each public key:                                           │  │
│  │    a. Find matching key pair                                      │  │
│  │    b. Decrypt private key (password or keychain)                  │  │
│  │    c. Create PrivateKey object (ED25519 or ECDSA)                 │  │
│  │    d. Sign: tx.sign(privateKey)                                   │  │
│  │ 5. Return: tx.toBytes()                                           │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                          IPC Response (signed bytes)
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│  ExecutePersonalRequestHandler.vue                                      │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ 1. Call IPC: executeTransaction(signedBytes)                      │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                          IPC BOUNDARY
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│  MAIN PROCESS - transactions.ts                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ executeTransaction(transactionBytes)                              │  │
│  │                                                                    │  │
│  │ 1. Parse: Transaction.fromBytes(transactionBytes)                 │  │
│  │ 2. Execute: response = await tx.execute(client)                   │  │
│  │ 3. Get receipt: receipt = await response.getReceipt(client)       │  │
│  │ 4. Return: { responseJSON, receiptBytes }                         │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│  Store Transaction Locally                                              │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ storeTransaction() → Prisma → SQLite                              │  │
│  │   - transaction_id, hash, body, status, executed_at               │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────┬────────────────────────────────────┘
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                         SHOW SUCCESS/FAILURE                            │
│                    (Toast notification + UI update)                     │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Step-by-Step Walkthrough

### Step 1: User Initiates Signing

The user clicks a "Sign" button in the UI. This triggers the `TransactionProcessor` to begin processing.

**Location**: `front-end/src/renderer/components/Transaction/TransactionProcessor/TransactionProcessor.vue`

```typescript
// TransactionRequest is built from the transaction data
const request = TransactionRequest.fromData({
  transactionKey: transaction.getSignatures()._signers,
  transactionBytes: transaction.toBytes(),
  name: 'My Transaction',
  description: 'Description',
  submitManually: false,
})

// Start the chain of responsibility
await transactionProcessor.process(request)
```

### Step 2: Chain of Responsibility Processing

The request passes through multiple handlers. Each handler can:
- Process the request and pass to next handler
- Short-circuit the chain (e.g., if user cancels)
- Modify the request before passing it on

### Step 3: SignPersonalRequestHandler

**Location**: `front-end/src/renderer/components/Transaction/TransactionProcessor/components/SignPersonalRequestHandler.vue`

This handler:
1. Determines which keys the user can sign with
2. Gets the user's password (from cache or prompts)
3. Calls the IPC method to sign in the main process

```typescript
async function handle(req: TransactionRequest) {
  // Find which of the transaction's required keys this user owns
  const localPublicKeys = flattenKeyList(req.transactionKey)
    .filter(pk => user.publicKeys.includes(pk.toStringRaw()))

  if (localPublicKeys.length === 0) {
    throw Error('You have no keys to sign this transaction')
  }

  // Get password (cached with 10-min TTL or prompt user)
  const password = user.getPassword()

  emit('transaction:sign:begin')

  // IPC call to main process
  const signed = await signTransaction(
    req.transactionBytes,
    localPublicKeys.map(k => k.toStringRaw()),
    user.personal.id,
    password
  )

  // Update request with signed bytes
  req.transactionBytes = signed

  // Pass to next handler (ExecutePersonalRequestHandler)
  await nextHandler?.handle(req)

  emit('transaction:sign:success')
}
```

### Step 4: IPC Communication

**Renderer → Main Process**

**Location**: `front-end/src/renderer/services/transactionService.ts`

```typescript
export const signTransaction = async (
  transactionBytes: Uint8Array,
  publicKeys: string[],
  userId: string,
  userPassword: string | null
) => {
  return await window.electronAPI.local.transactions.signTransaction(
    transactionBytes,
    publicKeys,
    userId,
    userPassword
  )
}
```

**Preload Bridge**

**Location**: `front-end/src/preload/localUser/transactions.ts`

```typescript
export default {
  transactions: {
    signTransaction: (
      transactionBytes: Uint8Array,
      publicKeys: string[],
      userId: string,
      userPassword: string | null
    ): Promise<Uint8Array> =>
      ipcRenderer.invoke('transactions:signTransaction',
        transactionBytes, publicKeys, userId, userPassword)
  }
}
```

### Step 5: Main Process Signing

**Location**: `front-end/src/main/services/localUser/transactions.ts`

```typescript
export const signTransaction = async (
  transactionBytes: Uint8Array,
  publicKeys: string[],
  userId: string,
  userPassword: string | null
): Promise<Uint8Array> => {
  // 1. Deserialize the transaction
  const transaction = Transaction.fromBytes(transactionBytes)

  // 2. Freeze the transaction (locks it for signing)
  transaction.freezeWith(client)

  // 3. Get user's encrypted key pairs from database
  const keyPairs = await getKeyPairs(userId)
  const useKeychain = await getUseKeychainClaim()

  // 4. Sign with each required key
  for (const publicKeyStr of publicKeys) {
    const keyPair = keyPairs.find(kp => kp.public_key === publicKeyStr)

    // 5. Decrypt the private key
    let decryptedPrivateKey: string

    if (useKeychain) {
      // macOS Keychain / Windows Credential Manager
      const buffer = Buffer.from(keyPair.private_key, 'base64')
      decryptedPrivateKey = safeStorage.decryptString(buffer)
    } else {
      // Password-based decryption
      decryptedPrivateKey = decrypt(keyPair.private_key, userPassword)
    }

    // 6. Create PrivateKey object based on key type
    const privateKey = keyPair.type === 'ECDSA'
      ? PrivateKey.fromStringECDSA(decryptedPrivateKey)
      : PrivateKey.fromStringED25519(decryptedPrivateKey)

    // 7. Sign the transaction
    await transaction.sign(privateKey)
  }

  // 8. Return the signed transaction bytes
  return transaction.toBytes()
}
```

### Step 6: Transaction Execution

**Location**: `front-end/src/main/services/localUser/transactions.ts`

```typescript
export const executeTransaction = async (
  transactionBytes: Uint8Array
): Promise<{ responseJSON: string; receiptBytes: Uint8Array }> => {
  const transaction = Transaction.fromBytes(transactionBytes)

  // Execute on the Hedera network
  const response = await transaction.execute(client)

  // Wait for consensus and get receipt
  const receipt = await response.getReceipt(client)

  return {
    responseJSON: JSON.stringify(response.toJSON()),
    receiptBytes: receipt.toBytes()
  }
}
```

### Step 7: Storing and Feedback

After successful execution, the transaction is stored locally and the UI is updated.

```typescript
// Store to local database
await storeTransaction({
  name: request.name,
  type: transactionType,
  transaction_id: response.transactionId.toString(),
  transaction_hash: Buffer.from(receipt.transactionHash).toString('hex'),
  status: 'SUCCESS',
  status_code: receipt.status._code,
  executed_at: Date.now() / 1000,
  user_id: user.personal.id,
  // ... other fields
})

// Show success notification
emit('transaction:executed', true, response, receipt)
```

---

## Data Structures

### TransactionRequest

```typescript
class TransactionRequest {
  transactionKey: Key              // Required signing keys
  transactionBytes: Uint8Array     // Raw transaction bytes
  name: string                     // User-provided name
  description: string              // User-provided description
  submitManually: boolean          // If true, don't auto-execute
  reminderMillisecondsBefore: number | null
}
```

### KeyPair (Prisma Model)

```typescript
model KeyPair {
  id            String   @id @default(cuid())
  public_key    String   // Hex-encoded public key
  private_key   String   // Encrypted private key
  type          String   // "ED25519" or "ECDSA"
  user_id       String
  user          User     @relation(fields: [user_id], references: [id])
  createdAt     DateTime @default(now())
}
```

### Transaction (Prisma Model)

```typescript
model Transaction {
  id                 String   @id @default(cuid())
  name               String
  type               String
  description        String?
  transaction_id     String   // Hedera transaction ID
  transaction_hash   String   // Hex-encoded hash
  body               String   // Hex-encoded transaction bytes
  status             String   // SUCCESS, FAILED, etc.
  status_code        Int?     // Hedera status code
  valid_start        String
  user_id            String
  network            String
  executed_at        Float?
  createdAt          DateTime @default(now())
}
```

---

## Security Mechanisms

### 1. Password-Based Key Encryption

Private keys are never stored in plaintext. They are encrypted with the user's password using PBKDF2 key derivation:

```typescript
// Encryption (when key pair is created)
const encryptedKey = encrypt(privateKeyString, userPassword)
// Uses crypto.subtle.encrypt() with PBKDF2-derived key

// Decryption (when signing)
const decryptedKey = decrypt(encryptedPrivateKey, userPassword)
```

### 2. OS Keychain Integration

On supported platforms, keys can be stored in the OS secure storage:

```typescript
if (useKeychain) {
  // Uses Electron's safeStorage API
  const buffer = Buffer.from(keyPair.private_key, 'base64')
  const decrypted = safeStorage.decryptString(buffer)
}
```

### 3. Password Caching with TTL

Passwords are cached for 10 minutes to avoid repeated prompts:

```typescript
// In storeUser.ts
setPassword(password: string) {
  personal.value.password = password
  personal.value.passwordExpiresAt = new Date(Date.now() + 10 * 60 * 1000)
}

getPassword(): string | null {
  if (Date.now() > personal.value.passwordExpiresAt) {
    personal.value.password = null
    return null
  }
  // Extend expiry on access
  personal.value.passwordExpiresAt = new Date(Date.now() + 10 * 60 * 1000)
  return personal.value.password
}
```

### 4. Transaction Freezing

Transactions must be "frozen" before signing to prevent modification:

```typescript
transaction.freezeWith(client)
// After freezing, transaction details cannot be changed
// This ensures signatures are valid for the exact transaction
```

### 5. IPC Isolation

The preload script acts as a secure bridge, exposing only specific IPC methods to the renderer process. The renderer cannot directly access Node.js APIs or the file system.

---

## Error Handling

### Signing Errors

```typescript
// In SignPersonalRequestHandler
try {
  const signed = await signTransaction(...)
  emit('transaction:sign:success')
} catch (error) {
  emit('transaction:sign:fail')
  throw error  // Propagates to show error to user
}
```

### Execution Errors

```typescript
// In executeTransaction
try {
  const response = await transaction.execute(client)
  const receipt = await response.getReceipt(client)
} catch (error) {
  // Extract Hedera status code from error
  let status = error.status?._code || 21  // 21 = UNKNOWN
  throw new Error(JSON.stringify({
    status,
    message: error.message
  }))
}
```

### Common Error Scenarios

| Error | Cause | Resolution |
|-------|-------|------------|
| "No keys to sign" | User doesn't own required keys | Add key pair or use different account |
| "Password required" | Password not cached, keychain disabled | Enter password when prompted |
| "Invalid signature" | Key mismatch or corrupted data | Verify correct key pair is selected |
| "INSUFFICIENT_PAYER_BALANCE" | Fee payer has insufficient HBAR | Fund the account |
| "INVALID_TRANSACTION_START" | Transaction valid_start expired | Create new transaction |

---

## Summary

The personal mode transaction signing flow:

1. **User initiates** → Clicks sign button
2. **Build request** → TransactionRequest with bytes and required keys
3. **Chain processing** → Validation, confirmation, signing handlers
4. **Password retrieval** → Cached (10-min TTL) or user prompt
5. **IPC to main process** → Secure bridge via preload script
6. **Key decryption** → Password-based or OS keychain
7. **Hedera SDK signing** → `transaction.sign(privateKey)`
8. **IPC return** → Signed bytes back to renderer
9. **Network execution** → `transaction.execute(client)`
10. **Receipt confirmation** → `response.getReceipt(client)`
11. **Local storage** → Prisma → SQLite
12. **User feedback** → Toast notification, UI update

This architecture ensures:
- **Security**: Private keys never leave the main process unencrypted
- **Separation**: Clear boundaries between UI, IPC, and signing logic
- **Flexibility**: Chain of responsibility allows easy extension
- **Reliability**: Comprehensive error handling at each step
