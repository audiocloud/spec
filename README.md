# If you are using audio software or hardware

Odds are that in the future you will be using audio processing that is both local to your workstation or hardware unit as well as units that are connected to the internet, somewhere far away. They may be at your home base studio, your friends' studio or a demo unit put up by the manufacturer to be used online by anyone interested.

This repository proposes a way to build a common vocabulary for audio software and hardware that is used through the internet in real-time  or as an offline bounce. It features control over parameters, feedback of meters (such as gain reduction) and a way to morph presets from one machine or vendor to another.

# If you are making audio software or hardware

There are many open standards out there already like MIDI and WebAudio, as well as proprietary standards like VST and AAX. None of these specifically tackle distributed audio processing and preset management that is core to emerging products in the pro audio industry.

If you want to make a product that can network with other gear and software over the internet, you will need to communicate setting changes, audio streams and other information.

This repository proposes a communications protocol that is easy to use over the internet and not too hard to implement.

# API Specification compatible endpoints

See [openapi.yaml](openapi.yaml) for OpenAPI 3.x compatible API declaration of the REST endpoint.

A conforming implementation must implement the REST endpoint, but can also implement a WebRTC endpoint (documented separately, TBD).
