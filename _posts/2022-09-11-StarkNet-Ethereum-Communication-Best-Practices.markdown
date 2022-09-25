---
layout:	post
title:	"StarkNet <-> Ethereum Communication Best Practices"
date:	2022-09-24 11:49:15 +0080
---

## Intro

The [Cairo Documentation](https://www.cairo-lang.org/docs/hello_starknet/l1l2.html) does a fantastic job of explaining the core concepts on how to communicate between *Ethereum* (L1) and *StarkNet* (L2), however there are many aspects that aren't covered which are important for a well designed and secure L1 <-> L2 communication implementation. This post serves to explore best practices and considerations when implementing L1 <-> L2 communication.

---

## The basics

To summarize the explanation in the [Cairo Documentation](https://www.cairo-lang.org/docs/hello_starknet/l1l2.html), the communication is handled differently whether it is **L1 -> L2** or **L2 -> L1**. 

In **L1 -> L2** communication the Ethereum contract must call `send_message()` on the StarkNet core contract, where a sequencer will receive this information and execute the message on the `@l1_handler` function in the desired contract on L2. 

In **L2 -> L1** communication the StarkNet contract must call `send_message_to_l1()`, where eventually a state update is done on the Ethereum StarkNet core contract at which point the intended recipient Ethereum contract can consume the message by calling `consumeMessageFromL2()`. 

An important distinction between these two flows is that for **L1 -> L2** the StarkNet `@l1_handler` function is automatically triggered, whereas in **L2 -> L1** the message must be consumed by calling `consumeMessageFromL2()`.

You may also see the value `PRIME` referred to throughout this post. `PRIME` is equal to `2**251 + 17 * 2**192 + 1` and is the largest possible size of a StarkNet `felt`.

---

## Address range checks

StarkNet and Ethereum have different address sizes and it is important to validate these addresses before sending messages. Sending an invalid address to the other layer can lead to unexpected behavior such as reverted transactions. 

Addresses on StarkNet are within the range `[0,PRIME)`, while addresses on Ethereum are 20 bytes which is the range `[0,2^20)`. 

An example of address validation for each layer is shown below:

{% highlight solidity %}
// Verify for valid StarkNet address in Solidity

function sendAddressToStarkNet(address addr) external {
	require(addr < 2**251 + 17 * 2**192 + 1, "Invalid StarkNet Address");
	starknetCore.sendMessageToL2(l2ContractAddress, FUNCTION_SELECTOR, addr);
}
{% endhighlight %}

{% highlight plaintext %}
# Verify for valid Ethereum address in Cairo

func sendAddressToEthereum{
	syscall_ptr : felt*,
	pedersen_ptr : HashBuiltin*,
	range_check_ptr,
}(addr : felt):
	# 1048575 == 2**20 - 1
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

---

## Ethereum word vs StarkNet felt

The Ethereum virtual machine has a word size of 32 bytes (256 bits). On StarkNet the type `felt` is used, which is a 252 bit value within the range `0 <= x < PRIME`. 

If you want to send a large value (let's say `type(uint256).max`) from **L1 -> L2** it would be larger than the `felt` type. To solve this there is the type `Uint256` from the StarkNet common library which breaks a `uint256` into the `high` 128 bits and the `low` 128 bits. Below is a function that splits an Ethereum `uint256` into a `low` and `high` to easily be sent to L2 (source: [Aave Starknet Bridge](https://github.com/aave-starknet-project/aave-starknet-bridge/blob/main/contracts/l1/libraries/helpers/Cairo.sol)).

{% highlight solidity %}

function toSplitUint(uint256 value)
	internal
	pure
	returns (uint256, uint256)
{
	uint256 low = value & ((1 << 128) - 1);
	uint256 high = value >> 128;
	return (low, high);
}

{% endhighlight %}

If you want to send some `uint256` value from **L1 -> L2** and want to verify that it will fit within a `felt`, there can be a misconception that the value should be less that `2**252` (because a `felt` is 252 bits) however this is incorrect because `PRIME < 2**252` so the value could still be larger than the max size of a `felt`.

---

## Message sanity checks on both layers

It is important to conduct sanity checks and input validation on both the message sender and message receiver side. These checks and validations are often done on the sender side, but sometimes are missed on the receiver side as the developers may wrongly assume that the message sender will always send correct data. This is a dangerous assumption to make as there may be unexpected edge cases or proxy implementation upgrades where the sender may send unexpected data. 

Let's say there is some protocol where a user can call a function on L2 to send either a `0` or `1` from **L2 -> L1**. The L2 sending function has input validation to ensure that the user inputs either `0` or `1` and will fail an assert otherwise. The function on L1 that consumes this message does not have input validation, as it was assumed that the data from the consumed message will always be valid. The developers decide to upgrade the L2 implementation however there is an edge case in the L2 sending function where some value other than `0` and `1` can be sent to L1. Without a sanity check on L1, invalid data will be accepted by the protocol.

---

## L1 message consumption ordering

As mentioned in the "basics" section, **L2 -> L1** messages do not automatically trigger a function call on L1. The function `consumeMessageFromL2()` with the correct L2 address and payload must be called. It can be dangerous for protocols to assume that messages from **L2 -> L1** will be consumed in the correct order, especially if the time at which the message is consumed is controlled by the user.

Let's say there is some protocol that has a "synced" storage array on both layers to track the order that users called the `joinProtocol` function on StarkNet. A user joins the protocol on L2, the L2 storage array is updated and a message is sent to L1 where the message will be consumed and the L1 array will also be updated. The ordering of the L1 and L2 storage arrays can be broken if a user joins on L2 and then waits for another user to join on L2 and consume on L1. After the second user has joined the first user then calls the consume on L1 function causing the arrays to be out-of-order. Example code is shown below.

{% highlight plaintext %}
# Cairo code

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
// Solidity code

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

You have to consider that users may consume messages from **L2 -> L1** late, out-of-order or maybe they may not decide to consume the message at all. Protocols should be designed to handle these cases to prevent unexpected behavior.

---

## L1 -> L2 message cancellation

A message can be cancelled if the **L1 -> L2** message did not lead to a finalized transaction on StarkNet. There are two reasons that a transaction may not have finalized:

- The StarkNet sequencer is not functioning or censoring transactions
- The L2 contract's `@l1_handler` function fails an assert during execution

If a **L1 -> L2** message does not lead to a finalized transaction on L2 then the L1 will have successfully completed it's call and its state will have changed, but the L2 contract state will have remained the same. This causes an inconsistency between L1 and L2 that can have significant impacts. 

Let's say there is some protocol that acts as a simple bridge for ERC-20 tokens where you can:

- `deposit` on L1 to lock your ERC-20 tokens in escrow and mint tokens on L2
- `withdraw` on L2 to burn your L2 tokens and release ERC-20 tokens from escrow on L1. 

If during a call to `deposit` the L2 contract's `@l1_handler` function fails then the L2 contract will have no knowledge of any deposited funds belonging to the user so they cannot withdraw. Their funds will be locked in escrow in L1 permanently. If message cancellation was implemented it would be possible for the user to recover their tokens.

Message cancellation is a two step process. First the function `startL1ToL2MessageCancellation` must be called, then you must wait 5 days, then the function `cancelL1ToL2Message` can be called to complete message cancellation.

The following is code from the scenario described above.

{% highlight solidity %}

// The deposit function. Transfers tokens from user and sends message to L2
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
		payload,
	);
}

// Function to start a message cancellation
// No permission checks here, this is just for demonstration purposes
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

// Function to complete the message cancellation
// No permission checks here, this is just for demonstration purposes
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

{% highlight plaintext %}
# The deposit function will fail, message cancellation needs to be used
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

---

## End

Hopefully this list of cross-chain communication best practices will help to improve your StarkNet protocol. If you think there is anything that I have missed, please let me know at [@kalzakdev](https://twitter.com/kalzakdev) and I'll be sure to add it to this post and credit you.
