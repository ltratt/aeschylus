# Aeschylus: Auto-Editing ScreenCasts

Aeschylus is a proof-of-concept Auto-Editing ScreenCast system that removes the
need to manually edit screencast videos. As you record a screencast, you tell
Aeschylus that you are at a safepoint, or need to rewind to the last safepoint
and try again. When you are finished, Aeschylus then automatically edits the
video, removing all the rewound scenes, and producing a ready-to-upload file as
output. I first saw a version of this idea in [this video from Vivek
Haldar](https://t.co/xus7b4osCM): relative to Vivek's implementation, Aeschylus
uses the time of events to edit the video. Example videos using Aeschylus
are [Don't Panic! Better, Fewer, Syntax Errors for LR Parsers](https://www.youtube.com/watch?v=PGRnk-bzTdU)
and [Virtual Machine Warmup Blows Hot and Cold](https://www.youtube.com/watch?v=vLl4GteL9Mw).

Aeschylus is extremely rough and ready, held together by strings, low-level,
and won't work on your system without at least some customisation. However, it
might serve as inspiration for someone to build, or extend, a more robust
system in the same vein.


## My requirements

To give you a rough idea of what problem Aeschylus solves, I wanted a system which:

 * Records screencasts with minimal CPU overhead.
 * Puts a picture of me yapping in the bottom right, with the background
   removed with the aid of a green screen.
 * Allows the camera to be recorded at a higher framerate than the screen (the
   latter is quite heavyweight).
 * Does not mix on the fly, and keeps all the raw parts around, so that (some)
   mistakes can be rectified after recording if necessary.
 * Auto-edits out retakes.
 * Sets my system up so that I don't have to remember all the things that make a
   screencast nicer or not.

Things I didn't care about:

 * Support for streaming: Aeschylus is meant for producing offline videos.
 * Readability: I didn't know what I wanted, or what I was doing, when I
   started this, so the code is something of a mess.
 * Portability: Aeschylus started out as OpenBSD/XFCE-only, though I have
   loosened those restrictions somewhat (e.g. it at least minimally works on
   Linux).


## Setting Aeschylus up

Aeschylus takes as its first argument a configuration file, which is a zsh
script. You need to set the following variables in your configuration file:

```
AUDIO_DEVICE="snd/0"
AUDIO_DRIVER="sndio"
# In order to prevent audio being continually converted between formats --
# which can downgrade the quality -- the following two settings should match
# whatever your audio interface is set to.
AUDIO_FMT="s16"
AUDIO_RATE="44100"
MINI_CAMERA_CROP="700:490:350:160"
MINI_CAMERA_SCALE="470:-1"
CAMERA_DEVICE="/dev/video0"
# If you're capturing the camera at a different framerate to X11, make sure the
# two are integer multiples to avoid weird frame duplication in the final
# video.
CAMERA_FRAMERATE=30
# If your audio/video are out of sync, use an audio sync test video such as:
#   24fps https://www.youtube.com/watch?v=lXcZSY4XCDc
#   30fps https://www.youtube.com/watch?v=TjAa0wOe5k4
# If your camera footage is behind the audio, this variable should be positive;
# if the camera footage is ahead of the audio it should be negative. Time in
# milliseconds. For autodetect, use the string "auto".
CAMERA_LATENCY="auto"
# Where to place the camera overlay.
CAMERA_OVERLAY="x=W-w+60:y=H-h+1"
CAMERA_RESOLUTION="1280x720"
# If you don't have a greenscreen, set this to "null"
CHROMAKEY="chromakey=85a869:0.0625:0.005,despill=type=green:mix=0.5:expand=0.3:brightness=0:green=-1:blue=0"
# How often to generate a new GOP in seconds
GOP=3
X11_SCREEN0_DISPLAY=":0.0"
# If you're capturing X11 at a different framerate to the camera, make sure the
# two are integer multiples to avoid weird frame duplication in the final
# video.
X11_SCREEN0_FRAMERATE=15
# By default we capture the entire screen.
X11_SCREEN0_RESOLUTION="1920x1080"
```

On Unix, you can probably copy most of the above, at least at first, but you'll
almost certainly need to change `AUDIO_DEVICE` and `AUDIO_DRIVER` at a minimum.

You also need to define the following functions which setup/teardown any setup
for the screencast (e.g. I kill/suspend several applications I don't want
appearing in the video), though you can leave all of their bodies blank:

```
setup_screencast() {}
restore_settings() {}
kill_apps() {}
unkill_apps() {}
```

`setup_screencast` and `kill_apps` are started before the screencast recording
starts: `restore_settings` and `unkill_apps` afterwards. Note that
`restore_settings` can be called more than once in a single session!

You then need some way of setting "markers" while a screencast is ongoing. The
`epochtime` script takes two arguments: `<output dir>/markers` and `<marker
name>` where `<marker name>` can be one of:

  * `safepoint`: the beginning of a new scene.
  * `rewind`: forget the previous scene and try again.
  * `fastforward`: the screencast is complete, ignore anything after this point.
  * `manual`: leave an edit marker for later manual inspection.
  * `undo`: ignore the previous marker (whatever that marker is).

For example, on XFCE you can set keyboard shortcuts from the command-line:

```
xfconf-query --channel xfce4-keyboard-shortcuts --property "/commands/custom/<Shift>F1" \
  --create --type string --set "`which epochtime` $markers_dir/markers safepoint"
```

Note that Aeschylus automatically synchronises screen, camera, and audio if
`CAMERA_LATENCY="auto"`. This typically works well enough for "dedicated"
webcams but separate capture cards, in particular, tend to introduce extra
latency beyond that which can be auto-detected. In such cases, you will need to
manually determine the latency between the audio and camera and set
`CAMERA_LATENCY` to an appropriate value.


## Running Aeschylus

To quickly test whether Aeschylus will even vaguely run, 

```
$ aeschylus <config> -test
```

where `<config>` is a shell file e.g. `example_configs/linux_xfce`. The `-test`
option captures your X11 screen, your camera, and plays them back (generally
with a small, but noticeable, delay).

To run Aeschylus properly:

```
$ aeschylus <config> <output dir>
```

e.g.

```
$ aeschylus linux_xfce test1
```

For reasons that I don't understand, the first and last second or so of an
ffmpeg recording is often slightly wonky and/or chopped. I therefore suggest
waiting a couple of seconds, pressing `rewind`, and then starting your actual
screencast. Talk as long as you want, set `safepoint`s, make `rewind`s. When
you're ready to finish press `fastforward`, wait at least 1 second and then go
to the terminal running aeschylus and press `q` to stop the ffmpeg recording
process. Aeschylus will then start editing the video. The
edited-but-not-audio-processed version will be in `<output dir>/pre_edit.mkv`.
If you choose to use audio processing, that will be processed further into
`<output dir>/final.mkv` and, if you request it, `<output dir>/final.mp4`. Note
that the `.mkv` file is lossless, and a better format for uploading to e.g.
YouTube.


## Audio processing

I've included my audio processing script `podcast_audio_process`, which makes
audio better sounding -- for me. If you record in a noisy area, it might not
sound very good, but feel free to give it a go. You will need to install [Steve
Harris's LADSPA plugins](http://plugin.org.uk/ladspa-swh/docs/ladspa-swh.html)
-- these are often available as a package in Unix distributions.

When Aeschylus comes to process audio, it will ask you for a signal level. You
can type `audacity` in here, which will load the audio track from the combined
video into `audacity`. You then need to play at least a portion of the audio
and observe the "typical" peak dB level. Quit Audacity, type that number in,
and `podcast_audio_process` will apply a highpass, a noisegate, a
compressor-as-limiter, a "proper" compressor, before running `ffmpeg-normalize`
to produce a consistent audio level. If you're not sure what to do here, type
"-18" and see if you prefer the output before or after. If you don't like
what's going on, it shouldn't be too difficult to disable
`podcast_audio_process`.


## Multiple scene types

Aeschylus has some support for different types of scene (e.g. screen only,
screen with camera): this at least partly works, but isn't tested, so is left
undocumented for now.


## Under the hood

Aeschylus works by associating time stamps with markers and, when recording is
complete, using those to edit videos. For example here is a `markers` file
after a recording:

```
1609515923.285604 rewind
1609515934.217265 safepoint
1609515958.586696 rewind
1609515972.216318 fastforward
```

The long numbers are times in seconds since the Unix epoch, allowing us to tell
when those markers were recorded in "wall-clock time" (in this case,
approximately 2021-01-01 15:45) down to a fraction of a second. This will lead
to a video which trims the start (the first `rewind`), then has a retake
(editing out the recording between the `safepoint` and the second `rewind`) and
then trimming the end (the `fastforward`).

However, by default, most video recording programs (including ffmpeg) record
time relative to the beginning of the recording, with no way of associating it
with an external clock. In ffmpeg's case we can force it to record wall-clock
timestamps instead with `-use_wallclock_as_timestamps` and `-copyts`:

    ffmpeg \
      -use_wallclock_as_timestamps 1 \
      -f v4l2 -i /dev/video0 \
      -map "0:v" -copyts recording.mkv

There are obvious potential problems with clock synchronisation in this setup:
ffmpeg's clock(s) is not necessarily the same as that used as `epochtime`.
Indeed, ffmpeg uses more than one clock source internally, depending on your
input device(s) and other options. Typically ffmpeg uses the "normal" system
clock or a "monotonic" clock (but e.g. on Linux it mostly uses
`CLOCK_MONOTONIC`, which isn't properly monotonic; although you can force the
`v4l2` backend, although virtually nothing else in ffmpeg, to use
`CLOCK_MONOTONIC_RAW`). As this suggests, the situation is something of a mess.
Although I couldn't find any real documentation for this feature, the option I
settled on is to force all recorded streams to synchronise to the clock used to
record audio (in my case, that's the non-monotonic system clock) using the `,`
syntax in `-map`:

    ffmpeg \
      -use_wallclock_as_timestamps 1 \
      -f v4l2 -i /dev/video0 \
      -f sndio -i snd/0 \
      -map "0:v,1:a" -map "1:a" -copyts recording.mkv

In my case I believe this works because the OpenBSD `sndio` backend uses the
non-monotonic system clock. If, for example, it used the monotonic system
clock, `epochtime` would need to use that clock to prevent the `markers` file
having inaccurate data.

A better solution would be to get the current recording timestamp from your
recording software: this would prevent all the messing around, and possible
problems with, the system clock. I have no experience with such software but
something like [this plugin for
OBS](https://obsproject.com/forum/resources/infowriter.345/) might do the trick
if you use OBS (though the description does suggest it only records times to
the resolution of a second, which, if true, wouldn't be sufficient for me).

`markers_to_makefile` then takes in the marker information and creates a
`Makefile` which processes each scene individually before merging them
altogether. The `Makefile` tends to have a high degree of parallelism in it, so
when it's running it is likely to use your CPU fairly intensively.
