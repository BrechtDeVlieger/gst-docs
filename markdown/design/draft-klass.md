# Element Klass definition

## Purpose

Applications should be able to retrieve elements from the registry of
existing elements based on specific capabilities or features of the
element.

A playback application might want to retrieve all the elements that can
be used for visualisation, for example, or a video editor might want to
select all video effect filters.

The topic of defining the klass of elements should be based on use
cases.

A list of classes that are used in a installation can be generated
using: gst-inspect-1.0 -a | grep -ho Class:.\* | cut -c8- | sed
"s/\\//\\\\n/g" | sort | uniq

## Proposal

The GstElementDetails contains a field named klass that is a pointer to
a string describing the element type.

In this document we describe the format and contents of the string.
Elements should adhere to this specification although that is not
enforced to allow for wild (application specific) customisation.

###string format

    <keyword>['/'<keyword]*

    The string consists of an _unordered_ list of keywords separated with a '/'
    character. While the / suggests a hierarchy, this is _not_ the case.

### keyword categories

- functional

    Categories are base on _intended usage_ of the element. Some elements
    might have other side-effects (especially for filers/effects). The purpose
    is to list enough keywords so that applications can do meaningful filtering,
    not to completely describe the functionality, that is expressed in caps etc..

    - Source : produces data

    - Sink : consumes data

    - Filter : filters/transforms data, no modification on the data is
    intended (although it might be unavoidable). The filter can
    decide on input and output caps independently of the stream
    contents (GstBaseTransform).

    - Effect : applies an effect to some data, changes to data are
    intended. Examples are colorbalance, volume. These elements can
    also be implemented with GstBaseTransform.

    - Demuxer : splits audio, video, … from a stream

    - Muxer : interleave audio, video, … into one stream, this is like
    mixing but without losing or degrading each separate input
    stream. The reverse operation is possible with a Demuxer that
    reproduces the exact same input streams.

    - Decoder : decodes encoded data into a raw format, there is
    typically no relation between input caps and output caps. The
    output caps are defined in the stream data. This separates the
    Decoder from the Filter and Effect.

    - Encoder : encodes raw data into an encoded format.

    - Mixer : combine audio, video, .. this is like muxing but with
    applying some algorithm so that the individual streams are not
    extractable anymore, there is therefore no reverse operation to
    mixing. (audio mixer, video mixer, …)

    - Converter : convert audio into video, text to audio, … The
    converter typically works on raw types only. The source media
    type is listed first.

    - Analyzer : reports about the stream contents.

    - Control : controls some aspect of a hardware device

    - Extracter : extracts tags/headers from a stream

    - Formatter : adds tags/headers to a stream

    - Connector : allows for new connections in the pipeline. (tee, …)

    - …

- Based on media type

    Purpose is to make a selection for elements operating on the different
    types of media. An audio application must be able to filter out the
    elements operating on audio, for example.

    - Audio : operates on audio data

    - Video : operates on video data

    - Image : operates on image data. Usually this media type can also
    be used to make a video stream in which case it is added
    together with the Video media type.

    - Text : operates on text data

    - Metadata : operates on metadata

    - …

- Extra features

    The purpose is to further specialize the element, mostly for
    application specific needs.

    - Network : element is used in networked situations

    - Protocol : implements some protocol (RTSP, HTTP, …)

    - Payloader : encapsulate as payload (RTP, RDT,.. )

    - Depayloader : strip a payload (RTP, RDT,.. )

    - RTP : intended to be used in RTP applications

    - Device : operates on some hardware device (disk, network, audio
    card, video card, usb, …)

    - Visualisation : intended to be used for audio visualisation

    - Debug : intended usage is more for debugging purposes.

- Categories found, but not yet in one of the above lists

    - Bin : playbin, decodebin, bin, pipeline

    - Codec : lots of decoders, encoder, demuxers should be removed?

    - Generic : should be removed?

    - File : like network, should go to Extra?

    - Editor : gnonlin, textoverlays

    - DVD, GDP, LADSPA, Parser, Player, Subtitle, Testing, …

3\) suggested order:

    <functional>[/<media type>]*[/<extra...>]*

4\) examples:

    apedemux         : Extracter/Metadata
    audiotestsrc     : Source/Audio
    autoaudiosink    : Sink/Audio/Device
    cairotimeoverlay : Mixer/Video/Text
    dvdec            : Decoder/Video
    dvdemux          : Demuxer
    goom             : Converter/Audio/Video
    id3demux         : Extracter/Metadata
    udpsrc           : Source/Network/Protocol/Device
    videomixer       : Mixer/Video
    videoconvert     : Filter/Video             (intended use to convert video with as little
                                                 visible change as possible)
    vertigotv        : Effect/Video             (intended use is to change the video)
    volume           : Effect/Audio             (intended use is to change the audio data)
    vorbisdec        : Decoder/Audio
    vorbisenc        : Encoder/Audio
    oggmux           : Muxer
    adder            : Mixer/Audio
    videobox         : Effect/Video
    alsamixer        : Control/Audio/Device
    audioconvert     : Filter/Audio
    audioresample    : Filter/Audio
    xvimagesink      : Sink/Video/Device
    navseek          : Filter/Debug
    decodebin        : Decoder/Demuxer
    level            : Filter/Analyzer/Audio
    tee              : Connector/Debug

### open issues:

  - how to differentiate physical devices from logical ones?
    autoaudiosink : Sink/Audio/Device alsasink : Sink/Audio/Device

## Use cases

- get a list of all elements implementing a video effect (pitivi):

        klass.contains (Effect & Video)

- get list of muxers (pitivi):

        klass.contains (Muxer)

- get list of video encoders (pitivi):

        klass.contains (Encoder & video)

- Get a list of all audio/video visualisations (totem):

        klass.contains (Visualisation)

- Get a list of all decoders/demuxer/metadata parsers/vis (playbin):

        klass.contains (Visualisation | Demuxer | Decoder | (Extractor & Metadata))

- Get a list of elements that can capture from an audio device
(gst-properties):

        klass.contains (Source & Audio & Device)

    - filters out audiotestsrc, since it is not a device
