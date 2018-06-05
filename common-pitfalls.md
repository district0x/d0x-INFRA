# Common Pitfalls

This document is for writing down all weird errors, that show up when developing a district. Please don't forget to include solution for the error as well.

## Problem 1 (SqliteError: near "/": syntax error)

```
$ node node memefactory-tests/memefactory-server-tests.js

...

throw [cljs.core.str.cljs$core$IFn$_invoke$arity$1(["could not start [",cljs.core.str.cljs$core$IFn$_invoke$arity$1(state),"] due to"].join(''))," ",cljs.core.str.cljs$core$IFn$_invoke$arity$1(t__22248__auto__)].join('');
^
could not start [#'memefactory.server.db/memefactory-db] due to SqliteError: near "/": syntax error
```

*Solution*

Clean compilation folder and recompile files.
