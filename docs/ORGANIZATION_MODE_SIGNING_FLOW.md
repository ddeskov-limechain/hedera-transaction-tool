# Transaction Signing Flow in Organization Mode

This document explains the complete flow of what happens when a transaction is signed in **organization mode** in the Hedera Transaction Tool, and how it differs from personal mode.

## Table of Contents

1. [Overview: Organization vs Personal Mode](#overview-organization-vs-personal-mode)
2. [Architecture Overview](#architecture-overview)
3. [Key Components](#key-components)
4. [Complete Flow Diagram](#complete-flow-diagram)
5. [Step-by-Step Walkthrough](#step-by-step-walkthrough)
6. [Backend Services Deep Dive](#backend-services-deep-dive)
7. [Signature Computation](#signature-computation)
8. [Multi-Signer Coordination](#multi-signer-coordination)
9. [Notification System](#notification-system)
10. [Approval Workflows](#approval-workflows)
11. [Transaction Execution](#transaction-execution)
12. [Database Entities](#database-entities)
13. [Key Differences from Personal Mode](#key-differences-from-personal-mode)

---

## Overview: Organization vs Personal Mode

| Aspect | Personal Mode | Organization Mode |
|--------|---------------|-------------------|
| **Users** | Single user | Multiple users in an organization |
| **Signing** | Local only | Local signing + backend coordination |
| **Storage** | Local SQLite (Prisma) | Remote PostgreSQL (TypeORM) |
| **Execution** | Direct to Hedera from Electron | Backend service executes to Hedera |
| **Multi-sig** | User must own all keys | Different users can provide signatures |
| **Notifications** | None | Real-time via WebSocket + Email |
| **Approvals** | None | Threshold-based approval workflows |
| **Backend** | Not used | NestJS API + Chain + Notifications services |

**When to use Organization Mode:**
- Multi-signature transactions requiring keys from different people
- Teams that need visibility into pending transactions
- Workflows requiring approvals before execution
- Audit trails and transaction history across an organization

---

## Architecture Overview

Organization mode introduces a **backend server** that coordinates multi-user signing:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        FRONTEND (Electron/Vue)                               │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  User A                    User B                    User C          │    │
│  │  (Creator)                 (Signer)                  (Approver)      │    │
│  │     │                         │                         │            │    │
│  │     ▼                         ▼                         ▼            │    │
│  │  Signs locally            Signs locally            Approves          │    │
│  │  Submits tx               Uploads signature        Signs approval    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                    │                 │                     │                 │
│                    └────────────┬────┴─────────────────────┘                 │
│                                 │ REST API + WebSocket                       │
└─────────────────────────────────┼───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BACKEND SERVER (Remote)                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐  │
│  │   API Service   │  │  Chain Service  │  │   Notifications Service     │  │
│  │   (NestJS)      │  │  (NestJS)       │  │   (NestJS)                  │  │
│  │                 │  │                 │  │                             │  │
│  │ - Create tx     │  │ - Schedule exec │  │ - NATS consumer             │  │
│  │ - Upload sigs   │  │ - Execute tx    │  │ - WebSocket gateway         │  │
│  │ - Manage users  │  │ - Cron jobs     │  │ - Email notifications       │  │
│  └────────┬────────┘  └────────┬────────┘  └──────────────┬──────────────┘  │
│           │                    │                          │                  │
│           └────────────────────┼──────────────────────────┘                  │
│                                │                                             │
│                    ┌───────────▼───────────┐                                │
│                    │      NATS (Pub/Sub)    │                                │
│                    └───────────────────────┘                                │
│                                │                                             │
│                    ┌───────────▼───────────┐                                │
│                    │  PostgreSQL Database   │                                │
│                    │  (TypeORM entities)    │                                │
│                    └───────────────────────┘                                │
└─────────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │     HEDERA NETWORK      │
                    │   (via Hedera SDK)      │
                    └─────────────────────────┘
```

### Backend Services

| Service | Location | Purpose |
|---------|----------|---------|
| **API** | `back-end/apps/api/` | REST API for transactions, users, keys |
| **Chain** | `back-end/apps/chain/` | Schedules and executes transactions |
| **Notifications** | `back-end/apps/notifications/` | WebSocket, email, NATS processing |

---

## Key Components

### Frontend (Renderer Process)

| File | Purpose |
|------|---------|
| `front-end/src/renderer/components/Transaction/TransactionProcessor/components/OrganizationRequestHandler.vue` | Handles org mode submission |
| `front-end/src/renderer/services/organization/transaction.ts` | API calls for transactions |
| `front-end/src/renderer/stores/storeWebsocketConnection.ts` | WebSocket connection management |
| `front-end/src/renderer/stores/storeNotifications.ts` | Real-time notification handling |

### Backend API Service

| File | Purpose |
|------|---------|
| `back-end/apps/api/src/transactions/transactions.controller.ts` | Transaction endpoints |
| `back-end/apps/api/src/transactions/transactions.service.ts` | Transaction business logic |
| `back-end/apps/api/src/transactions/signers/signers.service.ts` | Signature upload handling |
| `back-end/apps/api/src/transactions/approvers/approvers.service.ts` | Approval workflows |

### Backend Chain Service

| File | Purpose |
|------|---------|
| `back-end/apps/chain/src/transaction-scheduler/transaction-scheduler.service.ts` | Cron-based execution scheduling |
| `back-end/apps/chain/src/execute/execute.service.ts` | Hedera network execution |

### Backend Notifications Service

| File | Purpose |
|------|---------|
| `back-end/apps/notifications/src/receiver/receiver.service.ts` | Notification processing |
| `back-end/apps/notifications/src/websocket/websocket.gateway.ts` | WebSocket server |
| `back-end/apps/notifications/src/fan-out/fan-out.service.ts` | Notification delivery |

### Shared Libraries

| File | Purpose |
|------|---------|
| `back-end/libs/common/src/transaction-signature/transaction-signature.service.ts` | Compute required signatures |
| `back-end/libs/common/src/database/entities/` | TypeORM entity definitions |
| `back-end/libs/common/src/utils/transaction/` | Transaction utilities |

---

## Complete Flow Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    USER A (Creator) CREATES TRANSACTION                       │
└─────────────────────────────────────┬────────────────────────────────────────┘
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  FRONTEND: OrganizationRequestHandler.vue                                     │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │ 1. Sign transaction locally (same as personal mode)                     │  │
│  │    - Decrypt private key with password                                  │  │
│  │    - privateKey.sign(transactionBytes)                                  │  │
│  │                                                                          │  │
│  │ 2. Submit to backend API                                                │  │
│  │    POST /transactions                                                   │  │
│  │    { name, description, transactionBytes, signature, creatorKeyId }    │  │
│  │                                                                          │  │
│  │ 3. Upload observers and approvers (if any)                              │  │
│  │    POST /transactions/:id/observers                                     │  │
│  │    POST /transactions/:id/approvers                                     │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────┬────────────────────────────────────────┘
                                      │
                          REST API (HTTPS)
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  BACKEND API: TransactionsService.createTransaction()                         │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │ 1. Validate:                                                            │  │
│  │    - Creator key belongs to authenticated user                          │  │
│  │    - Signature is valid for transaction bytes                           │  │
│  │    - Transaction not expired                                            │  │
│  │                                                                          │  │
│  │ 2. Store in PostgreSQL:                                                 │  │
│  │    - Transaction entity (status: WAITING_FOR_SIGNATURES)                │  │
│  │    - Link to creatorKey                                                 │  │
│  │                                                                          │  │
│  │ 3. Emit notification via NATS:                                          │  │
│  │    emitTransactionStatusUpdate() → 'notifications.queue.transaction...' │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────┬────────────────────────────────────────┘
                                      │
                              NATS Pub/Sub
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  NOTIFICATIONS SERVICE: ReceiverService                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │ 1. Consume NATS message (TRANSACTION_STATUS_UPDATE)                     │  │
│  │                                                                          │  │
│  │ 2. Determine who needs to sign:                                         │  │
│  │    - TransactionSignatureService.computeSignatureKey()                  │  │
│  │    - Find UserKeys matching required public keys                        │  │
│  │    - Get user IDs from UserKeys                                         │  │
│  │                                                                          │  │
│  │ 3. Create notifications:                                                │  │
│  │    - NotificationType.TRANSACTION_INDICATOR_SIGN                        │  │
│  │    - One NotificationReceiver per user                                  │  │
│  │                                                                          │  │
│  │ 4. Emit to FanOutService via NATS                                       │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                        │
│                                      ▼                                        │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │ FanOutService.processNewNotifications()                                 │  │
│  │   - websocket.notifyUser(userId, 'notifications:new', payload)          │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────┬────────────────────────────────────────┘
                                      │
                              WebSocket
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  USER B's FRONTEND: Receives WebSocket notification                           │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │ storeNotifications.ts listens for 'notifications:new'                   │  │
│  │ UI shows: "Transaction X requires your signature"                       │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────┬────────────────────────────────────────┘
                                      │
                              User B clicks "Sign"
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  USER B SIGNS TRANSACTION                                                     │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │ 1. Fetch transaction: GET /transactions/:id                             │  │
│  │                                                                          │  │
│  │ 2. Sign locally:                                                        │  │
│  │    - Decrypt User B's private key                                       │  │
│  │    - transaction.sign(privateKey)                                       │  │
│  │    - Extract signature map                                              │  │
│  │                                                                          │  │
│  │ 3. Upload signature:                                                    │  │
│  │    POST /transactions/signers                                           │  │
│  │    [{ id: transactionId, signatureMap: {...} }]                        │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────┬────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  BACKEND API: SignersService.uploadSignatureMaps()                            │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │ 1. Validate:                                                            │  │
│  │    - Transaction status is WAITING_FOR_SIGNATURES                       │  │
│  │    - Transaction not expired                                            │  │
│  │    - Signature matches user's registered public key                     │  │
│  │                                                                          │  │
│  │ 2. Add signature to transaction:                                        │  │
│  │    - Parse SDK transaction from bytes                                   │  │
│  │    - Add signature to transaction's signature map                       │  │
│  │    - Update transactionBytes in database                                │  │
│  │                                                                          │  │
│  │ 3. Record signer:                                                       │  │
│  │    - Create TransactionSigner entity                                    │  │
│  │                                                                          │  │
│  │ 4. Check if all signatures collected:                                   │  │
│  │    processTransactionStatus()                                           │  │
│  │      - computeSignatureKey() → required keys                            │  │
│  │      - Compare with collected signatures                                │  │
│  │      - If complete: status → WAITING_FOR_EXECUTION                      │  │
│  │                                                                          │  │
│  │ 5. Emit notification via NATS                                           │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────┬────────────────────────────────────────┘
                                      │
                    (If all signatures collected)
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  CHAIN SERVICE: TransactionSchedulerService                                   │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │ Cron jobs run at intervals based on transaction validStart:             │  │
│  │                                                                          │  │
│  │ @Cron('*/10 * * * * *')  // Every 10 seconds                           │  │
│  │ async handleTransactionsWithin3Minutes() {                              │  │
│  │   const transactions = await this.getTransactionsNearValidStart(3min)   │  │
│  │   for (const tx of transactions) {                                      │  │
│  │     await this.collateAndExecute(tx)                                    │  │
│  │   }                                                                      │  │
│  │ }                                                                        │  │
│  │                                                                          │  │
│  │ collateAndExecute():                                                    │  │
│  │   1. Smart signature collation (optimize for threshold keys)            │  │
│  │   2. Schedule execution 5 seconds after validStart                      │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────┬────────────────────────────────────────┘
                                      │
                    (At scheduled execution time)
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  CHAIN SERVICE: ExecuteService.executeTransaction()                           │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │ 1. Parse transaction from bytes                                         │  │
│  │    const sdkTransaction = Transaction.fromBytes(tx.transactionBytes)    │  │
│  │                                                                          │  │
│  │ 2. Execute on Hedera network                                            │  │
│  │    const response = await sdkTransaction.execute(client)                │  │
│  │                                                                          │  │
│  │ 3. Get receipt                                                          │  │
│  │    const receipt = await response.getReceipt(client)                    │  │
│  │                                                                          │  │
│  │ 4. Update transaction status                                            │  │
│  │    - SUCCESS → status: EXECUTED                                         │  │
│  │    - FAILURE → status: FAILED, statusCode: receipt.status               │  │
│  │                                                                          │  │
│  │ 5. Emit notification via NATS                                           │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────┬────────────────────────────────────────┘
                                      │
                              NATS + WebSocket
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  ALL USERS RECEIVE EXECUTION NOTIFICATION                                     │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │ WebSocket: 'notifications:new'                                          │  │
│  │ Type: TRANSACTION_INDICATOR_EXECUTED                                    │  │
│  │ UI shows: "Transaction X was executed successfully"                     │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Step-by-Step Walkthrough

### Step 1: Creator Submits Transaction

The creator builds a transaction and submits it through the `OrganizationRequestHandler`.

**Location**: `front-end/src/renderer/components/Transaction/TransactionProcessor/components/OrganizationRequestHandler.vue`

```typescript
async function handle(req: Processable) {
  // Only handles if user has selectedOrganization
  if (!user.selectedOrganization) {
    await nextHandler.value?.handle(req)  // Falls through to personal mode
    return
  }

  const publicKey = user.keyPairs[0].public_key

  // 1. Sign locally (same crypto as personal mode)
  const signature = await sign(publicKey)

  // 2. Submit to backend
  const { id, transactionBytes } = await submit(publicKey, signature)

  // 3. Upload observers and approvers
  await Promise.allSettled([
    upload('observers', id),
    upload('approvers', id),
    draft.deleteIfNotTemplate(),
  ])

  emit('transaction:submit:success', id, transactionBytes)
}
```

**Key difference from personal mode**: Instead of passing to `ExecutePersonalRequestHandler`, it submits to the backend API.

### Step 2: Backend Creates Transaction

**Location**: `back-end/apps/api/src/transactions/transactions.service.ts`

```typescript
async createTransaction(dto: CreateTransactionDto, user: User): Promise<Transaction[]> {
  // 1. Validate creator's key belongs to this user
  const creatorKey = await this.userKeyRepo.findOne({
    where: { id: dto.creatorKeyId, user: { id: user.id } }
  })

  // 2. Validate signature matches transaction bytes
  const isValidSignature = verifySignature(
    dto.transactionBytes,
    dto.signature,
    creatorKey.publicKey
  )

  // 3. Check transaction isn't expired
  const sdkTransaction = Transaction.fromBytes(hexToBuffer(dto.transactionBytes))
  const validStart = sdkTransaction.transactionId.validStart

  // 4. Store in database
  const transaction = this.repo.create({
    name: dto.name,
    type: getTransactionType(sdkTransaction),
    transactionId: sdkTransaction.transactionId.toString(),
    transactionBytes: hexToBuffer(dto.transactionBytes),
    unsignedTransactionBytes: hexToBuffer(dto.transactionBytes),
    status: TransactionStatus.WAITING_FOR_SIGNATURES,
    creatorKey: creatorKey,
    signature: hexToBuffer(dto.signature),
    mirrorNetwork: dto.mirrorNetwork,
    validStart: validStart,
    isManual: dto.isManual,
  })

  await this.repo.save(transaction)

  // 5. Notify via NATS
  emitTransactionStatusUpdate(this.notificationsPublisher, [{
    entityId: transaction.id,
    additionalData: { status: transaction.status }
  }])

  return [transaction]
}
```

### Step 3: Notification Service Determines Required Signers

**Location**: `back-end/apps/notifications/src/receiver/receiver.service.ts`

```typescript
async processTransactionStatusUpdateNotifications(dtos: TransactionStatusUpdateDto[]) {
  for (const dto of dtos) {
    const transaction = await this.transactionRepo.findOne(dto.entityId)

    // Determine who needs to sign
    const requiredUserIds = await this.getUsersIdsRequiredToSign(transaction)

    // Create notifications for each required signer
    for (const userId of requiredUserIds) {
      const notification = this.notificationRepo.create({
        type: NotificationType.TRANSACTION_INDICATOR_SIGN,
        entityId: transaction.id,
      })

      const receiver = this.receiverRepo.create({
        notification,
        user: { id: userId },
        isRead: false,
      })

      await this.notificationRepo.save(notification)
      await this.receiverRepo.save(receiver)
    }

    // Emit to FanOut for WebSocket delivery
    this.fanOutPublisher.emit(FAN_OUT_NEW_NOTIFICATIONS, { userIds: requiredUserIds })
  }
}
```

### Step 4: Other Users Sign and Upload Signatures

**Location**: `front-end/src/renderer/services/organization/transaction.ts`

```typescript
export const uploadSignatures = async (
  userId: string,
  userPassword: string | null,
  organization: ConnectedOrganization,
  items: SignatureItem[]
) => {
  const formattedMaps: { id: number; signatureMap: object }[] = []

  for (const { publicKeys, transaction, transactionId } of items) {
    // Sign locally with each required key
    for (const publicKey of publicKeys) {
      const privateKeyRaw = await decryptPrivateKey(userId, userPassword, publicKey)
      const privateKey = getPrivateKey(publicKey, privateKeyRaw)
      await transaction.sign(privateKey)
    }

    // Extract signature map for the keys we signed with
    const signatureMap = getSignatureMapForPublicKeys(publicKeys, transaction)
    formattedMaps.push({
      id: transactionId,
      signatureMap: formatSignatureMap(signatureMap),
    })
  }

  // Upload to backend
  await axiosWithCredentials.post(
    `${organization.serverUrl}/transactions/signers`,
    formattedMaps
  )
}
```

### Step 5: Backend Processes Uploaded Signatures

**Location**: `back-end/apps/api/src/transactions/signers/signers.service.ts`

```typescript
async uploadSignatureMaps(dtos: UploadSignatureMapDto[], user: User) {
  for (const dto of dtos) {
    const transaction = await this.transactionRepo.findOne(dto.id)

    // 1. Validate transaction status
    if (![TransactionStatus.WAITING_FOR_SIGNATURES,
          TransactionStatus.WAITING_FOR_EXECUTION].includes(transaction.status)) {
      throw new BadRequestException('Transaction cannot accept signatures')
    }

    // 2. Parse and add signatures
    const sdkTransaction = Transaction.fromBytes(transaction.transactionBytes)

    for (const [publicKeyHex, signatureHex] of Object.entries(dto.signatureMap)) {
      // Verify the public key belongs to this user
      const userKey = await this.userKeyRepo.findOne({
        where: { publicKey: publicKeyHex, user: { id: user.id } }
      })

      // Add signature to SDK transaction
      sdkTransaction.addSignature(
        PublicKey.fromString(publicKeyHex),
        Buffer.from(signatureHex, 'hex')
      )

      // Record who signed
      const signer = this.signerRepo.create({
        transaction,
        userKey,
        user,
      })
      await this.signerRepo.save(signer)
    }

    // 3. Update transaction bytes with new signatures
    transaction.transactionBytes = Buffer.from(sdkTransaction.toBytes())
    await this.transactionRepo.save(transaction)

    // 4. Check if all signatures are collected
    await processTransactionStatus(transaction, this.transactionSignatureService)
  }
}
```

### Step 6: Status Transition Logic

**Location**: `back-end/libs/common/src/utils/transaction/index.ts`

```typescript
export async function processTransactionStatus(
  transaction: Transaction,
  transactionSignatureService: TransactionSignatureService
) {
  // 1. Compute what signatures are required
  const signatureKey = await transactionSignatureService.computeSignatureKey(transaction)

  // 2. Get what signatures we have
  const sdkTransaction = Transaction.fromBytes(transaction.transactionBytes)
  const collectedSignatures = [...sdkTransaction._signerPublicKeys]

  // 3. Check if we have enough signatures
  const hasAllSignatures = hasValidSignatureKey(collectedSignatures, signatureKey)

  if (hasAllSignatures && transaction.status === TransactionStatus.WAITING_FOR_SIGNATURES) {
    // 4. Smart collation for threshold keys
    const collatedTx = await smartCollate(transaction, signatureKey)

    if (collatedTx !== null) {
      transaction.status = TransactionStatus.WAITING_FOR_EXECUTION
      await transactionRepo.save(transaction)

      // 5. Notify that transaction is ready
      emitTransactionStatusUpdate(publisher, [{
        entityId: transaction.id,
        additionalData: { status: transaction.status }
      }])
    }
  }
}
```

### Step 7: Chain Service Executes Transaction

**Location**: `back-end/apps/chain/src/transaction-scheduler/transaction-scheduler.service.ts`

```typescript
@Cron('*/10 * * * * *')  // Every 10 seconds
async handleTransactionsWithin3Minutes() {
  const transactions = await this.getTransactionsNearValidStart(
    TransactionStatus.WAITING_FOR_EXECUTION,
    3 * 60 * 1000  // 3 minutes
  )

  for (const transaction of transactions) {
    await this.collateAndExecute(transaction)
  }
}

async collateAndExecute(transaction: Transaction) {
  // 1. Final signature collation (optimize threshold signatures)
  const collatedBytes = await smartCollate(transaction)

  // 2. Schedule execution 5 seconds after validStart
  const executeAt = transaction.validStart.getTime() + 5000

  setTimeout(async () => {
    await this.executeService.executeTransaction(transaction.id)
  }, executeAt - Date.now())
}
```

**Location**: `back-end/apps/chain/src/execute/execute.service.ts`

```typescript
async executeTransaction(transactionId: number) {
  const transaction = await this.transactionRepo.findOne(transactionId)

  const sdkTransaction = Transaction.fromBytes(transaction.transactionBytes)

  try {
    // Execute on Hedera
    const response = await sdkTransaction.execute(this.hederaClient)
    const receipt = await response.getReceipt(this.hederaClient)

    // Update status
    transaction.status = TransactionStatus.EXECUTED
    transaction.statusCode = receipt.status._code
    transaction.transactionHash = Buffer.from(receipt.transactionHash).toString('hex')

  } catch (error) {
    transaction.status = TransactionStatus.FAILED
    transaction.statusCode = error.status?._code || 21
  }

  await this.transactionRepo.save(transaction)

  // Notify all interested parties
  emitTransactionStatusUpdate(this.publisher, [{
    entityId: transaction.id,
    additionalData: { status: transaction.status }
  }])
}
```

---

## Backend Services Deep Dive

### API Service

The API service handles all REST endpoints and business logic.

**Key Controllers:**

| Controller | Endpoints | Purpose |
|------------|-----------|---------|
| `TransactionsController` | `/transactions` | CRUD operations |
| `SignersController` | `/transactions/signers` | Signature uploads |
| `ApproversController` | `/transactions/:id/approvers` | Approval management |
| `ObserversController` | `/transactions/:id/observers` | Observer management |

**Transaction Endpoints:**

```typescript
// Create transaction
POST /transactions
Body: { name, description, transactionBytes, signature, creatorKeyId, mirrorNetwork }

// Get transaction
GET /transactions/:id

// Upload signatures (batch)
POST /transactions/signers
Body: [{ id, signatureMap: { publicKey: signature } }]

// Manual execution trigger
PATCH /transactions/execute/:id

// Cancel transaction
PATCH /transactions/cancel/:id

// Check if user should sign
GET /transactions/sign/:transactionId
Response: { shouldSign: boolean, publicKeys: string[] }
```

### Chain Service

The chain service runs scheduled jobs to execute transactions.

**Cron Schedule:**

| Interval | Time to validStart | Purpose |
|----------|-------------------|---------|
| Every 10s | 0-3 minutes | Immediate preparation |
| Every 30s | 3-10 minutes | Near-term preparation |
| Every 1m | 10-30 minutes | Medium-term check |
| Every 5m | 30+ minutes | Long-term monitoring |

**Execution Timing:**
- Transactions are prepared 10 seconds before `validStart`
- Execution happens 5 seconds after `validStart`
- This ensures consensus happens within the valid window

### Notifications Service

The notifications service handles all real-time communication.

**Components:**

| Component | Purpose |
|-----------|---------|
| `ReceiverConsumerService` | NATS message consumer |
| `ReceiverService` | Notification creation logic |
| `FanOutService` | Notification delivery |
| `WebSocketGateway` | Socket.io server |
| `EmailService` | Email notifications |

**NATS Event Patterns:**

```typescript
// From API/Chain services
'notifications.queue.transaction.status-update'
'notifications.queue.transaction.update'
'notifications.queue.transaction.remind-signers'

// Internal fanout
'notifications.fan-out.new'
'notifications.fan-out.delete'

// Email
'notifications.queue.email'
```

---

## Signature Computation

The backend must determine **who** needs to sign a transaction. This is non-trivial because:
- Different transaction types require different signers
- Threshold keys may allow multiple valid signer combinations
- Receiver signature requirements vary

### TransactionSignatureService

**Location**: `back-end/libs/common/src/transaction-signature/transaction-signature.service.ts`

```typescript
async computeSignatureKey(transaction: Transaction): Promise<KeyList> {
  const sdkTransaction = Transaction.fromBytes(transaction.transactionBytes)
  const transactionModel = TransactionFactory.fromTransaction(sdkTransaction)

  const signatureKey = new KeyList()

  // 1. Fee payer (transaction creator's account)
  const feePayerAccount = transactionModel.getFeePayerAccountId()
  await this.addFeePayerKey(signatureKey, transaction, feePayerAccount)

  // 2. Signing accounts (depends on transaction type)
  const signingAccounts = transactionModel.getSigningAccounts()
  await this.addSigningAccountKeys(signatureKey, transaction, signingAccounts)

  // 3. Receiver accounts (if receiverSignatureRequired)
  const receiverAccounts = transactionModel.getReceiverAccounts()
  await this.addReceiverAccountKeys(signatureKey, transaction, receiverAccounts)

  // 4. Node keys (for node operations)
  const nodeId = transactionModel.getNodeId()
  if (nodeId) {
    await this.addNodeKeys(signatureKey, transaction, nodeId)
  }

  return signatureKey
}
```

### Transaction Models

Each transaction type has a model that knows its signature requirements:

**Location**: `back-end/libs/common/src/transaction-signature/model/`

```typescript
// TransferTransaction: Senders must sign
class TransferTransactionModel extends TransactionBaseModel {
  getSigningAccounts(): Set<string> {
    const accounts = new Set<string>()
    for (const transfer of this.transaction.hbarTransfersList) {
      if (transfer.amount.isNegative() && !transfer.isApproved) {
        accounts.add(transfer.accountId.toString())
      }
    }
    return accounts
  }
}

// AccountUpdateTransaction: Account being updated must sign
class AccountUpdateTransactionModel extends TransactionBaseModel {
  getSigningAccounts(): Set<string> {
    return new Set([this.transaction.accountId.toString()])
  }

  getNewKeys(): Key[] {
    // New key must also sign (proves ownership)
    return this.transaction.key ? [this.transaction.key] : []
  }
}

// TokenCreateTransaction: Treasury account must sign
class TokenCreateTransactionModel extends TransactionBaseModel {
  getSigningAccounts(): Set<string> {
    const accounts = new Set<string>()
    if (this.transaction.treasuryAccountId) {
      accounts.add(this.transaction.treasuryAccountId.toString())
    }
    return accounts
  }
}
```

### Key Resolution

To get actual signing keys, the backend fetches account info from mirror node:

```typescript
async addSigningAccountKeys(keyList: KeyList, transaction: Transaction, accounts: Set<string>) {
  for (const accountId of accounts) {
    // Fetch account info from mirror node
    const accountInfo = await this.mirrorNodeService.getAccountInfo(
      transaction.mirrorNetwork,
      accountId
    )

    // Add account's key to required signatures
    const accountKey = Key.fromString(accountInfo.key.key)
    keyList.push(accountKey)
  }
}
```

---

## Multi-Signer Coordination

### How Users Know They Need to Sign

1. **Backend computes required keys** via `TransactionSignatureService`
2. **Maps keys to UserKeys** in database
3. **Gets user IDs** from UserKeys
4. **Creates notifications** for each user
5. **Delivers via WebSocket** in real-time

### Frontend Notification Handling

**Location**: `front-end/src/renderer/stores/storeNotifications.ts`

```typescript
function listenForUpdates() {
  for (const serverUrl of connectedOrganizations) {
    websocket.on(serverUrl, 'notifications:new', (notifications) => {
      // Add to notification store
      notifications.value[serverUrl].push(...notifications)

      // Update UI indicators
      for (const notification of notifications) {
        if (notification.type === 'TRANSACTION_INDICATOR_SIGN') {
          pendingSignatures.value++
        }
      }
    })
  }
}
```

### Batch Signing

Users can sign multiple transactions at once:

```typescript
// Frontend calls signTransactions() with array
await signTransactions(
  [transaction1, transaction2, transaction3],
  password,
  accountCache,
  nodeCache
)

// Each transaction is signed and signatures uploaded in batch
POST /transactions/signers
[
  { id: 1, signatureMap: {...} },
  { id: 2, signatureMap: {...} },
  { id: 3, signatureMap: {...} }
]
```

---

## Notification System

### Notification Types

```typescript
enum NotificationType {
  // Transaction indicators (in-app badges)
  TRANSACTION_INDICATOR_SIGN = 'TRANSACTION_INDICATOR_SIGN',
  TRANSACTION_INDICATOR_APPROVE = 'TRANSACTION_INDICATOR_APPROVE',
  TRANSACTION_INDICATOR_EXECUTABLE = 'TRANSACTION_INDICATOR_EXECUTABLE',
  TRANSACTION_INDICATOR_EXECUTED = 'TRANSACTION_INDICATOR_EXECUTED',

  // Email notifications
  TRANSACTION_WAITING_FOR_SIGNATURES = 'TRANSACTION_WAITING_FOR_SIGNATURES',
  TRANSACTION_READY_FOR_EXECUTION = 'TRANSACTION_READY_FOR_EXECUTION',
  TRANSACTION_EXECUTED = 'TRANSACTION_EXECUTED',
  TRANSACTION_EXPIRED = 'TRANSACTION_EXPIRED',
}
```

### Status → Notification Mapping

```typescript
// In-app indicators
const IN_APP_NOTIFICATION_TYPES = {
  [TransactionStatus.WAITING_FOR_SIGNATURES]: NotificationType.TRANSACTION_INDICATOR_SIGN,
  [TransactionStatus.WAITING_FOR_EXECUTION]: NotificationType.TRANSACTION_INDICATOR_EXECUTABLE,
  [TransactionStatus.EXECUTED]: NotificationType.TRANSACTION_INDICATOR_EXECUTED,
}

// Email notifications
const EMAIL_NOTIFICATION_TYPES = {
  [TransactionStatus.WAITING_FOR_SIGNATURES]: NotificationType.TRANSACTION_WAITING_FOR_SIGNATURES,
  [TransactionStatus.WAITING_FOR_EXECUTION]: NotificationType.TRANSACTION_READY_FOR_EXECUTION,
  [TransactionStatus.EXECUTED]: NotificationType.TRANSACTION_EXECUTED,
}
```

### WebSocket Connection

**Location**: `front-end/src/renderer/stores/storeWebsocketConnection.ts`

```typescript
function connect(serverUrl: string, token: string) {
  const socket = io(serverUrl, {
    path: '/ws',
    auth: { token },
    transports: ['websocket'],
  })

  socket.on('connect', () => {
    // Join user-specific room
    socket.emit('join', { userId: user.id })
  })

  socket.on('notifications:new', handleNewNotifications)
  socket.on('notifications:indicators:delete', handleDeleteIndicators)

  connections.value[serverUrl] = socket
}
```

---

## Approval Workflows

Organization mode supports approval workflows where transactions require explicit approval before execution.

### Approver Tree Structure

Approvers are stored hierarchically to support threshold-based approvals:

```
Transaction
├── Approver List (threshold: 2 of 3)
│   ├── User A (approved: true)
│   ├── User B (approved: null - pending)
│   └── User C (approved: false - rejected)
```

### Adding Approvers

```typescript
// POST /transactions/:id/approvers
{
  "approvers": [
    {
      "threshold": 2,
      "approvers": [
        { "userId": 1 },
        { "userId": 2 },
        { "userId": 3 }
      ]
    }
  ]
}
```

### Approval Flow

1. Creator adds approvers when creating transaction
2. Approvers receive notifications
3. Each approver signs with their choice:

```typescript
// POST /transactions/:id/approvers/approve
{
  "userKeyId": 123,
  "signature": "0x...",  // Signature of transaction bytes
  "approved": true       // or false to reject
}
```

4. When threshold is met, transaction proceeds
5. If rejected, transaction status becomes `REJECTED`

---

## Transaction Execution

### Automatic Execution

The chain service automatically executes transactions when:
1. Status is `WAITING_FOR_EXECUTION`
2. All required signatures are collected
3. `validStart` time is reached
4. `isManual` is `false`

### Manual Execution

For transactions with `isManual: true`:

```typescript
// User triggers execution
PATCH /transactions/execute/:id

// Backend validates and executes immediately
async executeManually(transactionId: number, user: User) {
  const transaction = await this.getTransaction(transactionId)

  // Verify user has permission (creator or signer)
  this.verifyExecutionPermission(transaction, user)

  // Execute
  await this.executeService.executeTransaction(transactionId)
}
```

### Execution Result Handling

```typescript
async executeTransaction(transactionId: number) {
  const transaction = await this.repo.findOne(transactionId)
  const sdkTx = Transaction.fromBytes(transaction.transactionBytes)

  try {
    const response = await sdkTx.execute(this.client)
    const receipt = await response.getReceipt(this.client)

    transaction.status = TransactionStatus.EXECUTED
    transaction.statusCode = receipt.status._code
    transaction.transactionHash = receipt.transactionHash

  } catch (error) {
    transaction.status = TransactionStatus.FAILED
    transaction.statusCode = this.extractStatusCode(error)
  }

  await this.repo.save(transaction)
  this.emitStatusUpdate(transaction)
}
```

---

## Database Entities

### Transaction Entity

**Location**: `back-end/libs/common/src/database/entities/transaction.entity.ts`

```typescript
@Entity()
export class Transaction {
  @PrimaryGeneratedColumn()
  id: number

  @Column()
  name: string

  @Column({ type: 'enum', enum: TransactionType })
  type: TransactionType

  @Column({ nullable: true })
  description: string

  @Column()
  transactionId: string  // e.g., "0.0.123@1234567890.000000000"

  @Column({ nullable: true })
  transactionHash: string

  @Column({ type: 'bytea' })
  transactionBytes: Buffer  // Current bytes with all signatures

  @Column({ type: 'bytea' })
  unsignedTransactionBytes: Buffer  // Original unsigned bytes

  @Column({ type: 'enum', enum: TransactionStatus })
  status: TransactionStatus

  @Column({ nullable: true })
  statusCode: number  // Hedera response code

  @ManyToOne(() => UserKey)
  creatorKey: UserKey

  @Column({ type: 'bytea' })
  signature: Buffer  // Creator's signature

  @Column()
  validStart: Date

  @Column()
  mirrorNetwork: string

  @Column({ default: false })
  isManual: boolean

  @Column({ nullable: true })
  cutoffAt: Date

  // Relations
  @OneToMany(() => TransactionSigner, signer => signer.transaction)
  signers: TransactionSigner[]

  @OneToMany(() => TransactionApprover, approver => approver.transaction)
  approvers: TransactionApprover[]

  @OneToMany(() => TransactionObserver, observer => observer.transaction)
  observers: TransactionObserver[]

  @OneToOne(() => TransactionGroupItem, item => item.transaction)
  groupItem: TransactionGroupItem
}
```

### TransactionSigner Entity

```typescript
@Entity()
export class TransactionSigner {
  @PrimaryGeneratedColumn()
  id: number

  @ManyToOne(() => Transaction)
  transaction: Transaction

  @Column()
  transactionId: number

  @ManyToOne(() => UserKey)
  userKey: UserKey

  @Column()
  userKeyId: number

  @ManyToOne(() => User)
  user: User

  @Column()
  userId: number

  @CreateDateColumn()
  createdAt: Date
}
```

### TransactionApprover Entity

```typescript
@Entity()
export class TransactionApprover {
  @PrimaryGeneratedColumn()
  id: number

  @ManyToOne(() => Transaction, { nullable: true })
  transaction: Transaction

  @Column({ nullable: true })
  transactionId: number

  @ManyToOne(() => TransactionApprover, { nullable: true })
  list: TransactionApprover  // Parent in tree

  @Column({ nullable: true })
  listId: number

  @Column({ nullable: true })
  threshold: number

  @ManyToOne(() => UserKey, { nullable: true })
  userKey: UserKey

  @Column({ nullable: true })
  userKeyId: number

  @Column({ type: 'bytea', nullable: true })
  signature: Buffer

  @ManyToOne(() => User, { nullable: true })
  user: User

  @Column({ nullable: true })
  userId: number

  @Column({ nullable: true })
  approved: boolean  // null=pending, true=approved, false=rejected

  @OneToMany(() => TransactionApprover, approver => approver.list)
  approvers: TransactionApprover[]  // Children in tree
}
```

### UserKey Entity

```typescript
@Entity()
export class UserKey {
  @PrimaryGeneratedColumn()
  id: number

  @ManyToOne(() => User)
  user: User

  @Column()
  userId: number

  @Column()
  mnemonicHash: string

  @Column()
  index: number

  @Column()
  @Index()
  publicKey: string

  @OneToMany(() => Transaction, tx => tx.creatorKey)
  createdTransactions: Transaction[]

  @OneToMany(() => TransactionApprover, approver => approver.userKey)
  approvedTransactions: TransactionApprover[]

  @OneToMany(() => TransactionSigner, signer => signer.userKey)
  signedTransactions: TransactionSigner[]
}
```

---

## Key Differences from Personal Mode

| Aspect | Personal Mode | Organization Mode |
|--------|---------------|-------------------|
| **Transaction Creation** | Built in renderer, signed in main process | Built in renderer, signed locally, submitted to backend |
| **Storage** | Local SQLite (Prisma) | Remote PostgreSQL (TypeORM) |
| **Signature Collection** | All signatures from one user | Signatures from multiple users |
| **Signature Tracking** | Not tracked | TransactionSigner entities track who signed |
| **Execution Trigger** | Immediate after signing | Scheduled by chain service based on validStart |
| **Execution Location** | Electron main process | Backend chain service |
| **Notifications** | None | WebSocket + Email |
| **Approvals** | None | Threshold-based approval workflows |
| **Observers** | None | Users can observe without signing |
| **Transaction Groups** | None | Atomic/sequential execution groups |
| **Audit Trail** | Local only | Full audit trail in backend |

### Code Path Comparison

**Personal Mode:**
```
SignPersonalRequestHandler
  → IPC: signTransaction()
  → Main Process: sign with Hedera SDK
  → ExecutePersonalRequestHandler
  → IPC: executeTransaction()
  → Main Process: execute with Hedera SDK
  → IPC: storeTransaction()
  → Prisma: save to SQLite
```

**Organization Mode:**
```
OrganizationRequestHandler
  → Local: sign with Hedera SDK
  → REST: POST /transactions (submit to backend)
  → Backend: store in PostgreSQL
  → NATS: emit notification
  → WebSocket: notify other signers

[Other users sign]
  → REST: POST /transactions/signers
  → Backend: add signatures, check if complete
  → NATS: emit status update

[Chain service]
  → Cron: check transactions near validStart
  → Execute: submit to Hedera network
  → NATS: emit execution result
  → WebSocket: notify all users
```

---

## Summary

Organization mode transforms the Hedera Transaction Tool from a single-user application into a **multi-user collaboration platform**:

1. **Creator submits** → Signs locally, submits to backend API
2. **Backend stores** → PostgreSQL with status `WAITING_FOR_SIGNATURES`
3. **Notifications sent** → NATS → Notification service → WebSocket to required signers
4. **Other users sign** → Fetch transaction, sign locally, upload signatures
5. **Backend aggregates** → Adds signatures, tracks who signed
6. **Status transitions** → `WAITING_FOR_SIGNATURES` → `WAITING_FOR_EXECUTION`
7. **Chain service executes** → Cron job executes at `validStart + 5s`
8. **All users notified** → WebSocket delivers execution result

This architecture enables:
- **Multi-signature transactions** across organizational boundaries
- **Audit trails** for compliance and governance
- **Real-time collaboration** with instant notifications
- **Approval workflows** for sensitive transactions
- **Scheduled execution** for precise timing requirements
