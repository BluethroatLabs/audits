# Canto Name Service Code Review

## Conducted by Bluethroat Labs
### Auditor(s): Rahul Saxena (twitter.com/saxenism)

![Bluethroat Logo](https://www.bluethroatlabs.com/static/media/1-removebg-preview.0c9129a71f5e01fb531c.png)

#### Codebase

The commit of the codebase audited is as follows:
a3fe70bf2bdbbff1c812ff7711a744dee101d943

If you want to view the codebase, click [here](https://github.com/Zodomo/canto_name_service/blob/a3fe70bf2bdbbff1c812ff7711a744dee101d943/src/CantoNameService.sol).

## Disclaimer

As of the date of publication, the information provided in this report reflects the presently held understanding of the auditor’s knowledge of security patterns as they relate to the client’s contract(s), assuming that blockchain technologies, in particular, will continue to undergo frequent and ongoing development and therefore introduce unknown technical risks and flaws. The scope of the audit presented here is limited to the issues identified in the preliminary section and discussed in more detail in subsequent sections. The audit report does not address or provide opinions on any security aspects of the Solidity compiler, the tools used in the development of the contracts or the blockchain technologies themselves, or any issues not specifically addressed in this audit report.
 
The audit report makes no statements or warranties about the utility of the code, safety of the code, suitability of the business model, investment advice, endorsement of the platform or its products, the legal framework for the business model, or any other statements about the suitability of the contracts for a particular purpose, or their bug-free status.
 
To the full extent permissible by applicable law, the auditors disclaim all warranties, express or implied. The information in this report is provided “as is” without warranty, representation, or guarantee of any kind, including the accuracy of the information provided. The auditors hereby disclaim, and each client or user of this audit report hereby waives, releases and holds all auditors harmless from, any and all liability, damage, expense, or harm (actual, threatened, or claimed) from such use.

## Issues Found

### 1. Function `Allowlist.deleteReservation` can potentially become unusable

#### Explanation 

Consider the scenario where the owner included the address of a contract as `address _reserver` and all reservation mapping variables were set according to it.

Now, if the owner of that contract decided to call `deleteReservation`, they would not be able to do it since the `tx.origin` would always be the owner (EOA's address).

The function `CantoNameService.burnReservation` introduces the contracts to tx.origin phishing

#### Mitigation Suggestion

+ Consider using the msg.sender in the function `deleteReservation`.
+ Try whitelisting the CantoNameService contract address for Allowlist.deleteReservation
+ Or try using delegatecall
+ Or introduce a signature mechanism that authorises someone to burn a particular reservation.

### 2. Risk of reuse of signatures across forks due to lack of chainID validation

#### Explanation

The Allowlist contract computes the domain separator in the function `computeDomainSeparator` using the network's chainID, which is fixed at the time of deployment.

In the event of a post-deployment chain fork, the chainID cannot be updated, and the signatures may be replayed across both versions of the chain.

#### Mitigation Suggestion

To mitigate this risk, if a change in the chainID is detected, the domain separator can be cached and regenerated. Alternatively, instead of regenerating the entire domain separator, the chainID can be included in the schema of the signature passed to the order hash.

### 3. Single step contract ownership

#### Explanation

The owner of the Allowlist contract can be changed by calling the `transferOwnership` function in OpenZeppelin's Ownable contract. This function internally calls the `transferOwnership` function, which immediately sets the contract's new owner. Making such a critical change in a single step is error-prone and can lead to irrevocable mistakes.

#### Mitigation Suggestion

Implement a two-step process to transfer contract ownership, in which the owner proposes a new address and then the new address executes a call to accept the role, completing the transfer.

## General Recommendations

This section consists of general recommendations given to the maintainers of the protocol based on the overall design and coding philosophy of the contracts.

These are not necessarily security flaws and are dependent completely on the maintainers' discretion for their implementation.

0. Currently, there is no mechanism to protect users who want to buy a particular `name` from being `front-run`. For example, `Alice` wants to buy `sbf`, and submits a transaction for same. She could however be easily front run by a bot that wants the same name and provides some bribe to the miners/validators in the form of more gas fee. However, implementing front-run protection mechanisms would be a business decision and is left upto protocol maintainers to decide.
1. Rethink usage of _msgSender against msg.sender (depends on the business requirements of the project)
2. In the `Allowlist.sol` contracts, `getReservation`, `getReserver` and `getReservationExpiry` are redundant functions and can be removed since all three mappings are public and therefore will have a default getter function.
3. Declaring the constructors as `payable` will shave off some gas.
4. Consider declaring `uint256 cutoff` in `Allowlist.sol` as an immutable variable since it is set only once inside the constructor and not changed after that.
5. Consider using `ECDSA.recover` from OpenZeppelin instead of the default `ecrecover`.
6. Review the contract logic again with cognizance of the fact that `block.timestamp` can be manipulated by miners with the following constraints:
+ it cannot be stamped with an earlier time than its parent
+ it cannot be too far in the future
7. Consider changing the visibility of the `public` functions across different contracts to external. Saves gas.
8. In `Allowlist.sol`, instead of setting the `nameReserver` to address(0) in function `reserveName`, consider deleting that particular entry. It has the same effect as well as gets you some gas refunded.

```
nameReserver[nameReservation[msg.sender]] = address(0);

/////////////////////
// Recommended
/////////////////////

delete nameReserver[nameReservation[msg.sender]];
```
9. Consider changing the faulty error message in the function `transferFrom`. Change from `ERC721::safeTransferFrom::NOT_APPROVED` to `ERC721::transferFrom::NOT_APPROVED`.
10. Instead of setting the `nameRegistry[tokenId].name` and `nameRegistry[tokenId].expiry` in function `ERC721._burnData`, consider deleting them. It has the same effect as well as gets you some gas refunded.
11. Same as earlier point, but in the function `ERC721._afterTokenTransfer`.
12. In `CantoNameService.sol` Reentrancy guard is imported but never used. Can be removed.
13. For string parameters in functions that are not mutating them, switch to using `calldata` instead of `memory` to save gas. For example: Can change from `string memory _newBaseURI` to `string calldata _newBaseURI`. 
14. Don't initialize variables to their default value. For example: `uint256 charByteCount` is already initialized to 0. No need to re-initialize.
15. Pre-increments can be cheaper than post-increments. We can make this change in the implemented `for` loops
16. The use of `total` in the `tokenCounts` struct is not clear. Check if you remove it?
17. For a cleaner code consider creating a modifier that checks whether the passed _VRGDA is between [1 and 5]
18. unchecked{++i} would be more gas-efficient than `i++` in the function `vrgdaBatch` and other places where `for` loops are used.
19. The `vrgdaPrep` function has to be called 5 times for the `vrgdaBatch` function to work. Perhaps there is a more gas-efficient way to get the data for the entire batch?
20. In the function `_validateReservation`, the `require` statement should be `allowlist.getReservationExpiry(msg.sender) > block.timestamp`, instead of the used `allowlist.getReservationExpiry(msg.sender) >= block.timestamp`
21. A delegate cannot call the function `burnReservation` 
22. According to the logic for function `_delegate`, if a name owner wants to delegate to someone while another delegation is active, they will not be able to do so. For example, if A has delegation till time T and I want to delegate B from time T+1, I cannot do that before T. Is that something that the protocol is ok with?
23. In the function `extendDelegation(uint256 _tokenId, uint256 _term)`, check for overflow of newDelegationExpiry or make sure you have contingencies if the `uint newDelegationExpiry` gets overflown.
24. For LinearVRGDA.sol: In the comment of `int256 targetPrice` in struct `VRGDA`, there is a typo: `tokenamen` -> `token name` 
25. Shorter `require` strings can reduce contract size.
26. Comparing a boolean *looks* like an overkill but is perfectly alright logically

```
// Require reservation not used, currently restricts to one reservation
    modifier reservationValid() {
        require(reservationUsed[msg.sender] != true, "Allowlist::reservationValid::RESERVATION_USED");
        _;
    }

/////////////////////
// Recommended
/////////////////////

// Require reservation not used, currently restricts to one reservation
    modifier reservationValid() {
        require(!reservationUsed[msg.sender], "Allowlist::reservationValid::RESERVATION_USED");
        _;
    }
```

27. Remove `TODO` tags from the codebase. Makes users nervous.
28. Prepend internal/private functions and variable names with underscore (`_`).

## Protocol/Logic Review

Part of our audits are also analyses of the protocol and its logic. The `Bluethroat Labs` team went through the implementation and documentation of the implemented protocol.

Partial tests and no documentation has been provided to us, but the function and inline documentation are sufficient to fully understand the intended protocol that is being implemented.

According to our analysis, the protocol and logic are working as intended, given that the findings listed in the *Issues* section are fixed.

We were not able to discover any additional problems in the protocol implemented in the smart contract.



