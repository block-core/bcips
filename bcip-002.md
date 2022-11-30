# BCIP-0002 : Atomic Swaps

```
Number:  BCIP-0002
Title:   Atomic Swaps
Type:    Informational
Status:  Draft
Authors: Blockcore <post@blockcore.net>
Created: 2022-11-30
```

## Abstract

Enabling trustless swaps between Blockcore blockchains

## Motivation

Right now blockchain networks on Blockcore need to use centralized exchanges to be able to swap coins.  
With atomic swaps anyone will be able to swap coins without a 3rd party dependency,  
additionally teams will be able to provide liquidity for onboarding new users on their blockchain network.  

## High level flow

To enable trustless swaps we use HTLCs with a secret hash, the secret hash is a one time unique number that is hashed in to the ScriptPubkey, 
it is the link between the transactions on each chain.  
The link is done in such a way that when the initiator of the swap claims their swap coins on chain A they must 
reveal the secret that allows the other party to be able to claim coins on chain B.  

We define a seller and a buyer, the seller is the one initiating the swap and defining the swap amount and exchange rate,  
the seller also generates the secret hash.  Seller can be considered as providing swap liquidity.  
Then buyer is subscribing to swaps made by sellers.  


### The Flow

1. A seller creates a swap entry and defines the coin pair, the amount and the exchange rate. 
The seller can at this point generate the secret and share the hash of the secret.  
The seller will also shares their public key that will be used for the swap script.  
2. A buyer subscribes to the swap and shares their public key that will be used for the swap script, at this point the swap is locked.
3. The seller creates a MaxReorg + 48h locked HTLC swap transaction on chain A sending coins to the buyers pubkey and the hash of the secret, and broadcast it on chain A.  
4. IMPORTANT! The buyer waits for X confirmations until they are satisfied the swap transaction on chain A won't be reorged (the safest is to wait is MaxReorg blocks but perhaps for small amounts it can be settled faster)  
5. The buyer creates a MaxReorg + 24h locked HTLC swap transaction on chain B sending coins to the sellers pubkey and the hash of the secret, and broadcast it on chain B.  
6. The seller can now claim the coins on chain B using their pubkey and the secret, they broadcast a transaction claiming the swap coins.  
7. By revealing the secret the buyer can now claim the swap coin son chain A using their pubkey and the secret.  
8. Swap is complete!

### Claiming Coins of an Aborted Swap

Continuing from step 5, in case the seller does not claim the swap coins.

6. If after MaxReorg +24h the seller has not claimed the swap coins (and revealing the secret)
7. The buyer uses their pubkey to claim back the coins in the HTLC swap trx on chain B.  
8. After MaxReorg +48h the seller can claim  back the swap coins locked in the HTLC using their pubkey
9. Aborted swap completed!  

### The Swap Scripts 

The HTLC ScriptPubkey that will lock the swapped coins

```
 OP_IF
     OP_HASH160<H(shared_secret)>
     OP_EQUALVERIFY
     <receiver-key>
     OP_CHECKSIG
 OP_ELSE
     <time_lock>
     OP_CHECKLOCKTIMEVERIFY
     OP_DROP
     <sender-key>
     OP_CHECKSIG       
 OP_ENDIF

```

The ScriptSig to claim the swap coins using the secret
```
<sig> <secret> 1 <redeemScript>
```

The ScriptSig to abort the swap after the timeout
```
<sig> 0 <redeemScript>
```

To avoid broadcasting issues all scripts will be P2SH or P2WSH

### HD derivation of swap keys and secret hash

We will reuse the swap pubkey the derivation we define for the swap pubkey is 
`m / purpose' / coin_type' / account' / swap_key' / secret `

purpse = 2
coin_type = as defined in slip44
account = 0
swap_index = 0
secret = unique and incrementing for each swap

Deriving the secret allows us to recover swap wallets in the future

### Privacy
Reusing the `swap_key` means swaps can be linked on chain, if this is an issue for privacy then the swap index should be incremented for each swap 

## Known problems

**Optionality**  
One of the issue in this swap processes is the problem of optionality.
This is when the buyer waits for the seller to claim their coins, if the price has fluctuated on other exchanges    
the seller can wait with the swap until the HTCL lock time before acting.
To solve that we can potentially make a much smaller lock time on the buyers side 


## References

A collections of URLs with atomic swaps samples and discussions

https://blog.bitmex.com/atomic-swaps-and-distributed-exchanges-the-inadvertent-call-option
https://bitcointalk.org/index.php?topic=193281.msg2224949#msg2224949
https://medium.com/blockchainio/what-are-atomic-swaps-bc1d034634c9
http://atomic.bitcoinscri.pt/pages/about
https://en.bitcoin.it/wiki/Atomic_swap
http://diyhpl.us/wiki/transcripts/scalingbitcoin/tokyo-2018/atomic-swaps
https://diyhpl.us/wiki/transcripts/scalingbitcoin/tokyo-2018/edgedevplusplus/cross-chain-swaps
https://en.bitcoin.it/wiki/Contract#Example_5:_Trading_across_chains
https://github.com/decred/atomicswap/blob/7c6928f1d1a8cd6ebe4bb6e0a0c907d423f75e13/cmd/btcatomicswap/main.go#L1137
