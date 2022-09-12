# ✨ So you want to sponsor a contest

This `README.md` contains a set of checklists for our contest collaboration.

Your contest will use two repos: 
- **a _contest_ repo** (this one), which is used for scoping your contest and for providing information to contestants (wardens)
- **a _findings_ repo**, where issues are submitted (shared with you after the contest) 

Ultimately, when we launch the contest, this contest repo will be made public and will contain the smart contracts to be reviewed and all the information needed for contest participants. The findings repo will be made public after the contest report is published and your team has mitigated the identified issues.

Some of the checklists in this doc are for **C4 (🐺)** and some of them are for **you as the contest sponsor (⭐️)**.

---

# Contest setup

## ⭐️ Sponsor: Provide contest details

Under "SPONSORS ADD INFO HERE" heading below, include the following:

- [ ] Create a PR to this repo with the below changes:
- [ ] Name of each contract and:
  - [ ] source lines of code (excluding blank lines and comments) in each
  - [ ] external contracts called in each
  - [ ] libraries used in each
- [ ] Describe any novel or unique curve logic or mathematical models implemented in the contracts
- [ ] Does the token conform to the ERC-20 standard? In what specific ways does it differ?
- [ ] Describe anything else that adds any special logic that makes your approach unique
- [ ] Identify any areas of specific concern in reviewing the code
- [ ] Add all of the code to this repo that you want reviewed


---

# Contest prep

## ⭐️ Sponsor: Contest prep
- [ ] Provide a self-contained repository with working commands that will build (at least) all in-scope contracts, and commands that will run tests producing gas reports for the relevant contracts.
- [ ] Make sure your code is thoroughly commented using the [NatSpec format](https://docs.soliditylang.org/en/v0.5.10/natspec-format.html#natspec-format).
- [ ] Modify the bottom of this `README.md` file to describe how your code is supposed to work with links to any relevent documentation and any other criteria/details that the C4 Wardens should keep in mind when reviewing. ([Here's a well-constructed example.](https://github.com/code-423n4/2021-06-gro/blob/main/README.md))
- [ ] Please have final versions of contracts and documentation added/updated in this repo **no less than 24 hours prior to contest start time.**
- [ ] Be prepared for a 🚨code freeze🚨 for the duration of the contest — important because it establishes a level playing field. We want to ensure everyone's looking at the same code, no matter when they look during the contest. (Note: this includes your own repo, since a PR can leak alpha to our wardens!)
- [ ] Promote the contest on Twitter (optional: tag in relevant protocols, etc.)
- [ ] Share it with your own communities (blog, Discord, Telegram, email newsletters, etc.)
- [ ] Optional: pre-record a high-level overview of your protocol (not just specific smart contract functions). This saves wardens a lot of time wading through documentation.
- [ ] Delete this checklist and all text above the line below when you're ready.

---

# Y2K Finance contest details
- $47,500 USDC main award pot
- $2,500 USDC gas optimization award pot
- Join [C4 Discord](https://discord.gg/code4rena) to register
- Submit findings [using the C4 form](https://code4rena.com/contests/2022-09-y2k-finance-contest/submit)
- [Read our guidelines for more details](https://docs.code4rena.com/roles/wardens)
- Starts September 14, 2022 20:00 UTC
- Ends September 19, 2022 20:00 UTC

[ ⭐️ SPONSORS ADD INFO HERE ]
# Protocol overview
Y2K Finance is a protocol to insure(hedge) against pegged assets and speculate on depegs.  
The aim of this protocol is to allow users to HEDGE & RISK these insurances, also providing yield to incentivize early depositors on both side of insurance (buyers and sellers).  
We consider insurance sellers to be the same as Risk users, or collateral depositors. And insurance buyers to be the same as Hedge users, or premium depositors.  
This is made possible by having Vaults that leverage ERC4626 and a Chainlink Oracle to determine the current price.  
When the price of a pegged asset is below the strike price of a Vault, a Keeper(anyone actually) will trigger the depeg event and both Vaults(hedge and risk) will swap their total assets with the other party.  
These insurance's last a certain amount of time, and if a depeg event does not happen between that time, a Keeper will trigger the end of epoch event, Hedge Vault then has to give all of it's total assets to the Risk Vault.
It is issued a token representing the depositors position which is 1 to 1 ratio'ed to the deposited amount, and users can deposit that token in a fork of Synthetix's Rewards to "yield farm" governance tokens. 

# Smart Contracts - PRIORITY
Besides these Contracts and their libraries, the only interaction with external contracts is the Chainlink Oracle's

### Controller.sol - HIGH
Smart Contract used to call Oracles latest price data, and trigger epoch end or depeg events. They can be triggered if the Id of that Market/Vault epoch is less then the current block.timestamp, or if the current price of that insured pegged asset is below the Vault's Strike Price.
These functions will then call that Market, figure out the pair of Vaults of that Market, and transfer the underlying assets between the Risk/Hedge parties. Must mention that inside this contract we convert all price feed results to 18 decimal points, so we can compare to the Strike Price passed by the admin on "VaultFactory.sol"
> Responsability:  
- Trigger Depegs;  
- Trigger end of Epoch;  
- Check Oracle Price Feed converted to 18 decimals;  

### oracles/PegOracle.sol - HIGH
Since Y2K used pegged assets, chainlink does not have the convertion ratio from a Token to it's pegged Token, example: stETH to ETH. Since Chainlink does not have it, we implemented an Oracle that divides the price of stETH by the price of ETH and get a ratio from it. If that ratio is 1, the Token is pegged, otherwise is depeged.
> Responsability:  
- Calculate ratio between two given Oracles;  

### SemiFungibleVault.sol - MEDIUM
Y2K leverages ERC4626 Vault standard for this protocol, this contract is a fork of that standard, although we replaced all uses of ERC20 to ERC1155. This is done because we want vaults to have multiple tokens that represent each epoch, so in this contract, ID for the ERC1155 mints has to be the UNIX timestamp of the epoch end date. Also we want to make this into an EIP, Semi Fungible MultiToken Vaults.  
> Responsability:  
- Deposit;
- Withdraw;  

### Vault.sol - HIGH
Implements "SemiFungibleVault.sol", a Vault in this protocol is defined by it's market, a Vault/Market is thus a combination of these params: Token Insured & Strike Price.  
Each Vault has then multiple epochs, defined by the ERC1155 Id, allows deposits in WETH only, and mints ERC1155 backed 1 to 1 by that WETH, deposits can only be done until a certain date, called Epoch Begin, minus a variable called Timewindow which is used to make some time between the start of the insurance epoch and last day of deposits.  
Withdraws of WETH can only happen in case "Controller.sol" triggers the end of epoch or triggers a depeg event. Withdraws have a fee associated with it, which is sent to the treasury, such FEE is represented in this form: multiply your %(percent) value by 10, Example: if you want fee of 0.5% , insert 5. Fees are attached to epochs timestamps!  
To make withdraw amount calculations after the swap of assets, we calculate how much % a user owned of the vault's total assets prior to swap of assets using finalTVL to log that totalAssets, and then give him the % he owns of the vault after the swaps. Token symbol is used here to differentiate between Risk and Hedge Vault as the calculations for withdraw for both vaults differ!  
Must mention that we want to make these Vaults used by DAO's, so we accept deposits and withdraws for (on behalf of) other users, by using approve ERC1155 functions on withdraw, and recipient/owner params inside both deposit/withdraw functions.  
These contracts have the functions needed to swap assets between Vaults, from Hedge to Risk vault and vice-versa.
Also we allow deposits in ETH, but we convert it to WETH.
> Responsability:  
- Deposit;
- Withdraw;  

### VaultFactory.sol - MEDIUM
Factory for the creation of Vault pairs. A pair of Vaults constructs a Market, a Hedge and a Risk Vault. As said before Markets are determined by it's Token Insured & Strike Price, and have an Index associated with it.  Since Markets have Vaults that always come in pairs, some functions use that assumption for their logic, for example arrays that return the vault addresses have in the [0] position the Hedge Vault and on the [1] position the Risk Vault.
This contract allows the Admin role to change params inside the Vault, and also the Oracle associated with the insured token.  
Must mention that we expect admin to pass the StrikePrice as a 18 decimal int, so we can compare to the Oracle Price using the same decimals.
> Responsability:  
- Create pair of Vaults (Markets);
- Change Variables;  

### rewards/StakingRewards.sol - LOW
Y2K allows farming of Y2K token using these Synthetix forked contracts, we made some adjustments to these contracts, replaced all mentions of ERC20 to ERC1155 to allow deposits in "Vault.sol" tokens. And passed some params like RewardRate and RewardDuration to the constructor for admin use.
> Responsability:  
- Stake;
- Withdraw;  

### rewards/RewardsFactory.sol - LOW
Contract used to deploy the pair of "StakingRewards.sol" to reflect the Hedge/Risk vaults implementation above.  
Must mention we used a keccack256 to hash a market index from "VaultFactory.sol" and it's "Vault.sol" respective market epochEnd/id to represent the StakingRewards vaults as an index to be able to query it later.
> Responsability:  
- Create pair of Farms from Markets;

# Tests

# Concerns & Aditional Information

# Prepare Local Environment

# Testnet Deployment
