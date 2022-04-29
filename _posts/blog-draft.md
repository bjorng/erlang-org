# Improvements of the JIT

The [JIT compiler][jit] introduced in Erlang/OTP 24 improved
the performance for Erlang applications.

Erlang/OTP 25 introduces some major improvements of the JIT:

* The JIT now supports the [AArch64 (ARM64)][aarch64] architecture,
used by (for example) [Apple Silicon][apple_silicon] Macs and newer
[Raspberry Pi][rpi] devices.

* Better code generated based on types provided by the Erlang compiler.

* Better support for `perf` and `gdb` with line numbers for Erlang code.

[jit]: https://www.erlang.org/blog/my-otp-24-highlights/#beamasm---the-jit-compiler-for-erlang
[aarch64]: https://en.wikipedia.org/wiki/AArch64
[apple_silicon]: https://en.wikipedia.org/wiki/Apple_silicon
[rpi]: https://en.wikipedia.org/wiki/Raspberry_Pi

### Support for AArch64 (ARM64)

How much speedup one can expect from the JIT compared to the interpreter
varies from nothing to up to four times.

To get some more concrete figures we have run three different
benchmarks with the JIT disabled and enabled on a MacBook Pro (M1
processor; released in 2020).

First we ran the [EStone benchmark][estone]. Without the JIT, 691,962
EStones were achieved and with the JIT 1,597,949 EStones. That is,
more than twice as many EStones with the JIT.

Next we tried running Dialyzer to build a small PLT:

```
$ dialyzer --build_plt --apps erts kernel stdlib
```

With the JIT, the time for building the PLT was reduced from 18.38 seconds
down to 9.64 seconds. That is, almost but not quite twice as fast.

Finally, we ran a benchmark for the [base64][base64] module included
in [this Github issue][gh-5639].

Without the JIT:

```
== Testing with 1 MB ==
fun base64:encode/1: 1000 iterations in 11846 ms: 84 it/sec
fun base64:decode/1: 1000 iterations in 14617 ms: 68 it/sec
```

With the JIT:

```
== Testing with 1 MB ==
fun base64:encode/1: 1000 iterations in 25938 ms: 38 it/sec
fun base64:decode/1: 1000 iterations in 20603 ms: 48 it/sec
```

Encoding with the JIT is almost two and half times as fast, while the
decoding time with the JIT is about 75 percent of the decoding time
without the JIT.

[estone]: https://github.com/erlang/otp/blob/be860185407d6747dca32e8d328b041cc75ffdb3/erts/emulator/test/estone_SUITE.erl
[base64]: https://www.erlang.org/doc/man/base64.html
[gh-5639]: https://github.com/erlang/otp/issues/5639

### Type-based optimizations

The JIT translates one BEAM instruction at the time to native code
without any knowledge of previous instructions. For example, the native
code for the `+` operator must work for any operands: small integers that
fit in 64-bit word, large insters, floats, and non-numbers that should
result in raising an exception.

In Erlang/OTP 25, the compiler embeds type information in the BEAM file
to the help the JIT generate better native code without unnecessary type
tests.

For more details, see the blog post [Type-Based Optimizations in the
JIT][type-based-opts].

[type-based-opts]: https://www.erlang.org/blog/type-based-optimizations-in-the-jit

# Better support for `perf` and `gdb`

**Ask John or Lukas to provide examples.**

# Improved error information for failing binary construction

Erlang/OTP 24 introduced [improved BIF error information][ext_bif_info] to provide
more information when a call to a BIF failed.

[ext_bif_info]: https://www.erlang.org/blog/my-otp-24-highlights/#eep-54-improved-bif-error-information

In Erlang/OTP 25, improved error infomation is also given when the
creation of a binary using the [bit syntax][bit_syntax] fails.

Consider this function:

```
bin(A, B, C, D) ->
    <<A/float,B:4/binary,C:16,D/binary>>.
```

If we call this function with incorrect arguments in past releases
we will just be told that something was wrong and the line number:


```
1> t:bin(<<"abc">>, 2.0, 42, <<1:7>>).
** exception error: bad argument
     in function  t:bin/4 (t.erl, line 5)
```

But which part of line 5? Imagine that `t:bin/4` was called from deep
within an application and we had no idea what the actual values for
the arguments were. It could take a while to figure out exactly what
went wrong.

Erlang/OTP 25 gives us more information:

```
1> c(t).
{ok,t}
2> t:bin(<<"abc">>, 2.0, 42, <<1:7>>).
** exception error: construction of binary failed
     in function  t:bin/4 (t.erl, line 5)
        *** segment 1 of type 'float': expected a float or an integer but got: <<"abc">>
```

Note that the module must be compiled by the compiler in Erlang/OTP 25 in
order to get the more informative error message. The old-style message will
be shown if the module was compiled by a previous release.

Here the message tells us that first segment in the construction was given
the binary `<<"abc">>` instead of a float or an integer, which is the expected
type for a `float` segment.

It seems that we switched the first and second arguments for `bin/4`,
so we try again:

```
3> t:bin(2.0, <<"abc">>, 42, <<1:7>>).
** exception error: construction of binary failed
     in function  t:bin/4 (t.erl, line 5)
        *** segment 2 of type 'binary': the value <<"abc">> is shorter than the size of the segment
```

It seems that there was more than one incorrect argument. In this
case, the message tells us that the given binary is shorter than the
size of the segment.

Fixing that:

```
4> t:bin(2.0, <<"abcd">>, 42, <<1:7>>).
** exception error: construction of binary failed
     in function  t:bin/4 (t.erl, line 5)
        *** segment 4 of type 'binary': the size of the value <<1:7>> is not a multiple of the unit for the segment
```

A `binary` segment has a default unit of 8. Therefore, passing a bitstring of
size 7 will fail.

Finally:

```
5> t:bin(2.0, <<"abcd">>, 42, <<1:8>>).
<<64,0,0,0,0,0,0,0,97,98,99,100,0,42,1>>
```

[bit_syntax]: https://www.erlang.org/doc/reference_manual/expressions.html#bit-syntax-expressions
[pr5281]: https://github.com/erlang/otp/pull/5281
[eep54]: https://www.erlang.org/eeps/eep-0054.html

# Improved error information for failed record matching

Another improvement is the exceptions when matching of a record fails.

Consider this record and function:

```
-record(rec, {count}).

rec_add(R) ->
    R#rec{count = R#rec.count + 1}.

```

In past releases, failure to match a record or retrieve an element from
a record would result in the following exception:

```
1> t:rec_add({wrong,0}).
** exception error: {badrecord,rec}
     in function  t:rec_add/1 (t.erl, line 8)
```

Before Erlang/OTP 15 that introduced line numbers in exceptions, knowing
which record that was expected could be useful if the error occurred in
a large function.

Nowadays, unless several different records are accessed on the same
line, the line number makes it obvious which record was expected.

Therefore, in Erlang/OTP 25 the `badrecord` exception has been changed
to show the actual incorrect value:

```
2> t:rec_add({wrong,0}).
** exception error: {badrecord,{wrong,0}}
     in function  t:rec_add/1 (t.erl, line 8)
```

The new `badrecord` exceptions will show up for code that has been compiled
with Erlang/OTP 25.
