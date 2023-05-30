# hypercerts_audit
**Protocol Overview**
Hypercerts is a protocol that allows for Impact Funding Systems (IFSs) to be built efficiently on the blockchain. 
Users can mint ERC1155 semi-fungible tokens that are like "certificates" for work. 
Those "certificates" can be configured to be fractionalized and/or transferable based on rules. 

There are 3 ways for anyone to create a new Hypercert (also called a "claim"):
1. Calling mintClaim which will mint all of the units of the token to the caller
2. Calling mintClaimWithFractions which will split the token to fractions and mint them to the caller
3. Calling createAllowlist which will create an ERC1155 token that will need a whitelist access (via a Merkle tree) for a caller to mint it
For the functionality in 3. the following methods for minting a new Hypercert were added:
1. mintClaimFromAllowlist - the caller can mint to account by submitting a proof which authorizes him to mint units amount of the claimID type token
2. batchMintClaimsFromAllowlists - same as mintClaimFromAllowlist but for multiple mints in a single transaction
And for the owners of Hypercerts/claims the following functionalities exist:
1. splitValue - split a claim token into fractions
2. mergeValue - merge fractions into one claim token
3. burnValue - burn claim token


Threat Model

System 
* Protocol owner - can upgrade the HypercertMinter contract and pause/unpause it, set during protocol initialization
* Claim minter - can create a new claim type, no authorization control
* Whitelisted minter of a claim - can mint tokens from an already existing type, Merkle tree root is set during creation of the type
* Type creator - if policy is FromCreatorOnly only him can transfer the tokens
* Fraction owner - can transfer, burn, split and merge fractions of a claim
Q: What in the protocol has value in the market?
A: The claims and their fractional ownership are valuable because they might receive rewards in the future from (for example) retroactive funding and/or be used for governance.
Q: What is the worst thing that can happen to the protocol?
1. Stealing/burning claims by a user who doesn't own and isn't an operator of the claim
2. Generating more units than intended via splitting or merging
3. Unauthorized upgrading/pausing of the contract

Interesting/unexpected design choices:
The mintClaimFromAllowlist method checks msg.sender to be included in the Merkle tree but the token is minted to the account address argument instead. The same is the case for the batchMintClaimsFromAllowlists functionality where msg.sender should be in all of the leafs.
Minting a token with only 1 unit means it won't be splittable at a later stage. The UI recommends 100 fractions on mint - maybe this should be enforced on the smart contract level as a minimum value.

Severity classification
Severity	Impact: High	Impact: Medium	Impact: Low
Likelihood: High	Critical	High	Medium
Likelihood: Medium	High	Medium	Low
Likelihood: Low	Medium	Low	Low

Scope
The following smart contracts were in scope of the audit:
* AllowlistMinter
* HypercertMinter
* SemiFungible1155
* Upgradeable1155
* interfaces/**
* libs/**
The following number of issues were found, categorized by their severity:
* Critical & High: 2 issues
* Medium: 2 issues
* Low: 4 issues
* Informational: 12 issues


Findings Summary
		
		
		
		
ID	Title	Severity
[C-01]	Users can split a token to more fractions than the units held at tokenID	Critical
[H-01]	Calling splitValue when token index is not the latest will overwrite other claims' storage	High
[M-01]	Unused function parameters can lead to false assumptions on user side	Medium
[M-02]	Input & data validation is missing or incomplete	Medium
[L-01]	Comment has incorrect and possible dangerous assumptions	Low
[L-02]	Missing event and incorrect event argument	Low
[L-03]	Prefer two-step pattern for role transfers	Low
[L-04]	Contracts pausability and upgradeability should be behind multi-sig or governance account	Low
[I-01]	Transfer hook is not needed in current code	Informational
[I-02]	Unused import, local variable and custom errors	Informational
[I-03]	Merge logic into one smart contract instead of using inheritance	Informational
[I-04]	Incorrect custom error thrown	Informational
[I-05]	Typos in comments	Informational
[I-06]	Missing override keyword for interface inherited methods	Informational
[I-07]	Bit-shift operations are unnecessarily complex	Informational
[I-08]	Redundant check in code	Informational
Detailed Findings



[M-02] Input & data validation is missing or incomplete

Severity
Impact: High, because in some cases this can lead to DoS and unexpected behaviour
Likelihood: Low, as it requires malicious user or a big error on the user side

Description
Multiple methods are missing input/data validation or it is incomplete.
1. The splitValue method in SemiFungible1155 has the maxIndex[_typeID] += len; code so should also do a notMaxItem check for the index
2. The createAllowlist method accepts a units argument which should be the maximum units mintable through the allowlist - this should be enforced with a check on minting claims from allowlist
3. The _createAllowlist of AllowlistMinter should revert when merkleRoot == ""
4. The _mintClaim and _batchMintClaims methods in SemiFungible1155 should revert when _units == 0 or _units[i] == 0 respectively



[M-02] Input & data validation is missing or incomplete

Severity
Impact: High, because in some cases this can lead to DoS and unexpected behaviour
Likelihood: Low, as it requires malicious user or a big error on the user side




[L-03] Prefer two-step pattern for role transfers
The Upgradeable1155 contract inherits from OwnableUpgradeable which uses a single-step pattern for ownership transfer. It is a best practice to use two-step ownership transfer pattern, meaning ownership transfer gets to a "pending" state and the new owner should claim his new rights, otherwise the old owner still has control of the contract. Consider using a two-step approach.


[L-04] Contracts pausability and upgradeability should be behind multi-sig or governance account
A compromised or a malicious owner can call pause and then renounceOwnership to execute a DoS attack on the protocol based on pausability. The problem has an even wider attack surface with upgradeability - the owner can upgrade the contracts with arbitrary code at any time. I suggest using a multi-sig or governance as the protocol owner after the contracts have been live for a while or using a Timelock smart contract.


[I-09] Incomplete or wrong NatSpec docs
The NatSpec docs on the external methods are incomplete - missing @param, @return and other descriptive documentation about the protocol functionality. Also part of the NatSpec of IAllowlist is copy-pasted from IHypercertToken, same for AllowlistMinter. Make sure to write descriptive docs for each method and contract which will help users, developers and auditors.
