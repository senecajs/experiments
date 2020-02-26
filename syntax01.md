

From @seneca/user

```
/* Copyright (c) 2012-2020 Richard Rodger and other contributors, MIT License. */
'use strict'

module.exports = ctx => {
  const intern = ctx.intern
  // const options = ctx.options

  return async function make_verify(msg) {
    var seneca = this

    var found = await intern.find_user(this, msg, ctx)

    if (!found.ok) {
      return found
    }

    var kind = msg.kind
    var code = msg.code || intern.make_token()
    var once = null == msg.once ? true : !!msg.once
    var valid = null == msg.valid ? false : !!msg.valid
    var unique = null == msg.unique ? true : !!msg.unique

    var custom = msg.custom || {}

    // convenience field for `kind`
    if (null != msg[kind]) {
      custom[kind] = msg[kind]
    }

    var now_date = new Date()
    var t_c = now_date.getTime()
    var t_c_s = now_date.toISOString()
    var t_m = t_c
    var t_m_s = t_c_s

    var t_expiry =
      null != msg.expire_point
        ? msg.expire_point
        : null != msg.expire_duration
        ? t_c + msg.expire_duration
        : Number.MAX_SAFE_INTEGER

    var verify_data = {
      ...custom,
      user_id: found.user.id,
      kind: kind,
      code: code,
      once: once,
      used: false,
      valid: valid,
      t_expiry: t_expiry,
      t_c: t_c,
      t_c_s: t_c_s,
      t_m: t_m,
      t_m_s: t_m_s,
      sv: intern.SV
    }

    if (unique) {
      var unique_query = {
        user_id: found.user.id,
        kind: kind
      }

      if (null == custom[kind]) {
        unique_query.code = code
      } else {
        unique_query[kind] = custom[kind]
      }

      var existing_verify = await seneca
        .entity(ctx.sys_verify)
        .load$(unique_query)

      if (existing_verify) {
        return {
          ok: false,
          why: 'not-unique',
          details: {
            query: unique_query
          }
        }
      }
    }

    var verify = await seneca
      .entity(ctx.sys_verify)
      .data$(verify_data)
      .save$()

    return { ok: true, verify: verify }
  }
}

/* 

ABBREVIATED SYNTAX FOR ABOVE

TRANSLATE TO LISP TO EVALUATE?

whitespace is not significant

$msg - inbound
! - send response
& - last msg sent
% - last response
$x - name something via =
?: - conditional
# define pattern
; - end response matching, optional, to pop response matching stack
$ - current msg
~ match
// comment

$save = sys:entity,cmd:save,canon:sys/verify

# sys:user,make:verify,
  kind ~ string,
  code ~ string | "make_token()",
  once ~ bool | true,
  valid ~ bool | false,
  unique ~ bool | true,
  custom ~ object | {},

> sys:user, find:user, q:$msg
< ok:false !%

< ok:true, user~object $user = %.user

// auto fail if no matching response, or pass through if '<' alone

? $msg.unique
  > sys:entity,
    cmd:load,
    canon:sys/verify,
    q: {
      user_id: $user.id,
      kind: $msg.kind,
      $msg.kind: $msg[$msg.kind] | code:$msg.code
    }
  < exists ! ok:false, why:not-unique, details:{query:&.q}
  ;

$now = "new Date()"

> $save, ent: {
    $msg.custom,
    user_id: $user.id,
    kind: $msg.kind,
    code: $msg.code,
    once: $msg.once,
    valid: $msg.valid | false,
    t_expiry: $msg.expire_point |
      (? $msg.expire_duration "$.t_c+$msg.expire_duration" :
        "Number.MAX_SAFE_INTEGER")
    t_c: "$now.getTime()",
    t_c_s: "$now.toISOString()",
    t_m: $.t_c,
    t_m_s: $.t_c_s,

  }
< ! {ok:true, verify:%}


*/

```

Also: https://mail.python.org/archives/list/python-ideas@python.org/message/52DLME5DKNZYFEETCTRENRNKWJ2B4DD5/

```
 Guido van Rossum
15 Mar 2019 6:51 p.m.
There's been a lot of discussion about an operator to merge two dicts. I
participated in the beginning but quickly felt overwhelmed by the endless
repetition, so I muted most of the threads.

But I have been thinking about the reason (some) people like operators, and
a discussion I had with my mentor Lambert Meertens over 30 years ago came
to mind.

For mathematicians, operators are essential to how they think. Take a
simple operation like adding two numbers, and try exploring some of its
behavior.

add(x, y) == add(y, x)    (1)
Equation (1) expresses the law that addition is commutative. It's usually
written using an operator, which makes it more concise:

x + y == y + x    (1a)
That feels like a minor gain.

Now consider the associative law:

add(x, add(y, z)) == add(add(x, y), z)    (2)
Equation (2) can be rewritten using operators:

x + (y + z) == (x + y) + z    (2a)
This is much less confusing than (2), and leads to the observation that the
parentheses are redundant, so now we can write

x + y + z    (3)
without ambiguity (it doesn't matter whether the + operator binds tighter
to the left or to the right).

Many other laws are also written more easily using operators.  Here's one
more example, about the identity element of addition:

add(x, 0) == add(0, x) == x    (4)
compare to

x + 0 == 0 + x == x    (4a)
The general idea here is that once you've learned this simple notation,
equations written using them are easier to manipulate than equations
written using functional notation -- it is as if our brains grasp the
operators using different brain machinery, and this is more efficient.

I think that the fact that formulas written using operators are more easily
processed visually has something to do with it: they engage the brain's
visual processing machinery, which operates largely subconsciously, and
tells the conscious part what it sees (e.g. "chair" rather than "pieces of
wood joined together"). The functional notation must take a different path
through our brain, which is less subconscious (it's related to reading and
understanding what you read, which is learned/trained at a much later age
than visual processing).

The power of visual processing really becomes apparent when you combine
multiple operators. For example, consider the distributive law:

mul(n, add(x, y)) == add(mul(n, x), mul(n, y))  (5)
That was painful to write, and I believe that at first you won't see the
pattern (or at least you wouldn't have immediately seen it if I hadn't
mentioned this was the distributive law).

Compare to:

n * (x + y) == n * x + n * y    (5a)
Notice how this also uses relative operator priorities. Often
mathematicians write this even more compact:

n(x+y) == nx + ny    (5b)
but alas, that currently goes beyond the capacities of Python's parser.

Another very powerful aspect of operator notation is that it is convenient
to apply them to objects of different types. For example, laws (1) through
(5) also work when n, x, y and z are same-size vectors (substituting a
vector of zeros for the literal "0"), and also if x, y and z are matrices
(note that n has to be a scalar).

And you can do this with objects in many different domains. For example,
the above laws (1) through (5) apply to functions too (n being a scalar
again).

By choosing the operators wisely, mathematicians can employ their visual
brain to help them do math better: they'll discover new interesting laws
sooner because sometimes the symbols on the blackboard just jump at you and
suggest a path to an elusive proof.

Now, programming isn't exactly the same activity as math, but we all know
that Readability Counts, and this is where operator overloading in Python
comes in. Once you've internalized the simple properties which operators
tend to have, using + for string or list concatenation becomes more
readable than a pure OO notation, and (2) and (3) above explain (in part)
why that is.

Of course, it's definitely possible to overdo this -- then you get Perl.
But I think that the folks who point out "there is already a way to do
this" are missing the point that it really is easier to grasp the meaning
of this:

d = d1 + d2
compared to this:

d = d1.copy()
d = d1.update(d2)
and it is not just a matter of fewer lines of code: the first form allows
us to use our visual processing to help us see the meaning quicker -- and
without distracting other parts of our brain (which might already be
occupied by keeping track of the meaning of d1 and d2, for example).

Of course, everything comes at a price. You have to learn the operators,
and you have to learn their properties when applied to different object
types. (This is true in math too -- for numbers, xy == yx, but this
property does not apply to functions or matrices; OTOH x+y == y+x applies
to all, as does the associative law.)

"But what about performance?" I hear you ask. Good question. IMO,
readability comes first, performance second. And in the basic example (d =
d1 + d2) there is no performance loss compared to the two-line version
using update, and a clear win in readability. I can think of many
situations where performance difference is irrelevant but readability is of
utmost importance, and for me this is the default assumption (even at
Dropbox -- our most performance critical code has already been rewritten in
ugly Python or in Go). For the few cases where performance concerns are
paramount, it's easy to transform the operator version to something else --
once you've confirmed it's needed (probably by profiling).

-- 
--Guido van Rossum (python.org/~guido)
```
