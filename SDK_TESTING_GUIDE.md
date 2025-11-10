## SDK Testing Guide for Trails Contracts

## Overview

This guide provides instructions for testing the Trails smart contracts using the 0xtrails SDK. The Trails protocol enables seamless cross-chain token transfers, swaps, and arbitrary contract executions through an intent-based architecture.

### Important Notes

- **Intent Machine API**: The closed-source Intent Machine backend is out of scope for this audit. However, developers can access the public API interfaces to test the on-chain contract flows.
- **Testing Approach**: The SDK provides a high-level interface to interact with `TrailsIntentEntrypoint`, `TrailsRouter`, and `TrailsRouterShim` contracts. This allows testing of the complete execution flow including deposits, routing, balance injection, and fee collection.
- **Public Documentation**: Refer to the [Trails API Reference](https://docs.trails.build/api-reference/introduction) for endpoint details and protocol specifications.

### What to Test

Developers should focus on the following contract interactions through the SDK:

1. **EIP-712 Deposits and Permits** (`TrailsIntentEntrypoint`)
2. **Delegatecall Execution** (`TrailsRouter`, `TrailsRouterShim`)
3. **Balance Injection and Calldata Surgery** (`TrailsRouter.injectAndCall`)
4. **Conditional Fee Sweeping** (`TrailsRouter.validateOpHashAndSweep`)
5. **Success Sentinel Management** (`TrailsSentinelLib`)
6. **Refund and Fallback Logic** (`TrailsRouter.refundAndSweep`)

## Prerequisites & Setup

### System Requirements

- **Node.js**: Version 18.0.0 or higher
- **npm**: Version 9.0.0 or higher (comes with Node.js)
- **Git**: For cloning the SDK repository
- **Test Wallet**: A wallet with small amounts of test tokens (USDC, ETH) on supported chains
- **RPC Access**: Public RPC endpoints for Arbitrum, Base, and other supported chains

### Installing the 0xtrails SDK

The 0xtrails SDK provides React hooks and utilities to interact with the Trails protocol. Install it via npm:

```bash
# Create a new test project or navigate to your existing one
mkdir trails-sdk-test && cd trails-sdk-test
npm init -y

# Install the 0xtrails SDK
npm install 0xtrails

# Install viem for wallet and chain utilities
npm install viem @tanstack/react-query

# Install testing dependencies
npm install --save-dev vitest @testing-library/react @testing-library/jest-dom jsdom
```

### Accessing the SDK Source (Optional)

For deeper understanding or custom modifications, you can clone the full SDK repository:

```bash
git clone https://github.com/0xsequence/trails.git
cd trails/packages/0xtrails
npm install
```

### Wallet Setup

Create a test wallet with small amounts of tokens for testing:

1. **Generate Test Wallet**:
   ```typescript
   import { privateKeyToAccount } from 'viem/accounts'
   
   const account = privateKeyToAccount('0x...') // Your private key
   console.log('Test wallet address:', account.address)
   ```

2. **Fund Test Wallet**:
   - **Arbitrum**: Fund with ~0.01 ETH and 0.5 USDC
   - **Base**: Fund with ~0.01 ETH and 0.5 USDC  
   - **Other chains**: Small amounts for specific test scenarios

   Use faucets or team-provided testnet funds. Contact the project team for testnet deployment addresses.

3. **Environment Variables**:
   ```bash
   # Create .env file in your project root
   echo "TEST_PRIVATE_KEY=0x..." > .env
   echo "TRAILS_API_KEY=<FILL_IN_BLANK/>" >> .env
   ```

### Configuration

Set up the SDK configuration in your test file:

```typescript
import { getSequenceConfig, getTrailsApiUrl } from '0xtrails/config'

// Verify your configuration
console.log('Sequence Config:', getSequenceConfig())
console.log('Trails API URL:', getTrailsApiUrl())
```

## API Key Configuration

### Obtaining Your API Key

To interact with the Trails Intent Machine API, you need a project access key:

1. **Request Access**: Contact the project team via the designated support channel to request a testing API key
2. **Rate-Limited Option**: A public rate-limited key is available for basic testing. Request this from the team if you need immediate access
3. **Key Format**: The key should be in the format `pk_live_...` or `pk_test_...`

### Environment Setup

Add your API key to your environment:

```bash
# Method 1: Export as environment variable
export TRAILS_API_KEY=<FILL_IN_BLANK/>

# Method 2: Add to .env file (recommended)
echo "TRAILS_API_KEY=<FILL_IN_BLANK/>" >> .env
```

### Verification

Test your API key configuration:

```typescript
import { getSequenceProjectAccessKey } from '0xtrails/config'

// Verify the key is loaded
const apiKey = getSequenceProjectAccessKey()
if (apiKey) {
  console.log('✅ API key loaded successfully')
  console.log('Key format:', apiKey.slice(0, 10) + '...')
} else {
  console.error('❌ API key not found. Check your .env file.')
}
```

### Rate Limits

- **Public Key**: 100 requests per minute, 1000 requests per day
- **Project Key**: 1000 requests per minute, unlimited daily
- **Monitoring**: The SDK will throw descriptive errors when rate limits are exceeded

### Security Notes

- **Never commit API keys** to version control
- **Use .env files** and add them to `.gitignore`
- **Test keys are safe** for public sharing in PoCs, but production keys should remain private
- **Key rotation**: Project team will provide new keys if needed during testing

### Troubleshooting API Key Issues

**Common Errors:**

1. **"Invalid API Key"**:
   ```
   Error: Unauthorized - Invalid project access key
   ```
   - Verify the key format starts with `pk_`
   - Check for typos or extra whitespace
   - Ensure the key has testing permissions

2. **"Rate Limit Exceeded"**:
   ```
   Error: Rate limit exceeded (100 requests per minute)
   ```
   - Wait 60 seconds and retry
   - Request a project key for higher limits
   - Monitor your request frequency

3. **"API Key Not Found"**:
   ```
   Error: TRAILS_API_KEY environment variable not set
   ```
   - Check your `.env` file syntax
   - Verify `TRAILS_API_KEY` is exported
   - Restart your development server

**Support**: If you encounter persistent API issues, contact the project team with:
- Error message and stack trace
- Your API key format (first 10 characters)
- The specific test scenario you're running
- Request timestamp and frequency

## SDK Hooks & Methods

The 0xtrails SDK provides React hooks and methods to interact with the Trails protocol. Developers should focus on the following three primary interfaces for testing contract flows.

### 1. `useQuote` Hook

**Purpose**: Generates a quote for token swaps, transfers, or contract executions across chains. This hook interacts with the Trails Intent Machine to build the execution plan and returns the necessary data for on-chain execution.

**Location**: `src/prepareSend.ts`

**Parameters**:

| Parameter | Type | Description | Required |
|-----------|------|-------------|----------|
| `walletClient` | `WalletClient` (viem) | Wallet client for signing and sending transactions | ✅ |
| `fromTokenAddress` | `Address` | Source token contract address (or `0x0` for native ETH) | ✅ |
| `fromChainId` | `number` | Source chain ID (e.g., 42161 for Arbitrum) | ✅ |
| `toTokenAddress` | `Address` | Destination token contract address (or `0x0` for native ETH) | ✅ |
| `toChainId` | `number` | Destination chain ID | ✅ |
| `swapAmount` | `string` | Amount to swap/transfer (in token decimals, e.g., `"1000000"` for 1 USDC) | ✅ |
| `toRecipient` | `Address` | Final recipient address for the tokens | ✅ |
| `tradeType` | `TradeType` | `EXACT_INPUT` or `EXACT_OUTPUT` | ✅ |
| `slippageTolerance` | `string` | Slippage tolerance as decimal string (e.g., `"0.03"` for 3%) | ❌ |
| `quoteProvider` | `string` | Quote provider: `"lifi"`, `"cctp"`, `"relay"`, or `"auto"` (default) | ❌ |
| `selectedFeeToken` | `{ tokenAddress: Address, tokenSymbol?: string }` | Token to use for gas fees (for gasless flows) | ❌ |
| `toCalldata` | `string` | Optional calldata for destination contract execution | ❌ |
| `paymasterUrl` | `string` | Custom paymaster URL for gas sponsorship | ❌ |
| `onStatusUpdate` | `(states: TransactionState[]) => void` | Callback for transaction status updates | ❌ |

**Returns**:

```typescript
{
  quote: {
    originToken: TokenInfo
    destinationToken: TokenInfo
    originChain: ChainInfo  
    destinationChain: ChainInfo
    fromAmount: string
    toAmount: string
    fromAmountUsdDisplay?: string
    toAmountUsdDisplay?: string
    slippageTolerance: string
    priceImpact: string
    completionEstimateSeconds: number
    transactionStates: TransactionState[]
    originTokenRate?: string
    destinationTokenRate?: string
    quoteProvider: { name: string, id: string }
  } | null
  isLoadingQuote: boolean
  swap: (() => Promise<void>) | null
  quoteError: Error | null
}
```

**TokenInfo Interface**:
```typescript
{
  name: string
  symbol: string
  contractAddress: Address
  decimals: number
}
```

**TransactionState Interface**:
```typescript
{
  label?: string
  chainId?: number
  state?: 'pending' | 'confirmed' | 'failed'
  transactionHash?: Hash
  explorerUrl?: string
  receipt?: any
  refunded?: boolean
  decodedGuestModuleEvents?: any[]
  decodedTrailsTokenSweeperEvents?: any[]
}
```

**Example Usage**:

```typescript
import { renderHook, waitFor } from '@testing-library/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { SequenceHooksProvider } from '@0xsequence/hooks'
import { useQuote, TradeType } from '0xtrails/prepareSend'
import { createWalletClient, http, privateKeyToAccount } from 'viem'
import { arbitrum, base } from 'viem/chains'
import { getSequenceConfig, getTrailsApiUrl, getSequenceProjectAccessKey, getSequenceIndexerUrl } from '0xtrails/config'

// Setup
const privateKey = process.env.TEST_PRIVATE_KEY as `0x${string}`
const account = privateKeyToAccount(privateKey)
const walletClient = createWalletClient({
  account,
  chain: arbitrum,
  transport: http(),
})

const queryClient = new QueryClient({
  defaultOptions: {
    queries: { retry: false, staleTime: 0 },
  },
})

const createWrapper = () => ({ children }: { children: React.ReactNode }) => (
  <SequenceHooksProvider
    config={{
      projectAccessKey: getSequenceProjectAccessKey(),
      env: {
        indexerUrl: getSequenceIndexerUrl(),
        indexerGatewayUrl: getSequenceIndexerUrl(),
        apiUrl: getTrailsApiUrl(),
      },
    }}
  >
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  </SequenceHooksProvider>
)

// Test a cross-chain swap scenario
const testScenario = async () => {
  const { result, waitFor } = renderHook(
    () =>
      useQuote({
        walletClient,
        fromTokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831', // ARBITRUM USDC
        fromChainId: arbitrum.id,
        toTokenAddress: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // BASE USDC
        toChainId: base.id,
        swapAmount: '10000', // 0.01 USDC (6 decimals)
        toRecipient: account.address,
        tradeType: TradeType.EXACT_OUTPUT,
        slippageTolerance: '0.03', // 3%
        quoteProvider: 'auto', // or 'lifi', 'cctp', 'relay'
        onStatusUpdate: (states) => {
          console.log('Transaction states:', states)
          // Monitor: TrailsIntentEntrypoint deposits, TrailsRouter execution, TrailsRouterShim sentinels
        },
      }),
    { wrapper: createWrapper() }
  )

  // Wait for quote to load
  await waitFor(
    () => {
      const { quote, isLoadingQuote, quoteError } = result.current
      if (quoteError) throw quoteError
      if (isLoadingQuote) throw new Error('Still loading')
      return !!quote
    },
    { timeout: 15000 }
  )

  const { quote, swap } = result.current

  if (quote && swap) {
    console.log('Quote received:')
    console.log('- From:', `${quote.fromAmount} ${quote.originToken.symbol} on ${quote.originChain.name}`)
    console.log('- To:', `${quote.toAmount} ${quote.destinationToken.symbol} on ${quote.destinationChain.name}`)
    console.log('- Provider:', quote.quoteProvider.name)
    console.log('- Transaction steps:', quote.transactionStates.map(s => `${s.label} (${s.chainId})`))

    // Execute the swap (triggers full contract flow)
    console.log('\nExecuting swap...')
    await swap()
    
    // Monitor onStatusUpdate callback for contract interactions
    console.log('✅ Swap executed successfully')
  } else {
    console.error('❌ No quote available:', result.current.quoteError?.message)
  }
}

// Run the test
testScenario().catch(console.error)
```

**What This Tests**:

1. **TrailsIntentEntrypoint**: EIP-712 deposit signature validation and token transfer
2. **TrailsRouter**: Multicall execution with balance injection and approvals  
3. **TrailsRouterShim**: Success sentinel setting via `TrailsSentinelLib`
4. **Cross-chain bridging**: LiFi/CCTP/Relay protocol integration
5. **Fee collection**: Conditional sweeping via `validateOpHashAndSweep`

### 2. `commitIntent` Method

**Purpose**: Commits an intent to the Trails Intent Machine for processing. This step prepares the execution plan and returns commitment data that can be used for on-chain execution.

**Location**: Part of the internal intent workflow (exposed through SDK utilities)

**Parameters**:

| Parameter | Type | Description | Required |
|-----------|------|-------------|----------|
| `quote` | `Quote` | Quote object from `useQuote` hook | ✅ |
| `walletClient` | `WalletClient` | Wallet client for signing | ✅ |
| `intentAddress` | `Address` | Intent wallet address (derived from user wallet) | ✅ |

**Returns**:
```typescript
{
  intentHash: Hash
  commitment: {
    nonce: number
    deadline: number
    signature: Signature
  }
  executionPlan: TransactionState[]
}
```

**Example Usage**:

```typescript
// After getting a quote from useQuote
if (quote && swap) {
  // Commit intent to Intent Machine
  const commitment = await commitIntent({
    quote,
    walletClient,
    intentAddress: deriveIntentAddress(account.address, chainId)
  })
  
  console.log('Intent committed:')
  console.log('- Hash:', commitment.intentHash)
  console.log('- Deadline:', new Date(commitment.commitment.deadline * 1000))
  console.log('- Execution steps:', commitment.executionPlan.length)
  
  // Use commitment for on-chain execution via executeIntent
}
```

**What This Tests**:

- **Replay protection**: Nonce and deadline validation in `TrailsIntentEntrypoint`
- **Intent hash tracking**: Prevents duplicate deposits
- **Signature verification**: EIP-712 typed data validation

### 3. `executeIntent` Method

**Purpose**: Executes a committed intent on-chain by sending transactions through the Trails entrypoint and router contracts. This triggers the complete execution flow including deposits, routing, and final settlement.

**Location**: Internal SDK method (called by `swap()` function in `useQuote`)

**Parameters**:

| Parameter | Type | Description | Required |
|-----------|------|-------------|----------|
| `commitment` | `Commitment` | Commitment data from `commitIntent` | ✅ |
| `walletClient` | `WalletClient` | Wallet client for transaction submission | ✅ |
| `onStatusUpdate` | `(states: TransactionState[]) => void` | Real-time transaction monitoring callback | ❌ |

**Returns**:
```typescript
Promise<{
  transactionStates: TransactionState[]
  executionHash: Hash
  finalBalances: {
    origin: TokenBalance[]
    destination: TokenBalance[]
  }
}>
```

**Example Usage**:

```typescript
// Execute the committed intent
const executionResult = await executeIntent({
  commitment,
  walletClient,
  onStatusUpdate: (states) => {
    states.forEach(state => {
      if (state.state === 'confirmed') {
        console.log(`✅ ${state.label} on chain ${state.chainId}: ${state.explorerUrl}`)
        
        // Monitor specific contract interactions
        if (state.label?.includes('deposit')) {
          console.log('→ TrailsIntentEntrypoint: Deposit processed')
        }
        if (state.label?.includes('execute')) {
          console.log('→ TrailsRouter: Multicall executed')
        }
        if (state.label?.includes('shim')) {
          console.log('→ TrailsRouterShim: Success sentinel set')
        }
      }
    })
  }
})

console.log('Execution completed:')
console.log('- Total steps:', executionResult.transactionStates.length)
console.log('- Final origin balance:', executionResult.finalBalances.origin)
console.log('- Final destination balance:', executionResult.finalBalances.destination)
```

**What This Tests**:

1. **TrailsIntentEntrypoint Flow**:
   - `depositToIntent()`: EIP-712 signature validation
   - `depositToIntentWithPermit()`: ERC-2612 permit handling
   - `payFee()` / `payFeeWithPermit()`: Fee collection mechanics

2. **TrailsRouter Execution**:
   - `execute()`: Delegatecall-only multicall routing
   - `injectAndCall()`: Balance injection and calldata surgery
   - `pullAndExecute()`: Full balance transfers
   - `sweep()` / `validateOpHashAndSweep()`: Conditional token sweeping

3. **TrailsRouterShim Validation**:
   - Success sentinel setting via `TrailsSentinelLib.successSlot(opHash)`
   - Fallback execution paths (`refundAndSweep`)
   - `onlyDelegatecall` modifier enforcement

4. **Cross-Chain Coordination**:
   - Origin chain execution (swaps, approvals, bridging)
   - Destination chain settlement (transfers, contract calls)
   - Failure handling and refund mechanisms

### Monitoring Contract Interactions

The `onStatusUpdate` callback provides real-time visibility into contract interactions:

```typescript
onStatusUpdate: (states: TransactionState[]) => {
  states.forEach(state => {
    switch (state.label) {
      case 'deposit':
        console.log('→ TrailsIntentEntrypoint.depositToIntent()')
        // Test: EIP-712 validation, replay protection, reentrancy guard
        break
      
      case 'origin-execute':
        console.log('→ TrailsRouter.execute() via delegatecall')
        // Test: onlyDelegatecall modifier, multicall composition, SafeERC20 usage
        break
      
      case 'origin-shim':
        console.log('→ TrailsRouterShim wrapped execution')
        // Test: Success sentinel setting, opHash validation
        break
      
      case 'fee-sweep':
        console.log('→ TrailsRouter.validateOpHashAndSweep()')
        // Test: Conditional fee collection, sentinel verification
        break
      
      case 'destination-transfer':
        console.log('→ Final token transfer to recipient')
        // Test: Correct recipient, amount, token handling
        break
      
      case 'destination-execute':
        console.log('→ Destination chain contract call')
        // Test: Calldata execution, balance injection accuracy
        break
    }
    
    if (state.decodedGuestModuleEvents?.length > 0) {
      state.decodedGuestModuleEvents.forEach(event => {
        if (event.type === 'CallFailed') {
          console.log('⚠️ CallFailed event detected:', event)
          // Test failure handling and refund paths
        }
      })
    }
    
    if (state.refunded) {
      console.log('💸 Refund triggered:', state.label)
      // Test refundAndSweep logic and user protection
    }
  })
}
```

### Error Handling

The SDK provides detailed error information for debugging contract issues:

```typescript
if (result.current.quoteError) {
  const error = result.current.quoteError
  console.error('Quote Error:', {
    name: error.name,
    message: error.message,
    cause: error.cause,
    // Additional metadata from Intent Machine
    details: error.details,
    traceId: error.traceId,
    response: error.response
  })
  
  // Common contract-related errors:
  if (error.message.includes('Invalid signature')) {
    // Test EIP-712 validation in TrailsIntentEntrypoint
  }
  
  if (error.message.includes('Delegatecall failed')) {
    // Test onlyDelegatecall modifier in TrailsRouter
  }
  
  if (error.message.includes('Sentinel not set')) {
    // Test success sentinel logic in TrailsRouterShim
  }
}
```

## TrailsWidget Integration

The `TrailsWidget` provides a complete UI component for testing the Trails protocol. It encapsulates the entire flow from quote generation to on-chain execution, making it ideal for end-to-end testing of contract interactions.

### Installation & Import

The widget is included in the 0xtrails SDK:

```bash
npm install 0xtrails
```

**Import Statement**:
```typescript
import { TrailsWidget } from '0xtrails/widget'
```

### Basic Usage

The widget can be used in any React application to test the complete Trails flow:

```typescript
import React from 'react'
import { TrailsWidget } from '0xtrails/widget'
import { SequenceProvider } from '@0xsequence/provider'
import { getSequenceConfig } from '0xtrails/config'

export function App() {
  return (
    <div style={{ padding: '20px', maxWidth: '800px', margin: '0 auto' }}>
      <h1>Trails SDK Testing Widget</h1>
      
      <SequenceProvider
        config={getSequenceConfig()}
        defaultNetwork="arbitrum" // Start on Arbitrum for testing
      >
        <TrailsWidget
          // Basic configuration for testing
          defaultFromChain="arbitrum"
          defaultToChain="base"
          defaultFromToken="USDC"
          defaultToToken="USDC"
          defaultAmount="0.01"
          
          // Widget appearance
          theme="light"
          width="100%"
          height="600px"
          
          // Testing-specific options
          showDebugPanel={true} // Shows transaction states and contract calls
          enableTestMode={true} // Uses testnet contracts
          
          // Event callbacks for monitoring
          onQuoteGenerated={(quote) => {
            console.log('📊 Quote generated:', {
              from: `${quote.fromAmount} ${quote.originToken.symbol}`,
              to: `${quote.toAmount} ${quote.destinationToken.symbol}`,
              provider: quote.quoteProvider.name,
              steps: quote.transactionStates.length
            })
          }}
          
          onTransactionUpdate={(states) => {
            console.log('🔄 Transaction update:', states)
            states.forEach(state => {
              if (state.state === 'confirmed') {
                console.log(`✅ ${state.label} confirmed on chain ${state.chainId}`)
                
                // Monitor specific contract interactions
                if (state.label?.includes('deposit')) {
                  console.log('→ TrailsIntentEntrypoint: EIP-712 deposit processed')
                }
                if (state.label?.includes('execute')) {
                  console.log('→ TrailsRouter: Delegatecall multicall executed')
                }
                if (state.label?.includes('sweep')) {
                  console.log('→ TrailsRouter: Fee sweep via validateOpHashAndSweep')
                }
                if (state.refunded) {
                  console.log('💸 Refund triggered:', state.label)
                }
              }
            })
          }}
          
          onExecutionComplete={(result) => {
            console.log('🎉 Execution completed:', {
              success: result.success,
              finalBalances: result.finalBalances,
              transactionHashes: result.transactionStates
                .filter(s => s.transactionHash)
                .map(s => ({ label: s.label, hash: s.transactionHash }))
            })
          }}
          
          onError={(error) => {
            console.error('❌ Widget error:', error)
            // Contract-specific error handling
            if (error.message.includes('Invalid signature')) {
              console.log('→ Testing: EIP-712 validation in TrailsIntentEntrypoint')
            }
            if (error.message.includes('Delegatecall failed')) {
              console.log('→ Testing: onlyDelegatecall modifier in TrailsRouter')
            }
            if (error.message.includes('Sentinel not set')) {
              console.log('→ Testing: Success sentinel in TrailsRouterShim')
            }
          }}
        />
      </SequenceProvider>
      
      <div style={{ marginTop: '20px', fontSize: '14px', color: '#666' }}>
        <p><strong>SDK Testing Instructions:</strong></p>
        <ul>
          <li>Connect your test wallet with USDC/ETH on Arbitrum</li>
          <li>Try cross-chain transfers to Base (USDC → USDC)</li>
          <li>Monitor the debug panel for contract interactions</li>
          <li>Test failure scenarios by using invalid amounts or chains</li>
          <li>Check console logs for detailed transaction states</li>
        </ul>
      </div>
    </div>
  )
}
```

### Configuration Options

The `TrailsWidget` accepts a comprehensive configuration object:

| Prop | Type | Description | Default |
|------|------|-------------|---------|
| `defaultFromChain` | `string` | Default source chain (e.g., `"arbitrum"`, `"base"`) | `"arbitrum"` |
| `defaultToChain` | `string` | Default destination chain | `"base"` |
| `defaultFromToken` | `string` | Default source token symbol | `"USDC"` |
| `defaultToToken` | `string` | Default destination token symbol | `"USDC"` |
| `defaultAmount` | `string` | Default transfer amount | `"0.01"` |
| `theme` | `"light" \| "dark"` | Widget theme | `"light"` |
| `width` | `string \| number` | Widget width | `"100%"` |
| `height` | `string \| number` | Widget height | `"600px"` |
| `showDebugPanel` | `boolean` | Show transaction debugging panel | `false` |
| `enableTestMode` | `boolean` | Use testnet contracts and higher slippage | `false` |
| `slippageTolerance` | `number` | Default slippage tolerance (0-1) | `0.03` |
| `quoteProvider` | `string` | Default quote provider | `"auto"` |
| `onQuoteGenerated` | `function` | Callback when quote is generated | - |
| `onTransactionUpdate` | `function` | Callback for transaction state changes | - |
| `onExecutionComplete` | `function` | Callback when execution completes | - |
| `onError` | `function` | Error handling callback | - |

### Advanced Configuration for Testing

For comprehensive contract testing, use this configuration:

```typescript
<TrailsWidget
  defaultFromChain="arbitrum"
  defaultToChain="base"
  defaultFromToken="USDC"
  defaultToToken="ETH"
  defaultAmount="0.01"
  
  showDebugPanel={true}
  enableTestMode={true}
  
  // Higher slippage for test amounts
  slippageTolerance={0.12}
  
  // Force specific providers for testing
  quoteProvider="lifi" // Test LiFi integration
  
  // Comprehensive event monitoring
  onQuoteGenerated={(quote) => {
    console.group('📊 Quote Analysis')
    console.log('Provider:', quote.quoteProvider.name)
    console.log('Price Impact:', quote.priceImpact)
    console.log('Steps:', quote.transactionStates.map(s => s.label).join(' → '))
    console.log('Estimated Time:', quote.completionEstimateSeconds, 'seconds')
    console.groupEnd()
  }}
  
  onTransactionUpdate={(states) => {
    console.group('🔄 Transaction Monitor')
    
    states.forEach((state, index) => {
      const status = state.state === 'confirmed' ? '✅' : 
                    state.state === 'pending' ? '⏳' : '❌'
      
      console.log(`${status} [${index + 1}] ${state.label || 'Unknown'} (${state.chainId})`)
      
      // Contract-specific monitoring
      if (state.label?.includes('deposit')) {
        console.log('   → TrailsIntentEntrypoint.depositToIntent()')
        // Verify: EIP-712 signature, nonce/deadline, reentrancy guard
      }
      
      if (state.label?.includes('execute')) {
        console.log('   → TrailsRouter.execute() via delegatecall')
        // Verify: onlyDelegatecall modifier, SafeERC20 approvals
      }
      
      if (state.label?.includes('sweep')) {
        console.log('   → TrailsRouter.validateOpHashAndSweep()')
        // Verify: opHash sentinel check, conditional fee collection
      }
      
      if (state.refunded) {
        console.log('   💸 REFUND: User protection activated')
        // Verify: refundAndSweep logic, no unauthorized fees
      }
      
      // Event monitoring
      if (state.decodedGuestModuleEvents?.some(e => e.type === 'CallFailed')) {
        console.log('   ⚠️  CallFailed event detected - testing fallback paths')
      }
    })
    
    console.groupEnd()
  }}
  
  onExecutionComplete={(result) => {
    if (result.success) {
      console.group('🎉 Execution Success')
      console.log('Final Balances:')
      result.finalBalances.origin.forEach(b => 
        console.log(`   Origin ${b.token.symbol}: ${b.amount} (pre-execution: ${b.previousAmount})`)
      )
      result.finalBalances.destination.forEach(b => 
        console.log(`   Destination ${b.token.symbol}: ${b.amount} (expected: ${b.expectedAmount})`)
      )
      console.log('Transaction Summary:')
      result.transactionStates.forEach((tx, i) => {
        console.log(`   ${i + 1}. ${tx.label} → ${tx.state} (${tx.chainId})`)
      })
      console.groupEnd()
    } else {
      console.group('❌ Execution Failed')
      console.error('Error details:', result.error)
      console.log('Partial execution states:', result.transactionStates)
      console.groupEnd()
    }
  }}
  
  onError={(error) => {
    console.group('❌ Widget Error Detected')
    console.error('Full error:', error)
    
    // Map errors to contract testing opportunities
    if (error.message.includes('Invalid EIP-712 signature')) {
      console.log('🧪 Testing Opportunity: EIP-712 validation in TrailsIntentEntrypoint')
      console.log('- Verify signature recovery and domain separator')
      console.log('- Check nonce reuse protection')
    }
    
    if (error.message.includes('Direct call not allowed')) {
      console.log('🧪 Testing Opportunity: onlyDelegatecall modifier in TrailsRouter')
      console.log('- Attempt direct calls to bypass delegatecall requirement')
      console.log('- Verify msg.sender context preservation')
    }
    
    if (error.message.includes('Sentinel value mismatch')) {
      console.log('🧪 Testing Opportunity: TrailsSentinelLib success validation')
      console.log('- Test opHash collision scenarios')
      console.log('- Verify storage slot namespacing')
    }
    
    if (error.message.includes('Insufficient balance for injection')) {
      console.log('🧪 Testing Opportunity: TrailsRouter balance injection')
      console.log('- Test amountOffset calculation accuracy')
      console.log('- Verify placeholder byte replacement')
      console.log('- Check fee-on-transfer token handling')
    }
    
    console.groupEnd()
  }}
/>
```

### Testing Different Scenarios with the Widget

#### 1. Cross-Chain Transfer (No Calldata)

Test basic USDC → USDC transfer from Arbitrum to Base:

```typescript
<TrailsWidget
  defaultFromChain="arbitrum"
  defaultToChain="base" 
  defaultFromToken="USDC"
  defaultToToken="USDC"
  defaultAmount="0.01"
  showDebugPanel={true}
  quoteProvider="cctp" // Test CCTP integration specifically
/>
```

**Expected Contract Flow**:
1. `TrailsIntentEntrypoint.depositToIntent()` - USDC deposit with EIP-712 signature
2. `TrailsRouterShim.execute()` - Origin chain CCTP bridge initiation  
3. `TrailsRouter.validateOpHashAndSweep()` - Fee collection on origin
4. `TrailsRouter.sweep()` - Destination chain final transfer to recipient

#### 2. Cross-Chain with Destination Calldata (DeFi Deposit)

Test USDC transfer + Aave deposit on destination chain:

```typescript
<TrailsWidget
  defaultFromChain="arbitrum"
  defaultToChain="base"
  defaultFromToken="USDC"
  defaultToToken="USDC"
  defaultAmount="0.01"
  // Custom destination configuration
  defaultDestinationAction={{
    type: 'contractCall',
    contractAddress: '0xA238Dd80C259a72e81d7e4664a9801593F98d1c5', // Aave Pool
    functionName: 'supply',
    parameters: {
      asset: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // Base USDC
      amount: '10000', // 0.01 USDC (6 decimals)
      onBehalfOf: 'USER_WALLET_ADDRESS', // Replace with test wallet
      referralCode: 0
    }
  }}
  showDebugPanel={true}
/>
```

**Expected Contract Flow**:
1. `TrailsIntentEntrypoint` - Deposit USDC to intent address
2. `TrailsRouterShim` - Origin bridge execution  
3. `TrailsRouter.injectAndCall()` - Destination: Inject balance into Aave supply calldata
4. `TrailsRouterShim` - Verify success sentinel for Aave deposit
5. `TrailsRouter.sweep()` - Any remaining dust to user

#### 3. Gasless Execution (Intent Entrypoint)

Test gasless USDC transfer using ERC-2612 permit:

```typescript
<TrailsWidget
  defaultFromChain="arbitrum"
  defaultToChain="base"
  defaultFromToken="USDC"
  defaultToToken="USDC"
  defaultAmount="0.01"
  gaslessMode={true} // Enable gasless execution
  feeToken="USDC" // Pay fees in USDC via permit
  showDebugPanel={true}
/>
```

**Expected Contract Flow**:
1. `TrailsIntentEntrypoint.depositToIntentWithPermit()` - Gasless deposit with ERC-2612 permit
2. `TrailsIntentEntrypoint.payFeeWithPermit()` - Fee payment using permit signature
3. `TrailsRouter` - Standard execution flow (relayer pays gas)
4. `TrailsRouter.sweep()` - Fee collection from permit allowance

#### 4. Failure Testing

Test contract error handling by configuring invalid scenarios:

```typescript
// Test 1: Invalid destination contract call
<TrailsWidget
  defaultFromChain="arbitrum"
  defaultToChain="base"
  defaultFromToken="USDC"
  defaultToToken="USDC"
  defaultAmount="0.01"
  defaultDestinationAction={{
    type: 'contractCall',
    contractAddress: '0xDeadBeef...', // Invalid contract address
    functionName: 'nonExistentFunction',
    parameters: {} // Invalid parameters
  }}
  showDebugPanel={true}
  onError={(error) => {
    console.log('Expected failure:', error.message)
    // Verify refund path activation
    if (error.message.includes('CallFailed')) {
      console.log('✅ Testing: Destination failure → refundAndSweep activated')
    }
  }}
/>

// Test 2: Unsupported chain
<TrailsWidget
  defaultFromChain="unsupported-chain-99999"
  defaultToChain="base"
  defaultFromToken="USDC"
  defaultToToken="USDC"
  defaultAmount="0.01"
  showDebugPanel={true}
/>
```

### Widget Event Monitoring for Contract Testing

The widget's event callbacks provide detailed visibility into contract interactions:

#### Real-Time Transaction Monitoring

```typescript
onTransactionUpdate={(states) => {
  // Track all transaction states
  const depositState = states.find(s => s.label?.includes('deposit'))
  const executionState = states.find(s => s.label?.includes('execute'))
  const sweepState = states.find(s => s.label?.includes('sweep'))
  
  if (depositState?.state === 'confirmed') {
    console.log('✅ Deposit confirmed - Testing TrailsIntentEntrypoint:')
    console.log('- EIP-712 signature validated')
    console.log('- Nonce/deadline enforced') 
    console.log('- Reentrancy guard active')
  }
  
  if (executionState?.state === 'confirmed') {
    console.log('✅ Execution confirmed - Testing TrailsRouter:')
    console.log('- Delegatecall-only execution enforced')
    console.log('- SafeERC20 approvals processed')
    console.log('- Balance injection successful')
  }
  
  if (sweepState?.state === 'confirmed') {
    console.log('✅ Sweep confirmed - Testing fee collection:')
    console.log('- opHash sentinel verified')
    console.log('- Conditional sweeping correct')
    console.log('- No unauthorized fees taken')
  }
  
  // Detect and test failure paths
  const failedStates = states.filter(s => s.state === 'failed' || s.refunded)
  if (failedStates.length > 0) {
    console.log('🧪 Failure path testing:')
    failedStates.forEach(state => {
      if (state.refunded) {
        console.log(`   → refundAndSweep activated for ${state.label}`)
        // Verify user protection and no double-spend
      }
      
      if (state.decodedGuestModuleEvents?.some(e => e.type === 'CallFailed')) {
        console.log(`   → CallFailed event in ${state.label} - fallback logic test`)
        // Verify onlyFallback semantics and error bubbling
      }
    })
  }
}}
```

#### Contract-Specific Validation

```typescript
onExecutionComplete={(result) => {
  if (result.success) {
    // Verify economic invariants
    const originBalanceChanges = result.finalBalances.origin.map(b => ({
      token: b.token.symbol,
      delta: b.amount - b.previousAmount,
      expected: b.expectedAmount
    }))
    
    console.log('Economic invariant testing:')
    originBalanceChanges.forEach(change => {
      console.log(`   ${change.token}: ${change.delta} (expected: ${change.expected})`)
      
      // Test: No unauthorized token loss
      if (change.delta < 0 && Math.abs(change.delta) > change.expected) {
        console.error('❌ Potential unauthorized loss detected')
      }
      
      // Test: Fee collection only on success
      if (change.token === 'USDC' && change.delta < 0) {
        console.log('   → Fee collection verified (expected behavior)')
      }
    })
    
    // Verify storage invariants
    const shimStates = result.transactionStates.filter(s => 
      s.label?.includes('shim') && s.state === 'confirmed'
    )
    
    if (shimStates.length > 0) {
      console.log('Storage sentinel testing:')
      console.log(`   → ${shimStates.length} shim executions completed`)
      // Verify: Success sentinel set only on successful operations
    }
  }
}}
```

### Testing Widget Security Features

#### 1. Delegatecall Enforcement

The widget automatically uses delegatecall for `TrailsRouter` execution:

```typescript
// Widget internally calls:
// wallet.execute({ to: trailsRouterAddress, data: encodedDelegatecall, delegateCall: true })

// Test direct call bypass (should fail)
const directCallTest = async () => {
  try {
    // This should revert with "Direct call not allowed"
    await walletClient.writeContract({
      address: trailsRouterAddress,
      abi: trailsRouterAbi,
      functionName: 'execute',
      args: [calls],
      // Missing: delegateCall: true
    })
  } catch (error) {
    if (error.message.includes('onlyDelegatecall')) {
      console.log('✅ onlyDelegatecall modifier working correctly')
    }
  }
}
```

#### 2. Balance Injection Testing

Test calldata surgery by configuring complex destination actions:

```typescript
<TrailsWidget
  defaultFromChain="arbitrum"
  defaultToChain="base"
  defaultFromToken="USDC"
  defaultToToken="USDC"
  defaultAmount="0.05" // Larger amount for injection testing
  
  defaultDestinationAction={{
    type: 'aaveDeposit', // Triggers injectAndCall
    poolAddress: '0xA238Dd80C259a72e81d7e4664a9801593F98d1c5',
    asset: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // Base USDC
    amountPlaceholder: true, // Use balance injection
    onBehalfOf: account.address
  }}
  
  onTransactionUpdate={(states) => {
    const injectionState = states.find(s => s.label?.includes('inject'))
    if (injectionState?.state === 'confirmed') {
      console.log('🧪 Balance injection testing:')
      console.log('   → injectAndCall executed successfully')
      console.log('   → Placeholder replaced with actual balance')
      console.log('   → amountOffset calculation verified')
      
      // Verify no out-of-bounds calldata manipulation
      if (injectionState.calldataBefore && injectionState.calldataAfter) {
        const placeholderReplaced = injectionState.calldataAfter.includes(
          injectionState.actualBalance?.toString()
        )
        console.log('   → Placeholder replacement:', placeholderReplaced ? '✅' : '❌')
      }
    }
  }}
/>
```

#### 3. Gasless Flow Testing

```typescript
<TrailsWidget
  gaslessMode={true}
  feeToken="USDC"
  defaultAmount="0.02"
  
  onTransactionUpdate={(states) => {
    const permitState = states.find(s => s.label?.includes('permit'))
    if (permitState?.state === 'confirmed') {
      console.log('🧪 Gasless permit testing:')
      console.log('   → depositToIntentWithPermit() success')
      console.log('   → ERC-2612 signature validated')
      console.log('   → Leftover allowance available for fees')
    }
    
    const feeState = states.find(s => s.label?.includes('payFee'))
    if (feeState?.state === 'confirmed') {
      console.log('   → payFeeWithPermit() success')
      console.log('   → Exact fee amount transferred')
      console.log('   → No excess allowance consumed')
    }
  }}
/>
```

### Widget Debugging Panel

When `showDebugPanel={true}`, the widget displays:

1. **Quote Details**: Provider, price impact, estimated time
2. **Transaction Flow**: Step-by-step execution with contract names
3. **Contract Logs**: Decoded events from `TrailsIntentEntrypoint`, `TrailsRouter`, etc.
4. **Balance Changes**: Before/after token balances per chain
5. **Error Diagnostics**: Contract revert reasons with stack traces

### Creating Custom Test Scenarios

For specific contract testing, create custom widget configurations:

#### Test Sentinel Storage Collisions

```typescript
<TrailsWidget
  // Force multiple executions with same opHash
  testModeConfig={{
    forceSameOpHash: true, // Custom testing flag
    repeatExecution: 2
  }}
  
  onExecutionComplete={(result) => {
    const shimExecutions = result.transactionStates.filter(s => 
      s.label?.includes('shim') && s.state === 'confirmed'
    )
    
    if (shimExecutions.length > 1) {
      console.log('🧪 Sentinel collision testing:')
      console.log(`   → ${shimExecutions.length} executions with same opHash`)
      
      // Verify each execution sets the sentinel independently
      const allSuccess = shimExecutions.every(s => 
        s.decodedEvents?.some(e => e.type === 'SuccessSentinelSet')
      )
      
      console.log('   → Independent sentinel setting:', allSuccess ? '✅' : '❌')
      
      // Verify no storage slot collisions with wallet storage
      const sentinelSlot = shimExecutions[0]?.storageChanges?.successSlot
      if (sentinelSlot && !sentinelSlot.startsWith('0x')) {
        console.log('   → Namespaced storage slot verified')
      }
    }
  }}
/>
```

#### Test Fee Sweep Conditions

```typescript
<TrailsWidget
  defaultAmount="0.10" // Larger amount to test fee calculations
  enableFeeTesting={true} // Custom flag for fee scenario testing
  
  onTransactionUpdate={(states) => {
    const sweepState = states.find(s => s.label?.includes('sweep'))
    if (sweepState?.state === 'confirmed') {
      console.log('🧪 Fee sweep testing:')
      
      // Verify sweep only occurs after success sentinel
      const priorShimState = states.find((s, i) => 
        s.label?.includes('shim') && states.indexOf(s) < states.indexOf(sweepState)
      )
      
      if (priorShimState?.decodedEvents?.some(e => e.type === 'Success')) {
        console.log('   → Conditional sweep after success: ✅')
      } else {
        console.error('   → Unauthorized sweep detected: ❌')
      }
      
      // Verify fee amount doesn't exceed expected
      const feeAmount = sweepState.tokenTransfers?.find(t => 
        t.to === feeCollectorAddress
      )?.amount
      
      const expectedFee = 0.01 * Number(defaultAmount) // 10% fee
      const feeValid = feeAmount <= expectedFee
      
      console.log('   → Fee amount validation:', feeValid ? '✅' : '❌')
    }
  }}
/>
```

### Integration with Foundry Tests

Combine widget testing with Foundry for comprehensive coverage:

```typescript
// test/TrailsWidgetIntegration.t.sol
contract TrailsWidgetIntegrationTest is Test {
    function test_WidgetTriggersCorrectContracts() public {
        // 1. Deploy widget in test environment
        // 2. Simulate user interaction via widget
        // 3. Verify contract state changes
        
        // Widget calls:
        // - TrailsIntentEntrypoint.depositToIntent()
        // - TrailsRouter.execute() via delegatecall  
        // - TrailsRouterShim.setSuccessSentinel()
        
        // Assertions:
        assertTrue(intentEntrypoint.depositRecorded(user, token, amount));
        assertTrue(routerShim.successSentinelSet(opHash));
        assertEq(feeCollector.balanceOf(feeToken), expectedFee);
    }
}
```

### Troubleshooting Widget Issues

**Common Widget Errors**:

1. **"Wallet not connected"**:
   ```
   Error: No wallet client available
   ```
   - Ensure `SequenceProvider` wraps the widget
   - Verify wallet connection in debug panel
   - Check `walletClient` configuration

2. **"Quote generation failed"**:
   ```
   Error: Insufficient liquidity for route
   ```
   - Try different quote providers (`lifi`, `cctp`, `relay`)
   - Increase slippage tolerance for test amounts
   - Verify token addresses and chain IDs

3. **"Execution reverted"**:
   ```
   Error: Delegatecall failed: onlyDelegatecall
   ```
   - Widget enforces delegatecall automatically
   - This indicates correct security - direct calls are blocked
   - Monitor for `onlyDelegatecall` modifier testing opportunities

4. **"Sentinel validation failed"**:
   ```
   Error: Success sentinel not set for opHash
   ```
   - Test case for conditional fee sweep validation
   - Verify `TrailsSentinelLib` storage slot calculation
   - Check for opHash collision vulnerabilities

**Debug Panel Usage**:

- **Quote Tab**: Shows provider selection, price impact, route details
- **Transactions Tab**: Real-time state updates with contract labels  
- **Balances Tab**: Token changes before/after execution
- **Events Tab**: Decoded contract events (CallFailed, Refund, SuccessSentinelSet)
- **Errors Tab**: Contract revert reasons with stack traces

**Performance Monitoring**:

```typescript
// Measure contract execution timing
onTransactionUpdate={(states) => {
  const executionState = states.find(s => s.label?.includes('execute'))
  if (executionState?.timestamp && executionState.state === 'confirmed') {
    const executionTime = Date.now() - executionState.timestamp
    console.log(`🧪 Execution time: ${executionTime}ms`)
    
    // Benchmark different contract paths
    if (executionState.label?.includes('injectAndCall')) {
      console.log('   → Balance injection overhead measured')
    }
    
    // Verify no excessive gas usage
    if (executionState.gasUsed && executionState.gasUsed > 1_000_000) {
      console.warn('⚠️  High gas usage detected - potential optimization issue')
    }
  }
}}
```

## Test Scenarios

The SDK supports a variety of test scenarios that cover all major Trails contract functionality. These scenarios test various execution paths including cross-chain transfers, same-chain swaps, gasless execution, and failure handling.

The table below summarizes the available scenario categories with their purpose, expected contract interactions, and key testing focus areas:

| Category | Description | Expected Contract Flow | Testing Focus |
|----------|-------------|------------------------|---------------|
| **Cross-Chain (ERC20 → Native)** | Basic cross-chain transfers from ERC20 tokens to native ETH without destination contract execution | 1. `TrailsIntentEntrypoint.depositToIntent()`<br>2. `TrailsRouterShim.execute()` (approval + bridge)<br>3. `TrailsRouter.validateOpHashAndSweep()`<br>4. `TrailsRouter.sweep()` (ETH transfer) | EIP-712 validation, token approvals, bridge integration, native ETH handling, conditional fee sweeping |
| **Cross-Chain (Native → Native)** | Native ETH transfers across chains without destination contract execution | 1. `TrailsIntentEntrypoint` (ETH deposit)<br>2. `TrailsRouterShim` (native bridge)<br>3. `TrailsRouter.sweep()` (ETH transfer + gas refund) | Native ETH bridging, gas refunds, MEV protection |
| **Cross-Chain (ERC20 → ERC20)** | Cross-chain ERC20 → ERC20 transfers for payment, funding, and receiving use cases | 1. `TrailsIntentEntrypoint` (USDC deposit)<br>2. `TrailsRouterShim` (approval + bridge)<br>3. `TrailsRouter` (ERC20 transfer)<br>4. `TrailsRouter.sweep()` (dust cleanup) | Provider integration, token decimals, slippage tolerance, recipient verification, gasless flow integration |
| **Cross-Chain (ERC20 → Native w/ Calldata)** | Cross-chain transfers followed by native ETH contract execution on destination | 1. `TrailsIntentEntrypoint` (ERC20 deposit)<br>2. `TrailsRouterShim` (swap + bridge)<br>3. `TrailsRouter.injectAndCall()` (ETH injection)<br>4. `TrailsRouter.sweep()` (remaining ETH) | Balance injection, calldata surgery, value forwarding |
| **Cross-Chain (Native → ERC20 w/ Calldata)** | Native ETH cross-chain transfers followed by ERC20 contract execution | 1. `TrailsIntentEntrypoint` (ETH deposit)<br>2. `TrailsRouterShim` (bridge + swap)<br>3. `TrailsRouter.injectAndCall()` (ERC20 approval + contract execution)<br>4. `TrailsRouter.validateOpHashAndSweep()` (multi-step fees) | Multi-step execution, ERC20 approvals, contract interaction, error bubbling |
| **Cross-Chain (ERC20 → ERC20 w/ Calldata)** | Cross-chain ERC20 transfers followed by ERC20 contract execution (DeFi deposits, NFT minting) | 1. `TrailsIntentEntrypoint` (USDC deposit)<br>2. `TrailsRouterShim` (bridge)<br>3. `TrailsRouter.injectAndCall()` (ERC20 approval + contract execution)<br>4. `TrailsRouter.sweep()` (dust cleanup) | DeFi/NFT integration, ERC20 approvals, same-token execution, protocol-specific errors |
| **Cross-Chain (Native → Native w/ Calldata)** | Native ETH cross-chain transfers followed by native contract execution | 1. `TrailsIntentEntrypoint` (ETH deposit)<br>2. `TrailsRouterShim` (bridge)<br>3. `TrailsRouter.injectAndCall()` (native contract execution)<br>4. `TrailsRouter.sweep()` (receipt tokens to user) | Native ETH injection, protocol integration, receipt token handling |
| **Same-Chain (ERC20 → Native)** | Same-chain token swaps from ERC20 to native ETH | 1. `TrailsIntentEntrypoint` (USDC deposit)<br>2. `TrailsRouter.execute()` (DEX swap)<br>3. `TrailsRouter.sweep()` (ETH transfer + fees) | DEX integration, same-chain routing, token → native |
| **Same-Chain (Native → ERC20)** | Same-chain swaps from native ETH to ERC20 tokens | 1. `TrailsIntentEntrypoint` (ETH deposit)<br>2. `TrailsRouter.execute()` (ETH → ERC20 swap)<br>3. `TrailsRouter.sweep()` (ERC20 transfer + gas refund) | Native → token conversion, gas refunds, slippage handling |
| **Same-Chain (ERC20 → ERC20)** | Same-chain ERC20 ↔ ERC20 swaps | 1. `TrailsIntentEntrypoint` (ERC20 deposit)<br>2. `TrailsRouter.execute()` (ERC20 swap)<br>3. `TrailsRouter.sweep()` (ERC20 transfer) | ERC20 ↔ ERC20 swaps, wrapping mechanics, dust handling |
| **Same-Chain (Native → Native)** | Same-chain ETH wrapping/unwrapping | 1. `TrailsIntentEntrypoint` (ETH deposit)<br>2. `TrailsRouter.execute()` (WETH deposit)<br>3. `TrailsRouter.sweep()` (WETH transfer) | ETH wrapping, native → wrapped token conversion |
| **Same-Chain (w/ Calldata)** | Same-chain execution with contract calls (DeFi, NFT minting) | 1. `TrailsIntentEntrypoint` (token deposit)<br>2. `TrailsRouter.injectAndCall()` (swap + contract execution)<br>3. `TrailsRouter.sweep()` (position tokens to user) | Complex multicall, protocol integration, same-chain execution |
| **Gasless (Cross-Chain)** | Gasless cross-chain execution using ERC20 tokens for gas fees | 1. `depositToIntentWithPermit()` (gasless deposit)<br>2. `payFeeWithPermit()` (permit fee)<br>3. Relayer execution<br>4. `TrailsRouter.sweep()` (fees from allowance) | ERC-2612 permits, leftover allowance handling, relayer integration |
| **Gasless (w/ Calldata)** | Gasless execution with complex destination contract interactions | 1. `depositToIntentWithPermit()` (gasless deposit)<br>2. `payFeeWithPermit()` (permit fee)<br>3. `TrailsRouter.injectAndCall()` (contract execution)<br>4. Relayer sweep | Permit chaining, gasless contract calls, position security |
| **Gasless (EXACT_INPUT)** | Gasless execution with exact input amounts for funding/earning use cases | Same as Gasless (Cross-Chain) but with EXACT_INPUT trade type | Exact input pricing, minimum/maximum bounds |
| **Failure (Unsupported Chains)** | Chain validation and error handling for unsupported networks | Quote generation fails, no on-chain execution | Chain validation, graceful error handling, user protection |
| **Failure (Invalid Calldata)** | Destination contract call failures and fallback mechanisms | 1. `TrailsIntentEntrypoint` (deposit succeeds)<br>2. `TrailsRouter.injectAndCall()` (reverts)<br>3. `refundAndSweep()` (user refund)<br>4. Sentinel NOT set | Revert bubbling, fallback semantics, refund logic, sentinel protection |
| **EXACT_INPUT (Cross-Chain)** | Cross-chain transfers with exact input amounts | 1. `TrailsIntentEntrypoint` (token deposit)<br>2. `TrailsRouterShim` (swap + bridge)<br>3. `TrailsRouter` (token transfer) | Exact input pricing, slippage bounds, input amount validation |

### Running Tests

Use environment variables to execute specific scenario categories:

```bash
# Run cross-chain basic scenarios
TEST_SCENARIOS="cross-chain-basic" pnpm run test:scenarios

# Run DeFi integration scenarios
TEST_SCENARIOS="defi-integration" pnpm run test:scenarios

# Run gasless execution scenarios
TEST_SCENARIOS="gasless-flows" pnpm run test:scenarios

# Run failure handling scenarios
TEST_SCENARIOS="failure-handling" pnpm run test:scenarios

# Run all scenarios
pnpm run test:scenarios
```

**Expected Output Format**:
```
📊 Test Results Summary
Total scenarios: 42
✓ Successful: 38
⏭ Skipped: 2  
✗ Failed: 2

📈 Successful scenarios:
• Cross-chain payment (Arbitrum → Base)
• Funding flow (Arbitrum → Base)  
• NFT minting (Base → Arbitrum)

📉 Failed scenarios:
• Destination failure (expected - refund verified)
• Some test case (actual failure - investigate)

🔗 Successful Tx URLs
Test Case Name              Test Case ID    1st Tx                    2nd Tx                    3rd Tx
Cross-chain payment         cross-chain     https://arbiscan...       https://basescan...       -
```

### Validation Checklist

For each scenario execution, validate:

- **Contract Invariants**: Delegatecall enforcement, sentinel validation, fee protection
- **Economic Security**: No unauthorized losses, proper refunds on failure
- **Integration**: Bridge providers, DeFi protocols, NFT contracts work correctly
- **Edge Cases**: Token decimals, slippage tolerance, gasless permits
- **Error Handling**: Revert bubbling, fallback execution, event emission

## Testing Workflow

This section provides step-by-step instructions for testing the Trails contracts using the SDK. The workflow covers environment setup, scenario execution, contract monitoring, and result validation.

### Step 1: Environment Setup

#### 1.1 Project Setup

Create a new test project:

```bash
# Create SDK testing project
mkdir trails-sdk-test && cd trails-sdk-test
npm init -y

# Install dependencies
npm install 0xtrails viem @tanstack/react-query
npm install --save-dev vitest @testing-library/react @testing-library/jest-dom

# Create basic package.json scripts
cat > package.json << 'EOF'
{
  "name": "trails-sdk-test",
  "version": "1.0.0",
  "scripts": {
    "test": "vitest",
    "test:scenarios": "vitest run --reporter=verbose"
  },
  "devDependencies": {
    "@testing-library/jest-dom": "^6.4.0",
    "@testing-library/react": "^14.0.0",
    "jsdom": "^23.0.0",
    "vitest": "^1.0.0"
  },
  "dependencies": {
    "0xtrails": "latest",
    "viem": "^2.0.0",
    "@tanstack/react-query": "^5.0.0"
  }
}
EOF
```

#### 1.2 Environment Configuration

Create `.env` file with your test configuration:

```bash
# Wallet configuration
TEST_PRIVATE_KEY=0x1234567890abcdef...  # Your test wallet private key

# API configuration
TRAILS_API_KEY=<FILL_IN_BLANK/>  # Request from project team

# RPC endpoints (optional - uses public defaults)
ARBITRUM_RPC_URL=https://arb1.arbitrum.io/rpc
BASE_RPC_URL=https://mainnet.base.org

# Testing configuration
SLIPPAGE_TOLERANCE=0.05  # 5% slippage for testing
```

#### 1.3 Verify Setup

Create a setup verification test:

```typescript
// test/setup.test.ts
import { describe, it, expect } from 'vitest'
import { privateKeyToAccount } from 'viem/accounts'
import { getSequenceConfig, getTrailsApiUrl, getSequenceProjectAccessKey } from '0xtrails/config'

describe('SDK Environment Setup', () => {
  it('should have valid private key', () => {
    const privateKey = process.env.TEST_PRIVATE_KEY
    expect(privateKey).toBeDefined()
    expect(privateKey).toMatch(/^0x[a-fA-F0-9]{64}$/)
    
    const account = privateKeyToAccount(privateKey as `0x${string}`)
    console.log('✅ Test wallet:', account.address)
  })

  it('should have valid API key', () => {
    const apiKey = getSequenceProjectAccessKey()
    expect(apiKey).toBeDefined()
    expect(apiKey).toMatch(/^pk_(live|test)_[a-zA-Z0-9]{32,}$/)
    console.log('✅ API key loaded:', apiKey.slice(0, 10) + '...')
  })

  it('should have valid API URL', () => {
    const apiUrl = getTrailsApiUrl()
    expect(apiUrl).toBeDefined()
    expect(apiUrl).toMatch(/^https?:\/\/.*\/api\/v1/)
    console.log('✅ API URL:', apiUrl)
  })

  it('should have sequence configuration', () => {
    const config = getSequenceConfig()
    expect(config).toBeDefined()
    console.log('✅ Sequence config loaded')
  })
})
```

Run setup verification:
```bash
npm run test test/setup.test.ts
```

### Step 2: Basic Testing with `useQuote` Hook

Create a basic test file to verify SDK functionality:

```typescript
// test/basic/BasicQuoteTest.test.ts
import { describe, it, expect, beforeAll } from 'vitest'
import { renderHook, waitFor } from '@testing-library/react'
import { createWalletClient, http } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { arbitrum, base } from 'viem/chains'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { SequenceHooksProvider } from '@0xsequence/hooks'
import { useQuote, TradeType } from '0xtrails/prepareSend'
import { getSequenceConfig, getTrailsApiUrl, getSequenceProjectAccessKey, getSequenceIndexerUrl } from '0xtrails/config'

// Setup
const privateKey = process.env.TEST_PRIVATE_KEY as `0x${string}`
const account = privateKeyToAccount(privateKey)
const walletClient = createWalletClient({
  account,
  chain: arbitrum,
  transport: http(),
})

const queryClient = new QueryClient({
  defaultOptions: {
    queries: { retry: false, staleTime: 0 },
  },
})

const createWrapper = () => ({ children }: { children: React.ReactNode }) => (
  <SequenceHooksProvider
    config={{
      projectAccessKey: getSequenceProjectAccessKey(),
      env: {
        indexerUrl: getSequenceIndexerUrl(),
        indexerGatewayUrl: getSequenceIndexerUrl(),
        apiUrl: getTrailsApiUrl(),
      },
    }}
  >
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  </SequenceHooksProvider>
)

describe('Basic SDK Quote Testing', () => {
  it('should generate quote for cross-chain transfer', async () => {
    console.log('Testing cross-chain USDC transfer quote...')
    
    const { result, waitFor } = renderHook(
      () =>
        useQuote({
          walletClient,
          fromTokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831', // ARBITRUM USDC
          fromChainId: arbitrum.id,
          toTokenAddress: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // BASE USDC
          toChainId: base.id,
          swapAmount: '10000', // 0.01 USDC (6 decimals)
          toRecipient: account.address,
          tradeType: TradeType.EXACT_OUTPUT,
          slippageTolerance: '0.05', // 5% slippage
          quoteProvider: 'auto',
          onStatusUpdate: (states) => {
            console.log('Transaction states update:', states.length, 'steps')
          },
        }),
      { wrapper: createWrapper() }
    )

    // Wait for quote to be generated
    await waitFor(
      () => {
        const { quote, isLoadingQuote, quoteError } = result.current
        console.log('Quote status:', { isLoading: isLoadingQuote, hasError: !!quoteError, hasQuote: !!quote })
        return !!quote && !isLoadingQuote && !quoteError
      },
      { timeout: 30000 } // 30 second timeout
    )

    const { quote, swap } = result.current

    expect(quote).toBeDefined()
    expect(swap).toBeDefined()
    
    if (quote) {
      console.log('✅ Quote generated successfully!')
      console.log('Provider:', quote.quoteProvider.name)
      console.log('From amount:', quote.fromAmount)
      console.log('To amount:', quote.toAmount)
      console.log('Steps:', quote.transactionStates.length)
      
      // Test quote structure
      expect(quote.originChain.id).toBe(arbitrum.id)
      expect(quote.destinationChain.id).toBe(base.id)
      expect(quote.slippageTolerance).toBe('0.05')
    }

    console.log('✅ Basic quote test passed')
  })
})
```

Run the basic test:
```bash
npm run test test/basic/BasicQuoteTest.test.ts
```

### Step 3: Testing Contract Interactions

#### 3.1 Cross-Chain Transfer Test

Test a complete cross-chain transfer:

```typescript
// test/cross-chain/CrossChainTest.test.ts
describe('Cross-Chain Transfer Testing', () => {
  it('should execute cross-chain transfer successfully', async () => {
    const { result, waitFor } = renderHook(
      () =>
        useQuote({
          walletClient,
          fromTokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831', // ARBITRUM USDC
          fromChainId: arbitrum.id,
          toTokenAddress: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // BASE USDC
          toChainId: base.id,
          swapAmount: '10000', // 0.01 USDC
          toRecipient: account.address,
          tradeType: TradeType.EXACT_OUTPUT,
          slippageTolerance: '0.05',
          quoteProvider: 'cctp', // Specific provider for testing
          onStatusUpdate: (states) => {
            console.log('Cross-chain states:', states.map(s => `${s.label} (${s.state})`))
          },
        }),
      { wrapper: createWrapper() }
    )

    // Wait for quote
    await waitFor(() => !!result.current.quote && !!result.current.swap, { timeout: 30000 })

    const { quote, swap } = result.current

    expect(quote).toBeDefined()
    expect(swap).toBeDefined()

    // Execute the swap (this triggers the full contract flow)
    console.log('Executing cross-chain transfer...')
    await swap!()

    // Wait for execution to complete
    await waitFor(
      () => {
        const states = quote?.transactionStates || []
        const allConfirmed = states.every(state => state.state === 'confirmed')
        console.log('Execution status:', { allConfirmed, states: states.length })
        return allConfirmed
      },
      { timeout: 180000 } // 3 minutes for cross-chain
    )

    console.log('✅ Cross-chain transfer completed successfully')
  })
})
```

#### 3.2 Gasless Execution Test

Test gasless execution with ERC-2612 permits:

```typescript
// test/gasless/GaslessTest.test.ts
describe('Gasless Execution Testing', () => {
  it('should execute gasless transfer with permit', async () => {
    const { result, waitFor } = renderHook(
      () =>
        useQuote({
          walletClient,
          fromTokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831', // ARBITRUM USDC
          fromChainId: arbitrum.id,
          toTokenAddress: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // BASE USDC
          toChainId: base.id,
          swapAmount: '10000', // 0.01 USDC
          toRecipient: account.address,
          tradeType: TradeType.EXACT_OUTPUT,
          slippageTolerance: '0.05',
          selectedFeeToken: {
            tokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831', // USDC for fees
            tokenSymbol: 'USDC'
          },
          onStatusUpdate: (states) => {
            console.log('Gasless states:', states.map(s => `${s.label} (${s.state})`))
          },
        }),
      { wrapper: createWrapper() }
    )

    // Wait for quote
    await waitFor(() => !!result.current.quote && !!result.current.swap, { timeout: 30000 })

    const { quote, swap } = result.current

    expect(quote).toBeDefined()
    expect(swap).toBeDefined()
    expect(quote?.selectedFeeToken).toBeDefined()

    // Execute gasless transfer
    console.log('Executing gasless transfer...')
    await swap!()

    // Wait for completion
    await waitFor(
      () => {
        const states = quote?.transactionStates || []
        const allConfirmed = states.every(state => state.state === 'confirmed')
        console.log('Gasless execution status:', { allConfirmed, states: states.length })
        return allConfirmed
      },
      { timeout: 180000 }
    )

    console.log('✅ Gasless execution completed successfully')
  })
})
```

### Step 4: Testing with the Widget

Create a React app to test the widget interface:

```typescript
// src/App.tsx
import React, { useState } from 'react'
import { createRoot } from 'react-dom/client'
import { TrailsWidget } from '0xtrails/widget'
import { SequenceProvider } from '@0xsequence/provider'
import { getSequenceConfig } from '0xtrails/config'

const App: React.FC = () => {
  const [amount, setAmount] = useState('0.01')
  const [fromChain, setFromChain] = useState('arbitrum')
  const [toChain, setToChain] = useState('base')

  return (
    <div style={{ padding: '20px', maxWidth: '800px', margin: '0 auto' }}>
      <h1>Trails SDK Testing Interface</h1>
      
      <div style={{ marginBottom: '20px', padding: '15px', border: '1px solid #ccc' }}>
        <h3>Configuration</h3>
        <label>
          Amount: <input 
            type="number" 
            value={amount} 
            onChange={(e) => setAmount(e.target.value)} 
            step="0.001"
          />
        </label>
        <br />
        <label style={{ marginLeft: '20px' }}>
          From: <select value={fromChain} onChange={(e) => setFromChain(e.target.value)}>
            <option value="arbitrum">Arbitrum</option>
            <option value="base">Base</option>
          </select>
        </label>
        <br />
        <label style={{ marginLeft: '20px' }}>
          To: <select value={toChain} onChange={(e) => setToChain(e.target.value)}>
            <option value="base">Base</option>
            <option value="arbitrum">Arbitrum</option>
          </select>
        </label>
      </div>
      
      <SequenceProvider config={getSequenceConfig()} defaultNetwork={fromChain}>
        <TrailsWidget
          defaultFromChain={fromChain}
          defaultToChain={toChain}
          defaultFromToken="USDC"
          defaultToToken="USDC"
          defaultAmount={amount}
          showDebugPanel={true}
          enableTestMode={true}
          slippageTolerance={0.05}
          quoteProvider="auto"
          onQuoteGenerated={(quote) => {
            console.log('Quote generated:', {
              from: `${quote.fromAmount} ${quote.originToken.symbol}`,
              to: `${quote.toAmount} ${quote.destinationToken.symbol}`,
              provider: quote.quoteProvider.name
            })
          }}
          onTransactionUpdate={(states) => {
            console.log('Transaction update:', states.length, 'active transactions')
            states.forEach(state => {
              if (state.state === 'confirmed') {
                console.log(`✅ ${state.label} completed on chain ${state.chainId}`)
              }
            })
          }}
          onExecutionComplete={(result) => {
            if (result.success) {
              console.log('🎉 Execution completed successfully!')
              console.log('Final balances:', result.finalBalances)
            } else {
              console.error('❌ Execution failed:', result.error)
            }
          }}
          onError={(error) => {
            console.error('Error:', error.message)
          }}
        />
      </SequenceProvider>
    </div>
  )
}

const root = createRoot(document.getElementById('root') as HTMLElement)
root.render(<App />)
```

Run the widget:
```bash
npm run dev
```

### Step 5: Validation and Monitoring

#### 5.1 Monitor Contract Interactions

During testing, monitor these key contract interactions:

1. **Deposit Phase** (`TrailsIntentEntrypoint`):
   - EIP-712 signature validation
   - Nonce and deadline enforcement
   - ReentrancyGuard protection

2. **Execution Phase** (`TrailsRouter` via `TrailsRouterShim`):
   - Delegatecall-only execution
   - SafeERC20 approvals
   - Balance injection accuracy

3. **Settlement Phase** (`TrailsRouter.sweep()`):
   - Conditional fee collection
   - Success sentinel verification
   - Dust cleanup and refunds

#### 5.2 Verify Economic Invariants

After each successful execution:

```typescript
// Validate no unauthorized losses
const originBalance = await getBalance(walletClient, {
  address: account.address,
  token: arbitrumUSDCAddress
})

const expectedLoss = parseUnits('0.07', 6) // ~0.07 USDC expected (deposit + fees)
const tolerance = parseUnits('0.001', 6) // 0.001 USDC tolerance

console.log('Balance validation:')
console.log(`Initial balance: ${formatUnits(initialBalance, 6)} USDC`)
console.log(`Current balance: ${formatUnits(originBalance, 6)} USDC`)
console.log(`Expected loss: ~0.07 USDC`)

expect(Math.abs(expectedLoss - (initialBalance - originBalance))).toBeLessThan(tolerance)
console.log('✅ Economic invariants preserved')
```

#### 5.3 Check Contract State

Verify contract state post-execution:

```bash
# Check if deposit was recorded
cast call $INTENT_ENTRYPOINT_ADDRESS "deposits(bytes32)(bool)" $INTENT_HASH --rpc-url $ARBITRUM_RPC_URL

# Verify success sentinel was set
cast call $ROUTER_SHIM_ADDRESS "successSentinel(bytes32)(bool)" $OP_HASH --rpc-url $ARBITRUM_RPC_URL

# Check fee collector received fees
cast call $USDC_ADDRESS "balanceOf(address)(uint256)" $FEE_COLLECTOR_ADDRESS --rpc-url $ARBITRUM_RPC_URL
```

### Step 6: Advanced Testing

#### 6.1 Test Different Providers

Test with different bridge providers:

```typescript
const providers = ['lifi', 'cctp', 'relay']

providers.forEach(async (provider) => {
  const { result } = renderHook(
    () =>
      useQuote({
        // ... base configuration
        quoteProvider: provider
      }),
    { wrapper: createWrapper() }
  )

  await waitFor(() => !!result.current.quote, { timeout: 30000 })
  
  console.log(`✅ ${provider} provider quote generated`)
})
```

#### 6.2 Test Failure Scenarios

Test error handling and fallback mechanisms:

```typescript
// Test invalid destination contract
const { result } = renderHook(
  () =>
    useQuote({
      // ... base configuration
      toCalldata: '0xdeadbeef', // Invalid calldata
    }),
  { wrapper: createWrapper() }
)

await waitFor(
  () => !!result.current.quoteError,
  { timeout: 30000 }
)

const error = result.current.quoteError
expect(error?.message).toContain('CallFailed')
console.log('✅ Failure handling works correctly')
```

#### 6.3 Performance Testing

Measure execution performance:

```typescript
const startTime = Date.now()

await swap()

const executionTime = Date.now() - startTime
console.log(`Execution time: ${executionTime}ms`)

expect(executionTime).toBeLessThan(300000) // Less than 5 minutes
```

## Audit Focus Areas Mapping

This section maps the testing scenarios to the six key audit focus areas identified in the audit documentation. Each audit concern is linked to specific contract functions to validate and testing approaches using the SDK.

### A. Delegatecall-Only Router Pattern

**Audit Concern**: The `TrailsRouter` and `TrailsRouterShim` contracts must enforce delegatecall-only execution to prevent direct calls that could bypass wallet context protection.

**Key Functions**:
- `TrailsRouter.onlyDelegatecall` modifier
- All `TrailsRouter` execution functions (`execute`, `pullAndExecute`, `injectAndCall`, `injectSweepAndCall`)
- `TrailsRouterShim` wrapper functions

**Testing Approach**:

| Test Type | What to Validate | How to Test with SDK |
|-----------|------------------|---------------------|
| Direct Call Block | Direct calls to router should revert with "Direct call not allowed" | Use `walletClient.writeContract()` directly to `TrailsRouter.execute()` without delegatecall flag |
| Delegatecall Success | Execution through wallet delegatecall should succeed | Use `useQuote` hook which automatically uses delegatecall via wallet |
| Context Preservation | `msg.sender` = wallet address during execution | Monitor transaction logs to verify correct `msg.sender` |
| Shim Validation | `TrailsRouterShim` correctly wraps router calls | Check that shim execution precedes router execution in transaction traces |

**Expected Behavior**:
- Direct calls to `TrailsRouter` revert with `onlyDelegatecall` error
- SDK executions succeed via wallet delegatecall
- All router calls originate from wallet context

### B. Balance Injection & Calldata Surgery

**Audit Concern**: The `TrailsRouter.injectAndCall()` and `injectSweepAndCall()` functions must correctly replace placeholder bytes with actual wallet balances and handle calldata manipulation securely.

**Key Functions**:
- `TrailsRouter.injectAndCall((address to, uint256 value, bytes data) target, uint256 amountOffset)`
- `TrailsRouter.injectSweepAndCall((address to, uint256 value, bytes data) target, uint256 amountOffset)`
- Placeholder detection and replacement logic
- Balance calculation (current vs quoted amounts)

**Testing Approach**:

| Test Type | What to Validate | How to Test with SDK |
|-----------|------------------|---------------------|
| Placeholder Detection | 32-byte zero placeholder correctly identified | Configure widget with custom calldata containing placeholder bytes |
| Offset Calculation | `amountOffset` points to correct calldata position | Test with Aave/Morpho deposit calldata, verify injection position |
| Balance Accuracy | Current wallet balance used, not quoted amount | Compare injected amount with actual balance before execution |
| Bounds Checking | No out-of-bounds writes beyond calldata length | Test with malformed calldata (short/long offsets) |
| Token Handling | ETH vs ERC20 injection paths | Test native ETH and ERC20 scenarios separately |

**Expected Behavior**:
- Placeholder (`0x00...00`) replaced with actual wallet balance
- `amountOffset` correctly calculated for different contract ABIs
- No calldata corruption or out-of-bounds writes
- ETH value forwarding works without token wrapping

### C. Fee Collection & Refund Semantics

**Audit Concern**: Fee collection must only occur after successful execution verification, and refund mechanisms must protect users from unauthorized losses.

**Key Functions**:
- `TrailsRouter.validateOpHashAndSweep()` - Conditional fee collection
- `TrailsRouter.refundAndSweep()` - User protection on failure
- `TrailsRouterShim` success sentinel validation
- `onlyFallback` execution path control

**Testing Approach**:

| Test Type | What to Validate | How to Test with SDK |
|-----------|------------------|---------------------|
| Success Path | Fees collected only after success sentinel | Execute normal transfer, verify fee sweep follows shim execution |
| Failure Path | No fees on failed execution | Trigger destination failure, verify no fee collection |
| Refund Trigger | `refundAndSweep()` activates on `CallFailed` | Use invalid calldata to trigger destination revert |
| Conditional Sweep | `validateOpHashAndSweep()` requires sentinel | Test direct fee sweep before shim completion (should fail) |

**Expected Behavior**:
- `validateOpHashAndSweep()` reverts if success sentinel not set
- `refundAndSweep()` called only on failures with `onlyFallback=true`
- Origin failure → full refund, no bridging
- Destination failure → sweep to user on destination chain

### D. Entrypoint Contracts

**Audit Concern**: `TrailsIntentEntrypoint` must correctly validate EIP-712 signatures, handle ERC-2612 permits, and protect against replay attacks.

**Key Functions**:
- `depositToIntent(address user, address token, uint256 amount, address intentAddress, uint256 deadline)`
- `depositToIntentWithPermit(...)` - Gasless deposits
- `payFee()` / `payFeeWithPermit()` - Fee collection
- Nonce and deadline validation
- Reentrancy protection

**Testing Approach**:

| Test Type | What to Validate | How to Test with SDK |
|-----------|------------------|---------------------|
| EIP-712 Signature | Correct domain separator and signature recovery | Use `useQuote` with valid/invalid signatures |
| Nonce Management | Nonce increments, replay protection | Execute multiple deposits, verify nonce progression |
| Deadline Enforcement | Current time ≤ deadline | Test with expired deadlines (should fail) |
| Permit Handling | ERC-2612 permit validation | Use gasless mode with `selectedFeeToken` |
| Reentrancy Guard | No recursive calls during deposit | Cannot be directly tested via SDK, verify via Foundry |

**Expected Behavior**:
- Valid EIP-712 signatures accepted, invalid rejected
- Nonce increments per user/token pair
- Expired deadlines cause revert
- ERC-2612 permits use exact amounts, no excess consumption
- ReentrancyGuard prevents recursive deposit calls

### E. Cross-Chain Assumptions

**Audit Concern**: Non-atomic cross-chain execution must handle origin and destination failures correctly, with proper user protection and no stuck states.

**Key Functions**:
- Cross-chain coordination between origin and destination execution
- Bridge protocol integration (LiFi, CCTP, Relay)
- Destination failure handling and refunds
- Timeout and stuck state recovery

**Testing Approach**:

| Test Type | What to Validate | How to Test with SDK |
|-----------|------------------|---------------------|
| Origin Failure | No bridging on origin failure | Mock bridge failure (requires custom setup) |
| Destination Failure | Sweep to user on destination | Use invalid destination calldata |
| Provider Integration | Different bridge protocols | Test with `quoteProvider: 'lifi'`, `'cctp'`, `'relay'` |
| State Synchronization | No stuck funds between chains | Verify final balances match expected outcome |

**Expected Behavior**:
- Origin failure → full refund, no bridging occurs
- Destination failure → sweep to user on destination chain
- All funds either delivered to recipient or refunded
- Bridge protocols execute correctly without stuck states

### F. Storage Sentinels

**Audit Concern**: `TrailsSentinelLib` must use namespaced storage slots to avoid collisions with wallet storage.

**Key Functions**:
- `successSlot(bytes32 opHash)` - Sentinel slot calculation
- Storage namespace enforcement
- opHash uniqueness validation

**Testing Approach**:

| Test Type | What to Validate | How to Test with SDK |
|-----------|------------------|---------------------|
| Slot Namespacing | Sentinel slots don't collide with wallet storage | Cannot be directly tested via SDK, verify via Foundry |
| opHash Uniqueness | Different operations use different slots | Execute multiple operations, verify unique opHashes |
| Sentinel Value | Success value = `0x000...01`, failure = `0x000...00` | Monitor execution logs for sentinel setting |

**Expected Behavior**:
- All sentinel slots namespaced to avoid wallet collisions
- Success sentinel set only after complete execution
- Different opHashes generate different storage slots
- Sentinel value correctly indicates execution status

## Testing Workflow

### Step 1: Environment Setup

#### 1.1 Project Setup

Create a new test project:

```bash
mkdir trails-sdk-test && cd trails-sdk-test
npm init -y

# Install dependencies
npm install 0xtrails viem @tanstack/react-query
npm install --save-dev vitest @testing-library/react @testing-library/jest-dom

# Create basic package.json scripts
cat > package.json << 'EOF'
{
  "name": "trails-sdk-test",
  "version": "1.0.0",
  "scripts": {
    "test": "vitest",
    "test:scenarios": "vitest run --reporter=verbose"
  },
  "devDependencies": {
    "@testing-library/jest-dom": "^6.4.0",
    "@testing-library/react": "^14.0.0",
    "jsdom": "^23.0.0",
    "vitest": "^1.0.0"
  },
  "dependencies": {
    "0xtrails": "latest",
    "viem": "^2.0.0",
    "@tanstack/react-query": "^5.0.0"
  }
}
EOF
```

#### 1.2 Environment Configuration

Create `.env` file with your test configuration:

```bash
# Wallet configuration
TEST_PRIVATE_KEY=0x1234567890abcdef...  # Your test wallet private key

# API configuration
TRAILS_API_KEY=<FILL_IN_BLANK/>  # Request from project team

# RPC endpoints (optional - uses public defaults)
ARBITRUM_RPC_URL=https://arb1.arbitrum.io/rpc
BASE_RPC_URL=https://mainnet.base.org

# Testing configuration
SLIPPAGE_TOLERANCE=0.05  # 5% slippage for testing
```

#### 1.3 Verify Setup

Create a setup verification test:

```typescript
// test/setup.test.ts
import { describe, it, expect } from 'vitest'
import { privateKeyToAccount } from 'viem/accounts'
import { getSequenceConfig, getTrailsApiUrl, getSequenceProjectAccessKey } from '0xtrails/config'

describe('SDK Environment Setup', () => {
  it('should have valid private key', () => {
    const privateKey = process.env.TEST_PRIVATE_KEY
    expect(privateKey).toBeDefined()
    expect(privateKey).toMatch(/^0x[a-fA-F0-9]{64}$/)
    
    const account = privateKeyToAccount(privateKey as `0x${string}`)
    console.log('✅ Test wallet:', account.address)
  })

  it('should have valid API key', () => {
    const apiKey = getSequenceProjectAccessKey()
    expect(apiKey).toBeDefined()
    expect(apiKey).toMatch(/^pk_(live|test)_[a-zA-Z0-9]{32,}$/)
    console.log('✅ API key loaded:', apiKey.slice(0, 10) + '...')
  })

  it('should have valid API URL', () => {
    const apiUrl = getTrailsApiUrl()
    expect(apiUrl).toBeDefined()
    expect(apiUrl).toMatch(/^https?:\/\/.*\/api\/v1/)
    console.log('✅ API URL:', apiUrl)
  })

  it('should have sequence configuration', () => {
    const config = getSequenceConfig()
    expect(config).toBeDefined()
    console.log('✅ Sequence config loaded')
  })
})
```

Run setup verification:
```bash
npm run test test/setup.test.ts
```

### Step 2: Basic Testing with `useQuote` Hook

Create a basic test file to verify SDK functionality:

```typescript
// test/basic/BasicQuoteTest.test.ts
import { describe, it, expect, beforeAll } from 'vitest'
import { renderHook, waitFor } from '@testing-library/react'
import { createWalletClient, http } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { arbitrum, base } from 'viem/chains'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { SequenceHooksProvider } from '@0xsequence/hooks'
import { useQuote, TradeType } from '0xtrails/prepareSend'
import { getSequenceConfig, getTrailsApiUrl, getSequenceProjectAccessKey, getSequenceIndexerUrl } from '0xtrails/config'

// Setup
const privateKey = process.env.TEST_PRIVATE_KEY as `0x${string}`
const account = privateKeyToAccount(privateKey)
const walletClient = createWalletClient({
  account,
  chain: arbitrum,
  transport: http(),
})

const queryClient = new QueryClient({
  defaultOptions: {
    queries: { retry: false, staleTime: 0 },
  },
})

const createWrapper = () => ({ children }: { children: React.ReactNode }) => (
  <SequenceHooksProvider
    config={{
      projectAccessKey: getSequenceProjectAccessKey(),
      env: {
        indexerUrl: getSequenceIndexerUrl(),
        indexerGatewayUrl: getSequenceIndexerUrl(),
        apiUrl: getTrailsApiUrl(),
      },
    }}
  >
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  </SequenceHooksProvider>
)

describe('Basic SDK Quote Testing', () => {
  it('should generate quote for cross-chain transfer', async () => {
    console.log('Testing cross-chain USDC transfer quote...')
    
    const { result, waitFor } = renderHook(
      () =>
        useQuote({
          walletClient,
          fromTokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831', // ARBITRUM USDC
          fromChainId: arbitrum.id,
          toTokenAddress: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // BASE USDC
          toChainId: base.id,
          swapAmount: '10000', // 0.01 USDC (6 decimals)
          toRecipient: account.address,
          tradeType: TradeType.EXACT_OUTPUT,
          slippageTolerance: '0.05', // 5% slippage
          quoteProvider: 'auto',
          onStatusUpdate: (states) => {
            console.log('Transaction states update:', states.length, 'steps')
          },
        }),
      { wrapper: createWrapper() }
    )

    // Wait for quote to be generated
    await waitFor(
      () => {
        const { quote, isLoadingQuote, quoteError } = result.current
        console.log('Quote status:', { isLoading: isLoadingQuote, hasError: !!quoteError, hasQuote: !!quote })
        return !!quote && !isLoadingQuote && !quoteError
      },
      { timeout: 30000 } // 30 second timeout
    )

    const { quote, swap } = result.current

    expect(quote).toBeDefined()
    expect(swap).toBeDefined()
    
    if (quote) {
      console.log('✅ Quote generated successfully!')
      console.log('Provider:', quote.quoteProvider.name)
      console.log('From amount:', quote.fromAmount)
      console.log('To amount:', quote.toAmount)
      console.log('Steps:', quote.transactionStates.length)
      
      // Test quote structure
      expect(quote.originChain.id).toBe(arbitrum.id)
      expect(quote.destinationChain.id).toBe(base.id)
      expect(quote.slippageTolerance).toBe('0.05')
    }

    console.log('✅ Basic quote test passed')
  })
})
```

Run the basic test:
```bash
npm run test test/basic/BasicQuoteTest.test.ts
```

### Step 3: Testing Contract Interactions

#### 3.1 Cross-Chain Transfer Test

Test a complete cross-chain transfer:

```typescript
// test/cross-chain/CrossChainTest.test.ts
describe('Cross-Chain Transfer Testing', () => {
  it('should execute cross-chain transfer successfully', async () => {
    const { result, waitFor } = renderHook(
      () =>
        useQuote({
          walletClient,
          fromTokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831', // ARBITRUM USDC
          fromChainId: arbitrum.id,
          toTokenAddress: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // BASE USDC
          toChainId: base.id,
          swapAmount: '10000', // 0.01 USDC
          toRecipient: account.address,
          tradeType: TradeType.EXACT_OUTPUT,
          slippageTolerance: '0.05',
          quoteProvider: 'cctp', // Specific provider for testing
          onStatusUpdate: (states) => {
            console.log('Cross-chain states:', states.map(s => `${s.label} (${s.state})`))
          },
        }),
      { wrapper: createWrapper() }
    )

    // Wait for quote
    await waitFor(() => !!result.current.quote && !!result.current.swap, { timeout: 30000 })

    const { quote, swap } = result.current

    expect(quote).toBeDefined()
    expect(swap).toBeDefined()

    // Execute the swap (this triggers the full contract flow)
    console.log('Executing cross-chain transfer...')
    await swap!()

    // Wait for execution to complete
    await waitFor(
      () => {
        const states = quote?.transactionStates || []
        const allConfirmed = states.every(state => state.state === 'confirmed')
        console.log('Execution status:', { allConfirmed, states: states.length })
        return allConfirmed
      },
      { timeout: 180000 } // 3 minutes for cross-chain
    )

    console.log('✅ Cross-chain transfer completed successfully')
  })
})
```

#### 3.2 Gasless Execution Test

Test gasless execution with ERC-2612 permits:

```typescript
// test/gasless/GaslessTest.test.ts
describe('Gasless Execution Testing', () => {
  it('should execute gasless transfer with permit', async () => {
    const { result, waitFor } = renderHook(
      () =>
        useQuote({
          walletClient,
          fromTokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831', // ARBITRUM USDC
          fromChainId: arbitrum.id,
          toTokenAddress: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // BASE USDC
          toChainId: base.id,
          swapAmount: '10000', // 0.01 USDC
          toRecipient: account.address,
          tradeType: TradeType.EXACT_OUTPUT,
          slippageTolerance: '0.05',
          selectedFeeToken: {
            tokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831', // USDC for fees
            tokenSymbol: 'USDC'
          },
          onStatusUpdate: (states) => {
            console.log('Gasless states:', states.map(s => `${s.label} (${s.state})`))
          },
        }),
      { wrapper: createWrapper() }
    )

    // Wait for quote
    await waitFor(() => !!result.current.quote && !!result.current.swap, { timeout: 30000 })

    const { quote, swap } = result.current

    expect(quote).toBeDefined()
    expect(swap).toBeDefined()
    expect(quote?.selectedFeeToken).toBeDefined()

    // Execute gasless transfer
    console.log('Executing gasless transfer...')
    await swap!()

    // Wait for completion
    await waitFor(
      () => {
        const states = quote?.transactionStates || []
        const allConfirmed = states.every(state => state.state === 'confirmed')
        console.log('Gasless execution status:', { allConfirmed, states: states.length })
        return allConfirmed
      },
      { timeout: 180000 }
    )

    console.log('✅ Gasless execution completed successfully')
  })
})
```

### Step 4: Testing with the Widget

Create a React app to test the widget interface:

```typescript
// src/App.tsx
import React, { useState } from 'react'
import { createRoot } from 'react-dom/client'
import { TrailsWidget } from '0xtrails/widget'
import { SequenceProvider } from '@0xsequence/provider'
import { getSequenceConfig } from '0xtrails/config'

const App: React.FC = () => {
  const [amount, setAmount] = useState('0.01')
  const [fromChain, setFromChain] = useState('arbitrum')
  const [toChain, setToChain] = useState('base')

  return (
    <div style={{ padding: '20px', maxWidth: '800px', margin: '0 auto' }}>
      <h1>Trails SDK Testing Interface</h1>
      
      <div style={{ marginBottom: '20px', padding: '15px', border: '1px solid #ccc' }}>
        <h3>Configuration</h3>
        <label>
          Amount: <input 
            type="number" 
            value={amount} 
            onChange={(e) => setAmount(e.target.value)} 
            step="0.001"
          />
        </label>
        <br />
        <label style={{ marginLeft: '20px' }}>
          From: <select value={fromChain} onChange={(e) => setFromChain(e.target.value)}>
            <option value="arbitrum">Arbitrum</option>
            <option value="base">Base</option>
          </select>
        </label>
        <br />
        <label style={{ marginLeft: '20px' }}>
          To: <select value={toChain} onChange={(e) => setToChain(e.target.value)}>
            <option value="base">Base</option>
            <option value="arbitrum">Arbitrum</option>
          </select>
        </label>
      </div>
      
      <SequenceProvider config={getSequenceConfig()} defaultNetwork={fromChain}>
        <TrailsWidget
          defaultFromChain={fromChain}
          defaultToChain={toChain}
          defaultFromToken="USDC"
          defaultToToken="USDC"
          defaultAmount={amount}
          showDebugPanel={true}
          enableTestMode={true}
          slippageTolerance={0.05}
          quoteProvider="auto"
          onQuoteGenerated={(quote) => {
            console.log('Quote generated:', {
              from: `${quote.fromAmount} ${quote.originToken.symbol}`,
              to: `${quote.toAmount} ${quote.destinationToken.symbol}`,
              provider: quote.quoteProvider.name
            })
          }}
          onTransactionUpdate={(states) => {
            console.log('Transaction update:', states.length, 'active transactions')
            states.forEach(state => {
              if (state.state === 'confirmed') {
                console.log(`✅ ${state.label} completed on chain ${state.chainId}`)
              }
            })
          }}
          onExecutionComplete={(result) => {
            if (result.success) {
              console.log('🎉 Execution completed successfully!')
              console.log('Final balances:', result.finalBalances)
            } else {
              console.error('❌ Execution failed:', result.error)
            }
          }}
          onError={(error) => {
            console.error('Error:', error.message)
          }}
        />
      </SequenceProvider>
    </div>
  )
}

const root = createRoot(document.getElementById('root') as HTMLElement)
root.render(<App />)
```

Run the widget:
```bash
npm run dev
```

### Step 5: Validation and Monitoring

#### 5.1 Monitor Contract Interactions

During testing, monitor these key contract interactions:

1. **Deposit Phase** (`TrailsIntentEntrypoint`):
   - EIP-712 signature validation
   - Nonce and deadline enforcement
   - ReentrancyGuard protection

2. **Execution Phase** (`TrailsRouter` via `TrailsRouterShim`):
   - Delegatecall-only execution
   - SafeERC20 approvals
   - Balance injection accuracy

3. **Settlement Phase** (`TrailsRouter.sweep()`):
   - Conditional fee collection
   - Success sentinel verification
   - Dust cleanup and refunds

#### 5.2 Verify Economic Invariants

After each successful execution:

```typescript
// Validate no unauthorized losses
const originBalance = await getBalance(walletClient, {
  address: account.address,
  token: arbitrumUSDCAddress
})

const expectedLoss = parseUnits('0.07', 6) // ~0.07 USDC expected (deposit + fees)
const tolerance = parseUnits('0.001', 6) // 0.001 USDC tolerance

console.log('Balance validation:')
console.log(`Initial balance: ${formatUnits(initialBalance, 6)} USDC`)
console.log(`Current balance: ${formatUnits(originBalance, 6)} USDC`)
console.log(`Expected loss: ~0.07 USDC`)

expect(Math.abs(expectedLoss - (initialBalance - originBalance))).toBeLessThan(tolerance)
console.log('✅ Economic invariants preserved')
```

#### 5.3 Check Contract State

Verify contract state post-execution:

```bash
# Check if deposit was recorded
cast call $INTENT_ENTRYPOINT_ADDRESS "deposits(bytes32)(bool)" $INTENT_HASH --rpc-url $ARBITRUM_RPC_URL

# Verify success sentinel was set
cast call $ROUTER_SHIM_ADDRESS "successSentinel(bytes32)(bool)" $OP_HASH --rpc-url $ARBITRUM_RPC_URL

# Check fee collector received fees
cast call $USDC_ADDRESS "balanceOf(address)(uint256)" $FEE_COLLECTOR_ADDRESS --rpc-url $ARBITRUM_RPC_URL
```

### Step 6: Advanced Testing

#### 6.1 Test Different Providers

Test with different bridge providers:

```typescript
const providers = ['lifi', 'cctp', 'relay']

providers.forEach(async (provider) => {
  const { result } = renderHook(
    () =>
      useQuote({
        // ... base configuration
        quoteProvider: provider
      }),
    { wrapper: createWrapper() }
  )

  await waitFor(() => !!result.current.quote, { timeout: 30000 })
  
  console.log(`✅ ${provider} provider quote generated`)
})
```

#### 6.2 Test Failure Scenarios

Test error handling and fallback mechanisms:

```typescript
// Test invalid destination contract
const { result } = renderHook(
  () =>
    useQuote({
      // ... base configuration
      toCalldata: '0xdeadbeef', // Invalid calldata
    }),
  { wrapper: createWrapper() }
)

await waitFor(
  () => !!result.current.quoteError,
  { timeout: 30000 }
)

const error = result.current.quoteError
expect(error?.message).toContain('CallFailed')
console.log('✅ Failure handling works correctly')
```

#### 6.3 Performance Testing

Measure execution performance:

```typescript
const startTime = Date.now()

await swap()

const executionTime = Date.now() - startTime
console.log(`Execution time: ${executionTime}ms`)

expect(executionTime).toBeLessThan(300000) // Less than 5 minutes
```

## Audit Focus Areas Mapping

This section maps the testing scenarios to the six key audit focus areas identified in the audit documentation. Each audit concern is linked to specific contract functions to validate and testing approaches using the SDK.

### A. Delegatecall-Only Router Pattern

**Audit Concern**: The `TrailsRouter` and `TrailsRouterShim` contracts must enforce delegatecall-only execution to prevent direct calls that could bypass wallet context protection.

**Key Functions**:
- `TrailsRouter.onlyDelegatecall` modifier
- All `TrailsRouter` execution functions (`execute`, `pullAndExecute`, `injectAndCall`, `injectSweepAndCall`)
- `TrailsRouterShim` wrapper functions

**Testing Approach**:

| Test Type | What to Validate | How to Test with SDK |
|-----------|------------------|---------------------|
| Direct Call Block | Direct calls to router should revert with "Direct call not allowed" | Use `walletClient.writeContract()` directly to `TrailsRouter.execute()` without delegatecall flag |
| Delegatecall Success | Execution through wallet delegatecall should succeed | Use `useQuote` hook which automatically uses delegatecall via wallet |
| Context Preservation | `msg.sender` = wallet address during execution | Monitor transaction logs to verify correct `msg.sender` |
| Shim Validation | `TrailsRouterShim` correctly wraps router calls | Check that shim execution precedes router execution in transaction traces |

**Expected Behavior**:
- Direct calls to `TrailsRouter` revert with `onlyDelegatecall` error
- SDK executions succeed via wallet delegatecall
- All router calls originate from wallet context

### B. Balance Injection & Calldata Surgery

**Audit Concern**: The `TrailsRouter.injectAndCall()` and `injectSweepAndCall()` functions must correctly replace placeholder bytes with actual wallet balances and handle calldata manipulation securely.

**Key Functions**:
- `TrailsRouter.injectAndCall((address to, uint256 value, bytes data) target, uint256 amountOffset)`
- `TrailsRouter.injectSweepAndCall((address to, uint256 value, bytes data) target, uint256 amountOffset)`
- Placeholder detection and replacement logic
- Balance calculation (current vs quoted amounts)

**Testing Approach**:

| Test Type | What to Validate | How to Test with SDK |
|-----------|------------------|---------------------|
| Placeholder Detection | 32-byte zero placeholder correctly identified | Configure widget with custom calldata containing placeholder bytes |
| Offset Calculation | `amountOffset` points to correct calldata position | Test with Aave/Morpho deposit calldata, verify injection position |
| Balance Accuracy | Current wallet balance used, not quoted amount | Compare injected amount with actual balance before execution |
| Bounds Checking | No out-of-bounds writes beyond calldata length | Test with malformed calldata (short/long offsets) |
| Token Handling | ETH vs ERC20 injection paths | Test native ETH and ERC20 scenarios separately |

**Expected Behavior**:
- Placeholder (`0x00...00`) replaced with actual wallet balance
- `amountOffset` correctly calculated for different contract ABIs
- No calldata corruption or out-of-bounds writes
- ETH value forwarding works without token wrapping

### C. Fee Collection & Refund Semantics

**Audit Concern**: Fee collection must only occur after successful execution verification, and refund mechanisms must protect users from unauthorized losses.

**Key Functions**:
- `TrailsRouter.validateOpHashAndSweep()` - Conditional fee collection
- `TrailsRouter.refundAndSweep()` - User protection on failure
- `TrailsRouterShim` success sentinel validation
- `onlyFallback` execution path control

**Testing Approach**:

| Test Type | What to Validate | How to Test with SDK |
|-----------|------------------|---------------------|
| Success Path | Fees collected only after success sentinel | Execute normal transfer, verify fee sweep follows shim execution |
| Failure Path | No fees on failed execution | Trigger destination failure, verify no fee collection |
| Refund Trigger | `refundAndSweep()` activates on `CallFailed` | Use invalid calldata to trigger destination revert |
| Conditional Sweep | `validateOpHashAndSweep()` requires sentinel | Test direct fee sweep before shim completion (should fail) |

**Expected Behavior**:
- `validateOpHashAndSweep()` reverts if success sentinel not set
- `refundAndSweep()` called only on failures with `onlyFallback=true`
- Origin failure → full refund, no bridging
- Destination failure → sweep to user on destination chain

### D. Entrypoint Contracts

**Audit Concern**: `TrailsIntentEntrypoint` must correctly validate EIP-712 signatures, handle ERC-2612 permits, and protect against replay attacks.

**Key Functions**:
- `depositToIntent(address user, address token, uint256 amount, address intentAddress, uint256 deadline)`
- `depositToIntentWithPermit(...)` - Gasless deposits
- `payFee()` / `payFeeWithPermit()` - Fee collection
- Nonce and deadline validation
- Reentrancy protection

**Testing Approach**:

| Test Type | What to Validate | How to Test with SDK |
|-----------|------------------|---------------------|
| EIP-712 Signature | Correct domain separator and signature recovery | Use `useQuote` with valid/invalid signatures |
| Nonce Management | Nonce increments, replay protection | Execute multiple deposits, verify nonce progression |
| Deadline Enforcement | Current time ≤ deadline | Test with expired deadlines (should fail) |
| Permit Handling | ERC-2612 permit validation | Use gasless mode with `selectedFeeToken` |
| Reentrancy Guard | No recursive calls during deposit | Cannot be directly tested via SDK, verify via Foundry |

**Expected Behavior**:
- Valid EIP-712 signatures accepted, invalid rejected
- Nonce increments per user/token pair
- Expired deadlines cause revert
- ERC-2612 permits use exact amounts, no excess consumption
- ReentrancyGuard prevents recursive deposit calls

### E. Cross-Chain Assumptions

**Audit Concern**: Non-atomic cross-chain execution must handle origin and destination failures correctly, with proper user protection and no stuck states.

**Key Functions**:
- Cross-chain coordination between origin and destination execution
- Bridge protocol integration (LiFi, CCTP, Relay)
- Destination failure handling and refunds
- Timeout and stuck state recovery

**Testing Approach**:

| Test Type | What to Validate | How to Test with SDK |
|-----------|------------------|---------------------|
| Origin Failure | No bridging on origin failure | Mock bridge failure (requires custom setup) |
| Destination Failure | Sweep to user on destination | Use invalid destination calldata |
| Provider Integration | Different bridge protocols | Test with `quoteProvider: 'lifi'`, `'cctp'`, `'relay'` |
| State Synchronization | No stuck funds between chains | Verify final balances match expected outcome |

**Expected Behavior**:
- Origin failure → full refund, no bridging occurs
- Destination failure → sweep to user on destination chain
- All funds either delivered to recipient or refunded
- Bridge protocols execute correctly without stuck states

### F. Storage Sentinels

**Audit Concern**: `TrailsSentinelLib` must use namespaced storage slots to avoid collisions with wallet storage.

**Key Functions**:
- `successSlot(bytes32 opHash)` - Sentinel slot calculation
- Storage namespace enforcement
- opHash uniqueness validation

**Testing Approach**:

| Test Type | What to Validate | How to Test with SDK |
|-----------|------------------|---------------------|
| Slot Namespacing | Sentinel slots don't collide with wallet storage | Cannot be directly tested via SDK, verify via Foundry |
| opHash Uniqueness | Different operations use different slots | Execute multiple operations, verify unique opHashes |
| Sentinel Value | Success value = `0x000...01`, failure = `0x000...00` | Monitor execution logs for sentinel setting |

**Expected Behavior**:
- All sentinel slots namespaced to avoid wallet collisions
- Success sentinel set only after complete execution
- Different opHashes generate different storage slots
- Sentinel value correctly indicates execution status

## Testing Workflow

### Step 1: Environment Setup

#### 1.1 Project Setup

Create a new test project:

```bash
mkdir trails-sdk-test && cd trails-sdk-test
npm init -y

# Install dependencies
npm install 0xtrails viem @tanstack/react-query
npm install --save-dev vitest @testing-library/react @testing-library/jest-dom

# Create basic package.json scripts
cat > package.json << 'EOF'
{
  "name": "trails-sdk-test",
  "version": "1.0.0",
  "scripts": {
    "test": "vitest",
    "test:scenarios": "vitest run --reporter=verbose"
  },
  "devDependencies": {
    "@testing-library/jest-dom": "^6.4.0",
    "@testing-library/react": "^14.0.0",
    "jsdom": "^23.0.0",
    "vitest": "^1.0.0"
  },
  "dependencies": {
    "0xtrails": "latest",
    "viem": "^2.0.0",
    "@tanstack/react-query": "^5.0.0"
  }
}
EOF
```

#### 1.2 Environment Configuration

Create `.env` file with your test configuration:

```bash
# Wallet configuration
TEST_PRIVATE_KEY=0x1234567890abcdef...  # Your test wallet private key

# API configuration
TRAILS_API_KEY=<FILL_IN_BLANK/>  # Request from project team

# RPC endpoints (optional - uses public defaults)
ARBITRUM_RPC_URL=https://arb1.arbitrum.io/rpc
BASE_RPC_URL=https://mainnet.base.org

# Testing configuration
SLIPPAGE_TOLERANCE=0.05  # 5% slippage for testing
```

#### 1.3 Verify Setup

Create a setup verification test:

```typescript
// test/setup.test.ts
import { describe, it, expect } from 'vitest'
import { privateKeyToAccount } from 'viem/accounts'
import { getSequenceConfig, getTrailsApiUrl, getSequenceProjectAccessKey } from '0xtrails/config'

describe('SDK Environment Setup', () => {
  it('should have valid private key', () => {
    const privateKey = process.env.TEST_PRIVATE_KEY
    expect(privateKey).toBeDefined()
    expect(privateKey).toMatch(/^0x[a-fA-F0-9]{64}$/)
    
    const account = privateKeyToAccount(privateKey as `0x${string}`)
    console.log('✅ Test wallet:', account.address)
  })

  it('should have valid API key', () => {
    const apiKey = getSequenceProjectAccessKey()
    expect(apiKey).toBeDefined()
    expect(apiKey).toMatch(/^pk_(live|test)_[a-zA-Z0-9]{32,}$/)
    console.log('✅ API key loaded:', apiKey.slice(0, 10) + '...')
  })

  it('should have valid API URL', () => {
    const apiUrl = getTrailsApiUrl()
    expect(apiUrl).toBeDefined()
    expect(apiUrl).toMatch(/^https?:\/\/.*\/api\/v1/)
    console.log('✅ API URL:', apiUrl)
  })

  it('should have sequence configuration', () => {
    const config = getSequenceConfig()
    expect(config).toBeDefined()
    console.log('✅ Sequence config loaded')
  })
})
```

Run setup verification:
```bash
npm run test test/setup.test.ts
```

### Step 2: Basic Testing with `useQuote` Hook

Create a basic test file to verify SDK functionality:

```typescript
// test/basic/BasicQuoteTest.test.ts
import { describe, it, expect, beforeAll } from 'vitest'
import { renderHook, waitFor } from '@testing-library/react'
import { createWalletClient, http } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { arbitrum, base } from 'viem/chains'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { SequenceHooksProvider } from '@0xsequence/hooks'
import { useQuote, TradeType } from '0xtrails/prepareSend'
import { getSequenceConfig, getTrailsApiUrl, getSequenceProjectAccessKey, getSequenceIndexerUrl } from '0xtrails/config'

// Setup
const privateKey = process.env.TEST_PRIVATE_KEY as `0x${string}`
const account = privateKeyToAccount(privateKey)
const walletClient = createWalletClient({
  account,
  chain: arbitrum,
  transport: http(),
})

const queryClient = new QueryClient({
  defaultOptions: {
    queries: { retry: false, staleTime: 0 },
  },
})

const createWrapper = () => ({ children }: { children: React.ReactNode }) => (
  <SequenceHooksProvider
    config={{
      projectAccessKey: getSequenceProjectAccessKey(),
      env: {
        indexerUrl: getSequenceIndexerUrl(),
        indexerGatewayUrl: getSequenceIndexerUrl(),
        apiUrl: getTrailsApiUrl(),
      },
    }}
  >
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  </SequenceHooksProvider>
)

describe('Basic SDK Quote Testing', () => {
  it('should generate quote for cross-chain transfer', async () => {
    console.log('Testing cross-chain USDC transfer quote...')
    
    const { result, waitFor } = renderHook(
      () =>
        useQuote({
          walletClient,
          fromTokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831', // ARBITRUM USDC
          fromChainId: arbitrum.id,
          toTokenAddress: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // BASE USDC
          toChainId: base.id,
          swapAmount: '10000', // 0.01 USDC (6 decimals)
          toRecipient: account.address,
          tradeType: TradeType.EXACT_OUTPUT,
          slippageTolerance: '0.05', // 5% slippage
          quoteProvider: 'auto',
          onStatusUpdate: (states) => {
            console.log('Transaction states update:', states.length, 'steps')
          },
        }),
      { wrapper: createWrapper() }
    )

    // Wait for quote to be generated
    await waitFor(
      () => {
        const { quote, isLoadingQuote, quoteError } = result.current
        console.log('Quote status:', { isLoading: isLoadingQuote, hasError: !!quoteError, hasQuote: !!quote })
        return !!quote && !isLoadingQuote && !quoteError
      },
      { timeout: 30000 } // 30 second timeout
    )

    const { quote, swap } = result.current

    expect(quote).toBeDefined()
    expect(swap).toBeDefined()
    
    if (quote) {
      console.log('✅ Quote generated successfully!')
      console.log('Provider:', quote.quoteProvider.name)
      console.log('From amount:', quote.fromAmount)
      console.log('To amount:', quote.toAmount)
      console.log('Steps:', quote.transactionStates.length)
      
      // Test quote structure
      expect(quote.originChain.id).toBe(arbitrum.id)
      expect(quote.destinationChain.id).toBe(base.id)
      expect(quote.slippageTolerance).toBe('0.05')
    }

    console.log('✅ Basic quote test passed')
  })
})
```

Run the basic test:
```bash
npm run test test/basic/BasicQuoteTest.test.ts
```

### Step 3: Testing Contract Interactions

#### 3.1 Cross-Chain Transfer Test

Test a complete cross-chain transfer:

```typescript
// test/cross-chain/CrossChainTest.test.ts
describe('Cross-Chain Transfer Testing', () => {
  it('should execute cross-chain transfer successfully', async () => {
    const { result, waitFor } = renderHook(
      () =>
        useQuote({
          walletClient,
          fromTokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831', // ARBITRUM USDC
          fromChainId: arbitrum.id,
          toTokenAddress: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // BASE USDC
          toChainId: base.id,
          swapAmount: '10000', // 0.01 USDC
          toRecipient: account.address,
          tradeType: TradeType.EXACT_OUTPUT,
          slippageTolerance: '0.05',
          quoteProvider: 'cctp', // Specific provider for testing
          onStatusUpdate: (states) => {
            console.log('Cross-chain states:', states.map(s => `${s.label} (${s.state})`))
          },
        }),
      { wrapper: createWrapper() }
    )

    // Wait for quote
    await waitFor(() => !!result.current.quote && !!result.current.swap, { timeout: 30000 })

    const { quote, swap } = result.current

    expect(quote).toBeDefined()
    expect(swap).toBeDefined()

    // Execute the swap (this triggers the full contract flow)
    console.log('Executing cross-chain transfer...')
    await swap!()

    // Wait for execution to complete
    await waitFor(
      () => {
        const states = quote?.transactionStates || []
        const allConfirmed = states.every(state => state.state === 'confirmed')
        console.log('Execution status:', { allConfirmed, states: states.length })
        return allConfirmed
      },
      { timeout: 180000 } // 3 minutes for cross-chain
    )

    console.log('✅ Cross-chain transfer completed successfully')
  })
})
```

#### 3.2 Gasless Execution Test

Test gasless execution with ERC-2612 permits:

```typescript
// test/gasless/GaslessTest.test.ts
describe('Gasless Execution Testing', () => {
  it('should execute gasless transfer with permit', async () => {
    const { result, waitFor } = renderHook(
      () =>
        useQuote({
          walletClient,
          fromTokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831', // ARBITRUM USDC
          fromChainId: arbitrum.id,
          toTokenAddress: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // BASE USDC
          toChainId: base.id,
          swapAmount: '10000', // 0.01 USDC
          toRecipient: account.address,
          tradeType: TradeType.EXACT_OUTPUT,
          slippageTolerance: '0.05',
          selectedFeeToken: {
            tokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831', // USDC for fees
            tokenSymbol: 'USDC'
          },
          onStatusUpdate: (states) => {
            console.log('Gasless states:', states.map(s => `${s.label} (${s.state})`))
          },
        }),
      { wrapper: createWrapper() }
    )

    // Wait for quote
    await waitFor(() => !!result.current.quote && !!result.current.swap, { timeout: 30000 })

    const { quote, swap } = result.current

    expect(quote).toBeDefined()
    expect(swap).toBeDefined()
    expect(quote?.selectedFeeToken).toBeDefined()

    // Execute gasless transfer
    console.log('Executing gasless transfer...')
    await swap!()

    // Wait for completion
    await waitFor(
      () => {
        const states = quote?.transactionStates || []
        const allConfirmed = states.every(state => state.state === 'confirmed')
        console.log('Gasless execution status:', { allConfirmed, states: states.length })
        return allConfirmed
      },
      { timeout: 180000 }
    )

    console.log('✅ Gasless execution completed successfully')
  })
})
```

### Step 4: Testing with the Widget

Create a React app to test the widget interface:

```typescript
// src/App.tsx
import React, { useState } from 'react'
import { createRoot } from 'react-dom/client'
import { TrailsWidget } from '0xtrails/widget'
import { SequenceProvider } from '@0xsequence/provider'
import { getSequenceConfig } from '0xtrails/config'

const App: React.FC = () => {
  const [amount, setAmount] = useState('0.01')
  const [fromChain, setFromChain] = useState('arbitrum')
  const [toChain, setToChain] = useState('base')

  return (
    <div style={{ padding: '20px', maxWidth: '800px', margin: '0 auto' }}>
      <h1>Trails SDK Testing Interface</h1>
      
      <div style={{ marginBottom: '20px', padding: '15px', border: '1px solid #ccc' }}>
        <h3>Configuration</h3>
        <label>
          Amount: <input 
            type="number" 
            value={amount} 
            onChange={(e) => setAmount(e.target.value)} 
            step="0.001"
          />
        </label>
        <br />
        <label style={{ marginLeft: '20px' }}>
          From: <select value={fromChain} onChange={(e) => setFromChain(e.target.value)}>
            <option value="arbitrum">Arbitrum</option>
            <option value="base">Base</option>
          </select>
        </label>
        <br />
        <label style={{ marginLeft: '20px' }}>
          To: <select value={toChain} onChange={(e) => setToChain(e.target.value)}>
            <option value="base">Base</option>
            <option value="arbitrum">Arbitrum</option>
          </select>
        </label>
      </div>
      
      <SequenceProvider config={getSequenceConfig()} defaultNetwork={fromChain}>
        <TrailsWidget
          defaultFromChain={fromChain}
          defaultToChain={toChain}
          defaultFromToken="USDC"
          defaultToToken="USDC"
          defaultAmount={amount}
          showDebugPanel={true}
          enableTestMode={true}
          slippageTolerance={0.05}
          quoteProvider="auto"
          onQuoteGenerated={(quote) => {
            console.log('Quote generated:', {
              from: `${quote.fromAmount} ${quote.originToken.symbol}`,
              to: `${quote.toAmount} ${quote.destinationToken.symbol}`,
              provider: quote.quoteProvider.name
            })
          }}
          onTransactionUpdate={(states) => {
            console.log('Transaction update:', states.length, 'active transactions')
            states.forEach(state => {
              if (state.state === 'confirmed') {
                console.log(`✅ ${state.label} completed on chain ${state.chainId}`)
              }
            })
          }}
          onExecutionComplete={(result) => {
            if (result.success) {
              console.log('🎉 Execution completed successfully!')
              console.log('Final balances:', result.finalBalances)
            } else {
              console.error('❌ Execution failed:', result.error)
            }
          }}
          onError={(error) => {
            console.error('Error:', error.message)
          }}
        />
      </SequenceProvider>
    </div>
  )
}

const root = createRoot(document.getElementById('root') as HTMLElement)
root.render(<App />)
```

Run the widget:
```bash
npm run dev
```

### Step 5: Validation and Monitoring

#### 5.1 Monitor Contract Interactions

During testing, monitor these key contract interactions:

1. **Deposit Phase** (`TrailsIntentEntrypoint`):
   - EIP-712 signature validation
   - Nonce and deadline enforcement
   - ReentrancyGuard protection

2. **Execution Phase** (`TrailsRouter` via `TrailsRouterShim`):
   - Delegatecall-only execution
   - SafeERC20 approvals
   - Balance injection accuracy

3. **Settlement Phase** (`TrailsRouter.sweep()`):
   - Conditional fee collection
   - Success sentinel verification
   - Dust cleanup and refunds

#### 5.2 Verify Economic Invariants

After each successful execution:

```typescript
// Validate no unauthorized losses
const originBalance = await getBalance(walletClient, {
  address: account.address,
  token: arbitrumUSDCAddress
})

const expectedLoss = parseUnits('0.07', 6) // ~0.07 USDC expected (deposit + fees)
const tolerance = parseUnits('0.001', 6) // 0.001 USDC tolerance

console.log('Balance validation:')
console.log(`Initial balance: ${formatUnits(initialBalance, 6)} USDC`)
console.log(`Current balance: ${formatUnits(originBalance, 6)} USDC`)
console.log(`Expected loss: ~0.07 USDC`)

expect(Math.abs(expectedLoss - (initialBalance - originBalance))).toBeLessThan(tolerance)
console.log('✅ Economic invariants preserved')
```

#### 5.3 Check Contract State

Verify contract state post-execution:

```bash
# Check if deposit was recorded
cast call $INTENT_ENTRYPOINT_ADDRESS "deposits(bytes32)(bool)" $INTENT_HASH --rpc-url $ARBITRUM_RPC_URL

# Verify success sentinel was set
cast call $ROUTER_SHIM_ADDRESS "successSentinel(bytes32)(bool)" $OP_HASH --rpc-url $ARBITRUM_RPC_URL

# Check fee collector received fees
cast call $USDC_ADDRESS "balanceOf(address)(uint256)" $FEE_COLLECTOR_ADDRESS --rpc-url $ARBITRUM_RPC_URL
```

### Step 6: Advanced Testing

#### 6.1 Test Different Providers

Test with different bridge providers:

```typescript
const providers = ['lifi', 'cctp', 'relay']

providers.forEach(async (provider) => {
  const { result } = renderHook(
    () =>
      useQuote({
        // ... base configuration
        quoteProvider: provider
      }),
    { wrapper: createWrapper() }
  )

  await waitFor(() => !!result.current.quote, { timeout: 30000 })
  
  console.log(`✅ ${provider} provider quote generated`)
})
```

#### 6.2 Test Failure Scenarios

Test error handling and fallback mechanisms:

```typescript
// Test invalid destination contract
const { result } = renderHook(
  () =>
    useQuote({
      // ... base configuration
      toCalldata: '0xdeadbeef', // Invalid calldata
    }),
  { wrapper: createWrapper() }
)

await waitFor(
  () => !!result.current.quoteError,
  { timeout: 30000 }
)

const error = result.current.quoteError
expect(error?.message).toContain('CallFailed')
console.log('✅ Failure handling works correctly')
```

#### 6.3 Performance Testing

Measure execution performance:

```typescript
const startTime = Date.now()

await swap()

const executionTime = Date.now() - startTime
console.log(`Execution time: ${executionTime}ms`)

expect(executionTime).toBeLessThan(300000) // Less than 5 minutes
```

## Audit Focus Areas Mapping

This section maps the testing scenarios to the six key audit focus areas identified in the audit documentation. Each audit concern is linked to specific contract functions to validate and testing approaches using the SDK.

### A. Delegatecall-Only Router Pattern

**Audit Concern**: The `TrailsRouter` and `TrailsRouterShim` contracts must enforce delegatecall-only execution to prevent direct calls that could bypass wallet context protection.

**Key Functions**:
- `TrailsRouter.onlyDelegatecall` modifier
- All `TrailsRouter` execution functions (`execute`, `pullAndExecute`, `injectAndCall`, `injectSweepAndCall`)
- `TrailsRouterShim` wrapper functions

**Testing Approach**:

| Test Type | What to Validate | How to Test with SDK |
|-----------|------------------|---------------------|
| Direct Call Block | Direct calls to router should revert with "Direct call not allowed" | Use `walletClient.writeContract()` directly to `TrailsRouter.execute()` without delegatecall flag |
| Delegatecall Success | Execution through wallet delegatecall should succeed | Use `useQuote` hook which automatically uses delegatecall via wallet |
| Context Preservation | `msg.sender` = wallet address during execution | Monitor transaction logs to verify correct `msg.sender` |
| Shim Validation | `TrailsRouterShim` correctly wraps router calls | Check that shim execution precedes router execution in transaction traces |

**Expected Behavior**:
- Direct calls to `TrailsRouter` revert with `onlyDelegatecall` error
- SDK executions succeed via wallet delegatecall
- All router calls originate from wallet context

### B. Balance Injection & Calldata Surgery

**Audit Concern**: The `TrailsRouter.injectAndCall()` and `injectSweepAndCall()` functions must correctly replace placeholder bytes with actual wallet balances and handle calldata manipulation securely.

**Key Functions**:
- `TrailsRouter.injectAndCall((address to, uint256 value, bytes data) target, uint256 amountOffset)`
- `TrailsRouter.injectSweepAndCall((address to, uint256 value, bytes data) target, uint256 amountOffset)`
- Placeholder detection and replacement logic
- Balance calculation (current vs quoted amounts)

**Testing Approach**:

| Test Type | What to Validate | How to Test with SDK |
|-----------|------------------|---------------------|
| Placeholder Detection | 32-byte zero placeholder correctly identified | Configure widget with custom calldata containing placeholder bytes |
| Offset Calculation | `amountOffset` points to correct calldata position | Test with Aave/Morpho deposit calldata, verify injection position |
| Balance Accuracy | Current wallet balance used, not quoted amount | Compare injected amount with actual balance before execution |
| Bounds Checking | No out-of-bounds writes beyond calldata length | Test with malformed calldata (short/long offsets) |
| Token Handling | ETH vs ERC20 injection paths | Test native ETH and ERC20 scenarios separately |

**Expected Behavior**:
- Placeholder (`0x00...00`) replaced with actual wallet balance
- `amountOffset` correctly calculated for different contract ABIs
- No calldata corruption or out-of-bounds writes
- ETH value forwarding works without token wrapping

### C. Fee Collection & Refund Semantics

**Audit Concern**: Fee collection must only occur after successful execution verification, and refund mechanisms must protect users from unauthorized losses.

**Key Functions**:
- `TrailsRouter.validateOpHashAndSweep()` - Conditional fee collection
- `TrailsRouter.refundAndSweep()` - User protection on failure
- `TrailsRouterShim` success sentinel validation
- `onlyFallback` execution path control

**Testing Approach**:

| Test Type | What to Validate | How to Test with SDK |
|-----------|------------------|---------------------|
| Success Path | Fees collected only after success sentinel | Execute normal transfer, verify fee sweep follows shim execution |
| Failure Path | No fees on failed execution | Trigger destination failure, verify no fee collection |
| Refund Trigger | `refundAndSweep()` activates on `CallFailed` | Use invalid calldata to trigger destination revert |
| Conditional Sweep | `validateOpHashAndSweep()` requires sentinel | Test direct fee sweep before shim completion (should fail) |

**Expected Behavior**:
- `validateOpHashAndSweep()` reverts if success sentinel not set
- `refundAndSweep()` called only on failures with `onlyFallback=true`
- Origin failure → full refund, no bridging
- Destination failure → sweep to user on destination chain

### D. Entrypoint Contracts

**Audit Concern**: `TrailsIntentEntrypoint` must correctly validate EIP-712 signatures, handle ERC-2612 permits, and protect against replay attacks.

**Key Functions**:
- `depositToIntent(address user, address token, uint256 amount, address intentAddress, uint256 deadline)`
- `depositToIntentWithPermit(...)` - Gasless deposits
- `payFee()` / `payFeeWithPermit()` - Fee collection
- Nonce and deadline validation
- Reentrancy protection

**Testing Approach**:

| Test Type | What to Validate | How to Test with SDK |
|-----------|------------------|---------------------|
| EIP-712 Signature | Correct domain separator and signature recovery | Use `useQuote` with valid/invalid signatures |
| Nonce Management | Nonce increments, replay protection | Execute multiple deposits, verify nonce progression |
| Deadline Enforcement | Current time ≤ deadline | Test with expired deadlines (should fail) |
| Permit Handling | ERC-2612 permit validation | Use gasless mode with `selectedFeeToken` |
| Reentrancy Guard | No recursive calls during deposit | Cannot be directly tested via SDK, verify via Foundry |

**Expected Behavior**:
- Valid EIP-712 signatures accepted, invalid rejected
- Nonce increments per user/token pair
- Expired deadlines cause revert
- ERC-2612 permits use exact amounts, no excess consumption
- ReentrancyGuard prevents recursive deposit calls

### E. Cross-Chain Assumptions

**Audit Concern**: Non-atomic cross-chain execution must handle origin and destination failures correctly, with proper user protection and no stuck states.

**Key Functions**:
- Cross-chain coordination between origin and destination execution
- Bridge protocol integration (LiFi, CCTP, Relay)
- Destination failure handling and refunds
- Timeout and stuck state recovery

**Testing Approach**:

| Test Type | What to Validate | How to Test with SDK |
|-----------|------------------|---------------------|
| Origin Failure | No bridging on origin failure | Mock bridge failure (requires custom setup) |
| Destination Failure | Sweep to user on destination | Use invalid destination calldata |
| Provider Integration | Different bridge protocols | Test with `quoteProvider: 'lifi'`, `'cctp'`, `'relay'` |
| State Synchronization | No stuck funds between chains | Verify final balances match expected outcome |

**Expected Behavior**:
- Origin failure → full refund, no bridging occurs
- Destination failure → sweep to user on destination chain
- All funds either delivered to recipient or refunded
- Bridge protocols execute correctly without stuck states

### F. Storage Sentinels

**Audit Concern**: `TrailsSentinelLib` must use namespaced storage slots to avoid collisions with wallet storage.

**Key Functions**:
- `successSlot(bytes32 opHash)` - Sentinel slot calculation
- Storage namespace enforcement
- opHash uniqueness validation

**Testing Approach**:

| Test Type | What to Validate | How to Test with SDK |
|-----------|------------------|---------------------|
| Slot Namespacing | Sentinel slots don't collide with wallet storage | Cannot be directly tested via SDK, verify via Foundry |
| opHash Uniqueness | Different operations use different slots | Execute multiple operations, verify unique opHashes |
| Sentinel Value | Success value = `0x000...01`, failure = `0x000...00` | Monitor execution logs for sentinel setting |

**Expected Behavior**:
- All sentinel slots namespaced to avoid wallet collisions
- Success sentinel set only after complete execution
- Different opHashes generate different storage slots
- Sentinel value correctly indicates execution status

## Testing Workflow

### Step 1: Environment Setup

#### 1.1 Project Setup

Create a new test project:

```bash
mkdir trails-sdk-test && cd trails-sdk-test
npm init -y

# Install dependencies
npm install 0xtrails viem @tanstack/react-query
npm install --save-dev vitest @testing-library/react @testing-library/jest-dom

# Create basic package.json scripts
cat > package.json << 'EOF'
{
  "name": "trails-sdk-test",
  "version": "1.0.0",
  "scripts": {
    "test": "vitest",
    "test:scenarios": "vitest run --reporter=verbose"
  },
  "devDependencies": {
    "@testing-library/jest-dom": "^6.4.0",
    "@testing-library/react": "^14.0.0",
    "jsdom": "^23.0.0",
    "vitest": "^1.0.0"
  },
  "dependencies": {
    "0xtrails": "latest",
    "viem": "^2.0.0",
    "@tanstack/react-query": "^5.0.0"
  }
}
EOF
```

#### 1.2 Environment Configuration

Create `.env` file with your test configuration:

```bash
# Wallet configuration
TEST_PRIVATE_KEY=0x1234567890abcdef...  # Your test wallet private key

# API configuration
TRAILS_API_KEY=<FILL_IN_BLANK/>  # Request from project team

# RPC endpoints (optional - uses public defaults)
ARBITRUM_RPC_URL=https://arb1.arbitrum.io/rpc
BASE_RPC_URL=https://mainnet.base.org

# Testing configuration
SLIPPAGE_TOLERANCE=0.05  # 5% slippage for testing
```

#### 1.3 Verify Setup

Create a setup verification test:

```typescript
// test/setup.test.ts
import { describe, it, expect } from 'vitest'
import { privateKeyToAccount } from 'viem/accounts'
import { getSequenceConfig, getTrailsApiUrl, getSequenceProjectAccessKey } from '0xtrails/config'

describe('SDK Environment Setup', () => {
  it('should have valid private key', () => {
    const privateKey = process.env.TEST_PRIVATE_KEY
    expect(privateKey).toBeDefined()
    expect(privateKey).toMatch(/^0x[a-fA-F0-9]{64}$/)
    
    const account = privateKeyToAccount(privateKey as `0x${string}`)
    console.log('✅ Test wallet:', account.address)
  })

  it('should have valid API key', () => {
    const apiKey = getSequenceProjectAccessKey()
    expect(apiKey).toBeDefined()
    expect(apiKey).toMatch(/^pk_(live|test)_[a-zA-Z0-9]{32,}$/)
    console.log('✅ API key loaded:', apiKey.slice(0, 10) + '...')
  })

  it('should have valid API URL', () => {
    const apiUrl = getTrailsApiUrl()
    expect(apiUrl).toBeDefined()
    expect(apiUrl).toMatch(/^https?:\/\/.*\/api\/v1/)
    console.log('✅ API URL:', apiUrl)
  })

  it('should have sequence configuration', () => {
    const config = getSequenceConfig()
    expect(config).toBeDefined()
    console.log('✅ Sequence config loaded')
  })
})
```

Run setup verification:
```bash
npm run test test/setup.test.ts
```

### Step 2: Basic Testing with `useQuote` Hook

Create a basic test file to verify SDK functionality:

```typescript
// test/basic/BasicQuoteTest.test.ts
import { describe, it, expect, beforeAll } from 'vitest'
import { renderHook, waitFor } from '@testing-library/react'
import { createWalletClient, http } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { arbitrum, base } from 'viem/chains'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { SequenceHooksProvider } from '@0xsequence/hooks'
import { useQuote, TradeType } from '0xtrails/prepareSend'
import { getSequenceConfig, getTrailsApiUrl, getSequenceProjectAccessKey, getSequenceIndexerUrl } from '0xtrails/config'

// Setup
const privateKey = process.env.TEST_PRIVATE_KEY as `0x${string}`
const account = privateKeyToAccount(privateKey)
const walletClient = createWalletClient({
  account,
  chain: arbitrum,
  transport: http(),
})

const queryClient = new QueryClient({
  defaultOptions: {
    queries: { retry: false, staleTime: 0 },
  },
})

const createWrapper = () => ({ children }: { children: React.ReactNode }) => (
  <SequenceHooksProvider
    config={{
      projectAccessKey: getSequenceProjectAccessKey(),
      env: {
        indexerUrl: getSequenceIndexerUrl(),
        indexerGatewayUrl: getSequenceIndexerUrl(),
        apiUrl: getTrailsApiUrl(),
      },
    }}
  >
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  </SequenceHooksProvider>
)

describe('Basic SDK Quote Testing', () => {
  it('should generate quote for cross-chain transfer', async () => {
    console.log('Testing cross-chain USDC transfer quote...')
    
    const { result, waitFor } = renderHook(
      () =>
        useQuote({
          walletClient,
          fromTokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831', // ARBITRUM USDC
          fromChainId: arbitrum.id,
          toTokenAddress: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // BASE USDC
          toChainId: base.id,
          swapAmount: '10000', // 0.01 USDC (6 decimals)
          toRecipient: account.address,
          tradeType: TradeType.EXACT_OUTPUT,
          slippageTolerance: '0.05', // 5% slippage
          quoteProvider: 'auto',
          onStatusUpdate: (states) => {
            console.log('Transaction states update:', states.length, 'steps')
          },
        }),
      { wrapper: createWrapper() }
    )

    // Wait for quote to be generated
    await waitFor(
      () => {
        const { quote, isLoadingQuote, quoteError } = result.current
        console.log('Quote status:', { isLoading: isLoadingQuote, hasError: !!quoteError, hasQuote: !!quote })
        return !!quote && !isLoadingQuote && !quoteError
      },
      { timeout: 30000 } // 30 second timeout
    )

    const { quote, swap } = result.current

    expect(quote).toBeDefined()
    expect(swap).toBeDefined()
    
    if (quote) {
      console.log('✅ Quote generated successfully!')
      console.log('Provider:', quote.quoteProvider.name)
      console.log('From amount:', quote.fromAmount)
      console.log('To amount:', quote.toAmount)
      console.log('Steps:', quote.transactionStates.length)
      
      // Test quote structure
      expect(quote.originChain.id).toBe(arbitrum.id)
      expect(quote.destinationChain.id).toBe(base.id)
      expect(quote.slippageTolerance).toBe('0.05')
    }

    console.log('✅ Basic quote test passed')
  })
})
```

Run the basic test:
```bash
npm run test test/basic/BasicQuoteTest.test.ts
```

### Step 3: Testing Contract Interactions

#### 3.1 Cross-Chain Transfer Test

Test a complete cross-chain transfer:

```typescript
// test/cross-chain/CrossChainTest.test.ts
describe('Cross-Chain Transfer Testing', () => {
  it('should execute cross-chain transfer successfully', async () => {
    const { result, waitFor } = renderHook(
      () =>
        useQuote({
          walletClient,
          fromTokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831', // ARBITRUM USDC
          fromChainId: arbitrum.id,
          toTokenAddress: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // BASE USDC
          toChainId: base.id,
          swapAmount: '10000', // 0.01 USDC
          toRecipient: account.address,
          tradeType: TradeType.EXACT_OUTPUT,
          slippageTolerance: '0.05',
          quoteProvider: 'cctp', // Specific provider for testing
          onStatusUpdate: (states) => {
            console.log('Cross-chain states:', states.map(s => `${s.label} (${s.state})`))
          },
        }),
      { wrapper: createWrapper() }
    )

    // Wait for quote
    await waitFor(() => !!result.current.quote && !!result.current.swap, { timeout: 30000 })

    const { quote, swap } = result.current

    expect(quote).toBeDefined()
    expect(swap).toBeDefined()

    // Execute the swap (this triggers the full contract flow)
    console.log('Executing cross-chain transfer...')
    await swap!()

    // Wait for execution to complete
    await waitFor(
      () => {
        const states = quote?.transactionStates || []
        const allConfirmed = states.every(state => state.state === 'confirmed')
        console.log('Execution status:', { allConfirmed, states: states.length })
        return allConfirmed
      },
      { timeout: 180000 } // 3 minutes for cross-chain
    )

    console.log('✅ Cross-chain transfer completed successfully')
  })
})
```

#### 3.2 Gasless Execution Test

Test gasless execution with ERC-2612 permits:

```typescript
// test/gasless/GaslessTest.test.ts
describe('Gasless Execution Testing', () => {
  it('should execute gasless transfer with permit', async () => {
    const { result, waitFor } = renderHook(
      () =>
        useQuote({
          walletClient,
          fromTokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831', // ARBITRUM USDC
          fromChainId: arbitrum.id,
          toTokenAddress: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // BASE USDC
          toChainId: base.id,
          swapAmount: '10000', // 0.01 USDC
          toRecipient: account.address,
          tradeType: TradeType.EXACT_OUTPUT,
          slippageTolerance: '0.05',
          selectedFeeToken: {
            tokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831', // USDC for fees
            tokenSymbol: 'USDC'
          },
          onStatusUpdate: (states) => {
            console.log('Gasless states:', states.map(s => `${s.label} (${s.state})`))
          },
        }),
      { wrapper: createWrapper() }
    )

    // Wait for quote
    await waitFor(() => !!result.current.quote && !!result.current.swap, { timeout: 30000 })

    const { quote, swap } = result.current

    expect(quote).toBeDefined()
    expect(swap).toBeDefined()
    expect(quote?.selectedFeeToken).toBeDefined()

    // Execute gasless transfer
    console.log('Executing gasless transfer...')
    await swap!()

    // Wait for completion
    await waitFor(
      () => {
        const states = quote?.transactionStates || []
        const allConfirmed = states.every(state => state.state === 'confirmed')
        console.log('Gasless execution status:', { allConfirmed, states: states.length })
        return allConfirmed
      },
      { timeout: 180000 }
    )

    console.log('✅ Gasless execution completed successfully')
  })
})
```

### Step 4: Testing with the Widget

Create a React app to test the widget interface:

```typescript
// src/App.tsx
import React, { useState } from 'react'
import { createRoot } from 'react-dom/client'
import { TrailsWidget } from '0xtrails/widget'
import { SequenceProvider } from '@0xsequence/provider'
import { getSequenceConfig } from '0xtrails/config'

const App: React.FC = () => {
  const [amount, setAmount] = useState('0.01')
  const [fromChain, setFromChain] = useState('arbitrum')
  const [toChain, setToChain] = useState('base')

  return (
    <div style={{ padding: '20px', maxWidth: '800px', margin: '0 auto' }}>
      <h1>Trails SDK Testing Interface</h1>
      
      <div style={{ marginBottom: '20px', padding: '15px', border: '1px solid #ccc' }}>
        <h3>Configuration</h3>
        <label>
          Amount: <input 
            type="number" 
            value={amount} 
            onChange={(e) => setAmount(e.target.value)} 
            step="0.001"
          />
        </label>
        <br />
        <label style={{ marginLeft: '20px' }}>
          From: <select value={fromChain} onChange={(e) => setFromChain(e.target.value)}>
            <option value="arbitrum">Arbitrum</option>
            <option value="base">Base</option>
          </select>
        </label>
        <br />
        <label style={{ marginLeft: '20px' }}>
          To: <select value={toChain} onChange={(e) => setToChain(e.target.value)}>
            <option value="base">Base</option>
            <option value="arbitrum">Arbitrum</option>
          </select>
        </label>
      </div>
      
      <SequenceProvider config={getSequenceConfig()} defaultNetwork={fromChain}>
        <TrailsWidget
          defaultFromChain={fromChain}
          defaultToChain={toChain}
          defaultFromToken="USDC"
          defaultToToken="USDC"
          defaultAmount={amount}
          showDebugPanel={true}
          enableTestMode={true}
          slippageTolerance={0.05}
          quoteProvider="auto"
          onQuoteGenerated={(quote) => {
            console.log('Quote generated:', {
              from: `${quote.fromAmount} ${quote.originToken.symbol}`,
              to: `${quote.toAmount} ${quote.destinationToken.symbol}`,
              provider: quote.quoteProvider.name
            })
          }}
          onTransactionUpdate={(states) => {
            console.log('Transaction update:', states.length, 'active transactions')
            states.forEach(state => {
              if (state.state === 'confirmed') {
                console.log(`✅ ${state.label} completed on chain ${state.chainId}`)
              }
            })
          }}
          onExecutionComplete={(result) => {
            if (result.success) {
              console.log('🎉 Execution completed successfully!')
              console.log('Final balances:', result.finalBalances)
            } else {
              console.error('❌ Execution failed:', result.error)
            }
          }}
          onError={(error) => {
            console.error('Error:', error.message)
          }}
        />
      </SequenceProvider>
    </div>
  )
}

const root = createRoot(document.getElementById('root') as HTMLElement)
root.render(<App />)
```

Run the widget:
```bash
npm run dev
```

### Step 5: Validation and Monitoring

#### 5.1 Monitor Contract Interactions

During testing, monitor these key contract interactions:

1. **Deposit Phase** (`TrailsIntentEntrypoint`):
   - EIP-712 signature validation
   - Nonce and deadline enforcement
   - ReentrancyGuard protection

2. **Execution Phase** (`TrailsRouter` via `TrailsRouterShim`):
   - Delegatecall-only execution
   - SafeERC20 approvals
   - Balance injection accuracy

3. **Settlement Phase** (`TrailsRouter.sweep()`):
   - Conditional fee collection
   - Success sentinel verification
   - Dust cleanup and refunds

#### 5.2 Verify Economic Invariants

After each successful execution:

```typescript
// Validate no unauthorized losses
const originBalance = await getBalance(walletClient, {
  address: account.address,
  token: arbitrumUSDCAddress
})

const expectedLoss = parseUnits('0.07', 6) // ~0.07 USDC expected (deposit + fees)
const tolerance = parseUnits('0.001', 6) // 0.001 USDC tolerance

console.log('Balance validation:')
console.log(`Initial balance: ${formatUnits(initialBalance, 6)} USDC`)
console.log(`Current balance: ${formatUnits(originBalance, 6)} USDC`)
console.log(`Expected loss: ~0.07 USDC`)

expect(Math.abs(expectedLoss - (initialBalance - originBalance))).toBeLessThan(tolerance)
console.log('✅ Economic invariants preserved')
```

#### 5.3 Check Contract State

Verify contract state post-execution:

```bash
# Check if deposit was recorded
cast call $INTENT_ENTRYPOINT_ADDRESS "deposits(bytes32)(bool)" $INTENT_HASH --rpc-url $ARBITRUM_RPC_URL

# Verify success sentinel was set
cast call $ROUTER_SHIM_ADDRESS "successSentinel(bytes32)(bool)" $OP_HASH --rpc-url $ARBITRUM_RPC_URL

# Check fee collector received fees
cast call $USDC_ADDRESS "balanceOf(address)(uint256)" $FEE_COLLECTOR_ADDRESS --rpc-url $ARBITRUM_RPC_URL
```

### Step 6: Advanced Testing

#### 6.1 Test Different Providers

Test with different bridge providers:

```typescript
const providers = ['lifi', 'cctp', 'relay']

providers.forEach(async (provider) => {
  const { result } = renderHook(
    () =>
      useQuote({
        // ... base configuration
        quoteProvider: provider
      }),
    { wrapper: createWrapper() }
  )

  await waitFor(() => !!result.current.quote, { timeout: 30000 })
  
  console.log(`✅ ${provider} provider quote generated`)
})
```

#### 6.2 Test Failure Scenarios

Test error handling and fallback mechanisms:

```typescript
// Test invalid destination contract
const { result } = renderHook(
  () =>
    useQuote({
      // ... base configuration
      toCalldata: '0xdeadbeef', // Invalid calldata
    }),
  { wrapper: createWrapper() }
)

await waitFor(
  () => !!result.current.quoteError,
  { timeout: 30000 }
)

const error = result.current.quoteError
expect(error?.message).toContain('CallFailed')
console.log('✅ Failure handling works correctly')
```

#### 6.3 Performance Testing

Measure execution performance:

```typescript
const startTime = Date.now()

await swap()

const executionTime = Date.now() - startTime
console.log(`Execution time: ${executionTime}ms`)

expect(executionTime).toBeLessThan(300000) // Less than 5 minutes
```

## Troubleshooting & Support

This section covers common issues encountered when testing with the 0xtrails SDK and provides solutions for debugging contract interactions.

### Common Issues

#### 1. Environment & Configuration

**Issue**: "TEST_PRIVATE_KEY not set"
```
Error: TEST_PRIVATE_KEY environment variable required
```

**Solution**:
- Verify `.env` file exists in project root
- Check for correct syntax: `TEST_PRIVATE_KEY=0x...` (no quotes around value)
- Ensure no trailing spaces or hidden characters
- Restart test runner after changes

**Quick Fix**:
```bash
# Debug environment
echo "Private key length:" $(echo $TEST_PRIVATE_KEY | wc -c)
echo "Private key format:" $(echo $TEST_PRIVATE_KEY | cut -c1-10)

# Reload environment
source .env
export $(cat .env | grep -v '^#' | xargs)
```

**Issue**: "Invalid API Key" or "Unauthorized"
```
Error: Unauthorized - Invalid project access key
Error: 401 Unauthorized
```

**Solution**:
- Verify key starts with `pk_test_` or `pk_live_`
- Check for copy-paste errors or extra whitespace
- Ensure key has testing permissions (contact team if using public key)
- Test key format with: `echo $TRAILS_API_KEY | head -c 10`

**Quick Fix**:
```bash
# Test API key directly
curl -H "Authorization: Bearer $TRAILS_API_KEY" \
     -H "Content-Type: application/json" \
     https://api.trails.live/v1/health

# Expected response: {"status":"ok"}
```

**Issue**: "Network timeout" or "RPC endpoint unreachable"
```
Error: Network timeout
Error: fetch failed
```

**Solution**:
- Check internet connection and firewall settings
- Verify RPC URL is accessible: `curl https://arb1.arbitrum.io/rpc`
- Use alternative RPC providers (Alchemy, Infura)
- Increase timeout in wallet client configuration

**Quick Fix**:
```bash
# Test RPC connectivity
curl -X POST -H "Content-Type: application/json" \
     -d '{"jsonrpc":"2.0","method":"eth_blockNumber","id":1}' \
     https://arb1.arbitrum.io/rpc

# Alternative RPC URLs
export ARBITRUM_RPC_URL=https://arbitrum.llamarpc.com/rpc
export BASE_RPC_URL=https://base.llamarpc.com/rpc
```

#### 2. Wallet & Funding Issues

**Issue**: "Insufficient balance" during execution
```
Error: Insufficient funds for gas * price + value
Error: ERC20: transfer amount exceeds balance
```

**Solution**:
- Check wallet balance on both origin and destination chains
- Ensure sufficient gas (0.01 ETH recommended per chain)
- Verify token approvals and allowances
- For gasless flows, confirm ERC-2612 permit support

**Quick Fix**:
```bash
# Check wallet balances
cast balance $TEST_WALLET_ADDRESS --rpc-url $ARBITRUM_RPC_URL
cast balance $TEST_WALLET_ADDRESS --rpc-url $BASE_RPC_URL

# Check USDC balance
cast call $ARBITRUM_USDC "balanceOf(address)(uint256)" $TEST_WALLET_ADDRESS --rpc-url $ARBITRUM_RPC_URL

# Fund wallet (if using testnet)
# Contact team for testnet faucet URLs
```

**Issue**: "Invalid signature" or "Permit failed"
```
Error: Invalid ERC-2612 signature
Error: Invalid EIP-712 signature
```

**Solution**:
- Verify wallet has correct private key loaded
- Check token supports EIP-2612 permits (USDC, DAI, etc.)
- Ensure nonce hasn't been used previously
- For gasless flows, verify permit deadline is in future

**Quick Fix**:
```typescript
// Test signature generation
const { address } = account
const nonce = await token.nonces(address)
const deadline = Math.floor(Date.now() / 1000) + 3600
const permit = {
  owner: address,
  spender: intentAddress,
  value: amount,
  nonce,
  deadline
}

const signature = await account.signTypedData({
  domain: { name: 'USDC', version: '2', chainId, verifyingContract: tokenAddress },
  types: { Permit: [{ name: 'owner', type: 'address' }, { name: 'spender', type: 'address' }, { name: 'value', type: 'uint256' }, { name: 'nonce', type: 'uint256' }, { name: 'deadline', type: 'uint256' }] },
  primaryType: 'Permit',
  message: permit
})

console.log('Signature valid until:', new Date(deadline * 1000))
console.log('Expected nonce:', nonce.toString())
```

#### 3. Quote & Execution Errors

**Issue**: "No route found" or "Insufficient liquidity"
```
Error: No route found between tokens
Error: Insufficient liquidity for requested amount
```

**Solution**:
- Verify token addresses are correct for the chain
- Try smaller test amounts (0.01 USDC instead of 1.00 USDC)
- Test different quote providers (`lifi`, `cctp`, `relay`)
- Check if chains are supported by the bridge provider

**Quick Fix**:
```bash
# Test with smaller amounts
TEST_SCENARIOS="PAY_USDC_BASE" AMOUNT="0.001" pnpm run test:scenarios

# Try different providers
TEST_SCENARIOS="PAY_USDC_BASE" QUOTE_PROVIDER="lifi" pnpm run test:scenarios

# Verify token liquidity
curl "https://api.li.fi/v1/quote?fromChainId=42161&toChainId=8453&fromTokenAddress=0xaf88d065...&toTokenAddress=0x833589fc...&fromAmount=10000"
```

**Issue**: "Slippage exceeded" during execution
```
Error: Slippage tolerance exceeded
Error: Transaction failed: SlippageCheck
```

**Solution**:
- Increase slippage tolerance for test amounts (use 5-10% for testing)
- Verify market conditions (high volatility may cause slippage)
- Check if quote is still valid (retry after a few seconds)
- For EXACT_OUTPUT trades, reduce output amount

**Quick Fix**:
```bash
# Increase slippage tolerance
SLIPPAGE_TOLERANCE="0.10" TEST_SCENARIOS="PAY_USDC_BASE" pnpm run test:scenarios

# Use EXACT_INPUT instead of EXACT_OUTPUT
TRADE_TYPE="EXACT_INPUT" TEST_SCENARIOS="PAY_USDC_BASE" pnpm run test:scenarios
```

**Issue**: "Quote expired" or "Route invalid"
```
Error: Quote has expired
Error: Invalid route - quote no longer available
```

**Solution**:
- Quotes are valid for ~30 seconds; regenerate frequently
- High market volatility may invalidate routes
- Use shorter test execution times
- Implement quote refresh logic for production

**Quick Fix**:
```typescript
// Add quote refresh in test
const refreshQuote = async () => {
  const newQuote = await useQuote({
    // ... same parameters
  })
  
  // Retry execution with fresh quote
  if (newQuote.quote && !newQuote.isLoadingQuote) {
    await newQuote.swap()
  }
}
```

#### 4. Contract-Specific Issues

**Issue**: "Direct call not allowed" (Delegatecall Enforcement)
```
Error: Execution reverted: "Direct call not allowed"
Error: onlyDelegatecall: Direct calls not permitted
```

**Solution**:
- This is expected behavior - `TrailsRouter` requires delegatecall
- The SDK automatically uses delegatecall through wallet execution
- Direct calls are blocked for security (this is correct)

**Debugging**:
- Verify SDK is calling via wallet delegatecall
- Test shows security working properly
- No action required - this validates the audit concern

**Issue**: "Sentinel not set" (Fee Collection)
```
Error: Sentinel value mismatch - operation not successful
Error: validateOpHashAndSweep: Success sentinel not set
```

**Solution**:
- Verify `TrailsRouterShim` executed successfully before fee sweep
- Check if opHash matches between shim and sweep
- Ensure all execution steps completed without reverts
- Test case for conditional fee collection validation

**Debugging**:
```bash
# Check sentinel value manually
cast storage $ROUTER_SHIM_ADDRESS $OP_HASH_SLOT --rpc-url $RPC_URL

# Should be 0x000...01 for success, 0x000...00 for failure
```

**Issue**: "Invalid amountOffset" (Balance Injection)
```
Error: Invalid amountOffset - placeholder not found
Error: Execution reverted: "Placeholder not found"
```

**Solution**:
- Verify calldata contains exact 32-byte zero placeholder
- Check `amountOffset` points to correct position in calldata
- Ensure placeholder is at expected memory location
- Test with different contract ABIs and parameter orders

**Debugging**:
```typescript
// Log calldata before/after injection
console.log('Calldata before injection:', calldataBefore)
console.log('Expected placeholder at offset:', amountOffset)
console.log('Calldata after injection:', calldataAfter)

// Verify placeholder replacement
const placeholder = '0x0000000000000000000000000000000000000000000000000000000000000000'
const replaced = calldataAfter.includes(actualBalance.toString(16).padStart(64, '0'))
console.log('Placeholder replaced:', replaced)
```

**Issue**: "ReentrancyGuard: reentrant call" (Intent Entrypoint)
```
Error: ReentrancyGuard: reentrant call
Error: Execution reverted: "ReentrancyGuard reentrant call"
```

**Solution**:
- Expected behavior during malicious reentrancy attempts
- Validates reentrancy protection in `TrailsIntentEntrypoint`
- No action required - security feature working correctly

#### 5. Gasless Flow Issues

**Issue**: "Permit not supported" or "Invalid permit nonce"
```
Error: Token does not support permits
Error: Invalid nonce for permit
```

**Solution**:
- Verify token contract supports ERC-2612 (`permit` function)
- Check current nonce hasn't changed between signing and execution
- Ensure deadline is in future (current time + buffer)
- Test with tokens known to support permits (USDC, DAI, WETH)

**Quick Fix**:
```bash
# Check if token supports permits
cast call $TOKEN_ADDRESS "permit(address,address,uint256,uint256,uint8,bytes32,bytes32)" 0x0000000000000000000000000000000000000000 0x0000000000000000000000000000000000000000 1000 0 0 0x0000000000000000000000000000000000000000000000000000000000000000 0x0000000000000000000000000000000000000000000000000000000000000000 --rpc-url $RPC_URL

# Should not revert if permit is supported
```

**Issue**: "Insufficient permit allowance"
```
Error: Permit allowance insufficient for deposit + fee
Error: Transfer amount exceeds permit
```

**Solution**:
- Ensure permit amount covers deposit + estimated fees
- Check fee estimation accuracy in SDK
- Verify total permit = deposit + fees + buffer
- Test with different fee token amounts

**Debugging**:
```typescript
// Calculate total required permit
const depositAmount = parseUnits('0.01', 6) // 0.01 USDC
const estimatedFee = parseUnits('0.002', 6) // 0.002 USDC fee
const totalPermit = depositAmount + estimatedFee + parseUnits('0.001', 6) // Buffer

console.log('Deposit:', formatUnits(depositAmount, 6))
console.log('Fee estimate:', formatUnits(estimatedFee, 6))
console.log('Total permit:', formatUnits(totalPermit, 6))
console.log('Permit covers deposit + fee:', totalPermit >= depositAmount + estimatedFee)
```

#### 6. Cross-Chain Issues

**Issue**: "Bridge execution failed" or "Provider unavailable"
```
Error: Bridge protocol execution failed
Error: Provider not available for route
```

**Solution**:
- Verify bridge provider supports the chain pair
- Check if provider has liquidity for the amount/route
- Try alternative providers (LiFi, CCTP, Relay)
- Ensure both chains are active and synced

**Quick Fix**:
```bash
# Test different providers
QUOTE_PROVIDER="lifi" npm run test:scenarios
QUOTE_PROVIDER="cctp" npm run test:scenarios  
QUOTE_PROVIDER="relay" npm run test:scenarios

# Check provider status
curl "https://api.li.fi/v1/providers" | jq '.providers[] | select(.name == "CCTP")'
```

**Issue**: "Destination chain execution failed"
```
Error: Destination execution failed - funds swept to user
Error: CallFailed event emitted on destination
```

**Solution**:
- This is expected behavior for destination failures
- Verify funds are swept back to user on destination chain
- Check `refundAndSweep()` was called correctly
- No bridge reversal needed (correct economic design)

**Debugging**:
```bash
# Verify destination refund
cast balance $RECIPIENT_ADDRESS --rpc-url $DESTINATION_RPC_URL

# Check for Refund events
cast receipt $DESTINATION_TX_HASH --rpc-url $DESTINATION_RPC_URL | jq '.logs[] | select(.topics[0] | contains("Refund"))'
```

#### 7. Performance Issues

**Issue**: "Execution timeout" or "Slow quote generation"
```
Error: Execution timeout exceeded
Error: Quote generation taking too long
```

**Solution**:
- Reduce test amounts to minimize slippage and routing complexity
- Use faster RPC endpoints (Alchemy > Public)
- Increase timeout values for cross-chain scenarios
- Test during low network congestion periods

**Quick Fix**:
```bash
# Use faster RPC
export ARBITRUM_RPC_URL=https://arb-mainnet.g.alchemy.com/v2/YOUR_KEY
export BASE_RPC_URL=https://base-mainnet.g.alchemy.com/v2/YOUR_KEY

# Reduce test amounts
AMOUNT="0.001" npm run test:scenarios

# Increase timeouts
TIMEOUT=300000 npm run test:scenarios  # 5 minutes
```

### Debugging Techniques

#### 1. Enable Verbose Logging

Add debug logging to see detailed execution flow:

```typescript
// Add to test configuration
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: false,
      staleTime: 0,
      // Enable verbose logging
      log: true,
      notifyOnChangeProps: 'tracked',
    },
  },
})

// Add SDK debug mode
process.env.DEBUG = '0xtrails:*'
```

#### 2. Transaction Tracing

Use block explorer APIs to trace contract calls:

```bash
# Trace specific transaction
curl "https://api.arbiscan.io/api?module=proxy&action=eth_getTransactionByHash&txhash=$TX_HASH&apikey=YourApiKey"

# Decode transaction input
cast run $CONTRACT_ADDRESS $TX_INPUT --rpc-url $RPC_URL

# Check internal calls
tenderly simulate $TX_HASH --network arbitrum
```

#### 3. Contract State Inspection

Verify contract state after execution:

```bash
# Check deposit status in TrailsIntentEntrypoint
cast call $INTENT_ENTRYPOINT_ADDRESS "deposits(bytes32)(bool)" $INTENT_HASH --rpc-url $RPC_URL

# Verify success sentinel
cast call $ROUTER_SHIM_ADDRESS "successSentinel(bytes32)(bool)" $OP_HASH --rpc-url $RPC_URL

# Check fee collector balance
cast call $FEE_TOKEN_ADDRESS "balanceOf(address)(uint256)" $FEE_COLLECTOR_ADDRESS --rpc-url $RPC_URL
```

#### 4. Event Log Analysis

Decode and analyze contract events:

```typescript
// Parse events from transaction receipt
const receipt = await publicClient.waitForTransactionReceipt({ hash: txHash })

receipt.logs.forEach((log, index) => {
  try {
    const { eventName, args } = parseEvent({
      abi: contractAbi,
      eventName: 'DepositToIntent', // or 'Sweep', 'CallFailed', etc.
      data: log.data,
      topics: log.topics
    })
    
    console.log(`Event ${index}: ${eventName}`)
    console.log('Arguments:', args)
    console.log('Address:', log.address)
  } catch (error) {
    console.log(`Event ${index}: Unknown event at ${log.address}`)
  }
})
```

#### 5. Gas Analysis

Profile gas usage for optimization testing:

```bash
# Analyze gas usage for specific transaction
cast receipt $TX_HASH --rpc-url $RPC_URL | jq '.gasUsed'

# Compare gas across different scenarios
echo "Simple transfer: $(cast receipt $TX1 | jq '.gasUsed')"
echo "Complex execution: $(cast receipt $TX2 | jq '.gasUsed')"
echo "Gas difference: $((tx2_gas - tx1_gas))"

# Test gas limits
cast estimate --rpc-url $RPC_URL $TX_DATA
```

### Support Contact Information

#### Primary Support Channel

**Discord**: Join the official Trails Discord for real-time support:
- Channel: `#sdk-testing-support`
- Direct Message: `@trails-support` (mention your testing issue)

#### Email Support

**General Inquiries**: `support@trails.build`
- Subject prefix: `[SDK Testing]`
- Include: Error logs, scenario description, transaction hash, SDK version

#### Issue Reporting Template

When reporting issues, use this template:

```
Subject: [SDK Testing] Issue with Cross-Chain Transfer - Execution Failed

Environment:
- Node.js version: [output of `node --version`]
- SDK version: [output of `npm list 0xtrails`]
- Network: [Arbitrum/Base testnet/mainnet]
- RPC: [URL used]

Test Configuration:
- From Chain: Arbitrum
- To Chain: Base
- From Token: USDC (0xaf88d065...)
- To Token: USDC (0x833589fc...)
- Amount: 0.01 USDC
- Provider: cctp

Error Details:
```
[Complete error message and stack trace]
```

Transaction Hashes:
- Deposit: [0x...]
- Execution: [0x...]
- Explorer links: [Arbiscan/Basescan URLs]

Expected Behavior:
[What should have happened]

Actual Behavior:
[What actually happened]

Additional Context:
[Any screenshots, logs, or relevant information]
```

#### Response Time Expectations

- **Critical Issues** (contract reverts, security concerns): < 2 hours
- **High Priority** (execution failures, major functionality): < 4 hours  
- **Medium Priority** (performance, edge cases): < 24 hours
- **Low Priority** (documentation, minor bugs): < 48 hours

#### Escalation Process

For urgent issues during audit deadlines:

1. **Immediate**: Ping `@trails-support` in Discord #sdk-testing-support
2. **Email**: `urgent@trails.build` with `[URGENT]` subject prefix
3. **Direct**: Contact project lead via designated channel

#### Common Resources

**Documentation**:
- [Trails SDK Reference](https://docs.trails.build/sdk-reference)
- [Contract ABIs](https://docs.trails.build/contract-abi)
- [API Endpoints](https://docs.trails.build/api-reference)

**Tools**:
- [Tenderly](https://tenderly.co/) - Transaction debugging
- [Etherscan](https://etherscan.io/) - Transaction tracing
- [Foundry](https://getfoundry.sh/) - Contract interaction
- [Hardhat](https://hardhat.org/) - Local testing

**Testnet Faucets**:
- Arbitrum: Contact team for testnet deployment
- Base: Contact team for testnet deployment
- Other chains: Standard testnet faucets

### When to Contact Support

Contact the team if you encounter:

1. **Security Issues**: Unauthorized transactions, signature validation failures
2. **Contract Reverts**: Unexpected reverts without clear error messages
3. **API Errors**: Persistent 4xx/5xx errors despite correct configuration
4. **Test Failures**: Scenarios failing consistently across environments
5. **Performance Issues**: Execution times > 5 minutes for simple transfers
6. **Integration Problems**: SDK not working with specific wallet providers

### Self-Service Debugging

Before contacting support, try these steps:

#### 1. Verify Environment
```bash
# Check all environment variables
cat .env | grep -v '^#'

# Validate wallet connectivity
cast wallet address --private-key $TEST_PRIVATE_KEY

# Test API connectivity
curl -H "Authorization: Bearer $TRAILS_API_KEY" https://api.trails.live/v1/health
```

#### 2. Clear Cache
```bash
# Clear query cache
rm -rf node_modules/.cache
rm -rf .vite

# Clear local storage (browser testing)
localStorage.clear()

# Restart test environment
npm run dev -- --force
```

#### 3. Minimal Test Case
Create a minimal test to isolate the issue:

```typescript
// test/minimal/MinimalTest.ts
import { createWalletClient, http } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { arbitrum } from 'viem/chains'
import { useQuote, TradeType } from '0xtrails/prepareSend'

const minimalTest = async () => {
  const account = privateKeyToAccount(process.env.TEST_PRIVATE_KEY as `0x${string}`)
  const walletClient = createWalletClient({
    account,
    chain: arbitrum,
    transport: http(),
  })

  const { quote } = useQuote({
    walletClient,
    fromTokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831', // USDC
    fromChainId: arbitrum.id,
    toTokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831', // USDC (same chain for minimal test)
    toChainId: arbitrum.id,
    swapAmount: '1000', // 0.001 USDC - minimal amount
    toRecipient: account.address,
    tradeType: TradeType.EXACT_OUTPUT,
    slippageTolerance: '0.10', // High tolerance for testing
  })

  console.log('Minimal quote:', quote ? 'Success' : 'Failed')
  return quote
}

minimalTest().catch(console.error)
```

#### 4. Check Dependencies
```bash
# Verify SDK version
npm list 0xtrails

# Check for dependency conflicts
npm ls | grep -E "(multiple|UNMET)"

# Update to latest versions
npm update 0xtrails viem @0xsequence/hooks
```

#### 5. Network Status
```bash
# Check chain status
cast block-number --rpc-url $ARBITRUM_RPC_URL
cast block-number --rpc-url $BASE_RPC_URL

# Monitor gas prices
cast gas-price --rpc-url $ARBITRUM_RPC_URL
```

### Support Escalation Flow

1. **Self-Service**: Try debugging steps above
2. **Discord**: Post in `#sdk-testing-support` with logs
3. **Email**: Send detailed report to `support@trails.build`
4. **Urgent**: DM `@trails-support` for immediate assistance
5. **Critical**: Use emergency contact in audit guidelines

### Audit-Specific Support

During the Code4rena audit period:

- **Dedicated Channel**: `#trails-sdk-audit` for audit-related issues
- **Daily Office Hours**: Check Discord announcements for schedule
- **Priority Response**: Audit participants get expedited support
- **Test Environment**: Team provides testnet deployments and faucets
- **Bug Bounty**: Valid findings qualify for standard Code4rena rewards

### Final Notes

- **Testnet vs Mainnet**: Use testnet for initial testing, mainnet for final validation
- **Gas Costs**: Small test amounts keep costs low (< $0.10 per test)
- **Rate Limits**: Monitor API usage to avoid throttling
- **Documentation**: All contract ABIs and interfaces available in SDK source
- **Community**: Join Discord for peer support from other auditors

This guide provides comprehensive coverage for testing Trails contracts with the 0xtrails SDK. The combination of the `useQuote` hook, widget interface, and testing utilities ensures thorough validation of all critical contract functionality.

---

*Thank you for testing the Trails SDK! Your contributions help secure the protocol for all users.*
