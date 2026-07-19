# GROUND GLASS

Field Instrument FI-122. Version 0.20.5.

A single file console that puts a Canon EOS R6 on the bench: a monitor with real
scopes, and the camera's own knobs.

Open `groundglass-v0.20.5.html` by double clicking it. No server, no build step,
no dependencies, no network access. It works from `file://` and it works with
the network cable pulled.

## The four links

The instrument does not assume one way in. Four links share one camera model,
and the console only ever reads that model, so the whole UI behaves the same
whichever one is carrying.

| Link | Transport | Picture | Control |
|------|-----------|---------|---------|
| BENCH | none, synthetic | yes | yes, simulated |
| GLASS | HDMI capture through getUserMedia | yes, best quality | no |
| WIRE | Canon PTP over WebUSB | yes, polled and slow | yes, full |
| AETHER | Canon CCAPI over the network | yes | depends, see below |

The intended working configuration is GLASS and WIRE together: HDMI for the
picture, USB for the knobs. The R6 keeps HDMI alive while it is being controlled
over USB.

### BENCH

A synthetic scene and a simulated body. Four scenes, including a colour bar and
resolution chart. Everything downstream of the frame, the scopes, the overlays,
the exposure rings, the self test, runs with nothing plugged in. Start here.

### GLASS

Any USB HDMI capture device. Set the R6 to clean HDMI output. Note that the
camera reduces its HDMI resolution while it is recording internally.

This link carries picture only. There is no control path through a capture
device, and the instrument says so rather than pretending.

### WIRE

Canon PTP over WebUSB. Opens a session, puts the body in remote mode, subscribes
to the event stream, sets live view routing, and polls the viewfinder.

The host operating system claims the camera before the browser can:

- **Windows.** The MTP class driver takes the body. Replace the driver on that
  interface with WinUSB using Zadig, then reconnect.
- **macOS.** The system PTPCamera agent takes it. Quit that agent, then reconnect.
- **Linux.** The gvfs gphoto2 volume monitor takes it. Stop that service, or add
  a udev rule granting your user access to the device, then reconnect.

WebUSB needs Chrome or Edge. Firefox and Safari do not implement it.

#### Bench testing WIRE

The link no longer fails as one lump. The handshake runs as six named steps and
the log names the step that stopped and what the camera answered, by name rather
than by hex.

1. Free the camera from the host driver, then connect it by USB and turn it on.
2. Open the LINK panel, tick **Hex trace** (this switches the debug log on too).
3. Press **CONNECT** and pick the Canon body in the browser's device picker.
4. Watch the six handshake lines. If one fails, stop there and save the log.
5. Press **DEVICE INFO**. This is the body telling you, in its own words, which
   operations and properties it honours. If `0x9128` is absent from the
   operations list, the release path falls back to `0x910f` automatically.
6. Press **BENCH TEST**. Eleven steps, each with a timing, ending in a decoded
   frame. It deliberately does not fire the shutter.
7. Fire the shutter by hand with RELEASE once the rows above are green.
8. Save the log and send it along with which step went red.

What changed in the transport since the first cut, and why it matters at the
bench:

- **Leftover bytes are kept.** A single bulk transfer often carries both the
  data phase and the response phase. Discarding the remainder and waiting for
  another transfer is the classic Canon hang. There is now one receive buffer
  and a pure container splitter feeding off it.
- **Zero length packets are skipped** rather than treated as a short read.
- **Device busy is retried**, three times, with a widening gap. Canon answers
  `0x2019` routinely and it is not an error.
- **Unsolicited event containers are stepped over** when they arrive in the
  middle of a transaction.
- **The still image class interface is preferred** over the first bulk pair that
  happens to appear.
- **Session already open is tolerated** on connect, which is what you get after
  a reload without a clean release.
- **Writes are refused** for any property the body did not advertise, rather
  than sent and silently ignored.

### AETHER

Canon CCAPI. Activate CCAPI on the body once with Canon's activation tool, join
the camera to your network, then read the address and port off the camera screen
and enter them.

The camera answers HTTP but sends no cross origin headers, which splits this
link cleanly in two. An `<img>` tag is not subject to that rule, so live view
always comes through once the camera is reachable. Replies to `fetch` cannot be
read unless the camera allows this origin. PROBE settles which half you have.

Blind control is the fallback. It sends requests the browser is allowed to make
without asking permission first, which the camera acts on but does not report
back. The shutter works blind. Settings do not, because a setting change is not
a request the browser will send that way. The instrument refuses rather than
lying about a value it cannot read.

## The monitor

Grid, safe areas, focus peaking, zebras, false colour, and flip. Below the
picture: an RGB histogram, a luma waveform, and a clipping pair.

The scopes read the preview, not the sensor. They tell you about the picture you
are being shown, which is the honest thing they can tell you.

## The exposure rings

Three concentric engraved arcs, one per leg of the triangle, each with a brass
needle. The ring nearest the end of its own travel is the one holding the
exposure back, and it is the only one that lights. EV at ISO 100 sits in the
middle.

## Self test

162 assertions, in the app, on demand. Exposure arithmetic against known answers,
the Canon value tables, the PTP container codec, live view chunk extraction,
event stream parsing against malformed input, the container splitter against
split and malformed transfers, the DeviceInfo parser, the response code table,
the session schema against hostile input, and the binding leg logic. All of it runs with no camera attached.

## Session files

Export and import carry settings only: theme, link, overlays, thresholds,
camera address, scene, sound and debug state. Imported files are validated field
by field against a schema, every value is coerced rather than trusted, and
nothing is ever inserted as markup.

## When the shutter will not fire

A PTP transaction that answers `0x2001` means the request was accepted. It does
not mean a frame exists. Those are different claims and this instrument no
longer conflates them: the sound and the "exposure made" line both wait for a
new object to show up in the event stream.

The status strip names the sequence in use, and says `(assumed)` until a probe
has confirmed it against this body.

If RELEASE completes but nothing appears, press **RELEASE PROBE**. It works
through each way of asking an EOS body for an exposure and reports which one
produced a file. Known causes it will help you separate:

- The camera **blacks out and produces no file**. That is a half press the host
  never released, holding the body. Cleared automatically now, before and after
  every attempt.
- The body wants a **two stage press**, half then full, rather than a single
  press. This is the usual answer on R series bodies and is now tried first.
- The host has not **declared capacity**, so the camera refuses to shoot and
  refuses quietly. Pressed automatically before capture, and SHOOT TO CARD
  sidesteps it entirely.
- A **self timer** drive mode delays the file past the window the probe watches.
- Autofocus was requested and **could not confirm focus**, so the body never
  reached the exposure. Uncheck Half press AF.
- No card, a movie mode dial position, a menu open on the body, or a lens
  switched to manual with autofocus requested.

## When a setting will not change

Same rule as the shutter: an acknowledged write is not an applied write. The
console now waits for the body to report the new value back and refuses to
display anything it has not confirmed.

If a change does not take:

- **Check the mode shown in the status strip.** In Auto, scene, or movie
  positions the camera holds its own exposure settings and will accept a write
  from the host without acting on it. P, Av, Tv, M and Fv are the modes where
  writes take.
- Exposure compensation does nothing in M with ISO fixed, on the camera or from
  here.
- **PROPERTY PROBE** tries three write forms and keeps the one your body echoes.
- **EVENT TAP** answers the other direction. Press it, turn a dial on the
  camera, and watch the log. Records appearing means the body reports and the
  fault is in the decoding. Nothing appearing means the body is not reporting
  and no amount of parsing will help.

## Settings the mode is holding

A camera in aperture priority chooses the shutter speed. Writing a shutter speed
from here is not refused so much as ignored, because there is nothing for it to
act on, and the same is true of aperture in shutter priority and of both in
program.

Controls the current mode is holding are dimmed and carry a note saying which
mode is holding them. Changing one explains what the camera is doing rather than
reporting a failed write. Modes the console does not recognise, including custom
positions, are left alone rather than assumed to hold anything.

## When a setting will not land

If a change reports device busy, the camera is streaming pictures and answering
the console at the same time. Writes now pause the viewfinder and event polls
and wait for the body to come free, which handles it in most cases.

When it does not, tick **Pause live view** in the monitor panel. The console
stops asking for pictures, the camera has the cable to itself, and settings land
first time. The frame on screen is labelled as not live while this is on.

## Increments, and why a write can look like it failed

Exposure compensation and exposure settings come in third stop and half stop
schemes, and a body holds one or the other. The published tables here carry
both, because the console cannot know which a given body is set to until it
tries. Ask for a half stop on a body set to thirds and it will land on the
nearest third.

That is the write working. The console reports what the camera settled on and
keeps it, rather than insisting on an exact match and rolling back a change that
actually happened.

## Settings the camera never describes

The console learns the allowed values for a setting from description records on
the event stream. When those never arrive for a given setting, aperture being
the common case with some lenses, the control used to grey out.

It no longer does. The published Canon values are offered instead, the control
carries a note saying the list is unconfirmed, and the exposure ring for that
leg stays hollow so you can still see at a glance that the body is not
describing it. If the camera declines a value, it simply will not stick, which
the console reports rather than papering over.

## A lens the body cannot see

An adapted or manual lens with no electronic contacts reports no aperture. The
camera has none to give, so the exposure triangle loses a leg and EV cannot be
computed.

When that happens a **Lens is set to** row appears under the aperture control.
Whatever you pick there feeds the dial and the EV reading, and is marked with an
asterisk and a dashed needle wherever it shows, because it is your reading of
the lens rather than the camera's. If the body later reports a real aperture,
that always wins.

## The half press arms the shutter

A full press with no half press before it is not a faster way to take a
photograph. On an EOS body the half press is what arms the shutter, and a bare
full press answers device busy. If you see that sequence succeed, look at what
ran before it: something else half pressed and left the body armed.

So every sequence here presses half, then full, then lets go of both in reverse.
Whether the camera tries to focus during the half press is a setting on the
body, not a stage to leave out, which is what the Half press AF checkbox now
controls.

## Why the same sequence can work and then not

An EOS body streaming live view is doing something. It answers device busy, and
whether a release turns into an exposure depends on where in that stream the
request happens to land. Run a probe three times and three different rows will
pass, each one a little later in the list than the last, which looks like
sequences behaving inconsistently and is really a body that has finally come
free.

So the viewfinder is stopped before a release, the body is given a moment, and
the console waits until it stops answering busy before asking for the frame.
The stream comes back afterwards on every path, including failures.

## One press, at most one frame

An EOS body will make an exposure and still answer `0x2019 device busy`. The
acknowledgement and the frame are independent, which means a release can never
be safely retried on the strength of its return code: a retry is another
exposure, not another attempt at the same one.

So the rule here is that a sequence is fired once per press, and afterwards the
console looks for a file regardless of what came back. If a sequence produces
nothing, the console says so and stops rather than working down the list, since
working down the list means firing every one of them.

RELEASE PROBE is the exception, and it says so: it fires each sequence in turn
on purpose, and it will leave several frames on the card.

## If the camera shows Err 70

Switch the camera off, take the battery out for about thirty seconds, reseat the
lens, and switch it back on. That normally clears it. If it returns when this
instrument is not attached, the camera needs service rather than software.

Version 0.18 of this instrument could cause it. It wrote a focus point using a
guessed payload layout, and a body handed a payload it cannot parse can fault.
That feature is gone, and this build refuses to write any property whose layout
it has not confirmed.

## The capture panel

**Intervalometer.** Interval, frame count, start and stop. It goes through the
same release path as the button, so it inherits one press one frame, and it
stops itself the moment a frame fails to appear rather than hammering a camera
that is not cooperating. This is the one feature here that is entirely on this
side of the cable, which is why it is the one I can promise behaves.

**Image download.** Set **Images go to** to include this computer first. A
camera writing only to its own card has nothing queued for the host to collect,
and every transfer route will decline, correctly.

The handle of each new file arrives on the event stream.
DOWNLOAD LAST fetches it, and "Download each frame" does it automatically after
every release. Bytes that come back starting with a JPEG signature get a .jpg
name and are reported as a JPEG. An empty response is reported as a failure, not
as a file.

**Video.** START RECORDING writes the record property and waits for the body to
report the new state. If the camera confirms, it says so. If it does not, it
says that too, rather than showing a recording indicator on the strength of an
acknowledgement.

**Focus.** FOCUS NOW runs the camera's autofocus at whatever point the body is
already using. Choosing the point from here needs a payload layout this
instrument has not confirmed, and guessing at one caused a camera fault, so it
has been removed rather than left in behind a warning. Set the point on the
camera.

## Known limitations

- WIRE is written against the published Canon EOS PTP extensions and has not yet
  been proven against an R6. Run BENCH TEST and treat every write as unverified
  until those rows are green.
- The bench test writes ISO back to the value it already holds. That proves the
  write path without changing anything on the body, but it does not prove that a
  write to a different value is accepted in every camera mode.
- PTP live view is a polled JPEG, so it is a slideshow next to the GLASS link.
  This is a property of the protocol, not of the implementation.
- AETHER control depends on whether the camera tolerates a simple request. If it
  does not, live view still works and the knobs do not.
- Focus stacking and focus bracketing are not in this version.
- The focus point cannot be set from here. Only properties with a confirmed
  payload layout are written, and the focus point is not one of them.
- Video recording is started and stopped by property write. Bodies that require
  the mode dial to be on the movie position will decline it.
- The scopes read the preview, not the sensor.
- WebUSB permission does not persist across sessions from `file://`, so the
  device picker appears on every connect.
- Canon property codes cover the common third stop scheme. Values outside the
  tables are logged rather than guessed at.

## Changelog

- **0.20.5** Settings writes were competing with the viewfinder poll and the
  event poll for one bulk pipe, and a body with its hands full answers device
  busy. Releases already got the pipe to themselves and there was no reason
  writes should not. A write now pauses both polls, waits for the body to be
  free, and retries generously, which is safe for a write in a way it is never
  safe for a release. There is also a Pause live view control for working the
  knobs with the cable to yourself. 162 assertions.

- **0.20.4** The battery reading was a Canon level code with a percent sign
  appended, so a level of 3 displayed as 3 per cent. Those are now separate
  things. The standard PTP battery level property, which is defined as a
  percentage, is asked for every twenty seconds and shown as a percentage when
  the body answers. When it does not, the coarse level is shown as a level and
  labelled as one, because an invented percentage is worse than an honest
  absence of one. 158 assertions.

- **0.20.3** The capacity declaration tolerated a refusal and then reported
  success, so RELEASE PROBE has been showing a green row for something the
  camera declined. That matters most with images going to the computer, where
  Canon will not shoot without an accepted capacity, so a false green there is
  expensive. Capacity and the destination write both report what the body
  actually answered now, and a release that produces nothing says where images
  are going and whether capacity was accepted, rather than only that nothing
  arrived. 153 assertions.

- **0.20.2** Three different transfer opcodes all answering the same general
  error is not three faults, it is one handle the body will not part with. The
  raw object added record is now kept rather than parsed and discarded, and
  OBJECT PROBE prints it as hex next to the fields read out of it, asks the body
  whether it recognises the handle at all, and lists storage. If the body will
  not describe a handle it will certainly not send it, and that is a shorter
  path to the answer than trying a fourth opcode. 149 assertions.

- **0.20.1** Download was using the wrong opcodes. Canon numbers its own object
  transfer separately from the standard set, and the acknowledgement after a
  transfer is 0x9117; the build was using 0x9019, which belongs to another
  vendor entirely. The Canon whole object fetch, 0x9104, was missing. All three
  routes are now tried in the order a Canon body is likely to answer, with the
  standard fetch last because these bodies routinely decline it, and the log
  says which one worked. A failure now names every route it tried and points at
  the capture destination, which is the usual reason nothing is available: a
  camera writing only to its card has nothing queued for the computer to
  collect. That destination is now a control. 145 assertions.

- **0.20.0** Touch to focus is removed. It wrote a focus point using a payload
  layout that had been inferred rather than confirmed, and tried four candidate
  layouts in turn to find one the body liked. On an R6 that produced Err 70, a
  hardware fault needing the battery pulled. Guessing at a payload is acceptable
  against a parser and is not acceptable against a camera, because a camera can
  be harmed. What remains is FOCUS NOW, which drives the camera's autofocus at
  its existing point and invents no bytes. Every property write now passes
  through one builder that refuses any property whose layout this instrument has
  not confirmed, so a speculative payload cannot be sent from anywhere in the
  file. 142 assertions.

- **0.19.1** A capture stops the viewfinder stream, and the write that restarts
  it is often declined while the camera is still writing the file. The console
  trusted that write, so it sat holding the last frame and presenting it as
  live. Restoring the viewfinder now waits for frames to actually arrive rather
  than for the write to be acknowledged, and retries. A watchdog notices when
  the picture has not changed for two and a half seconds, says so, and asks for
  the stream back on its own. While the picture is stale it is labelled LAST
  FRAME, NOT LIVE across the top, because a still presented as live is the same
  kind of untruth as a shutter sound with no exposure behind it. 141 assertions.

- **0.19.0** Downloads ask the way Canon wants to be asked. The standard whole
  object fetch answers a general error on these bodies, so the EOS partial read
  is tried first, a megabyte at a time until a short chunk ends the file, and it
  no longer needs the body to have declared a size. The console also knows which
  settings a mode holds: in aperture priority the camera chooses the shutter, so
  a shutter write has nothing to act on. Those controls are dimmed with an
  explanation, and a write to one says what the mode is doing rather than
  reporting a failure. Modes it does not recognise are treated as unknown, which
  is not the same as refused. 138 assertions.

- **0.18.1** Two honesty bugs and a real one. A tolerated response was being
  reported as a passing row: agreeing not to throw on an answer is not the same
  as the camera doing the thing, and the autofocus row said the body accepted a
  request when it may have answered busy. Rows now report the actual response
  code. The record property write did the same and now says what it answered.
  The real bug: the record and focus point property codes were used but never
  defined, so those writes were addressed to property zero. A new assertion
  checks every entry in the property table is a number inside the Canon range,
  which is what caught it. Focus point layout is now probed across four
  candidates and the accepted one is kept, and a busy answer to a property write
  is retried, which is safe in a way that retrying a release is not.
  130 assertions.

- **0.18.0** Four capabilities, each labelled by how well grounded it is.
  Intervalometer: entirely on this side of the cable, reuses the release path,
  stops itself if a frame does not appear. Image download: standard PTP, the
  handle comes off the event stream, and bytes arriving with a JPEG signature is
  the proof. Video: two property writes confirmed by the body reporting the new
  state, and it says so plainly when it does not. Touch to focus: the least
  grounded, so it reports what each call answered rather than claiming a result.
  125 assertions.

- **0.17.0** The bare full press answered device busy on every run today without
  exception, and the runs where it appeared to work were the ones where an
  earlier sequence had already half pressed. The half press arms the shutter;
  skipping it is not a shorter press, it is an incomplete one. Every lead
  sequence now presses half then full, and the bare press is demoted to a last
  resort. The autofocus checkbox no longer decides which stages are sent: it
  puts the body in manual focus for the duration of the release and puts the
  previous focus mode back afterwards, on every exit path. 117 assertions.

- **0.16.0** Three probe runs in a row each passed a different sequence, and in
  each one the sequence that passed was a later row than the time before. That
  is not a sequence being right or wrong, it is a body that is free by the time
  the fourth attempt arrives. A camera streaming live view has its hands full:
  it answers device busy and whether an exposure happens depends on where in the
  stream the request lands. The viewfinder is now stopped before any release,
  the body is given time to settle, and the console waits until it stops
  answering busy before asking. The stream is restored on every exit path. A
  known sequence that fails twice stops being treated as known. 114 assertions.

- **0.15.1** The preferred sequence follows the autofocus checkbox again. Both
  candidates are half then full presses; the difference is whether the half
  press asks for focus, and a body that cannot focus never reaches the exposure.
  This coupling was removed in 0.14.0 because the no autofocus sequence kept
  answering device busy, which was the wrong reason: the body makes the exposure
  while answering busy. Asking for autofocus on a lens the body cannot drive now
  says so in the log. 110 assertions.

- **0.15.0** A write is confirmed by the value changing, not by it changing to
  exactly what was asked for. A camera set to third stop increments cannot hold
  a half stop, so it lands on the nearest value it can, which is the write
  working. That was being read as a failure and rolled back. The published
  exposure compensation table carries both increment schemes, so roughly half
  the values offered could never be held by any given body. When a write really
  does not take, the log now says what the body reported instead of only that
  nothing arrived. PROPERTY PROBE can aim at any setting rather than only ISO.
  108 assertions.

- **0.14.1** The bench test failed a body that had nothing to say. By the time
  that step runs, the connect drain and the event pump have already taken
  everything the camera had reported, so counting records measured timing rather
  than health. The step now writes ISO back to the value it already holds and
  listens for the echo, which tests the stream instead of the silence.
  104 assertions.

- **0.14.0** Reverses 0.13.0, which was wrong. An EOS body makes the exposure
  even when it answers device busy, so retrying a busy sequence does not get
  another chance at one frame, it puts another frame on the card. That produced
  long delays and frames nobody asked for. A release is now fired at most once
  per sequence per press, and after any answer, including an error, the console
  looks for a file. Evidence decides, never the return code. A known sequence
  that produces nothing no longer sweeps the remaining sequences, because
  sweeping means firing all of them. The preferred sequence is the half then
  full press and no longer changes with the autofocus checkbox, which decides
  what happens inside the sequence rather than which one to use. 103 assertions.

- **0.13.0** Device busy was being read as a verdict on the sequence rather than
  a statement about the moment. A sequence that answered 0x2019 was discarded and
  the sweep moved on to a worse one, which is why the first press failed and the
  second worked with the body warmed up. Transient answers are now retried with
  the same sequence, up to four attempts with a widening gap, before the sweep
  gives up on it. The event pump also stands down for the whole release, since
  the confirmation loop is already polling and two pollers on one pipe is how a
  body ends up answering busy in the first place. Connect drains until the body
  has nothing left to say instead of draining a fixed twice. 99 assertions.

- **0.12.1** Half press autofocus is off by default. With it off the preferred
  release sequence is the plain two stage press, which is also what a manual or
  adapted lens needs. 93 assertions.

- **0.12.0** The button and the probe now run the same function. They had two
  implementations of the same job, and the button's copy ordered the sequences
  with a comparator that returned -1 for one element and 0 for every other pair.
  That is not a consistent ordering, so the engine was free to return the list
  in any order, which is why the probe worked reliably and the button did not.
  Ordering is now a plain filter and concatenate, with tests that it keeps every
  sequence, puts the preferred one first, and leaves the rest alone. RELEASE
  PROBE is this function with reporting on, CAPTURE is the same function with
  reporting off, and a sequence that stops working falls back to the full sweep.
  91 assertions.

- **0.11.0** The root cause behind every "works on the second press" symptom.
  A PTP reply carries the transaction id of the request it belongs to, and the
  transaction loop never checked it: whatever container arrived next was taken
  as the answer. One stale container therefore put every later transaction one
  reply behind, which looks exactly like an instrument that needs two or three
  tries and behaves perfectly in between. Replies are now matched to their
  request, stale ones are dropped and logged, event containers are stepped over
  however many arrive, and the buffer is drained before every transaction and
  after any that fails. Property descriptions moved off the connect path to a
  READ LISTS button, since each extra transaction there was another chance to
  fall out of step. Exposure compensation in manual mode now says what is
  actually happening. 85 assertions.

- **0.10.0** Release is the only timing sensitive thing this instrument does,
  and it was sharing a thread with a compositor that reads back a full frame
  every animation tick. Both render loops now stand down for the duration of a
  release and resume in a finally, so a throw cannot leave the console frozen.
  Live view stands down for the whole release rather than only for writes.
  Confirmation waits six seconds and polls twice as often. COPY REPORT puts the
  version, body, firmware, release sequence, write form, shooting mode, reported
  scales, operations list and full log on the clipboard in one press.
  79 assertions.

- **0.9.0** The dial was still reading the raw option list while the controls
  had moved to the fallback, so a setting with a value but no reported scale got
  a ring with no ticks and no needle. Both now read one source. Long scales are
  thinned rather than drawn into a smear, unconfirmed scales are dimmed, and the
  ring labels fan around the arc instead of stacking along one radius. A lens
  with no electronic contacts reports no aperture, so the aperture can now be
  declared by hand: it drives the EV reading, it is marked with an asterisk and
  a dashed needle everywhere it appears, and a reported aperture always wins.
  74 assertions.

- **0.8.0** Connect now brings the body to a known state: press cleared, UI lock
  reset, capacity declared, event stream drained twice. That work used to happen
  as a side effect of the first press failing, which is why the first press was
  wasted and the second one worked. A setting the body never described is no
  longer disabled: the published values are offered, marked unconfirmed, and the
  write path decides, since a value the camera will not take simply never echoes
  back. Missing lists are also requested directly through the standard property
  description call. 68 assertions.

- **0.7.0** A blackout on the camera with no file is a half press that was never
  let go. Every release now clears the press state first and again afterwards,
  so a stale press cannot block the next one. Capture stops the viewfinder
  stream on EOS bodies, which is the blackout you see, so live view is asked for
  again after each attempt. Confirmation waits four seconds instead of two and
  also accepts the frame counter falling, which on some bodies lands first. The
  half press gets 700 ms to focus and meter, and a long settle variant is tried
  after it. 63 assertions.

- **0.6.0** Removes the defect that was blocking settings outright. Writes were
  gated on the standard PTP device property list, and Canon does not enumerate
  its EOS extension properties there: an R6 lists five entries and writes ISO
  regardless. Every write to a property outside that list was refused before it
  was sent. The gate is gone and a test reads the function back to make sure it
  cannot return. RELEASE now works through the sequences on the first press
  rather than trying one and reporting failure, so a cold body takes a picture
  without a probe first. A sequence that stops working is forgotten rather than
  reused. 59 assertions.

- **0.5.0** Property writes are confirmed rather than assumed. A setting change
  waits for the body to report the new value back, and if it does not, the
  console restores the old value instead of displaying one the camera never
  accepted. Shooting mode is read and shown, because an EOS body in an automatic
  mode accepts a write and ignores it. PROPERTY PROBE tries three write forms and
  keeps whichever the body echoes. EVENT TAP prints every record the body sends
  for ten seconds, which settles whether the camera is reporting at all. Live
  view stands down while a write is being confirmed, since both share one pipe.
  57 assertions.

- **0.4.0** Fixes a real defect: the two stage press was tried first by the
  probe but the plain RELEASE button still selected the old single press by the
  autofocus checkbox, so the probe worked and the button did not. There is now
  one named default, used by both, and it is covered by a test. A release that
  finds nothing probes once and finishes the job rather than handing back a
  suggestion. A discovered sequence survives a reconnect and rides along in the
  session file. The sequence in use is shown in the status strip, assumed or
  confirmed, so it is never a hidden mode. 51 assertions.

- **0.3.0** Release rebuilt around confirmation. Seven strategies, tried in
  order, each verified by watching the event stream for a new file rather than
  by trusting a success code. The two stage press, half then full, is tried
  first, which is what R series bodies want. RELEASE PROBE reports which one
  this body honours. The shutter sound now fires on evidence, not on
  acceptance. Capacity is declared to the body before capture. Dial no longer
  stacks its ring labels, draws a ring hollow when the body reports nothing,
  and names the silent leg. Unmapped property values are logged without needing
  debug on. 46 assertions.

- **0.2.0** WIRE transport hardened for the bench: receive buffering, zero length
  packets, busy retry, event interleaving, class 6 interface preference, session
  reuse. Staged handshake with named failure points. DeviceInfo parsing, which
  selects the release opcode from what the body advertises. Response codes by
  name. Hex trace. BENCH TEST and DEVICE INFO in the LINK panel. 42 assertions.
- **0.1.0** First cut. Four links, monitor and scopes, exposure rings, session
  files, 33 assertions.

## Roadmap to v1.0

1. Prove WIRE against the R6 and fix what the bench finds.
2. Touch to focus and focus step, both of which PTP exposes.
3. Video record start and stop.
4. Intervalometer and bracketing, driven off the same camera model.
5. Image download to disk after each release.
6. PWA layer, additive only, never load bearing.

## License

GPL-3.0
