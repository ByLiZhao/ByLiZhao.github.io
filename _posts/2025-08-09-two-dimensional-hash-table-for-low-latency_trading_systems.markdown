---
layout: post
mathjax: true
comments: true
title:  "Two dimensional hash tables for low latency trading systems"
author: John Z. Li
date:   2025-07-07 10:00:00 +0800
categories: programming
tags: trading, low-latency, software
---
It is well-known that, in order sto make a trading system low-latency, one needs
to optimize its memory access pattern. Not only that more hash-table looking-up
would add extra latency, but also, maybe more importantly, memory locality is crucial
to reduce cache miss. The more we keep hot data in a small-sized piece of memory,
the more likely, when CPU needs data, the data is hot (living in near CPU cache).

The sequential description of business logic often blurs the picture, sometimes leading
to sub-optimal design. A naive implementation of a trading system might follow the following pattern,
(a over-simplified version, for illustration purposes).
1. On receiving a client new order, validate whether the order has a valid `Account` (Tag `1` in FIX protocol).
2. Then check whether the order contains a valid symbol,
3. Then check whether the price of the order violates any risk control.
4. Then check whether the order uses a valid Client Order ID, for example, it is not duplicate ID already used by another order.
5. Whether the client has enough location or cash to support the order.
6. Find the proper Exchange/Broker to route the order to according to some pre-specified rules.
Each of the above steps involves at least one hash table looking-up. For example, to perform risk checks on a client order means
one at least need to find the latest market data for the required symbol, and client order duplication check usually involves one
hash table looking up on the order book that maintain a mapping from client order ID to the actual order.
Indeed, with modern day trading systems, we are swimming in a sea of hash tables.
For real world trading systems, because of regulation and compliance requirements, the number of hash tables used in a trading system
is high.

With all these constraints, however, trading systems can be made fast by optimizing its memory access pattern.
The problem with the above described pipeline pattern is that is too pessimistic. It is true that the system
should reject invalid orders, but the happy path where valid orders flow should be prioritized and made fast.
*It is usually OK to reject an invalid order slowly, but it is never OK to process a valid order slowly*, when low-latency means efficiency and money.

The trick here is to construct 2-dimensional hash tables. Higher dimensional hash tables follow the same idea,
but it is enough to only consider the 2-dimensional case here.
The idea is that for the happy path, we actually only need one hash table access.
Assuming that there are 2 types of keys, one is `Account` and one is `Symbol`, we construct a hash table like below:
1. The key type is tuple `{Account, Symbol}`.
2. When `Symbol` is empty, `{Account, EmptySymbol}` maps to account related settings.
3. When `Account` is empty, `{EmptyAccount, Symbol}` maps to symbol related information.
4. When both `Account` and `Symbol` are present, the key tuple maps to a triplet, consisting of a pointer to Account related info, a pointer to Symbol related info, and information that applies to the account-symbol combination.
![binding-rules](/assets/image/two_dim_cache.png)

Such a hash table does not follow the semantics of normal hash tables,
- Inserting a new key pair `Account, symbol` might lead to more than one insertions to the hash tables.
- If a value is referenced by other values, it should be removed.
- Updating a mapped value in place should not move the object to a new memory location.
Fortunately, these restrictions are not a problem for trading systems. Valid accounts and symbols usually are already determined in program startup.
The map can be constructed incrementally,
- First insert all `EmptyAccount, Symbol --> symbol related info` for all valid symbols.
- Secondly insert all `Account, EmptySymbol --> account related info` for all valid accounts.
- Lastly insert all `Account, symbol --> the troplet` for all account-symbol combinations.
  (Doing so will require 2 hash table looking up to find the `Account` related information and `Symbol` related information respectively.)

Thus, for an incoming client new order, we only need to do one hash table looking-up for its `Account-Symbol` combination.
In one go, we get all the `Account` related information, such as its corresponding allocation account, risk profile, client specific settings,
and all the `Symbol` related information, such as the symbol definition, the market price, any specific restricts imposed on the symbol,
and all the configurations that applies to the specific `Account, Symbol` pair, for example, certain clients might not be allowed to trade on some symbols,

If both `Account` and `Symbol` are valid, the hash table looking up will be successful and all information is readily at hand.
When the looking up fails, if means either the `Account` or the `Symbol` is invalid. In this case, it will take more steps to
identify what is exactly wrong with the order. But as we said, it is OK to reject an invalid order slowly.
