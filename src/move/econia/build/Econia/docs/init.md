
<a name="0xc0deb00c_init"></a>

# Module `0xc0deb00c::init`

Initializers for core Econia resources.


-  [Constants](#@Constants_0)
-  [Function `init_econia`](#0xc0deb00c_init_init_econia)


<pre><code><b>use</b> <a href="">0x1::signer</a>;
<b>use</b> <a href="market.md#0xc0deb00c_market">0xc0deb00c::market</a>;
<b>use</b> <a href="registry.md#0xc0deb00c_registry">0xc0deb00c::registry</a>;
</code></pre>



<a name="@Constants_0"></a>

## Constants


<a name="0xc0deb00c_init_E_NOT_ECONIA"></a>

When caller is not Econia


<pre><code><b>const</b> <a href="init.md#0xc0deb00c_init_E_NOT_ECONIA">E_NOT_ECONIA</a>: u64 = 0;
</code></pre>



<a name="0xc0deb00c_init_init_econia"></a>

## Function `init_econia`

Initialize Econia with core resources needed for trading


<pre><code><b>public</b> <b>fun</b> <a href="init.md#0xc0deb00c_init_init_econia">init_econia</a>(<a href="">account</a>: &<a href="">signer</a>)
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> entry <b>fun</b> <a href="init.md#0xc0deb00c_init_init_econia">init_econia</a>(
    <a href="">account</a>: &<a href="">signer</a>
) {
    // Assert caller is Econia <a href="">account</a>
    <b>assert</b>!(address_of(<a href="">account</a>) == @econia, <a href="init.md#0xc0deb00c_init_E_NOT_ECONIA">E_NOT_ECONIA</a>);
    <a href="registry.md#0xc0deb00c_registry_init_registry">registry::init_registry</a>(<a href="">account</a>); // Init <a href="registry.md#0xc0deb00c_registry">registry</a>
    // Administer Econia <a href="capability.md#0xc0deb00c_capability">capability</a> <b>to</b> <a href="market.md#0xc0deb00c_market">market</a>
    <a href="market.md#0xc0deb00c_market_init_econia_capability_store">market::init_econia_capability_store</a>(<a href="">account</a>);
}
</code></pre>



</details>
