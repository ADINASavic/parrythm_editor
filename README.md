# Parrythm Editor Guide

This document explains how the web-based editor behaves and how its data maps to the in-game runtime. The UI is driven by `EditorApp/public/legacy/markup.html`, `EditorApp/src/legacy/core.js`, and `EditorApp/src/App.jsx`, while runtime parsing and input are defined in `Assets/Features/Systems/Script/Parsiing/LevelChartData.cs`, `Assets/Features/Systems/Script/Parsiing/Jsonify.cs`, and `Assets/Features/Systems/Script/KeyboardInputManager.cs`. A Unity-side JSON helper exists at `Assets/Features/Editor/Jsonify.cs`.

## Shortcuts and Input

The editor is designed around keyboard-first editing. You can open and close the Quick Guide overlay with `?` or `Shift+/`, and dismiss it with `Esc`. Playback toggles with `Space`, while `Arrow Left/Right` seeks by the current snap unit. Editing history follows standard conventions: `Ctrl+Z` for undo and `Ctrl+Shift+Z` or `Ctrl+Y` for redo. Range copy/paste uses `Ctrl/Cmd+C` and `Ctrl/Cmd+V`. For grid selection, `Numpad1~9` picks a 3x3 cell and `Numpad+` cycles the next cell; `Numpad0` immediately adds a note for the active Type (Grid/Trail/Long). Slider point editing is intended to be quick: arrow keys nudge by 5px, `Shift`+Arrow nudges by 20px, `PageUp/PageDown` moves the selected point, and `Enter` applies the `ptXY` input. Line(Style) events can be added with `Enter` while focus is in the Line inputs, and anywhere with `Ctrl/Cmd+Enter`. Mouse gestures focus on timeline editing: click the waveform to seek/set time, drag to pan, `Shift`+drag to select a range, and `Ctrl`+wheel to zoom. Clicking a waveform note enters edit mode; `Ctrl/Cmd` click toggles selection and `Shift` click extends a range.

## In-Game Key Mapping (3x3 Grid)

The in-game 3x3 grid input is fixed: (0,0) is `Keypad1`/`Alpha1`/`Z`, (1,0) is `Keypad2`/`Alpha2`/`X`, (2,0) is `Keypad3`/`Alpha3`/`C`; (0,1) is `Keypad4`/`Alpha4`/`A`, (1,1) is `Keypad5`/`Alpha5`/`S`, (2,1) is `Keypad6`/`Alpha6`/`D`; (0,2) is `Keypad7`/`Alpha7`/`Q`, (1,2) is `Keypad8`/`Alpha8`/`W`, (2,2) is `Keypad9`/`Alpha9`/`E`.

## Editor UI Features and Fields

### Add/Edit Note

Note creation always starts by choosing a type in `noteType`. When a note is selected for editing, the editor shows `editBar` and `editLabel`, and the Save/Delete/Cancel buttons invoke `saveEdit()`, `deleteEdit()`, and `cancelEdit()`. Grid/Trail/Long notes use `cellCR` for the grid position and `judgeTimeSec`/`judgeBeat` for timing. Long notes also require `longDurSec` and `longDurBeats` and are created with `btnAddGrid`. CircleTap is defined by a position (`circlePos`) and a timing pair (`circleJudgeSec`/`circleJudgeBeat`), then added by `btnAddCircle`.

Sliders are built from a start time (`sliderStartSec`/`sliderStartBeat`), a duration (`sliderDurSec`/`sliderDurBeats`), and a path; `sliderCurved` toggles bezier interpolation. The active point is addressed by `ptIndex` and edited with `ptXY`, while the list view (`sliderPointList`) and `pathCount` show the current path state. The path control buttons?`finishPath`, `clearPath`, `applyPointXY`, `insertPointAfter`, `removePoint`?respectively finalize, reset, apply input, insert after, and remove points. `btnAddSlider` pushes the final note.

Bullets are defined by three anchor positions (`bulletSpawn`, `bulletDock`, `bulletDespawn`) plus a start time (`bulletStartSec`/`bulletStartBeat`). Movement timing is split into Spawn->Dock (`bulletToDockSec`/`bulletToDockBeats`) and Dock->Despawn (`bulletDockToDespawnSec`/`bulletDockToDespawnBeats`). For canvas-driven placement, `bulletClickTarget` selects which anchor is being clicked, `bulletCurved` enables bezier motion, `bullet3Click` enables three-click placement, and `bulletAutoAdd` automatically creates the note after those clicks; `bulletStageLabel` shows the current stage. Manual add uses `btnAddBullet`.

Special Bullet patterns generate batches of Bullet notes. The base pattern is chosen with `spPattern` and described in `spPatternHelp`. Common parameters include `spCount`, `spCenter`, `spRadius`, `spRadiusStep`, `spStartAngle`, `spArc`, `spTurns`, `spWaveAmp`, `spWaveFreq`, `spGridRows`, `spGridCols`, `spGridSpaceX`, `spGridSpaceY`, `spLineLength`, `spLinePoints`, and `spBurstRadius`. Dock patterns mirror the same idea with `spDockPattern`, `spDockHelp`, `spDockCenter`, `spDockRadius`, `spDockRadiusStep`, `spDockStartAngle`, `spDockArc`, `spDockTurns`, `spDockWaveAmp`, `spDockWaveFreq`, `spDockGridRows`, `spDockGridCols`, `spDockGridSpaceX`, `spDockGridSpaceY`, `spDockLineLength`, `spDockLinePoints`, and `spDockBurstRadius`. Despawn patterns use `spDespawnPattern`, `spDespawnHelp`, `spDespawnCenter`, `spDespawnRadius`, `spDespawnRadiusStep`, `spDespawnStartAngle`, `spDespawnArc`, `spDespawnTurns`, `spDespawnWaveAmp`, `spDespawnWaveFreq`, `spDespawnGridRows`, `spDespawnGridCols`, `spDespawnGridSpaceX`, `spDespawnGridSpaceY`, `spDespawnLineLength`, `spDespawnLinePoints`, and `spDespawnBurstRadius`. Timing is defined by `spStartSec`/`spStartBeat` and `spStepSec`/`spStepBeat`, with travel durations `spToDockSec`/`spToDockBeats` and `spDockToDespawnSec`/`spDockToDespawnBeats`; `spCurved` toggles bezier, and `btnAddSpecialBullets` performs the add.

Camera notes specify timing (`camStartSec`/`camDurSec` or `camStartBeat`/`camDurBeats`) and then selectively apply height, angle, or center with `camAffectHeight`/`camHeightPx`, `camAffectAngle`/`camAngleDegZ`, and `camAffectCenter`/`camCenterPx`. `camEase` selects easing, `camUseBeats` enables beat-based timing, and `camUseUIScaling` applies UI scaling; `btnAddCamera` adds the note.

Line(Style) events are used for grid color/line styling. You select the target with `stTarget` and `stIndex`, set timing with `stTimeSec`/`stTimeBeat`, and define fades with `stFadeBeats`/`stFadeSec`. Colors are entered via `stColorText`, `stColorPick`, `stAlpha`, and `stAlphaVal`, and the event is added with `stAddBtn`. Voice notes bind an FMOD path in `voiceEventPath`, set timing with `voiceStartSec`/`voiceStartBeat`, set fades with `voiceFadeInSec`/`voiceFadeOutSec`, and level/pitch with `voiceVolume`/`voicePitch`. Image notes bind an asset key `imgKey`, define timing with `imgStartSec`/`imgStartBeat` and `imgDurSec`/`imgDurBeats`, then position/size/visuals with `imgPos`, `imgSize`, `imgAlpha`, `imgRotDeg`, `imgFadeInSec`, and `imgFadeOutSec`.

### Chart Settings

The chart panel is toggled by `chartToggle` and edited inside `chartSettings`. Audio linkage uses `fmodEvent`, and metadata uses `title`, `bpm`, `artist`, `previewStart`, `previewDuration`, `tags`, and `coverFile`. Tempo and time signatures are configured via `tempoMapEnabled`, `tempoAddBtn`, `tempoMapList`, `timeSigMapEnabled`, `timeSigAddBtn`, and `timeSigMapList`. Time mode is selected with `timeMode` (Seconds/Beats), start offsets with `startOffsetSec`/`startOffsetBeats`, and explicit duration with `durationSec`.

### Spawn Calc Settings

Spawn timing uses `spawnToggle` and the fields under `spawnSettings`. Leads are `circleLead`/`circleLeadBeats` and `sliderLead`/`sliderLeadBeats`. Grid growth is defined by `gridTarget`, `gridGrow`, and `gridGrowUnit`, while target mode uses `gridTargetMode`. Bullet size is `bulletSize` and circle target radius is `circleTargetRadius`.

### Audio/Waveform

Audio is loaded through `audioFile`, played or paused with `playBtn`/`pauseBtn`, and history is tracked with `undoBtn`/`redoBtn`. The timeline display uses `curTime`, `durTime`, and `curBeat`. Snap subdivisions are chosen in `subdiv`, and when custom, the value is read from `subdivCustomWrap`/`subdivCustom`. The waveform add mode is set by `waveNoteType`, playback speed by `playbackRate`, snapping by `snap`, beat seek sync by `syncBeatSeek`, and zoom/fit by `zoomInBtn`, `zoomOutBtn`, and `fitBtn`. Amplitude scale uses `ampScale`, the scrollbar is `waveScroll`, and the actual rendering/playback elements are `wave` and `audio`.

### Preview/Mini Panels

The preview canvas is `view` and the grid UI is `grid`. The mini notes list is split into `miniNotes`, `miniList`, and sort control `miniSort`, with mini playback controlled by `miniPlayBtn` and `miniPauseBtn`. The mini waveform renders to `waveMini`.

### Note Ordering/Filtering/Multi Edit

Ordering and reindexing lives in `orderToolbar` with buttons for creation/time(spawn)/time(sec)/time(beats) plus `reindexNotes`. Filtering uses `noteFilterText` and `noteFilterType`. Multi edit uses `multiEditBar` with `multiSelCount`, `multiDeltaSec`, `multiDeltaBeat`, `multiApply`, `multiDelete`, `multiSelectAll`, and `multiClear`. The table view is `notesTable`, with row click to edit and drag to reorder.

### Import/Export

Exports are `downloadJSON()` for JSON and `exportSongZip()` for a song folder ZIP. Imports use `importFile`, and full reset uses `clearAll()`.

## Export Output (song.json)

`exportSongZip()` produces `charts/Normal.chart.json` and `song.json`. The `song.json` payload includes `title`, `artist`, `bpm`, `tempoMapEnabled`, `tempoMap`, `timeSigMapEnabled`, `timeSigMap`, `previewStart`, `previewDuration`, `coverImage`, `bgVideo`, `fmodEvent`, `durationSec`, `difficulties` (name/path/level), and `tags`.

## In-Game Schema (LevelChartData)

Chart-level fields include `title`, `bpm`, `timemode`, `startOffset`, `startOffsetBeats`, `tempoMapEnabled` with `tempoMap` (each entry uses `beat`, `bpm`), `timeSigMapEnabled` with `timeSigMap` (each entry uses `beat`, `numerator`, `denominator`), and `notes`. Every note shares `type`, `judgeTime`, `judgeBeat`, `startTime`, `startBeat`, `duration`, `durationBeats`, `timeSec`, `fadeSec`, `isTrail`, and `isLong`. Grid/Trail/Long add `cell` and reuse the timing fields; CircleTap uses `absPos` plus `judgeTime`/`judgeBeat`. Slider uses `absPath`, `sliderCurved`, `startTime`/`startBeat`, and `duration`/`durationBeats`. Bullet uses `bulletSpawnPos`, `bulletDockPos`, `bulletDespawnPos`, `bulletSpawnTime`/`bulletSpawnBeat`, `bulletToDockDuration`/`bulletToDockBeats`, `bulletDockToDespawnDuration`/`bulletDockToDespawnBeats`, and `bulletCurved`. Camera uses `camAffectHeight`, `camAffectAngle`, `camAffectCenter`, `camHeightPx`, `camAngleDegZ`, `camCenterPx`, `camUseBeats`, `camUseUIScaling`, and `camEase`. Line(Style) uses `timeSec`, `timeBeat`, `target` (grid-all/grid-line), `index`, `color`, `fadeSec`, and `fadeBeat`. Voice uses `fmodEvent`, `volume`, `pitch`, `fadeInSec`, `fadeOutSec`, `fadeInBeat`, `fadeOutBeat`, and `voiceEventPath` as a fallback. Image uses `imagePath`, `imageSizePx`, `imageScale`, `imageAnchor`, `imageRotationDeg`, `imageOpacity`, `imageFadeInSec`, `imageFadeOutSec`, `imageFadeInBeat`, `imageFadeOutBeat`, `imageZIndex`, and `imageBlend`.

## JSON Schema (Editor/Import)

The top-level is RootStr (string schema) or RootNum (numeric schema) with `schemaVersion`, `title`, `bpm`, `timeMode` (string: `Beats`/`Seconds`, number: 0/1), `startOffset`, `startOffsetBeats`, `tempoMapEnabled`, `tempoMap` (each entry uses `beat`, `bpm`), `timeSigMapEnabled`, `timeSigMap` (each entry uses `beat`, `numerator`, `denominator`), and `notes`.

NoteStr (string type) includes `type`, `cell`, `absPos`, `absPath`, `sliderCurved`, `isTrail`, `isLong`, timing fields `judgeTime`, `judgeBeat`, `startTime`, `startBeat`, `timeSec`, `duration`, `durationBeats`, `durationSec`, `fadeSec`, bullet fields `bulletSpawnPos`, `bulletDockPos`, `bulletDespawnPos`, `bulletSpawnTime`, `bulletSpawnBeat`, `bulletToDockDuration`, `bulletToDockBeats`, `bulletDockToDespawnDuration`, `bulletDockToDespawnBeats`, `bulletCurved`, camera fields `camAffectHeight`, `camAffectAngle`, `camAffectCenter`, `camHeightPx`, `camAngleDegZ`, `camCenterPx`, `camUseBeats`, `camUseUIScaling`, `camEase`, line fields `timeBeat`, `target`, `index`, `fadeBeat`, and the color variants `color`, `colorRGBA`, `colorArray`, `colorVec4`, `colorR`, `colorG`, `colorB`, `colorA`. Voice uses `fmodEvent`, `volume`, `pitch`, `fadeInSec`, `fadeInBeat`, `fadeOutSec`, `fadeOutBeat`, and Image uses `imagePath`, `sizePx`, `scale`, `anchor`, `rotationDeg`, `opacity`, `zIndex`, `blend`, `key`, `rotDeg`, `alpha`.

NoteNum (numeric type) includes `type` (0~8), `cell`, `absPos`, `absPath`, `sliderCurved`, `isTrail`, `isLong`, timing fields `judgeTime`, `judgeBeat`, `startTime`, `startBeat`, `timeSec`, `duration`, `durationBeats`, `durationSec`, `fadeSec`, bullet fields `bulletSpawnPos`, `bulletDockPos`, `bulletDespawnPos`, `bulletSpawnTime`, `bulletSpawnBeat`, `bulletToDockDuration`, `bulletToDockBeats`, `bulletDockToDespawnDuration`, `bulletDockToDespawnBeats`, `bulletCurved`, camera fields `camAffectHeight`, `camAffectAngle`, `camAffectCenter`, `camHeightPx`, `camAngleDegZ`, `camCenterPx`, `camUseBeats`, `camUseUIScaling`, `camEase`, line fields `timeBeat`, `target`, `index`, `fadeBeat`, `color`, `colorRGBA`, `colorArray`, `colorVec4`, `colorR`, `colorG`, `colorB`, `colorA`, voice fields `fmodEvent`, `volume`, `pitch`, `fadeInSec`, `fadeInBeat`, `fadeOutSec`, `fadeOutBeat`, and image fields `imagePath`, `sizePx`, `scale`, `anchor`, `rotationDeg`, `opacity`, `zIndex`, `blend`.

Parser notes: the string `type` value `Style` maps to Line, and `hold`/`longnote` map to Long. Both `timeMode` and `type` accept string or numeric values.
