
<a name="0xc0deb00c_order_id"></a>

# Module `0xc0deb00c::order_id`


<a name="@Bit_structure_0"></a>

## Bit structure


An order id is a 128-bit number, where the most-significant
("first") 64 bits indicate the price of the order, regardless of
whether it is an ask or bid. The least-significant ("last") 64 bits
are derived from a serial ID for the market on which the order is
placed, or more specifically, from the corresponding
<code>econia::market::OrderBook.counter</code>. This encoded serial ID is
unmodified in the case of an ask, but has each bit flipped in the
case of a bid.


<a name="@Example_ask_1"></a>

### Example ask


For a scaled integer price of <code>255</code> (<code>0b11111111</code>) and a serial ID
of <code>170</code> (<code>0b10101010</code>), an ask would have an order ID with the
first 64 bits
<code>0000000000000000000000000000000000000000000000000000000011111111</code>
and the last 64 bits
<code>0000000000000000000000000000000000000000000000000000000010101010</code>,
corresponding to the base-10 integer <code>4703919738795935662250</code>


<a name="@Example_bid_2"></a>

### Example bid


For a scaled integer price of <code>15</code> (<code>0b1111</code>) and a serial ID <code>63</code>
(<code>0b111111</code>), a bid would have an order ID with the first 64 bits
<code>0000000000000000000000000000000000000000000000000000000000001111</code>
and the last 64 bits
<code>1111111111111111111111111111111111111111111111111111111111000000</code>,
corresponding to the base-10 integer <code>295147905179352825792</code>


<a name="@Motivations_3"></a>

## Motivations


Positions in an order book are represented as outer nodes in an
<code>econia::critbit::CritBitTree</code>, which allows for traversal across
nodes during the matching process.


<a name="@Market_buy_example_4"></a>

### Market buy example


In the case of a market buy, the matching engine first fills against
the oldest ask at the lowest price, then fills against the second
oldest ask at the lowest price (if there is one). The process
continues, prioritizing older positions, until the price level has
been exhausted, at which point the matching engine moves onto the
next-lowest price level, similarly filling against positions in
chronological priority.

Here, with the first 64 bits of the order ID corresponding to price
and the last 64 bits corresponding to a serial ID, asks are
automatically sorted, upon insertion to the tree, into price-time
priority: first ascending from lowest price to highest price, then
ascending from lowest serial ID to highest serial ID within a price
level. All the matching engine must do is iterate through inorder
successor traversals until the market buy has been filled.


<a name="@Market_sell_example_5"></a>

### Market sell example


In the case of a market sell, the ordering of prices is reversed,
but the price-time priority is not: first the matching engine should
fill against bids at the highest price level, starting with the
oldest position, then fill against newer positions, before moving
onto the next price level. Hence, the final 64 bits of the order ID
are all flipped, because this allows the matching engine to simply
iterate through inorder predecessor traversals until the market buy
has been filled.

More specifically, by flipping the final 64 bits, order IDs from
lower serial IDs numbers are sorted above those from higher serial
IDs, within a given price level: at a scaled integer price of
<code>1</code> (<code>0b1</code>), an order with serial ID <code>15</code> (<code>0b1111</code>) has an order
ID with bits
<code>11111111111111111111111111111111111111111111111111111111111110000</code>,
corresponding to the base-10 integer <code>36893488147419103216</code>, while
an order at the same price with serial ID <code>63</code> (<code>0b111111</code>) has an
order ID with bits
<code>11111111111111111111111111111111111111111111111111111111111000000</code>,
corresponding to the base-10 integer <code>36893488147419103168</code>. The
order with the serial ID <code>63</code> thus has an order ID of lesser value
than that of the order with serial ID <code>15</code>, and as such, during
the matching engine's iterated inorder predecessor traversal, the
order with serial ID <code>63</code> will be filled second.


-  [Bit structure](#@Bit_structure_0)
    -  [Example ask](#@Example_ask_1)
    -  [Example bid](#@Example_bid_2)
-  [Motivations](#@Motivations_3)
    -  [Market buy example](#@Market_buy_example_4)
    -  [Market sell example](#@Market_sell_example_5)
-  [Constants](#@Constants_6)
-  [Function `order_id`](#0xc0deb00c_order_id_order_id)
-  [Function `order_id_ask`](#0xc0deb00c_order_id_order_id_ask)
-  [Function `order_id_bid`](#0xc0deb00c_order_id_order_id_bid)
-  [Function `price`](#0xc0deb00c_order_id_price)
-  [Function `serial_id_ask`](#0xc0deb00c_order_id_serial_id_ask)
-  [Function `serial_id_bid`](#0xc0deb00c_order_id_serial_id_bid)


<pre><code></code></pre>



<a name="@Constants_6"></a>

## Constants


<a name="0xc0deb00c_order_id_HI_64"></a>

<code>u64</code> bitmask with all bits set


<pre><code><b>const</b> <a href="order_id.md#0xc0deb00c_order_id_HI_64">HI_64</a>: u64 = 18446744073709551615;
</code></pre>



<a name="0xc0deb00c_order_id_ASK"></a>

Ask flag


<pre><code><b>const</b> <a href="order_id.md#0xc0deb00c_order_id_ASK">ASK</a>: bool = <b>true</b>;
</code></pre>



<a name="0xc0deb00c_order_id_BID"></a>

Bid flag


<pre><code><b>const</b> <a href="order_id.md#0xc0deb00c_order_id_BID">BID</a>: bool = <b>false</b>;
</code></pre>



<a name="0xc0deb00c_order_id_FIRST_64"></a>

Positions to bitshift for operating on first 64 bits


<pre><code><b>const</b> <a href="order_id.md#0xc0deb00c_order_id_FIRST_64">FIRST_64</a>: u8 = 64;
</code></pre>



<a name="0xc0deb00c_order_id_order_id"></a>

## Function `order_id`

Return order ID for <code>price</code> and <code>serial_id</code> on given <code>side</code>


<pre><code><b>public</b> <b>fun</b> <a href="order_id.md#0xc0deb00c_order_id">order_id</a>(price: u64, serial_id: u64, side: bool): u128
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> <b>fun</b> <a href="order_id.md#0xc0deb00c_order_id">order_id</a>(
    price: u64,
    serial_id: u64,
    side: bool
): u128 {
    // Return corresponding order ID type based on side
    <b>if</b> (side == <a href="order_id.md#0xc0deb00c_order_id_ASK">ASK</a>) <a href="order_id.md#0xc0deb00c_order_id_order_id_ask">order_id_ask</a>(price, serial_id) <b>else</b>
        <a href="order_id.md#0xc0deb00c_order_id_order_id_bid">order_id_bid</a>(price, serial_id)
}
</code></pre>



</details>

<a name="0xc0deb00c_order_id_order_id_ask"></a>

## Function `order_id_ask`

Return order ID for ask with <code>price</code> and <code>serial_id</code>


<pre><code><b>public</b> <b>fun</b> <a href="order_id.md#0xc0deb00c_order_id_order_id_ask">order_id_ask</a>(price: u64, serial_id: u64): u128
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> <b>fun</b> <a href="order_id.md#0xc0deb00c_order_id_order_id_ask">order_id_ask</a>(
    price: u64,
    serial_id: u64
): u128 {
    (price <b>as</b> u128) &lt;&lt; <a href="order_id.md#0xc0deb00c_order_id_FIRST_64">FIRST_64</a> | (serial_id <b>as</b> u128)
}
</code></pre>



</details>

<a name="0xc0deb00c_order_id_order_id_bid"></a>

## Function `order_id_bid`

Return order ID for bid with <code>price</code> and <code>serial_id</code>


<pre><code><b>public</b> <b>fun</b> <a href="order_id.md#0xc0deb00c_order_id_order_id_bid">order_id_bid</a>(price: u64, serial_id: u64): u128
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> <b>fun</b> <a href="order_id.md#0xc0deb00c_order_id_order_id_bid">order_id_bid</a>(
    price: u64,
    serial_id: u64
): u128 {
    (price <b>as</b> u128) &lt;&lt; <a href="order_id.md#0xc0deb00c_order_id_FIRST_64">FIRST_64</a> | (serial_id ^ <a href="order_id.md#0xc0deb00c_order_id_HI_64">HI_64</a> <b>as</b> u128)
}
</code></pre>



</details>

<a name="0xc0deb00c_order_id_price"></a>

## Function `price`

Return price of given <code><a href="order_id.md#0xc0deb00c_order_id">order_id</a></code>, (works for ask or bid)


<pre><code><b>public</b> <b>fun</b> <a href="order_id.md#0xc0deb00c_order_id_price">price</a>(<a href="order_id.md#0xc0deb00c_order_id">order_id</a>: u128): u64
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> <b>fun</b> <a href="order_id.md#0xc0deb00c_order_id_price">price</a>(<a href="order_id.md#0xc0deb00c_order_id">order_id</a>: u128): u64 {(<a href="order_id.md#0xc0deb00c_order_id">order_id</a> &gt;&gt; <a href="order_id.md#0xc0deb00c_order_id_FIRST_64">FIRST_64</a> <b>as</b> u64)}
</code></pre>



</details>

<a name="0xc0deb00c_order_id_serial_id_ask"></a>

## Function `serial_id_ask`

Return serial ID of an ask having <code><a href="order_id.md#0xc0deb00c_order_id">order_id</a></code>


<pre><code><b>public</b> <b>fun</b> <a href="order_id.md#0xc0deb00c_order_id_serial_id_ask">serial_id_ask</a>(<a href="order_id.md#0xc0deb00c_order_id">order_id</a>: u128): u64
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> <b>fun</b> <a href="order_id.md#0xc0deb00c_order_id_serial_id_ask">serial_id_ask</a>(
    <a href="order_id.md#0xc0deb00c_order_id">order_id</a>: u128
): u64 {
    (<a href="order_id.md#0xc0deb00c_order_id">order_id</a> & (<a href="order_id.md#0xc0deb00c_order_id_HI_64">HI_64</a> <b>as</b> u128) <b>as</b> u64)
}
</code></pre>



</details>

<a name="0xc0deb00c_order_id_serial_id_bid"></a>

## Function `serial_id_bid`

Return serial ID of a bid having <code><a href="order_id.md#0xc0deb00c_order_id">order_id</a></code>


<pre><code><b>public</b> <b>fun</b> <a href="order_id.md#0xc0deb00c_order_id_serial_id_bid">serial_id_bid</a>(<a href="order_id.md#0xc0deb00c_order_id">order_id</a>: u128): u64
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> <b>fun</b> <a href="order_id.md#0xc0deb00c_order_id_serial_id_bid">serial_id_bid</a>(
    <a href="order_id.md#0xc0deb00c_order_id">order_id</a>: u128
): u64 {
    (<a href="order_id.md#0xc0deb00c_order_id">order_id</a> & (<a href="order_id.md#0xc0deb00c_order_id_HI_64">HI_64</a> <b>as</b> u128) <b>as</b> u64) ^ <a href="order_id.md#0xc0deb00c_order_id_HI_64">HI_64</a>
}
</code></pre>



</details>
