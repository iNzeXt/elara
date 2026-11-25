# Roadmap

Milestones from `docs/SCOPE.md`. Each task should fit in one working session and
one pull request, with a test that proves it. Tick the box in the same PR.

## Milestone 0 - Foundation

- [x] Repository layout, contributor guide, ADRs, glossary
- [x] CMake presets, vcpkg manifest, pinned JUCE, Qt 6.12 install script
- [x] Task runner `scripts/elara.py` (doctor, check, build, test, format, guard, cli, app)
- [x] Architecture guard and clang-format check
- [x] `elara-cli`: `version`, `render-tone`, `devices`, `tone`
- [x] Qt Quick window with a test-tone button through JUCE; `--smoke-test`
- [x] CI workflow for Windows, macOS and Linux
- [x] First green build on Windows (MSVC 19.44, Qt 6.12.1, JUCE 9.0.6)
- [x] First green CI run on GitHub (Windows, macOS, Linux)

## Milestone 1 - Musical model (`elara_core`)

Done when a song can be built, edited, undone and saved/loaded from the CLI.

- [x] Rational durations (`Fraction`) and time positions; tuplet arithmetic
- [x] Pitch and spelling (step, alter, octave), MIDI number conversion
- [x] Tuning and fret/string <-> pitch mapping, capo
- [x] Song > Track > Staff > Measure > Voice > Beat > Note hierarchy
- [x] Time and key signatures, tempo map
  - [x] A tempo change from any measure, with the CLI `tempo-from` step and the
        app's measure popup (ADR-0042)
  - [x] Gradual tempo changes, accelerando and ritardando (ADR-0082)
    - [x] In the model and playback, read from Guitar Pro 6-8, kept by the
          native format
    - [x] Drawn after the tempo mark
    - [x] Typed in the CLI and the measure popup
    - [x] Written in MusicXML (words and dashes, both ways) and MIDI (steps)
- [x] Signs and jumps (segno, coda, D.C., D.S., Fine, To Coda) in the play order,
      `.elara` and the CLI (ADR-0049)
  - [x] Read from Guitar Pro 5 and GPIF, read and written in MusicXML
  - [x] Drawn in the score
  - [x] Set from the measure popup in the app
- [x] Sections of a song (intro, verse, chorus) named where they start
  - [x] In the model, `.elara` and the CLI `section` step; read from Guitar
        Pro 3-8, read and written in MusicXML and MIDI (ADR-0062)
  - [x] Drawn in the score, boxed above the first staff (ADR-0063)
  - [x] Named from the measure popup in the app
- [x] Note articulations and effects (bend curve, slide, hammer/pull, palm mute,
      dead note, harmonics, vibrato, ties) as data
- [x] Measure validation (voice fills its measure), rests auto-fill
- [x] Command pattern with undo/redo
- [x] Native format v1 (zip container + JSON), round-trip tests
- [x] `elara-cli new`, `elara-cli dump --json` (model snapshot used by later tests)

## Milestone 2 - Importers (`elara_io`)

- [x] Self-made test corpus with `SOURCES.md`
- [x] Guitar Pro 7-8 (`.gp`) import with `elara-cli import`
- [x] Guitar Pro 6 (`.gpx`) import
  - [x] Arpeggios of Guitar Pro 6-8 as rolled chords: a strum spread from the
        lowest note upwards, from the highest downwards (ADR-0025, ADR-0076)
- [x] Guitar Pro 3/4/5 (binary) import
  - [x] Keyboards reaching into both clefs on a grand staff, split at middle C
        (ADR-0075)
- [x] MusicXML import (partwise, `.mxl`) with `elara-cli import`
- [x] MusicXML export (text and `.mxl`) with `elara-cli export`, tested by round trip
- [x] MIDI import/export with `elara-cli import`/`export`, tested by round trip
  - [x] Played velocities as a dynamic per measure, accents and ghost notes (ADR-0045)
- [x] `elara-cli import <file> --json` snapshot tests per corpus file
- [ ] Export as MusicXML… and Export as MIDI… in the app, beside Export as PDF…,
      telling what the format leaves out (`docs/release-review.md`)
- [ ] Import warnings in the app: what opening a file approximated or dropped,
      shown with that document (`docs/release-review.md`)

## Milestone 3 - Notation engine (`elara_notation`)  -> usable as a viewer

- [x] SMuFL metadata loading (Bravura), glyph metrics
- [x] Horizontal spacing, beaming, stems, accidentals for one staff
- [x] Tablature staff layout with effects notation
- [x] Systems and pages: line breaking, justification, multi-track scores
  - [x] Grand staves (a piano's treble and bass staves): a brace and barlines
        through both (ADR-0073)
  - [x] Rolled chords of keyboards and other instruments without frets: an
        arpeggio line left of the chord (ADR-0076)
- [x] Repeat barlines, repeat counts and alternate endings (ADR-0016)
- [x] Tuplet numbers, with a bracket when no beam shows the group (ADR-0040)
- [x] Rhythm under tablature shown without standard notation (ADR-0041)
- [x] Tempo marks at the start and at every tempo change (ADR-0042)
- [x] Dynamics marks under the staff where the loudness changes (ADR-0044)
- [x] Hairpins (crescendo, diminuendo; ADR-0078)
  - [x] In the model, read from Guitar Pro 6-8 and kept by the native format
  - [x] Read and written in MusicXML
  - [x] Drawn under the staff
  - [x] Played as a velocity ramp
  - [x] Typed at the caret
- [x] Whammy bar curves and depths, and tapping, slap and pop, above the staff
      (ADR-0061)
- [x] Grace notes (ADR-0079)
  - [x] In the model, read from Guitar Pro 3-8 and kept by the native format
  - [x] Read and written in MusicXML
  - [x] Drawn as small notes and fret numbers
  - [x] Played by borrowing time
  - [x] Typed at the caret
- [x] Chord symbols: names and diagrams above the staff (ADR-0080)
  - [x] In the model, read from Guitar Pro 3-8 and kept by the native format
  - [x] Read and written in MusicXML
  - [x] Drawn above the staff, with diagrams on fretted staves
  - [x] Typed at the caret
  - [x] Transposed with the notes
- [x] Incremental relayout (only the systems an edit touches)
  - [x] Pages an edit leaves as they were keep their textures instead of being
        painted again
  - [x] Only the measures and systems an edit changed are measured and laid out
        again; the rest is reused (ADR-0052)
- [x] `elara-cli render --svg` snapshot tests: every corpus file on pages
- [x] PDF export: `elara-cli render --pdf` and Export as PDF… in the app, the
      glyphs drawn from the music font's outlines (ADR-0074)
- [x] Score view in the app: pages painted on worker threads and shown as scene
      graph textures, open-file dialog and drag and drop, track and view selection,
      smooth zoom (ADR-0015)
  - [x] Continuous view: the music as one scroll, cut into strips of whole
        systems between them (ADR-0057)
  - [x] Recent scores on the welcome pane and in an Open recent menu, kept for
        the next start (ADR-0071)
- [x] Beat text: words above the beat, "let it ring", "ad lib." (ADR-0081)
  - [x] In the model, read from Guitar Pro 3-8, MusicXML both ways and kept by
        the native format
  - [x] Drawn above the staff
  - [x] Typed at the caret
- [x] Lyrics: a syllable per verse under the beats that sing it (ADR-0084)
  - [x] In the model, read from Guitar Pro 4-8 and kept by the native format;
        verses typed as text placed on the beats with notes
  - [x] Read and written in MusicXML
  - [x] Drawn under the staff
  - [x] Typed at the caret
- [x] Fermatas: short, normal and long holds on notes and rests (ADR-0085)
  - [x] In the model, held in the timeline, playback and MIDI export
  - [x] Read and written in MusicXML and kept by the native format
  - [x] Drawn above the staff, under it for a lower voice
  - [x] Set at the caret
- [x] Slurs: beats slurred to the next, drawn over the notes they join (ADR-0086)
- [x] Clef changes: a measure changes its staff's clef, drawn smaller before the barline (ADR-0087)
- [x] Tenuto and portato: held notes, drawn by the notehead, beyond the staccato dot (ADR-0088)
  - [x] In the model and kept by the native format
  - [x] Read and written in MusicXML
  - [x] Drawn on standard notation, clear of the notes, across lines
  - [x] Set at the caret
- [x] Octave lines: 8va, 8vb, 15ma and 15mb over the beats whose notes they write nearer the staff (ADR-0089)
  - [x] In the model and kept by the native format
  - [x] Read and written in MusicXML, read from Guitar Pro 6-8
  - [x] Drawn on standard notation, over barlines, cut and repeated at system breaks
  - [x] Set at the caret
- [x] Trills: notes alternated with the next step of the key, or any note up to an octave away, at their speed (ADR-0090)
  - [x] In the model, played and kept by the native format
  - [x] Read and written in MusicXML, read from Guitar Pro 3-8
  - [x] Drawn on standard notation with a wavy line, and in a lane over tablature
  - [x] Set at the caret
- [x] Mordents and turns: quick notes on the steps of the key at the start of a note (ADR-0091)
  - [x] In the model, played and kept by the native format
  - [x] Read and written in MusicXML, read from Guitar Pro 6-8
  - [x] Drawn on standard notation and in a lane over tablature
  - [x] Set at the caret

## Milestone 4 - Playback  -> usable as a player

- [x] Sequencer: model -> timed events, tempo map, repeats (ADR-0017)
- [x] FluidSynth integration, MuseScore General SF3
  - [x] SoundFont playback through FluidSynth: sound banks in the renderer and the
        player, `elara-cli soundfont`, `render-audio`/`play --soundfont` (ADR-0043)
  - [x] A SoundFont chosen in the app (any SF2 or SF3, such as MuseScore
        General), kept for the next start, `--soundfont FILE`
  - [x] MuseScore General in the packages, played until another sound is
        chosen, `--included-soundfont` (ADR-0069)
- [x] SFZ instruments and a guitar SFZ library
  - [x] SFZ instruments through Elara's own sampler, each note bent on its own:
        `elara-cli sfz`, `render-audio`/`play --sfz` (ADR-0066)
  - [x] SFZ instruments for the guitars and the basses: chosen in the app and
        kept for the next start, `--guitar-sfz`/`--bass-sfz`, and
        `--sfz guitars=FILE`/`basses=FILE` in `elara-cli`
  - [x] FLAC samples, read by the sampler like WAV ones (ADR-0067)
  - [x] Takes picked at random for each note (`lorand`/`hirand`), as many
        free guitar libraries do
  - [x] Guitar techniques mapped to a library's keyswitches and controllers:
        palm mutes, dead notes and harmonics, found by name (ADR-0068)
  - [x] A library's legato samples for hammer-ons and pull-offs: a keyswitch
        named for legato or `trigger=legato` regions (ADR-0070)
  - [x] A clean electric guitar SFZ (FreePats, CC0) in the packages: "Use the
        included guitar" and `--included-guitar-sfz` (ADR-0072)
- [x] Articulation rendering (bends, slides, legato, palm mute, harmonics, strums, tremolo picking; ADR-0022, ADR-0023, ADR-0025)
- [x] Mixer (volume, pan, mute, solo): playback (ADR-0020) and the app, in the song panel (ADR-0039)
- [x] `elara-cli render-audio` offline render + event-log tests, with a built-in synthesizer (ADR-0018)
- [x] Real-time playback through the built-in synthesizer, `elara-cli play` (ADR-0019)
- [x] Transport, playback cursor and click-to-seek in the app, `elara-cli render --cursor-at`
  and `--time-at`
- [ ] Playback follows the output device when it restarts at another sample rate:
      reproduce first (`docs/release-review.md`)

## Milestone 5 - Editor

- [x] Selection model and keyboard-driven entry (fret numbers, durations)
  - [x] Drum entry: lanes for the kit pieces, hits toggled at the caret, in the
        CLI and the app (ADR-0035)
  - [x] Keyboard entry in the app: a caret on the score, frets (two digits as one
        undo step), note values and dots, rests, delete, clicks that place the
        caret (ADR-0032)
  - [x] Note names at the caret on any pitched staff, chords, semitone moves, in
        the CLI and the app (ADR-0034)
  - [x] Note effects toggled at the caret: ties, legato, slides, bends, vibrato,
        palm mutes, dead and ghost notes, let ring, harmonics, accents,
        staccato, in the CLI and the app: one key each and a panel in the entry
        bar (ADR-0037, ADR-0038)
  - [x] Dynamics at the caret, holding until the next change; new notes take
        the dynamic in force (ADR-0047)
  - [x] Tuplets at the caret: a triplet button and menu in the entry bar,
        Ctrl+2 to Ctrl+9 (ADR-0050)
  - [x] Voices 1 to 4 at the caret: `voice` in the CLI, Alt+1 to Alt+4 and a
        voice button in the app (ADR-0032)
- [x] Copy/paste of measures, insert/delete measures, transposition (ADR-0026, ADR-0027)
  - [x] Measures at the caret in the app: insert before or after, delete, time
        and key signatures from that measure on (ADR-0033)
  - [x] Measure selection in the app: Shift+arrows, select all, copy, paste over
        or insert, transpose (ADR-0036)
- [x] Tuning and capo changes, keeping the pitches or the frets
  - [x] `makeRetune()` and the CLI `tuning` step (ADR-0046)
  - [x] Tuning presets and capo in the song panel
- [x] Edit every element of milestone 1 from the UI
  - [x] Song panel: new song, title, artist, tempo, time and key signatures,
        length, tracks of any instrument, undo and redo, saving (ADR-0031)
  - [x] Every line of the credits (subtitle, album, words, music, copyright,
        transcriber, notes): the CLI `credit` step and the song panel (ADR-0059)
  - [x] Bends of any form at the caret: bend, bend and release, pre-bend,
        pre-bend and release, 1/4 to 2 steps, in the CLI and the effects panel
        (ADR-0055)
  - [x] Strums, pick strokes and tremolo picking on the caret's beat, in the
        CLI and the effects panel (ADR-0056)
  - [x] Kinds of note effects at the caret: slides in and out, harmonics of
        every kind, wide vibrato and marcato, in the CLI and the effects panel
        (ADR-0058)
  - [x] Whammy bar (dives, dips, pre-dives, returns) and tapping, slap and pop
        on the caret's beat, in the CLI and the effects panel (ADR-0060)
  - [x] Dynamics, ppp to fff, from the caret's notes to the next change, in
        the effects panel (ADR-0047)
  - [x] Techniques of strings (hammer-ons, palm mutes, dead notes, harmonics)
        on fretted staves only; the effects panel shows what the caret's
        instrument plays (ADR-0077)
  - [x] Repeats and alternate endings at the caret's measure
    - [x] One undo step, "Change repeats", and the CLI `repeat` and `ending`
          steps (ADR-0016)
- [x] Spanish and English translations (Qt Linguist, ADR-0030)

## Milestone 6 - Practice tools

- [x] Metronome with count-in, A-B loop, speed 25-200 %
  - [x] Speed and metronome in the sequencer, the CLI and the app's transport; playback keeps its place in the music when the speed changes (ADR-0024)
  - [x] Count-in and A-B loop in the player and the CLI (ADR-0024)
  - [x] Count-in and A-B loop in the app's transport; the loop keeps its place in the music when the speed changes
- [x] Tuner: pitch detection, microphone input, the app panel, `elara-cli tune` and `listen` (ADR-0028, ADR-0029)

## Milestone 7 - Distribution

- [x] Installers: MSIX/MSI (Windows), DMG (macOS), AppImage/Flatpak (Linux)
  - [x] Portable packages built and smoke-tested by CI: ZIP (Windows), disk image
        (macOS), tarball (Linux), `elara.py package` (ADR-0048)
  - [x] The app icon on the window, `elara.exe`, the macOS bundle and a Linux
        desktop entry, drawn from its SVG at build time (ADR-0064)
  - [x] An MSI installer (Windows) and an AppImage (Linux), built and
        smoke-tested by CI with `elara.py package --installers` (ADR-0065)
- [ ] Packages built with the vcpkg that CI tests, at the manifest's baseline
      (`docs/release-review.md`)
- [ ] The importers and `elara-cli` run under ASan and UBSan over the test corpus
      in CI (`docs/release-review.md`)
- [ ] Release automation and 1.0

## After 1.0 - Performance

From `docs/optimization-study.md`, smallest and surest first. Each task measures
its effect and keeps the complete path it shortcuts to compare against.

- [ ] Counters of sequences, layouts, rasters and bytes per edit, to measure the
      tasks below
- [ ] An SFZ sample decoded in stereo freed as soon as it is made mono (H05)
- [ ] One sequence built at a time after edits, only the latest edit waiting (H02)
- [ ] Commands that move a shared measure instead of copying it, so the history
      shares measures with the song (H10, H11)
- [ ] Edits of the mix, credits, chord symbols and beat text need no new sequence;
      debug builds check that it would have been equal (H01)
- [ ] The player's events per track prepared on the sequencing worker instead of
      the interface thread (H03)
- [ ] Export as PDF off the interface thread (H06)
