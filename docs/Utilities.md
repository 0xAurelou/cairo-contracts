# Utilities

The following documentation provides context, reasoning, and examples for methods and constants found in `tests/utils.py`.  

> Expect this module to evolve (as it has already done).

## Table of Contents

* [Constants](#constants)
* [Strings](#strings)
  * [`str_to_felt`](#str_to_felt)
  * [`felt_to_str`](#felt_to_str)
* [Uint256](#uint256)
  * [`uint`](#uint)
  * [`to_uint`](#to_uint)
  * [`from_uint`](#from_uint)
  * [`add_uint`](#add_uint)
  * [`sub_uint`](#sub_uint)
* [Assertions](#assertions)
  * [`assert_revert`](#assert_revert)
  * [`assert_revert_entry_point`](#assert_revert_entry_point)
  * [`assert_events_emitted`](#assert_event_emitted)
* [Memoization](#memoization)
  * [`get_contract_class`](#get_contract_class)
  * [`cached_contract`](#cached_contract)
* [MockSigner](#mocksigner)

## Constants

To ease the readability of Cairo contracts, this project includes reusable [constants variables](../src/openzeppelin/utils/constants.cairo) like `UINT8_MAX`, or EIP165 interface IDs such as `IERC165_ID` or `IERC721_ID`. For more information on how interface ids are calculated, see the [ERC165 documentation](../docs/Introspection.md#interface-calculations).

## Strings

Cairo currently only provides support for short string literals (less than 32 characters). Note that short strings aren't really strings, rather, they're representations of Cairo field elements. The following methods provide a simple conversion to/from field elements.

### `str_to_felt`

Takes an ASCII string and converts it to a field element via big endian representation.

### `felt_to_str`

Takes an integer and converts it to an ASCII string by trimming the null bytes and decoding the remaining bits.

## Uint256

Cairo's native data type is a field element (felt). Felts equate to 252 bits which poses a problem regarding 256-bit integer integration. To resolve the bit discrepancy, Cairo represents 256-bit integers as a struct of two 128-bit integers. Further, the low bits precede the high bits e.g.

```python
1 = (1, 0)
1 << 128 = (0, 1)
(1 << 128) - 1 = (340282366920938463463374607431768211455, 0)
```

### `uint`

Converts a simple integer into a uint256-ish tuple.

> Note `to_uint` should be used in favor of `uint`, as `uint` only returns the low bits of the tuple.

### `to_uint`

Converts an integer into a uint256-ish tuple.

```python
x = to_uint(340282366920938463463374607431768211456)
print(x)
# prints (0, 1)
```

### `from_uint`

Converts a uin256-ish tuple into an integer.

```python
x = (0, 1)
y = from_uint(x)
print(y)
# prints 340282366920938463463374607431768211456
```

### `add_uint`

Performs addition between two uint256-ish tuples and returns the sum as a uint256-ish tuple.

```python
x = (0, 1)
y = (1, 0)
z = add_uint(x, y)
print(z)
# prints (1, 1)
```

### `sub_uint`

Performs subtraction between two uint256-ish tuples and returns the difference as a uint256-ish tuple.

```python
x = (0, 1)
y = (1, 0)
z = sub_uint(x, y)
print(z)
# prints (340282366920938463463374607431768211455, 0)
```

### `mul_uint`

Performs multiplication between two uint256-ish tuples and returns the product as a uint256-ish tuple.

```python
x = (0, 10)
y = (2, 0)
z = mul_uint(x, y)
print(z)
# prints (0, 20)
```

### `div_rem_uint`

Performs division between two uint256-ish tuples and returns both the quotient and remainder as uint256-ish tuples respectively.

```python
x = (1, 100)
y = (0, 25)
z = div_rem_uint(x, y)
print(z)
# prints ((4, 0), (1, 0)) 
```

## Assertions

In order to abstract away some of the verbosity regarding test assertions on StarkNet transactions, this project includes the following helper methods:

### `assert_revert`

An asynchronous wrapper method that executes a try-except pattern for transactions that should fail. Note that this wrapper does not check for a StarkNet error code. This allows for more flexibility in checking that a transaction simply failed. If you wanted to check for an exact error code, you could use StarkNet's [error_codes module](https://github.com/starkware-libs/cairo-lang/blob/ed6cf8d6cec50a6ad95fa36d1eb4a7f48538019e/src/starkware/starknet/definitions/error_codes.py) and implement additional logic to the `assert_revert` method.

 To successfully use this wrapper, the transaction method should be wrapped with `assert_revert`; however, `await` should precede the wrapper itself like this:

```python
await assert_revert(signer.send_transaction(
    account, contract.contract_address, 'foo', [
        recipient,
        *token
    ])
)
```

This wrapper also includes the option to check that an error message was included in the reversion. To check that the reversion sends the correct error message, add the `reverted_with` keyword argument outside of the actual transaction (but still inside the wrapper) like this:

```python
await assert_revert(signer.send_transaction(
    account, contract.contract_address, 'foo', [
        recipient,
        *token
    ]),
    reverted_with="insert error message here"
)
```

### `assert_revert_entry_point`

An extension of `assert_revert` that asserts an entry point error occurs with the given `invalid_selector` parameter. This assertion is especially useful in checking proxy/implementation contracts. To use `assert_revert_entry_point`:

```python
await assert_revert_entry_point(
    signer.send_transaction(
        account, contract.contract_address, 'nonexistent_selector', []
    ),
    invalid_selector='nonexistent_selector'
)
```

### `assert_event_emitted`

A helper method that checks a transaction receipt for the contract emitting the event (`from_address`), the emitted event itself (`name`), and the arguments emitted (`data`). To use `assert_event_emitted`:

```python
# capture the tx receipt
tx_exec_info = await signer.send_transaction(
    account, contract.contract_address, 'foo', [
        recipient,
        *token
    ])

# insert arguments to assert
assert_event_emitted(
    tx_exec_info,
    from_address=contract.contract_address,
    name='Foo_emitted',
    data=[
        account.contract_address,
        recipient,
        *token
    ]
)
```

## Memoization

Memoizing functions allow for quicker and computationally cheaper calculations which is immensely beneficial while testing smart contracts.

### `get_contract_class`

A helper method that returns the contract class from the contract's name. To capture the contract class, simply add the contract's name as an argument like this:

```python
contract_class = get_contract_class('ContractName')
```

If multiple contracts exist with the same name, then the contract's path must be passed along with the `is_path` flag instead of the name. To pass the contract's path:

```python
contract_class = get_contract_class('path/to/Contract.cairo', is_path=True)
```

### `cached_contract`

A helper method that returns the cached state of a given contract. It's recommended to first deploy all the relevant contracts before caching the state. The requisite contracts in the testing module should each be instantiated with `cached_contract` in a fixture after the state has been copied. The memoization pattern with `cached_contract` should look something like this:

```python
# get contract classes
@pytest.fixture(scope='module')
def contract_classes():
  foo_cls = get_contract_class('Foo')
  return foo_cls

# deploy contracts
@pytest.fixture(scope='module')
async def foo_init(contract_classes):
    foo_cls = contract_classes
    starknet = await Starknet.empty()
    foo = await starknet.deploy(
        contract_class=foo_cls,
        constructor_calldata=[]
    )
    return starknet.state, foo  # return state and all deployed contracts

# memoization
@pytest.fixture(scope='module')
def foo_factory(contract_classes, foo_init):
    foo_cls = contract_classes                          # contract classes
    state, foo = foo_init                               # state and deployed contracts
    _state = state.copy()                               # copy the state
    cached_foo = cached_contract(_state, foo_cls, foo)  # cache contracts
    return cached_foo                                   # return cached contracts
```

## MockSigner

`MockSigner` is used to perform transactions with an instance of [Nile's Signer](https://github.com/OpenZeppelin/nile/blob/main/src/nile/signer.py) on a given Account, crafting the transaction and managing nonces. The `Signer` instance manages signatures and is leveraged by `MockSigner` to operate with the Account contract's `__execute__` method. See [MockSigner utility](../docs/Account.md#mocksigner-utility) for more information.
