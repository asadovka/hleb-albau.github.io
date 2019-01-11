---
layout: post
title:  "Your chain genesis based on Ethereum network activity"
date:   2019-01-11
excerpt: ""
keywords:
  - drop
  - airdrop
  - eth
  - ethereum
  - genesis
  - chain
  - blockchain
  - crypto
---

 Recently, I worked on launching our chain [**cyberd**](https://github.com/cybercongress/cyberd) 
public testnet.
To attract active crypto users we decided to create genesis,
where 70% of genesis tokens will be dropped to Ethereum users. 
But how to select those users? Yeah, it a bit complicated question and worthy of a separate post. 
All we need to know for now, that we can select them from other chain based on their activity. 
But ... how to transfer them tokens safely? 


## When crypto helps

 Most of the chains based on same crypto principle. A user generates private\public keys pair. 
Keeping secret and signing transaction using a private key, 
announcing public key to others, thus they can verify transaction signer is you. 
Here we come to the interesting part: 
you can use a same private\public pair for different chains 
if they use a same signature verification mechanism. 
Knowing that we can assign tokens(based on some calculated rank) to **public** keys in our genesis chain, 
so that only users holding corresponding private keys can **unlock** assets.
Seems easy, but ...


## Address is not public key

 A normal user doesn't see any private or public keys. 
Usually, key management is hidden by crypto wallets. 
Also, we are not working with public keys directly. 
Instead, chains generate more short and readable **addresses**, derived from public key.
Mostly each chain has own address derivation logic, 
thus different chains for the same public key will generate different addresses.
For example, **Ethereum** use next algorithm to obtain your account address:
```text
1. Start with the public key (64 bytes)
2. Take the Keccak-256 hash of the public key. You should now have a string that is 32 bytes.
3. Take the last 20 bytes of this public key (Keccak-256). These 20 bytes(40 characters) are the address. 
 When prefixed with 0x it becomes 42 characters long.
```

 One point I forgot to mention is that public key can be restored from a private one. Now we have next triple: private key -> public key -> address. Our drop scheme is transforming to:

1. Calculate a rank for initial chain addresses. One of the best candidates currently is Ethereum.
2. Parse initial chain and extract public keys for addresses from (1).
3. Derive from collected public keys your chain addresses.
4. Based on (1) drop assets to derived addresses in your genesis.

 As I said earlier, the first part challenging itself. 
There are a lot of goals your drop can aim. 
According to drop intention you want to select a different group of user from an initial chain. 
It can be wales, or centralized exchange users or some dapp active participators. 

 The third and fourth parts are just regular coding routine. 
But the task of collecting public keys a bit tricky.


## Collecting public keys

 When you create a first outgoing transaction, you also attach your public key. 
Usually, it posted in compressed format as curve points.
So, to collect public keys you have to download all first accounts outgoing transactions, 
restore public key to uncompressed format if necessary and put it somewhere for later access.
Seems not complicated, but here we also should deal with a network, disk, or system errors. 
Also synced full node of an initial chain is required.

 In our chain we use Ethereum is the initial chain. 
I found only one GitHub repository implementing required logic, but with some specific features.
So I create [simple CLI app](https://github.com/hleb-albau/ethereum-pubkey-collector) 
to collect Ethereum public keys. 
Contributions are welcome!
Currently, it takes about 1 day to parse whole Ethereum network. 
This project, besides collecting public keys, supports other commands such as create CSV file or serve HTTP endpoint. 


## Limitations

 There are few limitations with the current drop approach.
The first is about security. Users should trust your chain wallets to import their private keys.
I strongly suggest using different keys for different chains. 
The best tactic here is to install wallet withing offline computer, 
just sign transaction, copy transaction bytes using a flash driver and delete wallet. 
Then a signed transaction can be sent via any chain provider securely.

 The second limitation: only addresses with outgoing transactions can be used for a drop. 
No out transaction - no public key. 
It is almost impossible to reveal your private key using your public key, 
but some big holders prefer to use cold wallets with only incoming transactions for additional security.

 Despite described limitations,
generating dropping tokens based on others chains activity is very powered tool.
Used wisely it can boost your projects from start. Buidl!