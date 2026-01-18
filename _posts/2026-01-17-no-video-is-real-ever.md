---
layout: post
title: "No Video Is Real Ever"
params:
    license: CC-BY-4.0
date: 2026-01-17
permalink: /no-video-is-real-ever.html
custom_js:
    - vega-6
    - vega-lite-6
    - vega-embed-6
---

It's a pasttime within *video-related* circles
to argue over video codec parameters for transcoding
high-quality/lossless content
down to sizes that mere mortals can actually store and share.
Here's some fuel for that fire.

# Why?

Within the fine art of SVT-AV1 transcoding,
there are two answers to "what CRF do I use?":
"it depends", and "38".

Thing is, while CRF 38 looks "good enough"
on frame sequences like `park_joy`,
I had a hunch for a while that
this is not true of all footage.
In my personal experience, certain types of footage
seem to have held up better at CRF 38 than others;
clearly, CRF 38 is not the universal "good enough" value.

What people tend to use for testing is not what people encode.
Does a deep-fried meme encode the same as the `park_joy` sequence?
Does a talking head encode the same as an action scene?
Does anime encode the same as live-action?
Is web video the same as traditionally-published video?

I figured "hey, what if I just did it myself?".
It turns out that "it depends" is doing so much heavy lifting
it's triumphantly sinking into the Earth's core.

# CRF is non-Constant

That's a great phrase, but I would like to be more precise.
I believe there are characteristics of some videos that throw off SVT-AV1's CRF,
that cause it to correspond less accurately to "real" quality than
everyone tells you CRF does.

<aside markdown="1">
CRF is the "constant quality"-based bitrate control method
that most codecs provide a version of.
You'll see a thing called "CRF" in x264, x265, libvpx, and so on.
If you have no particular need to control bitrate, CRF
tends to be what people recommend to produce a fixed quality.

It's an arbitrary unit, where a higher CRF corresponds to worse quality,
but otherwise the rules are arbitrary.
It's said that you need to give

* x264 a CRF of 18 for visual losslessness,

* x265 a CRF of 23, and

* SVT-AV1 a CRF of 30-ish.

But all these numbers are debatable depending on the exact source you're encoding.
</aside>

Did I watch hundreds of near-identical copies
of the same video to ascertain Quality from un-Quality?
As funny as it would be to say "yes", I must admit: no.
Metrics to estimate video quality
(both "objective" and "subjective")
are an active field of research.
There are plenty of metrics out there, but let's plot only one:
VMAF, by our new best friends at Netflix.

VMAF is an "arbitrary unit" sorta deal,
it doesn't mean anything physically
(in the way that PSNR relates to signal-to-noise ratio),
but I've heard most say that VMAF 95 is "fine",
and *I* think VMAF 92 is "tolerable".

Before you get too confused by the visualizations that follow:

* the x-axis is encoder preset; the higher, the faster the encoder,

* the y-axis is CRF; nominally, the lower, the better the quality,

* the colors are VMAF, "real" quality
  (green is "too high", red "too low", yellow "enough"),

* the whitened cells are to highlight VMAF 95 ± 0.5 (higher) and VMAF 92 ± 0.5 (lower),
  where the intended effect is something like a height contour on a map,

* the numbers on the whitened cells are bitrate, to help answer the question
  "at VMAF 95 and 92, what kind of bitrates do I get?".

* there's a drop-down for resolution,
  though I only had the patience to do 480p and 720p for most of these.

If you're on desktop, you can hover over the cells to get more precise numbers.

I don't really have a unified theory,
this is basically a gallery of case studies,
but I hope the examples here mean something.

# [park_joy](https://media.xiph.org/)

If you see suggestions like "CRF 38 is enough", this is why.
I would expect that video codec research
congregates around these frame sequences.
They've been teaching to the test, as it were.

<viz id="parkjoy-viz"></viz>

# '[ANTHONY FANTANO THICC](https://www.youtube.com/watch?v=HMKUlsJpov8)' by fantano

Before I show a video where VMAF 95 is harder to attain, here's
one where >95 is so attainable that you'd have to be *trying* to go lower.

Low-quality sources tend to make VMAF 95/92 much easier to reach;
"forgiving", as it were.

<viz id="damn-boi-viz"></viz>

# '[I played National Park from Pokemon Gold/Silver at the park](https://www.youtube.com/watch?v=UqA1QsvKwgw)' by Jordan D Piano

Here we go. If you went with CRF 38, you'd get VMAF **92** at Preset 8 and lower,
and you'd go below *that* bar at Preset 9 and 10.

Detail (in this case, foliage) means you have to use lower CRFs to reach
VMAF 95/92 ("less forgiving"), looks like.

You might've noticed it at park_joy, but notice how Preset 9 and 10
produce a "jump" in the VMAF curves,
that's to say, you get substantially worse quality for the same CRF.
it seems like the fastest worthwhile Preset is 8.

`film-grain` might be useful in these situations.

<viz id="national-park-viz"></viz>

# '[Trophy](https://www.youtube.com/watch?v=_T9uppDEzc8)' by Crumb

Low-light seems to also produce less forgiving videos.
Maybe variance boost can help here?

<viz id="trophy-viz"></viz>

# '[Human](https://www.youtube.com/watch?v=hbuNxmAXdNk)' by FLAVOR FOLEY

This one juxtaposes animation and static elements
against live-action footage.
Relatively forgiving.

<viz id="human-viz"></viz>

# '[BAD PIGGIES](https://www.youtube.com/watch?v=wirwlJo9fYY)' by InstrumentManiac

It seems like rapid cuts between scenes tends to produce very "forgiving" video.

<viz id="bad-piggies-viz"></viz>

# '[meow mix](https://www.youtube.com/watch?v=4lBvkbtBYjU)' by meowballz

I expected this video to be a lot less forgiving,
but maybe it triggers frequent intra refreshes or something.

<viz id="meow-mix-viz"></viz>

## Experimental setup

For reproducibility,
and because web video is more diverse than traditional video,
all of the videos tested are from YouTube.
They were downloaded with `yt-dlp` with cookies from a YouTube Premium account.
All source videos are the best quality available at download-time.

Testing on lossy encodes is *questionable*,
but YouTube definitely overestimates bitrates
(especially at 1080p and higher).
I'm "probably" fine here.

I wrote Vega-Lite specifications to generate the visualizations
for this article, and used Vega-Embed to embed said visualizations.

I ran CachyOS's FFmpeg and SVT-AV1 c.a. 15th of January 2026.

Relevant output:

```
$ ffmpeg -version
ffmpeg version n8.0.1 Copyright (c) 2000-2025 the FFmpeg developers
built with gcc 15.2.1 (GCC) 20251112
configuration: --prefix=/usr --disable-debug --disable-static --disable-stripping --enable-amf --enable-avisynth --enable-cuda-llvm --enable-lto --enable-fontconfig --enable-frei0r --enable-gmp --enable-gnutls --enable-gpl --enable-ladspa --enable-libaom --enable-libass --enable-libbluray --enable-libbs2b --enable-libdav1d --enable-libdrm --enable-libdvdnav --enable-libdvdread --enable-libfreetype --enable-libfribidi --enable-libglslang --enable-libgsm --enable-libharfbuzz --enable-libiec61883 --enable-libjack --enable-libjxl --enable-libmodplug --enable-libmp3lame --enable-libopencore_amrnb --enable-libopencore_amrwb --enable-libopenjpeg --enable-libopenmpt --enable-libopus --enable-libplacebo --enable-libpulse --enable-librav1e --enable-librsvg --enable-librubberband --enable-libsnappy --enable-libsoxr --enable-libspeex --enable-libsrt --enable-libssh --enable-libsvtav1 --enable-libtheora --enable-libv4l2 --enable-libvidstab --enable-libvmaf --enable-libvorbis --enable-libvpl --enable-libvpx --enable-libwebp --enable-libx264 --enable-libx265 --enable-libxcb --enable-libxml2 --enable-libxvid --enable-libzimg --enable-libzmq --enable-nvdec --enable-nvenc --enable-opencl --enable-opengl --enable-shared --enable-vapoursynth --enable-version3 --enable-vulkan
libavutil      60.  8.100 / 60.  8.100
libavcodec     62. 11.100 / 62. 11.100
libavformat    62.  3.100 / 62.  3.100
libavdevice    62.  1.100 / 62.  1.100
libavfilter    11.  4.100 / 11.  4.100
libswscale      9.  1.100 /  9.  1.100
libswresample   6.  1.100 /  6.  1.100

Exiting with exit code 0
```

```
$ SvtAv1EncApp
Svt[info]: -------------------------------------------
Svt[info]: SVT [version]:       SVT-AV1 Encoder Lib v3.1.2-dirty
Svt[info]: SVT [build]  :       GCC 15.2.1 20250813      64 bit
Svt[info]: LIB Build date: Aug 28 2025 13:59:13
Svt[info]: -------------------------------------------
```


Have the scripts I used for this:

<pre-header tag="script">run-svtav1-bench.sh</pre-header>
```bash
#!/usr/bin/env bash
# author: multiplealiases, 2026
# SPDX-License-Identifier: CC0-1.0
# Runs a single (CRF, preset, res) data point.
# run as ./run-svtav1-bench.sh file.mkv $crf $preset $res

set -o nounset

file="$1"
crf="$2"
preset="$3"
res="$4"

output=tc-crf"$crf"-preset"$preset"-"$res"p.mkv

# -benchmark shows times,
# -map_* prevents metadata from throwing off bitrate readings,
# -an because I don't need the audio track,
# I'm using 10-bit because why not,
# and the filtergraph basically means
# "downscale to $res if the source res is higher"
ffmpeg -nostats -hide_banner -benchmark -i "$file" -map_metadata -1 -map_chapters -1 -an -c:v libsvtav1 -crf "$crf" -preset "$preset" -/filter_complex /dev/stdin -pix_fmt yuv420p10le "$output" -y << EOF |& tee log/"$output".log
scale='if(gte(ih, iw), min($res, iw), -4)':'if(lt(ih, iw), min($res, ih), -4)
EOF

# this is actually incorrect:
# VMAF docs say you should scale ref and distorted to 1080p.
# VMAF is more forgiving if you use a lower res than intended.
ffmpeg -nostats -hide_banner -i "$file" -i "$output" -map_metadata -1 -map_chapters -1 -filter_complex:v '[0][1] scale2ref [orig][compressed]; [compressed][orig] libvmaf=n_threads=8' -f null AAA |& tee -a log/"$output".log
```

<pre-header tag="script">runall.sh</pre-header>
```bash
#!/usr/bin/env bash
# author: multiplealiases, 2026
# SPDX-License-Identifier: CC0-1.0

# Runs all combinations shown in the article.
parallel -q -j1 --lb ./run-svtav1-bench "$1" {1} {2} {3} ::: 18 50 23 28 35 38 42 25 32 36 40 19 20 21 22 24 26 27 29 30 31 33 34 37 39 41 43 44 45 46 47 48 49 ::: {4..10} ::: 480

parallel -q -j1 --lb ./run-svtav1-bench "$1" {1} {2} {3} ::: 18 50 23 28 35 38 42 25 32 36 40 19 20 21 22 24 26 27 29 30 31 33 34 37 39 41 43 44 45 46 47 48 49 ::: {4..10} ::: 720

parallel -q -j1 --lb ./run-svtav1-bench "$1" {1} {2} {3} ::: 18 50 23 28 35 38 42 25 32 36 40 19 20 21 22 24 26 27 29 30 31 33 34 37 39 41 43 44 45 46 47 48 49 ::: {4..10} ::: 1080
```

Place these files into a new directory, and make `log/` under it.

<pre-header tag="script">log/processdata.sh</pre-header>
```bash
#!/usr/bin/env bash
# author: multiplealiases, 2026
# SPDX-License-Identifier: CC0-1.0
# Produces CSVs from the raw logs in the current working dir.
cat <(echo crf,preset,res,bitrate,vmaf,time) <(grep -E -R . -e 'bitrate: [0-9]+ kb/s' -e 'rtime=[0-9]*\.[0-9]*s' -e 'VMAF score: [0-9]*\.[0-9]*' -o | pcre2grep --om-separator ',' -M -o1 -o2 -o3 -o5 -o6 -o4 '\.\/tc-crf([0-9]+)-preset([0-9]+)-([0-9]+)p.*rtime=([0-9]+\.[0-9]+)s\n.+\n.+bitrate: ([0-9]+) kb\/s\n.+VMAF score: ([0-9]+.[0-9]+)')
```

Excuse the severe jank.

# Copyright stuff

To minimize confusion (and because these file formats don't accept comments),
I'm putting the following files (data and Vega-Lite specifications)
under Creative Commons Zero v1.0 Universal.

If I may dunk a little bit:
I wrote the Vega-Lite specs entirely using my brain, my two hands, and the Vega-Lite docs.
**Zero** AI was involved in the creation of these specs.
No chatbots were passed a prompt, nor were any websites ending in `.ai` looked at.
They should be clean with respect to copyright,
because **I** wrote them.

* [parkjoy.csv](/assets/no-video-is-real-ever/parkjoy.csv), [parkjoy.vl.json](/assets/no-video-is-real-ever/parkjoy.vl.json)

* [damn-boi.csv](/assets/no-video-is-real-ever/damn-boi.csv), [damn-boi.vl.json](/assets/no-video-is-real-ever/damn-boi.vl.json)

* [national-park.csv](/assets/no-video-is-real-ever/national-park.csv), [national-park.vl.json](/assets/no-video-is-real-ever/national-park.vl.json)

* [meow-mix.csv](/assets/no-video-is-real-ever/meow-mix.csv), [meow-mix.vl.json](/assets/no-video-is-real-ever/meow-mix.vl.json)

* [bad-piggies.csv](/assets/no-video-is-real-ever/bad-piggies.csv), [bad-piggies.vl.json](/assets/no-video-is-real-ever/bad-piggies.vl.json)

* [trophy.csv](/assets/no-video-is-real-ever/trophy.csv), [trophy.vl.json](/assets/no-video-is-real-ever/trophy.vl.json)

* [human.csv](/assets/no-video-is-real-ever/human.csv), [human.vl.json](/assets/no-video-is-real-ever/human.vl.json)

<script type="text/javascript">
  var spec = '/assets/no-video-is-real-ever/national-park.vl.json';
  vegaEmbed('#national-park-viz', spec)
</script>

<script type="text/javascript">
  var spec = '/assets/no-video-is-real-ever/damn-boi.vl.json';
  vegaEmbed('#damn-boi-viz', spec)
</script>

<script type="text/javascript">
  var spec = '/assets/no-video-is-real-ever/parkjoy.vl.json';
  vegaEmbed('#parkjoy-viz', spec)
</script>

<script type="text/javascript">
  var spec = '/assets/no-video-is-real-ever/bad-piggies.vl.json';
  vegaEmbed('#bad-piggies-viz', spec)
</script>

<script type="text/javascript">
  var spec = '/assets/no-video-is-real-ever/meow-mix.vl.json';
  vegaEmbed('#meow-mix-viz', spec)
</script>

<script type="text/javascript">
  var spec = '/assets/no-video-is-real-ever/trophy.vl.json';
  vegaEmbed('#trophy-viz', spec)
</script>

<script type="text/javascript">
  var spec = '/assets/no-video-is-real-ever/human.vl.json';
  vegaEmbed('#human-viz', spec)
</script>
