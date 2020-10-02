[![](https://travis-ci.com/sudara/stitches.svg?branch=master)](https://travis-ci.com/sudara/stitches)

![stitches](logo.svg)

Stitching Together MP3 Playback in HTML5 Audio with ES6

## The goal

To distill and codify 13+ years experience of building dozens of web music players into a tight, usable library with 0 dependencies.

## Features

Stitches...

* Is written in ES6+
* Deals with "unlocking" audio elements from their auto-play restrictions to enable playlist playback
* Completely ignores the Web Audio API (which doesn't allow buffering, therefore useless for music playback)
* Only handles the MP3 format (pragmatically, the only format that matters)
* Lets you decide if you want to babel things or just include in a `<script type=module>`
* Is defensive, but doesn't test for browsers or feature detect
* Assumes you have a playlist
* Assumes you might want to preload one of the tracks
* Assumes you want to have continuous playback of a playlist (preloads next tracks)
* Assumes you care as much about mobile as you do desktop

## We worked hard so you don't have to

Unfortunately the state of HTML5 Audio support on browsers has not evolved much in the last decade, leaving the API incomplete and unreliable across platforms. Stitches has your back by:

* Unlocking multiple audio nodes on an interaction so you can play through multiple tracks

* Abstracts out and normalizes HTML5 audio events so that they actually work cross-browser (For example, [onended in iOS was broken for years](https://bugs.webkit.org/show_bug.cgi?id=173332)).

* Comes with defaults that enable gapless playback, with an ability to tune.

## Installation

`yarn install stitches`

## Usage

With a bundler like Webpack you can:

`
import Playlist from 'stitches'
`

With raw html, you can:

```
<script type='module'>
  import Playlist from './src/playlist.js'
  const playlist = Playlist.newFromSelector('a')
</script>
```

## How does it work?

We stitch together `<audio>` elements to provide a seamless (and hopefully excellent) playback experience.

Ideally browsers would be able to play a list of audio tracks. Instead we are stuck with creating individual `<audio>` elements, which in many browsers are each blocked from playing until after an user interaction event, such as a click.

From an experience point of view, we want to provide gapless playback of an album or playlist. We definitely don't want to force the user to keep interacting to get the next song. Ideally, they can open up new browser tabs and do their thing and an album will happily stream in the background.

To mitigate the fact that browsers sabotage this ability, we create a `NodePool` that holds `AudioNodes` (a wrapper around an individual `<audio>` element). On user interaction, we unlock (ie. call `.play()`) on each element in the `NodePool`, using a tiny silent mp3 (it's sketchy on some browsers to call `.play()` with an empty `src`)

For continuous/gapless playback we really only need two `<audio>` elements: One to handle a currently playing track and another to preload the next audio track in. When a file is done playing, the node is released back to the pool.

## Options

The following options can be passed to `new Playlist`

### `tracksSelector`

The only required option. No default, so you need to pass in some selector such as `.track`.

This selector determines which elements are considered to be tracks in this playlist.

The element type doesn't matter. What matters is other selectors such as `playButtonSelector` need to be children of this selector.

### `preloadIndex = -1`

Optional. Defaults to `-1` which doesn't load a track in the playlist.

If set to an integer, it'll preload that index, so `0` will preload the first track in the playlist.

### `playButtonSelector = "a"`

Optional. Defaults to the first `<a>` child of `tracksSelector`.

Which child element of `tracksSelector` should be considered the playButton?

### `progressSelector = "progress"`

Optional. Defaults to the first `progress` child of `tracksSelector` and fails silently if not present.

if the element is a progress element, this will update the attribute called `value` during the `whilePlaying` callback.

For other elements, it will set the element's `style.width` to be the appropriate percentage.

### `seekSelector = "progress"`

Optional. Defaults to the first `progress` child of `tracksSelector` and fails silently if not present.

This will register a click handler on the element so it can be used to seek the track.


### `timeSelector = "time"`

Optional. Defaults to the first `<time>` child of `tracksSelector` and fails silently if not present.

### `whilePlaying`

Optional. No default. Expects a function.

The provided function is called *repeatedly* during a track's playback.

### `onError`

Optional. No default. Expects a function.

The provided function is called if there's a problem with loading or playback.

Good for registering error notification such as Bugsnag, etc.

Returns an attribute named `error` which contains a `name` and a `message` as [described in the standard](https://dev.w3.org/html5/spec-author-view/spec.html#mediaerror).

### `logToConsole = false`

Optional. Defaults to false, keeping the console nice and clear.

This is also exposed a static setter `Log.logToConsole` in case there's a need for a lil runtime funtime.

## Events

This events API is based on what an app would ideally wan't from a player and not necessarily [the dumpster fire that is HTML5 audio events](https://developer.mozilla.org/en-US/docs/Web/Guide/Events/Media_events).

The following events fire:

### `track:preload`

This fires on *attempt* to preload a track in a playlist on page load, if and only if `preloadIndex` is set.

### `track:play`

This fires as soon as `play()` has been called on a track.

It does not mean the track is playing, only that play has been called.

### `track:loading`

An `AudioNode` was assigned for the track and it has been told to play the appropriate url.

### `track:notPlaying`

This will be fired if the attempt to grab an `AudioNode` fails.


### `track:playing`

This is fired once as soon as we know for sure the track is actually producing audio and happily playing.

### `track:whilePlaying`

This is *repeatedly* called, a few times a second, while a track is actively producing audio.

### `track:ended`

This is called when a track is finished. It does not rely on the somewhat sketchy nature of `<audio>` tag events fired from the browser, it will fire approximately 200ms near the end of the track.

## Why do we need this library?

10 years ago, I launched alonetone.com. Since then, I've written several wrappers around audio libraries such as SoundManager2 and Howler.js, both which are fantastic projects and enabled me to launch and maintain the site.

However, as time wore on, I found myself constantly having to keep up with the changes to audio behavior in browsers anyway, and more recently, have found the implementations lacking. In particular, other libraries tend to:

* Contain legacy/unrelated support of other methods of delivery like Flash and Web Audio
* Don't have first class support for sequential playback of a playlist
* Are written in Ye Olde JS™ vs. ES6/ES+
* Are full of browser/feature detection code
* Don't have cross browser tests

## Visualizing the object relationships

Each `Playlist` has `Tracks` that communicate to a `NodePool` containing `AudioNode`s.

```
                   ┌-----------┐
                   | Playlist  |
                   └-----------┘
                    /    |    \
                   /     |     \
          ┌-------┐  ┌-------┐  ┌-------┐
          | Track |  | Track |  | Track |
          └-------┘  └-------┘  └-------┘
                         ↓
          hi, can i have an unlocked node plz?
                         ↓
                   ┌----------┐
                   | NodePool |
                   └----------┘
                  /      |     \
                 /       |      \
          ┌------┐    ┌------┐   ┌------┐
         AudioNode   AudioNode  AudioNode
          └------┘    └------┘   └------┘
              |           |          |
           <audio>      <audio>   <audio>
```

These `AudioNode`s map 1-1 with HTML5 audio elements which are "unlocked" on any user interaction. The `NodePool` manages these unlocked `AudioNode`s, supplying them as needed.

Each `NodePool` (there's one per playlist) has exactly 3 `AudioNode`s, which allows for seamless preloading and playback with minimum amount of 🤹🏻‍♂️. For example they might all be in use in this situation:

1. The last track that just played
2. The current track that's playing now
3. The next track that's preloading

## Running tests

Given the somewhat sketchy state of audio playback in browsers (especially Safari), it's absolutely critical to run tests against as many current browsers as possible.

Selenium tests are currently written in [Nightwatch](http://nightwatchjs.org)

The full testing stack is by its nature VERY.... brittle.

There are lots of places things can go wrong: The tests themselves, the testing framework, the webdriver for each browser, selenium, all sorts of infrastructure related things on browserstack's end... the most difficult part of this project is definitely maintaining this testing harness.

### Locally Against Chrome

You can run tests locally with:

`yarn test`

This will run tests against headless chrome as configured in `nightwatch-local.conf.json`.

### Browserstack

![Sessions Overview - BrowserStack Automate 2020-02-27 00-57-41](https://user-images.githubusercontent.com/472/75399491-9ef3c480-58fc-11ea-802a-301012d23aff.jpg)

We are using Browserstack to run tests against multiple browsers. This is configured in `nightwatch-browserstack.conf.js` and uses a runner defined in `browserstack.runner.js`. Travis is setup to run Browserstack.

Test failures are manually reported via `afterEach` callbacks via Browserstack's API which manually send a request per test case to Browserstack, renaming the name of the test as well as reporting it as `passed` or `failed`.

### Locally against Browserstack

You can also run Browserstack locally (assuming you have the `BROWSERSTACK_USERNAME` and `BROWSERSTACK_ACCESS_KEY` credentials set as ENV variables)

`yarn browserstack`

This is useful for figuring out why something might be failing on one particular browser. For this use case, I recommend reducing the number of browsers that are being run in package.json, isolating the one browser that is causing problems.

PROTIP: You won't see any "live" output from nightwatch when more than one browser is being run, as in this case tests are being run in parallel and the log output wouldn't be very coherent. If you want to see nightwatch output as it happens, just use one browser.


## Releasing

Bump package number and publish:

```
npm version x.y.z
npm publish
```


## Acknowledgements

* Thanks to [@smoofles](https://twitter.com/smoofles) for the name and logo!
* Thanks to [@scottschiller](https://github.com/scottshiller) for creating [soundmanager](http://www.schillmania.com/projects/soundmanager2/), which had alonetone's back for many many years
* Thanks to [@blackslate](https://github.com/blackslate) for [an example of, and detailed instructions to render](https://gist.github.com/wittnl/8a1a0168b94f3b6abfaa) a small base64 encoded mp3.
* Thanks to [@scottanderson42](https://github.com/scottanderson42) for [the idea of creating a pool of unlocked nodes.](https://github.com/goldfire/howler.js/pull/1008)
