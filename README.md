# A proof-of-concept for batching meta transactions

When you say "meta transactions" people think of **gasless** transactions, which means that someone else (the relayer) makes an onchain token transaction for you and pays for it in Ether. In return, you can pay the relayer in tokens (instead of Ether).

The problem with the current implementation of meta transactions is that it allows the relayer to **relay only one meta tx at a time**. While this allows the original (meta) tx sender to avoid using ETH, it doesn't lower the transaction cost for her/him, because the relayer has to be compensated in tokens in approx. the same (or higher) value as the gas fees for the onchain transaction.

The solution is to **batch multiple meta transactions** into **one onchain transaction**.

This would **lower the cost** of a meta tx which should be the main purpose of using meta transactions.

## The implementation

The implementation should be pretty straightforward. A user sends a meta transaction to a relayer. Relayer waits for multiple meta txs to come up in a mempool until the meta tx fees (at least) cover the cost of the onchain gas fee.

### What if the relayer forges a meta tx?

Every meta tx should be digitaly signed with a user's Ethereum private key. The token smart contract must then check the validity of a signature (`ecrecover()`).

### What if the relayer sends the same meta tx twice (replay attack)?

The meta tx must include a nonce. Hence the token smart contract must somehow do validation based on the nonce.

Perhaps the easiest would be to keep a **separate mapping for nonces** only:

```solidity
// hashmap of meta tx nonces for each address. New nonce must be always bigger.
mapping(address => uint256) private __nonceOf;
```

Every time a meta transaction is completed, the smart contract should update the latest nonce in the mapping. The rule should be that a new meta tx must not have a nonce lower than the previous meta tx (of the same user/address). This way it's easier to check the validity of nonces.

### What data is needed in a meta tx?

- sender address (the user who is sending the meta tx)
- receiver address
- token amount to be transfered
- relayer fee (in tokens)
- nonce

### What is the relayer fee?

The meta tx sender defines how big fee they want to pay to the relayer.

### How can the sender know which relayer to pay the fee to?

The sender does not know that because there are probably multiple relayers that can pick the meta tx and relay it onchain.

Instead, the token smart contract will take care of this. The smart contract knows who the relayer is (`msg.sender`) and can then give the relayer the appropriate amount of tokens based on all the relayer fees in all the meta txs (in that onchain tx).

### Can a relayer pass meta transactions to different token contracts in one onchain tx?

Not sure about that, probably not. But if we're talking about a token that has a lot of transactions, this shouldn't be a problem. Otherwise the relayer might want to wait longer until the sufficient amount of meta txs (for a certain token) join the mempool.

### Where is the mempool?

The relayers should sync mempools between each other, although because they are competing, this might not happen.

As a solution, there could be nodes that only serve as mempool holders. Not sure how to incentivise them, though (maybe the big dApps maintainers like Synthetix and Aave would want to run them).

Another option could be that the meta tx sender sends the meta tx to many relayers at once.

### What about approve and allowance?

This proof-of-concept only targets the basic token transfer functionality, so the approve txs are not needed here (might be added later). But it should work in a similar way.
