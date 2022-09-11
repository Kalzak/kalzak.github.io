---
layout:	post
title:	"StarkNet <-> Ethereum Communication Best Practices"
date:	2022-09-11 20:49:15 +0080
---

# Intro

The [Cairo Documentation](https://www.cairo-lang.org/docs/hello_starknet/l1l2.html) does a fantastic job of explaining the core concepts on how to communicate between *Ethereum* (L1) and *StarkNet* (L2), however there are many aspects that aren't covered which are important for a well designed and secure L1 <-> L2 communication implementation. This post serves to explore best practices and considerations when implementing L1 <-> L2 communication.

# The basics

To summarize the explanation in the [Cairo Documentation](https://www.cairo-lang.org/docs/hello_starknet/l1l2.html), the communication is handled differently whether it is L1 -> L2 or L2 -> L1. In L1 -> L2 communication the Ethereum contract must call `send_message()` to the StarkNet core contract, where a sequencer will receive this information and execute the message on a `@l1_handler` function in your desired contract on L2. In L2 -> L1 communication the StarkNet contract must call `send_message_to_l1()`, where eventually a state update is done on the Ethereum StarkNet core contract at which point the Ethereum contract can consume the message by calling `consumeMessageFromL2()`. An important distinction between these two flows is that for L1 -> L2 the StarkNet `@l1_handler` function is automatically triggered, whereas in L2 -> L1 the message must be consumed by calling `consumeMessageFromL2()`.

# Address range checks

StarkNet and Ethereum have different size addresses and it is important to validate these addresses before sending messages. Sending an invalid address to the other layer can lead to unexpected behavior such as reverted transactions. Addresses on StarkNet are within the range `[0,PRIME)` with `PRIME` being `2**251 + 17 * 2**192 + 1`. Addresses on Ethereum are 20 bytes which is `[0,2^20)`. An example of address validation for each layer is shown below:

{% highlight solidity %}
// Ethereum
function sendAddressToStarkNet(address addr) external {
	require(addr <= 2**251 + 17 * 2**192 + 1, "Invalid StarkNet Address");
	starknetCore.sendMessageToL2(l2ContractAddress, FUNCTION_SELECTOR, addr);
}
{% endhighlight %}

{% highlight cairo %}
func sendAddressToEthereum{
	syscall_ptr : felt*,
	pedersen_ptr : HashBuiltin*,
	range_check_ptr,
}(addr : felt):
	// 1048575 == 2**20 - 1
	assert_le_felt(addr, 1048575)

	let (message_payload : felt*) = alloc()
	assert message_payload[0] = addr

	send_message_to_l1(
		to_address=L1_CONTRACT_ADDRESS,
		payload_size=1,
		payload=message_payload,
	)

	return ()
end

{% endhighlight %}

# Message sanity checks on both layers

It is important to conduct sanity checks and input validation on both the message sender and message receiver side. These checks and validations are often done on the sender side, but sometimes are missed on the receiver side as the developers may wrongly assume that the message sender will always send correct data. This is a dangerous assumption to make as there may be unexpected edge cases or proxy implementation upgrades where the sender may send unexpected data. 

Let's say there is some protocol that sends one bit messages from L2 to L1 and the bit is controlled by user input. The L2 sending function has input validation to ensure that the user inputs the values `0` or `1`, but the function that consumes the message on L1 does not have this check. The developers decide to upgrade the L2 proxy implementation but the new input validation for the sender function has an edge case that allows for some values other than `0` and `1`. Without a check on the L1 function than consumes the message the protocol experiences unexpected behavior.

# L1 message consumption ordering

As mentioned in the "basics" section, L2 -> L1 messages do not automatically trigger a function call on L1. The function `consumeMessageFromL2()` with the correct L2 address and payload must be called. It can be dangerous for protocols to assume that messages from L2 -> L1 will be consumed in the correct order, especially if the time at which the message is consumed is solely up to the user. 

Let's say there is some protocol that has a storage array on both layers to track at what order users joined the protocol. A user joins on the L2 and then a message is sent to L1 where it is consumed and the L1 array is updated. The consistency between the L2 join order and L1 join order can be broken if a user waits to call `consumeMessageFromL2` until after another user who joined after them on L2 consumes the message on L1.

{% highlight cairo %}
@storage_var
func joined_addrs(index : felt) -> (addr : felt):
end

@storage_var
func joined_addrs_count() -> (count : felt):
end

func joinProtocol{
	syscall_ptr : felt*,
	pedersen_ptr : HashBuiltin*,
	range_check_ptr,
}(addr : felt):
	let (count) = joined_addrs_count.read()
	joined_addrs.write(count, addr)
	
	let (message_payload : felt*) = alloc()
	assert message_payload[0] = addr

	send_message_to_l1(
		to_address=L1_CONTRACT_ADDRESS,
		payload_size=1,
		payload=message_payload,
	)

	return ()
end
{% endhighlight %}

{% highlight solidity %}

uint256[] storage joined_addrs;
uint256 storage joined_addrs_count;

function consumeStarkNetProtocolJoin(uint256 l2_address) external {
	uint256 memory payload = new uint256[](1);
	payload[0] = l2_address;

	starknetCore.consumeMessageFromL2(l2ContractAddress, payload);

	joined_addrs[joined_addrs_count] = l2_address;
	joined_addrs_count += 1;
}

{% endhighlight %}

You have to consider that users may consume messages from L2 -> L1 late, out-of-order or maybe they may not decide to consume the message at all. Protocols should be designed to handle these cases to prevent unexpected behavior.

## L1 -> L2 message cancellation

A message can be cancelled if the L1 -> L2 message did not lead to a finalized transaction on StarkNet. There are two reasons that a transaction may not have finalized:

- The sequencer is not functioning or censoring transactions
- The L2 contract's `@l1_handler` function fails an assert during execution

If a L1 -> L2 message does not lead to a finalized transaction on L2 then the L1 will have successfully completed it's call and its state will have changed, but the L2 contract state will have remained the same. This causes an inconsistency between L1 and L2 that can have significant impacts. 

Let's say there is some protocol that acts as a simple bridge for ERC-20 tokens where you can `deposit` on L1 to lock your ERC-20 tokens in escrow and mint tokens on L2, and `withdraw` on L2 to burn your L2 tokens and release ERC-20 tokens from escrow on L1. If during a call to `deposit` the L2 contract's `@l1_handler` function fails then it will have no knowledge of any deposited funds beloning to the user so they cannot withdraw. Their funds will be locked in escrow in L1. If message cancellation was implemented it would be possible for the user to recover their tokens.

Message cancellation is a two step process. First the function `startL1ToL2MessageCancellation` must be called, then you must wait 5 days, then the function `cancelL1ToL2Message` can be called to complete message cancellation.

{% highlight solidity %}
function deposit(address token, uint256 amount, uint256 l2_address) external {
	require(l2_address <= 2**251 + 17 * 2**192 + 1, "Invalid StarkNet Address");

	IERC20(token).transferFrom(msg.sender, address(this), amount);

	uint256 memory payload = new uint256[](3);
	payload[0] = token
	payload[1] = amount
	payload[2] = l2_address;

	starknetCore.sendMessageToL2(
		l2ContractAddress,
		DEPOSIT_SELECTOR,

sit
	IERC20(token).transfer(msg.sender)
		payload,
	);
}

// Anybody can cancel anybody elses deposits here
// This is just to demonstrate message cancellation, don't use it's insecure
function startCancelDeposit(address token, uint256 amount, uint256 l2_address, uint256 nonce) external {
	uint256 memory payload = new uint256[](3);
	payload[0] = token
	payload[1] = amount
	payload[2] = l2_address;

	starknetCore.startLlToL2MessageCancellation(
		l2ContractAddress,
		DEPOSIT_SELECTOR,
		payload,
		nonce,
	);
}

// Anybody can cancel anybody elses deposits here
// This is just to demonstrate message cancellation, don't use it's insecure
function cancelDeposit(address token, uint256 amount, uint256 l2_address, uint256 nonce) external {
	uint256 memory payload = new uint256[](3);
	payload[0] = token
	payload[1] = amount
	payload[2] = l2_address;

	starknetCore.cancelL1ToL2Message(
		l2ContractAddress,
		DEPOSIT_SELECTOR,
		payload,
		nonce,
	);

	IERC20(token).transfer(msg.sender)
}
{% endhighlight %}

{% highlight cairo %}
// The deposit function will fail, message cancellation needs to be used
@l1_handler
func deposit{
	syscall_ptr : felt*,
	pedersen_ptr : HashBuiltin*,
	range_check_ptr,
}(token : felt, amount : felt, target_address : felt):
	with_attr error_message("Time to use message cancellation"):
		assert 0 = 1
	end
end
{% endhighlight %}

## Footnote

If you think there is anything that I have missed, please let me know at [@kalzakdev](https://twitter.com/kalzakdev) and I'll be sure to add it to this post and credit you.
