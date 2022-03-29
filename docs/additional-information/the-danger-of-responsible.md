---
layout: default
title: The danger of default responsible
parent: Additional information
nav_order: 4
---

## The danger of default “responsible”

To start, let’s think of “responsible” like sugar. 

You also need to know that the default responsible function is a potential place for a vulnerability, with the help of which an attacker can bring the balance of your contract to 0. And if the cost of gas is lowered, this will definitely be used, because it will be a way to earn ever by stealing them from your account . (You will spend less ever on a call than you will receive back).

The following is incorrect:
```solidity
function owner() override external responsible returns (address) {
   return owner_;
}
```

<br />

This is correct:
```solidity
function owner() override external responsible returns (address) {
  return { value: 0, flag: 64, bounce: false } owner_;
}
```

<br />

The key here is the magic that is hidden behind the Solidity compiler and the code word “responsible”

How does responsible work? Under the hood, one more variable has to be added, which points to the id of the function that should be called in the return message. In fact, the example above after compilation turns into something like this:

```solidity
//PSEUDOCODE FOR UNDERSTANDING, THE ACTUAL RETURN ANSWER IS MORE COMPLICATED

function owner() override external responsible returns (address) {
        return owner_;
}

//this becomes something like this

function owner(uint32 asnwerID) override external returns (address) {
    msg.sender.call({
        // value depend on compiler version
        flag: 0,
        bounce: true,
        functionToCall: asnwerID,
        values: owner_
    })
}
```

So this sugar simply adds to the answerID call, and the compiler forces the creation of the callback. But as you should have understood, from “Carefully working with VALUE when creating messages” with a flag of 0, the value can be paid from the contract account. So, an attacker could pick an amount of ever to attach to your responsible call that it is enough to pay for gas but not enough to pay for the return 7_500_000. And these 7_500_000 will be paid from the contract account.

Why is a minimal amount of value attached to the default answer for responsible? Because sending a message with a 0 value is pointless, it will not pass the initial checks of the smart contract and will not reach the tvm.accept() call.

And if we set flag: 64 and value: 0, then it will turn out that in the return message we will simply send the entire remaining balance from the incoming message and will not spend the smart contract money (this works if there were no other messages created in this transaction. If there are, use rawReverse and 128, also see “Carefully working with VALUE when creating messages”).

```solidity

  function owner() override external responsible returns (address) {
  return { value: 0, flag: 64, bounce: false }owner_;
}

// This is compiled into something like this
// PSEUDOCODE FOR UNDERSTANDING, THE ACTUAL RETURN ANSWER IS MORE COMPLICATED.
function owner(uint32 asnwerID) override external returns (address) {
    msg.sender.call({
        value: 0,
        flag: 64,
        bounce: false,
        functionToCall: asnwerID,
        values: owner_
    })
}
```

Why doesn’t responsible have a value: 0 and flag: 64 by default? Because again, as explained in the previous chapter, flag: 64 can only be used if you did not create any other outgoing messages in this transaction. That is, the vulnerability would still remain, it just wouldn’t be a prevalent.

By the way, it hasn’t been tested, but it seems that payment from the contract account also goes to create an External message (a message from the contract that goes nowhere). For example, you also need to be careful with the emit Event() call.

There is also a problem with the default setting for responsible bounce: true, and there is a problem if your contract uses onBounce. An attacker can pick  a similar answerID to the one you use for onBounce, and when he receives a response, create an error so that onBounce is unexpectedly sent to you. It is complicated to understand, you can look on example of vulnerability [here](https://github.com/tonlabs/TON-Solidity-Compiler/issues/87)

<br />
