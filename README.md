# Aeschylus: Auto-Editing ScreenCasts

Aeschylus is a proof-of-concept Auto-Editing ScreenCast system that removes the
need to manually edit screencast videos. As you record a screencast, you tell
Aeschylus that you are at a safepoint, or need to rewind to the last safepoint
and try again. When you are finished, Aeschylus then automatically edits the
video, removing all the rewound scenes, and producing a ready-to-upload file as
output. I first saw a version of this idea in [this video from Vivek
Haldar](https://t.co/xus7b4osCM): relative to Vivek's implementation, Aeschylus
uses the time of events to edit the video.

Aeschylus is extremely rough and ready, held together by strings, low-level,
and won't work on your system without at least some customisation. However, it
might serve as inspiration for someone to build, or extend, a more robust
system in the same vein.


# My requirements

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

 * Portability.
 * Readability.


# Setting Aeschylus up

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

You also need to define the following functions:

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

Note that Aeschylus tries to automatically synchronise screen, camera, and
audio: this definitely won't work in all cases, but seems to work OK for simple
cases.


# Running Aeschylus

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


# Audio processing

I've included my audio processing script, which makes audio better sounding --
for me. If you record in a noisy area, it might not sound very good, but feel
free to give it a go. When Aeschylus comes to process audio, it will ask you
for a signal level. You can type `audacity` in here, which will load your audio
file up. You then need to play the audio and observe the "typical" peak dB
level. Quit Audacity, type that number in, and wait to see what happens. If
you're not sure what to do here, type "-18" -- it might not make the output
sound great, but it probably won't mangle it beyond recognition.


# Multiple scene types

Aeschylus has some support for different types of scene (e.g. screen only,
screen with camera): this at least partly works, but isn't tested, so is left
undocumented for now.
