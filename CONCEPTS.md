# Concepts

The audiocloud API manages several kinds of resources that organize hardware and software audio components into
a distributed graph for real-time audio mixing and processing. Following is a description of these components
and resources that model them in the graph, their state transitions and events they generate through the API.

## Endpoint

An `Endpoint` is an audiocloud API URL that can execute API requests. It is expected that there will be multiple kinds
of endpoints of varying implementations and capabilities. Where the API for example provides multiple kinds of sources
of audio, a certain endpoint may only support certain ones and not others. It is up to the endpoint provider to document
what features and compoents are supported.

The API is designed in such a way to allow one endpoint to be used by another endpoint in order to distribute the
processing and use types of components that are not available to itself.

For example, one endpoint may provide a specific kind of effect or hardware device that another endpoint would like to
use in its process. An application with access to both endpoints could orchestrate into subgraphs on both endpoints and
slave one of them to the a master endpoint that is then used for final processing.

## Mixer

A Mixer is an endpoint-level resource that mixes tracks of audio to produce resulting audio streams. It organizes the
work of the tracks, aligns them in time and mixes the final output, while collecting stats and meters from each point
in the process in order to deliver those as a stream of events.

To time reference its audio mixing work, the mixer takes care of two kinds of clocks, a timeline clock and a streaming
clock. Both clocks run in samples and measure audio time, however the timeline clock can jump around as the timeline is
looped or manipulated such as seeking to a new position on demand. The streaming clock in contrast is monotonically
increasing while the `Mixer` is playing and only resets to zero once the mixer is stopped.

## Track

There are several kinds of tracks and they all produce audio by their own unique means. First tracks that use `Source`s:

- `ClipTrack` organizes a potentially overlapping list of clips (audio files) referenced to the timeline clock. Each clip is backed by a `Source`
- `SourceTrack` plays back a real-time source (such as a synth output), referenced to the stream clock. The entire track is backed by a single `Source`

And special kinds of tracks that don't use `Source`s:

- `RemoteTrack` connects to another, remote `Endpoint` and slaves a remote mixer to the one owning this track - play and stop commands will be forwarded to the remote mixer, remote mixer events will be passed through as local mixer events and remote device instances would be proxied by the local mixer to appear as if they were local.
- `BusTrack` collects outputs of other tracks, mixes it together and runs the resulting audio through a series of inserts

### Sources

Sources generate audio. It's best to think of them as factories for streams of audio. `Source`s and are divided into two
categories, those available to the `ClipTrack` and those available to the `SourceTrack` as well.

Valid sources for the `ClipTrack` are:

- `LocalSource` - a local file, useful for debugging, obviously a security risk for an endpoint available to the internet
- `ObjectSource` - S3-compatible object storage source, such as Amazon S3, Wasabi, ...
- `SignalSource` - audio generated with a signal generation function such as a sine wave

Additionally, these sources are valid for the `SourceTrack`:

- `PushSource` - a source that hands over audio from a queue that is pushed into through the endpoint by an API user, referenced to the stream or the timeline clock (configured when created)
- `InstanceSource` - Some `Instance`s have Audio IO channels on them, such as sound cards. An `InstanceSource` will read audio from such an instance, referenced to the stream clock

### Bus Track special consideration

Bus tracks are special in three ways.

1. The `Mixer` produces produces audio by selecting a bus track as its overall output.
2. Tracks (including other `BusTrack`s) can be routed into a `BusTrack` through links
3. Bus tracks own an ordered list of inserts

Links have an associated channel routing and gain for each route. For example, a link can take a mono track and spread it
out to channels 3 and 4 on a bus, while channels 1 and 2 come from another link. The bus will wait to receive audio from
all links before processing them with an insert.

### Inserts

Inserts are audio processes that map input audio into output audio. These could be effects like reverb, delay, distortion
or analog inserts such as "send audio through channels 3 and 4 of this sound card and receive the audio back through
channels 1 and 2 of the same sound card". Such inserts are called "IO instance inserts" because they reference an IO device
instance.

Inserts can produce their own metering and can be controlled through parameters. So just like `Instance`s they have a
structural `Model` that lists parameters and metering.

## Instances

An instance is a hardware or software thing that can be controlled and/or produces readouts/metering but is not an `Insert`.
Instances have a structural `Model` associated with them that lists the controls that can be modified and meters that will
be produced as well as any MIDI capabilities.

Some instances may be `IO` instances. This means that they can accept or produce digital audio directly through an internal
API (like ASIO, CoreAudio, ...).

# Examples

## Simple playback through a mid

## Real-time synth with backing track and post processing

- "Site A" has a Moog synth with MIDI input connected into an RME soundcard channels 1 and 2 with an ASIO driver
- "Site B" has a pair of Pultec EQs connected into a Dante soundcard channels 5 and 6

An application with access to both sites would like to connect the synth output into the pultecs, mixed with a backing track
from an URL and deliver the resulting live stream to the end user.

1. Create a mixer on "Site A" with a `SourceTrack` "moog" referencing the RME soundcard channels 1 and 2 as source and instance "moog/1"
2. Add a `BusTrack` "output" with no inserts, and routing "moog" to "output"
3. Create a mixer on "Site B" with a `RemoteTrack` "synth" referencing the "Site A" mixer and instances "pultec/1" and "pultec/2"
4. Add a `ClipTrack` "backing" with one clip, referencing the backing file URL.
5. Add a `BusTrack` "synth_fx" with one analog insert, sending the audio to the pultecs and back and routing the "synth" track to "synth_fx" through the sound card "asio/hdnative/1"
6. Add a `BusTrack` "output" with no inserts and routing both "synth_fx" and "backing" to "output"
7. Play "Site B" mixer with the "output" bus as the output. The API endpoint of "Site B" has access to three instances: "moog/1", "pultec/1" and "pultec/2"
8. Send note on/off and CC MIDI messages to instance "moog/1" and present the output to the user by decoding the audio stream

The "Site A" mixer configuration could be something like this (represented as YAML for terseness, though actually JSON):

```yaml
sample_rate: 48000
tracks:
  moog:
    type: source
    source:
      type: instance
      instance:
        instance_id: asio/rme/1
        sample_rate: 48000
        channels: [0, 1]
  output:
    type: bus
links:
  link1:
    source_track: moog
    bus: output
    channels:
      - source: 0
        bus: 0
        gain: 1.0
      - source: 1
        bus: 1
        gain: 1.0
instances:
  moog/1: moog/1
authentications:
  - header: Site2Site cMQrA5ehmOgWEHgfHmQGsWTWlxO5ufVvuIEmwRzSWq
    scopes:
      - mixer:read
      - mixer:write
      - settings:read
      - settings:write
      - stream:read
```

After POSTing the mixer to "Site A", it has given us the identifier "n5zEfoorqnk395sD2G70R". The "Site B" mixer configuration could be
something along these lines:

```yaml
sample_rate: 192000
tracks:
  synth:
    type: remote
    remote:
      sample_rate: 48000 # sample rate conversion and clock translation will happen when the remote mixer is connected to us
      endpoint: https://api.mysynth.io
      mixer_id: n5zEfoorqnk395sD2G70R
      authentication: Site2Site cMQrA5ehmOgWEHgfHmQGsWTWlxO5ufVvuIEmwRzSWq
      instances:
        - moog/1
  backing:
    type: clips
    clips:
      - start: 0
        length: 11520000 # a 4 minute backing track at 48000hz
        source:
          type: object
          endpoint_or_id: "@sharedstorage" # this shared storage actual endpoint, access and secret keys is something that "Site B" knows about as a part of its configuration and is just referenced here
          bucket: "user1835"
          object_id: "projects/untitled/track1.wav"
          format: wav
  synth_fx:
    type: bus
    inserts:
      - input_channels: [0, 1]
        output_channels: [0, 1]
        process:
          type: instance_exchange
          instance_id: asio/hdnative/1
          send_channels: [3, 4]
          receive_channels: [0, 1]
          sample_rate: 192000
  output:
    type: bus
links:
  link1:
    source_track: synth
    bus: synth_fx
    channels:
      - source: 0
        bus: 0
        gain: 1.0
      - source: 1
        bus: 1
        gain: 1.0
  link2:
    source_track: synth_fx
    bus: output
    channels:
      - source: 0
        bus: 0
        gain: 1.0
      - source: 1
        bus: 1
        gain: 1.0
  link3:
    source_track: backing
    bus: output
    channels:
      - source: 0
        bus: 0
        gain: 1.0
      - source: 1
        bus: 1
        gain: 1.0
instances:
  pultec/1: pultec/1
  pultec/2: pultec/2
```
