# GROUND GLASS

Field Instrument FI-122. Version 0.3.0.

A single file console that puts a Canon EOS R6 on the bench: a monitor with real
scopes, and the camera's own knobs.

Open `groundglass-v0.3.0.html` by double clicking it. No server, no build step,
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

46 assertions, in the app, on demand. Exposure arithmetic against known answers,
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

If RELEASE completes but nothing appears, press **RELEASE PROBE**. It works
through each way of asking an EOS body for an exposure and reports which one
produced a file. Known causes it will help you separate:

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
- Video record start and stop, focus stacking, focus bracketing, touch to focus,
  and image download are not in this version.
- The scopes read the preview, not the sensor.
- WebUSB permission does not persist across sessions from `file://`, so the
  device picker appears on every connect.
- Canon property codes cover the common third stop scheme. Values outside the
  tables are logged rather than guessed at.

## Changelog

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
