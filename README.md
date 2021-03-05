
# CRT 240p Scale Shader

![Hero](doc-img/hero.png?raw=true)

## What's this?

This is a [RetroArch](https://www.retroarch.com/)
[GLSL](https://en.wikipedia.org/wiki/OpenGL_Shading_Language) shader for scaling a wide
range of emulated consoles, handhelds and arcade systems to look high-quality and
authentic on a standard definition, horizontal, 4:3, CRT TV through a single 240p super
resolution.

I'm using the shader with a customized [RetroPie](https://retropie.org.uk/) installation on a
[Raspberry Pi](https://www.raspberrypi.org/) 3B with a
[PI2SCART](http://arcadeforge.net/Pi2Jamma-Pi2SCART/PI2SCART::264.html) hat, but it should
work fine with different 240p RGB output solutions or software setups.

While I cannot claim the results to be absolutely perfect, it has certainly satisfied me,
somebody deeply familiar with how most of the systems in RetroPie look as original hardware on
a CRT or through an OSSC. I went from constantly tweaking my RetroArch scaling
options to simply playing games.

## Features

![Super](doc-img/super.jpg?raw=true)

- Supports a wide range of emulators with no need for per-system or per-game tweaking. Set
this up once and play thousands of games without ever thinking about aspect ratios,
resolutions, scaling, centering, shimmering, interpolation, etc. again
- Care is taken to have the authentic looking aspect ratio for all systems
- Never introduce blurring through filtering on the horizontal axis and filtering is never
visible vertically unless downscaling is required
- TATE games will be automatically displayed in the correct orientation and aspect ratio.
Super-sampling is employed to reduce vertical shimmering and a gentle sharpening filter
mitigates the loss of vertical resolution. An optional overscan correction can be applied
as TATE arcade games are generally not designed with consumer CRT overscan in mind
- Displays handhelds with the correct aspect ratio and by centering them on-screen. TATE
handhelds like the WonderSwan and Lynx are supported, including mid-game rotation
- Code is very short, well-documented and simple. It's trivial to add special cases
for a specific resolution or system
- Negligible performance impact

## Motivation

![Clarkson](doc-img/clarkson.jpg?raw=true)

If you've ever tried to get a number of RetroArch cores emulating dozens of consoles,
handhelds and arcade systems to look correct on your CRT, you don't need to read this.
You know what a glitchy mess the Settings->Video->Scaling menu is and how ill-equipped
RetroArch scaling is to deal with things like non-square pixel super wide resolutions.
But if you need a reason why this shader exists, let's see what you'd have to do
without it.

We want want to use a standard definition 4:3 CRT and have configured our system to output
at a reasonable 320x240 resolution. We'll now go through a few typical game resolutions
and see what we have to do to convince RA to output in a matter that looks authentic and
high quality.

- **320x240** -
Nothing to be done, great!
- **320x224** -
The wrong thing would be to stretch 224 lines to 240. This would give both the wrong
aspect ratio and introduce severe blurring. We can achieve the correct vertical centering
of 224 lines inside 240 with integer scaling, but the image would then not use the full 320
horizontal pixels in each line as the aspect ratio math is done incorrectly on 224 lines.
Even for this simple case already we'd have to use the 'custom' mode and configure the
scaling completely manually
- **320x256** -
Arcade games like R-Type often output in >240p resolutions, we need to downscale.
Unfortunately, RA does not support this very well as the input is not MIP-mapped and
there's no build-in supersampling, so shimmering is unavoidable. The best we can do is
turn on bilinear filtering in the video menu. This can only be done for both axis at the
same time and can introduce a loss of sharpness and brightness in small features on an
axis where it isn't even required.
- **160x144** -
Handheld resolutions like these can only be properly displayed by centering them
on-screen. One complication is that many handhelds do not have square pixels. Time to
pull out the calculator to find the correct image width and on-screen X/Y offsets.
- **256x224** -
There is no way to make this horizontal resolution look good with our chosen output
resolution. We can either assume square pixels and a 8:7 aspect, which would look sharp but
wrong with a squeezed image and black borders or we can stretch 256 to 320 and introduce
severe blurring. At this point we'd have no choice but to detect the system in the
`runcommand-onstart.sh` script and chose a different output resolution for each console.
- **352x240** -
Many consoles like the PC Engine or the PS1 have a wide range of output resolutions and
there is no one resolution correct for the entire library. We need a per-game resolution
database to handle all those systems correctly. But even this doesn't work in all cases as
some games switch resolution mid-game or let the user chose the output resolution (Soldier
Blade on PCE, for instance). The only thing we can do now is to use a CRT super resolution.

_CRT Super Resolution?_

The idea behind resolutions like 1920x240 is that it allows quality scaling for a range of
horizontal resolutions used by different systems. Vertical resolution mismatches like 224
or 192 lines in a 240 output are resolved by centering, but horizontal resolution
mismatches have to be resolved by stretching / scaling. Stretching 256 to 320 pixels will
always look bad, but 352/320/256 pixels can be quite easily stretched to 1920 or 2048
pixels with minimal scaling artifacts, invisible on a typical consumer CRT display. Since
CRTs do not have a fixed horizontal resolution or any hard upper limit it is possible to
drive even standard definition TVs with 1920, 2048, 3840 etc. horizontal pixels.

Unfortunately, this means our output display will have a non-square pixel aspect ratio,
something RA is not specifically equipped to handle. We'll need to use the 'custom'
scaling mode and do all of the aspect ratio and centering computation ourselves.
- **240x320** -
This is typical for TATE / vertical arcade games. If we want these to display properly
we'll first have to downsize them as nicely as possible (not something RA supports
    natively, as discussed earlier) and then we'll have to center and stretch them to
maintain their correct aspect ratio. All this is further complicated by the non-square
pixel aspect ratio of our super resolution. To do this with RA's default scaling pipeline
we'd need a database of which games are TATE, their resolution and then do the math and
custom per-game setup in the `runcommand-onstart.sh` script. Or manually create a custom
per-game configuration for each game.
- **224x144 / 144x224** -
Even if we're willing to do all the manual or script development work above all of this
finally breaks down with systems like the WonderSwan which can be used both horizontally
and vertically and has some games that even switch the handheld orientation mid-game.

![Mind Blown](doc-img/mind-blown.gif?raw=true)

_A typical user trying to figure out the correct width of the image of a 3:4 game
running on a 4:3 TV with a 6:1 pixel aspect ratio_

At this point it should be abundantly clear how tedious and complicated configuring
all of this manually is. Thankfully, this shader handles all of the above without any
manual tweaking required or you ever having to get out the calculator to figure out the
correct settings.

## Setup

![Raspberry Pi](doc-img/raspberry-pi.jpg?raw=true)

These are the setup instructions for RetroPie 4.7.1 running on a series 3 Raspberry Pi.

First, we have to setup super resolution output for all emulators. Since EmulationStation
can't deal with these we want to use a standard square pixel resolution for it and only
switch to our super resolution for the actual emulators.

A good place to do so are the onstart / onend scripts. 


`/opt/retropie/configs/all/runcommand-onstart.sh`:

```
vcgencmd hdmi_timings 1920 1 79  208 241 240 1 6 10 6 0 0 0 60 0 38400000 1 > /dev/null
tvservice -e "DMT 87" > /dev/null
fbset -depth 8 && fbset -depth 16 -xres 1920 -yres 240 > /dev/null
```

`/opt/retropie/configs/all/runcommand-onend.sh`:

```
vcgencmd hdmi_timings 320 1 11 30 38 240 1 8 3 16 0 0 0 60 0 6400000 1 > /dev/null
tvservice -e "DMT 87" > /dev/null
fbset -depth 8 && fbset -depth 16 -xres 320 -yres 240 > /dev/null
```

You'll have to adjust your TV's service menu and / or the [display
timings](https://www.reddit.com/user/ErantyInt/comments/g3c98h/crtpiproject_presents_adjusting_hv_position_with/)
used here to center and size the image correctly.

Next we have to configure RetroArch scaling. Specifically, we want RA to do as little as
possible, fill out the entire screen and not filter anything. We want all the work to
be done by the shader. Here are settings that can be written to the global RA configuration.

`/opt/retropie/configs/all/retroarch.cfg`:

```
aspect_ratio_index = "23"
custom_viewport_width = "1920"
custom_viewport_height = "240"
custom_viewport_x = "0"
custom_viewport_y = "0"
video_smooth = "false"
video_scale_integer = "false"
```

You can of course also do that manually in the RGUI Settings->Video->... menus by setting
the aspect ratio to 'Custom', dialing in your chosen super resolution and disabling
integer scaling and bilinear filtering.

By default RGUI doesn't cope well with our super resolution, but we can add
`rgui_aspect_ratio_lock = "3"` or manually change
Settings->User Interface->Appearance->Lock Menu Aspect Ratio->Stretch to fix this.

Keep in mind that any per-system settings, core or game overrides might overwrite the
global settings. Please read
[this guide](https://retropie.org.uk/forum/topic/22816/guide-retroarch-system-emulator-core-and-rom-config-files)
if you're confused about how to get your settings used everywhere.

Now all that is left is to configure RA to use the shader. Clone the repository on your
Pi and use RGUI to load the shader with Quick Menu->Shaders->Load Shader Preset.
Finally, save it as a preset to be used by all systems with Save->Save Global Preset
from the same menu.

That's it, now nearly all your games should display properly without any further tweaking.

## More Setup

![RetroPie](doc-img/retropie.png?raw=true)

If you need help setting up your RetroPie CRT system in general, please see my extensive
[notes](https://github.com/blitzcode/retropie-setup-notes/blob/master/notes.txt).

They cover not only CRT and scaling specific issues but also more general things like
input lag, overclocking, USB sound cards, turbo fire, BIOS files, backups, etc.

## TODO / Limitations

- Vertical downscaling could probably use something better than the simple tent +
sharpening combination
- Using a single resolution will never correctly accommodate arcade games running at
wildly varying refresh rates and scaling on the horizontal axis will not be pixel-perfect
for most systems (not that big of a deal on a typical consumer CRT TV)
- There are going to be some unusual arcade games that have resolutions or screen
setups that are not properly handled by this shader (...but adding special case support is
straightforward)

## Code

You are encouraged to have a look at the [shader](crt-240p-scale-shader.glsl) source code. It
has less than 100 lines of actual code and is well-documented and easy to understand.
Customizing it should be very simple.

## Special Thanks

I didn't end up using either of these in my personal setup, but both the
[CRTPi Project](https://www.reddit.com/user/ErantyInt/comments/gqz3qo/crtpiproject_project_image_megathread/)
and
[Snap-Shader](https://github.com/ektgit/snap-shader-240p)
were hugely helpful when figuring out how build my own custom setup. 

## Legal

This program is published under the [MIT License](http://en.wikipedia.org/wiki/MIT_License).

## Author

Developed by Tim C. Schroeder, visit my [website](http://www.blitzcode.net) to learn more.

