# aud_midi

[English](README.md) | Deutsch

MIDI für Dart- und Flutter-Apps auf iOS, macOS, Android, Windows, Linux und
im Web: USB, Bluetooth LE, virtuelles und Netzwerk-MIDI in einem
MIDI-Isolate. Es spricht Nachrichten für MIDI 1.0 und MIDI 2.0 (UMP). Alle
MIDI-Ein- und -Ausgaben laufen in einem eigenen Isolate, ein ausgelastetes
App-Isolate verzögert sie nie. Keine Platform Channels: Die Backends
sprechen über FFI, JNI und JS-Interop mit dem Betriebssystem.

## Motivation

Dart- und Flutter-Apps konnten die MIDI-Geräte ihrer Nutzer nicht auf
jeder Plattform über eine API erreichen: Jedes Betriebssystem hat seinen
eigenen MIDI-Dienst, und eine MIDI-Bibliothek, die im Isolate der App
läuft, verliert oder verzögert Noten, sobald die UI beschäftigt ist.
aud_midi bündelt die Backends der Familie hinter einem `Midi.open()` und
verlegt alle MIDI-Ein- und -Ausgaben in ein eigenes Isolate, damit eine
App ihren MIDI-Code einmal schreibt und Musiker die Noten hören, wenn sie
fällig sind.

- Eine API für sechs Plattformen; Ports sind die zentrale Einheit
- USB, Bluetooth LE, virtuelle Ports und Netzwerk-Sessions (AppleMIDI,
  Network MIDI 2.0)
- Nachrichten für MIDI 1.0 und MIDI 2.0 mit expliziter Übersetzung
- Genaues Timing: OS-Zeitstempel beim Empfang, OS-Scheduling beim Senden
- Ein MIDI-Isolate, das die Isolates der App nicht verzögern können
- Verluste werden als Diagnosen sichtbar, Streams liefern nie Fehler

## Installation

```bash
dart pub add aud_midi
```

Das Package hängt von allen Backends ab; `aud_midi_android` braucht das
Flutter-SDK (`jni`), daher das `dart` einer Flutter-Installation nutzen.

Einrichtung je Plattform:

- iOS: `NSBluetoothAlwaysUsageDescription`,
  `NSLocalNetworkUsageDescription`, `NSBonjourServices`
  (`_apple-midi._udp`), `UIBackgroundModes` `audio` für virtuelle Ports
  (und `bluetooth-central` für BLE im Hintergrund)
- macOS: Entitlements `com.apple.security.device.bluetooth`,
  `com.apple.security.network.client`,
  `com.apple.security.network.server`
- Android: die Berechtigungen und Services aus dem
  [README von aud_midi_android](https://github.com/audmidi/aud_midi_android)
- Windows: MSIX-Capabilities `bluetooth`, `internetClient`,
  `privateNetworkClientServer`
- Linux: `libasound2`, Zugriff auf `/dev/snd/seq`; BlueZ und Avahi über
  D-Bus für BLE und Netzwerk
- Web: https oder localhost, die MIDI-Berechtigung des Browsers

Die Beispiel-App in `example/` zeigt diese Einstellungen.

## Verwendung

`example/cli/bin/hello_midi.dart` öffnet MIDI, listet die Ports, erzeugt
eine virtuelle Quelle und spielt eine Note mit einem vom OS geplanten
Note Off:

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

Es gibt aus, auf einem Mac mit aktiviertem IAC-Treiber, je eine
`Input:`- und `Output:`-Zeile pro Port des Rechners:

```text
Backend: apple
Input:  IAC-Treiber Bus 1
Output: IAC-Treiber Bus 1
```

Weitere Beispiele: `example/cli` (`aud_midi_cli`, `midi_bench`,
`hello_midi`) und die Flutter-App in `example/`.

## Einschränkungen

- Windows und Linux: nur die Dart-Logik ist geprüft; die nativen Pfade
  wurden auf diesen Plattformen noch nicht gebaut oder ausgeführt (siehe
  [Status](#status))
- Im Web läuft die Engine im Hauptthread, wo Web MIDI zu Hause ist; ein
  MIDI-Isolate gibt es dort nicht
- `midi.bluetooth!.showSystemDevicePicker()` gibt es nur auf iOS;
  anderswo wirft es eine `MidiUnsupported`, dort stattdessen mit
  `midi.bluetooth!.scan()` suchen
- Native Taps (`input.attachNativeTap`) gibt es nur, wo
  `input.supportsNativeTap` gilt: CoreMIDI und `android.media.midi`

## Dokumentation

- [Architektur](https://github.com/audmidi/aud_midi_pm/blob/main/doc/2026-Q4/architecture/architecture.md)
- [Plan](https://github.com/audmidi/aud_midi_pm/blob/main/doc/2026-Q4/tickets/2026-10-06-aud_midi_01-initial-midi-implementation.md)
  und [Entscheidungen](https://github.com/audmidi/aud_midi_pm/blob/main/doc/2026-Q4/concepts/decisions/000-index.md)
- Packages: [aud_midi_standard](https://github.com/audmidi/aud_midi_standard)
  (Nachrichten, Codecs), [aud_midi_core](https://github.com/audmidi/aud_midi_core)
  (Engine), die Backends und [aud_midi_network](https://github.com/audmidi/aud_midi_network)
- [Guides](doc/guides/) und der [Blog](doc/de/blog/)

## Funktionsweise

`Midi.open()` startet ein MIDI-Isolate pro Prozess. Es betreibt eine
`MidiEngine` aus `aud_midi_core` mit dem Backend der Plattform, bei Bedarf
kombiniert mit der Netzwerk-Session und Bluetooth LE. Die Objekte `Midi`,
`MidiInput` und `MidiOutput` in der App sind Proxys, die über `SendPort`s
mit ihm sprechen; andere Isolates verbinden sich mit
`Midi.attach(midi.connector)`. Geplante Nachrichten erreichen das
MIDI-Isolate sofort und gehen 100 ms vor ihrer Zeit an das Betriebssystem,
so bleiben sie abbrechbar. Im Web läuft die Engine im Hauptthread, wo Web
MIDI zu Hause ist.

Ein nativer Verbraucher, z. B. eine Audio-Engine, nimmt die Rohdaten eines
Eingangs ohne Isolate dazwischen ab: Wo `input.supportsNativeTap` gilt
(CoreMIDI, `android.media.midi`), registriert `input.attachNativeTap(
MidiNativeTap(function: ..., user: ...))` eine C-Funktion, die das Backend
auf seinem Empfangs-Thread mit UMP-Wörtern oder MIDI-1.0-Bytes und einem
Host-Zeitstempel in Nanosekunden aufruft, und mit `gone`, wenn der Port
verschwindet; `detach()` des Handles ist erst nach dem letzten Aufruf
abgeschlossen. Die Taps eines Proxys gehen mit ihm.

Auf iOS zeigt `midi.bluetooth!.showSystemDevicePicker()` die
Bluetooth-MIDI-Geräteauswahl des Betriebssystems; die dort verbundenen
Geräte erscheinen als Ports. Anderswo wirft es eine `MidiUnsupported`;
dort sucht man stattdessen mit `midi.bluetooth!.scan()`.

## Status

Vor 1.0 — die API kann sich zwischen Minor-Versionen ändern.

| Plattform | Backend | Geprüft |
| --- | --- | --- |
| macOS | CoreMIDI (UMP), Bonjour, AppleMIDI in Dart | echter Loopback, Build der Beispiel-App |
| iOS | CoreMIDI (UMP), `MIDINetworkSession` | Loopback im Simulator, Build der Beispiel-App |
| Android | `android.media.midi` + AMidi | Loopback im Emulator, Build der Beispiel-APK |
| Windows | WinRT MIDI, optional Windows MIDI Services | nur Dart-Logik, nicht unter Windows gebaut |
| Linux | ALSA-Sequencer, BlueZ, Avahi | nur Dart-Logik, nicht unter Linux gelaufen |
| Web | Web MIDI | Chrome, Web-Build der Beispiel-App |

`midi_bench` auf einem Mac (Apple-Backend, zwei saubere Läufe mit je 500
Nachrichten):

| Prüfung | Median | p95 | p99 |
| --- | --- | --- | --- |
| Round Trip über virtuelle Endpunkte bis ins App-Isolate | 152–158 µs | 228–249 µs | 378–384 µs |
| Von CoreMIDI geplantes Senden bei ausgelastetem App-Isolate, Ankunft − Soll | 62–63 µs | 133–138 µs | 164–726 µs |
| In Software geplantes Senden über eine eigene virtuelle Quelle, Ankunft − Soll | 2,5–2,6 ms | 3,2 ms | 3,3 ms |

Fragmentiertes SysEx mit Clocks dazwischen wird korrekt zusammengesetzt,
Überläufe sind vollständig durch Diagnosen belegt, Hotplug wird nach etwa
290 ms gemeldet. Die Testabdeckung des Packages beträgt 100 %.

## Mitwirken

Siehe [doc/guides/develop-guide.md](doc/guides/develop-guide.md).

## Lizenz

MIT — siehe [LICENSE](LICENSE).
