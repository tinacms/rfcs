# Deterministic Plugin Ordering

This RFC proposes additional metadata for plugins that can influence the order they are retrieved from the system.

## Problem

Plugins are internally represented as a Map, which iterates over entries in insertion order.

Thus, whatever order plugins are initially registered via `cms.plugins.add` is the order they will be retrieved via `cms.plugins.all`. In the case of forms and other ui-influencing plugins, this determines the order they appear when presented to the user.

It is possible to influence plugin order by changing which plugins get added first, but in practice this is clumsy.

## Proposal

We can make plugin order more explicit by adding a `__weight` field to the `Plugin` interface, and sorting the plugin list by this weight value before it's returned in `PluginType#all`.

All plugins can have some standard arbitrary weight value by default - say, 50? Plugins with equal weights will fall back to insertion order for priority.

## Challenges

- What are the semantics of `__weight`? Are lower values first, or higher values first? Is there a minimum/maximum value that `__weight` should be?
- What if consumers want to ensure their plugin sites between two other plugins registered by third-party libraries? Should we have a mechanism for influencing weight of already-registered plugins?
