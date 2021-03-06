# Transform elements

Transform elements transform input buffers to output buffers based on
the sink and source caps.

An important requirement for a transform is that the output caps are
completely defined by the input caps and vice versa. This means that a
typical decoder element can NOT be implemented with a transform element,
this is because the output caps like width and height of the
decompressed video frame, for example, are encoded in the stream and
thus not defined by the input caps.

Typical transform elements include:

  - audio convertors (audioconvert, audioresample,…)

  - video convertors (colorspace, videoscale, …)

  - filters (capsfilter, volume, colorbalance, …)

The implementation of the transform element has to take care of the
following things:

  - efficient negotiation both up and downstream

  - efficient buffer alloc and other buffer management

Some transform elements can operate in different modes:

  - passthrough (no changes are done on the input buffers)

  - in-place (changes made directly to the incoming buffers without
    requiring a copy or new buffer allocation)

  - metadata changes only

Depending on the mode of operation the buffer allocation strategy might
change.

The transform element should at any point be able to renegotiate sink
and src caps as well as change the operation mode.

In addition, the transform element will typically take care of the
following things as well:

  - flushing, seeking

  - state changes

  - timestamping, this is typically done by copying the input timestamps
    to the output buffers but subclasses should be able to override
    this.

  - QoS, avoiding calls to the subclass transform function

  - handle scheduling issues such as push and pull based operation.

In the next sections, we will describe the behaviour of the transform
element in each of the above use cases. We focus mostly on the buffer
allocation strategies and caps negotiation.

## Processing

A transform has 2 main processing functions:

- **`transform()`**: Transform the input buffer to the output buffer. The
output buffer is guaranteed to be writable and different from the input buffer.

- **`transform_ip()`**: Transform the input buffer in-place. The input buffer
is writable and of bigger or equal size than the output buffer.

A transform can operate in the following modes:

- *passthrough*: The element will not make changes to the buffers, buffers are
pushed straight through, caps on both sides need to be the same. The element
can optionally implement a `transform_ip()` function to take a look at the data,
the buffer does not have to be writable.

- *in-place*: Changes can be made to the input buffer directly to obtain the
output buffer. The transform must implement a `transform_ip()` function.

- *copy-transform*: The transform is performed by copying and transforming the
input buffer to a new output buffer. The transform must implement a `transform()` function.

When no `transform()` function is provided, only in-place and passthrough
operation is allowed, this means that source and destination caps must
be equal or that the source buffer size is bigger or equal than the
destination buffer.

When no `transform_ip()` function is provided, only passthrough and
copy-transforms are supported. Providing this function is an
optimisation that can avoid a buffer copy.

When no functions are provided, we can only process in passthrough mode.

## Negotiation

Typical (re)negotiation of the transform element in push mode always
goes from sink to src, this means triggers the following sequence:

  - the sinkpad receives a new caps event.

  - the transform function figures out what it can convert these caps
    to.

  - try to see if we can configure the caps unmodified on the peer. We
    need to do this because we prefer to not do anything.

  - the transform configures itself to transform from the new sink caps
    to the target src caps

  - the transform processes and sets the output caps on the src pad

We call this downstream negotiation (DN) and it goes roughly like this:

```
          sinkpad              transform               srcpad
CAPS event   |                    |                      |
------------>|  find_transform()  |                      |
             |------------------->|                      |
             |                    |       CAPS event     |
             |                    |--------------------->|
             | <configure caps> <-|                      |
```

These steps configure the element for a transformation from the input
caps to the output caps.

The transform has 3 function to perform the negotiation:

- **`transform_caps()`**: Transform the caps on a certain pad to all the
possible supported caps on the other pad. The input caps are guaranteed to be
a simple caps with just one structure. The caps do not have to be fixed.

- **`fixate_caps()`**: Given a caps on one pad, fixate the caps on the other
pad. The target caps are writable.

- **`set_caps()`**: Configure the transform for a transformation between src
caps and dest caps. Both caps are guaranteed to be fixed caps.

If no `transform_caps()` is defined, we can only perform the identity
transform, by default.

If no `set_caps()` is defined, we don’t care about caps. In that case we
also assume nothing is going to write to the buffer and we don’t enforce
a writable buffer for the `transform_ip()` function, when present.

One common function that we need for the transform element is to find
the best transform from one format (src) to another (dest). Some
requirements of this function are:

  - has a fixed src caps

  - finds a fixed dest caps that the transform element can transform to

  - the dest caps are compatible and can be accepted by peer elements

  - the transform function prefers to make src caps == dest caps

  - the transform function can optionally fixate dest caps.

The `find_transform()` function goes like this:

  - start from src aps, these caps are fixed.

  - check if the caps are acceptable for us as src caps. This is usually
    enforced by the padtemplate of the element.

  - calculate all caps we can transform too with `transform_caps()`

  - if the original caps are a subset of the transforms, try to see if
    the the caps are acceptable for the peer. If this is possible, we
    can perform passthrough and make src == dest. This is performed by
    simply calling `gst_pad_peer_query_accept_caps()`.

  - if the caps are not fixed, we need to fixate it, start by taking the
    peer caps and intersect with them.

  - for each of the transformed caps retrieved with `transform_caps()`:

  - try to fixate the caps with `fixate_caps()`

  - if the caps are fixated, check if the peer accepts them with
  `_peer_query_accept_caps()`, if the peer accepts, we have found a dest caps.

  - if we run out of caps, we fail to find a transform.

  - if we found a destination caps, configure the transform with
    `set_caps()`.

After this negotiation process, the transform element is usually in a
steady state. We can identify these steady states:

  - src and sink pads both have the same caps. Note that when the caps
    are equal on both pads, the input and output buffers automatically
    have the same size. The element can operate on the buffers in the
    following ways: (Same caps, SC)

  - passthrough: buffers are inspected but no metadata or buffer data is
    changed. The input buffers don’t need to be writable. The input
    buffer is simply pushed out again without modifications. (SCP)

    ```
              sinkpad              transform               srcpad
      chain()    |                    |                      |
    ------------>|   handle_buffer()  |                      |
                 |------------------->|      pad_push()      |
                 |                    |--------------------->|
                 |                    |                      |
    ```

  - in-place: buffers are modified in-place, this means that the input
    buffer is modified to produce a new output buffer. This requires the
    input buffer to be writable. If the input buffer is not writable, a
    new buffer has to be allocated from the bufferpool. (SCI)

    ```
              sinkpad              transform               srcpad
      chain()    |                    |                      |
    ------------>|   handle_buffer()  |                      |
                 |------------------->|                      |
                 |                    |   [!writable]        |
                 |                    |   alloc buffer       |
                 |                  .-|                      |
                 |  <transform_ip>  | |                      |
                 |                  '>|                      |
                 |                    |      pad_push()      |
                 |                    |--------------------->|
                 |                    |                      |
    ```

  - copy transform: a new output buffer is allocate from the bufferpool
    and data from the input buffer is transformed into the output
    buffer. (SCC)

    ```
              sinkpad              transform               srcpad
      chain()    |                    |                      |
    ------------>|   handle_buffer()  |                      |
                 |------------------->|                      |
                 |                    |     alloc buffer     |
                 |                  .-|                      |
                 |     <transform>  | |                      |
                 |                  '>|                      |
                 |                    |      pad_push()      |
                 |                    |--------------------->|
                 |                    |                      |
    ```

  - src and sink pads have different caps. The element can operate on
    the buffers in the following way: (Different Caps, DC)

  - in-place: input buffers are modified in-place. This means that the
    input buffer has a size that is larger or equal to the output size.
    The input buffer will be resized to the size of the output buffer.
    If the input buffer is not writable or the output size is bigger
    than the input size, we need to pad-alloc a new buffer. (DCI)

    ```
              sinkpad              transform               srcpad
      chain()    |                    |                      |
    ------------>|   handle_buffer()  |                      |
                 |------------------->|                      |
                 |                    | [!writable || !size] |
                 |                    |     alloc buffer     |
                 |                  .-|                      |
                 |  <transform_ip>  | |                      |
                 |                  '>|                      |
                 |                    |      pad_push()      |
                 |                    |--------------------->|
                 |                    |                      |
    ```

  - copy transform: a new output buffer is allocated and the data from
    the input buffer is transformed into the output buffer. The flow is
    exactly the same as the case with the same-caps negotiation. (DCC)

We can immediately observe that the copy transform states will need to
allocate a new buffer from the bufferpool. When the transform element is
receiving a non-writable buffer in the in-place state, it will also need
to perform an allocation. There is no reason why the passthrough state
would perform an allocation.

This steady state changes when one of the following actions occur:

  - the sink pad receives new caps, this triggers the above downstream
    renegotation process, see above for the flow.

  - the transform element wants to renegotiate (because of changed
    properties, for example). This essentially clears the current steady
    state and triggers the downstream and upstream renegotiation
    process. This situation also happens when a RECONFIGURE event was
    received on the transform srcpad.

## Allocation

After the transform element is configured with caps, a bufferpool needs
to be negotiated to perform the allocation of buffers. We have 2 cases:

  - The element is operating in passthrough we don’t need to allocate a
    buffer in the transform element.

  - The element is not operating in passthrough and needs to allocation
    an output buffer.

In case 1, we don’t query and configure a pool. We let upstream decide
if it wants to use a bufferpool and then we will proxy the bufferpool
from downstream to upstream.

In case 2, we query and set a bufferpool on the srcpad that will be used
for doing the allocations.

In order to perform allocation, we need to be able to get the size of
the output buffer after the transform. We need additional function to
retrieve the size. There are two functions:

- `transform_size()`: Given a caps and a size on one pad, and a caps on the
other pad, calculate the size of the other buffer. This function is able to
perform all size transforms and is the preferred method of transforming
a size.

- `get_unit_size()`: When the input size and output size are always
a multiple of each other (audio conversion, ..) we can define a more simple
`get_unit_size()` function. The transform will use this function to get the
same amount of units in the source and destination buffers. For performance
reasons, the mapping between caps and size is kept in a cache.
