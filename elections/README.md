# Elections Smart Contract

## Deployments

|   environment   |                           address |
| :-------------: | --------------------------------: |
|  mainnet prod   |          `elections.ndc-gwg.near` |
| mainnet testing | `elections-v1-1.gwg-testing.near` |
|     testnet     |        `elections-v1.gwg.testnet` |

## Requirements

- Only I Am Human verified accounts can vote.
- Each account can vote at most one time. Votes are not revocable, and can't be changed.
- Contract has a fair voting `policy` attribute. Each user, before voting, has to firstly accept the policy by making a transaction matching the contract policy.
- Only the authority (set during contract initialization) can create proposals. Each proposal specifies:

  - `typ`: must be HouseType variant
  - `start`: voting start time as UNIX time (in miliseconds)
  - `end`: voting start time as UNIX time (in miliseconds)
  - `cooldown`: cooldown duration when votes from blacklisted accounts can be revoked by an authority (in miliseconds)
  - `ref_link`: string (can't be empty) - a link to external resource with more details (eg near social post). Max length is 120 characters.
  - `quorum`: minimum amount of legit accounts to vote to legitimize the elections.
  - `seats`: max number of candidates to elect, also max number of credits each user has when casting a vote.
  - `min_candidate_support`: minimum amount of votes a candidate needs to receive to be considered a winner.

## Flow

- GWG deploys the elections smart contract and sets authority for creating new proposals.
- GWG authority creates new proposals before the election starts, with eligible candidates based on the `nominations` result. All proposals are created before the elections start.
  - NOTE: we may consider querying the candidates directly from the nominations contract.
- Once the proposals are created and the elections start (`now >= proposal.start`), all human verified near accounts can vote according to the NDC Elections [v1 Framework](../README.md#elections).
- Anyone can query the proposal and the ongoing result at any time.
- Voting is active until the `proposal.end` time.
- Vote revocation is active until the `proposal.end` + `cooldown` time.

## Bonding

- [SPEC](https://github.com/near-ndc/gov/blob/main/framework-v1/elections-voting.md#bonding)
- Each verified voter must bond 3N to cast their vote. Each Non-verified voter must bond 300N to cast their vote.
- Bond can be deposited using `bond` function that must be used via is_human_call.

  ```rust
  near call REGISTRY is_human_call '{"ctr": "elections.near", "function": "bond", "paylad": "{}"}' --accountId YOU.near --deposit 3
  ```

- One bond is enough to cast votes for all proposals.
- `finish_time`: max(`finish_time`, `end` + `cooldown`) of all the proposals.
- User can unbond after the `finish_time`. All tokens minus storage fees will be returned.
- Bonded tokens can be slashed by executing `vote_revoke`. 100% of bonded tokens will be slashed and will be tracked in `total_slashed` variable.
- `unbond`: To unbond deposit, unbond function needs to be called via IAH `registry.is_human_call`.

  ```rust
  near call REGISTRY is_human_call '{"ctr": "elections.near", "function": "unbond", "payload": "{}"}' --accountId YOU.near
  ```

  The `unbond` will also mint I VOTED SBT for [eligible voters](https://github.com/near-ndc/gov/blob/main/framework-v1/elections-voting.md#i-voted-sbt).

## Voting

User who made sufficient bond and accepted Fair Voting Policy can call `vote` function to vote for an active proposal.
User can vote at most once for each proposal, votes can not be updated. [Full specification](https://github.com/near-ndc/gov/blob/main/framework-v1/elections-voting.md)

### Setup Package

Setup Package proposal is a proposal with `seats=1` (at most one option can be selected) and `candidates = ["yes", "no", "abstain"]`.

Voting for setup package uses the same API as voting for candidates. The vote must be list of exactly one element: `["yes"]` or `["no"]` or `["abstain"]`.

## Usage

Below we show few CLI snippets:

```shell
CTR=elections-v1.gwg.testnet
REGISTRY=registry-1.i-am-human.testnet

# create proposal
# note: start time, end time and cooldown must be in milliseconds

near call $CTR create_proposal '{"start": 1686221747000, "end": 1686653747000, "cooldown": 604800000  "ref_link": "example.com", "quorum": 10, "candidates": ["candidate1.testnet", "candidate2.testnet", "candidate3.testnet", "candidate4.testnet"], "typ": "HouseOfMerit", "seats": 3, "min_candidate_support": 5}' --accountId $CTR

# fetch all proposal
near view $CTR proposals ''

# query proposal by ID
near view $CTR proposals '{"prop_id": 2}'

# accept fair voting policy
near call $CTR accept_fair_voting_policy '{"policy": "f1c09f8686fe7d0d798517111a66675da0012d8ad1693a47e0e2a7d3ae1c69d4"}' --deposit 0.001 --accountId me.testnet

# query the accepted policy by user. Returns the latest policy user accepted or `None` if user did not accept any policy
near call $CTR accepted_policy '{"user": "alice.testnet"}' --accountId me.testnet

# bonding - see a section above how to bond and unbond

# query if a IAH holder bonded (by IAH SBT)
near view $CTR has_voted_on_all_proposals '{"sbt": 123}'

# vote
near call $CTR vote '{"prop_id": 1, "vote": ["candidate1.testnet", "candidate3.testnet"]}' --gas 70000000000000 --deposit 0.0005 --accountId me.testnet

# revoke vote (authority only)
near call $CTR admin_revoke_vote '{"prop_id": 1, "token_id": 1}'

# revoke vote (anyone can call this method)
near call $CTR revoke_vote '{"prop_id": 1, "user": "alice.testnet"}'

# check if a user voted for all proposals (note user votes with SBTs, so it may happen that
# we should query by TokenID instead)
near view $CTR has_voted_on_all_proposals '{"user": "alice.testnet"}'

# query winners by a proposal
# NOTE: the function doesn't return "ongoing" winners, it only returns a valid response once
# the proposal finished (voting ended and is past the cooldown).
near view $CTR winners_by_proposal '{"prop_id": 1}'
```

## Deployed Contracts

### Mainnet

Coming Soon

- mainnet testing: `elections-v1.gwg-testing.near` - [deployment tx](https://explorer.mainnet.near.org/transactions/k8CYckfdqrubJovPTX8UreZkdxgwxkxjaFTv955aJbS)
  registry: `registry-v1.gwg-testing.near`

### Testnet

- `elections-v1.gwg.testnet` - [deployment tx](https://explorer.testnet.near.org/transactions/6mQVLLsrEkBithTf1ys36SHCUAhDK9gVDEyCrgV1VWoR).
  registry: `registry-1.i-am-human.testnet`
