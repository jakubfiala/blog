---
title: 'Optimised HTML video 101 - with transparency!'
description: 'How to encode & play it back in any browser'
pubDate: '2026-01-05'
---

<video autoplay playsinline id="video">
  <source media="(min-width: 2250px)" type="video/mp4; codecs=hvc1" src="https://search.khora.world/video/splash01.4k.h265.mp4">
  <source media="(min-width: 2250px)" type="video/mp4; codecs=av01.0.09M.08.0.110.01.01.02.0" src="https://search.khora.world/video/splash01.4k.av1.mp4">
  <source media="(min-width: 2250px)" type="video/mp4; codecs=avc1.f4.00.20" src="https://search.khora.world/video/splash01.4k.h264.mp4">
  <source media="(min-width: 2250px)" type="video/webm; codecs=vp9" src="https://search.khora.world/video/splash01.4k.vp9.webm">
  <source media="(min-width: 1500px)" type="video/mp4; codecs=hvc1" src="https://search.khora.world/video/splash01.2k.h265.mp4">
  <source media="(min-width: 1500px)" type="video/mp4; codecs=av01.0.09M.08.0.110.01.01.02.0" src="https://search.khora.world/video/splash01.2k.av1.mp4">
  <source media="(min-width: 1500px)" type="video/mp4; codecs=avc1.f4.00.20" src="https://search.khora.world/video/splash01.2k.h264.mp4">
  <source media="(min-width: 1500px)" type="video/webm; codecs=vp9" src="https://search.khora.world/video/splash01.2k.vp9.webm">
  <source media="" type="video/mp4; codecs=hvc1" src="https://search.khora.world/video/splash01.1k.h265.mp4">
  <source media="" type="video/mp4; codecs=av01.0.09M.08.0.110.01.01.02.0" src="https://search.khora.world/video/splash01.1k.av1.mp4">
  <source media="" type="video/mp4; codecs=avc1.f4.00.20" src="https://search.khora.world/video/splash01.1k.h264.mp4">
  <source media="" type="video/webm; codecs=vp9" src="https://search.khora.world/video/splash01.1k.vp9.webm">
</video>
<output id="video-source-output" style="font-family: monospace; font-size: 0.75rem;"></output>
<script is:inline type="text/javascript">
const video = document.getElementById("video");
const output = document.getElementById("video-source-output");
video.addEventListener("canplay", () => {
  console.log("canplay");
  const sources = video.childNodes;
  const currentSource = Array.from(video.childNodes).find(source => source.src === video.currentSrc);
  if (!currentSource) return;
  const [size, codec] = currentSource.src.replace("https://search.khora.world/video/splash01.", "").replace(".mp4", "").replace(".webm", "").split(".");
  output.innerText = `Your browser loaded this video in "${size}" resolution with codec "${codec}" and MIME type "${currentSource.type}"`;
}, { once: true });
</script>

Over the past year, I had the pleasure to work on [Khora](https://khora.world), a new platform for exploring philosophy. From the beginning, we wanted Khora to be as visually compelling as possible, reflecting the Platonic ideas behind it. The visuals ended up being a combination of Three.js 3D graphics, GLSL shaders (my personal favourite) and a sprinkling of short video artworks created by artist [Gareth Polmeer](https://garethpolmeer.com/).

Using HTML `<video>` for parts of the experience like the summoned "companion" figures and the background that shimmers whenever Khora is "thinking" or retrieving results was a great help, because I could directly include Gareth's artworks without having to re-interpret them in 3D. To seamlessly integrate the videos into the UI, I decided to simply render the videos with transparency. The workflow was as follows:

1. The artist produces an animation as a sequence of high-resolution PNG frames (with an alpha channel if needed)
2. I use `ffmpeg` to combine the frames into a video and encode it to play back with transparency.
3. I add it to the app as a `<video>` element with multiple `<source>`s.
4. I upload the video files to our CDN and the app pulls them from there.

A big downside of video on the Web is that videos first have to be loaded over the network and decoded before playing. For a search engine-like app like Khora, I found it extremely important for this to be as efficient as possible. There's nothing worse than opening a website half-broken because a 30MB video must first load for anything to work properly (see lots of flashy auto maker sites). That may be acceptable for a marketing site, but not for a learning app. Integrating video into Khora, I set out a few goals:

* load the video in the **most appropriate resolution**
* use the **most efficient encoding available** on any given device
* **support transparency** on all major browser versions where transparent video is possible

After a lot of revisions (mostly triggered by testing the app on some lovely colleagues' ancient Safari installations), I managed to find an optimal configuration which should work on major browsers released after 2018. In the rest of the article, I describe how I addressed each of the above goals.

## Choosing the appropriate resolution

This objective was the most straightforward - using the HTML `<source>` element and the `media` attribute, the browser will load the video from a different URL based on a [CSS media query](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Media_queries). We just need to choose a reasonable set of resolutions. 

I decided to use width-based media queries. Given most devices with small screens are phones with high-DPI screens and require double or triple the pixels to look sharp enough, I chose the lowest resolution to be **1k** or 1280x720 (in the olden days, this would be called [HD-ready](https://en.wikipedia.org/wiki/HD_ready)). That way, a 390px-wide browser view on an iPhone 13 will get 3 video pixels per logical pixel, covering the screen's pixel density of 3. For tablets and laptops which are viewed from further away and the DPI doesn't matter as much, I added **2k** or 1920x1080 (or Full HD). Lastly, for large desktop screens I added a **4k** version, or 3840x2160.

> It would of course be even more optimal to use `resolution` in the media queries, add a 3k version etc. My current resolution is a starting point that can be adjusted based on real performance measurements. However, with efficient codecs the video files tend to be very small already, and it's worth avoiding extra network requests if the available resolution is good enough.

I started by writing a bash script, which runs the following `ffmpeg` commands for each frame sequence. Using `-1` as video height ensures the aspect ratio isn't affected if the video isn't 16:9.

```sh
# run this in a directory with sequentially ordered PNG files
$name="my-video"

ffmpeg -framerate 60 -pattern_type glob -i '*.png' -vf 'scale=1280:-1' "$name.1k.webm"
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -vf 'scale=1920:-1' "$name.2k.webm"
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -vf 'scale=3840:-1' "$name.4k.webm"
```

In my web app, I added a mapping between media queries and resolutions. Then I used the map to create a set of `<source>` elements for the video.

```tsx
const sizesMediaMap = {
  "(min-width: 2250px)": "4k",
  "(min-width: 1500px)": "2k",
  "": "1k",
};

export default function MyVideo() {
  return (
  <video autoPlay muted playsInline>
    {Object.entries(sizesMediaMap).map(([media, size]) => (
        <source
          key={size}
          media={media}
          src={`/video/my-video.${size}.webm`}
        />
      ),
    )}
  </video>
  );
};
```

Now the browser will select the appropriate resolution and load the correct video dynamically.

## Choosing the most efficient codec

When choosing the video encoding for a Web-based video, we have to consider both **browser support** and the **resulting file size**. Fortunately, HTML video implementations will gracefully fall back to the first `<source>` that's actually playable on the given system, so if we include a widely supported codec such as H264 or VP9 (WebM), users will always be able to play our video. For newer browsers, we can progressively enhance and allow them to access the most efficient codec first, ensuring most of our users will have to load the smallest possible video file.

There are currently two well-supported and efficient video codecs available in different browsers. **AV1** is fully supported by Chromium browsers and Firefox, while **HEVC/H265** has good support on Safari since 2017 and other browsers partially support it, too. Let's look at how we can encode AV1 + H265 versions of our frame sequence.

### AV1

On my Ubuntu system, I initially installed `ffmpeg` version 7 using `apt`. According to [the documentation](https://trac.ffmpeg.org/wiki/Encode/AV1), `ffmpeg` supports two AV1 encoding libraries: `libaom` and `libsvtav1`. I decided to go with SVT-AV1 and MP4 container format, resulting in the following ffmpeg commands:

```sh
# AV1 encoding
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -c:v libsvtav1 -vf 'scale=1280:-1' "$name.1k.av1.mp4"
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -c:v libsvtav1 -vf 'scale=1920:-1' "$name.2k.av1.mp4"
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -c:v libsvtav1 -vf 'scale=3840:-1' "$name.4k.av1.mp4"
```

With AV1, we already get a remarkable size reduction - for a 2k (1080p) video I used, I got:

* **VP9**: 3.7 MB
* **H264**: 2.1 MB
* **AV1**: 905 kB

### H265

Since AV1 doesn't have great support in Safari, we also have to encode as H265 (a.k.a HEVC). For this, we can use the `libx265` encoder and MP4 containers:

```sh
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -c:v libx265 -vf 'scale=1280:-1' "$name.1k.av1.mp4"
```

The resulting video opens nicely in VLC and I can even play it in Firefox on my machine. It's also really small, only **530 kB** compared to AV1's 905 kB. However, when I opened the file in Safari, the video was broken. After a *lot* of research I came across a forum thread suggesting that for H265 video to work in Safari, the MP4 file needs to contain a so called ["four-character code" tag](https://ithy.com/article/ffmpeg-tag-mp4-oy3zmaah) telling the browser which codec is being used. For H265, this code turns out to be `hvc1`. Let's add it to our commands:

```sh
# H265 encoding
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -c:v libx265 -tag:v hvc1 -vf 'scale=1280:-1' "$name.1k.av1.mp4"
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -c:v libx265 -tag:v hvc1 -vf 'scale=1920:-1' "$name.2k.av1.mp4"
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -c:v libx265 -tag:v hvc1 -vf 'scale=3840:-1' "$name.4k.av1.mp4"
```

This produces a *tiny* video file playable in Safari.

### Fallback codecs

To ensure playability even in older browsers, I also exported each sequence in WebM (with the VP9 codec) and H264:

```sh
# WebM/VP9
ffmpeg8 -framerate 60 -pattern_type glob -i '*.png' -vf 'scale=1280:-1' "$name.1k.vp9.webm"
ffmpeg8 -framerate 60 -pattern_type glob -i '*.png' -vf 'scale=1920:-1' "$name.2k.vp9.webm"
ffmpeg8 -framerate 60 -pattern_type glob -i '*.png' -vf 'scale=3840:-1' "$name.4k.vp9.webm"
# H264
ffmpeg8 -framerate 60 -pattern_type glob -i '*.png' -vf 'scale=1280:-1' "$name.1k.h264.mp4"
ffmpeg8 -framerate 60 -pattern_type glob -i '*.png' -vf 'scale=1920:-1' "$name.2k.h264.mp4"
ffmpeg8 -framerate 60 -pattern_type glob -i '*.png' -vf 'scale=3840:-1' "$name.4k.h264.mp4"
```

### Declaring encodings in HTML

Now we have the video in all necessary codecs and sizes, we can add the appropriate `<source>` elements. We can use the `type` attribute
to specify the codec for each `<source>`, so that the browser can choose the right file to play without having to load and analyse all of them. I was disappointed to find out the format for the `type` attribute isn't that well [documented on MDN](https://developer.mozilla.org/en-US/docs/Web/API/HTMLSourceElement/type). This is because while video *container* formats can be specified as a simple MIME type like `video/webm` or `video/mp4`, the codec itself has to be a very specific, non-obvious string that includes various details about pixel format etc. For instance, the correct codec string for my AV1 videos is `av01.0.09M.08.0.110.01.01.02.0` 🤷

Fortunately, I stumbled upon [this fantastic Python script](https://gist.github.com/Steve-Tech/0f0e37785f3148b03fdb4ab96e4d27ff) by [Stephen Horvath](https://gist.github.com/Steve-Tech) which extracts the correct MIME type string from a video file (using `ffmpeg` and `ffprobe` under the hood). Then I could create a mapping from codec names to MIME types:

```tsx
const sizesMediaMap = {
  "(min-width: 2250px)": "4k",
  "(min-width: 1500px)": "2k",
  "": "1k",
};

const codecMap = {
  h265: "hvc1",
  av1: "av01.0.09M.08.0.110.01.01.02.0",
  h264: "avc1.f4.00.20",
  vp9: "vp9",
};

export default function MyVideo() {
  return (
    <video autoPlay muted playsInline>
      {Object.entries(sizesMediaMap).map(([media, size]) =>
        Object.entries(codecMap).map(([name, codec]) => {
          // VP9 comes in a .webm container, all others come in MP4
          const container = name === "vp9" ? "webm" : "mp4";
          return (
            <source
              key={size + name}
              media={media}
              type={`video/${container}; codecs=${codec}`}
            src={`/video/my-video.${size}.${name}.${container}`}
            />
          );
        }),
      )}
    </video>
  );
};
```

Now each browser will choose the appropriate resolution *and* encoding for the video. For a video that can be played at up to 4k, we can easily reduce the actually loaded file size 15-20x just by choosing the right resolution and codec.

## Transparent video

The last requirement was to display videos with transparency, overlaid on top of other parts of the app. Not all codecs even support transparency (i.e. an alpha channel). Specifically, H264 and AV1 are out of the picture completely, and while as [Jake Archibald](https://jakearchibald.com/2024/video-with-transparency/) points out, animated AVIF images are basically AV1, they do not work on Safari and perform poorly on other browsers. That means we're left with VP9/WebM and H265.

For the transparent videos in Khora, I first simply stuck to VP9 and called it a day. This does not require any changes to the `ffmpeg` command - if the PNGs are transparent, then the WebM video will be transparent. However, some time later my lovely Mac user colleagues pointed out that on *their* machine, the video [had a black background](https://bugs.webkit.org/show_bug.cgi?id=275908). While Safari does support VP9 video, it doesn't use the alpha channel.

H265 to the rescue, I thought! After all, Jake Archibald had good results with it. To enable transparency with the `libx265` encoder, we can simply specify the pixel format:

```sh
# H265 with transparency
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -c:v libx265 -tag:v hvc1 -pix_fmt yuva420p -vf "scale=1280:-1" "$name.h265.1k.mp4"
```

Except, no - `ffmpeg` errors out and claims that `[libx265 @ 00000126451b7f80] Loaded libx265 does not support alpha layer encoding.`. After some more digging, I found out this [was a known issue](https://github.com/UniversalMediaServer/UniversalMediaServer/issues/5486) and was only [patched](https://www.mail-archive.com/ffmpeg-cvslog@ffmpeg.org/msg65032.html) in `ffmpeg` version 8. Sadly, this version isn't available via `apt` yet, so I had to [download a static build of ffmpeg 8](https://github.com/BtbN/FFmpeg-Builds/releases). Running the command with that finally resulted in a H265 video playable on Safari **with transparency**.

### One last hurdle

I finally sat back and enjoyed the beautiful transparent videos looping around in Khora, until we decided to put in another one - this time, it was a "splash screen" with a solid black background, no alpha channel. I ran it through my scripts and plopped it into the app, only to get a message from a colleague saying something about a green tint all over the splash screen.

Well, it turns out as I exported the H265 video with `ffmpeg 8` using the original script for non-transparent video, the resulting H265 file would play just fine on VLC and in Firefox, but in Safari, all black pixels were green. Another bunch of digging later, I tried exporting the same video using my transparent video script. It worked! Turns out, for some reason Safari now needed the pixel format of H265 to be `yuva420p` even if the video itself didn't contain an alpha channel.

## Conclusion

I hope you enjoyed following my journey making efficient transparent video playback work across browsers. Here's the final script I use to export the videos (`ffmpeg` version 8 required!):

```sh
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -c:v libx265 -tag:v hvc1 -c:v libx265 -pix_fmt yuva420p -vf 'scale=1280:-1' "$name.1k.h265.mp4"
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -vf 'scale=1280:-1' "$name.1k.h264.mp4"
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -vf 'scale=1280:-1' "$name.1k.vp9.webm"
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -c:v libsvtav1 -vf 'scale=1280:-1' "$name.1k.av1.mp4"
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -c:v libx265 -tag:v hvc1 -c:v libx265 -pix_fmt yuva420p -vf 'scale=1920:-1' "$name.2k.h265.mp4"
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -vf 'scale=1920:-1' "$name.2k.h264.mp4"
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -vf 'scale=1920:-1' "$name.2k.vp9.webm"
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -c:v libsvtav1 -vf 'scale=1920:-1' "$name.2k.av1.mp4"
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -c:v libx265 -tag:v hvc1 -c:v libx265 -pix_fmt yuva420p "$name.4k.h265.mp4"
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -vf 'scale=3840:-1' "$name.4k.h264.mp4"
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -vf 'scale=3840:-1' "$name.4k.vp9.webm"
ffmpeg -framerate 60 -pattern_type glob -i '*.png' -vf 'scale=3840:-1' -c:v libsvtav1 "$name.4k.av1.mp4"
```
