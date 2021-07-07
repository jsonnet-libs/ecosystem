# Jsonnet Principles
Jsonnet is a rich language. Here we look at some of the mechanisms available within the
language. This is not intended as a full introduction to the language itself, but rather
to provide context to a discussion around programming style within Jsonnet code.

## Patching

In Jsonnet, multiple JSON trees can be combined with the `+` operator.
This results in a merge. Thus:

   {
       foo: 1,
   } + {
       foo: 2,
   }

will result in this JSON: `{foo:2}`.

However, we need to be careful, and use another feature if we want a 
proper deep merge:

    {
        foo: {
            bar: 1,
            baz: 2,
        }
    } + {
        foo: {
            baz: 3,
        }
    }

This will result in `{foo: {baz: 3}}`, with `bar` having been lost. This is
because the second `foo` entirely replaced the first. To _merge_ rather than _replace_, we need to use `+:` like so:

    {
        foo: {
            bar: 1,
            baz: 2,
        }
    } + {
        foo+: {
            baz: 3,
        }
    }

The first `foo` did not use `+:`, as it is where `foo` was first defined.
However, as a convention, for a tree that we plan to extend, it makes sense
to always use `+:`, as this prevents accidental replacement, should some
previous code have already defined a value for `foo`. Thus:

    {
        foo+: {
            bar: 1,
            baz: 2,
        }
    } + {
        foo+: {
            baz: 3,
        }
    }

## Local Variables and functions
A local variable can contain both elements and functions, like so:

    local x = {
        foo: 1,
        doBar(p):: {
            result: p * foo,
        },
    };
    {
        baz: x.doBar(2),
    }

In this example, `baz` will be 2.

## Scopes and `$`
Every Jsonnet program has a root 'scope' accessible via `$`. Code in functions within local variables also have their own root 'scope', so 
`$` within a function in a local references the root of the local
variable.

    local x = {
        foo: 1,
        doBar(p):: {
            result: p * $.foo,
        },
    };
    {
        foo:2,
        baz: x.doBar(2),
    }

In this adapted example, `$` references the root of the local variable, meaning `foo` is still `1`, and thus `baz` is still `2`.

## Merging Scopes
In the examples above around scope, it was shown that `$` refered to the 
root of the local variable.

    local x = {
        foo: 1,
        doBar(p):: {
            result: p * $.foo,
        },
    };
    x + {
        foo:2,
        baz: $.doBar(2),
    }

In this scenario, x has been merged with the main code's root tree, and 
thus, when `doBar` is executed, `$` will be the program's root scope, not
that of the local variable `x`. As a result, in this version, `baz` will be `4`, because it will have used a differend value for `foo`.
