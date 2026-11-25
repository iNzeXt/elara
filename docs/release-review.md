# Review before 1.0: what to fix beyond performance

Date: 2026-11-25. Code read: `main` at [`217bff8`][commit].

A wider review of Elara looked beyond speed: import and export, saving, audio
devices, the safety of the importers, the build and the packages. This file keeps
only what was checked in the code and matters most for 1.0. Performance is in
`docs/optimization-study.md`; the tasks are in `docs/roadmap.md`.

## 1. Before 1.0

### The app cannot export MusicXML or MIDI

`docs/SCOPE.md` puts "Import and export MusicXML and MIDI" in 1.0 ([scope]). The
exporters exist in `elara_io` and behind `elara-cli export`, but the app only saves
`.elara` files and exports PDF ([save dialog], [PDF export]). Someone who writes a
song in the app has no way to take it to another program.

The exporters already list what each format leaves out (let ring and the whammy bar
in MusicXML, techniques in MIDI). The app should tell the user.

**Done when:** the File menu has Export as MusicXML… and Export as MIDI… beside
Export as PDF…; a test exports from the app; an export that leaves something out
says what.

### Import warnings never reach the app

Every importer lists what it approximated or dropped in `ImportResult::warnings`
([result]), and `elara-cli import --json` prints them ([cli warnings]). In the app,
`openAndLayOut()` keeps the song and drops the list ([open]). Opening a Guitar Pro
file whose clef changes, volume changes or fifth voice are dropped says nothing.

**Done when:** a corpus file with a known loss shows a notice after it opens, with
the details on demand; a document opened after it does not show its warnings; a file
without warnings shows nothing.

### Playback may not follow a device that restarts at another sample rate

The player is made once, at the rate the device had then ([player made]). When the
device restarts, `audioDeviceAboutToStart` reads its new rate ([device start]), but
nothing tells the player, which goes on rendering at the old one. Headphones plugged
in, Bluetooth switching profiles or the system's format changed can all restart the
device at another rate.

Not reproduced yet. If it happens, the music plays at the wrong pitch and speed.

**Done when:** ruled out or reproduced by hand; if reproduced, a test with a device
that restarts at another rate shows the player rebuilt at that rate, in the same
place in the music.

### The packages are not built with the vcpkg that CI tests

`ci.yml` checks vcpkg out at the manifest's `builtin-baseline` ([ci vcpkg]);
`package.yml` clones vcpkg's latest commit ([package vcpkg]). The baseline keeps the
ports at the same versions, but vcpkg's own scripts and triplets can differ. So the
packages are not built exactly as CI tested them, and they miss their binary cache
whenever vcpkg moves on.

**Done when:** `package.yml` checks out the same baseline as `ci.yml`.

### The importers have never run under a sanitizer

Guitar Pro, MusicXML and MIDI files come from the internet, and the binary readers
parse untrusted bytes. The tests cover damaged containers, but no preset or CI job
builds with AddressSanitizer or UndefinedBehaviorSanitizer. Tests that pass do not
rule out memory errors; one sanitized run over the corpus is cheap and finds them.

**Done when:** a CI job builds the `elara_io` tests and `elara-cli` with ASan and
UBSan and imports every corpus file, the damaged ones included.

## 2. Worth deciding before 1.0

- **Crash recovery.** The app asks before closing with unsaved changes, but keeps
  no recovery copy: a crash loses everything since the last save. A recovery file
  written now and then would fix it. It is not in `docs/SCOPE.md`, so it is a
  product decision.
- **How the native format evolves.** The reader ignores keys it does not know and
  refuses a newer `formatVersion` ([native reader], [unknown keys test]); the
  writer writes only the keys it knows. Once 1.0 files are out, an older Elara can
  open a newer file of the same version and drop what it does not know when it
  saves. Write down which changes raise `formatVersion` before 1.0.
- **Saving.** `writeFileAtomically()` writes `<file>.saving` and renames it over
  the file ([atomic write]): a good base. It does not ask the system to write the
  data to disk before the rename, so after a power cut some file systems can keep
  the rename without the data. Saving off the interface thread (H06 in
  `docs/optimization-study.md`) should wait until the document knows which edit a
  save wrote, so that a later edit is not marked as saved.

## 3. Left out on purpose

- **A pool thread that fails to start aborts the app** ([pool]). True, but it can
  only happen at start-up, when the app could not run anyway.
- **No size limit before reading a whole file** ([read file]). The user picks the
  file; a huge one is slow or fails with an error, it does not corrupt anything.
  Each ZIP entry is already limited to 512 MiB ([zip limit]).
- **Release gates, templates and a register for each action.** Too much process
  for this project: one roadmap task, one pull request and one test are enough.

[commit]: https://github.com/ghost-in-the-groove/elara/commit/217bff80a44ab87d566ebede45db3d9d3d371e52
[scope]: https://github.com/ghost-in-the-groove/elara/blob/217bff80a44ab87d566ebede45db3d9d3d371e52/docs/SCOPE.md#L26
[save dialog]: https://github.com/ghost-in-the-groove/elara/blob/217bff80a44ab87d566ebede45db3d9d3d371e52/src/app/qml/Main.qml#L238-L246
[PDF export]: https://github.com/ghost-in-the-groove/elara/blob/217bff80a44ab87d566ebede45db3d9d3d371e52/src/app/qml/Main.qml#L507-L509
[result]: https://github.com/ghost-in-the-groove/elara/blob/217bff80a44ab87d566ebede45db3d9d3d371e52/src/io/include/elara/io/result.hpp#L22
[cli warnings]: https://github.com/ghost-in-the-groove/elara/blob/217bff80a44ab87d566ebede45db3d9d3d371e52/src/cli/src/file_commands.cpp#L197
[open]: https://github.com/ghost-in-the-groove/elara/blob/217bff80a44ab87d566ebede45db3d9d3d371e52/src/app/src/score_document.cpp#L215-L240
[player made]: https://github.com/ghost-in-the-groove/elara/blob/217bff80a44ab87d566ebede45db3d9d3d371e52/src/app/src/audio_controller.cpp#L168
[device start]: https://github.com/ghost-in-the-groove/elara/blob/217bff80a44ab87d566ebede45db3d9d3d371e52/src/audio/backend_juce/src/system_audio.cpp#L88-L90
[ci vcpkg]: https://github.com/ghost-in-the-groove/elara/blob/217bff80a44ab87d566ebede45db3d9d3d371e52/.github/workflows/ci.yml#L68-L79
[package vcpkg]: https://github.com/ghost-in-the-groove/elara/blob/217bff80a44ab87d566ebede45db3d9d3d371e52/.github/workflows/package.yml#L72-L78
[native reader]: https://github.com/ghost-in-the-groove/elara/blob/217bff80a44ab87d566ebede45db3d9d3d371e52/src/io/src/native_format.cpp#L527-L532
[unknown keys test]: https://github.com/ghost-in-the-groove/elara/blob/217bff80a44ab87d566ebede45db3d9d3d371e52/tests/io/test_native_format.cpp#L336
[atomic write]: https://github.com/ghost-in-the-groove/elara/blob/217bff80a44ab87d566ebede45db3d9d3d371e52/src/io/src/archive.cpp#L92-L116
[pool]: https://github.com/ghost-in-the-groove/elara/blob/217bff80a44ab87d566ebede45db3d9d3d371e52/src/core/src/parallel.cpp#L8-L13
[read file]: https://github.com/ghost-in-the-groove/elara/blob/217bff80a44ab87d566ebede45db3d9d3d371e52/src/io/src/archive.cpp#L118-L134
[zip limit]: https://github.com/ghost-in-the-groove/elara/blob/217bff80a44ab87d566ebede45db3d9d3d371e52/src/io/src/archive.hpp#L22
