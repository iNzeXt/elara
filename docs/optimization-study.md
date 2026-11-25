# Elara: study of the optimizations left

## About this study

- **Date:** 2026-11-25.
- **Repository:** [ghost-in-the-groove/elara](https://github.com/ghost-in-the-groove/elara).
- **Revision audited:** [`ddc819278915ee311bd3b5cd89d9f2797cccf827`][S01].
- **Commit date:** 2026-11-25 09:17:46 Europe/Madrid (08:17:46 UTC).
- **Method:** static reading of the code, cross-review of the subsystems, and comparison
  with the documentation and benchmarks the project publishes.
- **Scope:** model and commands; QML interface; job scheduling; layout and
  rasterization; sequencing and playback; SoundFont and SFZ; import, export and build.
- **Status of this document:** recommendations awaiting implementation and validation,
  except for the improvements expressly marked as already done.

> Elara was not built and no new benchmarks were run for this study. The times credited
> to the project come from its own log. Memory estimates are marked as calculations,
> not as measurements of the process. A cost mechanism can be confirmed while its
> absolute impact still needs a profile.

## 1. Conclusion

Relevant optimizations remain, several of them structural. The most defensible margin
lies in avoiding work that does not change the result, updating only the regions that
need it, and keeping in check how snapshots, geometry, events and samples coexist in
memory.

The first actions proposed are: invalidation by kind of change; coalescing and
cancelling stale requests; preparing the player entirely off the interface thread;
lowering the peak of SFZ loading; and asynchronous saving and export.

After that, the incremental architecture should be completed with reusable per-track
events, shared pages and graphic systems, and commands that swap shared measures
without copying them needlessly. Budgets for textures and history matter for the goal
of running well on modest machines.

C++/QML and the current boundaries between modules should stay. The causes verified
can be addressed within that architecture. Replacing the renderer entirely,
reorganizing the model as a whole or reading samples in parts must each be justified
by specific profiles.

## 2. How to read the evidence

This report tells four levels apart:

1. **Confirmed in the code:** a call chain, copy, allocation, traversal or policy is
   present at the pinned revision.
2. **Measured by the project:** a figure published in `docs/performance.md`, with the
   machine and revision given there; it does not necessarily match the exact commit
   audited.
3. **Derived:** a memory or complexity calculation from the data and operations in the
   code, under stated assumptions.
4. **Hypothesis to measure:** a reasonable opportunity whose benefit is not
   quantified, or whose priority depends on the size of documents, banks and hardware.

Priority is a technical judgement of probable benefit, frequency and effort. It is not
a proven percentage of improvement. The savings of different findings overlap and must
not be added up as if they were independent.

## 3. What is already solved

The previous study, [docs/performance-study.md][S03], analyses `main` at `4c32790`, on
2026-11-24. Several of its conclusions were overtaken by later changes.

| Area | State checked | Implication |
| --- | --- | --- |
| Song snapshots | Measures are shared, copied on write | Copying a song no longer duplicates all its notes |
| Parallelism | A persistent shared pool, in which the caller takes part | Threads are not created and destroyed on every parallel call |
| Layout | Incremental cache of measurements and systems | A local edit can recompute a small part |
| Sequencing | Background work, separate from layout | Pages do not have to wait for the sequence |
| Rasterization | Background work, and textures kept for pages that did not change | Many repaints are avoided |
| Zoom | A 120 ms wait before repainting a page already drawn | During the gesture the stretched texture can be reused |
| Visible pages | Heavy content activated by a Loader near the viewport | Textures of every page are not kept |
| Faders | A light preview while dragging | The command is committed on release; the document is not rebuilt on every movement |
| Audio | Buffers prepared outside the callback, atomic swap and retirement on the control side | This separation is worth keeping |
| SFZ | Samples shared among the tracks of one bank, and region indexes per key | There is no copy of the PCM per track and no naive search of every region |
| SoundFont | Sounds kept across ordinary compatible edits | Not every instrument is reloaded for each note edited |
| Seek | Binary search of events per track | Placing the cursor does not walk the whole sequence linearly |

**Sources:** [performance log][S04], [shared measures][S05], [worker pool][S06],
[document][S02], [view][S07], [page][S08], [mixer][S09], [player][S10],
[playback events][S11] and [SFZ sampler][S12].

### 3.1. Published figures that serve as reference

The reference machine of the log is an AMD Ryzen 9 5900X with 12 cores and 24 threads,
32 GB of RAM, Windows 11 and MSVC 19.44, with the `release` preset based on
RelWithDebInfo. The large song holds 20 tracks, 500 measures, 80,000 beats and 160,000
notes.

| Operation | Result logged | Caveat |
| --- | --- | --- |
| Copy the song | 11.6 ms before and 63 µs after sharing measures | Later runs vary; not a guarantee for every document |
| Run 64 empty jobs in parallel | 697 µs before and 27 µs with the pool | Measures scheduling overhead, not the time of a whole musical operation |
| Layout after a local edit | 0.46 ms for one track and 4.1 ms for 20 staves | Reuses almost every system; does not describe a change that redistributes the whole piece |
| Sequence the large song | About 30–32 ms in recent runs of the log | Not a new measurement on the audited revision |
| Encode the song as a native file | 70.4 ms in an earlier measurement | Not the full current duration of saving to disk |
| Render one second of 32 SFZ voices | 16.6 ms at first and 10.3 ms after optimizing the interpolation | Not the total CPU use of the app nor the worst callback |

Source: [docs/performance.md][S04]. The log itself warns of drift between sessions and
asks to alternate revisions on the same machine when they need comparing.

## 4. Priority matrix

| ID | Finding | Main benefit | Relevant scenario | Priority | Rough effort |
| --- | --- | --- | --- | --- | --- |
| H01 | Invalidation by kind of change | CPU, allocations and responsiveness | Frequent editing | Very high | Medium |
| H02 | Coalesce and cancel stale jobs | Latency under load and transient memory | Bursts and modest machines | High | Medium |
| H03 | Prepare the player off the interface thread | GUI availability | First Play, banks and instruments | High | Medium-high |
| H04 | Apply sound banks selectively and in one go | Fewer reloads and intermediate states | Sound changes | Medium | Medium |
| H05 | Lower the SFZ loading peak | Peak RAM | Large banks, especially stereo | High | Low-medium |
| H06 | Asynchronous saving and PDF | Responsiveness during long operations | Large scores and slow disks | High | Medium |
| H07 | Per-track events and incremental sequencing | CPU and memory per edit | Many tracks | High | High |
| H08 | Shared geometry and reusable pages | Fewer copies and less RAM | Long documents | Medium-high | High |
| H09 | Global raster/texture budget | RAM and VRAM | HiDPI and high zoom | High for modest machines | Medium; high with tiles |
| H10 | Commands that swap shared measures | CPU and allocations | Transposing, pasting or deleting large ranges | Medium-high | Medium |
| H11 | Memory budget for the history | Retained RAM | Long sessions | Medium-high | Medium |
| H12 | Suspend silent audio work | Idle CPU and battery | Long pause, mute and solo | Medium, to be measured | Medium |
| H13 | Import with owned buffers and in place | Peak RAM | Large XML/GPIF/JSON | Medium | Low-medium |
| H14 | Export with output sinks and bounded buffers | Peak RAM and copy traffic | Large exports | Medium-high | Medium-high |

Efforts are rough, not schedule estimates. Priorities H02, H08, H09, H11 and H12
depend especially on the actual load and the target hardware.

## 5. Main findings

### H01. Invalidate only the subsystems affected

**Status:** mechanism confirmed. **Priority:** very high.

The general path of an edit is `execute()` → `edited()` → `relayout()` →
`resequence()`. `edited()` creates a snapshot and requests a layout. Since it is a new
snapshot, a sequence is requested too. `sequenceFor()` builds a whole Sequence and
then compares it with the previous one; if they are equal it keeps the previous one,
but the work of building it has already been done.

This keeps playback going when the result does not change, yet it still pays for
sequencing and comparing on operations such as committing the mix, credits, texts or
chord symbols. Chord symbols and beat text do not sound in the current model.

**Evidence:** [score_document.cpp][S02], `sequenceFor()` 156–163; `relayout()`
620–645; `execute()`, `edited()` and `setTrackMix()` 1379–1430. [measure.hpp][S26]
documents that chord symbols and beat text do not sound.

**Proposal:** commands produce a descriptor of the kind and extent of the change, on
undo and redo too. It should tell apart musical content, geometry/text,
tempo/navigation, instrument/bank, mix and metadata.

| Operation | Expected invalidation |
| --- | --- |
| Volume, pan, mute, solo | Mix and history |
| Text or chord symbol | Notation and document state |
| Title and credits | Credits rendering and the pagination affected |
| Note change | Notation and playback of the affected regions |
| Instrument change | Sound and the visual properties that depend on it |
| Tempo or repeats | Time dependencies, potentially wide |

The first benefit can be had without incremental sequencing: it is enough to skip it
when the command guarantees that the audible result stays the same.

**Nuance:** dragging a fader already uses `previewTrackMix`; `setTrackMix` is committed
on release. There is no basis for claiming that every pixel of movement causes a
sequencing. Source: [SongPane.qml][S09], 516–571.

**Validation:** count the sequences built, layouts, systems and pages per operation.
Mix and text should produce zero new sequences while keeping exactly their functional
effects and undo/redo. Test separately changes of pitch, duration, tempo, repeats and
instrument.

### H02. Coordinate requests and avoid stale work

**Status:** mechanism confirmed; frequency and impact under normal use still to be
measured. **Priority:** high under load.

Each layout and sequence request can start a QtConcurrent task. Generations discard old
results once they finish. PageLayoutCache holds a mutex through its whole computation,
so several tasks can wait to produce states nobody needs any more. The raster uses
tickets with an equivalent late discard.

Sound bank loads are also identified per request, but an old request can finish
reading and decoding samples before being discarded.

**Evidence:** [score_document.cpp][S02] 598–681; [page_layout.cpp][S13] 581–716;
[score_page.cpp][S08] 120–135; [audio_controller.cpp][S14] 342–404.

**Proposal:** a coordinator per document or flow with one active operation and only
the latest pending request. Check staleness before starting and between expensive
phases. Add cooperative cancellation between systems, tracks or files. Coordinate the
QtConcurrent work budget with the core pool, and give priority to what is visible and
to interactive response.

A task waiting on a mutex should not hold a worker only to produce a revision that is
already outdated. The owner of the layout cache can serialize requests before queueing
them.

**Limits:** the measures of the snapshots are already shared; each task does not
duplicate the whole song. The extra retention may lie in complete results of
sequences, geometry and images. Nor has a significant queue been shown for an isolated
edit whose incremental layout takes a few milliseconds.

**Design risk:** cancelling everything when a key arrives could prevent showing any
result during continuous typing. If intermediate results are allowed, they must keep
the identity of their revision and not restore an old editable model. What is shown
and heard must be explicitly tied to the document it comes from.

**Validation:** bursts of edits, undo, changes of track/view/zoom and bank selection.
Log requests made, run and discarded, time waiting on the mutex, peak concurrency,
total CPU, peak memory and latency from the last input to its presentation.

### H03. Prepare the player entirely off the interface thread

**Status:** call chain confirmed. **Priority:** high.

`AudioController::setSoundFont()` opens the file in the background. However,
`SoundFont::open()` lists presets with a temporary synthesizer and returns an object
holding the path and the presets. Preparing the instrument that will sound happens
later.

`AudioController::loadSequence()` calls `Player::load()` synchronously. That converts
every event to frames, splits them by track, creates buffers and, for new sounds,
calls `SoundFont::sound()`. That function runs `loadSynth()`, `fluid_synth_sfload()`
and `fluid_synth_program_select()`.

**Evidence:** [audio_controller.cpp][S14] 121–139, 160–172 and 342–363;
[player.cpp][S10] 100–142; [playback.hpp][S11] 35–49; [sound_font.cpp][S15] 87–116,
310–339 and 386–426.

Elara enables `synth.dynamic-sample-loading`. The FluidSynth documentation confirms
that selecting presets can load samples and allocate memory. Selection must happen
outside the callback. Source: [official FluidSynth settings][E01],
dynamic-sample-loading.

**Proposal:** separate preparation, publication and retirement. A worker prepares
events, buffers and new instrument instances. Publication hands over a finished state
through a brief swap. The old state is freed outside real time.

Wrapping the current `Player::load()` in QtConcurrent as it stands would not be
enough: it modifies `kept_`, `latest_`, `duration_` and the retirement of states, and
reuses sounds the audio may be using. Clear ownership and generations are needed, and
a worker must not mutate an instance the callback is using.

**Already solved:** compatible sounds are kept across ordinary edits; their samples are
not all reloaded for every note edited. What remains is the global preparation of
events and the cases that do create instruments.

**Validation:** first Play with a cold and a warm cache, changing bank or instrument,
editing a large score after the first Play and doing so during playback. Separate the
time of sequencing, event preparation and instrument creation. Measure long GUI tasks,
first audio, callback durations and dropouts.

### H04. Apply sound banks selectively and in one go

**Status:** mechanism confirmed. **Priority:** medium; complements H03.

`AudioController::useSoundBank()` rebuilds the bank tree when the guitar or bass SFZ
part changes. `Player::setSoundBank()` compares the pointer of the general bank and, if
it changed, empties `kept_`. Then the sounds of tracks whose effective backend may be
the same are recreated as well.

`useBuiltinSound()` walks three parts and calls `useSoundPart()` for each. Every call
ends up applying the bank and loading the sequence, so intermediate states that are
not the final result requested may be built.

**Evidence:** [audio_controller.cpp][S14] 414–418 and 458–499; [player.cpp][S10] 84–91
and 113–120.

**Proposal:** group related changes and apply a single final configuration. Reuse
sounds by a stable key that identifies the effective bank, preset, kind of instrument
and sample rate, instead of relying only on the pointer of the container of every
bank.

**Note on memory:** FluidSynth 2.6.2 has a global sample cache shared between
instances. One synthesizer per track does not prove that all the PCM is duplicated per
track. Synthesizer state, voices, channels, metadata and loading work can be repeated,
though. Source: [FluidSynth 2.6.2 fluid_samplecache.c][E02].

**Validation:** change only the guitar and count how many sounds of other tracks are
recreated. Check that a single prepared state is built when restoring the built-in
sound. Measure change time and transient memory without altering the behaviour of
bends, techniques, mix and held notes.

### H05. Lower the memory peak when loading SFZ samples

**Status:** mechanism confirmed and PCM size computable. **Priority:** high for large
banks.

`readSamples()` loads every file into WavAudio in parallel. Then `SfzBank::load()`
builds the final samples. For mono files it moves the vector; for stereo ones it calls
`mono()`, which creates an extra vector, while the stereo one stays held in `decoded`
until the end.

**Evidence:** [sfz_bank.cpp][S12] 758–765 and 875–893; [wav_reader.cpp][S16],
conversion to mono.

If a fully stereo bank finally needs M bytes of mono PCM, that phase can hold about 2M
of stereo PCM plus M of mono PCM: **3M of PCM**, besides file buffers, decoding and
other structures. This is derived from the code, not a measurement of RSS nor a ratio
to the compressed size of the bank.

**Proposal:** each task reads, decodes, converts and returns the final sample directly,
freeing temporaries before moving on to another. Bound the work in flight by estimated
memory too. As a minimal step, free each stereo vector right after converting it,
though that does not remove the initial peak of having decoded every file.

**Already solved:** one SfzBank shares its samples among the tracks. The aim is to cut
loading temporaries, not to add a sharing that does not exist.

**Conditional evolution:** a sample budget, a cache keyed by file identity and
selective preloading. Reading long tails in parts needs profiles that justify its
complexity; for small banks, loading everything into RAM may remain the right choice.

**Validation:** compare mono, stereo and mixed banks, cold and warm. Log final PCM,
bytes of temporaries, peak RAM and load time. Include changing bank while the previous
one is still playing, and a load cancelled by another request.

### H06. Save and export PDF off the interface thread

**Status:** synchronous paths confirmed. **Priority:** high for interface
availability.

`ScoreDocument::save()` calls `io::saveNative()` directly. `exportPdf()` gets the
fonts, computes the export layout, generates and compresses the PDF and writes it in
the same call.

**Evidence:** [score_document.cpp][S02] 1296–1320 and 1341–1376;
[native_format.cpp][S17] 1074–1106.

The project log recorded 70.4 ms to encode its reference song as a native file at an
earlier revision. That figure shows the possible cost, but does not measure the
current full blocking nor a save to any disk. Source: [performance.md][S04].

**Proposal:** use an immutable snapshot and an asynchronous success/error API.
Serialization, compression, export layout and writing can run on workers. Keep
writing through a temporary file and a final replace, as now.

The revision actually saved must be recorded, so that editing during a save does not
mark later changes as saved. Saves to the same destination must be ordered so that an
old one cannot finish by overwriting a newer one. Flows that close or replace the
document must wait for the right confirmation.

**Limit:** this reduces GUI blocking; it does not necessarily reduce total CPU.
Allowing too many simultaneous saves or exports can raise RAM and contention, so it
must be coordinated with H02.

**Validation:** measure interface availability while saving and exporting the PDF of a
large score. Edit while saving and check the modified state. Test write errors and two
saves in a row. Verify the final content and that no partial files are left.

### H07. Keep per-track events and sequence incrementally

**Status:** global rebuild and reorganization confirmed. **Priority:** high;
structural change.

`Sequence::Sequence()` builds the play order and the time map, computes the notes that
sound and their articulations, generates events per track and merges those lists into
one global list. `Player::load()` splits that global list by track again while
converting times to frames.

A local musical edit can still run this walk over the whole song. Doing the work in
the background avoids part of the interface blocking, but not CPU, allocations or
contention.

**Evidence:** [sequence.cpp][S18] 564–634 and 846–945; [playback.hpp][S11] 35–49;
[player.cpp][S10] 100–109.

**Proposal in stages:**

1. Apply H01 to skip the sequence when the audible result does not change.
2. Keep immutable blocks of events per track as a reusable representation.
3. Generate an ordered global view only for consumers that need it.
4. Reuse the tracks not affected.
5. Then split long tracks into blocks per measure or span, with explicit
   dependencies.

Starting with whole tracks avoids solving all the dependencies between measures right
away. Time information can be shared per version, separating local changes from
global changes of tempo or order.

**Functional risks:** ties, let ring, legato, hairpins, grace notes, tempo and repeats
can widen the extent of an edit. The incremental unit must not naively be the edited
measure. Keep a complete path as an equivalence reference.

Besides, when adopting a new sequence the player resets the sounds. An incremental
publication should keep intact tracks as they are and define what happens to the held
notes of the ones that change. Source: [player.cpp][S10] 227–254.

**Validation:** the same edit on scores of 1, 20 and 100 tracks; exact comparison of
events and times between the complete and the incremental path; tests of techniques
and cross dependencies. Log total CPU, allocations, bytes copied and latency of the
audible update.

### H08. Reuse pages and share their geometry

**Status:** copies after the incremental layout confirmed. **Priority:** medium-high
for memory and scalability.

PageLayoutCache reuses systems, but `stackPages()` copies again the glyphs, lines,
texts, curves and boxes of every system into the assembled pages. The result keeps
geometry both in the system cache and in the pages.

In continuous view a long representation is built and `cutIntoStrips()` walks and
copies its elements into strips. So recomputing a few systems does not mean copying a
few elements in the rest of the pipeline.

**Evidence:** [page_layout.cpp][S13] 333–432, 703–709 and 721–787;
[layout_common.cpp][S19], `appendShifted()`; [ADR on incremental layout][S20].

**Initial proposal:** reuse whole pages when their composition and positions stay the
same. Build the strips directly from the systems where feasible.

**Structural evolution:** immutable shared systems, and pages made of references plus
transforms. Attach reliable revisions to pages and systems to avoid deep comparisons
when recognizing unchanged content.

`ScorePage::repaint()` compares `*page == *shown_` on the interface thread before
keeping a texture. DisplayList equality compares its vectors. A stable identity can
avoid that work, although today the comparison only concerns the active pages and has
not been shown to be a big bottleneck.

`ScoreDocument::page()` returns an aliasing shared_ptr that keeps the whole vector of
pages alive. A job for a single page can keep alive the geometry of that whole
generation; finer-grained ownership can reduce that retention.

**Additional sources:** [score_page.cpp][S08] 110–115; [display_list.hpp][S21]
203–217; [score_document.cpp][S02] 683–688.

**Validation:** resident and peak geometry bytes, allocations and bytes copied per
edit. Include a note that does not change the breaks, an edit that redistributes
systems, continuous view and text that changes heights. Require visual equivalence,
hit testing and caret position against the complete layout.

### H09. Global raster and texture budget

**Status:** per-page limit confirmed; usual consumption not measured. **Priority:**
high if low RAM/VRAM with HiDPI is a goal.

The current limit allows 8192 pixels per side and `32 × 1024 × 1024` pixels per page.
Painting uses premultiplied ARGB32, four bytes per pixel. The maximum area comes to
about **128 MiB per image**.

Raster memory grows roughly as `logical width at 100 % zoom × logical height at 100 %
zoom × zoom² × DPR² × 4`, until the renderer's limits apply. Zoom is a multiplier: 2
for 200 %. DPR is the ratio between physical and logical pixels.

**Evidence:** [score_page.cpp][S08] 20–25 and 96–135; [score_painter.cpp][S22]
286–301; [ScoreView.qml][S07] 409–484.

**Nuances:** 128 MiB is a rough ceiling per image, not a measurement of a usual page.
The QImage is freed after creating the texture in `updatePaintNode()`; no permanent
CPU/GPU duplication has been shown. During transitions, old textures, new images and
pending results can coexist. The driver's actual memory needs measuring.

**Initial proposal:** a global budget in bytes for pending images and textures, with a
limit on simultaneous rasters and priority for what is visible. Heavy content is
already activated near the viewport, but that is not a total memory budget.

**Conditional evolution:** tiles or visible regions of bounded size, with a small
margin and a bounded cache. A local edit could invalidate its region instead of the
whole page image. The C++ model and the QML interface stay.

Switching ARGB32 for RGB32 can change how the alpha channel is handled, but both
formats take four bytes per pixel; it must not be presented as a large direct saving
of RAM.

**Validation:** zoom and scroll at DPR 1 and 2, resizes, dense documents and
continuous view. Measure RAM, graphics memory where it can be observed, number of
images in flight, texture bytes and presentation time. Separate stable resident memory
from transient peaks.

### H10. Swap shared measures in commands

**Status:** potential needless copy confirmed by the access chain. **Priority:**
medium-high in bulk operations.

Staff stores its measures as SharedValue. The mutable accessor `measure()` calls
`edit()`, which copies the content if it is shared. `ReplaceMeasure::swapWith()` asks
for that mutable access before swapping contents; `RemoveMeasures::apply()` also asks
for it before moving them into the history.

With a live snapshot, replacing or removing a measure can copy its old version only to
store it in the undo command. In transposition, the new version was already built by
copying the original, and applying the command can add another copy of the old
version.

**Evidence:** [song.hpp][S23], storage of Staff; [song.cpp][S24] 70–75;
[shared_value.hpp][S25] 24–27; [commands.cpp][S27] 33–44 and 90–93;
[transpose.cpp][S28] 232–237.

**Proposal:** add explicit operations to extract, replace and swap the shared
reference of a measure. The command keeps the old shared version, and mutable access
stays for the changes that really need it. The change is confined to Staff,
SharedValue and the commands; it does not require redoing the whole model.

The clipboard could later benefit from a similar strategy when the tuning and the
operation allow content to be reused. Measure the change to the commands first.

**Limit:** editing a single ordinary measure may cost little. The relevant benefit is
to be sought in wide transpositions, pastes and deletions.

**Validation:** editing one measure, deleting and undoing 100 measures across 20
tracks, transposing everything, and pasting with the same or a different tuning. Keep
a snapshot alive during each operation.

The current benchmarks create a Document without keeping another Song snapshot alive,
so they do not reproduce the copy-on-write split of the GUI. Source:
[bench_core.cpp][S29] 128–145.

As an implementation precaution, the mutable accessor can allocate memory after a
removal of measures has started. When changing this path, exception guarantees and
undo/redo integrity must be kept; no concrete failure has been reproduced.

### H11. Bound the history by memory too

**Status:** policy confirmed. **Priority:** medium-high for long sessions.

`Document::kDefaultUndoLimit` is 1000. The history drops old entries by number of
commands. A transposition, paste or deletion of many measures can be a single step, so
that limit does not determine how many bytes are retained.

**Evidence:** [document.hpp][S30] 38–80; [document.cpp][S31], `execute()`;
[transpose.cpp][S28] and [clipboard.cpp][S32], grouping of commands.

**Proposal:** a memory budget alongside the maximum number of steps, with an explicit
policy for a very large single operation. Roughly account for the data retained only
by the history, without counting each shared reference as another physical copy.

It is also worth measuring the time to destroy large redo branches or a whole history
when opening another song. If that cost is visible, study releasing it outside the
interactive path without compromising object ownership.

**Validation:** sessions of copy/paste, deletion and transposition; partial undo
followed by a new edit; opening another score. Measure stable RAM, peak, memory
retained by the history and the longest GUI pauses. Choose the budget after
measuring, without fixing an arbitrary figure now.

### H12. Suspend audio work that no longer produces sound

**Status:** traversal and rendering of every track confirmed; absolute saving not
measured. **Priority:** medium, of interest for battery life.

Player clears buffers and renders every track. When paused it keeps rendering to let
the tails ring. During playback, muted tracks are synthesized too; the mix discards
the zero gain afterwards. FluidSound keeps calling `fluid_synth_process()` in chunks
even when no voice is active.

**Evidence:** [player.cpp][S10] 303–332, 349–354 and 374–388; [playback.hpp][S11]
100–119; [sound_font.cpp][S15] 207–228; [system_audio.cpp][S33], the device callback.

**Proposal:** explicit states for playback, tail and idle. When no voices, tails or
relevant events remain, a silent path can skip synthesis, clearing and mixing. The
`activeVoices()` interface already exists and can help, although on its own it does
not represent every possible future tail. Source: [sound_bank.hpp][S34].

A muted track needs a different policy: skipping its rendering without advancing its
state can freeze envelopes or alter round robins, release samples and its return to
the mix. The necessary state must be advanced, or rebuilt in a defined way on
reactivation.

For a long pause, removing the source or suspending the device from the control side
can be evaluated. That decision depends on the trade-off between consumption and
resume time.

**Validation:** idle CPU after the tails end, a long pause, and playback with many
tracks muted or soloed. Check re-entry, envelopes, release samples and the absence of
clicks. Keep every allocation, lock and complex release outside the callback.

### H13. Import with owned buffers and in-place parsing

**Status:** full copies of the input confirmed. **Priority:** medium for peak RAM.

Elara loads the whole file; compressed containers also extract the XML or JSON.
MusicXML and GPIF call `pugi::xml_document::load_buffer()`. The native reader uses
yyjson without INSITU mode.

**Evidence:** [archive.cpp][S35], `readFile()`; [musicxml.cpp][S36] 2411–2414;
[guitar_pro.cpp][S37] 1474–1477; [native_format.cpp][S17] 1056–1065.

The pugixml manual confirms that `load_buffer()` copies the text into a private buffer
and that the in-place variants can avoid it if the buffer's lifetime is managed
correctly. Source: [pugixml manual, section 4.2][E03].

**Proposal:** keep the public API for foreign, immutable text and add internal paths
that take ownership of the buffers read or decompressed. Use in-place parsing when the
format, the encoding and the lifetime allow. For JSON, the buffer and padding
requirements of the API used must be respected too.

**Derived impact:** removing one copy of an XML of X bytes can save on the order of X
bytes during that phase. It does not remove the DOM or the building of the model and
does not make import twice as fast.

**Validation:** small and large files, different encodings, compressed and
uncompressed; equal model, warnings and errors; peak RAM and read time. Do not replace
every importer with a different parser up front.

### H14. Export with output sinks and bounded buffers

**Status:** accumulation of complete representations confirmed. **Priority:**
medium-high for large files.

The native writer builds a mutable DOM, generates a JSON buffer and copies it into a
string. The whole JSON is kept while the ZIP is created. `writeZip()` generates the
compressed file in memory and copies its result into a vector.

Parallel MusicXML generates one string per track and then concatenates them into
another complete string. During that concatenation the parts and the unified output
coexist. For MXL, a later phase keeps the XML while creating the ZIP and copies its
bytes into the result.

**Evidence:** [native_format.cpp][S17] 88–115 and 1074–1079; [archive.cpp][S35]
60–89; [musicxml_export.cpp][S38] 1961–1988.

**Nuance:** these representations must not all be added up as if they coexisted
through the whole operation; some buffers are destroyed before the next phase. There
are, though, several phases where complete data is duplicated.

**Proposal:** an output sink abstraction for a file, a buffer or a compressor, keeping
the API that returns bytes where it is needed. Avoid final copies by transferring
ownership or writing directly. For parallel MusicXML, write fragments in order with a
limit on finished fragments pending, instead of keeping all of them plus the final
file.

Emitting JSON directly can be studied later if the profile shows that the DOM and its
allocations matter. Keep writing through a temporary file and a final replace.

**Validation:** identical output, or functional equivalence as the format's contract
defines it, determinism across thread counts, peak memory, bytes copied and total
time. A bounded pipeline may give up part of the parallelism; compare it especially
on machines with little RAM.

## 6. Experiments and conditional opportunities

### C01. IPO/LTO in distribution

The `release` preset selects RelWithDebInfo and packaging uses that preset. No
explicit IPO/LTO was found in the CMake and packaging files reviewed. RelWithDebInfo
is already an optimized configuration; it must not be described as an unoptimized
Debug.

Try a distribution preset with IPO/LTO where the toolchain supports it, keeping
suitable symbols or diagnostics. Compare startup time, CPU, binary size and results.
Do not credit it with a percentage of improvement in advance.

**Sources:** [CMakePresets.json][S39], [CMakeLists.txt][S40], [scripts/elara.py][S41]
and [CMake's CheckIPOSupported][E04].

### C02. Current size of the model

Beat now holds a vector of grace notes and SharedOptional for the chord symbol and
the text. SharedOptional uses shared_ptr; an empty value avoids allocating its object,
but the control member still takes space. The old figure of 64 bytes for Beat must not
be presented as its current size.

Update the size report per platform and add dynamic memory, vector capacities and
bytes per representative song. Give priority to a storage redesign only if that cost
competes significantly with samples, textures, sequences and history.

**Sources:** [measure.hpp][S26], [shared_value.hpp][S25] and
[model memory benchmark][S29].

### C03. Indexes of temporal and musical context

Song::tempoAt walks the headers from the start. Drawing tempo marks looks for the
context before and, for ramps, after. Layout also rebuilds certain dynamic contexts.
In very long scores, with few changes or long stretches of rests, these rescans are
candidates for profiling.

Study indexes of changes or summaries per measure/system, updated from the edited
point until the state matches again. The benefit is not measured. Invalidation must
consider forward and backward dependencies, including tempo ramps and hairpins.

**Sources:** [song.cpp][S24], `tempoAt()`; [system_layout.cpp][S42], `markTempo()`;
[layout_common.cpp][S19] and [page_layout.cpp][S13].

### C04. Fine QML virtualization and outline cache

The current view uses Flickable and Repeater, with one Item and Loader per page. Heavy
content is limited to the visible surroundings; for thousands of pages it may be worth
virtualizing that skeleton and its bindings as well. For normal scores no significant
cost has been shown.

Painting creates fonts and outline caches within its traversal. A cache per worker and
size could be studied after measuring; removing needless rasters and copies has
priority.

The usual QML compilation and cache are already set up through `qt_add_qml_module()`,
with no disabling observed. Enabling that mechanism must not be recommended as if it
were missing, nor should every binding be assumed to compile to native code
automatically.

**Sources:** [ScoreView.qml][S07], [score_painter.cpp][S22],
[the application's CMake][S43] and [qt_add_qml_module documentation][E05].

### C05. Streaming, SIMD and synthesizer topology

Reading samples in parts, extra SIMD, a single multichannel FluidSynth synthesizer or
an entirely different graphics renderer are conditional on specific profiles. Each
brings decisions about state, memory, I/O, routing or quality that the main
improvements will let us evaluate with more information.

This study has no comparative benchmark against Guitar Pro. Nor does it set minimum
RAM requirements, CPU percentages, frame rates or guaranteed global improvements for
Elara.

## 7. Measurement and acceptance plan

### 7.1. Minimal instrumentation

The suite reviewed holds useful microbenchmarks of the model, import/export,
sequencing, synthesizer, sampler and layout. To decide structural priorities they
should be complemented with measurements of the complete experience and of memory.

| Scenario | Measurements | Functional check |
| --- | --- | --- |
| Type and undo quickly | Input to presentation, p50/p95/p99, jobs run and discarded | Consistent final result and caret |
| Edit during playback | Visual and audible latency, maximum callback and dropouts | Correct position, notes and techniques |
| Commit the mix and edit texts | Sequences, layouts and rasters requested | Zero needless sequences |
| First Play and bank change | Preparation, long GUI tasks, first audio, peak RAM | Expected sound and mix |
| Load mono/stereo SFZ | Final PCM, temporaries, peak RAM and time | Equal samples and behaviour |
| HiDPI zoom and scroll | Raster/texture bytes and simultaneous jobs | Sharpness, continuity and caret |
| Transpose, paste and delete | Time, allocations, history RAM | Exact undo/redo |
| Save, import and export | Total time, GUI availability and peak RAM | Correct file, errors and determinism |

**Starting sources:** [bench_core.cpp][S29], [bench_audio.cpp][S44],
[bench_notation.cpp][S45], [bench_io.cpp][S46] and [performance.md][S04].

Log separately stable and peak RAM, geometry, samples, events, history and rasters.
Process RSS is not all graphics memory; on unified-memory systems, counters that
describe the same physical pages must not be added up either.

For audio, measure maxima and dropouts as well as percentiles. As a mathematical
reference, 128 samples at 48 kHz are about 2.67 ms per callback. A comfortable mean
does not prove that every callback meets its deadline.

### 7.2. Representative loads

- Small documents of everyday use and large scores with many tracks.
- The reference case of 20 tracks and 500 measures, for continuity with the log.
- Scores with several voices, ties, legato, let ring, grace notes, hairpins, tempo
  ramps and repeats.
- Varied content besides the repetitive benchmark pattern: compression and musical
  dependencies change.
- Small and large banks, mono and stereo, with cold and warm loads.
- Editing at the same time as playback, bank loading and scrolling the score.
- DPR 1 and 2, different zoom levels and continuous view.
- A machine with two or four cores and limited memory, besides the reference machine.

### 7.3. A/B method

Compare before and after on the same machine, configuration and load. Alternate
executables or revisions when there is drift. Log the distribution and the noise, not
just a mean. The project log already requires this.

Each change must keep the relevant musical and visual equivalence. In incremental
paths, keep the complete rebuild as a comparison oracle. Do not introduce rigid time
thresholds on noisy shared runners without a reproducible methodology.

Quantitative criteria will be set after the baseline. The initial structural criteria
can be concrete: zero sequences for purely visual changes, a single pending request
per flow, reuse of intact tracks and verifiable limits on the memory in flight.

## 8. Recommended implementation order

### Phase A. Measure and remove avoidable work safely

1. Add counters for invalidation, jobs and relevant bytes.
2. Implement H01, semantic invalidation.
3. Implement H02, coalescing requests and cooperative cancellation.
4. Separate preparation and publication of the player, H03.
5. Group and selectively apply banks, H04.
6. Lower the SFZ loading peak, H05.
7. Make saving and PDF asynchronous, H06.

This phase gathers concrete causes and bounded changes. Its fine order can be adjusted
if a measurement of the first Play or of memory shows greater urgency.

### Phase B. Complete the incremental architecture

1. Per-track events and reuse of intact tracks, H07.
2. Reusable pages and shared geometry, H08.
3. Swapping shared measures in commands, H10.
4. History budget, H11.

Each stage must keep the complete path as a reference and measure its effect
separately. The change descriptor of H01 saves each subsystem from rediscovering the
extent of an edit.

### Phase C. Tune budgets and scalability by profiles

1. Global budget of rasters and textures; tiles if justified, H09.
2. Idle state and silent audio work, H12.
3. Owned buffers and in-place parsing on import, H13.
4. Output sinks and bounded buffers, H14.
5. IPO/LTO trials and opportunities C02–C05.

The basic texture budget can be brought forward if HiDPI measurements show significant
pressure. Streaming, complete renderer changes and global redesigns of the model are
subject to further evidence.

## 9. Technical decision

The current architecture already holds the pieces needed for an efficient application:
a model shared per measure, reversible operations, a persistent pool, visual work off
the interface thread and incremental layout. The next step is to extend reuse to the
layers that still rebuild or copy global information.

The most immediate proposals aim to reduce the work per action and the peak memory.
The more structural ones aim to make the cost of an edit grow with the affected region
and its real dependencies, instead of always growing with the whole piece.

The implementation must be judged by measurements of complete latency, CPU, RAM/VRAM
and audio under load, while keeping notation, playback, formats and undo correct. This
document identifies concrete paths to get there, without crediting them with gains
that have not been measured yet.

## 10. References

Every S link points to the pinned commit, so the study stays reproducible as `main`
moves on. E references are official documentation or code of dependencies. The line
ranges given in the findings belong to the audited revision.

### Elara code and documentation

- **S01.** [Audited commit][S01]
- **S02.** [Document and editing][S02]
- **S03.** [Previous study][S03]
- **S04.** [Performance log][S04]
- **S05.** [ADR: shared measures][S05]
- **S06.** [ADR: persistent pool][S06]
- **S07.** [Score view][S07]
- **S08.** [Page and textures][S08]
- **S09.** [Mixer and faders][S09]
- **S10.** [Player][S10]
- **S11.** [Event preparation and playback][S11]
- **S12.** [SFZ bank and sampler][S12]
- **S13.** [Page layout][S13]
- **S14.** [Audio controller][S14]
- **S15.** [SoundFont backend][S15]
- **S16.** [WAV reading and mono conversion][S16]
- **S17.** [Native format][S17]
- **S18.** [Sequencer][S18]
- **S19.** [Layout utilities][S19]
- **S20.** [ADR: incremental layout][S20]
- **S21.** [Display lists][S21]
- **S22.** [Score rasterizer][S22]
- **S23.** [Song and Staff model][S23]
- **S24.** [Model implementation][S24]
- **S25.** [Shared values][S25]
- **S26.** [Measures, notes and beats][S26]
- **S27.** [Editing commands][S27]
- **S28.** [Transposition][S28]
- **S29.** [Model benchmarks][S29]
- **S30.** [Document and history: interface][S30]
- **S31.** [Document and history: implementation][S31]
- **S32.** [Clipboard][S32]
- **S33.** [Audio device backend][S33]
- **S34.** [Bank and sound interface][S34]
- **S35.** [Archives and ZIP][S35]
- **S36.** [MusicXML import][S36]
- **S37.** [Guitar Pro import][S37]
- **S38.** [MusicXML export][S38]
- **S39.** [CMake presets][S39]
- **S40.** [Root CMake][S40]
- **S41.** [Build and packaging tool][S41]
- **S42.** [System layout][S42]
- **S43.** [Application CMake][S43]
- **S44.** [Audio benchmarks][S44]
- **S45.** [Notation benchmarks][S45]
- **S46.** [Import/export benchmarks][S46]

### Dependencies and official documentation

- **E01.** [FluidSynth settings: dynamic sample loading][E01]
- **E02.** [FluidSynth 2.6.2 global sample cache][E02]
- **E03.** [Official pugixml manual: loading from memory][E03]
- **E04.** [CMake: CheckIPOSupported][E04]
- **E05.** [Qt: qt_add_qml_module and the QML cache][E05]

[S01]: https://github.com/ghost-in-the-groove/elara/commit/ddc819278915ee311bd3b5cd89d9f2797cccf827
[S02]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/app/src/score_document.cpp
[S03]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/docs/performance-study.md
[S04]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/docs/performance.md
[S05]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/docs/adr/0051-shared-measures.md
[S06]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/docs/adr/0053-worker-pool.md
[S07]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/app/qml/ScoreView.qml
[S08]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/app/src/score_page.cpp
[S09]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/app/qml/SongPane.qml
[S10]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/audio/src/player.cpp
[S11]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/audio/src/playback.hpp
[S12]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/audio/src/sfz_bank.cpp
[S13]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/notation/src/page_layout.cpp
[S14]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/app/src/audio_controller.cpp
[S15]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/audio/backend_fluid/src/sound_font.cpp
[S16]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/audio/src/wav_reader.cpp
[S17]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/io/src/native_format.cpp
[S18]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/audio/src/sequence.cpp
[S19]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/notation/src/layout_common.cpp
[S20]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/docs/adr/0052-incremental-relayout.md
[S21]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/notation/include/elara/notation/display_list.hpp
[S22]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/app/src/score_painter.cpp
[S23]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/core/include/elara/core/song.hpp
[S24]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/core/src/song.cpp
[S25]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/core/include/elara/core/shared_value.hpp
[S26]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/core/include/elara/core/measure.hpp
[S27]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/core/src/commands.cpp
[S28]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/core/src/transpose.cpp
[S29]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/tests/bench/bench_core.cpp
[S30]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/core/include/elara/core/document.hpp
[S31]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/core/src/document.cpp
[S32]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/core/src/clipboard.cpp
[S33]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/audio/backend_juce/src/system_audio.cpp
[S34]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/audio/include/elara/audio/sound_bank.hpp
[S35]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/io/src/archive.cpp
[S36]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/io/src/musicxml.cpp
[S37]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/io/src/guitar_pro.cpp
[S38]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/io/src/musicxml_export.cpp
[S39]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/CMakePresets.json
[S40]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/CMakeLists.txt
[S41]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/scripts/elara.py
[S42]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/notation/src/system_layout.cpp
[S43]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/src/app/CMakeLists.txt
[S44]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/tests/bench/bench_audio.cpp
[S45]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/tests/bench/bench_notation.cpp
[S46]: https://github.com/ghost-in-the-groove/elara/blob/ddc819278915ee311bd3b5cd89d9f2797cccf827/tests/bench/bench_io.cpp
[E01]: https://www.fluidsynth.org/api/settings_synth.html#settings_synth_dynamic-sample-loading
[E02]: https://github.com/FluidSynth/fluidsynth/blob/v2.6.2/src/sfloader/fluid_samplecache.c
[E03]: https://pugixml.org/docs/manual.html
[E04]: https://cmake.org/cmake/help/latest/module/CheckIPOSupported.html
[E05]: https://doc.qt.io/qt-6/qt-add-qml-module.html
