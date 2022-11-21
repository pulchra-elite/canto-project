# Canto contest details
- Total Prize Pool: $24,500 worth of CANTO
  - HM awards: $17,000 worth of CANTO
  - QA report awards: $2,000 worth of CANTO
  - Gas report awards: $1,000 worth of CANTO
  - Judge + presort awards: $4,000 worth of CANTO
  - Scout awards: $500 USDC
- Join [C4 Discord](https://discord.gg/code4rena) to register
- Submit findings [using the C4 form](https://code4rena.com/contests/2022-11-canto-contest/submit)
- [Read our guidelines for more details](https://docs.code4rena.com/roles/wardens)
- Starts November 23, 2022 20:00 UTC
- Ends November 28, 2022 20:00 UTC

## C4udit / Publicly Known Issues

The C4audit output for the contest can be found [here](add link to report) within an hour of contest opening.

*Note for C4 wardens: Anything included in the C4udit output is considered a publicly known issue and is ineligible for awards.*

# Table of Contents

1. [Contest Scope](#contest-scope)
2. [Project Overview](#project-overview)
3. [Cosmos SDK Module](#cosmos-sdk-modules)
4. [Smart Contracts](#smart-contracts)

---

## Contest Scope:

The contest scope includes:

Code for a new cosmos module (`x/csr`).

Code for a smart contract (`contracts/turnstile.sol`), which is the same as (`CIP-001/src/Turnstile.sol`). The smart contract is a modification of ERC 721 while `Canto/x/csr` is a standard cosmos module that was scaffolded by Ignite CLI.

*List all files in scope in the table below -- and feel free to add notes here to emphasize areas of focus.*

| Contract | SLOC | Purpose | Libraries used |  
| ----------- | ----------- | ----------- | ----------- |
| contracts/folder/sample.sol | 123 | This contract does XYZ | [`@openzeppelin/*`](https://openzeppelin.com/contracts/) |

## Out of scope

*List any files/contracts that are out of scope for this audit.*

---

## Project Overview:

We present a novel economic mechanism which modifies EIP-1559 to distribute a portion of the total base fee (an amount that would otherwise be burnt) to the deployers of the contracts that consume gas within a given block.  Our goal is to implement the CSR protocol with as few changes as possible to the existing EIP-1559 specification while also providing a simple and flexible user experience.

---

### CSR Store

The Canto CSR Store is a revenue-sharing-per-transaction model that allows smart contract developers to accumulate revenue to a tradable NFT.  In this model, developers deploy smart contracts that generate revenue via transaction fees that go directly to an NFT. Developers register their dApps with a special CSR smart contract that mints an NFT or adds smart contracts to an existing NFT. The split between transactions fees that go to network operators and NFTs is implemented and configurable by the `x/csr` module.

---

### Turnstile Smart Contract

- [smart contract code](https://github.com/code-423n4/2022-11-canto/tree/add-readme/CIP-001)

On the application layer, CSR functions as a series of smart contracts responsible for generating and maintaining a registry of eligible contract addresses. As a contract creator, participation is on an opt-in basis. Should a contract creator choose to deploy a CSR enabled contract, they must integrate support for the CSR Turnstile, described in the section below. Upon deployment of a CSR enabled contract, the contract creator is minted a CSR NFT. This NFT acts as a claim ticket for all future fees accrued. Smart contract developers can add smart contracts to existing NFTs. Smart contracts that are written using the factory pattern can be automatically CSR-enabled when the turnstile code is injected. The CSR Turnstile contract is deployed by the module account upon genesis.

---

### CSR NFT Smart Contract

The CSR NFT Smart contract is an extension of ERC721 and is deployed by the module account on genesis. Upon registration of a smart contract, the CSR module account will mint a new NFT from the CSR NFT smart contract. The register defaults to minting a new NFT as the `beneficiary` and sending that NFT to `fromAddr`, but the function can be called to assign an existing NFT as the beneficiary or send the newly minted NFT to another address. The `beneficiary` must call the withdrawal method on the smart contract along with an NFT ID to retrieve transaction revenue.

---

### Registration

Developers register their application in the CSR Store by 

1. Injecting turnstile code – `register` or `assign` – into their smart contracts
    1. `Turnstile` can be called with two possible function signatures. One will allow the user to add the deployed smart contract to an existing NFT (`assign`), the other will allow the user to mint a new NFT (`register`).
    2. either function will have a corresponding event with `msg.sender` being the new smart contract that needs to be registered.
    3. on the client side, the `PostTxProcessing` hook will listen for registration events coming from the turnstile address and will update the CSR store accordingly.

```solidity
// register the smart contract to an existing CSR nft
function assign(uint64 _tokenId) public {
		....
    emit UpdateCSREvent(msg.sender, id);
}

// register and mint a new CSR nft that will be transferred 
// to the to address entered
function register(address to) public {
		....
    emit RegisterCSREvent(msg.sender, to);
}
```

---

### EVM Transaction Fees

When a transaction is executed, the entire gas fee amount is sent to the `FeeCollector` module account during the Cosmos SDK AnteHandler execution. After the EVM transaction finishes executing, the user receives a refund of `(gasLimit - gasUsed) * gasPrice`. In total, a user will pay a gas fee of `txFee = gasUsed * gasPrice` to complete an arbitrary transaction on the EVM. This transaction fee is distributed between the NFTs minted by the CSR smart contract and network operators (validators). The distribution between the CSR smart contract and network operators is defined as follows, 

```go
	// Calculate fees to be distributed = intFloor(GasUsed * GasPrice * csrShares)
	fee := sdk.NewIntFromUint64(receipt.GasUsed).Mul(sdk.NewIntFromBigInt(msg.GasPrice()))
	csrFee := sdk.NewDecFromInt(fee).Mul(params.CsrShares).TruncateInt()
	evmDenom := h.k.evmKeeper.GetParams(ctx).EvmDenom
	csrFees := sdk.Coins{{Denom: evmDenom, Amount: csrFee}}

```

`csrFees` is then sent to `Turnstile.sol` smart contract that was deployed by the module account. Users can then withdraw their revenue by calling `withdraw(uint256 _tokenId, address payable _recipient, uint256 _amount)` on the `Turnstile`.

---

### Fee Distribution

Any set of registered contracts will be associated with a NFT. Each NFT will have a different set of smart contracts accumulating revenue. As such, distributing transaction fees requires the implementation of a `beneficiary` account which will be accumulating rewards on behalf of the NFT. Every NFT will have a single `beneficiary` account which is sent transaction revenue when smart contracts pertaining to the NFT the `beneficiary` belongs to. When users withdrawal revenue from the NFT, they are sending funds the the `beneficiary` account to their own account.

Users are **lazily** allocated fees. This means that each user withdraws all fees they have accrued since the last time they have withdrawn from a NFT. 

----

## Cosmos SDK Modules:

### Contract Secure Revenue Cosmos Module

---

************Keeper / Client / Types (164 LOC)************

Most of the keeper, client, and types code was scaffolded by ignite and filled in by the developers. It follows the standard Cosmos paradigm of writing module code.

`x/csr` stores the following state the address of the turnstile smart contract and CSR objects which look like the following:

```protobuf
// The CSR struct is a wrapper to all of the metadata associated with a given CST NFT
message CSR {
    // Contracts is the list of all EVM address that are registered to this NFT
    repeated string contracts = 1;
    // The NFT id which this CSR corresponds to
    uint64 id = 2;
    // The total number of transactions for this CSR NFT
    uint64 txs = 3;
    // The cumulative revenue for this CSR NFT -> represented as a sdk.Int
    string revenue = 4 [
        (gogoproto.customtype) = "github.com/cosmos/cosmos-sdk/types.Int",
        (gogoproto.nullable) = false
    ];
}
```

****************EVM Hook****************

- event_handler.go (49 loc)
    - This defines the events that the module will be looking out for in the EVM hook defined and implemented in evm_hooks.go.
- evm_hooks.go (62 loc)
    - This is where the core fee distribution logic exists. TLDR is every transaction will have the set of events that were emitted. We check if the tx had either a `assign` or `register` event emitted, internally store the contract to its associated NFT if necessary, and distribute the fees accordingly.
    - If the smart contract was previously registered, we check if the smart belongs to some CSR NFT by looking through the keeper. If so, we distribute
- evm.go (47 loc)
    - Contains all of the functionality that allows us to interact with the EVM and `Turnstile` contract.

## Smart Contracts:

- Turnstile.sol (54 loc)
    - Contains the functionality described in the Turnstile and CSR NFT sections listed above
    - Uses open zeppelin ERC721
