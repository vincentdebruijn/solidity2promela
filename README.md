# solidity2promela
Converting Solidity contracts into PROMELA models

## Motivation
PROMELA (http://spinroot.com/spin/Man/Quick.html) is a verification modeling
language used to model distributed systems and verify the correctness of these
systems. Model checkers such as SPIN (http://spinroot.com/spin/whatispin.html)
check the state space of PROMELA models to find "invalid" states.
Invalidity is a user given property to the state, via "never" claims, e.g. the
tool can verify it is never the case that (my_funds > 10) and report any states
and traces to those states which invalidate the never claim.

Solidity (http://solidity.readthedocs.io/en/develop/index.html) is a programming
language used to encode "contracts" in the popular crypto currency Ethereum.
Correctness of these contracts is essential: one mistake can lead to third
parties seizing funds, manipulating voting systems, etc.

Contracts run on "gas", a predefined amount of operations the contract may
execute. This prevents infinite loops. However, this gas limit can be used by
attackers to halt the contract half-way; potentially yielding them some benefit.
Uncovering the state space yields a tree structure with a width and a depth.
the depth of this tree is the maximum amount of operations the contract can
execute. The model checking process can therefore assist with determining the
gas limit and with finding dangerous loops.

Applying model checking to Solidity contracts is therefore a benefit to the
Ethereum community.


## Abstractions
Sometimes in the conversion it is required to make abstractions to make the
testing feasible. This happens for example in the type conversion and in
limiting data ranges. The goal of this project is to get the tooling working
for simple contracts while making clear notes of abstractions and restrictions.
More complex contracts are incrementally supported by finding solutions and
workarounds for the abstractions and restrictions.

The tooling will automatically provide a list of abstractions made for the
provided Solidity contract and in the future the user will also be prompted to
influence these.


## General process
This section describes the general process of converting Solidity contracts into
PROMELA models.

PROMELA is a C-like language. In fact, it can be converted from a C program.
Therefore it supports conditionals, expressions, loops, variables and
structs (typedef). The full analysis is under [Typing].

The solidity contract is parsed (https://github.com/ConsenSys/solidity-parser)
and the AST is used to construct the PROMELA model.

In order to simulate user interaction via the contract, a user-given number of
agents is initialized. Each agent represents a "process" in the PROMELA model.
All generated code is surrounded by an "atomic" statement: the user agents calls
to the contract cannot be interleaved, this ensures the model reflects that.
The contract itself is also a process with state. All external functions can be
called by the user agents' processes via "message channels". More info on the
processes and message channels under [Processes & Channels].

Variables such as "msg", "block" and "tx" are initialized to the appropriate
values by the model. E.g. user agent 1 will have address 1 and thus when a call
is made from user agent 1 to the contract, msg.address will yield 1. More info
under [Solidity variables].

The user defines a set of properties via the never claims, e.g. never(F2 | ¬C1),
where F2 and C1 are properties. F2 represents the property "Agent 2 obtains
funds from the wallet" and C1 represents "Agent 1 gives consent to agent 2 to
obtain funds from the wallet". The claim therefore states that agent 2 cannot
obtain funds given agent 1 has not consented to this. More on LTL (Linear
Temporal Logic) here: https://en.wikipedia.org/wiki/Linear_temporal_logic.
More on never claims for Solidity contracts under [Properties].

Apart from the custom properties the tooling is able to automatically check for
undesirable behavior such as aforementioned gas attack and unreachable code.

Running the PROMELA model in Spin yields potential vulnerabilities, with an
analysis of how these vulnerabilities can be exploited. The execution depth is
limited to a factor of the gas (more under Gas limit). The tool will warn
whether this depth is reached and in which states.


## Typing
This section provides a detailed description of how types in Solidity are
converted to PROMELA.

### int
Solidity has uint8 until uint256 and int8 until int256, in steps of 8.
PROMELA has signed and unsigned bytes, shorts, ints and longs. Therefore, there
currently is only support for the 8, 16, 32 and 64 bit values.

In the first prototype, the unsupported values are converted to one of these
types (user preference). In the future this will be investigated to see
whether bit arrays or other structures can be feasibly used to implement this.

### float/Time
Solidity does not contain floats or doubles. Time is represented as an uint
where 1 is 1 second.

### bool
Solidity has no type conversion of non-bool to bool, such as in C and PROMELA,
therefore there is no problem with truthy or falsy values.

### string
A string is mapped to a byte[n], where n is the length of the string in bytes.

### Array
Solidity contains static and dynamic arrays. PROMELA only contains static arrays.
Therefore, a maximum length is set for each dynamic array (this value can be
influenced in the future). The dynamic array is a new type:
  typedef DynamicArray {
    \_ElementType[N] elements;
    unsigned int length;
  }

The getter and setter calls to the dynamic array are passed to elements. The
'length' member call is converted to return the length value. The
'length = var.push(value)' member call is converted to:
  var.elements[length] = value;
  length = var.length++;

### Mapping
"Mapping types are declared as mapping(\_KeyType => \_ValueType). Here \_KeyType
can be almost any type except for a mapping, a dynamically sized array, a
contract, an enum and a struct. \_ValueType can actually be any type, including
mappings."

PROMELA does not contain a type for Mapping, so one has to be constructed using:
  typedef Mapping {
    \_KeyType keys[n];
    \_ValueType values[n];
  }

where n needs to be set to a suitable number. Getters and setters have to be
defined.

### Function
Functions are mapped to channels in PROMELA. A channnel (chan) is an object
which can be be passed around, like a function in Solidity. More about this in
[Processes & Channels].

### Event
Events are used for logging, they do not alter control flow. However, they are
suitable for expressing properties. Event is mapped to a new type:
  typedef Event {
    int count;
  }

Calling the event ups its count by one.

### enum
An enum in Solidity is mapped to a Symbolic constant in PROMELA.

### address
Solidity has a special type address, which is an identifier of a user and
cannot be modified or used in algebraic operations. In the PROMELA model, this
is defined as a constant byte value.

### require
The require statement ensures some condition to be true. If the condition is not
true, an error is raised and execution is stopped. In PROMELA, this is
considered as a valid end state. In other words: a leaf of the state space tree.


## Processes & Channels
This section describes how processes and channels in PROMELA are utilized to
model users, contracts and function calls.

### Contracts
Contracts are converted to processes which only contain variables and channels.
The channels execute the function code, which is described in the next section.

If a contract dynamically creates and call to another contract this also the
channels of that contract.

### Channels
External functions of a contract are mapped to channels in the PROMELA model.
Calling a function is done by using a 'send' on the channel with the argument
values (or some dummy value if there are not arguments). It then proceeds to
call 'receive' on a second channel to wait for the result. The contract process
calls a 'receive' on the first channel (which was blocking until now), executes
the function code and sends the return result to the second channel (or a dummy
value again).

### Users
The user agent process is a loop over a conditional expression with
receives/sends to the channels of the contract as branches. A lock is obtained
and once released the block count and timestamp are upped and other users may
claim the lock. This allows Spin to mimic users calling functions in sequence
within the same block or spread out over several blocks, possibly interleaved
with other user calls.

The values given to the channels are determined by Spin. It will exhaustively
explore all data inputs to the functions and will therefore encounter a
state space explosion. Restrictions on data are needed to counter this. For each
argument, a set of possible values is defined (may include ranges). The user
agent processes will first choose the input values according to this set.


## Solidity variables
The variables provided to Solidity contracts is mentioned here:
http://solidity.readthedocs.io/en/develop/units-and-global-variables.html
This section will describe how the model sets and updates these values.

### block.coinbase
Always returns Address with byte 0. In future, miner agents can be added.

### block.difficulty
Returns random uint.

### block.gaslimit
Returns the give gas limit value.

### block.number
Returns a number starting at 1 and is increased by the user agents.

### block.timestamp (now)
Is initialized to the current time and set to new current time between blocks.
TODO: Is long enough to store the timestamp?

### msg.data
TODO: check what this should be.

### msg.gas
Keep track of the gas value in a variable in the model.
TODO: how is the gas depletion calculated?

### msg.sender
Is set to the user agent's address making the request and set to the contract's
address if it dynamically calls another contract.

### msg.sig
the first 4 bytes of the msg.data.

### tx.gasprice
TODO: check what this should be.

### tx.origin
Is always set to user agent's address making the request.


## Properties
TODO: stub

Determining number of user agents and the properties is an exercise for the user.
Complete and thorough testing depend on this. Benefit is the abstraction from
code details: higher level analysis.

Why determine number of user agents? More agents == more states. The fewer
agents required the better.


## Gas limit
TODO: How does state space depth relate to gas limit?

## Examples
TODO: provide some contract examples with their converted counterparts.
