![Pedalboard Logo](https://user-images.githubusercontent.com/213293/131147303-4805181a-c7d5-4afe-afb2-f591a4b8e586.png)


[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
![PyPI - Python Version](https://img.shields.io/pypi/pyversions/pedalboard)
![Supported Platforms](https://img.shields.io/badge/platforms-macOS%20%7C%20Windows%20%7C%20Linux-green)
![Apple Silicon supported](https://img.shields.io/badge/Apple%20Silicon-supported-brightgreen)
![PyPI - Wheel](https://img.shields.io/pypi/wheel/pedalboard)
[![Test Badge](https://github.com/spotify/pedalboard/actions/workflows/all.yml/badge.svg)](https://github.com/spotify/pedalboard/actions/workflows/all.yml)
![Coverage Badge](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/psobot/8736467e9952991ef44a67915ee7c762/raw/coverage.json)
![PyPI - Downloads](https://img.shields.io/pypi/dm/pedalboard)
![GitHub Repo stars](https://img.shields.io/github/stars/spotify/pedalboard?style=social)

`pedalboard` is a Python library for adding effects to audio. It supports a number of common audio effects out of the box, and also allows the use of [VST3®](https://www.steinberg.net/en/company/technologies/vst3.html) and [Audio Unit](https://en.wikipedia.org/wiki/Audio_Units) plugin formats for third-party effects. It was built by [Spotify's Audio Intelligence Lab](https://research.atspotify.com/audio-intelligence/) to enable using studio-quality audio effects from within Python and TensorFlow.

Internally at Spotify, `pedalboard` is used for [data augmentation](https://en.wikipedia.org/wiki/Data_augmentation) to improve machine learning models. `pedalboard` also helps in the process of content creation, making it possible to add effects to audio without using a Digital Audio Workstation.

## Features 

 - Built-in support for a number of basic audio transformations: 
   - `Convolution`
   - `Compressor`
   - `Chorus`
   - `Distortion`
   - `Gain`
   - `HighpassFilter`
   - `LadderFilter`
   - `Limiter`
   - `LowpassFilter`
   - `Phaser`
   - `Reverb`
 - Supports VST3® plugins on macOS, Windows, and Linux (`pedalboard.load_plugin`)
 - Supports Audio Units on macOS
 - Strong thread-safety, memory usage, and speed guarantees
   - Releases Python's Global Interpreter Lock (GIL) to allow use of multiple CPU cores
     - No need to use `multiprocessing`!
   - Even when only using one thread:
     - Processes audio up to **300x** faster than [pySoX](https://github.com/rabitt/pysox) for single transforms, and 2-5x faster<sup>[1](https://github.com/iCorv/pedalboard_with_tfdata)</sup> than [SoxBindings](https://github.com/pseeth/soxbindings)
 - Tested compatibility with TensorFlow - can be used in `tf.data` pipelines!

## Installation

`pedalboard` is available via PyPI (via [Platform Wheels](https://packaging.python.org/guides/distributing-packages-using-setuptools/#platform-wheels)):
```
pip install pedalboard
```

If you are new to Python, follow [INSTALLATION.md](https://github.com/spotify/pedalboard/blob/master/INSTALLATION.md) for a robust guide.

### Compatibility

`pedalboard` is thoroughly tested with Python 3.6, 3.7, 3.8, 3.9, and 3.10 as well as experimental support for PyPy 7.3.

- Linux
  - Tested heavily in production use cases at Spotify
  - Tested automatically on GitHub with VSTs
  - Platform `manylinux` wheels built for `x86_64`
  - Most Linux VSTs require a relatively modern Linux installation (with glibc > 2.27)
- macOS
  - Tested manually with VSTs and Audio Units
  - Tested automatically on GitHub with VSTs
  - Platform wheels available for both Intel and Apple Silicon
  - Compatible with a wide range of VSTs and Audio Units
- Windows
  - Tested automatically on GitHub with VSTs
  - Platform wheels available for `amd64` (x86-84, Intel/AMD)

#### Plugin Compatibility

`pedalboard` allows loading VST3® and Audio Unit plugins, which could contain _any_ code.
Most plugins that have been tested work just fine with `pedalboard`, but some plugins may
not work with `pedalboard`; at worst, some may even crash the Python interpreter without
warning and with no ability to catch the error. For an ever-growing compatibility list,
see [COMPATIBILITY.md](COMPATIBILITY.md).

Most audio plugins are "well-behaved" and conform to a set of conventions for how audio
plugins are supposed to work, but many do not conform to the VST3® or Audio Unit
specifications. `pedalboard` attempts to detect some common programming errors in plugins
and can work around many issues, including automatically detecting plugins that don't
clear their internal state when asked. Even so, plugins can misbehave without `pedalboard`
noticing.

If audio is being rendered incorrectly or if audio is "leaking" from one `process()` call
to the next in an undesired fashion, try:

1. Passing silence to the plugin in between calls to `process()`, to ensure that any
   reverb tails or other internal state has time to fade to silence
1. Reloading the plugin every time audio is processed (with `pedalboard.load_plugin`)

## Examples

A very basic example of how to use `pedalboard`'s built-in plugins:

```python
import soundfile as sf
from pedalboard import (
    Pedalboard,
    Convolution,
    Compressor,
    Chorus,
    Gain,
    Reverb,
    Limiter,
    LadderFilter,
    Phaser,
)

audio, sample_rate = sf.read('some-file.wav')

# Make a Pedalboard object, containing multiple plugins:
board = Pedalboard([
    Compressor(threshold_db=-50, ratio=25),
    Gain(gain_db=30),
    Chorus(),
    LadderFilter(mode=LadderFilter.Mode.HPF12, cutoff_hz=900),
    Phaser(),
    Convolution("./guitar_amp.wav", 1.0),
    Reverb(room_size=0.25),
], sample_rate=sample_rate)

# Pedalboard objects behave like lists, so you can add plugins:
board.append(Compressor(threshold_db=-25, ratio=10))
board.append(Gain(gain_db=10))
board.append(Limiter())

# Run the audio through this pedalboard!
effected = board(audio)

# Write the audio back as a wav file:
with sf.SoundFile('./processed-output-stereo.wav', 'w', samplerate=sample_rate, channels=len(effected.shape)) as f:
    f.write(effected)

```

### Loading a VST3® plugin and manipulating its parameters

```python
import soundfile as sf
from pedalboard import Pedalboard, Reverb, load_plugin

# Load a VST3 package from a known path on disk:
vst = load_plugin("./VSTs/RoughRider3.vst3")

print(vst.parameters.keys())
# dict_keys([
#   'sc_hpf_hz',
#   'input_lvl_db',
#   'sensitivity_db',
#   'ratio',
#   'attack_ms',
#   'release_ms',
#   'makeup_db',
#   'mix',
#   'output_lvl_db',
#   'sc_active',
#   'full_bandwidth',
#   'bypass',
#   'program',
# ])

# Set the "ratio" parameter to 15
vst.ratio = 15

# Use this VST to process some audio:
audio, sample_rate = sf.read('some-file.wav')
effected = vst(audio, sample_rate=sample_rate)

# ...or put this VST into a chain with other plugins:
board = Pedalboard([vst, Reverb()], sample_rate=sample_rate)
# ...and run that pedalboard with the same VST instance!
effected = board(audio)
```

For more examples, see:
 - [the `examples` folder of this repository](https://github.com/spotify/pedalboard/tree/master/examples)
 - [the _Pedalboard Demo_ Colab notebook](https://colab.research.google.com/drive/1bHjhJj1aCoOlXKl_lOfG99Xs3qWVrhch)
 - [an interactive web demo on Hugging Face Spaces and Gradio](https://huggingface.co/spaces/akhaliq/pedalboard) (via [@AK391](https://github.com/AK391)) 

## Contributing

Contributions to `pedalboard` are welcomed! See [CONTRIBUTING.md](https://github.com/spotify/pedalboard/blob/master/CONTRIBUTING.md) for details.

## Frequently Asked Questions


### Can Pedalboard be used with live (real-time) audio?

Technically, yes, Pedalboard could be used with live audio input/output. See [@stefanobazzi](https://github.com/stefanobazzi)'s [guitarboard](https://github.com/stefanobazzi/guitarboard) project for an example that uses the `python-sounddevice` library to wire Pedalboard up to live audio.

However, there are a couple big caveats when talking about using Pedalboard in a live context. Python, as a language, is [garbage-collected](https://devguide.python.org/garbage_collector/), meaning that your code randomly pauses on a regular interval to clean up unused objects. In most programs, this is not an issue at all. However, for live audio, garbage collection can result in random pops, clicks, or audio drop-outs that are very difficult to prevent.

Note that if your application processes audio in a streaming fashion, but allows for large buffer sizes (multiple seconds of audio) or soft real-time requirements, Pedalboard can be used there without issue. Examples of this use case include streaming audio processing over the network, or processing data offline but chunk-by-chunk.

### Does Pedalboard support changing a plugin's parameters over time?

Yes! While there's no built-in function for this, it is possible to
vary the parameters of a plugin over time manually:

```python
import numpy
from pedalboard import Pedalboard, Compressor, Reverb

input_audio = ...
output_audio = np.zeros_like(input_audio)
board = Pedalboard([Compressor(), Reverb()])
reverb = board[-1]

# smaller step sizes would give a smoother transition,
# at the expense of processing speed
step_size_in_samples = 100

# Manually step through the audio 100 samples at a time
for i in range(0, input_audio.shape[0], step_size_in_samples):
    # Set the reverb's "wet" parameter to be equal to the percentage through the track
    # (i.e.: make a ramp from 0% to 100%)
    percentage_through_track = i / input_audio.shape[0]
    reverb.wet_level = percentage_through_track
    
    # Process this chunk of audio, setting `reset` to `False`
    # to ensure that reverb tails aren't cut off
    chunk = board.process(input_audio[i : i + step_size_in_samples], reset=False)
    output_audio[i : i + step_size_in_samples] = chunk
```

With this technique, it's possible to automate any parameter. Usually, using a step size of somewhere between 100 and 1,000 (2ms to 22ms at a 44.1kHz sample rate) is small enough to avoid hearing any audio artifacts, but big enough to avoid slowing down the code dramatically.

### Can Pedalboard be used with VST instruments, instead of effects?

Not yet! The underlying framework (JUCE) supports VST and AU instruments just fine, but Pedalboard itself would have to be modified to support instruments.

### Can Pedalboard plugins accept MIDI?

Not yet, either - although the underlying framework (JUCE) supports passing MIDI to plugins, so this would also be possible to add.

## License
`pedalboard` is Copyright 2021 Spotify AB.

`pedalboard` is licensed under the [GNU General Public License v3](https://www.gnu.org/licenses/gpl-3.0.en.html), because:
 - The core audio processing code is pulled from JUCE 6, which is [dual-licensed under a commercial license and the GPLv3](https://juce.com/juce-6-licence)
 - The VST3 SDK, bundled with JUCE, is owned by [Steinberg® Media Technologies GmbH](https://www.steinberg.net/en/home.html) and licensed under the GPLv3.

_VST is a registered trademark of Steinberg Media Technologies GmbH._
