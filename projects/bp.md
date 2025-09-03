---
layout: page
title: (bp) - Bitcoin Protocol
permalink: /projects/bp/
---

<p align="center">
  <img src="https://raw.githubusercontent.com/rodentrabies/bp/refs/heads/master/docs/logo.svg" width="70%" alt="(bp) logo">
</p>

[**(bp)**][bp-on-github] is a Common Lisp library for interacting with the
Bitcoin Network. Example of reading a Bitcoin transaction from
[Emacs/SLIME][slime] for subsequent introspection:

```lisp
CL-USER> (bp:with-chain-supplier (bp.rpc:node-rpc-connection
                                  :url "http://localhost:8332"
                                  :username "btcuser"
                                  :password "btcpassword")
           (bp:get-transaction "0e3e2357e806b6cdb1f70b54c3a3a17b6714ee1f0e68bebb44a74b1efd512098"))
#<BP.CORE.TRANSACTION:TX 0e3e2357e806b6cdb1f70b54c3a3a17b6714ee1f0e68bebb44a74b1efd512098>
```

[bp-on-github]: https://github.com/rodentrabies/bp
[slime]: https://slime.common-lisp.dev/
