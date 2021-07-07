# Jsonnet Style Guide

## Introduction
At present, there is a somewhat informal programming style in place within
Jsonnet libraries. 

## A Basis for Change
While various people have suggestions as to how this
could be improved, a valuable starting point is to pin down this existing
style down, allowing it to be updated by the community, using PRs as a
change management process.

Please do not take this as a recommendation. At present, this is a statement
of a state of play, and is intended to clarify these practices, and to form
the basis for discussion about possible changes.

## Jsonnet Principles
Jsonnet is a rich language. Before we can fully discuss programming style
choices, we need to clarify some features of the language itself.

### Patching

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

### Local Variables and functions
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

### Scopes and `$`

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

### Merging Scopes
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

## Jsonnet Programming Style Guide
### Lint
Jsonnet has a sibling, jsonnetfmt. Jsonnet code is expected to be formatted
in such a way as jsonnetfmt will not make changes.

### Merging into the root object
According to convention, it is preferred to merge objects with functions
into the root scope and access them via `$`, rather than accessing them
via local variables.

## Configuration with _config
Within the root scope, there are a few elements that are commonly used, 
by convention. The two most common are `_config` and `_images`.

It is common to see code such as this:

    mylib.libsonnet:
    {
        _config+:: {
            foo: 1,
            bar: $._config.foo,
        },
        ...

    }

    main.jsonnet:
    local mylib = import 'mylib.libsonnet';
    mylib + {
        _config+:: {
            foo: 2,
        }
    }

Here, we have a library having defined `$._config.foo` as being 1. However,
later we overrule this, setting `$._config.foo` to 2. Because Jsonnet is 
a lazy evaluated language, `$._config.bar` will be 2, as `foo` was overridden.

`$._images` follows the same pattern, but is used to define Docker image
versions.

## Merging root object
In the above section, [merging scopes](./style.md#merging_scopes), a method
was detailed whereby libraries containing functions are merged into the 
root scope, and those functions are accessed via `$`. This is the approach
used for accessing library functions.

The most common pattern using this is to import a Kubernetes library, and 
merge that with the root object, and access it via `$`, like so:

    local k = import 'ksonnet-util/kausal.libsonnet';

    k + {
      local configMap = $.core.v1.configMap,

      my_configmap: configMap.new('my-config'),
    }

In this example, we can see that we have created a shortcut `configMap` by
accessing the Kubernetes library via `$`, i.e. `$.core.v1.configMap`.

## Extending existing libraries with patches
Functions return snippets of JSON. If that JSON is not exactly how you need
it, then you can change it via Jsonnet patching.

Thus, we might have code such as the below:

    local x = {
        y():: {
            foo: {
                bar: 1,
                baz: 2,
            },
        },
    };
    {
        out: x.y() + {
            foo+: {
                baz: 3,
            },
        },
    }
## Camel case vs underscores
There is currently no convention in place around camelCase vs underscored_names.

## local shortcuts to k.libsonnet resources

