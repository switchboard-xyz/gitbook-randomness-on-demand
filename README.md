# Switchboard Randomness

Welcome to the Switchboard randomness documentation!

Switchboard Randomness advances the foundational principles of Switchboard-On-Demand by offering unparalleled speed and safety in randomness for blockchain applications. It aims to counteract a broad spectrum of potential threats, ensuring robust support not only for basic applications but also for infrastructure worth billions of dollars.

### Building Trust By Being Trustless

It's crucial to exercise caution when dealing with oracles or services that claim to provide randomness. While randomness itself is a simple concept, ensuring its verifiability can be highly complex.

Let us take a simple scenario:

```
1. Alice decides to participate in a blockchain-based coin flip game, wagering her lunch money.
2. Alice initiates a "flip" transaction. Upon confirmation, the outcome is determined by taking the modulo 2 of the blockhash.
```

From Alice's perspective, this solution appears satisfactory.

> _"I mean, it's always random looking bytes, they don't seem dangerous,"_ she might reassure herself.&#x20;

However, this line of thinking fails to adopt the critical lens of the [Adversarial Mindset](https://papers.ssrn.com/sol3/papers.cfm?abstract\_id=3573099), which is essential in assessing the security and reliability of cryptographic systems.[ ](https://papers.ssrn.com/sol3/papers.cfm?abstract\_id=3573099)

Now lets reimagine the scenario:

What if Alice's wager wasn't just her lunch money, but her life's savings? In such a high-stakes scenario, Alice would undoubtedly probe deeper into the randomness mechanism.&#x20;

Would Alice be content with merely the appearance of randomness, or would she demand robust assurances of its reliability and verifiability?

### Different Entropy Sources and Knowing Your Adversary

#### Dangers of using Latest Blockhash as Randomness

Utilizing the latest blockhash as randomness does impose limitations on who can influence its value at a specific time. However in the context of Solana, each [Solana Leader](https://docs.solanalabs.com/consensus/leader-rotation) has the power and opportunity to determine slot ordering. Thus, at any given moment, the current slot leader becomes your adversary. Over time, your adversary encompasses every Solana slot leader in history.

#### Elliptic Curve Signatures As Entropy

Now that we've covered a randomness situation with no upper bound on adversaries, let's look at improvements. The next course of action then becomes:

_If randomness cannot be sourced on chain, then let's grab some randomness off chain_

You know what's really good at fetching off-chain information for blockchains?

<figure><img src=".gitbook/assets/secrets-giphy-1.gif" alt=""><figcaption><p>Yepppp</p></figcaption></figure>

You guessed it: ORACLES

Oracle's are a common component in blockchain applications for not only data propagation, but for entropy.

Great, we've figured it all out right?&#x20;

<figure><img src=".gitbook/assets/secrets-giphy-2.gif" alt=""><figcaption><p>Well, not exactly...</p></figcaption></figure>

Oracle's don't always get this right..

Let's break it down how things can and HAVE gone wrong:

***

* _**Method:**_ Oracle **chooses** random value every slot
* **Explanation and Limitations**
  * This is, in some ways, better than using the blockhash since you limit the adversary from ANY leader to just the oracle, but the oracle has **complete** control in this situation. And if you are okay with this, then why build in Web3 to begin with?
* _**Adversaries**_
  * The Oracle has FULL control

***

* _**Method:**_ Randomness from elliptic curve signatures
  * A user **requests** randomness by submitting a unique seed, and the oracle responds by:&#x20;
    * `rand = Sha256(Ed25519Sign(oracleSigner, seed))`
* **Explanation and Limitations**
  * Now this method is a good improvement BUT has a baseline misunderstanding of elliptic curve signatures
  * The INTENTION is that no single party chooses the randomness alone;&#x20;
    * The user chooses a seed they can guarantee has never been used before
    * Since the oracle cannot predict that key beforehand, it would not know the output of the signature and the signature can be verified on chain
  * Unfortunately, `Sha256(Ed25519Sign(oracleSigner, seed))` is non-deterministic
    * `Ed25519Sign(oracleSigner, seed)` is actually `Ed25519Sign(oracleSigner, seed, NONCE)` under the hood. This nonce parameter is used to prevent exposing the elliptic curve that makes up the secret key and can be derived deterministically from the signer and message.  A malicious oracle, though, can set this nonce to whatever they please and subsequently produce INFINITE valid signatures resulting in any hash/randomness they please&#x20;
* _**Adversaries**_
  * The Oracle has FULL control

***

* _**Method: User Secrets**_
  1. User generates unique `secret_key`
  2. Users commits wager and `Sha256(secret_key)`
  3. User reveals randomness as `Sha256(secret_key + [commit_slot+1].hash)`
* **Explanation and Limitations**
  * This model plays much in favor of the user, such that it is up to the user to not leak their secret key to the slot leader before commit\_slot+1
  * Although the \***user** could have ultimate adversarial guarantees in this scenario, this scheme is dangerous for the protocol, where the **user**.
  * Since their is no trusted party in this setup (remember, ANYONE can be a leader), the user may offer to coerce the block leader for a favorable slothash. The user could even be the leader themselves!
* _**Adversaries**_
  * No user adversaries, but the user has strong potential to nefariously coerce slot leaders to favor **against** the protocol injesting the randomness

***



## So How Does Switchboard Do It?!?

### The Switchboard Way

With all the risks on the above potential methods, Switchboard takes advantage of the already integrated TEE supported primitives to bring stronger randomness guarantees on-chain:

* _**Method - the TEE oracle**_
  1. User commits to use the latest slothash as a seed and commits an oracle to use
  2. User awaits next slot production and requests randomness from the TEE oracle, using the unique slothash
     1. (Even the slot leader is incapable of preempting the oracle to reveal entropy before the seed slot is finalized)
  3. Oracle responds with entropy from the TEE and signs with a TEE secured secret
* **Explanation and Limitations**
  * Using Switchboard TEEs, we can incorporate the added confidentiality security primitives SGX has to offer
  * The oracle will never reveal randomness seeded with the latest slothash to prevent adversaries from "peaking"
  * When the slothash is finalized, the commit  transaction MUST have been completed and the user may request randomness from the oracle
  * Since the Switchboard On-Demand oracle network processes all requests in-enclave, the oracle operator could not have foreseen the randomness before responding
* _**Adversaries**_
  * For an adversary to predict this randomness they would need all of the following:
    * Be the next slot leader of the chain
    * Access to the oracle authority
    * Know of an active unpatched TEE vulnerability

***

## Future Work

* Currently the randomness oracle queries the solana RPC itself to ensure it doe not reveal randomness for a still-confidential randomness slot.
* Rather than relying on the oracle rpc, or creating a slothash oracle, integrating a light client (see [tinydancer](https://docs.tinydancer.io/)) would make grabbing the current slothash of the chain verifiable within the TEE

## Gotchas

* Protocols must be intentful on when a user "commits" to using a randomness value and when randomness is "revealed"
* Since oracles rotate enclave keys about once weekly, we do not let oracles that are within an hour of rotation commit to producing a random value.
* If "reveal" is not settled within an hour of commit, the randomness request will be considered as expired and protocols should register this as a manageable user flow
* If you need flexibility around this deadline, please contact the Switchboard team for customizability on expiration options.

## Getting the Picture

To say it again, fair and verifiable randomness is not as straightforward as you would think, but Switchboard put that footwork in for you. We hope this gives you some insight on how these technologies work and how Switchboard keeps you safe.  With that, Switchboard is happy to offer TEE secured randomness today on Solana Devnet!
