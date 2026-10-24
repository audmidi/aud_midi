# Changelog

## 0.1.4 - 2026-10-24

### Changed

- Make all repos run on Windows

## 0.1.3 - 2026-10-14

## 0.1.2 - 2026-10-13

### Added

- Add examples that resolve outside a full ticket
- The Flutter and the command line example override no siblings; the family comes from pub.dev

## 0.1.1 - 2026-10-12

### Added

- Add native taps to MidiInput

- MidiInput.supportsNativeTap and attachNativeTap through the MIDI isolate; MidiNativeTapHandle.detach

- The constraints on core, apple and android at ^0.1.1

- Add MidiBluetooth.showSystemDevicePicker

- The system device picker of the backend, through the MIDI isolate

- Add aud_midi_standard from pub.dev to the example

- The example resolves aud_midi_standard from pub.dev instead of a sibling checkout

- Add review fixes of ticket 26

- Add a goodbye that answers when a tap fails to detach

- Add independent constraints on the aud_midi packages

- Add review fixes of ticket 26

- A failed detach of a native tap other than MidiPortGone leaves the tap attached, so that detach can be called again

- A tap whose proxy left during the attach is detached at once

- aud_midi_core, aud_midi_android and aud_midi_apple ^0.2.0

- README: native taps in German, the Bluetooth system picker in both languages

## 0.1.0 - 2026-10-07

### Changed

- Turn aud_midi into the app-facing MIDI umbrella
- Reference the aud_midi packages on pub.dev and the audmidi organization
- Bound loopback arrival times by the block, not by 20 ms

## 0.0.1 - 2026-10-06

### Added

- Add dna_audanika

### Changed

- Replace DNA
- Prepare publishing

## 0.0.0 - 2026-10-06

- Initial Boilerplate
