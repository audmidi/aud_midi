# aud_midi

English | [Deutsch](README.de.md)

MIDI for Dart and Flutter apps on iOS, macOS, Android, Windows, Linux and
the web: USB, Bluetooth LE, virtual and network MIDI in a MIDI isolate.
It speaks MIDI 1.0 and MIDI 2.0 (UMP) messages. All MIDI I/O runs in its
own isolate, so a busy app isolate never delays it. No platform channels:
the backends talk to the operating system through FFI, JNI and JS interop.

## Motivation

Dart and Flutter apps had no way to reach the MIDI devices of their users
on every platform through one API: each operating system has its own MIDI
service, and a MIDI library that runs on the app's isolate drops or delays
notes as soon as the UI is busy. aud_midi bundles the backends of the
family behind one `Midi.open()` and moves all MIDI I/O into its own
isolate, so an app writes its MIDI code once and musicians hear notes
when they are due.

- One API for six platforms; ports are the primary entity
- USB, Bluetooth LE, virtual ports and network sessions (AppleMIDI,
  Network MIDI 2.0)
- MIDI 1.0 and MIDI 2.0 messages with explicit translation
- Exact timing: OS timestamps on input, OS scheduling on output
- A MIDI isolate that the app's isolates cannot delay
- Losses are visible as diagnostics, streams never error

## Installation

```bash
dart pub add aud_midi
```

The package depends on all backends; `aud_midi_android` needs the Flutter
SDK (`jni`), so use the `dart` of a Flutter installation.

Platform setup:

- iOS: `NSBluetoothAlwaysUsageDescription`,
  `NSLocalNetworkUsageDescription`, `NSBonjourServices`
  (`_apple-midi._udp`), `UIBackgroundModes` `audio` for virtual ports
  (and `bluetooth-central` for BLE in the background)
- macOS: entitlements `com.apple.security.device.bluetooth`,
  `com.apple.security.network.client`,
  `com.apple.security.network.server`
- Android: the permissions and services from the
  [aud_midi_android README](https://github.com/audmidi/aud_midi_android)
- Windows: MSIX capabilities `bluetooth`, `internetClient`,
  `privateNetworkClientServer`
- Linux: `libasound2`, access to `/dev/snd/seq`; BlueZ and Avahi over
  D-Bus for BLE and network
- Web: https or localhost, the browser's MIDI permission

The example app in `example/` shows these settings.

## Usage

`example/cli/bin/hello_midi.dart` opens MIDI, lists the ports, creates a
virtual source and plays one note with an OS-scheduled Note Off:

```dart
import 'package:aud_midi/aud_midi.dart';

Future<void> main() async {
  // Start the MIDI isolate with the backend of this platform
  final midi = await Midi.open();
  print('Backend: ${midi.backendName}');
  for (final input in midi.inputs) {
    print('Input:  ${input.info.name}');
  }
  for (final output in midi.outputs) {
    print('Output: ${output.info.name}');
  }

  // Create a virtual source that other apps can receive from
  final source = await midi.virtualPorts!.createSource('aud_midi hello');

  // Play C4 now and release it 500 ms later; the MIDI isolate schedules
  // the Note Off, so a busy app isolate cannot delay it
  await source.send(const MidiNoteOn(channel: 0, note: 60, velocity: 100));
  await source.send(
    const MidiNoteOff(channel: 0, note: 60),
    at: midi.now() + const Duration(milliseconds: 500),
  );
  await Future<void>.delayed(const Duration(seconds: 1));

  await midi.close();
}
```

It prints, on a Mac with the IAC driver enabled, one `Input:` and one
`Output:` line per port of the machine:

```text
Backend: apple
Input:  IAC-Treiber Bus 1
Output: IAC-Treiber Bus 1
```

More examples: `example/cli` (`aud_midi_cli`, `midi_bench`, `hello_midi`)
and the Flutter app in `example/`.

## Limitations

- Windows and Linux: only the Dart logic is verified; the native paths
  were not built or run on those platforms yet (see
  [Status](#status))
- On the web the engine runs on the main thread, where Web MIDI lives;
  there is no MIDI isolate there
- `midi.bluetooth!.showSystemDevicePicker()` exists on iOS only;
  elsewhere it throws a `MidiUnsupported`, scan with
  `midi.bluetooth!.scan()` instead
- Native taps (`input.attachNativeTap`) exist only where
  `input.supportsNativeTap` holds: CoreMIDI and `android.media.midi`

## Documentation

- [Architecture](https://github.com/audmidi/aud_midi_pm/blob/main/doc/2026-Q4/architecture/architecture.md)
- [Plan](https://github.com/audmidi/aud_midi_pm/blob/main/doc/2026-Q4/tickets/2026-10-06-aud_midi_01-initial-midi-implementation.md)
  and [decisions](https://github.com/audmidi/aud_midi_pm/blob/main/doc/2026-Q4/concepts/decisions/000-index.md)
- Packages: [aud_midi_standard](https://github.com/audmidi/aud_midi_standard)
  (messages, codecs), [aud_midi_core](https://github.com/audmidi/aud_midi_core)
  (engine), the backends and [aud_midi_network](https://github.com/audmidi/aud_midi_network)
- [Guides](doc/guides/) and the [blog](doc/blog/)

## How it works

`Midi.open()` spawns one MIDI isolate per process. It hosts a
`MidiEngine` from `aud_midi_core` with the backend of the platform,
composed with the network session and Bluetooth LE where needed. The
`Midi`, `MidiInput` and `MidiOutput` objects in the app are proxies that
talk to it through `SendPort`s; other isolates attach with
`Midi.attach(midi.connector)`. Scheduled sends reach the MIDI isolate at
once and are handed to the operating system 100 ms before they are due,
so they stay cancellable. On the web the engine runs on the main thread,
where Web MIDI lives.

A native consumer, e.g. an audio engine, takes the raw data of an input
without any isolate in between: where `input.supportsNativeTap` holds
(CoreMIDI, `android.media.midi`), `input.attachNativeTap(MidiNativeTap(
function: ..., user: ...))` registers a C function that the backend calls
on its receive thread with UMP words or MIDI 1.0 bytes and a host
timestamp in nanoseconds, and with `gone` when the port disappears; the
handle's `detach()` completes after the last call. The taps of a proxy
leave with it.

On iOS, `midi.bluetooth!.showSystemDevicePicker()` shows the Bluetooth
MIDI device picker of the operating system; the devices connected there
appear as ports. Elsewhere it throws a `MidiUnsupported`; scan with
`midi.bluetooth!.scan()` there instead.

## Status

Before 1.0 — the API may change between minor versions.

| Platform | Backend | Verified |
| --- | --- | --- |
| macOS | CoreMIDI (UMP), Bonjour, pure-Dart AppleMIDI | real loopback, example app build |
| iOS | CoreMIDI (UMP), `MIDINetworkSession` | simulator loopback, example app build |
| Android | `android.media.midi` + AMidi | emulator loopback, example APK build |
| Windows | WinRT MIDI, optional Windows MIDI Services | Dart logic only, not built on Windows |
| Linux | ALSA sequencer, BlueZ, Avahi | Dart logic only, not run on Linux |
| Web | Web MIDI | Chrome, example web build |

`midi_bench` on a Mac (Apple backend, two clean runs, 500 messages
each):

| Check | Median | p95 | p99 |
| --- | --- | --- | --- |
| Round trip through virtual endpoints to the app isolate | 152–158 µs | 228–249 µs | 378–384 µs |
| CoreMIDI-scheduled send with a busy app isolate, arrival − due | 62–63 µs | 133–138 µs | 164–726 µs |
| Software-scheduled send from an own virtual source, arrival − due | 2.5–2.6 ms | 3.2 ms | 3.3 ms |

Fragmented SysEx with clocks in between is reassembled, overflows are
fully accounted for by diagnostics, hotplug is reported after about
290 ms. Coverage of the package is 100 %.

## Contributing

See [doc/guides/develop-guide.md](doc/guides/develop-guide.md).

## License

MIT — see [LICENSE](LICENSE).
