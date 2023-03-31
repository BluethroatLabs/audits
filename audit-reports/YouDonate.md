# YouDonate Protocol Security Review

A security review of the [YouDonate-Protocol](https://github.com/YouDonate-Protocol/YouDonate-Protocol) smart contract protocol was done by [Parth](https://twitter.com/__parthpatel__) and [Rahul](https://twitter.com/saxenism). \
This audit report includes all the vulnerabilities, issues and code improvements found during the security review.

## Disclaimer

"Audits are a time, resource and expertise bound effort where trained experts evaluate smart
contracts using a combination of automated and manual techniques to find as many vulnerabilities
as possible. Audits can show the presence of vulnerabilities **but not their absence**."

\- Secureum


### Impact

- **High** - leads to a significant material loss of assets in the protocol or significantly harms a group of users.
- **Medium** - only a small amount of funds can be lost (such as leakage of value) or a core functionality of the protocol is affected.
- **Low** - can lead to any kind of unexpected behaviour with some of the protocol's functionalities that's not so critical.

### Likelihood

- **High** - attack path is possible with reasonable assumptions that mimic on-chain conditions and the cost of the attack is relatively low to the amount of funds that can be stolen or lost.
- **Medium** - only conditionally incentivized attack vector, but still relatively likely.
- **Low** - has too many or too unlikely assumptions or requires a huge stake by the attacker with little or no incentive.

### Actions required by severity level

- **Critical** - client **must** fix the issue.
- **High** - client **must** fix the issue.
- **Medium** - client **should** fix the issue.
- **Low** - client **could** fix the issue.

## Executive summary

### Overview

|               |                                                                                              |
| :------------ | :------------------------------------------------------------------------------------------- |
| Project Name  | Crypta Digital                                                                           |
| Repository    | https://github.com/YouDonate-Protocol/YouDonate-Protocol                                                           |
| Commit hash   | [d849dfb80015331976f247cdee46630a8f2a11cd](https://github.com/YouDonate-Protocol/YouDonate-Protocol/tree/d849dfb80015331976f247cdee46630a8f2a11cd) |
| Documentation | Not provided                                 |
| Methods       | Manual review                                                                                |
|               |

### Issues found

| Severity      |                                                     Count |  
| :------------ | -------------------------------------------------------- 
| High risk     |       6| 7 |
| Medium risk   |     2 | 2| 
| Low risk      |        0 | 2| 
| Informational | 5 | 4|
| Gas | 8| |


# Findings

## High severity

### [H-1] Incorrect checking of range of ticketID in `viewRewardsForTicketId`

#### **Context**

- https://github.com/YouDonate-Protocol/YouDonate-Protocol/blob/d849dfb80015331976f247cdee46630a8f2a11cd/contracts/YDTSwapLottery.sol#L546-L551

#### **Description**

The above require condition will never be satisfied and always evaluate to false.

#### **Recommended Mitigation Steps**

This condition does not require an `&&` operator but rather an `||` operator


### [H-2] Incorrect Price Calculation in function `_calculateTotalPriceForBulkTickets`

#### **Context**

- https://github.com/YouDonate-Protocol/YouDonate-Protocol/blob/d849dfb80015331976f247cdee46630a8f2a11cd/contracts/YDTSwapLottery.sol#L641-L647

#### **Description**

The problem could arise in the following cases:
* `_numberTickets > 1 + _discountDivisor` 
* Maximum Value of `_numberTickets = 100`
* Minimum Value of `_discountDivisor = 300`

Normally this won't be an issue. However, if someone calls the function `setMaxNumberTicketsPerBuy`, then this function will start reverting since the price will become negative and all related transactions will revert.


#### **Recommended Mitigation Steps**

Come with a new formula for the discounted price or account for the above mentioned cases

### [H-3] Function `getPriceData` will not work in certain cases

#### **Context**

- https://github.com/YouDonate-Protocol/YouDonate-Protocol/blob/main/contracts/YDT.sol#L82-L107

#### **Description**

When the token in question has less than 3 decimal places or do not have a chainLinkFeed, the function would fail. This function would not work if the `chainLinkFeed` for the particular token does not exist.


#### **Recommended Mitigation Steps**

Include checks for the cases described above.

### [H-4] Members can `processProposal` before duration.


#### **Context**

- https://github.com/YouDonate-Protocol/YouDonate-Protocol/blob/main/contracts/SimiDAO.sol#L272

#### **Description**

The function can be called when the proposal voting time has expired. To either act on the proposal or cancel if not a majority yes vote.

Consider the following line of code:

```
require(getCurrentTime() >= proposals[proposalId].startingTime, "voting period has not started");
```

This is wrongly checking the start of the voting period rather than its expiration.

#### **Recommended Mitigation Steps**

The condition to check should be

```
getCurrentTime() > startingTime + duration
```


### [H-5] Users cannot claim reward for any digits less than 5 matching the winning number.

#### **Context**

- https://github.com/YouDonate-Protocol/YouDonate-Protocol/blob/main/contracts/YDTSwapLottery.sol#L221-L226
- https://github.com/YouDonate-Protocol/YouDonate-Protocol/blob/main/contracts/YDTSwapLottery.sol#L216-L219

#### **Description**

In the function `claimTickets`, if 4 last digits are matching and user gets a reward, the block of code will revert and not give user any rewards.

If the protocol wanted to impose the condition of only users with all last 5 digits matching, that should have been checked earlier.

If both the above mentioned code blocks exist, it would not be possible for this function to work properly without reverting.



#### **Recommended Mitigation Steps**

Reconsider the reward mechanism or remove this condition.

### [H-6] DoS can happen in `claimTickets` function.

#### **Context**

- https://github.com/YouDonate-Protocol/YouDonate-Protocol/blob/main/contracts/YDTSwapLottery.sol#L191-L207

#### **Description**

Since this function `claimTickets` is only callable by the user and they'll be the one passing their function params, this function refuses to give out **ANY LEGITIMATE** reward even if only one of the `ticketIds` is passed incorrectly. 

For example 
```
require(rewardForTicketId != 0, "No prize for this bracket");
```

The `require` used in the for loop will revert for **Any** wrong ticketId and the user will not get **ANY** reward. 


#### **Recommended Mitigation Steps**

Make sure that the users can retrieve their funds even if one entry of their `ticketsIds` is correct.

## Medium severity

### [M-1] The `modifier notContract` breaks EIP4337 wallets

#### **Context**

- https://github.com/YouDonate-Protocol/YouDonate-Protocol/blob/main/contracts/YDTSwapLottery.sol#L83-L87

#### **Description**

The issue is that the `msg.sender == tx.origin` check forces people to use EOA wallets instead of smart contract wallets or EIP-4337-style account abstraction.

Also, for the other check of `_isContract`, the following things need to be kept in mind:
It is unsafe to assume that an address for which this function returns false is an externally-owned account (EOA) and not a contract.
     * Among others, `isContract` will return false for the following types of addresses:
     *  - an externally-owned account
     *  - a contract in construction
     *  - an address where a contract will be created
     *  - an address where a contract lived, but was destroyed
     
Furthermore, `isContract` will also return true if the target contract within the same transaction is already scheduled for destruction by `SELFDESTRUCT`, which only has an effect at the end of a transaction.

Preventing calls from contracts is highly discouraged. It breaks composability, breaks support for smart wallets like Gnosis Safe, and does not provide security since it can be circumvented by calling from a contract constructor.


#### **Recommended Mitigation Steps**

Either make a conscious decision to leave out people using the EIP-4337 style wallets or redo the design of the lottery itself.


### [M-2] Missing Sanity Checks for `constructor` initialized params.


#### **Context**

- https://github.com/YouDonate-Protocol/YouDonate-Protocol/blob/main/contracts/YDTSwapLottery.sol#L116-L127
- https://github.com/YouDonate-Protocol/YouDonate-Protocol/blob/main/contracts/SimiDAO.sol#L96-L105

#### **Description**

Incorrectly initialized constructor can lead to loss of time and economic resources. It would be better to include sanity checks to avoid them.


#### **Recommended Mitigation Steps**

Add sanity checks for `_YDTtokenAddress` and `_randomGeneratorAddress`
Consider using `_randomGeneratorAddress != address(0)` and `require(_isContract(randomGeneratorAddress))`


## Informational

### [I-1]  Ease of usage
In function `startLottery` , the parameter `uint256 _priceTicketInYDT` as it is. It is recommended for the ease of the user to create a mechanism to enter dollar value and convert it to equivalent YDT tokens.

### [I-2] Improve comments
In `YDTSwapLottery`, there is a line of code which says the following: `uint256 public constant MAX_LENGTH_LOTTERY = 7 days; // 4 days
`
Recommendation: Remove the 4 days comment and change it to 7 days.

### [I-3] Usage of `safeTransfer` instead of `transfer`
`YDTSwapLottery` uses SafeERC20 for ERC20 operations.

Recommendation:  can use transfer since token YDT is trusted.

### [I-4] Swap of variables
The names of `userNumber` and `winningTicketNumber` are swapped in function `_calculateRewardsForTicketId`

### [I-5] Hardcoded values
In function `donateToProject`, there are hardcoded values of 95% and 5%. No way to adjust the percentages of splitting the amount.



### Gas

### [G-1] 
Save `_ticketNumber.length` as uint256 numTicketLength in the function buyTickets

### [G-2] 
Consider using custom errors instead of require strings to save gas.

### [G-3] 
For simple mathematical operations that are guaranteed not to overflow, consider using unchecked blocks.

### [G-4]
Do not initialize variables that are being initialized now with their current value. For example: `uint256 baseYield = 0;` can be changed to `uint256 baseYield;`   


### [G-5] 
Pre-increment operators are cheaper than post-increment operators

### [G-6] 
Do not read values for length inside of a loop to save gas.

### [G-7] 
Consider changing the public functions which aren’t called anywhere else inside the contracts to external to save on gas costs.

### [G-8] 
In function `setMinAndMaxTicketPriceInYDT`, consider replacing <= with < .





