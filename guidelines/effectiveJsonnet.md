# Effective Jsonnet libraries

**Author:** Tom Braack, Malcolm Holmes  
**Date:** 2020/04/02  
**State:**
~~[Draft 1](https://docs.google.com/document/d/1r8HcwwPMgpUBIkJTwm7wzAcQ0dOZeKDnD4Q9cwBpRWg)~~
Draft 2

## Target audience

This primarily concerns the Jsonnet communities focused on Kubernetes or monitoring. Of
course everybody else, with or without a Jsonnet background is welcome to comment.

## Motivation

Some patterns currently observed throughout the Jsonnet ecosystem are far from
optimal. These especially include:

- **Use of the global namespace `$`**: Many libraries, especially
  `prometheus-ksonnet` and the Monitoring Mixins inject whatever they supply
  into certain locations in the global namespace, based on conventions. This has
  several drawbacks:

  - **Internals**: Library users must be aware of the library internals in
    the long run, because that's what their namespace is composed of.
  - **Mental complexity**: At any time, you must keep track of what's in the
    global namespace. This can become insanely, challenging
    if code is highly abstracted.
  - **Interference**: Because libraries share the same namespace, they can
    interfere with each other, which must be carefully avoided. This is risky.

- **Unstructured APIs**: Current API's are mostly based around overriding
  subsets of a shared `_config` object, to influence library behavior. This
  again requires the user to read the libraries code to figure out what's
  available. It is not discoverable and thus a bad developer experience.

## Proposals

The following proposals are ideas we have come up with at Grafana Labs based on
our challenges. Not all of these necessarily apply to everybody.

For that reason, we did categorize them in terms of anticipated ecosystem consistency:

| Icon     | Meaning                                                                       |
| -------- | ----------------------------------------------------------------------------- |
| :scroll: | Expected to be applied to all libraries. We think these are non-controversial |
| :new:    | Recommended to be applied. More opinionated, but good to have                 |
| :eyes:   | Early stage, discussion expected                                              |

### :scroll: Packages

#### What

All libraries define a carefully crafted public API in a file called
`main.libsonnet`. This file is the (only) one being imported.

#### Why

- Gives users a consistent place to look for possible options
- Allows code-analysis tools to easily find what's available
  - especially thinking of LSP and autocomplete
- Possibly allows dropping the file from the import path by promoting this to the upstream Jsonnet
  importer

### :new: Documentation

### :new: Composable API's

### :eyes: Functional API's
