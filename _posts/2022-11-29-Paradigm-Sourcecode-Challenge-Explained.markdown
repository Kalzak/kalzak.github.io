---
layout: post
title:  "Paradigm CTF Sourcecode Challenge Explained"
date:   2022-11-29 11:57:16 +0800
---

Since participating in the 2022 Paradigm CTF event, I've been spending some of my free time going back and trying to solve some challenges that I didn't have time or knowledge to complete beforehand. I believe that simply looking at the solutions once the event is over robs you of the real learning, so instead I spent the weekend trying to solve the Sourcecode challenge without any hints. This post covers not just the solution, but also shows what I tried along the way so if you are just looking for the solution you can scroll to the end.

## The challenge

{% highlight solidity %}
// SPDX-License-Identifier: UNLICENSED

pragma solidity 0.8.16;

contract Deployer {
    constructor(bytes memory code) {
        assembly {
            return (
                add(code, 0x20),
                mload(code)
            )
        }
    }
}

contract Challenge {

    bool public solved = false;

    function safe(bytes memory code) private pure returns (bool) {
        uint i = 0;
        while (i < code.length) {
            uint8 op = uint8(code[i]);

            if (op >= 0x30 && op <= 0x48) {
                return false;
            }

            if (
                   op == 0x54 // SLOAD
                || op == 0x55 // SSTORE
                || op == 0xF0 // CREATE
                || op == 0xF1 // CALL
                || op == 0xF2 // CALLCODE
                || op == 0xF4 // DELEGATECALL
                || op == 0xF5 // CREATE2
                || op == 0xFA // STATICCALL
                || op == 0xFF // SELFDESTRUCT
            ) return false;
            
            if (op >= 0x60 && op < 0x80) i += (op - 0x60) + 1;
            
            i++;
        }
        
        return true;
    }

    function solve(bytes memory code) external {
        require(code.length > 0);
        require(safe(code), "deploy/code-unsafe");
        address target = address(new Deployer(code));
        (bool ok, bytes memory result) = target.staticcall("");

        require(ok, "1");
        require(keccak256(code) == target.codehash, "2");
        require(keccak256(result) == target.codehash, "3");

        solved = true;
    }
}

{% endhighlight %}

A first glance of the code indicates that we have to successfully call `solve`, by passing an argument `code` that is treated as the runtimecode for a deployed contract. The argument `code` needs to pass some checks in the function `safe` and the deployed contract must pass some conditions after being called. If all these requirements pass then the challenge is solved.

Starting with the group of three require statements:

1. Condition **1** can be satisfied by when a call to the `target` contract returns successfully.

2. Condition **2** can be satisfied by when the argument `code` is equal to the runtimecode of `target`.

3. Condition **3** can be satisfied by when a call to the `target` will return its own code.

It's possible to get the code of a contract with the opcodes `EXTCODECOPY` or `CODECOPY`, but the function `safe` prevents these opcodes from being in our runtimecode. I first considered there may be a way around the check as noticed the following line:

{% highlight solidity %}
while (i < code.length) {

	...

            
	if (op >= 0x60 && op < 0x80) i += (op - 0x60) + 1;

	i++;
}
{% endhighlight %}

The range check for opcodes within `[0x60, 0x80)` is designed to handle `PUSH` operations from `PUSH1 - PUSH32`. Any data inside the push would not be considered by the `safe` function. I thought there may have been a vulnerability where I could have `safe` treat data as part of a push operation where it actually will be used as runtimecode during deployment. Unfortunately I couldn't find a way to achieve this and figured the `safe` function was solid.

That left me with developing a contract that returns itself when called without the help of the `EXTCODECOPY` or `CODECOPY` opcodes. I felt stuck for a while, but after searching online I found that this type of challenge has a name: "quine". A quine is a program that returns its own source code. With this information I learned some techniques for writing quines and started learning the [Huff](https://huff.sh/) language to make writing this solution easier. The general solution for a quine is to have some data in the code that represents the code which can be duplicated, cut or otherwise manuipulated to eventually return the entire code. 

In the end I came up with this:

```
#define macro MAIN() = takes(0) returns(0) {

        push16 0x8060801b17606f595259526021601ff3

        // Combine both stack entries
        dup1
        0x80
        shl
        or

        // Store the push16 opcode
        0x6f
        msize mstore

        // mstore 32 bytes at slot 0
        msize mstore

        // Return 32 bytes
        0x21
        0x1f
        return
}
```

The runtimecode for this contract is: 

```
6f8060801b17606f595259526021601ff38060801b17606f595259526021601ff3
```

The data in the `PUSH16` is equal to all the code that follows, which you can see if I add some spacing:


```
6f      8060801b17606f595259526021601ff3  8060801b17606f595259526021601ff3
^       ^                                 ^
PUSH16  DATA REPRESENTING CODE            CODE
```

The approach that I took is to duplicate the data representing code twice, store the `PUSH16` into memory and then store the duplicated data now representing the entire code (aside from the missing `PUSH16` at the beginning) right next to the `PUSH16`. In memory we now have the entire runtimecode of the contract, and all we need to do is return it.

If you would like to follow the program execution step-by-step I have linked this contract code in evm.codes [here](https://www.evm.codes/playground?fork=merge&unit=Wei&codeType=Bytecode&code='6f~~'~80z801b17z6fyyz21z1ff3z60y5952%01yz~_).

This challenge was a lot of fun and was a great way to introduce me to the Huff language, big thanks to the creator [@rileyholterhus](https://mobile.twitter.com/rileyholterhus)!

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
