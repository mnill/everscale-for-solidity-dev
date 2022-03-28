---
layout: default
title: Introduction. Blockchain. Everything is a contract. Types of messages. Gas.
nav_order: 2
description: "Just the Docs is a responsive Jekyll theme with built-in search that is easily customizable and hosted on GitHub Pages."
permalink: /introdutction
---

## Introduction. Blockchain. Everything is a contract. Types of messages. Gas.

<p>Everscale (hereafter ES) was created on the basis of the whitepaper and initial code of the TON (Telegram Open Network) project from Nikolai Durov. Essentially, the blockchain repeats the behavior described in this whitepaper: https://ton.org/tblkch.pdf , however, not everything works as is presented in the whitepaper. </p>

<br/>

### The Everscale Philosophy

<p>In order to introduce the tutorial, we would like to speak about the Everscale blockchain and what makes it so promising.</p>

<p>When Nikolai Durov was creating TON, he was faced with the task of designing a blockchain platform that could accommodate millions of users while having low, stable transaction fees.</p>

<br/>

<p>Durov was able to achieve this, and this is how: </p>

1. **Infinite Sharding.** <p>On ES shards are dynamically added as the load increases and then merged back. This is possible because all contracts on the chain communicate with each other asynchronously, and therefore we can split one shard into two shards without any problems occurring. (Shards are just divided in half according to the ranges of contract addresses.)</p>

2. **Rejection of radical decentralization.** <p>The ES blockchain was not built to allow just anyone to become a validator. Validation is a critical process, requires professional equipment and access to an appropriate server. The total number of validators will at most be in the thousands, not in the tens of thousands. And validator machines have high server and channel requirements (the current requirements are 48 CPUs, 128 RAM and 1TB SSD) and a 1GB channel (the network is used extensively). This allows for the blockchain to support a very quick block release speed and often rotate validators in the shards.</p>

3. **Paid storage**. <p>This is a completely brilliant and daring decision that no other blockchain has implemented.</p> <p>When you write something on a classic blockchain (like Ethereum), you put information on the chain forever.</p> <p>So, if you buy a meme coin, data reflecting that purchase will be on the chain until the chain dies. You only pay on creating and accessing that information. However, validators must store that information forever. </p><p>This give rise to a curious economy - blockchains are forced to artificially limit their rate of recording so that the size of the blockchain state does not grow faster than the rate at which data storage becomes cheaper (In fact, they are even forced to try and prevent the blockchain state from growing faster than the rate at which RAM becomes cheaper). As a result, users are forced to compete with each other in auctions for the right to write data to the blockchain, and transaction fees are increasing all the time. </p> <p>On ES, this problem is solved very simply - each contract is required to pay rent for the validators to store its state, and when the contract runs out of money, it gets deleted. Yes, this is radical, by doing this, users do not need to compete with each other for the right to write indelibly on the blockchain. On ES, each user determines how long they want their data to remain on the blockchain and has the option to extend that time frame. This makes the tokenomics of the chain completely unique, adding flexibility to the data inputed to the chain.</p>

<p>Essentially, ES aims to be a decentralized replacement for AWS. Just as you can host your application on AWS, you can host it on ES. Hosting it on ES will not be much more expensive (if it is rarely used, it will be cheaper), but it will have maximum fault tolerance.</p>

<br/>

### Blockchain

<p>This tutorial is not going to delve into the details of how exactly the ES blockchain works, because you don’t need all of that to understand how to write smart contracts for ES. In the course of the tutorial, we will analyze the guarantees that the blockchain gives and everything you need to know to write smart contracts.</p>

<p>What do you need to know about the blockchain? ES is a multithreaded blockchain. There are different work chains (like global shards, they differ in parameters—there are 2 of them now), and the work chains are further divided into processing threads (like shards, just different validators execute transactions in different smart contracts in parallel, they are added dynamically with increasing load and then deleted).</p>

<p>But what you really need to know is that you don't have to think about where your smart contracts are. All communications between contracts are asynchronous via sending messages. It doesn’t matter if two contracts are in the same processing thread or even in different workchains, any call to another contract is a message being sent, and if you are waiting for a response, it will come in another transaction.</p>

<br/>

### Everything is a smart contract!

<p>On Ethereum, there are two built-in types of accounts: contract accounts that are deployed to the network and externally owned (wallet) accounts that are controlled by anyone with private keys and can initiate transactions. On ES, there are only contracts, there are no built-in contracts in the form of wallets on the blockchain. A wallet is just a smart contract, and there are many different kinds. Transaction chains on ES can be started by any contract with the help of an external message (if the contract supports receiving external messages).</p>

<p>Transactions are started when a smart contract receives a message. During a transaction, a contract can send as many messages as it wants to another contract. (Once received, these messages will begin another transaction).</p>

<br/>

### There are two types of messages:

#### External message 

<p>A message from nowhere or to nowhere :-) This kind of message has only a destination address or a sender’s address. Primarily, this kind of message is used to call contracts from the real world. This is a fairly unique concept that allows any contract to start a chain of messages. </p>
<p>External messages work like this: you send a message with data from nowhere to a contract.</p>
<p>A validator allocates 10k of gas credit to this message, and attempts to complete the transaction by calling the contract and passing your message to it. The smart contract must agree to pay for the transaction from the contract account by calling the tvm.accept() method before it runs out of gas. </p>
<p>If this method is called then the transaction continues and the contract can create other outgoing messages.</p>
<p>If an exception occurs or the contract does not call tvm.accept() or the gas credit runs out, then the  message does not go to the blockchain and is discarded by the validator. (In simple terms, a message cannot get into the mempool if it cannot be successfully added to the blockchain).</p>
<p>Interestingly, an external message can contain any data and does not have to contain a signature. (For example, you can make a contract receive an arbitrary message every minute from anyone you’d like and as a result perform some kind of action, like a timer).</p>
<p>Here is an example of a simple wallet smart contract, which can only transfer money after receiving an external message:</p>

```solidity
pragma ton-solidity >= 0.35.0;


// This header informs sdk which will create the external message has to be signed by a key.
// Also directing the compiler that it should only accepted signed external
// messages
pragma AbiHeader pubkey;

contract Wallet {
    constructor() public {
        // We check that the conract has a pubkey set.
        // tvm.pubkey() - is essentially a static variable,
        // which is set at the moment of the creation of the contract,
        // We can set any pubkey here or just leave it empty.
        require(tvm.pubkey() != 0, 101);
        // msg.pubkey() - public key with which the message was signed, 
        // it can be  0 if the pragma AbiHeader pubkey has not been defined;
        // We check that the constructor was called by a message signed by a private key
        // from that pubkey that was set when the contract was deployed.
        require(msg.pubkey() == tvm.pubkey(), 102);
        // we agree to pay for the transaction calling the constructor
        // from the balance of the smart contract
        tvm.accept();
    }

    function send(address dest, uint128 value, bool bounce) public pure {
        // we check that the signature of the external message is right 
        require(msg.pubkey() == tvm.pubkey(), 100);
        // we aggree to pay for the external message from the contract balance
        tvm.accept();
        // everything is simple here, we create an outgoing message 
        // internal message which carries the value nano evers
        // bounce - a flag, which tells TVM what to do if dest contract 
        // does not exist or throws an exception on our internal message.
        // bounce - true - this will try to create a return message to us with an error 
        // and the remaining money (if there are enough funds to create a message),
        // bounce - false, this will leave the funds there.
        // 0 - the flag of message creation, we will get to that later.
        // 0 means that with the message we have to send value EVER 
        // and pay the fee for the creation of a message from that value.
        // (by sending a little less than the value)
        dest.transfer(value, bounce, 0);
    }
}
```

