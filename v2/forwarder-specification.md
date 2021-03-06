1.  [Architecture](#architecture)
1.  [Contracts](#contracts)
    1.  [Forwarder](#forwarder)
1.  [Contract Interactions](#contract-interactions)
    1.  [Fee Abstraction](#fee-abstraction)
1.  [Events](#events)
    1.  [Forwarder events](#forwarder-events)
1.  [Miscellaneous](#miscellaneous)
    1.  [Optimizing calldata](#optimizing-calldata)

# Architecture

<div style="text-align: center;">
<img src="../img/0x_forwarder.png" style="padding-bottom: 20px; padding-top: 20px;" width="80%" />
</div>

The Forwarding contract acts as a middleman between the user and the 0x Exchange contract. Its purpose is to perform a number of useful actions on the users behalf. Conveniently reducing the number of steps and transactions.

The 0x protocol [AssetProxy](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#assetproxy) contracts work as operators that are authorized to move tokens on behalf of users if an order with a valid signature can be produced. Since ETH is not a token and cannot be moved natively by a third party operator, Wrapped ETH (WETH) is introduced to adapt ETH to work in this environment. WETH requires a user to deposit ETH into the WETH contract (transaction) and approve (transaction) the AssetProxy operator to transfer the WETH on the user's behalf.

When performing an exchange on 0x, fees may be required to be paid by the maker and takers in ZRX token. The AssetProxy needs approval to move these tokens (transaction) and ZRX tokens need to be pre-purchased (transaction).

A user is then able to find an order and perform a TokenA/WETH `fillOrder` on the 0x [Exchange contract](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#exchange) (transaction).

For first time purchasers of tokens, this is non-trivial setup that is unavoidable due to the design of the ERC20 standard. The Forwarding contracts purpose is to enable users to buy tokens in one transaction.

# Contracts

## Forwarder

The Forwarder contains all of the following logic:

- Presets approval to the [ERC20Proxy](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#erc20proxy) and the [ERC721Proxy](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#erc721proxy)
- Converts ETH into WETH on behalf of the user
- Performs the exchange via the 0x Exchange contract
- Purchases ZRX on behalf of the user (ZRX fee abstraction)
- Transfers the newly purchased asset to the user

There are two public entry points for users of the Forwarding contract, `marketSellOrdersWithEth` and `marketBuyOrdersWithEth`.

The implementation of these methods follows a 4 step process:

1.  Convert ETH to WETH.
2.  Fill the orders using a `marketBuy` or `marketSell`. Note that any ZRX fees will be paid using the Forwarder contract's own ZRX balance.
3.  If any ZRX fees were paid, repurchase that amount of ZRX.
4.  If `feePercentage` and `feeRecipient` are supplied deduct the fees from the amount traded.
5.  Transfer any remaining ETH back to sender.

### marketBuyOrdersWithEth

This function buys a specific amount of assets (ERC20 or ERC721) given a set of TokenA/WETH orders. If fees are required on the orders, then `feeOrders` of ZRX/WETH are supplied and this is used to purchase the fees required. The user specifies the amount of assets they wish to buy with the `makerAssetFillAmount` parameter. `makerAssetFillAmount` must equal 1 if buying unique ERC721 tokens. The `makerAsset` of all supplied `orders` should be equivalent - any orders that contain `makerAssetData` different than the first order supplied will be ignored (gas will still be paid for attempting to fill these orders).

This function will revert under the following conditions:

- No ETH value was included in the transaction
- The `makerAssetData` contains an [`assetProxyId`](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#assetdata) that is not supported (currently ERC20 and ERC721 only)
- The entire `makerAssetFillAmount` cannot be filled
- The amount of ZRX paid in fees cannot be repurchased
- The supplied `feePercentage` is greater than 5% (represented as 0.05 \* 10^18)
- The required ETH fee cannot be paid

```solidity
    /// @dev Attempt to purchase makerAssetFillAmount of makerAsset by selling ETH provided with transaction.
    ///      Any ZRX required to pay fees for primary orders will automatically be purchased by this contract.
    ///      Any ETH not spent will be refunded to sender.
    /// @param orders Array of order specifications used containing desired makerAsset and WETH as takerAsset.
    /// @param makerAssetFillAmount Desired amount of makerAsset to purchase.
    /// @param signatures Proofs that orders have been created by makers.
    /// @param feeOrders Array of order specifications containing ZRX as makerAsset and WETH as takerAsset. Used to purchase ZRX for primary order fees.
    /// @param feeSignatures Proofs that feeOrders have been created by makers.
    /// @param feePercentage Percentage of WETH sold that will payed as fee to forwarding contract feeRecipient.
    /// @param feeRecipient Address that will receive ETH when orders are filled.
    /// @return Amounts filled and fees paid by maker and taker for both sets of orders.
    function marketBuyOrdersWithEth(
        LibOrder.Order[] memory orders,
        uint256 makerAssetFillAmount,
        bytes[] memory signatures,
        LibOrder.Order[] memory feeOrders,
        bytes[] memory feeSignatures,
        uint256  feePercentage,
        address feeRecipient
    )
        public
        payable
        returns (
            LibFillResults.FillResults memory orderFillResults,
            LibFillResults.FillResults memory feeOrderFillResults
        );
```

As filling these orders is a moving target (a partial fill can occur while this transaction is pending) it is safe to send in additional orders and additional ETH. The `makerAssetFillAmount` is the total amount of assets the user wishes to purchase, and any additional ETH is returned to the user. This allows for the possibility of safely sending in 100 orders and 100 ETH to purchase 1 token, with all unspent ETH being returned. Any unused orders will increase the gas cost of the transaction due to additional calldata that must be stored in the blockchain's history.

#### marketBuyOrdersWithEth when ZRX is the makerAsset

Buying an exact amount of ZRX assets is a special case as fees (paid in ZRX) need to be accounted for. If a user wishes to purchase 100 ZRX and there is a 2 ZRX fee on this order, the Forwarding contract guarantees they will receive the expected amount (100 ZRX). To accomplish this we calculate an adjusted exchange rate (taking into account the fees). There is no need to provide any `feeOrders` since fees can be purchased from the same pool of orders. In this scenario the caller can supply an empty fee orders and fee signatures array.

### marketSellOrdersWithEth

This function attempts to buy as many ERC20 tokens as possible given the amount of ETH sent in by performing a `marketSell`. If fees are required on the orders, then `feeOrders` of ZRX/WETH are supplied and this is used to purchase the fees required. 5% of the ETH included in the transaction are reserved for paying fees (i.e the sum of the value of the fees denominated in ZRX and ETH). The `makerAsset` of all supplied `orders` should be equivalent - any orders that contain `makerAssetData` different than the first order supplied will be ignored (gas will still be paid for attempting to fill these orders).

This function will revert under the following conditions:

- No ETH value was included in the transaction
- The `makerAssetData` contains an [`assetProxyId`](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#assetdata) that is not supported (currently ERC20 and ERC721 only)
- The amount of ZRX paid in fees cannot be repurchased
- The supplied `feePercentage` is greater than 5% (represented as 0.05 \* 10^18)
- The required ETH fee cannot be paid
- More ETH is sold than the value included in the transaction

```solidity
    /// @dev Purchases as much of orders' makerAssets as possible by selling up to 95% of transaction's ETH value.
    ///      Any ZRX required to pay fees for primary orders will automatically be purchased by this contract.
    ///      5% of ETH value is reserved for paying fees to order feeRecipients (in ZRX) and forwarding contract feeRecipient (in ETH).
    ///      Any ETH not spent will be refunded to sender.
    /// @param orders Array of order specifications used containing desired makerAsset and WETH as takerAsset.
    /// @param signatures Proofs that orders have been created by makers.
    /// @param feeOrders Array of order specifications containing ZRX as makerAsset and WETH as takerAsset. Used to purchase ZRX for primary order fees.
    /// @param feeSignatures Proofs that feeOrders have been created by makers.
    /// @param feePercentage Percentage of WETH sold that will payed as fee to forwarding contract feeRecipient.
    /// @param feeRecipient Address that will receive ETH when orders are filled.
    /// @return Amounts filled and fees paid by maker and taker for both sets of orders.
    function marketSellOrdersWithEth(
        LibOrder.Order[] memory orders,
        bytes[] memory signatures,
        LibOrder.Order[] memory feeOrders,
        bytes[] memory feeSignatures,
        uint256  feePercentage,
        address feeRecipient
    )
        public
        payable
        returns (
            LibFillResults.FillResults memory orderFillResults,
            LibFillResults.FillResults memory feeOrderFillResults
        );
```

#### marketSellOrdersWithEth when ZRX is the makerAsset

Similar to `marketBuyOrdersWithEth`, there is no need to provide `feeOrders` when the `makerAsset` of the primary orders is ZRX. ZRX fees can be baked into the price of the orders and withheld when transferring the purchased ZRX back to the sender.

### feePercentage and feeRecipient

The `feePercentage` and `feeRecipient` are a feature in the Forwarding contract which is distinct from the ZRX fees of the orders. This incentivizes integrations from wallets and other dApps who are not the feeRecipients in these orders to host front ends for the Forwarder contract and earn revenue. The `feePercentage` is calculated off the total value of ETH traded, not the total amount of ETH sent to the contract when calling the function.

`feePercentage` supports up to 18 decimial places, for example 0.59% is represented as 0.0059 \* 10^18. This value is limited to less than or equal to 5%.

# Miscellaneous

## Optimizing calldata

Calldata is expensive. As per Appendix G of the [ETHeum Yellowpaper](#https://ETHeum.github.io/yellowpaper/paper.pdf), every non-zero byte of calldata costs 68 gas, and every zero byte costs 4 gas. There are certain off-chain optimizations that can be made in order to maximize the amount of zeroes included in calldata.

### Assuming order parameters

The Forwarding contract assumes that:

- `takerAssetData` for both `orders` and `feeOrders` refers to WETH
- `makerAssetData` for `orders` after the first order are equivalent to the `makerAssetData` of the first order (i.e `orders[1].makerAssetData == orders[0].makerAssetData`)
- `makerAssetData` for `feeOrders` refers to ZRX

This means users may pass in zero values for these parameters and the functions will still execute as if the values had been passed in as calldata.

# Events

There are no additional events emitted by the Forwarding contract. However, `Transfer` (ERC20 and ERC721), `Deposit` and `Withdraw` (WETH), and `Fill` (0x Exchange) are emitted in their respective contracts.

## Known Issues

### Over buying ZRX

In order to avoid any rounding issues when performing ZRX fee abstraction, an additional 1 wei worth of ZRX is purchased per `feeOrder` that is filled. In certain scenarios this is left behind in the contract. It is unlikely this would ever accrue to a value worth extracting and is not worth the gas in transferring per trade.

### feePercentage and feeRecipient modifications

These values are submitted by the user and can therefore be modified. After talking with many projects, we believe this is acceptable for multiple reasons.

1.  It is possible for a wallet to copy this contract and hard code these values.
2.  If a wallet is the source of liquidity, they can filter orders by `senderAddress` or `takerAddress`.
3.  It is cheaper for an "attacker" to avoid the Forwarding contract entirely and transact directly with the 0x Exchange.
