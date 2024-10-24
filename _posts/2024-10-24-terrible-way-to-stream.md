---
layout: post
title: A Terrible Way To Stream (using SSH and FFmpeg)
date: 2024-10-24
params:
    license: CC-BY-4.0
permalink: /terrible-way-to-stream.html
---

A pair of commands.

```sh
ssh user@host '
    ffmpeg -i "$file" -f matroska -map 0:s -map 0:d - \
 ' > /tmp/subs.mkv

ssh user@host '
    ffmpeg -i "$file" -map 0:a:1 -map 0:v:0 -f matroska \
    -b:a 128k -c:v libx264 -b:v 5M -preset fast - \
 ' | mpv - --sub-file /tmp/subs.mkv --demuxer-max-bytes=512M
```

And a lot of explanation.

1. SSH connects the remote machine's standard I/O streams to your machine's.
   This can be used for invocations like
   `ssh user@host 'tar -c $path' > backup.tar`[^1]
    to back up a remote machine's files and stream it to the local machine,
    or, more ambitiously,
   `ssh user@host 'tar -c $path' | tar -x` for a bootleg `scp` or `rsync`.

2. FFmpeg supports streamable output formats,
   the most useful ones being NUT (`-f nut`) and Matroska/MKV (`-f matroska`),
   which you can use to dump media into a media player through its stdin
   by specifying an output file of `-` or `/dev/stdin`[^2].

3. While you can pipe it into `ffplay -`,
   it doesn't cache enough to allow seeking.
   This is where mpv comes in.
   You can set `--demuxer-max-bytes` to set the size of its cache,
   allowing you to seek, at least a little bit.

But there's a catch, two, actually!

While audio and video tracks are streamable,
subtitles and extra data (typically fonts) aren't.
What you need to do is run an initial pass
to fetch all subs (`-map 0:s`) and all extra data (`-map 0:d`) to
a file on your computer, in this case `/tmp/subs.mkv`
You also need to specify `-f matroska`
because the remote command is just writing to stdout,
which FFmpeg does not assume any output format for,
and your local machine is redirecting it to a file.

You also can't switch audio tracks if you stream like this.
Hence, you need to do it in the second command itself.

I have no good solution for this.
Run `ffprobe` on the media file, on the remote side if need be,
and look at the streams.

Here's some output from some media I had on hand:

```
  Stream #0:0: Video: av1 (libdav1d) (Main), yuv420p10le(tv), 1520x1080, SAR 1:1 DAR 38:27, 23.98 fps, 23.98 tbr, 1k tbn (default)
  Stream #0:1(jpn): Audio: opus, 48000 Hz, stereo, fltp (default)
      Metadata:
        title           : 2.0 Japanese
  Stream #0:2(eng): Audio: opus, 48000 Hz, stereo, fltp
      Metadata:
        title           : 2.0 English
```

Ignore the numbering `ffprobe` gives you
and follow the stream[^3]
types counting from 0,
like so:

* There's only 1 video stream, so it's video stream #0, addressed by `-map 0:v:0`.

* There are 2 audio streams, the first (`-map 0:a:0`) is Japanese,
  and the second (`-map 0:a:1`) is English.

Let's say you want English audio and (trivially) the only video stream.
You would now write `-map 0:a:1 -map 0:v:0`
to tell FFmpeg to select those streams, after the `-i $file` but before
the actual output options, like so:

<pre>
    ffmpeg -i $file -map <span style="color:red">0:</span><span style="color:green">a:1</span> -map <span style="color:red">0:</span><span style="color:blue">v:0</span> -f matroska \
</pre>

You need to pass `-f matroska` for the same reason as stated above:
FFmpeg does not assume any output format if it's to standard output.
This effectively means
"from the <span style="color:red">first file</span>,
select <span style="color:green">audio stream #1</span> and <span style="color:blue">video stream #0</span>
out of the inputs".
If we had more inputs,
they would've been addressed with `-map 1:`, `-map 2:`, and so on.

You can then add output args.

```
    -c:a aac -b:a 128k -c:v libx264 -b:v 5M -preset fast -
```

This means "encode the audio track(s) with the `aac` encoder
at a target bitrate of 128 kilobits per second,
encode the video tracks using the `libx264` encoder, at an
average bitrate[^4] of 5 Mbit/s
using its `fast` preset,
and then push the output through standard output"

You could also do `-c:a copy -c:v copy`
if you know your network can handle
streaming the file at the original bitrate
and your local machine is capable of decoding the original codecs.

Then pass that output into `mpv`.

```
| mpv - --sub-file /tmp/subs.mkv --demuxer-max-bytes=512M
```

This is simple enough:
play (on the local machine) whatever comes into standard input (`-`),
using the subtitles contained in `/tmp/subs.mkv`,
using a max cache size of 512 megabytes
(which gives you a decent amount of buffer to seek around in).

Congratulations, you're now streaming media over SSH,
transcoding on the remote machine!
Someone should write a better interface for this.

And maybe improve on the "can't really seek" problem,
since it's not possible for `mpv` to ask the remote machine to encode
'*that* part' of the file again in the event the user seeks to a
position in the file that's no longer in cache.

I realize what I'm asking for is close to what Plex does,
but I'd like one that wasn't a do-it-all solution with media cataloging,
and closer to a minimalist remote video streaming app.

# Footnotes

[^1]: Notice the lack of an `-f`!
      If you don't specify `-f`, `tar` writes output to standard out.

[^2]: Both are equivalent on Unix-likes,
      but the device node `/dev/stdin` doesn't exist on Windows.

[^3]: What other programs might call a "track".

[^4]: ABR (or target bitrate) mode
      [isn't recommended](https://slhck.info/video/2017/03/01/rate-control.html#average-bitrate-abr-also-target-bitrate),
      it's usually a bad mode to use,
      but supposing you don't want to do 2-pass encoding,
      I believe CRF (variable-bitrate) and CRF+VBV (requires some fiddling)
      would be your only other options, anyway.
