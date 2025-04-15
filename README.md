# Stretching Google Prefetching demo

This is a demonstration of how large the prefetched data can be when using Signed Exchanges (SXG).
You can experience the [live demo](https://www.planujemywesele.pl/sxg-tests/offline-abuse),
watch a [video recording](https://www.pawelpokrywka.com/p/stretching-google-prefetching)
or deploy the demo yourself to your website (described later).

## How does it work

The `instructions.html` file is for instructing the user to find the `show.html` page in Google
and then distracting him/her long enough for the data to be prefetched.

The `show.html` contains the actual demo. It containg tags for prefetching the video on Google search results using SXG.
To understand SXG, check out my [deep-dive](https://www.pawelpokrywka.com/p/how-i-took-lcp-down-under-350ms).

The HTML is composed of three pages, only one being visible at a time.
Right after the user visits the page, the embedded script decides which page to display based on user's online status.

Online status is being tested by using browser API and by trying to perform network requests, because in some cases
browser API is unreliable (such as when using a firewall or disconnecting WAN cable from the router).

### The online page

This page is displayed when the user didn't follow instructions and visited the page while being online.
It's also for Googlebot to index it, which allows the user to find the page in Google Search.

You may notice the page contains *slop* about being offline. It was required, because otherwise the page won't have
enough content and Google treats it as so-called soft-404 and rejects indexing it.

### The offline page

If the page was prefetched, user went offline and visited it, the offline page is being presented.
It's a copy of default Chrome offline page to give an impression the page didn't load.

But it did and the script tests if the SXG subresources were correctly prefetched. In case user didn't wait enough,
some of the subresources may be missing and the demo will fail. In this case, the script stops and the user is left
with the offline page.

If the subresources were correctly loaded, the script displays a speech-bubble near the dinosaur with a link.
The speech-bubble is blinking, as many people didn't see it otherwise.

### The video page

This page appears when the link is clicked. The script starts to concatenate prefetched video parts. It takes a lot of RAM
and may fail if the user's device is low on memory. The resulting video data is converted to blob,
placed as an `src` attribute of a video element, and the video is started.

The reason for the dinousaur page is to force the user to click. Otherwise the browser would mute the video taking the
fun out of the demo.

## FAQ

### Where does the SXG prefetching happens?

Look for `<link rel="preload">` tags in the `show.html`. They are enclosed in `<template data-sxg-only>`
so that the prefetching happens only for SXG visit. That's because they are not used otherwise,
so prefetching them doesn't make sense.

To learn more about prefetching subresource
read [my blog post](https://www.pawelpokrywka.com/p/subresources-prefetching-with-signed-exchanges).

### Why the video is split into parts?

[Google won't cache SXG files larger than 1044 KB](https://www.pawelpokrywka.com/p/other-errors-with-signed-exchanges),
therefore the video has to be splitted into parts.

### Why .gif extension is used for parts?

Google won't cache SXG files with unpopular extensions.

### How does the page know if the subresources were prefetched?

It uses [SXG Status](https://github.com/pepawel/sxg-status), my another project implementing the technique I discovered,
for that.

### Can I use another video or use something entirely different?

You can use any file that's no larger than about 19 MB. Potential ideas:

- a very large picture,
- compressed archive, uncompressed later by the browser and containing entire web application,
- executable for the user to download.

## Installation & usage

To deploy the demo to your server follow steps below.

### Download a video

You'll need 19 MB video. The demo won't work for smaller files without adjustments and larger files will most
likely exceed the SXG limit.

To download the clip used in the original demo
install [yt-dlp](https://github.com/yt-dlp/yt-dlp) and execute in the shell:

```shell
yt-dlp -f 609+251 --merge-output-format mp4 -o rick.mp4 'https://www.youtube.com/watch?v=dQw4w9WgXcQ'
```

In case the command failed or the resulting `rick.mp4` is of a different size than required (~19 MB),
you will need to check available formats by executing:

```shell
yt-dlp -F 'https://www.youtube.com/watch?v=dQw4w9WgXcQ'
```

Find video and audio formats that together have around 19 MB and pass their ids
to the `-f` option instead of `609+251`. For example if the video is 42 and audio is 23 execute:

```shell
yt-dlp -f 42+23 --merge-output-format mp4 -o rick.mp4 '<https://www.youtube.com/watch?v=dQw4w9WgXcQ>'
```

If the video size is about 19 MB, you can proceed to the next step.

### Split the video

To split the file into parts run:

```shell
split -b 1044000 --additional-suffix=.gif -d rick.mp4 part
```

Make sure you have 19 parts (the last past should be named part18.gif),
otherwise the demo won't work without adjusting `show.html` file.

Afterwards you can remove the original file as only parts will be used:

```shell
rm rick.mp4
```

### Upload files

Put all the files from the repository and video parts in a directory on
your Cloudflare-proxied web server.

### Configure Cloudflare

- Enable Signed Exchanges in Cloudflare ([instructions](https://developers.cloudflare.com/speed/optimization/other/signed-exchanges/enable-signed-exchange/))
- Deploy [SXG Status](https://github.com/pepawel/sxg-status) worker at `/sxg/resolve-status.js`
  as explained in steps 1-3 in the [README](https://github.com/pepawel/sxg-status).
- Create Cloudflare transform rule to strip `Etag` header and add proper `Vary` header:

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

- Create Cloudflare cache rule to make all the files cachable:

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

### Populate SXG cache

First generate SXG version of the demo page to fill Cloudflare ASX cache. It's important, otherwise Google will
need more time to put it into its cache.

```shell
curl -siH "Accept: application/signed-exchange;v=b3" https://www.yourdomain.com/the-directory-you-placed-the-files/show.html
```

Repeat it until you see this line in the response:

```yaml
content-type: application/signed-exchange;v=b3
```

Now, let Google cache the page:

```shell
curl -siH "Accept: application/signed-exchange;v=b3" https://www-yourdomain-com.webpkgcache.com/doc/-/s/www.yourdomain.com/the-directory-you-placed-the-files/show.html
```

Again, repeat it until you see `signed-exchange` `content-type` the response.

### Test the demo

Go to [prefetch page](https://signed-exchange-testing.dev/prefetch/), open the `Network` tab in
Chrome Dev Tools and fill the input on the page URL of `show.html` on your website:

```text
https://www.yourdomain.com/the-directory-you-placed-the-files/show.html
```

Hit `Submit` and observe the files being prefetched. In case of CORS errors, repeat clicking `Submit` until
all the errors are gone.

When all the video parts are prefetched go offline and click `link to target`. You should see the demo page with video.

### Submit the page to Google index

For the demo to be complete, the `show.html` page has to be discoverable in Google as told in demo instructions.
Log in to Google Search Console and paste the full URL containing the `show.html` file to the input at the top of the page.
Then hit the `Request indexing` button.

It may take some time for Google to index the new page. In my experience waiting 1 hour should be enough.

### Experience the demo

Visit instructions page:

```text
https://www.yourdomain.com/the-directory-you-placed-the-files/instructions.html
```

and follow the steps. Have fun!

## Author

My name is Paweł Pokrywka and I'm the author of this demo.

If you want to contact me or get to know me better, check out
[my blog](https://www.pawelpokrywka.com).

## License

The software is available as open source under the terms of the
[MIT License](https://opensource.org/licenses/MIT).
