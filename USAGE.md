
# Using the Retirement Aggregator

This guide covers everything needed to interact with an already-deployed Retirement Aggregator instance. No source code or development setup required.

## Contract Addresses

### Base Mainnet (Chain ID: 8453)

| Contract              | Address                                      |
|-----------------------|----------------------------------------------|
| Retirement Aggregator | `0xda0a793d7c32ab80bcdab7f8c725c96db22464f4` |
| Klima Protocol (AAM)  | `0x1C24239309398220883207681602BfF4D10fbde1` |
| kVCM                  | `0x00fbac94fec8d4089d3fe979f39454f48c71a65d` |
| USDC                  | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` |

CARBON subgraph: https://api.goldsky.com/api/public/project_cmgzise2h00195np2gbp35g3d/subgraphs/cm-base-carbon-production/latest/gn

PROTOCOL subgraph: https://api.goldsky.com/api/public/project_cmgzise2h00195np2gbp35g3d/subgraphs/cm-base-protocol-production/latest/gn

NOTE: The CARBON and PROTOCOL subgraphs are public rate limited endpoints.

### Base Sepolia Testnet (Chain ID: 84532)

| Contract              | Address                                      |
|-----------------------|----------------------------------------------|
| Retirement Aggregator | `0xc0309c29162f699a3445a7a9aeb0acbe568f5fe0` |

## Supported Credit Types

| Credit Type                              | Token Standard | Bridge |
|------------------------------------------|----------------|--------|
| Toucan Puro TCO2                         | ERC-20         | Toucan |
| CMARK (Carbonmark Global Credit Factory) | ERC-20         | CMARK  |
| UCR                                      | ERC-20         | CMARK  |
| ** Coming Soon ** ECO                    | ERC-1155       | ECO    |

## How to Retire Credits

There are two retirement paths:

### Path 1: Direct Retirement (You Own the Credits)

You already hold digital carbon credit assets supported by the Retirement Aggregator and want to retire them.

**Step 1 — Approve the Retirement Aggregator**

For ERC-20 credits (Toucan Puro, CMARK, UCR):

```solidity
IERC20(creditToken).approve(RETIREMENT_AGGREGATOR_ADDRESS, amount);
```

For ERC-1155 credits (ECO):

```solidity
IERC1155(creditToken).setApprovalForAll(RETIREMENT_AGGREGATOR_ADDRESS, true);
```

**Step 2 — Call `retireCredit`**

```solidity
IRetireFacet(RETIREMENT_AGGREGATOR_ADDRESS).retireCredit(
    creditToken, // Address of the carbon credit token
    tokenId,     // ERC-1155 token ID (use 0 for ERC-20 credits)
    batchId,     // Required for Puro retirements; use 0 otherwise
    amount,      // Tonnes to retire
    details      // Retirement metadata (see below)
);
```

**Puro retirements**: To retrieve the current correct `batchId`, query the Klima CARBON subgraph.

```graphql
query GetBatchIdForCredit {
  creditTokens(where: {tokenAddress: "<insert credit address here>"}) {
    batchId
  }
}
```

Note: For subsequent retirements, unless retiring a whole batch in one transaction, you will need to requery for an updated `batchId`.

---

### Path 2: Klima-Mediated Retirement (Pay with kVCM or USDC)

The Retirement Aggregator accepts payment in kVCM or USDC for retirements via Klima Protocol.

**Step 1 — Select a Credit and its Carbon Class**

A carbon class is a class vault address that groups related credits. You must pass the correct class vault for the credit you want to retire.

Select a credit from the Klima dashboard: < link to klima dashboard >

Or query the Klima PROTOCOL subgraph for credits and their carbon classes:

```graphql
query GetKlimaCredits {
  carbonClasses {
    registeredCredits {
      creditAddress
    }
    carbonClassId
  }
}
```

**Step 2 — Get a Quote**

```solidity
(uint256 tonnes, uint256 price, , ) = IRetireFacet(RETIREMENT_AGGREGATOR_ADDRESS).quoteRetireCreditViaKlima(
    creditToken,  // Credit address
    tokenId,      // 0 for ERC-20 credits
    amount,
    inputToken,   // kVCM or USDC address
    carbonClass,  // Carbon class vault address for the credit
    0             // couponTonnes — no coupons currently issued; pass 0
);
```

**Step 3 — Approve the Input Token**

Approval targets differ depending on the input token:

- **kVCM**: The Klima Protocol (AAM) pulls kVCM directly from the caller — approve the AAM:

```solidity
IERC20(KVCM_ADDRESS).approve(AAM_ADDRESS, maxInputTokenIn);
```

- **USDC**: The Retirement Aggregator pulls USDC from the caller and swaps it internally — approve the Retirement Aggregator:

```solidity
IERC20(USDC_ADDRESS).approve(RETIREMENT_AGGREGATOR_ADDRESS, maxInputTokenIn);
```

**Step 4 — Execute the Retirement**

```solidity
IRetireFacet(RETIREMENT_AGGREGATOR_ADDRESS).retireCreditViaKlima(
    creditToken,
    tokenId,
    batchId,          // Required for Puro retirements; see batchId query above
    amount,
    inputToken,       // kVCM or USDC address
    carbonClass,      // Carbon class vault address for the credit
    maxInputTokenIn,  // Max input tokens to spend (slippage protection)
    0,                // couponTonnes — no coupons currently issued; pass 0
    details
);
```

---

## Retirement Details

Every retirement requires a `RetireDetails` struct:

```solidity
struct RetireDetails {
    address retiringAddress;        // Who is retiring (defaults to msg.sender)
    string retiringEntityString;    // Entity name (defaults to "Carbonmark Retirement Aggregator")
    address beneficiaryAddress;     // Who benefits (defaults to retiringAddress)
    string beneficiaryString;       // Beneficiary name
    string retirementMessage;       // Free-text message
    string beneficiaryLocation;     // REQUIRED for Toucan Puro
    string consumptionCountryCode;  // REQUIRED for Toucan Puro
    uint256 consumptionPeriodStart; // REQUIRED for Toucan Puro (unix timestamp)
    uint256 consumptionPeriodEnd;   // REQUIRED for Toucan Puro (unix timestamp)
}
```

**Minimum for CMARK/ECO retirements:**

```solidity
RetireDetails({
    retiringAddress: address(0),        // defaults to msg.sender
    retiringEntityString: "",           // defaults to "Carbonmark Retirement Aggregator"
    beneficiaryAddress: address(0),     // defaults to retiringAddress
    beneficiaryString: "My Company",
    retirementMessage: "Offsetting 2025 emissions",
    beneficiaryLocation: "",
    consumptionCountryCode: "",
    consumptionPeriodStart: 0,
    consumptionPeriodEnd: 0
})
```

**For Toucan Puro retirements**, all location/period fields are required:

```solidity
RetireDetails({
    retiringAddress: address(0),
    retiringEntityString: "",
    beneficiaryAddress: address(0),
    beneficiaryString: "My Company",
    retirementMessage: "Puro removal credit retirement",
    beneficiaryLocation: "Berlin, Germany",
    consumptionCountryCode: "DE",
    consumptionPeriodStart: 1704067200, // 2024-01-01
    consumptionPeriodEnd: 1735689600    // 2025-01-01
})
```

---

## Batch Retirements

Retire multiple credits in a single transaction using `batchCall`:

```solidity
IBatchCallFacet.Call[] memory calls = new IBatchCallFacet.Call[](2);

calls[0] = IBatchCallFacet.Call(
    abi.encodeCall(IRetireFacet.retireCredit, (token1, 0, 0, amount1, details1))
);
calls[1] = IBatchCallFacet.Call(
    abi.encodeCall(IRetireFacet.retireCredit, (token2, 0, 0, amount2, details2))
);

IBatchCallFacet(RETIREMENT_AGGREGATOR_ADDRESS).batchCall(calls);
```

Individual calls in the batch can fail without reverting the entire transaction. The batch only reverts if all calls fail.

---

## Events

Every successful retirement emits:

```solidity
event AggregatorRetired(
    CarbonBridge carbonBridge,           // TOUCAN, CMARK, or ECO
    address indexed retiringAddress,
    string retiringEntityString,
    address indexed beneficiaryAddress,
    string beneficiaryString,
    string retirementMessage,
    address carbonToken,
    uint256 tokenId,
    uint256 batchId,
    HasId hasId,                         // HasTokenId, HasBatchId, HasBoth, or HasNeither
    uint256 retiredAmount
);
```

You can filter by `retiringAddress` or `beneficiaryAddress` since both are indexed.

---

## Error Reference

| Error                                    | Cause                                                                                             |
|------------------------------------------|---------------------------------------------------------------------------------------------------|
| `InvalidAmout()`                         | Retirement amount is zero                                                                         |
| `TokenNotSupported()`                    | Credit token not registered in any configured bridge registry, or input token is not kVCM or USDC |
| `InvalidAmountReceived()`                | Klima returned unexpected amount of credits                                                       |
| `SlippageExceeded(required, maxAllowed)` | USDC needed exceeds your `maxInputTokenIn`                                                        |
| `InvalidBeneficiaryLocation()`           | Toucan Puro: `beneficiaryLocation` is empty                                                       |
| `InvalidCountryCode()`                   | Toucan Puro: `consumptionCountryCode` is empty                                                    |
| `InvalidPeriodStart()`                   | Toucan Puro: `consumptionPeriodStart` is zero                                                     |
| `InvalidPeriodEnd()`                     | Toucan Puro: `consumptionPeriodEnd` is zero                                                       |

---

## Introspection

The Diamond supports [EIP-2535 Loupe](https://eips.ethereum.org/EIPS/eip-2535) for listing all available functions:

```solidity
IDiamondLoupe.Facet[] memory allFacets = IDiamondLoupe(RETIREMENT_AGGREGATOR_ADDRESS).facets();
```
