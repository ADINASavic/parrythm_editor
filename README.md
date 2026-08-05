# Parrythm Editor Guide

##https://www.notion.so/27102ad4e0a1802b90bbfee8ed14f30f 한글에디션


This document explains how the web-based editor behaves and how its data maps to the in-game runtime. The UI is driven by `EditorApp/public/legacy/markup.html`, `EditorApp/src/legacy/core.js`, and `EditorApp/src/App.jsx`, while runtime parsing and input are defined in `Assets/Features/Systems/Script/Parsiing/LevelChartData.cs`, `Assets/Features/Systems/Script/Parsiing/Jsonify.cs`, and `Assets/Features/Systems/Script/KeyboardInputManager.cs`. A Unity-side JSON helper exists at `Assets/Features/Editor/Jsonify.cs`.

## Shortcuts and Input

### Keyboard-only charting

Mode keys switch the active note Type without touching the dropdown:

| Key | Type |
|---|---|
| `G` | Grid |
| `T` | Trail |
| `Y` | Long |
| `R` | Slider |
| `F` | CircleTap |

Cell keys place a note in one keystroke, laid out exactly like the in-game 3x3 grid:

```
Q W E   →  (0,2) (1,2) (2,2)
A S D   →  (0,1) (1,1) (2,1)
Z X C   →  (0,0) (1,0) (2,0)
```

Long notes are placed with two presses. The first marks a start for that cell and the cell turns
amber with a `LONG` badge; the waveform also shows a dashed marker (or an edge arrow when the start
is scrolled off screen). The **next note placed in that same cell** becomes the end and is absorbed
into the Long. Pending state is per cell, so other cells stay free. Placing the end earlier than the
start swaps the two. Pending Longs that are never closed are simply discarded — they never reach the
saved chart. `Ctrl+Z` on a closed Long returns it to pending.

All of the above are ignored while a text field has focus, and any modifier (`Ctrl`/`Cmd`/`Alt`)
falls through to the existing shortcuts.

### Placement snap (CircleTap / Slider)

With Type set to CircleTap or Slider, a Snap panel appears. `Distance` fixes the gap from the
previous point and lets the mouse choose direction (optionally quantised by Angle Step);
`Direction` also reuses the previous two points' heading, so the mouse position is ignored.
CircleTap anchors on the previously placed note; Slider anchors on the previous point of the path
currently being drawn.

A CircleTap is one point, so a canvas click places the note immediately — no confirm step. The
`Add CircleTap` button stays for placing at a typed `Pos` instead, and while a note is open for
editing a click only moves its position. `찍을 때마다 재생막대를 스냅 한 칸 뒤로` then advances the
playhead by one snap subdivision after each placement — the same step `Arrow Right` uses, so it
follows the Snap/Subdiv setting rather than a separate amount. Placing and stepping stay in sync
because both go through `seekBySubdivision`. A slider has no fixed point count, so its path is built by
clicking the canvas repeatedly and then confirmed from the left inspector with `Add Slider`;
`Path pts` shows the current count.

### Special Bullet — Line, picked on the canvas

The `Line` pattern spreads bullets along the polyline in `Line Points` **by arc length**, so bullet
`i` takes the same normalised position on every path regardless of where the corners fall. That
field is the single source of truth and stays hand-editable; `Pick Points` just fills it from the
canvas.

Each of Spawn / Dock / Despawn owns its own `Pick Points` and `Clear Path` buttons, shown only while
that panel's pattern is `Line`. `Pick Points` arms the panel — every canvas click then appends one
point to its `Line Points` — and pressing it again disarms. Only one panel is armed at a time, so
picking Dock is just a matter of pressing Dock's button. **Canvas clicks are ignored unless a panel
is armed**, and switching an armed panel away from `Line` disarms it automatically.

A path takes any number of points, so an L or a zigzag works, not just a straight segment. Every
click after the first obeys the same Snap settings as Slider and CircleTap. Paths are drawn on the
canvas in Spawn green / Dock amber / Despawn red, with the armed panel thickened.

### Special Bullet — Slider Path

`Slider Path` is a **dock-only** pattern — it appears in the Dock dropdown and nowhere else. It
means "these bullets dock onto that slider": the dock position comes from the path and, with
`타이밍을 슬라이더 진행에 맞춤`, the dock time comes from when the player reaches that progress.
Spawn and despawn coordinates stay completely independent, set by whatever pattern their own panels
use. Its settings live in the Dock panel and appear when the dock pattern is `Slider Path`.

`Slider Path Target` picks the slider. Rather than hunting through the dropdown, press
`Pick on Wave` and click the slider's block in the waveform panel; that selects it and disarms. A
click on anything that is not a slider says so and leaves picking armed, and while armed a wave
click never opens the note editor. Unarmed, wave clicks behave normally.

`Slider Progress Start/End` sets the span the batch covers, so bullet `i` docks at an evenly spaced
progress along it. It defaults to the whole path, 0 → 1.

The path sampled is the **same curve the slider itself draws** — `buildPath(points, sliderCurved,
CurveMode.ROUND)`. On a curved slider the dock therefore follows the rounded B-spline, not the raw
control polyline; on a sharp corner the two differ by well over 100px.

Everything except dock is kept clear of the path automatically. Any spawn or despawn point closer
than the circle radius (`chart.circleRadius`, the reflect hit radius) is pushed straight out to
exactly that distance, re-checked a few times since one push can move a point near a different
segment. Dock is never moved — it is supposed to be on the path.

### Special Bullet — Despawn `Continue`

The despawn dropdown has one extra pattern the others don't: `Continue (Spawn→Dock 직진)`. Instead
of a shape of its own, it takes the spawn→dock direction and carries straight on until the bullet
leaves the screen, so the bullet never bends at dock. `Continue Margin (px)` is how far past the
1920×1080 edge the despawn point lands. It uses the spawn position *after* clearance has been
applied, so the preview and the generated notes stay identical.

With `타이밍을 슬라이더 진행에 맞춤` enabled (default), the `Start`/`Step` fields are ignored — the
batch is anchored to the slider instead of to its own start time. Each bullet's dock time is
`slider start + duration × progress` and its spawn time is that minus `ToDock`, so the bullet lands
exactly when the player's head reaches that point. Because sliders render in `Round` mode, the docks
land on the real runtime path rather than on the authored control points.

The slider's start and duration are resolved the same way `ChartTimeLineRuntime.StartSec/DurSec`
does: seconds fields in Seconds mode, `startBeat`/`durationBeats` in Beats mode. Reading the seconds
fields in Beats mode would be wrong — they are not kept in sync there.


The editor is designed around keyboard-first editing. You can open and close the Quick Guide overlay with `?` or `Shift+/`, and dismiss it with `Esc`. Playback toggles with `Space`, while `Arrow Left/Right` seeks by the current snap unit. Editing history follows standard conventions: `Ctrl+Z` for undo and `Ctrl+Shift+Z` or `Ctrl+Y` for redo. Range copy/paste uses `Ctrl/Cmd+C` and `Ctrl/Cmd+V`. For grid selection, `Numpad1~9` picks a 3x3 cell and `Numpad+` cycles the next cell; `Numpad0` immediately adds a note for the active Type (Grid/Trail/Long). Slider point editing is intended to be quick: arrow keys nudge by 5px, `Shift`+Arrow nudges by 20px, `PageUp/PageDown` moves the selected point, and `Enter` applies the `ptXY` input. Line(Style) events can be added with `Enter` while focus is in the Line inputs, and anywhere with `Ctrl/Cmd+Enter`. Mouse gestures focus on timeline editing: click the waveform to seek/set time, drag to pan, `Shift`+drag to select a range, and `Ctrl`+wheel to zoom. Clicking a waveform note enters edit mode; `Ctrl/Cmd` click toggles selection and `Shift` click extends a range.

## In-Game Key Mapping (3x3 Grid)

The in-game 3x3 grid input is fixed: (0,0) is `Keypad1`/`Alpha1`/`Z`, (1,0) is `Keypad2`/`Alpha2`/`X`, (2,0) is `Keypad3`/`Alpha3`/`C`; (0,1) is `Keypad4`/`Alpha4`/`A`, (1,1) is `Keypad5`/`Alpha5`/`S`, (2,1) is `Keypad6`/`Alpha6`/`D`; (0,2) is `Keypad7`/`Alpha7`/`Q`, (1,2) is `Keypad8`/`Alpha8`/`W`, (2,2) is `Keypad9`/`Alpha9`/`E`.

## Editor UI Features and Fields

### Add/Edit Note

Note creation always starts by choosing a type in `noteType`. When a note is selected for editing, the editor shows `editBar` and `editLabel`, and the Save/Delete/Cancel buttons invoke `saveEdit()`, `deleteEdit()`, and `cancelEdit()`. Grid/Trail/Long notes use `cellCR` for the grid position and `judgeTimeSec`/`judgeBeat` for timing. Long notes also require `longDurSec` and `longDurBeats` and are created with `btnAddGrid`. CircleTap is defined by a position (`circlePos`) and a timing pair (`circleJudgeSec`/`circleJudgeBeat`), then added by `btnAddCircle`. CircleTap/Slider positions are clamped to the center 4:3 play area (x `240~1680`, y `0~1080`) including `circleRadius`.

Sliders are built from a start time (`sliderStartSec`/`sliderStartBeat`), a duration (`sliderDurSec`/`sliderDurBeats`), and a path; `sliderCurved` toggles bezier interpolation. The active point is addressed by `ptIndex` and edited with `ptXY`, while the list view (`sliderPointList`) and `pathCount` show the current path state. The path control buttons?`finishPath`, `clearPath`, `applyPointXY`, `insertPointAfter`, `removePoint`?respectively finalize, reset, apply input, insert after, and remove points. `btnAddSlider` pushes the final note.

Bullets are defined by three anchor positions (`bulletSpawn`, `bulletDock`, `bulletDespawn`) plus a start time (`bulletStartSec`/`bulletStartBeat`). Movement timing is split into Spawn->Dock (`bulletToDockSec`/`bulletToDockBeats`) and Dock->Despawn (`bulletDockToDespawnSec`/`bulletDockToDespawnBeats`). For canvas-driven placement, `bulletClickTarget` selects which anchor is being clicked, `bulletCurved` enables bezier motion, `bullet3Click` enables three-click placement, and `bulletAutoAdd` automatically creates the note after those clicks; `bulletStageLabel` shows the current stage. Manual add uses `btnAddBullet`.

Special Bullet patterns generate batches of Bullet notes. The base pattern is chosen with `spPattern` and described in `spPatternHelp`. Common parameters include `spCount`, `spCenter`, `spRadius`, `spRadiusStep`, `spStartAngle`, `spArc`, `spTurns`, `spWaveAmp`, `spWaveFreq`, `spGridRows`, `spGridCols`, `spGridSpaceX`, `spGridSpaceY`, `spLineLength`, `spLinePoints`, and `spBurstRadius`. Dock patterns mirror the same idea with `spDockPattern`, `spDockHelp`, `spDockCenter`, `spDockRadius`, `spDockRadiusStep`, `spDockStartAngle`, `spDockArc`, `spDockTurns`, `spDockWaveAmp`, `spDockWaveFreq`, `spDockGridRows`, `spDockGridCols`, `spDockGridSpaceX`, `spDockGridSpaceY`, `spDockLineLength`, `spDockLinePoints`, and `spDockBurstRadius`. Despawn patterns use `spDespawnPattern`, `spDespawnHelp`, `spDespawnCenter`, `spDespawnRadius`, `spDespawnRadiusStep`, `spDespawnStartAngle`, `spDespawnArc`, `spDespawnTurns`, `spDespawnWaveAmp`, `spDespawnWaveFreq`, `spDespawnGridRows`, `spDespawnGridCols`, `spDespawnGridSpaceX`, `spDespawnGridSpaceY`, `spDespawnLineLength`, `spDespawnLinePoints`, and `spDespawnBurstRadius`. Timing is defined by `spStartSec`/`spStartBeat` and `spStepSec`/`spStepBeat`, with travel durations `spToDockSec`/`spToDockBeats` and `spDockToDespawnSec`/`spDockToDespawnBeats`; `spCurved` toggles bezier, and `btnAddSpecialBullets` performs the add.

Camera notes specify timing (`camStartSec`/`camDurSec` or `camStartBeat`/`camDurBeats`) and then selectively apply height, angle, or center with `camAffectHeight`/`camHeightPx`, `camAffectAngle`/`camAngleDegZ`, and `camAffectCenter`/`camCenterPx`. `camEase` selects easing, `camUseBeats` enables beat-based timing, and `camUseUIScaling` applies UI scaling; `btnAddCamera` adds the note.

Line(Style) events are used for grid color/line styling. You select the target with `stTarget` and `stIndex`, set timing with `stTimeSec`/`stTimeBeat`, and define fades with `stFadeBeats`/`stFadeSec`. Colors are entered via `stColorText`, `stColorPick`, `stAlpha`, and `stAlphaVal`, and the event is added with `stAddBtn`. Voice notes bind an FMOD path in `voiceEventPath`, set timing with `voiceStartSec`/`voiceStartBeat`, set fades with `voiceFadeInSec`/`voiceFadeOutSec`, and level/pitch with `voiceVolume`/`voicePitch`. Image notes bind an asset key `imgKey`, define timing with `imgStartSec`/`imgStartBeat` and `imgDurSec`/`imgDurBeats`, then position/size/visuals with `imgPos`, `imgSize`, `imgAlpha`, `imgRotDeg`, `imgFadeInSec`, and `imgFadeOutSec`.

### Chart Settings

The chart panel is toggled by `chartToggle` and edited inside `chartSettings`. Metadata uses `title`, `bpm`, `artist`, `previewStart`, `previewDuration`, `tags`, `coverFile`, and the selected `audioFile`. Tempo and time signatures are configured via `tempoMapEnabled`, `tempoAddBtn`, `tempoMapList`, `timeSigMapEnabled`, `timeSigAddBtn`, and `timeSigMapList`. Time mode is selected with `timeMode` (Seconds/Beats), start offsets with `startOffsetSec`/`startOffsetBeats`, and explicit duration with `durationSec`. `circleRadius` is a song-level radius used for editor clamp/preview and is exported to `song.json`.

### Spawn Calc Settings

Spawn timing uses `spawnToggle` and the fields under `spawnSettings`. Leads are `circleLead`/`circleLeadBeats` and `sliderLead`/`sliderLeadBeats`. Grid growth is defined by `gridTarget`, `gridGrow`, and `gridGrowUnit`, while target mode uses `gridTargetMode`. Bullet size is `bulletSize` and circle target radius is `circleTargetRadius` (linked with `circleRadius`).

### Audio/Waveform

Audio is loaded through `audioFile`, played or paused with `playBtn`/`pauseBtn`, and history is tracked with `undoBtn`/`redoBtn`. The timeline display uses `curTime`, `durTime`, and `curBeat`. Snap subdivisions are chosen in `subdiv`, and when custom, the value is read from `subdivCustomWrap`/`subdivCustom`. The waveform add mode is set by `waveNoteType`, playback speed by `playbackRate`, snapping by `snap`, beat seek sync by `syncBeatSeek`, and zoom/fit by `zoomInBtn`, `zoomOutBtn`, and `fitBtn`. Amplitude scale uses `ampScale`, the scrollbar is `waveScroll`, and the actual rendering/playback elements are `wave` and `audio`.

### Preview/Mini Panels

The preview canvas is `view` and the grid UI is `grid`. The mini notes list is split into `miniNotes`, `miniList`, and sort control `miniSort`, with mini playback controlled by `miniPlayBtn` and `miniPauseBtn`. The mini waveform renders to `waveMini`.

### Note Ordering/Filtering/Multi Edit

Ordering and reindexing lives in `orderToolbar` with buttons for creation/time(spawn)/time(sec)/time(beats) plus `reindexNotes`. Filtering uses `noteFilterText` and `noteFilterType`. Multi edit uses `multiEditBar` with `multiSelCount`, `multiDeltaSec`, `multiDeltaBeat`, `multiApply`, `multiDelete`, `multiSelectAll`, and `multiClear`. The table view is `notesTable`, with row click to edit and drag to reorder.

### Import/Export

Exports are `downloadJSON()` for JSON and `exportSongZip()` for a song folder ZIP. Imports use `importFile`, and full reset uses `clearAll()`.

## Export Output (song.json)

`exportSongZip()` produces `charts/Normal.chart.json`, `song.json`, and the selected audio file in the song folder root. The `song.json` payload includes `title`, `artist`, `bpm`, `circleRadius`, `tempoMapEnabled`, `tempoMap`, `timeSigMapEnabled`, `timeSigMap`, `previewStart`, `previewDuration`, `coverImage`, `bgVideo`, `audioFile`, `durationSec`, `difficulties` (name/path/level), and `tags`.

## In-Game Schema (LevelChartData)

Chart-level fields include `title`, `bpm`, `timemode`, `startOffset`, `startOffsetBeats`, `tempoMapEnabled` with `tempoMap` (each entry uses `beat`, `bpm`), `timeSigMapEnabled` with `timeSigMap` (each entry uses `beat`, `numerator`, `denominator`), and `notes`. Every note shares `type`, `judgeTime`, `judgeBeat`, `startTime`, `startBeat`, `duration`, `durationBeats`, `timeSec`, `fadeSec`, `isTrail`, and `isLong`. Grid/Trail/Long add `cell` and reuse the timing fields; CircleTap uses `absPos` plus `judgeTime`/`judgeBeat`. Slider uses `absPath`, `sliderCurved`, `startTime`/`startBeat`, and `duration`/`durationBeats`. Bullet uses `bulletSpawnPos`, `bulletDockPos`, `bulletDespawnPos`, `bulletSpawnTime`/`bulletSpawnBeat`, `bulletToDockDuration`/`bulletToDockBeats`, `bulletDockToDespawnDuration`/`bulletDockToDespawnBeats`, and `bulletCurved`. Camera uses `camAffectHeight`, `camAffectAngle`, `camAffectCenter`, `camHeightPx`, `camAngleDegZ`, `camCenterPx`, `camUseBeats`, `camUseUIScaling`, and `camEase`. Line(Style) uses `timeSec`, `timeBeat`, `target` (grid-all/grid-line), `index`, `color`, `fadeSec`, and `fadeBeat`. Voice uses `fmodEvent`, `volume`, `pitch`, `fadeInSec`, `fadeOutSec`, `fadeInBeat`, `fadeOutBeat`, and `voiceEventPath` as a fallback. Image uses `imagePath`, `imageSizePx`, `imageScale`, `imageAnchor`, `imageRotationDeg`, `imageOpacity`, `imageFadeInSec`, `imageFadeOutSec`, `imageFadeInBeat`, `imageFadeOutBeat`, `imageZIndex`, and `imageBlend`.

## JSON Schema (Editor/Import)

The top-level is RootStr (string schema) or RootNum (numeric schema) with `schemaVersion`, `title`, `bpm`, `timeMode` (string: `Beats`/`Seconds`, number: 0/1), `startOffset`, `startOffsetBeats`, `tempoMapEnabled`, `tempoMap` (each entry uses `beat`, `bpm`), `timeSigMapEnabled`, `timeSigMap` (each entry uses `beat`, `numerator`, `denominator`), and `notes`.

NoteStr (string type) includes `type`, `cell`, `absPos`, `absPath`, `sliderCurved`, `isTrail`, `isLong`, timing fields `judgeTime`, `judgeBeat`, `startTime`, `startBeat`, `timeSec`, `duration`, `durationBeats`, `durationSec`, `fadeSec`, bullet fields `bulletSpawnPos`, `bulletDockPos`, `bulletDespawnPos`, `bulletSpawnTime`, `bulletSpawnBeat`, `bulletToDockDuration`, `bulletToDockBeats`, `bulletDockToDespawnDuration`, `bulletDockToDespawnBeats`, `bulletCurved`, camera fields `camAffectHeight`, `camAffectAngle`, `camAffectCenter`, `camHeightPx`, `camAngleDegZ`, `camCenterPx`, `camUseBeats`, `camUseUIScaling`, `camEase`, line fields `timeBeat`, `target`, `index`, `fadeBeat`, and the color variants `color`, `colorRGBA`, `colorArray`, `colorVec4`, `colorR`, `colorG`, `colorB`, `colorA`. Voice uses `fmodEvent`, `volume`, `pitch`, `fadeInSec`, `fadeInBeat`, `fadeOutSec`, `fadeOutBeat`, and Image uses `imagePath`, `sizePx`, `scale`, `anchor`, `rotationDeg`, `opacity`, `zIndex`, `blend`, `key`, `rotDeg`, `alpha`.

NoteNum (numeric type) includes `type` (0~8), `cell`, `absPos`, `absPath`, `sliderCurved`, `isTrail`, `isLong`, timing fields `judgeTime`, `judgeBeat`, `startTime`, `startBeat`, `timeSec`, `duration`, `durationBeats`, `durationSec`, `fadeSec`, bullet fields `bulletSpawnPos`, `bulletDockPos`, `bulletDespawnPos`, `bulletSpawnTime`, `bulletSpawnBeat`, `bulletToDockDuration`, `bulletToDockBeats`, `bulletDockToDespawnDuration`, `bulletDockToDespawnBeats`, `bulletCurved`, camera fields `camAffectHeight`, `camAffectAngle`, `camAffectCenter`, `camHeightPx`, `camAngleDegZ`, `camCenterPx`, `camUseBeats`, `camUseUIScaling`, `camEase`, line fields `timeBeat`, `target`, `index`, `fadeBeat`, `color`, `colorRGBA`, `colorArray`, `colorVec4`, `colorR`, `colorG`, `colorB`, `colorA`, voice fields `fmodEvent`, `volume`, `pitch`, `fadeInSec`, `fadeInBeat`, `fadeOutSec`, `fadeOutBeat`, and image fields `imagePath`, `sizePx`, `scale`, `anchor`, `rotationDeg`, `opacity`, `zIndex`, `blend`.

Parser notes: the string `type` value `Style` maps to Line, and `hold`/`longnote` map to Long. Both `timeMode` and `type` accept string or numeric values.
