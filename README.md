# vtt-adjust

[![NPM](https://nodei.co/npm/vtt-adjust.png)](https://nodei.co/npm/vtt-adjust/)

Utility for moving and, if necessary, stretching a .vtt (WebVTT) file.  Useful for correcting desync issues after a transcode or cropping.

## Example

During a format conversion, 3 seconds of blank frames were trimmed from the classic motion picture "Revenge of the Ghost Dinosaur".  This resulted in all the captions now showing 3 seconds too early:

![Illustration of the Problem](img/the-problem.png?raw=true "Illustration of the Problem")

We can make `vtt-adjust` correct all the caption cues by telling it the correct position of *one* cue:

```javascript
// Log and move all cues forward by 3 seconds.

var vttAdjust = require('vtt-adjust');

var adjuster = vttAdjust([
  'WEBVTT',
  '',
  '00:14:10.000 --> 00:14:12.000',
  'Sally, the Masked Ranger... was me all along.',
  '',
  '00:14:13.000 --> 00:14:15.000',
  'Oh... Harry! I had no idea...',
  '',
  '00:14:16.000 --> 00:14:18.000',
  'Rarrrr!',
  'OMG! Run',
  ''
  '00:14:19.000 --> 00:14:21.000',
  'Aaaaahhh, my arm...'
].join('\n'));

console.log(adjuster.cues.map(JSON.stringify).join('\n'));
/* Output: -------------------
{"id":0,"start":850000,"text":"Sally, the Masked Ranger... was me all along."}
{"id":1,"start":853000,"text":"Oh... Harry! I had no idea..."}
{"id":2,"start":856000,"text":"Rarrrr!\nOMG! Run!"}
{"id":3,"start":859000,"text":"Aaaaahhh, my arm..."}
                            */

// Fix all cues by using the first cue as a reference:
// It should start at 14 minutes, 13 seconds (=853000ms).
adjuster.move(adjuster.cues[0], 853000);

console.log(adjuster.toString());
/* Output: -------------------
WEBVTT

00:14:13.000 --> 00:14:15.000
Sally... the Masked Ranger was me all along.

00:14:16.000 --> 00:14:18.000
Oh... Harry! I had no idea...

00:14:19.000 --> 00:14:21.000
Rarrrr!
OMG! Run!

00:14:22.000 --> 00:14:24.000
Aaaaahhh, my arm...
```

<a name="download" />
## Download

For [Node.js](http://nodejs.org/), use [npm](http://npmjs.org/):

    npm install vtt-adjust

<a name="browser" />
## In the Browser
Download and include as a `<script>`.  The module will be available as
the global object `window.vttAdjust`, similarly to if you'd written
`var vttAdjust = require('vtt-adjust');` in Node.js.

__Development:__ [vtt-adjust.js](https://github.com/deestan/vtt-adjust/raw/master/build/vtt-adjust.js) - 7Kb Uncompressed

__Production:__ [vtt-adjust.min.js](https://github.com/deestan/vtt-adjust/raw/master/build/vtt-adjust.min.js) - 4Kb Minified

__Example__

```html
<script src="vtt-adjust.js"></script>

<textarea style="width: 300px; height: 500px" id="vtt">WEBVTT

00:00:01.000 --> 00:00:02.000
Herp derp?

00:00:20.000 --> 00:00:21.300
Niort!</textarea>

<br/><button onclick="moveClicked();void(0);">Move cues 1 second forward</button>

<script>
function moveClicked() {
  var vttEl = document.getElementById('vtt');
  var adjuster = vttAdjust(vttEl.value);
  var referenceCue = adjuster.cues[0];
  adjuster.move(referenceCue, referenceCue.start + 1000);
  var output = adjuster.toString();
  vttEl.value = output;
}
</script>
```

## Documentation

### Constructor

[vttAdjust](#vttAdjust)

<a name="vttAdjust" />
### vttAdjust (vtt)

Returns an adjuster object, with the following properties:

* [cues](#cues)
* [move](#move)
* [moveAndScale](#moveAndScale)
* [toString](#toString)

__Arguments__

* vtt - `string` containing the entire source .vtt file.

__Example__
```javascript
var vttAdjust = require('vtt-adjust');
var vtt = require('fs').readFileSync('captions.vtt').toString();
var adjuster = vttAdjust(vtt);
```

<a name="cues" />
### adjuster.cues

Property containing an array of all cues found in .vtt file.  Array elements are of the form `{ id: <opaque identifier>, start: <start time in milliseconds>, text: <text found in cue> }`.

The `id`s of these cues are used as references when calling the `move` and `moveAndScale` functions.

__Example__

```javascript
console.log(adjuster.cues);
//> [ { id: 0, start: 10000, text: 'bim\nbum' },
//>   { id: 1, start: 20000, text: 'bam\nbom' },
//>   { id: 2, start: 30000, text: 'weh' } ]
```

<a name="move" />
### move(refCueId, newStart)

Move the reference cue to the new starting position, and all other cues by a correspinding amount.

![Illustration of move()](img/move.png?raw=true "Illustration of move()")

Mathematically, each cue's position is mapped by a function `f(t) = t + c`,
where `c` is the difference between the reference cue's new and
original positions.

__Arguments__

* refCueId - the `id` value from one element in `adjuster.cues`.
* newStart - `integer`, the reference cue's desired start time in milliseconds.

__Example__

```javascript
// The first cue should begin at 31 seconds.
var cue = adjuster.cues[0];
adjuster.move(cue.id, 31000);
```

<a name="moveAndScale" />
### moveAndScale(refCueId1, newStart1, refCueId2, newStart2)

Move and scale all cues equally, such that refCue1 and refCue2 end up in their new positions.

![Illustration of moveAndScale()](img/moveAndScale.png?raw=true "Illustration of moveAndScale()")

If the video file has a subtly changed framerate, the cues will drift slowly out of sync during the video, which is not a problem that can be corrected with just moving all cues by a fixed amount.  For this problem, you can stretch or shrink the timeline to fit by giving two reference cues.  The further apart in the video the reference cues are, the more precise the result will be.

Mathematically, each cue's position is mapped by a function `f(t) = t * a + c`
where `c` and `a` are calculated such that the two reference cues end up in the
desired positions.

Both start and end positions are affected individually by the scaling, so if the cues end up closer together, they are also shorter.

__Arguments__

* refCueId1 - the `id` value from one element in `adjuster.cues`.
* newStart1 - `integer`, the first reference cue's desired start time in milliseconds.
* refCueId2 - the `id` value from one element in `adjuster.cues`.  Must not be the same as refCueId1.
* newStart2 - `integer`, the second reference cue's desired start time in milliseconds.

__Example__

```javascript
// The first cue should begin at 4 seconds, and the 42nd and
// last cue should begin at 14 minutes 15 seconds.
var cue1 = adjuster.cues[0];
var cueZ = adjuster.cues[42];
adjuster.move(cue1, 4 * 1000, cueZ, 14 * 60000 + 15 * 1000);
```

<a name="toString" />
### toString()

Return a string representation of the (possibly) adjusted .vtt file.

__Example__

```javascript
var vtt = adjuster.toString();
require('fs').writeFileSync("adjustedCaptions.vtt", vtt);
```

## Notes

In the event that a `move()` or `moveAndScale()` calls throw an error,
the adjuster object will remain in the state it was before the call.
