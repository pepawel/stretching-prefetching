# Stretching Google's Prefetching Demo

This is a demonstration of how large the prefetched data can be when using Signed Exchanges (SXG).

[![Watch the video](https://img.youtube.com/vi/6cixE1x5AeU/maxresdefault.jpg)](https://www.youtube.com/watch?v=6cixE1x5AeU)

You can experience the [live demo](https://www.planujemywesele.pl/sxg-tests/offline-abuse), read a
[blog post](https://www.pawelpokrywka.com/p/stretching-google-prefetching) about it,
or deploy the demo yourself to your website (described later).

## How It Works

The `instructions.html` file guides the user to find the `show.html` page in Google
and then distracts them long enough for the data to be prefetched.

The `show.html` contains the actual demo. It includes tags for prefetching the video on Google search results using SXG.
To understand SXG, check out my [deep-dive](https://www.pawelpokrywka.com/p/how-i-took-lcp-down-under-350ms).

The HTML is composed of three pages, with only one being visible at a time.
Right after the user visits the page, the embedded script decides which page to display based on the user's online status.

Online status is tested using both the browser API and network requests, because in some cases
the browser API is unreliable (such as when using a firewall or disconnecting the WAN cable from the router).

### The Online Page

This page is displayed when the user didn't follow instructions and visited the page while being online.
It's also for Googlebot to index, which allows the user to find the page in Google Search.

You may notice the page contains some *filler content* about being offline. This is required because otherwise, the page wouldn't have
enough content, and Google would treat it as a so-called *soft-404* and reject indexing it.

### The Offline Page

If the page was prefetched, the user went offline, and then visited it, the offline page is presented.
It's a copy of the default Chrome offline page to give the impression that the page didn't load.

But it did load, and the script tests if the SXG subresources were correctly prefetched. If the user didn't wait long enough,
some of the subresources may be missing and the demo will fail. In this case, the script stops and the user is left
with the offline page.

If the subresources were correctly loaded, the script displays a speech bubble near the dinosaur with a link.
The speech bubble blinks, as many people wouldn't notice it otherwise.

### The Video Page

This page appears when the link is clicked. The script starts to concatenate the prefetched video parts. This process takes a lot of RAM
and may fail if the user's device is low on memory. The resulting video data is converted to a blob,
placed as an `src` attribute of a video element, and the video is played.

The reason for the dinosaur page is to force the user to click. Otherwise, the browser would mute the video, taking the
fun out of the demo.

## FAQ

### Where does the SXG prefetching happen?

Look for `<link rel="preload">` tags in the `show.html`. They are enclosed in `<template data-sxg-only>`
so that the prefetching happens only for an SXG visit. That's because they are not used otherwise,
so prefetching them wouldn't make sense.

To learn more about prefetching subresources,
read [my blog post](https://www.pawelpokrywka.com/p/subresources-prefetching-with-signed-exchanges).

### Why is the video split into parts?

[Google won't cache SXG files larger than 1044 KB](https://www.pawelpokrywka.com/p/other-errors-with-signed-exchanges),
therefore the video has to be split into parts.

### Why is the .gif extension used for parts?

Google won't cache SXG files with unpopular extensions.

### How does the page know if the subresources were prefetched?

It uses [SXG Status](https://github.com/pepawel/sxg-status), my other project implementing the technique I discovered.

### Can I use another video or something entirely different?

You can use any file that's no larger than the limit. Potential ideas:

- A very large picture
- A compressed archive containing an entire web application, uncompressed later by the browser
- An executable for the user to download

## Installation & Usage

To deploy the demo to your server, follow these steps:

### Download a Video

You'll need a 19 MB video. The demo won't work for smaller files without adjustments, and larger files will most
likely exceed the SXG limit.

To download the clip used in the original demo,
install [yt-dlp](https://github.com/yt-dlp/yt-dlp) and execute in the shell:

```shell
yt-dlp -f 609+251 --merge-output-format mp4 -o rick.mp4 'https://www.youtube.com/watch?v=dQw4w9WgXcQ'
```

If the command fails or the resulting `rick.mp4` is a different size than required (~19 MB),
you'll need to check available formats by executing:

```shell
yt-dlp -F 'https://www.youtube.com/watch?v=dQw4w9WgXcQ'
```

Find video and audio formats that together total around 19 MB and pass their IDs
to the `-f` option instead of `609+251`. For example, if the video is 42 and audio is 23, execute:

```shell
yt-dlp -f 42+23 --merge-output-format mp4 -o rick.mp4 'https://www.youtube.com/watch?v=dQw4w9WgXcQ'
```

If the video size is about 19 MB, you can proceed to the next step.

### Split the Video

To split the file into parts, run:

```shell
split -b 1044000 --additional-suffix=.gif -d rick.mp4 part
```

Make sure you have 19 parts (the last part should be named part18.gif);
otherwise, the demo won't work without adjusting the `show.html` file.

Afterward, you can remove the original file as only parts will be used:

```shell
rm rick.mp4
```

### Upload Files

Put all the files from the repository and video parts in a directory on
your Cloudflare-proxied web server.

### Configure Cloudflare

- Enable Signed Exchanges in Cloudflare ([instructions](https://developers.cloudflare.com/speed/optimization/other/signed-exchanges/enable-signed-exchange/))
- Deploy [SXG Status](https://github.com/pepawel/sxg-status) worker at `/sxg/resolve-status.js`
  as explained in steps 1-3 in the [README](https://github.com/pepawel/sxg-status).
- Create a Cloudflare transform rule to strip the `Etag` header and add a proper `Vary` header:

```yaml
Rule name: Remove Etag and set Vary headers for SXG demo
If incoming requests match: Custom filter expression
Field: URL Path
Operator: wildcard
Value: /the-directory-you-placed-the-files/*
Then: Modify response header
Remove: etag
Set static: vary = Accept-Encoding
```

- Create a Cloudflare cache rule to make all the files cacheable:

```yaml
Rule name: Cache SXG demo
If incoming requests match: Custom filter expression
Field: URL Path
Operator: wildcard
Value: /the-directory-you-placed-the-files/*
Then: Eligible for cache
Edge TTL: Ignore cache-control header and use this TTL
Input time-to-live: 1 year
Browser TTL: Override origin and use this TTL
Input time-to-live: 1 year
```

### Populate SXG Cache

First, generate the SXG version of the demo page to fill the Cloudflare ASX cache. This is important; otherwise, Google will
need more time to put it into its cache.

```shell
curl -siH "Accept: application/signed-exchange;v=b3" https://www.yourdomain.com/the-directory-you-placed-the-files/show.html
```

Repeat until you see this line in the response:

```yaml
content-type: application/signed-exchange;v=b3
```

Now, let Google cache the page:

```shell
curl -siH "Accept: application/signed-exchange;v=b3" https://www-yourdomain-com.webpkgcache.com/doc/-/s/www.yourdomain.com/the-directory-you-placed-the-files/show.html
```

Again, repeat until you see the `signed-exchange` `content-type` in the response.

### Test the Demo

Go to the [prefetch page](https://signed-exchange-testing.dev/prefetch/), open the `Network` tab in
Chrome Dev Tools, and enter the URL of your `show.html` page:

```text
https://www.yourdomain.com/the-directory-you-placed-the-files/show.html
```

Hit `Submit` and observe the files being prefetched. In case of CORS errors, repeat clicking `Submit` until
all the errors are gone.

When all the video parts are prefetched, go offline and click the `link to target`. You should see the demo page with the video.

### Submit the Page to Google Index

For the demo to be complete, the `show.html` page has to be discoverable in Google as mentioned in the demo instructions.
Log in to Google Search Console and paste the full URL containing the `show.html` file into the input at the top of the page.
Then hit the `Request indexing` button.

It may take some time for Google to index the new page. In my experience, waiting 1 hour should be enough.

### Experience the Demo

Visit the instructions page:

```text
https://www.yourdomain.com/the-directory-you-placed-the-files/instructions.html
```

and follow the steps. Have fun!

## Author

My name is Paweł Pokrywka, and I'm the author of this demo.

If you want to contact me or get to know me better, check out
[my blog](https://www.pawelpokrywka.com).

## License

The software is available as open source under the terms of the
[MIT License](https://opensource.org/licenses/MIT).
