# Hash Byte Order

Dash Core RPCs accept and return the byte-wise reverse of computed SHA-256 hash values. For example, the Unix `sha256sum` command displays the SHA256(SHA256()) hash of mainnet block 300,000's header as:

``` shell
> /bin/echo -n '020000007ef055e1674d2e6551dba41cd214debbee34aeb544c7ec670000000000000000d3998963f80c5bab43fe8c26228e98d030edf4dcbe48a666f5c39e2d7a885c9102c86d536c890019593a470d' \
    | xxd -r -p \
    | sha256sum -b \
    | xxd -r -p \
    | sha256sum -b

5472ac8b1187bfcf91d6d218bbda1eb2405d7c55f1f8cc820000000000000000 (Resulting hash)
```

The result above is also how the hash appears in the previous-header-hash part of [block](../resources/glossary.md#block) 300,001's header:

<pre>02000000<b>5472ac8b1187bfcf91d6d218bbda1eb2405d7c55f1f8cc82000\
0000000000000</b>ab0aaa377ca3f49b1545e2ae6b0667a08f42e72d8c24ae\
237140e28f14f3bb7c6bcc6d536c890019edd83ccf</pre>

However, Dash Core's RPCs use the byte-wise reverse for hashes, so if you want to get information about block 675,776 using the [`getblock` RPC](../api/remote-procedure-calls-blockchain.md#getblock), you need to reverse the requested hash:

``` shell
> dash-cli getblock \
    000000000000327a66cd1011b2d1defd1417b7d9e39b439e8e67ba996ee92602
```

> 📘
>
> Note: hex representation uses two characters to display each byte of data, which is why the reversed string looks somewhat mangled.

The rationale for the reversal is unknown, but it likely stems from Dash Core's use of hashes (which are byte arrays in C++) as integers for the purpose of determining whether the hash is below the network target. Whatever the reason for reversing header hashes, the reversal also extends to other hashes used in RPCs, such as [TXIDs](../resources/glossary.md#transaction-identifiers) and merkle roots.

As header hashes and TXIDs are widely used as global identifiers in other Dash software, this reversal of hashes has become the standard way to refer to certain objects. The table below should make clear where each byte order is used.

| Data | Internal Byte Order | RPC Byte Order |
|---------------|---------------------|-----------------|
| Example: SHA256(SHA256(0x00))  | Hash: 1406...539a         | Hash: 9a53...0614     |
|---------------|---------------------|-----------------|
| Header Hashes: SHA256(SHA256(block header))  | Used when constructing block headers  | Used by RPCs such as `getblock`; widely used in block explorers |
|---------------|---------------------|-----------------|
| Merkle Roots: SHA256(SHA256(TXIDs and merkle rows))  | Used when constructing block headers  | Returned by RPCs such as `getblock` |
|---------------|---------------------|-----------------|
| TXIDs: SHA256(SHA256(transaction))  | Used in transaction inputs | Used by RPCs such as `gettransaction` and transaction data parts of `getblock`; widely used in wallet programs |
|---------------|---------------------|-----------------|
| P2PKH Hashes: RIPEMD160(SHA256(pubkey))  | Used in both addresses and pubkey scripts  | **N/A:** RPCs use addresses which use internal byte order |
|---------------|---------------------|-----------------|
| P2SH Hashes: RIPEMD160(SHA256(redeem script))  | Used in both addresses and pubkey scripts | **N/A:** RPCs use addresses which use internal byte order |
|---------------|---------------------|-----------------|

> 📘
>
> Note: RPCs which return raw results, such as `getrawtransaction` or the raw mode of `getblock`, always display hashes as they appear in blocks ([internal byte order](../resources/glossary.md#internal-byte-order)).

The code below may help you check byte order by generating hashes from raw hex.

``` python
from sys import byteorder
from hashlib import sha256
import codecs

decode_hex = codecs.getdecoder('hex_codec')
encode_hex = codecs.getencoder('hex_codec')

# You can put in $data an 80-byte block header to get its header hash,
# or a raw transaction to get its txid
data = decode_hex('00')[0]
data_hash = sha256(sha256(data).digest()).digest()

print("Warning: this code only tested on a little-endian x86_64 arch")
print()
print("System byte order:        ", byteorder)
print("Internal-Byte-Order Hash: ", encode_hex(data_hash)[0])
print("RPC-Byte-Order Hash:      ", encode_hex(data_hash[::-1])[0])
```
