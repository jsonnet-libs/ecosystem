# Effective Jsonnet libraries

**Author:** Tom Braack, Malcolm Holmes  
**Date:** 2020/04/02  
**State:** ~~[Draft
1](https://docs.google.com/document/d/1r8HcwwPMgpUBIkJTwm7wzAcQ0dOZeKDnD4Q9cwBpRWg)~~,
Draft 2

**Table of contents**

- [Target audience](#target-audience)
- [Motivation](#motivation)
- [Proposals](#proposals)
  - [Packages][packages]
  - [Documentation][docs]
  - [Composable API's][composable]
  - [Functional API's][functional]
  - [Explicity][explicity]
- [Effects on Monitoring mixins](#effects-on-monitoring-mixins)

## Target audience

This primarily concerns the Jsonnet communities focused on Kubernetes or
monitoring. Of course everybody else, with or without a Jsonnet background is
welcome to comment.

## Motivation

Some patterns currently observed throughout the Jsonnet ecosystem are far from
optimal. These especially include:

- **Use of the global namespace `$`**: Many libraries, especially
  `prometheus-ksonnet` and the Monitoring Mixins inject their resources
  into certain locations in the global namespace, based on conventions at best.
  This has several drawbacks:

  - **Internals**: Library users must be aware of the library internals and
    structure in the long run, because that's what their namespace is composed
    of.
  - **Mental complexity**: At any time, you must keep track of what's in the
    global namespace. This can become insanely challenging if code is highly
    abstracted.
  - **Interference**: Because libraries share the same namespace, they can
    interfere with each other. Special care must be taken to avoid that.

- **Unstructured APIs**: Current API's are mostly based around overriding
  subsets of a shared `_config` object to influence library behavior. Again,
  the user is required to read the libraries code to figure out what's available.

## Proposals

The following proposals are ideas we have come up with at Grafana Labs based on
our challenges.

While not everything outlined here necessarily applies to everybody, we still
hope to come up with something that can be useful everywhere.

For that reason, we did categorize them in terms of anticipated ecosystem
consistency:

| Icon     | Meaning                                                                       |
| -------- | ----------------------------------------------------------------------------- |
| :scroll: | Expected to be applied to all libraries. We think these are non-controversial |
| :new:    | Recommended to be applied, but more opinionated.                              |
| :eyes:   | Highly opinionated. **Discussion wanted**                                     |

### :scroll: Packages

#### What

All libraries define a carefully crafted public API in a file called
`main.libsonnet`. This file is the (only) one being imported.

#### Why

1. Gives users a consistent place to look for possible options
2. Allows code-analysis tools to easily find what's available, especially
   thinking of LSP and autocomplete
3. Possibly allows dropping the file from the import path by promoting this to
   the upstream Jsonnet importer

### :new: Documentation

#### Why

Proper documentation is a must for developer friendly software. Current projects
have barely any documentation however, mostly related to a lack of appropriate
tooling.

A possible, novel approach is explored in
[`sh0rez/docsonnet`](https://github.com/sh0rez/docsonnet).

#### What (`docsonnet`)

Along your actual API components, you specify `docsonnet`, a special Jsonnet
object format that can be transformed into `godoc`-like HTML pages.

```jsonnet
local d = import "github.com/sh0rez/docsonnet/doc-util/main.libsonnet";

{
  "#new": d.fn("new creates a new `ConfigMap` of given `name` and `data`",
    [d.arg("name", d.T.string), d.arg("data", d.T.string)]),
  new(name, data): {
    apiVersion: "v1",
    kind: "ConfigMap",
    metadata: { name: name },
    data: data,
  }
}
```

Because `docsonnet` is a logical representation of any Jsonnet API it's use can
be extended over textual documentation, e.g. simple to implement code-completion
for imported packages.

### :new: Composable API's

#### What

Instead of the `values.yml` inspired approach (`_config`, etc), libraries by
default only return sane defaults. Customization is done using Patches (`+:`).

```jsonnet
local grafana = import "github.com/jsonnet-libs/grafana/main.libsonnet";

{
  foo: grafana.new()
       + grafana.withVersion("6.4")
       + grafana.withTLS(importstr "cert.pem", importstr "key.pem")
       + grafana.addDashboards({
           foo: import "your-dashboard.libsonnet",
         })
}
```

The patches are defined on the [package][packages], not the individual
instances, to access them regardless of the current scope.

#### Why

1. **Instance separation**: Because there is no shared state, one can have
   multiple, separate instances of the same library
2. **Discoverable**: By reading the `docsonnet` for the library (or even having
   it auto-suggested by the editor), users can easily explore the API
3. **Less logic**: Extensive `if .. then .. else` checking is less frequently
   required, because what's not requested is just not there.

### :eyes: Functional API's

#### What

A Package does not directly consist of it's instance and patches, but holds
functions returning these.

#### Why

- **Parameters**: Some fields like `image` or `name` can't have sane defaults. A
  `function` provides a clear way to pass these.

- **Simplicity**: Users are not required to understand library internals. All
  they need to care about are what modifier functions they applied to their
  respective instance.

- **Compatibility**: Because functions allows arbitrary logic, future changes
  can be worked around without user impact.

### :eyes: Explicity

#### What

Building on [composeable][composable] and [functional][functional] API's, we
propose strict explicity for inter-library communication:

```jsonnet
local grafana = import "github.com/jsonnet-libs/grafana/main.libsonnet";
local loki = import "github.com/grafana/loki/production/loki-mixin/main.libsonnet";

{
  grafana: grafana.new()
           + grafana.addDashboards(loki.grafanaDashboards),
}
```

Libraries don't communicate with each other on their own anymore, but instead
whatever party holds instances of both connects those.

#### Why

- **Obvious**: It is very clear from the code **why** and **how** things happen.
  No more magic. Easier debugging
- **Portable**: A monitoring mixin does not need to be concerned how it's
  resources are consumed
- **Composing**: Creates a feel of plugging components together
- Works **without global state**

## Effects on Monitoring Mixins

This section is subject to being moved into a separate document
(`Monitoring Mixins v2` or similar) eventually.

Many Jsonnet libraries at this time are based on the [Monitoring mixins
document](https://github.com/monitoring-mixins/docs/blob/master/design.pdf),
which proposes patterns that are incompatible to the proposals made here.

### Packages

Mixins seem to use `mixin.libsonnet` as their main file. For simplicities sake
and reasons [previously outlined][packages], they should use `main.libsonnet`
instead.

This change is **non-breaking**, as a `mixin.libsonnet` can be provided
importing the `main.libsonnet`.

### Schema / Constructors

Current Monitoring mixins inject into the following conventional locations in
the global namespace:

- `$.grafanaDashboards` for dashboards
- `$.prometheusAlerts` for Prometheus alert rules
- `$.prometheusRules` for Prometheus recording rules

Because they won't inject anywhere anymore, the ecosystem (Grafana Labs, RedHat,
?) needs to agree on locations / function names where resources are expected, so
they can be passed to the respective Grafana and Prometheus libraries.

The current schema would work fine for that purpose, even though it might be
better to use [functions](#functional). This is mostly a question of whether:

- Mixins need required parameters
- Mixins are considered "fire and forget" bundles, or are subject to extensive
  user customization

<!-- Link definitions -->

[packages]: #scroll-packages
[docs]: #new-documentation
[composable]: #new-composable-apis
[functional]: #eyes-functional-apis
[explicity]: #eyes-explicity
