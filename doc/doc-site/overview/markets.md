# Market structure

- [Market structure](#market-structure)
  - [Trading pairs](#trading-pairs)
  - [Scaling](#scaling)
  - [Markets](#markets)
  - [Dynamic scaling](#dynamic-scaling)
  - [Social effects](#social-effects)

## Trading pairs

Assets in Econia are represented as `AptosFramework::Coin::Coin<phantom CoinType>`, thus having a `u64` value and associated `CoinInfo` metadata, including `decimals` and `symbol` fields.
Using terminology inherited from Forex markets, a "trading pair" is thus defined as a "base coin" denominated in terms of a "quote coin", for instance `FOO/BAR` denoting `FOO` denominated in `BAR`:
A `FOO/BAR` "price" of `12.34` means that one `FOO` costs 12.34 `BAR`.

## Scaling

Notably, in the above example values are listed as decimal amounts, which is what a user would probably see on a front-end web interface.
In Move, however, `Coin` types are ultimately represented as integers.
More specifically, if a front-end user were to trade 1 `FOO` for 12.34 `BAR`, they would actually be trading a `Coin<FOO>` of `value` 1000 for a `Coin<BAR>` of `value` 1234000000, or perhaps a `Coin<FOO>` of `value` 1000000000 for a `Coin<BAR>` of `value` 123400, depending on the actual `decimals` field defined in each `Coin`'s respective `CoinInfo`.

Since Econia's matching engine operates on the underlying integer values, not decimals, granularity problems can arise when a trading pair involves two assets with disparate valuations relative to one another, because the matching engine similarly denotes price as an integer.

Econia thus implements a "scaled price", formally defined as the number of indivisible quote coin subunits (`Coin<BAR>.value`) per `SF` base coin indivisible subunits (`Coin<FOO>.value`), with `SF` denoting scale factor.
For example:

* Scale factor of 100
    * A user submits a bid to buy a `Coin<FOO>` of `value` 1200 at a scaled price of 34.
    * The scaled size of the order is 12 (`1200 / 100 = 12`), and it takes a `Coin<BAR>` of `value` 408 (`12 * 34 = 408`) to fill the order

* Scale factor of 1
    * A user submits a bid to buy a `Coin<FOO>` of `value` 123 at a scaled price of 4.
    * The scaled size of the order is 123 (`123 / 1 = 123`), and it takes a `Coin<BAR>` of `value` 492 (`123 * 4 = 492`) to fill the order.

Alternatively, consider the scale factor as denoting the number of indivisible subunits in a "parcel" of base coins, with scaled price denoting the number of quote coin subunits per parcel:
At a scale factor of 100, a user can only transact in parcels of 100 base coin (`Coin<FOO>.value = 100`), and multiplying the number of parcels by scaled price yields the number of quote coins (`Coin<BAR>.value`) on the other side of the trade:

* Scale factor of 10
    * A user submits a bid to buy a `Coin<FOO>` of `value` 120 at a price of 3.
    * The "scaled size" of the order (number of parcels) is 12 (`120 / 10 = 12`), and it takes a `Coin<BAR>` of `value` 36 (`12 * 3 = 36`) to fill the order
    * The order is stored in memory with a scaled price of 3 and a scaled size 12

In terms of conventional trading semantics, "scale factor" is analogous to lot size, with Econia implementing a fixed tick size of 1 for every trading pair.
In future releases, Econia will likely switch to using these terms exclusively, with configurable lot and tick size scheduled for a [future release](https://github.com/econia-labs/econia/issues/10#issuecomment-1201406918).

## Markets

In Econia, a "market" is uniquely specified by 3 type arguments, having the canonical syntax `<B, Q, E>`:

* `B`: Base coin type (`FOO` in above example)
* `Q`: Quote coin type (`BAR` in above example),
* `E`: "Scale exponent", denoting the base-10 logarithm of the scale factor used for the market:

    | Scale exponent | Scale factor |
    |-|-|
    | 0 | 1 |
    | 1 | 10 |
    | 2 | 100 |
    | 3 | 1000 |
    | 4 | 10000 |
    | ... | ... |

In [`econia::registry`](../../../src/move/econia/sources/registry.move), scale exponents are defined as the types `E0`, `E1`, `E2`, ..., while scale factors are defined as the `u64` values `F0 = 0`, `F1 = 10`, `F2 = 100`, ..., with scale exponent type arguments used to identify markets, and scale factor `u64` values used to perform arithmetic operations pertaining to a given market.

For a user to trade on a market, the market must first be "registered" by a "host", meaning that:

1. An empty order book of type `<B, Q, E>` is initialized under the host's account
2. A registry is updated with an entry mapping from `<B, Q, E>` to the address of the corresponding host

The registry can only be initialized by the Econia account, and the registry must be initialized before hosts can register markets.
A market can only be registered once, meaning that `<FOOAccount::FOOModule::FOO, BARAccount::BARModuleBAR, E0>`, for instance, can only ever be registered to one host, and markets can only be registered using scale exponents defined in [`econia::registry`](../../../src/move/econia/sources/registry.move) (e.g. `E0` actually denotes `econia::registry::E0`)
Notably, this does not prevent multiple markets from being initialized using the same trading symbol, because the Aptos VM treats `FOOAccount::FOOModule::FOO` and `ImposterAccount::ImposterModule::FOO` as different types, hence front-end applications are advised to exercise caution accordingly.

## Dynamic scaling

Multiple markets may be registered for a given base coin/quote coin trading pair by simply varying the scale exponent, which means that `<A1::M1::FOO, A2::M2::BAR, E0>` and `<A1::M1::FOO, A2::M2::BAR, E3>` can co-exist in the registry, along with any other such market containing a variation on the scale exponent type.
Essentially this functionality is implemented because:

1. Market registration is permissionless, and
1. Trading granularity varies as asset prices change relative to one another

For example:

Recently-issued protocol token `PRO` has 3 decimal places, a circulating supply of 1 billion (`10 ^ 9`), and a market capitalization denominated in `USDC` (6 decimal places) of $100,000.
Hence the user-facing decimal representation `1.000 PRO` corresponds to `Coin<PRO>.value = 1000`, an amount that trades for `100,000 / 10 ^ 9 = 0.0001 USDC`.
This means that a single indivisible subunit of `PRO` (`0.001 PRO`, `Coin<PRO>.value = 1`) should trade for only `0.0000001 USDC`, an amount that is actually impossible to represent, since `USDC` only has 6 decimal places.
Thus a host registers the market `<PRO, USDC, E3>`, corresponding to a scale factor of `1000`, such that `PRO` can only be transacted in parcels of 1000 indivisible subunits, or `1.000 PRO` at a time.
A user submits a bid to buy `5.000 PRO` (5 parcels) at a scaled price of `99`, for instance, and in return receives `0.000495 USDC` (`Coin<USDC>.value = 495`, `5 * 99 = 495`).

Later, the `USDC`-denominated market capitalization of `PRO` increases to $100B, such that each `1.000 PRO` nominally trades for $100.
Here, a scale factor of `1000` is no longer appropriate, because users are still limited to transacting in parcels of `Coin<PRO>.value = 1000` (`1.000 PRO`), each having a nominal price of $100, such that it is impossible to place a bid to buy `.5 PRO` ($50 nominal).
Hence, to re-establish appropriate trading granularity, a host registers the market `<PRO, USDC, E0>`, corresponding to a scale factor of 1, which enables a user to place a bid for `1.234 PRO` (1234 parcels) at a scaled price of `99999`, receiving `123.398766 USDC` in return (`Coin<USDC>.value = 123398766`, `1234 * 99999 = 123398766`).

## Social effects

Econia's market registration system is permissionless, to the effect that for any given trading pair, there is no official scaling factor.
This is by design, because a permissionless market registration system dispenses with the need for an individual, a panel, or a DAO to dictate others' trading granularity, and instead leaves the decision "up to the market".
Here, the expectation is that as asset prices drastically change relative to one another, liquidity will naturally migrate to the market with the most appropriate granularity, and as spreads thicken on markets with sub-optimal granularity, a positive feedback cycle will eventually lead users and front-end applications to switch over their orders to the new trading venue.
Not that users are necessarily forced to migrate their orders over:
in the above example, if a user wanted to continue placing orders on `<PRO, USDC, E3>` even when 99% of liquidity had fled to `<PRO, USDC, E0>`, there would ultimately be nothing stopping them from placing orders on the outdated venue, in parcels of `1.000 PRO` at a time.
The uninformed (or perhaps stubborn? idealistic?) user would almost certainly receive a worse price than if they had simply followed the pack and migrated to the new marketplace, but in one sense, it is a form of economic freedom that they are not strictly forced to.