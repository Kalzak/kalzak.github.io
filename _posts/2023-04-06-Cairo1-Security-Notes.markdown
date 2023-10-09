---
layout: post
title:  "A collection of StarkNet/Cairo1 security tips"
date:   2023-4-07 11:57:16 +0800
---

NOTE: This article was partially completed and I've moved on, so it's written in a casual style but the information is still good so I want to keep this post up for people to read.

# Introduction

This post aims to be a collection of security tips for people interested in Starknet/Cairo1.
Some of these tips apply to all blockchain contracts, and some are specifically applicable to SN/Cairo.
Even for the general tips, they're written with StarkNet in mind and have examples using StarkNet contracts.
Hopefully this can be some help, and if you think there is anything you would like to add, feel free to message me at [@0xKalzak](https://twitter.com/0xKalzak).
But before we dive into the technicals let's just do a quick recap on StarkNet and Cairo.

# Quick background on StarkNet

StarkNet is an L2 network for Ethereum that aims to address Ethereum's scalability issues. 
It uses zero-knowledge proofs to prove that StarkNet execution has occurred, without actually having to re-execute to verify.
That way you can verify StarkNet computation on Ethereum in a super cheap way.
In order to make program execution verifiable with zk-proofs there's a lot of math behind the scenes, and execution must be mathematically "provable".
This leads well into the next section...

# Quick background on Cairo

First there was Cairo 0.X which was very painful. Now we have Cairo1.0 which is much nicer because it's rust-like (well still a bit painful if you don't know rust).
Remember how execution needs to be mathematically "provable"? Well there are some interesting features about program execution on Starknet.
Memory is immutable and sequential. You write to a slot, then move to the next and write again. There are plenty of other restrictions too.
To handle this, there are some layers to compilation. You have "Cairo" which compiles to "Sierra' which is the intermediary language.
Finally the sierra code can be compiled into the CASM (cairo assembly) that will actually be deployed on-chain.
There are some other quirks like dealing with recursion instead of loops, but in the future that'll be abstracted away to improve dev UX anyways.

---

## Reentrancy

This is when a contract makes a call to another contract, but the CALLED contract makes a call back to the calling contract in an unexpected way. 
You might have state in your victim contract that could be using outdated data, and then can do operations using the outdated data.
That's the issue described in the most simple way possible. This could be done with pricing data, balance tracking, permission bits, or anything else.
Always be aware of external calls and how they could come back into the contract.
You can defend yourself from this in two ways.
You could have a function that acts as a reentrancy preventer. On a first call, that function would set the "reentreed" bit to true.
On a reentrancy call, that bit would already be set to true so it would revert.
This works fine, but there is also a design pattern which can help address reentrancy too.
The Check-Effect-Interactions patterns. Rememeber how I said that outdated data could be accesses and used in the reentrancy call?
Well the idea behind this pattern is to do checks, then effects (IE: update the data) and then interactions.
That way, during the reentarancy call you contract can't have outdated state, because the state would have been updated _before_ the external call.

## Arithmetic precision

Computer math can be different to pen and paper math. In smart contracts usually arithmetic precision comes down to integer divsion.
Integer division will never show the remained, only the number of times that `x` can divide into `y` cleanly.
Imagine a protocol that has some fee on transfer amount, they want to charge a 1% fee.
They calculate this by doing:
```
let feeAmount: u256 = 100 * transferAmount / 10000
```
The idea being that 100 * anything divided by 10000 will lead to 100/10000 = 1/100 = 1% of the original number.
An example input could be 2500 tokens. 
```
100 * 2500 / 10000
// This results in 25, which is 1% of 2500
```
Works good here, but remeber that this is integer division. So it only shows how many times `100 * 2500` _cleanly_ divided with `10000`.
So if we pass `2550`, we see that the fee amount is the same, even though we technically are transferring more tokens.
```
100 * 2550 / 10000
// Still results in 25, which is 0.980392156% of 2500
```
Instead of 1% it's 0.98% which isn't too bad. But with integer division, if the numerator is less than the denominator then the result will be zero.
So if you wanted to pay no fees, just make sure that `100 * x` is less than `10000`.
```
100 * 99 / 10000
// Results in zero, because 100 * 99 = 9900 which is less than 10000
```
Now you can pay no fees thanks to integer division. Arithmetic precision can be very important. 

On the topic of arithmetic precision, if you're going to multiply and divide in one statement, it always makes sense to multiply first and then divide.
If you divide and then multiply, any presision loss or change from the division will be multiplied once the multiplication is done. 
Better to multiply and then divide. But if arithmetic precision really matters to you, then a fixed point math library is recommended.

## Weak randomness

If your protocol is going to rely on randomness, you need a good source for it. Currently, the starknet validation is centralized, as it's all handled by StarkWare.
In the future, it will be opened up so anybody can run a StarkNet node. Also it's just best practice to have good randomness so get in the habit now.
Blockchains are deterministic. Timestamps, hash outputs, block numbers, etc. Trying to source randomness from these could allow people to predict or control the random outcome.
If you are going to be using randomness, it's recommended that you use a good source. There are already some services online that allow good random number generation.
I'll link to some here: ...
A simple example of randomness could be a lottery. Let's say they decide the random win for the lottery based on the timestamp. It could be controlled, or manipulated to wait.
If it costs $100 per attempt, you don't want to get a wrong guess. How could an attacker deal with this?
An attacker could create a smart contract that only calls the function when the timestamp meets the win reqirements for the lottery contract, now it's a risk free win.

## Signature replay attacks

But more generally, you always want to make sure that you have a nonce system in place because otherwise that opens you to replay attacks.
You can also implement timestamps into your signatures, to ensure that even with the nonce, the signature would only be valid for some specified time window.
In your smart contract if you really wanted, you could also have a storage map that tracks whether a nonce has been used. 
The storage map approach would be costly as you have to interact with storage, so it would be a last line of defense, but definitely the strongest.
Sure, on StarkNet we're using pedersen/poseidon instead of keccak256 so generally sigatures will be incompatible, but what about mainnet vs testnet?
Even if you have nonces is this case, you might not be safe, because they could repeat the same transactions you've done on the testnet onto mainnet.
The solution is to make use of the chainid as well. That way you can have different signatures for testnet vs mainnet, which is very important.

## Uninitialized state

If you have a storage variables, it'll be set to zero by default. Not "empty", but zero. This is important to be aware of.
Let's say you have a contract with 1000 tokens in it, and you have to call "propose()" to propose that the funds should be unlocked.
Once some time period has passed, you can then call "claim()" to claim the 1000 tokens after the time has passed.
```
struct Storage {
    waittime: u64
}

fn propose() {
    let blocktimestamp = getblocktimestamp();
    waittime::write(blocktimestamp + 7days)
}

fn claim() {
    let current_time = getblocktimestamp();
    let allowed_time = waittime::read();
    if allowed_time < current_time {
        transferFundsToCaller();
    }
}
```

In this example here, `waittime` is our storage variable. If we call "propose" the unlock time is written there. Only once the current time is greater than the unlock time we can take.
But state is defaulted to zero. So we could simply decide to call "claim()" without making a call to "propose()", because if we don't call "propose" then `waittime` will be zero.
Then if we call "claim" without making the call to "propose" then `allowed_time` will be zero, which is smaller than the current time, so the funds can be transferred!

## Felt252 vs uint256

From what I have read and heard, the idea is to abstract away the concept of felts so that developers stick with the normal signed and unsigned integers.
This is a great idea, because it helps keep familiarity between Ethereum developers if they want to try StarkNet out.
It's also great because felt can come with some footguns that you need to be aware of. 
I'm not going to go into great detail on the inner workings of felts here, but if you want, you can have a read here (link cairo docs explaining felts).
Basically, we have the type `felt252`. When you read this you might think you have 252 bits to work with, which is true, but isn't at the same time.
StarkNet proves execution with zero-knowledge, which as I said before is a whole bunch of math behind the scene.
As a part of this math, the native datatype used by StarkNet is the felt. A property of the felt it can be a max size of some specific prime number minus one.
Prime in this case is: (`PRIME = 2**251 + 17 * 2**192 + 1`).
This prime is actually less than the max integer you could fit in 252bits. 
This is very important to be aware of, because of the next notable thing about felts: you can overflow/underflow them.
You should try to use the `u2^n` eg `u256` types wherever possible, but if you are using felts, be aware of these two properties.
If you aren't aware, they could work together to cause some big problems.
```
let halfof_2pow252: felt252 = 2^252/2

let max252_as_u256: u256 = 2^252-1
let max252_as_felt252: felt252 = halfof_2pow252 + halfof_2pow252 - 1

max252_as_u256.print()      // Correct value
max252_as_felt252.print()   // Incorrect, the result of overflow when reached prime-1
```

## View functions modifying state

Currently the `#[view]` tag is just a marker, but it is still possible to change state within these functions. 
I have seen this myself in previous Cairo audits, so it does happen and it is something you should keep in mind.
Since it's just a marker right now, developers may assume it doesn't modify state, but it actually can and that could lead to vulnerabilities.
Trusting something to not modify state, when it actually does could be really bad.
```
struct Storage {
    something: felt252,
}

#[view]
fn get_variable() -> felt252 {
   let old_something = something::read();
   something::write(12); // Oops, no compiler or runtime issues here
   return old_something;
}
```

## Library calls

In StarkNet we have librarycalls, which are the equivalent to Ethereum's delegatecalls, with some slight differences.
In Ethereum a delegatecall will use the logic at the designated contract address. So you often find many deployed contracts all with the same implementation, being used separately.
StarkNet introduces the concept of a `class_hash`. Each compiled Cairo contract will point to one particular class hash which is unique to it.
This is super neat because for commonly used implementations you don't need to declare the contract code every time, you just use the existing classhash.
This isn't just use for proxies though, _every_ contract has a classhash.
So when you want to use a library call, you pass the classhash of the implementation you want to use, rather than the address of a contract using that classhash.
So with library calls and classhashes explained, let's touch on the security concern.
A library calls allows some other contract to make changes to its own state. It definitely has its uses (eg: proxy), but much care should be taken when using this.
You need to be sure that the classhash you are making the library call to is trusted, so that it behaves how you would expect.
If you allow library calls based on arbitrary input that any user could control, they could use this to change the state of the contract to whatever they like.
The following is an example
```
// Library call contract here
```

## Classhash changes and upgrades

If you have a contract with upgradability as part of its design (either as a proxy or though `replace_class`, you need to be very careful.
Unlike Solidity which runs on storage slots 0,1,2... for declared storage variables, StarkNet determines the location based on hashing the name.
This can be great, because now you don't run into the issue of adding a variable in the middle and messing up all storage reads/writes.
But at the same time, developers who are familiar with Solidity should be aware that changing storage variables names in updated implementations will point to a different slot.
This could lead to loading empty slots or in a case where one variable name is replaced with another, it could read non-zero data expected to be used elsewhere.
Be careful when changing an implementation with classhash, make sure you aren't messing with your storage.
```
// Contract originally has a reward only for staking.
// When the wrote it, there was only one reward so they just called it "reward"
oldContract {
    struct Storage {
        tokenAddress: ContractAddress,
        rewardAmount: felt252,
    }

    fn claimRewards() {
        let rewards = rewardAmount::read();
        transferToCaller(rewards);
    }
}

// The shiny new implementation has rewards for staking AND holding
// So they want to make a difference in the variable names
// That way it's a better dev experience when reading the code, more specific (:
newContract {
    struct Storage {
        tokenAddress: ContractAddress,
        stakeRewardAmount: felt252,
        holdRewardAmount: felt252,
    }

    fn claimRewards() {
        let stakeRewards = stakeRewardAmount::read();
        let holdRewards = holdRewardAmount::read();
        transferToCaller(stakeRewards + holdRewards);
    }
}
```
Looks great right? Well, what about the users who had been staking _before_ the implementation upgrade? Well, their rewards are located at the key derived from the name "rewardAmount".
But the new implementation has named it "stakeRewardAmount", and that'll point to a different storage key, so it'll be empty.
That means after the upgrade all existing users will lose their rewards.

## starknet keccak vs ethereum keccak

As cited [here](https://docs.starknet.io/documentation/architecture_and_concepts/Hashing/hash-functions/#starknet_keccak), the `keccak` hash function on StarkNet behaves differently to what might be expected on Ethereum.
This is due to the native `felt252` type not being able to contain a 256bit hash digest. 
The `sn_keccak` function will return the first 250bits of the keccak, while ethereum keccak will return the whole 256bits.
On its own, this isn't anything to worry about and hardly qualifies as a security note.
However, keeping in mind that StarkNet is an L2, capable of interacting with L1 contracts, any contracts that have L1 <-> L2 communication should consider this.
If you were to hash some data on L1 and pass it to L2, where L2 would also `keccak` this data to verify they are the same, you would get different outputs for the same data.
As Cairo continues to abstract the `felt252` type away, I expect that we will get a `u256` returning keccak function in the future.
But for right now, the difference between the traditional Etheruem keccak and StarkNet's keccak should be noted.






[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
