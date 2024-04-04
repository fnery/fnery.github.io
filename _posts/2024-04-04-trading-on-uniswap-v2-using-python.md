---
layout: post
title:  "Trading on Uniswap V3 using Python"
date: 2024-04-04 11:02:38 +0100
tags: [blockchains]
---

## Introduction

Let's implement a simple[^1] [Python](https://www.python.org/) script using [web3.py](https://web3py.readthedocs.io/en/stable/) that demonstrates how to swap [ERC-20](https://ethereum.org/en/developers/docs/standards/tokens/erc-20/) tokens using [Uniswap V3](https://blog.uniswap.org/uniswap-v3).

We'll do this on the [Sepolia](https://sepolia.dev/) network. This is one of many [Ethereum Testnets](https://ethereum.org/en/developers/docs/networks/#ethereum-testnets), networks created for testing/development which follow the same protocol rules as the Ethereum mainnet. This way, we avoid having to use any real funds to follow this tutorial.

Specifically, we'll buy 0.01 units of token [TOK](https://sepolia.etherscan.io/address/0x005E562D1F58D94C3EE79dC02084039Bd0Db35bE) using [WETH](https://sepolia.etherscan.io/address/0xfFf9976782d46CC05630D1f6eBAb18b2324d6B14).

Here's a summary of what we'll do:

1. Fund a fresh Sepolia wallet with WETH
2. Approve spending of some of our WETH
3. Spend WETH to buy TOK

## Setting up a development environment

> A GitHub repo containing the full code for this tutorial can be found [here](https://github.com/fnery/uniswap-v3-swap-demo).
{: .prompt-tip }

First, install the Python requirements specified in [requirements.txt](https://github.com/fnery/uniswap-v3-swap-demo/blob/main/requirements.txt).

We'll need access to a Sepolia testnet node. Using a 3rd party node (from services like [Infura](https://www.infura.io/) or [Alchemy](https://www.alchemy.com/)) is a straightforward option[^2]. This provides us with an endpoint to Sepolia's API. The figure below shows where you'd get this endpoint if using Infura.

![sepolia_endpoint](/assets/img/sepolia_endpoint.png){: width="500" }
*Grabbing an endpoint URL from Infura*

Finally we'll need to grab the account address *and* private key of our Sepolia account. If you need a primer on how to get an Ethereum account, check [this](https://ethereum.org/en/guides/how-to-create-an-ethereum-account/). If you're using [Metamask](https://metamask.io/), [here's](https://moralis.io/how-to-add-the-sepolia-network-to-metamask-full-guide/) a guide to add the Sepolia testnet to your wallet and [instructions](https://support.metamask.io/hc/en-us/articles/360015289632-How-to-export-an-account-s-private-key) to export private keys.

Next, we'll add everything to an `.env` file. Assuming we cloned or forked the [fnery/uniswap-v3-swap-demo](https://github.com/fnery/uniswap-v3-swap-demo) repo, we'll create the `.env` file in its root directory (where the [main.py](https://github.com/fnery/uniswap-v3-swap-demo/blob/main/main.py) file lives). It should look like this:

```
PROVIDER=https://sepolia.infura.io/v3/046c4fb87c1347c4acee712d6dc561b5
ADDRESS=0x0FFB...
PRIVATE_KEY=0c0aab...
```
{: file=".env" }

> The above contains sensitive information which should not be shared or pushed to a GitHub repo. Adding this info to a `.env` file (in combination with our [.gitignore](https://github.com/fnery/uniswap-v3-swap-demo/blob/main/.gitignore) file) prevents us from doing so.
{: .prompt-warning }

## Funding a wallet with WETH

First, we'll get test ETH from a faucet (such as [this](https://sepolia-faucet.pk910.de/) one). [Here's](https://sepolia.etherscan.io/tx/0x76fa9ca96528abb6d38a7265a7282f9d6d83bd0858ef746f01577def3400a37c) the resulting transaction. Then, we deposit some of the ETH into the WETH [contract](https://sepolia.etherscan.io/address/0xfff9976782d46cc05630d1f6ebab18b2324d6b14). An easy way to achieve this is by using Etherscan's [write contract](https://info.etherscan.com/how-to-use-read-or-write-contract-features-on-etherscan/) feature (which requires you to connect your account), as demonstrated in the figure below.

![deposit_eth](/assets/img/deposit_eth.png){: width="650" }
*Depositing ETH into the WETH [contract](https://sepolia.etherscan.io/token/0xfff9976782d46cc05630d1f6ebab18b2324d6b14#writeContract). Resulting [transaction](https://sepolia.etherscan.io/tx/0x0104443428eec375ef8a864b37aa8d753787fa64587334f0eee80d4a9cef0a04)*

Once the transaction is confirmed, we should have 0.02 WETH tokens in our wallet. We're now ready to implement the code for swapping [WETH](https://sepolia.etherscan.io/address/0xfFf9976782d46CC05630D1f6eBAb18b2324d6B14) for [TOK](https://sepolia.etherscan.io/address/0x005E562D1F58D94C3EE79dC02084039Bd0Db35bE).

## Implementing the swap

### Swap preview on the Uniswap UI

Before starting the implementation, let's navigate to the [Uniswap UI](https://app.uniswap.org/swap), connect our Sepolia account, and preview the swap.

<a id="swap_uniswap_figure" href="#swap_uniswap_figure"></a>
![swap_uniswap](/assets/img/swap_uniswap.png){: width="500"}
*Previewing the swap on the Uniswap UI*

In summary:

- We'll buy 0.01 units of TOKEN.
- We expect to pay 0.000226784 WETH.
- Our trade will have a [price impact](https://docs.uniswap.org/concepts/protocol/swaps#price-impact) of ~1.5%.
- The trade will only execute if the [slippage](https://docs.uniswap.org/concepts/protocol/swaps#slippage) does not exceed 10%.

### Implementation

First, let's add our imports and constants.

```python
import os
from web3 import Web3
from dotenv import load_dotenv

# User parameters
OUT_AMOUNT_READABLE = 0.01
POOL_ADDRESS = "0x41E3F1A4F715A5C05b1B90144902db17CA91BF5c"
MAX_SLIPPAGE = 0.1

# Other constants
load_dotenv()
PRIVATE_KEY = os.getenv("PRIVATE_KEY")
PROVIDER = os.getenv("PROVIDER")
WALLET_ADDRESS = os.getenv("ADDRESS")

POOL_ABI_PATH = "UniswapV3PoolABI.json"
QUOTER_ADDRESS = "0xEd1f6473345F45b75F8179591dd5bA1888cf2FB3"
QUOTER_ABI_PATH = "UniswapV3QuoterV2ABI.json"
ROUTER_ADDRESS = "0x3bFA4769FB09eefC5a80d6E87c3B9C650f7Ae48E"
ROUTER_ABI_PATH = "UniswapV3SwapRouter02ABI.json"
ERC20_ABI_PATH = "ERC20ABI.json"
```

The main swap parameters are set by the `OUT_AMOUNT_READABLE` (the amount of `token0` we'll purchase), `POOL_ADDRESS` (pool we'll swap against) and `MAX_SLIPPAGE` (0.1 for a maximum allowable slippage of 10%).

> Note that to keep the Python script as simple as possible, we confirmed[^3] that [TOK](https://sepolia.etherscan.io/address/0x005E562D1F58D94C3EE79dC02084039Bd0Db35bE) (the token we want to purchase) is the pool's `token0`.
{: .prompt-tip }

Here, we also read the variables on our `.env` file and initialize a few required Uniswap contract [addresses](https://docs.uniswap.org/contracts/v3/reference/deployments) and paths to their [Application Binary Interface](https://docs.soliditylang.org/en/develop/abi-spec.html) files (ABIs), which we got from Etherscan ([example](https://api-sepolia.etherscan.io/api?module=contract&action=getabi&address=0x287B0e934ed0439E2a7b1d5F0FC25eA2c24b64f7&format=raw)). All the ABI files can be found [here](https://github.com/fnery/uniswap-v3-swap-demo).

Let's establish a connection with the node provider:

```python
web3 = Web3(Web3.HTTPProvider(PROVIDER))
```

We'll initialize the pool contract and read some of its data (stored on the blockchain): the pool `fee` and `sqrtPriceX96` (we'll get the price of TOK in terms of WETH from the latter[^4]).

```python
pool = web3.eth.contract(address=POOL_ADDRESS, abi=load_abi(POOL_ABI_PATH))
fee = pool.functions.fee().call()
sqrtPriceX96 = pool.functions.slot0().call()[0]
```

Above, `load_abi` is just a simple function to read the ABI files:

```python
def load_abi(abi_path):
    with open(abi_path, "r") as file:
        return file.read()
```

Now let's retrieve and derive data related to the tokens in the pool:

```python
# Assume token going "out" of the pool (being purchased) is token0
out_address = pool.functions.token0().call()
out_contract = web3.eth.contract(address=out_address, abi=load_abi(ERC20_ABI_PATH))
out_symbol = out_contract.functions.symbol().call()
out_decimals = out_contract.functions.decimals().call()
out_amount = int(OUT_AMOUNT_READABLE * 10**out_decimals)
out_price = (sqrtPriceX96 / (2**96)) ** 2  # price of token0 in terms of token1

# Assume token going "in" to the pool (being sold) is token1
in_address = pool.functions.token1().call()
in_contract = web3.eth.contract(address=in_address, abi=load_abi(ERC20_ABI_PATH))
in_symbol = in_contract.functions.symbol().call()
in_decimals = in_contract.functions.decimals().call()
in_amount = int(out_amount * out_price)
in_amount_readable = in_amount / 10**in_decimals
```

For each token, we'll get its address, token symbol and swap amounts. We'll also get the token `decimals`, which we'll use to convert human-readable token amounts (variables with `_readable` suffix) to and from their corresponding amounts used within the smart contract's logic (large integer values)[^5].

The exact price we'll pay for the `TOK` tokens is not exactly what we obtain in `out_price` because our trade will itself [move](https://docs.uniswap.org/concepts/protocol/swaps#price-impact) the prices within the pool. Let's use the `QuoterV2` [contract](https://sepolia.etherscan.io/address/0xEd1f6473345F45b75F8179591dd5bA1888cf2FB3) to get an estimate of the amount of `WETH` we'll have to pay for the desired amount of `TOK` tokens, which will also allow us to estimate the [price impact](https://docs.uniswap.org/concepts/protocol/swaps#price-impact) of our swap.

```python
quoter = web3.eth.contract(address=QUOTER_ADDRESS, abi=load_abi(QUOTER_ABI_PATH))
quote = quoter.functions.quoteExactOutputSingle(
    [
        in_address,
        out_address,
        out_amount,
        fee,
        0,  # sqrtPriceLimitX96: a value of 0 makes this parameter inactive
    ]
).call()

in_amount_expected = quote[0]
in_amount_expected_readable = in_amount_expected / 10**in_decimals

price_impact = (in_amount_expected - in_amount) / in_amount

print(
    f"Expected amount to pay: {in_amount_expected_readable} {in_symbol} (price impact: {price_impact:.2%})"
)
```

Note the last statement, which prints:

> Expected amount to pay: 0.000226784946144417 WETH (price impact: 1.55%)

This is consistent with what we saw in the Uniswap UI in the figure [above](#swap_uniswap_figure).

Now let's specify the maximum `WETH` amount we're willing to pay for the desired `TOK` tokens. We'll derive this from the price of `TOK` before executing the swap and the maximum allowed slippage we defined with `MAX_SLIPPAGE`:

```python
in_amount_max = int(out_amount * out_price * (1 + MAX_SLIPPAGE))
in_amount_max_readable = in_amount_max / 10**in_decimals

print(
    f"Max. amount to pay: {in_amount_max_readable} {in_symbol} (max. slippage: {MAX_SLIPPAGE:.2%})"
)
```

> Max. amount to pay: 0.000245650082067667 WETH (max. slippage: 10.00%)

Therefore, if anything (including factors beyond our own swap[^6]) causes the `TOK` price to change in such a way that we'd have to pay more than ~0.00024 WETH for the desired TOK tokens, the swap will not be executed.

At this stage, only one thing is left to do before we can execute the swap, which is to [approve](https://docs.openzeppelin.com/contracts/2.x/api/token/erc20#IERC20-approve-address-uint256-) spending of our `WETH` tokens by the Uniswap `SwapRouter02` [contract](https://sepolia.etherscan.io/address/0x3bFA4769FB09eefC5a80d6E87c3B9C650f7Ae48E#code) (whose address is given by `ROUTER_ADDRESS`)

```python
# Approval transaction
transaction = in_contract.functions.approve(
    ROUTER_ADDRESS, in_amount_max
).build_transaction(
    {
        "chainId": web3.eth.chain_id,
        "gas": int(1e7),
        "gasPrice": web3.eth.gas_price,
        "nonce": web3.eth.get_transaction_count(WALLET_ADDRESS),
    }
)
signed_txn = web3.eth.account.sign_transaction(transaction, private_key=PRIVATE_KEY)
tx_hash = web3.eth.send_raw_transaction(signed_txn.rawTransaction)
tx_receipt = web3.eth.wait_for_transaction_receipt(tx_hash)
```

Above, we call the `approve` method in the contract of the token going `in`to the pool (i.e. `WETH`) to specify that we allow the router contract to spend `WETH` up to the amount given by `in_amount_max`, which accounts for the maximum allowable slippage.

Finally, we can execute the swap:

```python
# Swap transactions
router = web3.eth.contract(address=ROUTER_ADDRESS, abi=load_abi(ROUTER_ABI_PATH))
transaction = router.functions.exactOutputSingle(
    [
        in_address,
        out_address,
        fee,
        WALLET_ADDRESS,
        out_amount,
        in_amount_max,
        0,  # sqrtPriceLimitX96: a value of 0 makes this parameter inactive
    ]
).build_transaction(
    {
        "chainId": web3.eth.chain_id,
        "gas": int(1e7),
        "gasPrice": web3.eth.gas_price,
        "nonce": web3.eth.get_transaction_count(WALLET_ADDRESS),
    }
)
signed_txn = web3.eth.account.sign_transaction(transaction, private_key=PRIVATE_KEY)
tx_hash = web3.eth.send_raw_transaction(signed_txn.rawTransaction)
tx_receipt = web3.eth.wait_for_transaction_receipt(tx_hash)
```

We want to buy (i.e. for the pool to output) an exact amount of `TOK`, so we call the `exactOutputSingle` [method](https://docs.uniswap.org/contracts/v3/reference/periphery/interfaces/ISwapRouter#exactoutputsingle) from the `SwapRouter02` [contract](https://sepolia.etherscan.io/address/0x3bFA4769FB09eefC5a80d6E87c3B9C650f7Ae48E#code). Parameters include the addresses of both tokens involved in the swap and the pool `fee` (these allow the router to determine the pool to use), the address of our wallet, as well as the exact amount `TOK` we'll buy and maximum amount of `WETH` we're willing to pay. For simplicity, the parameter `sqrtPriceLimitX96` is left inactive (not [advisable](https://docs.uniswap.org/contracts/v3/guides/swaps/single-swaps#swap-input-parameters) for production environments).

## Executing the swap

Putting everything together (as well as some helper functions used in the snippets above), we arrive at the final `main.py` script that lives [here](https://github.com/fnery/uniswap-v3-swap-demo/blob/main/main.py).

Let's run it:

```bash
$ python3 main.py
web3 connection successful
Goal: Buy 0.01 TOK (worth 0.000223318256425151 WETH)
Expected amount to pay: 0.000226784946144417 WETH (price impact: 1.55%)
Max. amount to pay: 0.000245650082067667 WETH (max. slippage: 10.00%)
Approval transaction 0xb221cc3ba27da14d5053ed11d82a739b9d69e52950488c30306a7541b26e3580 successful
Swap transaction 0xec95f9a985b82a3dd2195b4256a1a7cb270c655647c635e3acefdebb84789672 successful
```

Two transactions were successfully submitted and executed:

- Approval transaction: [0xb221cc3ba27da14d5053ed11d82a739b9d69e52950488c30306a7541b26e3580](https://sepolia.etherscan.io/tx/0xb221cc3ba27da14d5053ed11d82a739b9d69e52950488c30306a7541b26e3580).
- Swap transaction: [0xec95f9a985b82a3dd2195b4256a1a7cb270c655647c635e3acefdebb84789672](https://sepolia.etherscan.io/tx/0xec95f9a985b82a3dd2195b4256a1a7cb270c655647c635e3acefdebb84789672).

---

[^1]: This is a simple toy implementation and should _not_ be used to trade tokens with any value!
[^2]: If you're curious about running your own node, check [this](https://ethereum.org/en/run-a-node) out.
[^3]: To confirm, call the `token0` method on the pool's contract, e.g. on [Etherscan](https://sepolia.etherscan.io/address/0x41E3F1A4F715A5C05b1B90144902db17CA91BF5c#readContract), which returns the address of [TOK](https://sepolia.etherscan.io/address/0x005E562D1F58D94C3EE79dC02084039Bd0Db35bE).
[^4]: [Here's](https://blog.uniswap.org/uniswap-v3-math-primer) a primer on Uniswap V3 math.
[^5]: More on decimals [here](https://docs.openzeppelin.com/contracts/3.x/erc20#a-note-on-decimals).
[^6]: Onchain activity can change the market conditions within the time between transaction submission and its verification (e.g. [frontrunning](https://scsfg.io/hackers/frontrunning) attacks).
