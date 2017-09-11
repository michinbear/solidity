###############################
Introduction to Smart Contracts
###############################

.. _simple-smart-contract:

***********************
A Simple Smart Contract
***********************

Let us begin with the most basic example. It is fine if you do not understand everything
right now, we will go into more detail later.

Storage
=======

::

    pragma solidity ^0.4.0;

    contract SimpleStorage {
        uint storedData;

        function set(uint x) {
            storedData = x;
        }

        function get() constant returns (uint) {
            return storedData;
        }
    }

The first line simply tells that the source code is written for
Solidity version 0.4.0 or anything newer that does not break functionality
(up to, but not including, version 0.5.0). This is to ensure that the
contract does not suddenly behave differently with a new compiler version. The keyword ``pragma`` is called that way because, in general,
pragmas are instructions for the compiler about how to treat the
source code (e.g. `pragma once <https://en.wikipedia.org/wiki/Pragma_once>`_).

A contract in the sense of Solidity is a collection of code (its *functions*) and
data (its *state*) that resides at a specific address on the Ethereum
블록체인. The line ``uint storedData;`` declares a state variable called ``storedData`` of
type ``uint`` (unsigned integer of 256 bits). You can think of it as a single slot
in a database that can be queried and altered by calling functions of the
code that manages the database. In the case of Ethereum, this is always the owning
contract. And in this case, the functions ``set`` and ``get`` can be used to modify
or retrieve the value of the variable.

To access a state variable, you do not need the prefix ``this.`` as is common in
other languages.

This contract does not do much yet (due to the infrastructure
built by Ethereum) apart from allowing anyone to store a single number that is accessible by
anyone in the world without a (feasible) way to prevent you from publishing
this number. Of course, anyone could just call ``set`` again with a different value
and overwrite your number, but the number will still be stored in the history
of the 블록체인. Later, we will see how you can impose access restrictions
so that only you can alter the number.

.. note::
    All identifiers (contract names, function names and variable names) are restricted to
    the ASCII character set. It is possible to store UTF-8 encoded data in string variables.

.. warning::
    Be careful with using Unicode text as similarly looking (or even identical) can have different
    code points and as such will be encoded as a different byte array.

.. index:: ! subcurrency

Subcurrency Example
===================

The following contract will implement the simplest form of a
cryptocurrency. It is possible to generate coins out of thin air, but
only the person that created the contract will be able to do that (it is trivial
to implement a different issuance scheme).
Furthermore, anyone can send coins to each other without any need for
registering with username and password - all you need is an Ethereum keypair.


::

    pragma solidity ^0.4.0;

    contract Coin {
        // The keyword "public" makes those variables
        // readable from outside.
        address public minter;
        mapping (address => uint) public balances;

        // Events allow light clients to react on
        // changes efficiently.
        event Sent(address from, address to, uint amount);

        // This is the constructor whose code is
        // run only when the contract is created.
        function Coin() {
            minter = msg.sender;
        }

        function mint(address receiver, uint amount) {
            if (msg.sender != minter) return;
            balances[receiver] += amount;
        }

        function send(address receiver, uint amount) {
            if (balances[msg.sender] < amount) return;
            balances[msg.sender] -= amount;
            balances[receiver] += amount;
            Sent(msg.sender, receiver, amount);
        }
    }

This contract introduces some new concepts, let us go through them one by one.

The line ``address public minter;`` declares a state variable of type address
that is publicly accessible. The ``address`` type is a 160-bit value
that does not allow any arithmetic operations. It is suitable for
storing addresses of contracts or keypairs belonging to external
persons. The keyword ``public`` automatically generates a function that
allows you to access the current value of the state variable.
Without this keyword, other contracts have no way to access the variable.
The function will look something like this::

    function minter() returns (address) { return minter; }

Of course, adding a function exactly like that will not work
because we would have a
function and a state variable with the same name, but hopefully, you
get the idea - the compiler figures that out for you.

.. index:: mapping

The next line, ``mapping (address => uint) public balances;`` also
creates a public state variable, but it is a more complex datatype.
The type maps addresses to unsigned integers.
Mappings can be seen as `hash tables <https://en.wikipedia.org/wiki/Hash_table>`_ which are
virtually initialized such that every possible key exists and is mapped to a
value whose byte-representation is all zeros. This analogy does not go
too far, though, as it is neither possible to obtain a list of all keys of
a mapping, nor a list of all values. So either keep in mind (or
better, keep a list or use a more advanced data type) what you
added to the mapping or use it in a context where this is not needed,
like this one. The :ref:`getter function<getter-functions>` created by the ``public`` keyword
is a bit more complex in this case. It roughly looks like the
following::

    function balances(address _account) returns (uint) {
        return balances[_account];
    }

As you see, you can use this function to easily query the balance of a
single account.

.. index:: event

The line ``event Sent(address from, address to, uint amount);`` declares
a so-called "event" which is fired in the last line of the function
``send``. User interfaces (as well as server applications of course) can
listen for those events being fired on the 블록체인 without much
cost. As soon as it is fired, the listener will also receive the
arguments ``from``, ``to`` and ``amount``, which makes it easy to track
transactions. In order to listen for this event, you would use ::

    Coin.Sent().watch({}, '', function(error, result) {
        if (!error) {
            console.log("Coin transfer: " + result.args.amount +
                " coins were sent from " + result.args.from +
                " to " + result.args.to + ".");
            console.log("Balances now:\n" +
                "Sender: " + Coin.balances.call(result.args.from) +
                "Receiver: " + Coin.balances.call(result.args.to));
        }
    })

Note how the automatically generated function ``balances`` is called from
the user interface.

.. index:: coin

The special function ``Coin`` is the
constructor which is run during creation of the contract and
cannot be called afterwards. It permanently stores the address of the person creating the
contract: ``msg`` (together with ``tx`` and ``block``) is a magic global variable that
contains some properties which allow access to the 블록체인. ``msg.sender`` is
always the address where the current (external) function call came from.

Finally, the functions that will actually end up with the contract and can be called
by users and contracts alike are ``mint`` and ``send``.
If ``mint`` is called by anyone except the account that created the contract,
nothing will happen. On the other hand, ``send`` can be used by anyone (who already
has some of these coins) to send coins to anyone else. Note that if you use
this contract to send coins to an address, you will not see anything when you
look at that address on a 블록체인 explorer, because the fact that you sent
coins and the changed balances are only stored in the data storage of this
particular coin contract. By the use of events it is relatively easy to create
a "블록체인 explorer" that tracks transactions and balances of your new coin.

.. _블록체인-basics:

*****************
블록체인 기초
*****************

블록체인이라는 개념은 프로그래머가 이해하기에는 어렵지 않다. 왜냐하면 대부분의 complications (mining, `hashing <https://en.wikipedia.org/wiki/Cryptographic_hash_function>`_, `elliptic-curve cryptography <https://en.wikipedia.org/wiki/Elliptic_curve_cryptography>`_, `peer-to-peer networks <https://en.wikipedia.org/wiki/Peer-to-peer>`_, etc.) 들은 기능과 약속들의 집합을 제공하기 위한 것 뿐이기 때문이다.
여러분이 이런 기능들을 받아들이기만 한다면, 그 밑에 깔린 기술에 대해서는 신경쓰지 않아도 무방하다. 아니면 당신은 아마존의 AWS를 사용하기 위해서 그것이 내부적으로 어떻게 동작하는지 알아야만 속이 풀리나?

.. index:: transaction

트랜잭션
============

블록체인은 글로벌하게 공유되는 장부 데이터베이스이다.
그 말은 네트워크에 참여하는 아무나 데이터베이스의 entry들을 읽을 수 있다는 뜻이다.
당신이 데이터베이스의 뭔가를 수정하려면 다른 모두가 납득할 수 있는 트랜잭션을 생성해야만 한다.
트랜잭션이란 당신이 의도한 변화 또는 수정 ( 아마도 당신은 한번에 두가지 값을 변화시키고 싶을 것이다. )이 전혀 이루어지지 않거나, 완벽하게 수행된다는 것을 내포한다.
게다가, 그 트랜잭션이 데이터베이스에 적용되는 동안 다른 트랜잭션으로 바꿔치기 될 수 없다.

예를 들어서 전자화폐의 모든 계좌의 잔고 목록을 상상해보라.
만약 한 계좌에서 다른 계좌로 이체를 요청했다면, 데이터베이스의 트랜잭션은 한 계정에서는 잔고를 줄였다면, 다른 계좌에서는 잔고를 늘리는 것을 보장한다.
어떤 이유에서든지, 이체 계좌의 잔고를 늘리는 것이 불가능 하다면, 송금 계좌의 잔고 또한변함이 없어야한다.

게다가 트랜잭션은 트랜잭션 생성자, 송신측에 의해 항상 서명된다. 이것이 데이터베이스의 변조를 막는다(This makes it straightforward to guard access to specific modifications of the
database.)
전자화폐의 예에서 단순한 검증만으로도 계좌의 키를 가지고 있는 사람만이 돈을 이체할 수 있도록 보장한다.

.. index:: ! block

블록
======

비트코인에서 한가지 장애물은 대체 "중복 결제 공격 ( double-spend attack )"이 뭐냐는 것이다.:
동일한 네트워크에서 계정을 비워버리라는 두개의 트랜잭션이 존재할 때, 즉 충돌할 때 대체 무슨 일이 벌어지는가.

간단한 대답은 여러분은 그것을 신경쓸 필요 없다는 것이다. 트랜잭션 순서는 여러분을 위해 선택되어, 해당 트랜잭션은 "블록"으로 감싸지고 모든 참여자 노드에서 실행되고 배포될 것이다.
2개의 트랜잭션이 충돌한다면 두번째 트랜잭션은 거절되고 블록에 포함되지 않을 것이다.(If two transactions contradict each other, the one that ends up being second will
be rejected and not become part of the block.)

이 블록들은 시간이 지남에 따라 사슬처럼 연쇄적으로 연결되고, 이것이 블록체인이라고 부르는 이유다.
블록은 일정 간격 ( 이더리움에서는 대충 17초마다 )마다 체인에 추가된다.

채굴이라고 불리는 "order selection mechanism"에서는
As part of the "order selection mechanism" (which is called "mining") it may happen that
blocks are reverted from time to time, but only at the "tip" of the chain.

The more blocks that are added on top, the less likely it is. So it might be that your transactions
are reverted and even removed from the 블록체인, but the longer you wait, the less
likely it will be.


.. _the-ethereum-virtual-machine:

.. index:: !evm, ! ethereum virtual machine

****************************
이더리움 가상 머신
****************************

개요
========

이더리움 가상머신 ( EVM )은 이더리움의 스마트 컨트랙트를 위한 실행환경이다.
이것은 샌드박스일 뿐만 아니라 실제로 완벽하게 격리되어 있어서 EVM내부에서 실행되는 코드는 네트워크, 파일시스템, 다른 프로세스로 접근할 수 없다.
스마트 컨트랙트는 다른 스마트 컨트랙트로의 접근도 제한되어 있다.

.. index:: ! account, address, storage, balance

Accounts
========

이더리움에서는 Address 항목을 공유하는 두가지 종류의 계정이 있다. : **External accounts** 는 공개키-개인키 쌍 ( 예를 들어 인간 )에 의해 제어되고, **contract accounts**는 계정 내부에 들어있는 코드에 의해서 제어된다.
외부 계정의 주소는 공개키로 정해지는 데 반해 계약 계정의 주소는 계약이 생성될 때 정해진다. ( 이것은 생성자의주소와 그 주소로부터 발생한 트랜잭션의 횟수 - nonce - 에 의해 정해진다 )
계정이 코드를 가지고 있던 그렇지 않던지 간에, EVM에서는 동일하게 다뤄진다.
모든 계정은 **storage**라고 불리는 영구적인 256비트 words를 256비트 words로 맵핑하는 키-값 저장소를 가진다.
게다가 모든 계정은 이더를 포함하는 트랜잭션을 보낼 때 수정될 수 있는 **balance** ( 이더, 정확히는 Wei )를 가지고 있다.

.. index:: ! transaction

Transactions
============

트랜잭션은 한 계정에서 다른 계정으로 보내는 메세지이다. 받는 계정이 보내는 계정과 동일하거나 zero-account일 수도 있는데 아래를 참고하라.

만약 받는 계정이 코드를 포함하고 있다면 그 코드가 실행되고 payload는 입력 데이터로 제공된다.

만약 받는 계정이 주소가 ``0`` 인 zero-account라면 트랜잭션은 **신규 컨트랙트**를 생성한다.
이미 언급한 것 처럼, 그 컨트랙트의 주소는 ``0`` 이 아니고, 송신자와 트랜잭션이 보내진 횟수("nonce")로 정해진다.
이런 컨트랙트 생성을 위한 트랜잭션의 payload는 EVM 코드로 변환되고 실행된다.
실행 결과는 컨트랙트의 코드로 영구히 저장된다.
즉, 컨트랙트를 생성하기 위해서, 여러분은 컨트랙트의 실제 코드를 보내는 것이 아니라 실제 코드를 반환하는 코드를 보내는 것이다.
The payload
of such a contract creation transaction is taken to be
EVM bytecode and executed. The output of this execution is
permanently stored as the code of the contract.
This means that in order to create a contract, you do not
send the actual code of the contract, but in fact code that
returns that code.

.. index:: ! gas, ! gas price

Gas
===

생성된 후에 각 트랜잭션은 일정한 양의 **gas**를 지불해야하는데, 이것은 트랜잭션을 실행하는데 필요한 일의 양을 제한하고,
이 실행에 대한 비용을 지불하기 위함이다. EVM이 트랜잭션을 실행하는 동안 gas는 특정한 규칙에 따라서 점차적으로 감소한다.
Upon creation, each transaction is charged with a certain amount of gas, whose purpose is to limit the amount of work that is needed to execute the transaction and to pay for this execution. While the EVM executes the transaction, the gas is gradually depleted according to specific rules.


**gas price**는 ``gas_price * gas`` 를 송신 계정으로부터 선금으로 지불해야 하는 트랜잭션 생성자에 의해 정해진다.(?)
만약 실행 이후에 gas가 남아있다면 동일한 방식으로 환불된다.
The gas price is a value set by the creator of the transaction, who has to pay gas_price * gas up front from the sending account. If some gas is left after the execution, it is refunded in the same way.

만약 gas가 어느 순간 소진되었다면, out-of-gas 예외가 발생하는데, 이는 현재 call frame에서 발생한 모든 변경 사항을 되돌린다.
If the gas is used up at any point (i.e. it is negative), an out-of-gas exception is triggered, which reverts all modifications made to the state in the current call frame.


.. index:: ! storage, ! memory, ! stack

Storage, Memory and the Stack
=============================

각 계정은 **storage** 라는 영구 메모리 영역을 가진다. Storage는 256-bit words에서 256-bit words로 매핑되는 Key-value 쌍을 저장한다.
컨트랙트 내부에서 storage 데이터를 하나하나 열거하는 것은 불가능하며,
컨트랙트는 외부 storage를 읽거나 쓸 수 없다.
.. Each account has a persistent memory area which is called **storage**.
.. Storage is a key-value store that maps 256-bit words to 256-bit words.
.. It is not possible to enumerate storage from within a contract
.. and it is comparatively costly to read and even more so, to modify
.. storage. A contract can neither read nor write to any storage apart
.. from its own.

두번째 메모리 영역은 **memory**인데, 컨트랙트는 각 메세지 호출마다 초기화된 memory 인스턴스를 얻어온다.
Memory는 연속적이며 (선형적이며) 바이트 레벨의 주소를 가지지만 쓰기가 8 bit 혹은 256 bit로 가능한데 반해, 읽기는 256 bit 단위로만 가능하다.
Memory는 예전에 접근한 적 없는 memory word ( 256-bit )에 접근할 때 word(256-bit) 단위로 확장된다.
메모리 영역을 과장할 때 가스 비용을 내야한다. 메모리는 커질 수록 비싸진다. (quadratically하게 비싸진다.)

.. The second memory area is called **memory**, of which a contract obtains
.. a freshly cleared instance for each message call. Memory is linear and can be
.. addressed at byte level, but reads are limited to a width of 256 bits, while writes
.. can be either 8 bits or 256 bits wide. Memory is expanded by a word (256-bit), when
.. accessing (either reading or writing) a previously untouched memory word (ie. any offset
.. within a word). At the time of expansion, the cost in gas must be paid. Memory is more
.. costly the larger it grows (it scales quadratically).

EVM은 register 머신이 아니라 stack 머신이므로 모든 계산은 **stack**이라고 불리는 영역에서 이뤄진다.
**stack**의 최대 사이즈는 1024 elements이며 256 bits의 word를 포함한다. stack은 아래와 같은 방법으로
최상위에만 접근할 수 있다.:
stack의 최상위 16 elements 중 하나를 복사하거나 최상위 element를 그 아래 16개 element 중 하나와 교체하는 것이 가능하다.
다른 모든 operation은 최상위 2개 ( 혹은 하나, 혹은 그 이상. opration에 따라 다르다.) elements를 꺼내고 그 결과를 stack위에 쌓는다.
물론 stack의 element를 storage나 memory에 옮기는 것도 가능하지만 스택의 최상위 요소를 제거하지 않고는 임의 접근이 불가능하다.

.. The EVM is not a register machine but a stack machine, so all
.. computations are performed on an area called the **stack**. It has a maximum size of
.. 1024 elements and contains words of 256 bits. Access to the stack is
.. limited to the top end in the following way:
.. It is possible to copy one of
.. the topmost 16 elements to the top of the stack or swap the
.. topmost element with one of the 16 elements below it.
.. All other operations take the topmost two (or one, or more, depending on
.. the operation) elements from the stack and push the result onto the stack.
.. Of course it is possible to move stack elements to storage or memory,
.. but it is not possible to just access arbitrary elements deeper in the stack
.. without first removing the top of the stack.

.. index:: ! instruction

Instruction Set
===============

EVM의 instruction 지합은 consensus 문제를 야기할 수 있는 잘못된 구현을 피하기 위해 최소한으로 유지한다.
모든 instruction 256-bit words와 같은 기본적인 데이터 타입을 다룬다.
일반적인 산술연산, 비트, 로직, 비교 연산 등이 제공된다
조건, 무조건적 점프도 가능하다.
게다가 컨트랙트는 현재 블록의 숫자와 타임스탬프와 같은 관련된 프로퍼티에 접근할 수 있다.

.. The instruction set of the EVM is kept minimal in order to avoid
.. incorrect implementations which could cause consensus problems.
.. All instructions operate on the basic data type, 256-bit words.
.. The usual arithmetic, bit, logical and comparison operations are present.
.. Conditional and unconditional jumps are possible. Furthermore,
.. contracts can access relevant properties of the current block
.. like its number and timestamp.

.. index:: ! message call, function;call

Message Calls
=============

컨트랙트는 메세지 호출을 통해 다른 컨트랙트를 호출하거나 비-컨트랙트 계정으로 이더를 보낼 수 있다.
메세지 호출은 송신자, 수신자, 데이타 페이로드, 이더, 가스, 반환값을 가진다는 점에서 트랜잭션과 비슷하다.
사실 모든 트랜잭션은 추가적인 메세지 호출을 차례대로 생성할 수 있는 최상위 메세지 호출로 구성된다.

.. Contracts can call other contracts or send Ether to non-contract
.. accounts by the means of message calls. Message calls are similar
.. to transactions, in that they have a source, a target, data payload,
.. Ether, gas and return data. In fact, every transaction consists of
.. a top-level message call which in turn can create further message calls.

컨트랙트는 내부 메세지 호출을 통해 어느 정도의 잔여 갓를 보내고 얼마나 남길지 정할 수 있다.
만약 내부 호출에서 가스 부족 예외와 같은 예외가 발생하면, stack위에 에러값이 추가되는 것으로 알 수 있다.
이 경우 호출과 함께 전송된 가스만이 사용된다.
solidity에서는 특정 상황에서 컨트랙트를 호출하면 기본값으로 수동 예외를 발생시키는데, 이로인해 call stack이 예외로 가득차게 된다.

.. A contract can decide how much of its remaining **gas** should be sent
.. with the inner message call and how much it wants to retain.
.. If an out-of-gas exception happens in the inner call (or any
.. other exception), this will be signalled by an error value put onto the stack.
.. In this case, only the gas sent together with the call is used up.
.. In Solidity, the calling contract causes a manual exception by default in
.. such situations, so that exceptions "bubble up" the call stack.

이미 언급한 것 처럼 호출된 컨트랙트는 초기화된 메모리 인스턴스를 전달받아 **calldata** 라고 하는 분리된 영역에 있는
call payload에 접근할 수 있는 권한을 가진다.
실행이 끝나면 호출자의 할당된 메모ㄷ리에 반환값을 저장한다.
.. As already said, the called contract (which can be the same as the caller)
.. will receive a freshly cleared instance of memory and has access to the
.. call payload - which will be provided in a separate area called the **calldata**.
.. After it has finished execution, it can return data which will be stored at
.. a location in the caller's memory preallocated by the caller.

호출은 1024 깊이로 **제한**되어있으므로 복잡한 연산을 위해서는 재귀 호출보다는 루프를 추천한다.
.. Calls are **limited** to a depth of 1024, which means that for more complex
.. operations, loops should be preferred over recursive calls.

.. index:: delegatecall, callcode, library

Delegatecall / Callcode and Libraries
=====================================

**delegatecall**이라는 특수한 메세지 호출이 있다. 이것은 몇가지를 제외하면 메세지 호출과 동일하다.
메세지 호출과 다른 점은 수신자 주소의 코드가 호출 컨트랙트의 컨텍스트에서 실행되고,
``msg.sender`` 와 ``msg.value`` 값이 변하지 않는 다는 것이다.

.. There exists a special variant of a message call, named **delegatecall**
.. which is identical to a message call apart from the fact that
.. the code at the target address is executed in the context of the calling
.. contract and ``msg.sender`` and ``msg.value`` do not change their values.

이것은 컨트랙트가 런타임에 다른 주소에서 동적으로 코드를 가져올 수 있다는 것을 의미한다.
Storage, 현재 주소와 잔고는 여전히 호출하는 컨트랙트를 가리키지만 오직 코드만이 호출된 주소로부터 가져오게 된다.

.. This means that a contract can dynamically load code from a different
.. address at runtime. Storage, current address and balance still
.. refer to the calling contract, only the code is taken from the called address.

이것이 solidity에서 라이브러리 기능 구현을 가능하게 한다.
예를 들어 복잡한 구조체를 구현하기 위하여 컨트랙트의 storage에 재사용 가능한 라이브러리 코드를 적용할 수 있다.
.. This makes it possible to implement the "library" feature in Solidity:
.. Reusable library code that can be applied to a contract's storage, e.g. in
.. order to  implement a complex data structure.

.. index:: log

Logs
====

블록레벨까지 모두 맵핑되어 있는 특수한 열거형 데이터 구조체에 데이터를 저장할 수 있다.
**logs** 라는 이 기능은 **events**를 구현하기 위해 Solidity에서 사용되었다.
컨트랙트는 생성된 이후에는 로그 데이터에 접근할 수 없지만, 블록체인 밖에서는 효율적으로 접근할 수 있다.
로그 데이터의 일부가 `bloom filters <https://en.wikipedia.org/wiki/Bloom_filter>`_에 저장되어 있는데,
이 데이터를 효율적이고 cryptographically secure한 방법으로 검색할 수 있어서, 블록체인 전체를 다운로드받지 않은
네트워크 피어 ( "light clients" )도 이 로그들을 검색할수 있다.

.. It is possible to store data in a specially indexed data structure
.. that maps all the way up to the block level. This feature called **logs**
.. is used by Solidity in order to implement **events**.
.. Contracts cannot access log data after it has been created, but they
.. can be efficiently accessed from outside the 블록체인.
.. Since some part of the log data is stored in `bloom filters <https://en.wikipedia.org/wiki/Bloom_filter>`_, it is
.. possible to search for this data in an efficient and cryptographically
.. secure way, so network peers that do not download the whole 블록체인
.. ("light clients") can still find these logs.

.. index:: contract creation

Create
======

컨트랙트는 심지어 특수한 opcode를 이용해서 다른 컨트랙트를 생성할 수 있다.
(i.e. they do not simply call the zero address).
**create calls**와 일반 메세지 호출의 유일한 차이점은 페이로드 데이터가 실행되고 결과가 코드형태로 저장되며,
호출자/생성자가 스택의 신규 컨트랙트의 주소를 받는 다는 점이다.


.. Contracts can even create other contracts using a special opcode (i.e.
.. they do not simply call the zero address). The only difference between
.. these **create calls** and normal message calls is that the payload data is
.. executed and the result stored as code and the caller / creator
.. receives the address of the new contract on the stack.

.. index:: selfdestruct

Self-destruct
=============

블록체인에서 코드가 제거되는 것은 그 주소의 컨트랙트가 ``selfdestruct`` 를 수행했을 때 뿐이다.
그 주소에 남아있는 이더는 designated target에게 본지고 storage와 코드가 state로부터 제거된다.

.. The only possibility that code is removed from the 블록체인 is
.. when a contract at that address performs the ``selfdestruct`` operation.
.. The remaining Ether stored at that address is sent to a designated
.. target and then the storage and code is removed from the state.



.. warning:: Even if a contract's code does not contain a call to ``selfdestruct``,
  it can still perform that operation using ``delegatecall`` or ``callcode``.

.. note:: The pruning of old contracts may or may not be implemented by Ethereum
  clients. Additionally, archive nodes could choose to keep the contract storage
  and code indefinitely.

.. note:: Currently **external accounts** cannot be removed from the state.
