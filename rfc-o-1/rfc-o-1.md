  # RFC-O-1: Octra Provider JavaScript API

  | Field | Value |
  | --- | --- |
  | Status | Draft |
  | Category | Wallet/Dapp Interface |
  | Created | 2026-05-14 |
  | Inspired by | [EIP-1193](https://eips.ethereum.org/EIPS/eip-1193) |
  | Target | Octra browser wallets, dapps, SDKs |

  ## Abstract

  RFC-O-1 defines a JavaScript provider API for Octra wallets. It gives dapps a standard `request()` interface for Octra JSON-RPC, account access, transaction signing, contract calls, privacy operations, and wallet events.

  The provider SHOULD be exposed as:

  ```ts
  window.octra
  ```

  Multiple providers MAY be exposed as:

  ```ts
  window.octraProviders
  ```

  ## Problem Statement

  Octra has a documented JSON-RPC API, but browser dapps also need a wallet interface. Those are related problems, but they are not the same problem.

  The JSON-RPC API lets software talk to an Octra node. It can read balances, inspect transactions, query epochs, estimate fees, submit signed transactions, call contracts, and fetch privacy-related chain data.

  A browser wallet interface must do more than forward RPC calls. It must also:

  - Let dapps discover an installed Octra wallet.
  - Ask the user before exposing accounts.
  - Track which permissions a dapp has been granted.
  - Protect signing keys from the web page.
  - Show transaction approvals with recipient, amount, fee, network, and staging status.
  - Explain that Octra `pending` means accepted into staging, not finalized.
  - Handle privacy workflows where encrypted balances, private transfers, proofs, and stealth outputs may involve sensitive local key material.
  - Provide consistent events when accounts, networks, balances, permissions, or transactions change.

  Without a standard provider API, every Octra wallet and dapp can invent a slightly different integration shape. That creates avoidable friction:

  - Dapps need wallet-specific adapters.
  - Wallets need to mimic one another without a shared contract.
  - Users see inconsistent permission and transaction approval flows.
  - Privacy features become harder to integrate safely.
  - SDKs cannot rely on a stable browser primitive like `window.octra`.

  This also creates a simple but serious confusion problem: users and developers will not know what they are supposed to use. One wallet may expose `window.octra`, another may require an SDK object, another may only document raw RPC calls, and another may wrap signing behind different method names. Developers then have to ask "which interface is the real one?", while users are left with inconsistent connect buttons, permission prompts, and transaction flows.

  RFC-O-1 solves this by defining a small shared boundary between dapps and wallets. Native Octra RPC remains positional and node-shaped. Wallet actions become permissioned, user-mediated provider methods. The goal is not to replace Octra JSON-RPC, but to standardize how browser applications safely reach it through a wallet.

  ## Motivation

  Octra dapps need a small, predictable browser interface for discovering wallets, requesting accounts, reading chain state, signing messages, sending transactions, interacting with contracts, and using Octra privacy features.

  EIP-1193 provides a useful minimal model: one `request()` function plus events. Octra needs the same simplicity, but with Octra-native behavior:

  - JSON-RPC 2.0 pass-through using positional params.
  - String network identifiers instead of Ethereum hex chain IDs.
  - Staging-aware transaction results.
  - Operation unit and fee helpers.
  - Contract call support.
  - Explicit permissions for encrypted balances, private transfers, and stealth workflows.

  ## Terminology

  The key words `MUST`, `MUST NOT`, `REQUIRED`, `SHOULD`, `SHOULD NOT`, and `MAY` are to be interpreted as described in RFC 2119.

  | Term | Meaning |
  | --- | --- |
  | Provider | A JavaScript object injected or exposed by an Octra wallet. |
  | Dapp | A web application using an Octra provider. |
  | Wallet | Software that manages Octra accounts, keys, permissions, and signing. |
  | Native RPC method | A method documented by the Octra JSON-RPC API. |
  | Provider-native method | A wallet-level method defined by this RFC. |
  | Staging | Octra's pending transaction pool before confirmation. |
  | OU | Operation units used for Octra fees. |

  ## Provider Interface

  ```ts
  interface OctraProvider {
    readonly isOctra: true;
    readonly providerId?: string;
    readonly version?: string;

    request(args: OctraRequestArguments): Promise<unknown>;

    on(event: OctraProviderEvent, listener: (...args: any[]) => void): this;
    removeListener(event: OctraProviderEvent, listener: (...args: any[]) => void): this;
  }

  interface OctraRequestArguments {
    readonly method: string;
    readonly params?: readonly unknown[] | object;
  }
  ```

  Native Octra RPC methods MUST use positional array params, matching Octra JSON-RPC:

  ```ts
  await window.octra.request({
    method: 'octra_balance',
    params: ['oct...']
  });
  ```

  Provider-native wallet methods SHOULD use object params for forward compatibility:

  ```ts
  await window.octra.request({
    method: 'octra_sendTransaction',
    params: [{
      to: 'oct...',
      amount: '1000000',
      fee: '1'
    }]
  });
  ```

  ## Network Identity

  Octra networks MUST use string IDs, not Ethereum-style hex chain IDs.

  ```ts
  interface OctraNetworkInfo {
    id: string;
    name: string;
    rpcUrl: string;
    explorerUrl?: string;
    supportsPrivacy: boolean;
    isTestnet: boolean;
  }
  ```

  Example network IDs:

  ```txt
  octra:mainnet-alpha
  octra:devnet
  octra:local
  ```

  A provider is connected when it can service requests to at least one Octra RPC endpoint.

  If a provider is disconnected from all Octra networks, it MUST reject requests with error code `4900`.

  If a provider is connected to Octra but cannot service the requested network, it MUST reject the request with error code `4901`.

  ## Required Methods

  ### `octra_requestAccounts`

  Requests access to one or more wallet accounts.

  ```ts
  octra_requestAccounts(params?: [{
    permissions?: OctraPermission[];
    networkId?: string;
  }]): Promise<string[]>
  ```

  The wallet MUST request user consent before exposing accounts.

  ### `octra_accounts`

  Returns accounts currently exposed to the dapp.

  ```ts
  octra_accounts(): Promise<string[]>
  ```

  If the dapp is not authorized, the provider SHOULD return an empty array.

  ### `octra_networkId`

  Returns the active Octra network ID.

  ```ts
  octra_networkId(): Promise<string>
  ```

  ### `octra_networkInfo`

  Returns metadata for the active Octra network.

  ```ts
  octra_networkInfo(): Promise<OctraNetworkInfo>
  ```

  ### `octra_permissions`

  Returns permissions granted to the dapp.

  ```ts
  octra_permissions(): Promise<OctraPermission[]>
  ```

  ### `octra_switchNetwork`

  Requests a switch to another Octra network.

  ```ts
  octra_switchNetwork(params: [{
    networkId: string;
  }]): Promise<OctraNetworkInfo>
  ```

  The wallet MAY request user confirmation before switching networks.

  ## Permissions

  ```ts
  type OctraPermission =
    | 'read_address'
    | 'read_balance'
    | 'read_public_key'
    | 'sign_messages'
    | 'send_transactions'
    | 'contract_calls'
    | 'view_encrypted_balance'
    | 'encrypt_balance'
    | 'decrypt_balance'
    | 'private_transfers'
    | 'stealth_scan'
    | 'stealth_claim';
  ```

  | Permission | Allows |
  | --- | --- |
  | `read_address` | Read exposed Octra account addresses. |
  | `read_balance` | Read public balances. |
  | `read_public_key` | Read public keys. |
  | `sign_messages` | Request message signatures. |
  | `send_transactions` | Send standard OCT transactions. |
  | `contract_calls` | Send contract transactions. |
  | `view_encrypted_balance` | Request access to encrypted balance data. |
  | `encrypt_balance` | Encrypt public balance into private balance. |
  | `decrypt_balance` | Decrypt private balance into public balance. |
  | `private_transfers` | Send private transfers. |
  | `stealth_scan` | Scan stealth outputs. |
  | `stealth_claim` | Claim stealth outputs. |

  ## Transaction Methods

  ### `octra_signMessage`

  Signs an arbitrary message.

  ```ts
  octra_signMessage(params: [{
    message: string;
    address?: string;
  }]): Promise<{
    address: string;
    publicKey: string;
    signature: string;
  }>
  ```

  The wallet MUST require `sign_messages` permission and SHOULD display the message before signing.

  ### `octra_sendTransaction`

  Creates, signs, and submits a standard Octra transaction.

  ```ts
  octra_sendTransaction(params: [{
    to: string;
    amount: string;
    fee?: string;
    message?: string;
  }]): Promise<OctraTransactionResult>
  ```

  The wallet SHOULD use `octra_recommendedFee` or `staging_estimateOu` when `fee` is omitted.

  ### `octra_signTransaction`

  Signs a transaction without submitting it.

  ```ts
  octra_signTransaction(params: [{
    to: string;
    amount: string;
    fee: string;
    nonce?: string;
    message?: string;
  }]): Promise<SignedOctraTransaction>
  ```

  ### `octra_submitTransaction`

  Submits a signed transaction.

  ```ts
  octra_submitTransaction(params: [{
    tx: SignedOctraTransaction;
  }]): Promise<OctraTransactionResult>
  ```

  ### Transaction Result

  ```ts
  interface OctraTransactionResult {
    hash: string;
    accepted: boolean;
    status: 'pending' | 'confirmed' | 'rejected' | 'dropped';
    nonce?: number;
    ouCost?: string;
    explorerUrl?: string;
  }
  ```

  `accepted: true` means the transaction was accepted into Octra staging. It does not mean the transaction is finalized.

  Dapps SHOULD watch `octra_transaction` or the `transactionChanged` event until the status is `confirmed`, `rejected`, or `dropped`.

  ## Contract Methods

  ### `octra_callContract`

  Executes a read-only contract call.

  ```ts
  octra_callContract(params: [{
    address: string;
    method: string;
    params?: unknown[];
    caller?: string;
  }]): Promise<unknown>
  ```

  This method maps to native RPC method `contract_call`.

  ### `octra_sendContractTransaction`

  Creates, signs, and submits a contract transaction.

  ```ts
  octra_sendContractTransaction(params: [{
    address: string;
    method: string;
    params?: unknown[];
    amount?: string;
    fee?: string;
  }]): Promise<OctraTransactionResult>
  ```

  The wallet MUST require `contract_calls` permission.

  ### `octra_getContractReceipt`

  Fetches a contract execution receipt.

  ```ts
  octra_getContractReceipt(params: [{
    hash: string;
  }]): Promise<unknown>
  ```

  This method maps to native RPC method `contract_receipt`.

  ## Privacy Methods

  ### `octra_getEncryptedBalance`

  Returns encrypted balance information for an address.

  ```ts
  octra_getEncryptedBalance(params?: [{
    address?: string;
  }]): Promise<{
    address: string;
    cipher?: string;
    cipherType?: string;
    decryptedAmount?: string;
    hasPvacPubkey: boolean;
  }>
  ```

  The wallet MUST require `view_encrypted_balance` permission before exposing decrypted amounts.

  ### `octra_encryptBalance`

  Encrypts public OCT balance into private balance.

  ```ts
  octra_encryptBalance(params: [{
    amount: string;
    fee?: string;
  }]): Promise<OctraTransactionResult>
  ```

  ### `octra_decryptBalance`

  Decrypts private OCT balance into public balance.

  ```ts
  octra_decryptBalance(params: [{
    amount: string;
    fee?: string;
  }]): Promise<OctraTransactionResult>
  ```

  ### `octra_sendPrivateTransfer`

  Sends a private transfer.

  ```ts
  octra_sendPrivateTransfer(params: [{
    to: string;
    amount: string;
    fee?: string;
  }]): Promise<OctraTransactionResult>
  ```

  The wallet MUST require `private_transfers` permission.

  ### `octra_scanStealth`

  Scans stealth outputs.

  ```ts
  octra_scanStealth(params?: [{
    fromEpoch?: number;
  }]): Promise<unknown[]>
  ```

  The wallet MUST require `stealth_scan` permission.

  ### `octra_claimStealth`

  Claims a stealth output.

  ```ts
  octra_claimStealth(params: [{
    outputId: string;
    fee?: string;
  }]): Promise<OctraTransactionResult>
  ```

  The wallet MUST require `stealth_claim` permission.

  ## Native RPC Pass-Through

  A compliant provider SHOULD support pass-through for documented Octra JSON-RPC methods.

  Native Octra RPC methods MUST preserve positional array params.

  ### Read Methods

  | Category | Methods |
  | --- | --- |
  | Node | `node_version`, `node_status`, `node_stats`, `node_metrics` |
  | Accounts | `octra_balance`, `octra_account`, `octra_nonce`, `octra_publicKey`, `octra_validateAddress`, `octra_supply` |
  | Transactions | `octra_transaction`, `octra_recentTransactions`, `octra_transactions`, `octra_transactionsByAddress`, `octra_transactionsByEpoch`, `octra_totalTransactions`, `octra_search` |
  | Epochs | `epoch_current`, `epoch_get`, `epoch_list`, `epoch_summaries` |
  | Fees and staging | `octra_recommendedFee`, `staging_view`, `staging_stats`, `staging_estimateOu` |
  | Contracts | `vm_contract`, `octra_contractAbi`, `octra_contractStorage`, `octra_listContracts`, `contract_receipt`, `contract_call`, `octra_computeContractAddress` |
  | Compilation | `octra_compileAssembly`, `octra_compileAml`, `octra_compileAmlMulti` |
  | Privacy | `octra_encryptedCipher`, `octra_encryptedBalance`, `octra_pvacPubkey`, `octra_viewPubkey`, `octra_stealthOutputs` |
  | Contract source | `contract_source` |

  ### Sensitive Write Methods

  Sensitive write methods SHOULD require wallet confirmation or be wrapped by safer provider-native methods.

  | Method | Suggested handling |
  | --- | --- |
  | `octra_submit` | Prefer `octra_sendTransaction` or `octra_submitTransaction`. |
  | `octra_submitBatch` | Require explicit batch confirmation. |
  | `octra_privateTransfer` | Prefer `octra_sendPrivateTransfer`. |
  | `octra_registerPublicKey` | Require wallet ownership proof. |
  | `octra_registerPvacPubkey` | Require wallet confirmation and key validation. |
  | `staging_remove` | Require explicit user confirmation. |
  | `contract_verify` | May be exposed to developer tools. |
  | `contract_saveAbi` | May be exposed to developer tools. |

  ## Events

  ```ts
  type OctraProviderEvent =
    | 'connect'
    | 'disconnect'
    | 'networkChanged'
    | 'accountsChanged'
    | 'permissionsChanged'
    | 'balanceChanged'
    | 'transactionChanged'
    | 'message';
  ```

  ### `connect`

  Emitted when the provider connects to an Octra network.

  ```ts
  {
    networkId: string;
    networkInfo: OctraNetworkInfo;
  }
  ```

  ### `disconnect`

  Emitted when the provider disconnects from all Octra networks.

  ```ts
  OctraProviderError
  ```

  ### `networkChanged`

  Emitted when the active Octra network changes.

  ```ts
  OctraNetworkInfo
  ```

  ### `accountsChanged`

  Emitted when exposed accounts change.

  ```ts
  string[]
  ```

  ### `permissionsChanged`

  Emitted when dapp permissions change.

  ```ts
  OctraPermission[]
  ```

  ### `balanceChanged`

  Emitted when a known account balance changes.

  ```ts
  {
    address: string;
    public?: string;
    encrypted?: string;
  }
  ```

  ### `transactionChanged`

  Emitted when a known transaction changes status.

  ```ts
  {
    hash: string;
    status: 'pending' | 'confirmed' | 'rejected' | 'dropped';
    receipt?: unknown;
  }
  ```

  ### `message`

  Generic event channel for provider-specific messages.

  ```ts
  {
    type: string;
    data: unknown;
  }
  ```

  Privacy implementations MAY use this event for proof generation progress:

  ```ts
  window.octra.on('message', (message) => {
    if (message.type === 'octra_proofProgress') {
      console.log(message.data);
    }
  });
  ```

  ## Errors

  ```ts
  interface OctraProviderError extends Error {
    code: number;
    data?: unknown;
  }
  ```

  ### Standard Error Codes

  | Code | Meaning |
  | --- | --- |
  | `4001` | User rejected the request. |
  | `4100` | The requested method/account is unauthorized. |
  | `4200` | The provider does not support the requested method. |
  | `4900` | The provider is disconnected from all Octra networks. |
  | `4901` | The provider cannot service the requested Octra network. |

  ### Octra Error Reasons

  Octra-specific reasons SHOULD appear in `data.reason`.

  ```txt
  wallet_locked
  invalid_address
  invalid_amount
  invalid_nonce
  insufficient_balance
  fee_too_low
  staging_full
  duplicate_transaction
  invalid_signature
  proof_generation_failed
  privacy_not_supported
  recipient_view_pubkey_missing
  ```

  Example:

  ```ts
  throw {
    code: 4100,
    message: 'Unauthorized',
    data: {
      reason: 'wallet_locked'
    }
  };
  ```

  ## Security Considerations

  Wallets MUST request user consent before exposing accounts.

  Wallets MUST require permission checks for signing, sending transactions, viewing encrypted balances, decrypting balances, private transfers, and stealth claims.

  Wallets SHOULD show the exact transaction recipient, amount, fee/OU, network, and staging/finality meaning before approval.

  Dapps MUST NOT assume `pending` means finalized.

  Wallets SHOULD treat all dapp-provided strings as untrusted input.

  Wallets SHOULD make native write RPC pass-through opt-in or confirmation-gated.

  Wallets SHOULD clearly distinguish public balance operations from encrypted/private balance operations.

  ## Minimal Example

  ```ts
  const accounts = await window.octra.request({
    method: 'octra_requestAccounts',
    params: [{
      permissions: ['read_address', 'send_transactions']
    }]
  });

  const balance = await window.octra.request({
    method: 'octra_balance',
    params: [accounts[0]]
  });

  const tx = await window.octra.request({
    method: 'octra_sendTransaction',
    params: [{
      to: 'oct...',
      amount: '1000000',
      fee: '1'
    }]
  });

  window.octra.on('transactionChanged', (update) => {
    if (update.hash === tx.hash) {
      console.log(update.status);
    }
  });
  ```

  ## References

  - [EIP-1193: Ethereum Provider JavaScript API](https://eips.ethereum.org/EIPS/eip-1193)
  - [Octra RPC scheme](https://octrascan.io/docs.html)
  - [Octra docs](http://docs.octra.org/)
  - [0xio SDK](https://github.com/0xio-xyz/0xio-sdk)
  - [octra-labs/webcli](https://github.com/octra-labs/webcli)

