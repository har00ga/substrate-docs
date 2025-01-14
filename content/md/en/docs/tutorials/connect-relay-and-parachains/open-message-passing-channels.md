---
title: Open message passing channels
description: Demonstrates how to use horizontal relay-routed message passing (HRMP) for communication between parachains.
keywords:
  - Polkadot
  - cross-consensus messaging
  - HRMP
  - XCM
  - parachain
---

In the Polkadot ecosystem, chains can communicate with each other by passing messages over secure channels.
There are three main communication channels:

- Upward message passing (UMP) to enable a parachain to pass messages to its relay chain. 
- Downward message passing (DMP) to enable the relay chain to pass messages to a parachain. 
- Cross-consensus messaging passing (XCMP) to enable parachains to send messages to each other.

Horizontal relay-routed message passing (HRMP) is an interim version of cross-consensus message passing (XCMP). 
This interim solution—also sometimes referred to as XCMP-Lite—provides the same functionality as XCMP but stores all of the messages passed between chains in the relay chain storage.

Although HRMP is intended to be phased out when XCMP is fully implemented, this tutorial uses HRMP to illustrate how you can open message passing channels to enable parachains to communicate with each other.

## About the XCM message format

The messages passed through the Polkadot communication channels use the [XCM message format](https://github.com/paritytech/xcm-format) to express what should be done by the message receiver.
The XCM message format is designed to support communication between chains, but also messages from smart contracts and pallets, over bridges, and through other transport protocols.

## Opening HRMP channels

It is important to note that HRMP channels are unidirectional.
If you want parachain 1000 to communicate with parachain 1001, you must first make a request to open an HRMP channel from parachain 1000 to parachain 1001.
Parachain 1001 must then accept the request before parachain 1000 can pass messages to it.
Because the channel is unidirectional, however, parachain 1000 can't receive messages from parachain 1001 over the channel.

For parachain 1000 to receive messages from parachain 1001, you must open another channel from parachain 1001 to parachain 1000. 
After parachain 1000 confirms that it will accept messages from parachain 1001, the chains can exchange messages at the next session change.

## Before you begin

In this tutorial, you'll open HRMP channels that enable a parachain with the unique identifier 1000 and a parachain with the unique identifier 1001 to exchange XCM messages.
Before you begin, verify the following:

- You have set up a [parachain test network](/test/simulate-parachains) using Zombienet or a local relay chain using the `rococo-local` chain specification.
  
- You have set up two local or virtual parachains for testing purposes.
  
  For the purposes of this tutorial, Parachain A has the unique identifier 1000 and Parachain B has the unique identifier 1001.

- You have the Sudo pallet available for both local parachains to use.
  
  For this tutorial in a test environment, you can add the Sudo pallet to each collator node.
  The pallet is not included by default if you use the `substrate-parachain-template` to build your node.
  To add the Sudo pallet, update the `runtime/src/lib.rs` file and the `node/src/chain_spec.rs` files.
  For an example of a chain specification with the Sudo pallet, see [parachain-template-1001.rs](/assets/tutorials/relay-chain-specs/parachain-template-1001.rs).
  In a production environment, you would use governance proposals and voting instead of the `sudo` pallet for privileged transactions.

## Add the sovereign accounts

Before the parachain can exchange messages with another parachain, it must have an account on the relay chain that has funds available to pay XCM transaction fees.

1. Open the [Polkadot/Substrate Portal](https://polkadot.js.org/apps) and connect to a relay chain endpoint.

2. Calculate the parachain [sovereign account](https://substrate.stackexchange.com/questions/1200/how-to-calculate-sovereignaccount-for-parachain) on the relay chain.
      
   Parachain A (1000) address: 5Ec4AhPZk8STuex8Wsi9TwDtJQxKqzPJRCH7348Xtcs9vZLJ

   Parachain B (1001) address: 5Ec4AhPZwkVeRmswLWBsf7rxQ3cjzMKRWuVvffJ6Uuu89s1P
  
   Note that if the parachain identifier registered for a parachain changes, the sovereign account and address will also change. 
   You should also note that the account address used for a parachain on the relay chain is different from the address used for the parachain on another parachain. 

3. Click **Accounts** and select **Address Book**.

4. Click **Add contact**.

5. Add the address and a name for parachain A (1000), then click **Save**.

6. Click **Accounts** and transfer some tokens from Alice to the parachain A (1000) account.
   
   Repeat step 3 through step 6 for parachain B (1001).

## Prepare the open channel encoded call

To set up communication between the parachains, you must first send a request to open a message passing channel.
The request must take the form of an encoded call with parameters that specify the parachain to receive the request, the message capacity for the channel, and maximum message size. 
You'll need to include the encoded version of this information in the message you create to make the request, but you can generate the encoded call without submitting the transaction to prepare the request.

To prepare the encoded call to open a channel:

1. Open the [Polkadot/Substrate Portal](https://polkadot.js.org/apps) and connect to a relay chain validator node such as the endpoint for the `alice` node.

2. Check the channel configuration limits for the relay chain, if needed.
   
   To check the configuration settings for the current relay chain:

   - Click **Developer** and select **Chain State**.
   - Select **configuration**, then select **activeConfig()** and click **+**.
   - Check the parameter values for `hrmpChannelMaxCapacity` and `hrmlChannelMaxMessageSize`.
   
   For example:
     
      ```text
      hrmpChannelMaxCapacity: 8
      hrmlChannelMaxMessageSize: 1,048,576
      ```

3. Click **Developer** and select **Extrinsics**.

4. Select **hrmp**, then select **hrmpInitOpenChannel(recipient, proposedMaxCapacity, proposedMaxMessageSize)** to initialize the request to open a new channel.
   
   For the transaction parameters, specify the following to prepare the call data:
   
   - recipient: Type the identifier for the parachain you want to open the channel with (1001).
   - proposedMaxCapacity: Type the maximum number of messages that can be in the channel at once (8).
   - proposedMaxMessageSize: Specify the maximum size of the messages (1048576).
    
   Note that the values you set for **proposedMaxCapacity** and **proposedMaxMessageSize** shouldn't exceeded the values defined for the `hrmpChannelMaxCapacity` and `hrmpChannelMaxMessageSize` parameters for the relay chain.

5. Copy the **encoded call data**.
   
   ![Copy the encoded call data](/media/images/docs/tutorials/parachains/hrmp-encoded-call.png)
   
   You'll need this information to craft the XCM message.
   The following is an example of encoded call data in Rococo:
   0x3c00e90300000800000000001000
   
## Configure the open channel request

Now that you have the encoded call, you can configure the request to open a channel from parachain A to parachain B through the relay chain.

1. Connect to the endpoint for parachain A (1000) using the [Polkadot/Substrate Portal](https://polkadot.js.org/apps).

2. Click **Developer** and select **Extrinsics**.

3. Select **sudo**, then select **sudo(call)** to use the Sudo pallet to execute privileged transactions.

3. Select **polkadotXcm**, then select **send(dest, message)** to notify the relay chain that you want to open a channel with parachain B (1001).
   
   ![Send message using the Sudo pallet](/media/images/docs/tutorials/parachains/hrmp-sudo-call.png)

4. Specify the destination parameters to indicate the relative location for the message to be delivered.
   
   The destination parameters specify where the XCM should be executed. 
   In this example, the parent of parachain 1000 is the relay chain and in the context of the parent the interior setting of Here means that the relay chain is going to execute the XCM. 
   For more information about specifying relative locations for XCM, see [Universal Consensus Location Identifiers](https://github.com/paritytech/xcm-format#7-universal-consensus-location-identifiers).

   ![Destination parameters](/media/images/docs/tutorials/parachains/hrmp-destination.png)
   
5. Specify the XCM version, then click **Add item** to construct the message to be executed.
   
   At a minimum, you need to add the following set of instructions for this message:
   
   - [WithdrawAsset](https://github.com/paritytech/xcm-format#withdrawasset) to move the specified on-chain assets into the virtual [holding register](https://polkadot.network/blog/xcm-the-cross-consensus-message-format/#-the-holding-register).
  
   - [BuyExecution](https://github.com/paritytech/xcm-format#buyexecution) to pay for the execution of the current message using the assets that were deposited in the virtual holding register using the WithdrawAsset instruction.
     For more information about paying fees, see [Fee payment in XCM](https://polkadot.network/blog/xcm-the-cross-consensus-message-format/#-fee-payment-in-xcm).

   - [Transact](https://github.com/paritytech/xcm-format#transact) to specify the encoded call that you prepared on the relay chain.  

   Note that each instruction requires you to specify the location parameters to identify the message recipient that will be executing the XCM instruction.
   Be sure that you construct the relative paths for each instruction from the point of view of the receiving system. 
   For more information about specifying locations, see [Concrete identifiers](https://github.com/paritytech/xcm-format#concrete-identifiers).

   In most cases, you also want to include the following instructions:
   
   - [RefundSurplus](https://github.com/paritytech/xcm-format#refundsurplus) to move any overestimate of fees previously paid using the BuyExecution instruction into a second virtual register called the refunded weight register.
  
   - [DepositAsset](https://github.com/paritytech/xcm-format#depositasset) to subtract assets from the refunded weight register and deposit on-chain equivalent assets under the ownership of the beneficiary. 
   
     Typically, the beneficiary for the DepositAsset instruction is the sovereign account of the message sender.
     In this case, you can specify parachain A (1000) as `parents: 0`, `interior: X1`, `Parachain: 1000` so that any surplus assets are returned to the account and can be used to deliver other XCM messages or to open additional HRMP channels.
     For more information about the RefundSurplus and DepositAsset instructions, see [Weight]9https://polkadot.network/blog/xcm-part-three-execution-and-error-management/#-weight).

     This set of instructions:
     - Withdraw assets from the parachain A sovereign account to the virtual holding register.
     - Uses the assets in the holding register to pay for the execution time the XCM instructions require.
     - Executes the initialization request for an open channel on the relay chain. 
     - Refunds any left over assets and deposits the refunded assets into the account owned by the specified beneficiary.

     For more information and answers to specific technical questions, try the following tags on [Substrate and Polkadot Stack Exchange](https://substrate.stackexchange.com/):
     
     - xcm
     - hrmp 
     - weight 
     - cumulus

    For an example that illustrates all of the settings for this set of instructions, see the sample [xcm-instructions](/assets/tutorials/relay-chain-specs/xcm-instructions.txt) file.

1. Click **Submit Transaction**.

## Verify the request

After you submit the transaction, you should verify that the message was received and executed on the relay chain.

To verify the request:

1. Open the [Polkadot/Substrate Portal](https://polkadot.js.org/apps) in two separate browser tabs or windows with one instance connecting to the relay chain and one instance connecting to the parachain endpoint.
   
2. In the browser instance connecting to the parachain, click **Network** then select **Explorer**.

3. Check the list of recent events and verify that there is a **sudo.Sudid** event and a **polkadotXcm.Sent** event.
   
   ![Events for XCM instructions on the parachain](/media/images/docs/tutorials/parachains/hrmp-parachain-events.png)

   You can expand the **polkadotXcm.Sent** event to see details about the instructions included in the XCM message.

4. In the browser instance connecting to the relay chain, click **Network** then select **Explorer**.

3. Check the list of recent events and verify that the **ump.UpwardMessagesReceived**, **hrmp.OpenChannelRequested**, and **ump.ExecutedUpward** events are displayed.
   
   ![Events for XCM instructions on the relay chain](/media/images/docs/tutorials/parachains/hrmp-relay-chain-request-complete.png)
      
   You can expand these events to see details about the instructions that were executed.

4. Check the sovereign account balance for parachain A (1000).
   
   ![Account balance after message execution](/media/images/docs/tutorials/parachains/hrmp-after-msg-execution-balance.png)

## Prepare the acceptance encoded call

After you have successfully created the open channel request, parachain B (1001) must accept the request before you can send any messages using the channel you've requested.
The steps are similar to the steps for initiating an open channel request, but you'll use a different encoded call and you'll send the message from parachain B instead of parachain A.

The first step is to prepare the encoded call on the relay chain.

To prepare the encoded call:

1. Open the [Polkadot/Substrate Portal](https://polkadot.js.org/apps) and connect to the relay chain.

2. Click **Developer** and select **Extrinsics**.

3. Select **hrmp**, then select **hrmpAcceptOpenChannel(sender)** and specify the **sender**, in this case, parachain A (1000).

4. Copy the **encoded call data**.
   
   ![Copy the encoded call data](/media/images/docs/tutorials/parachains/hrmp-relay-chain-accept-request.png)
   
   You'll need this information to craft the XCM message.
   The following is an example of encoded call data in Rococo:
   0x3c01e8030000

## Accept the open channel request

Now that you have the encoded call, you can construct the XCM message to accept the open channel requested by parachain A.

1. Connect to the endpoint for parachain B (1001) using the [Polkadot/Substrate Portal](https://polkadot.js.org/apps).

2. Click **Developer** and select **Extrinsics**.

3. Select **sudo**, then select **sudo(call)** to use the Sudo pallet to execute privileged transactions.

4. Select **polkadotXcm**, then select **send(dest, message)** to notify the relay chain that you want to open a channel with parachain B (1001).
   
5. Specify the destination parameters to indicate the relative location for the message to be delivered.
      
6. Specify the XCM version, then click **Add item** to construct the message to be executed.
   
   You can use a similar set of instructions for this message:
   
   - [WithdrawAsset](https://github.com/paritytech/xcm-format#withdrawasset) to move the specified on-chain assets into the virtual holding register.
  
   - [BuyExecution](https://github.com/paritytech/xcm-format#buyexecution) to pay for the execution of the current message using the assets that were deposited in the virtual holding register.

   - [Transact](https://github.com/paritytech/xcm-format#transact) to specify the encoded call—for example, 0x3c01e8030000—that you prepared on the relay chain.  

   - [RefundSurplus](https://github.com/paritytech/xcm-format#refundsurplus) to move any overestimate of fee into the refunded weight register.
  
   - [DepositAsset](https://github.com/paritytech/xcm-format#depositasset) to subtract assets from the refunded weight register and deposit on-chain equivalent assets under the ownership of the beneficiary. 

7. Click **Submit Transaction**.
   
   After you submit the transaction, you must wait for the next epoch to see the change reflected in the chain state.

## Verify the open channel

After you submit the transaction, you can verify that the message was received and executed on the relay chain.

To verify the request:

1. Open the [Polkadot/Substrate Portal](https://polkadot.js.org/apps) in two separate browser tabs or windows with one instance connecting to the relay chain and one instance connecting to the parachain endpoint.
   
2. In the browser instance connecting to the parachain, click **Network** then select **Explorer**.

3. Check the list of recent events and verify that there is a **sudo.Sudid** event and a **polkadotXcm.Sent** event.
   
4. In the browser instance connecting to the relay chain, click **Network** then select **Explorer**.

5. Check the list of recent events and verify that the **hrmp.OpenChannelAccepted** and **ump.ExecutedUpward** events are displayed.
   
   ![Events for XCM instructions on the relay chain](/media/images/docs/tutorials/parachains/hrmp-relay-chain-request-accepted.png)

   After the start of the next epoch, you can also query the chain state for **hrmpChannels** to verify that you've opened a channel from parachain A (1000) to parachain B (1001).
   
   ![Query HRMP channels](/media/images/docs/tutorials/parachains/hrmp-verify-channel.png)

## Open a second channel

If you only want to send XCM instructions to parachain B (1001), your work is done.

However, if you want to enable two-way communication-where both chains can receive and execute instructions sent by the other chain, you need to follow the same steps for parachain B that you just completed for parachain A. After parachain A accepts the open channel requests, both chains can send, receive, and execute XCM instructions.

Repeat all of the steps for preparing the encoded calls, sending the open channel request, and accepting the request to enable communication from parachain B (1001) to parachain A (1000).
After you open the channel from parachain B to parachain A, the two parachains can send messages to each other routed through the relay chain.

<!--
To illustrate the interaction between the two chains, in the following example, parachain B sends XCM instructions to deposit assets into an account on parachain A.

1. Connect to the endpoint for parachain B (1001) using the [Polkadot/Substrate Portal](https://polkadot.js.org/apps).

2. Click **Developer** and select **Extrinsics**.

3. Select **sudo**, then select **sudo(call)** to use the Sudo pallet to execute privileged transactions.

4. Select **polkadotXcm**, then select **send(dest, message)**.
   
5. Specify the destination parameters to indicate the relative location for the message to be delivered.
      
6. Specify the XCM version, then click **Add item** to construct the message to be executed.

-->

## Where to go next

- [XCM Part I: The Cross-Consensus Message Format](https://polkadot.network/blog/xcm-the-cross-consensus-message-format)
- [XCM Part II: Versioning and Compatibility](https://polkadot.network/blog/xcm-part-two-versioning-and-compatibility)
- [XCM Part III: Execution and Error Management](https://polkadot.network/blog/xcm-part-three-execution-and-error-management)
