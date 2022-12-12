---
layout: post
title:  "The Smallest Hack of 2022"
date:   2022-12-06 11:57:16 +0800
---

Most of the time when you hear about a DeFi hack the attacker gets away with hundreds of thousands, if not millions of dollars. Today I would like to show you my investigation of the smallest hack I've come across where the attacker managed to steal just `1.12` BNB. 

I came across the attack through [this](https://twitter.com/CertiKAlert/status/1596646156951384064) twitter post from [@CertiKAlert](https://twitter.com/CertiKAlert) while looking for exploits I can study. I only grab the transaction hash and then do some research on my own, and then compare my analysis with their article as a way to improve my security knowledge. For anybody interested in doing the same, I have linked the transaction hash below so you can have a look for yourself before getting to the explanation.

[0xe4c2089468fc5eb115d83e8a89382e53ad3df564f8c1abd02e1d74b40ab4c8de](https://bscscan.com/tx/0xe4c2089468fc5eb115d83e8a89382e53ad3df564f8c1abd02e1d74b40ab4c8de)

## What happened?

On May 27, 2022 an attacker managed to steal WBNB out of a Pancake pair between LegendCoin (LDC) and WBNB using funds from a flashloan. The pair didn't have much value on it, having `~0.66` BNB and `~20738` LDC. They borrowed WBNB using a flashloan and swapped the WBNB for a large amount of LDC. They would then send that LDC _back_ to the same pair again and `skim` it to a _different_ LDC pair, this time being a LDC and BSCUSD pair. Once this was done they managed to get fantastic pricing when exchanging LDC for WBNB in the initial pair. How did this work?

## How did this happen?

The attacker managed to manipulate the pricing of the LDC-WBNB pair by sending tokens to the same pair and then skimming. In the LDC ERC20 contract there is some non-standard code in the `transfer` logic which allows this to happen:

The attacker managed to manipulate the pricing of the LDC-WBNB pair by sending `~19565` LDC to the pair and then skimming that same amount of LDC, sending it to the LDC-BSCUSD pair. However after the skim, the LDC-WBNB pair lost `~20738` LDC instead of the expected `~19565`. This is how the price was manipulated and how the attacker wale able to extract WBNB from the pair. 

If we look at the LDC token `_transfer` logic we can see where our problem lies:

{% highlight solidity %}
function _transfer(address sender, address recipient, uint256 amount) internal {
    _transfer_amount(sender,recipient,amount);
    if(sender==swapAddress||recipient==swapAddress){
        fomoTime=uint40(block.timestamp)+fomoDiff;
        uint256 feeAmount;
        address feeAddress;
        if (sender==swapAddress){
                feeAddress=recipient;
                feeAmount=amount*buyFeeRate/feeRateMax;
        }else {
                feeAddress=sender;
                feeAmount=amount*sellFeeRate/feeRateMax;
        }
        if(whiteList[feeAddress]==true){return;}
        if(feeAmount==0){
            return;
        }

        _transfer_burn(feeAddress,feeAmount*3/10);
        _transfer_pool(feeAddress,feeAmount*3/10);
        _transfer_recommend(feeAddress,feeAmount*4/10);
    }

    if(sender==swapAddress){
        emit AddTransferEvent(recipient);
    }
}

function _transfer_amount(address from,address to,uint256 amount)private{
    balanceOf[from]=balanceOf[from]-amount;
    balanceOf[to]=balanceOf[to]+amount;
    emit Transfer(from, to, amount);
}

function _transfer_burn(address feeAddress,uint256 amount)private{
    _transfer_amount(feeAddress,address(0),amount);
}

function _transfer_pool(address feeAddress,uint256 sendAmount)private{
    uint256 fomoAmount = sendAmount*35/100;
    uint256 dividendsAmount = sendAmount*60/100;
    uint256 projectAmount = sendAmount*5/100;

    if (projectAddress==address(0)){
        _transfer_amount(feeAddress,address(this),dividendsAmount+projectAmount);
    }else{
        _transfer_amount(feeAddress,projectAddress,dividendsAmount+projectAmount);
    }

    fomoPool=fomoPool+fomoAmount;
    _transfer_amount(feeAddress,address(this),fomoAmount);

    emit AddDividendsEvent(dividendsAmount);
}

function _transfer_recommend(address feeAddress,uint256 amount)private{
    if (projectAddress==address(0)){
        _transfer_amount(feeAddress,address(this),amount);
    }else{
        _transfer_amount(feeAddress,projectAddress,amount);
    }
    emit AddRecommendEvent(feeAddress,amount);
}
{% endhighlight %}

To summarize the code above, the LDC token charges a fee if the sender or recipient of a transfer is the `swapAddress`. The address which is not the `swapAddress` will incur an additional 6% fee on-top of the transferred amount. An interesting thing to note here is that there can only be one `swapAddress`, which is the address to the LDC-BSCUSD pair. This pair was likely created by the developers and was intended to be the only pair to swap tokens with.

At some point there was a savvy DeFi user that had figured this out and decided to circumvent the fee system by creating a LDC-WBNB pair, which of course has a different address to the LDC-BSCUSD address that incurs fees. That means we have two pair for the LDC token. One is the "official" pair that the developers intended users to swap with and charge fees on, and the other is an "unofficial" swap that doesn't incur fees.

The idea behind the attack is to use the `skim` function of the LDC-WBNB pair to transfer tokens to the fee-incurring LDC-BSCUSD pair. When this transfer happens an extra 6% of the transfer amount will be deducted from the sending LDC-WBNB pair which will lead to the price curve strongly favoring LDC. 

To briefly explain how `skim` works, pairs have the storage variables `reserve0` and `reserve1` that track the balance of `token0` and `token1` in the pair. This can become out of sync with the actual balance amount, in which case it is possible to call `skim` to transfer any excess token amounts above the reserve amount to any specified address. The `skim` function is callable by anyone, and anybody can trigger it by sending tokens to the pair and then calling `skim`.

The original balance of the LDC-WBNB pair was `20738.98156752`. The attacker swapped `11.0338586181` WBNB for `19565.07225487` LDC, leaving `1173.90931265` LDC in the LDC-WBNB pair. When the attacker sends the `19565.07225487` LDC back to the pair, they then call `skim` on the pair and send the tokens to the fee-incurring LDC-BSCUSD pair. This causes an extra 6% fee for the transfer leading to a balance of `0.00497740` LDC in the LDC-WBNB pair. The price has now been manipulated.

Tle manipulated price is taken advantage of by swapping all the attackers LDC for as much WBNB as possible. Now the pricing is manipulated in the other way where a little WBNB is equal to a lot of LDC. The LDC token prices may be wrong when using the LDC-WBNB pair, but on the LDC-BSCUSD pair the pricing is still correct and each LDC token is worth `~0.009` USD on that pair. It's actually more profitable for the attacker to swap a little WBNB back for a lot of LDC, which is then swapped on the LDC-BSCUSD pair and then from BSCUSD to WBNB. The attacker pays off the flashloan amount and walks away with a small sum of `1.12` WBNB. 

## What to learn from this

Developers need to be aware of the `skim` functionality that exists on most liqudity pairs. It can be called by anybody and can send tokens to any address. Any protocols or tokens that have any special functionality on token transfers involving liquidity pair addresses can have that special functionality triggered outside of a normal token swap. In this case the special functionality was a fee mechanism built into the token itself.

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
