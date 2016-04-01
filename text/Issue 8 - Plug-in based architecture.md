[TOC]

# Back to basics

We currently have three big centralized enums:

- `ChannelKind`, designed to help discovering channels based on their features;
- `Value`, designed to automate (de)serialization of values, on behalf of adapters;
- `Type`, designed to help catch errors.


## Pros

Having such enums is good for standardization, ensuring that adapters agree on
a set of `ChannelKind` and on (de)serialization format.

Having such enums saves time to adapter developers, who do not need to write
their own (de)serializers.

Having such enums means that we can dispatch the same `Value` to different
adapters that offer the same feature, without every single adapter having to
reparse the same `Value`.

Having such enums is good for optimization of the foxbox.

Having such enums makes it possible to make big changes/fixes to the entire architecture
without touching the code of adapters. For instance, ongoing work on
https://github.com/fxbox/taxonomy/issues/37 demonstrates that we can entirely
change the (de)serialization format, or even provide several formats that can
be picked by users at runtime, without the need to patch every single `Adapter`.

## Cons

Decentralized is more webby – for better or worse, this is what most developers
expect.

Experience shows that centralizing everything hurts extensibility and adoption.

We will eventually end up with thousands of variants for these enums, which
is bad for readability, documentation, testing, etc.

This adds a non-negligible maintenance burden to the Taxonomy repo.

## Bottom line

We need to find a solution that:

- lets us standardize channel kinds whenever possible;
- abstracts away (de)serialization whenever possible;
- lets us optimize;
- does not require centralization.

# Plug-in based architecture

The general idea behind this proposal is to keep types that will help discoverability
and (de)serialization, but maintain them as plug-ins instead of centralized enums.
The Foxbox can come with a number of bundled plug-ins, which represent
standardized features, Adapters can come with other plug-ins, and plug-ins
that prove successful can progressively be moved from the code of Adapters
to the standard bundle.


## Values, serialization, deserialization

We replace `Value` and `Type` by a new, extensible, trait `Message`. The (only)
role of a `Message` is to standardize (de)serialization.

A crate `standardized` provides a library of standard `Message`s to cover most uses:
- boolean;
- string;
- various number types;
- dates;
- time;
- temperature;
- image;
- `RawJSON`;
- `RawBinary`;
- ...

Use of a `Message` is decided per-channel, by the Adapter developer.

When implementing a channel, adapter developers are encouraged to pick one
of the full-featured `Message`s (e.g. temperature or time), or can also provide
new implementations of `Message`, if they prefer. Additional instances of `Message`
can easily be published by third-party developers on third-party crates.

While this is less optimal, Adapter developers can fallback to `RawJSON` or `RawBinary`
if they prefer implementing their own mechanism internally, although this may
come at some cost performance cost (typically, they may need to reparse the
same value several times from JSON or Binary).


### Implementation

```rust
trait Message: Sized {
  /// Developer-oriented human-readable name, used mainly for error messages.
  ///
  /// For instance, `bool`, `positive number`, `temperature`.
  fn name() -> String;

  /// Values carried by messages of this type.
  type Type: Any;

  /// A mechanism used to serialize values of this type.
  type Serializer: Serializer<Source = Self::Type>;

  /// A mechanism used to deserialize values of this type.
  type Deserializer: Deserializer<Target = Self::Type>;
}

/// A mechanism used to serialize values of a given type.
trait Serializer {
  type Source;

  /// Serialize a value of type `Source` to JSON. If the value has binary
  /// components, they should be placed in `binary`.
  fn serialize(source: &Self::Source, binary: &mut BinaryComponents) -> Result<JSON, SerializeError>;
}
trait Deserializer {
  type Target;

  /// Deserialize a value of type `Target`. If the value has binary components,
  /// they can be fetched from `binary`.
  fn deserialize(source: &JSON, binary: &BinaryComponents) -> Result<Self::Target, ParseError>;
}
```

Note that `BinaryComponents` are taken from ongoing work on https://github.com/fxbox/taxonomy/issues/37
and are not specific to this RFC.


## Channel features

We replace `ChannelKind` with `Feature`. As `ChannelKind`, the role of a
`Feature` is to allow an application to discover channels (respectively service)
that implement a given feature (respectively feature set), and to standardize
the (de)serialization format for this channel.

A crate `standardized` provides a library of standard `Feature`s to cover a number
of uses:

- garage door opened;
- lock unlocked;
- oven temperature;
- ...

Use of a `Feature` is decided per-channel, by the Adapter developer.

Third-party developers are encouraged to pick a standardized `Feature`, but
are not forced to. Additional instances of `Feature` can easily be published
by third-party developers to third-party crates.

### Implementation


```rust
struct Feature<T> where T: Message {
  /// Keys used to discover this feature.
  ///
  /// For instance: `["x-oven-temperature", "oven-temperature"]`, where
  /// `x-oven-temperature` was the name used prior to standardization and
  /// `oven-temperature` is the standardized name.
  pub keys: Vec<String>,

  phantom: PhantomData<T>
}
```


## Propagating the changes

We modify the definition of channels to use `Feature<T>` instead of
`ChannelKind`. Otherwise, this definition is unchanged. We similarly modify
`add_getter`/`add_setter` to use this new definition of `Channel<T>`.

In the implementation of `Adapter`, type `Value` is replaced by `Any`.

In the Taxonomy API, type `Value` is replaced by `JSON`, with a companion `BinaryParts`.

### Implementation

```rust
pub enum WatchEvent {
  EnterRange {
    from: Id<Getter>,
    json: JSON,
    binary: BinaryParts
  },
  ExitRange {
    from: Id<Getter>,
    json: JSON,
    binary: BinaryParts
  },
  // ...
}

trait API {
  fn fetch_values(&self, Vec<GetterSelector>, user: User) ->
    (ResultMap<Id<Getter>, Option<JSON>, Error>, BinaryParts);
  fn send_values(&self, TargetMap<SetterSelector, JSON>, binary: &BinaryParts,
    user: User) -> ResultMap<Id<Setter>, (), Error>;
}
```

```rust
trait Adapter {
  fn fetch_values(&self, set: &[Id<Getter>], user: User) ->
    ResultMap<Id<Getter>, Option<Box<Any>>, Error>;

  fn send_values(&self, values: HashMap<Id<Setter>, &Any>, user: User) ->
    ResultMap<Id<Setter>, (), Error>;
}
```


# Consequences

## Conflicts

Having the same `Message` in several Adapters causes no conflict.

Several instances of `Feature` may share some or all `keys`, possibly with
different instances of `Message`. This will commonly be the case for features that
have not been standardized yet but that are implemented by several
devices/adapters, or that have been standardized recently. This is considered
normal. The AdapterManager will handle this, by dispatching per key, rather
than by `Feature`.

If several Adapters use distinct instances of `Message` for the same `keys`, or
use the `RawBinary`/`RawJSON` `Message`, this may cause a request to the Foxbox to
be fail due to a parse error or a serialization error with some adapters and
succeed for others. Similarly, this may cause a request to the Foxbox to return
messages with completely unrelated serialization formats for several adapters
implementing the same key.

We consider that this is an acceptable trade-off for decentralization.

## Performance

The `AdapterManager` can be refactored to attempt to group channels with the
same key + `Message` together. This will let the `AdapterManager` perform a single
parsing for each value for all such channels, without loss of performance wrt
the current solution when the channels all the same precise `Message`
(i.e. not `RawJSON` or `RawBinary`).

Channels using `RawJSON` or `RawBinary` are expected to be slower than those
using more precise types.

There will be a small cost associated to using `Any` instead of `Value`, but
this should be extremely small.


## Changes to developers of Adapters

With this solution, developers of Adapters now have more choice. Any `Message`
and `Feature` can come from the standard bundle, third-party crates, or their
own code.

This solution should also bring us one step forward towards implementing
Adapters in other programming languages.

## Changes to developers of web applications

This proposal doesn't affect the REST API.