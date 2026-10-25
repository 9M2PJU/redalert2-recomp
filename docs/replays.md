# Replays and observer (design)

> Status: **design / not yet implemented.** This records the plan and the
> engine map so the build can start from a known footing. Nothing in the tree
> does any of this yet.

## Why it is feasible

RA2/YR is a deterministic lockstep simulation. Every player and AI order
becomes an `EventClass` stamped with the frame it executes on; given the same
map, the same player setup and the same random seed, replaying that stream of
events reproduces the game exactly, frame for frame. The engine already proves
this every network game: it runs the same events on every machine and compares
a per-frame CRC to catch desyncs. A replay is just that event stream written to
a file, plus the setup needed to start an identical game, played back by
feeding the events in instead of reading live input.

The recomp makes the hook points reachable from C: guest memory is at the
original addresses (`GUEST_PTR`), and `run_lift.py` can inject C at any guest
instruction (the HD-voxel patches do exactly this).

## The engine map (Yuri's Revenge, gamemd.exe)

Addresses from YRpp (the Ares/Phobos headers). Red Alert 2 (game.exe) has the
same structures at different addresses, found by shape the way `run_lift.py`
already does for modern controls and queue-five -- see *Open questions*.

| What | Address | Notes |
|------|---------|-------|
| `CurrentFrame` | `0x00A8ED84` | int, the game frame counter |
| initial seed | `0x00A8ED94` | DWORD, the game's starting random seed |
| `ScenarioClass::Instance` | `0x00A8B230` | `Randomizer Random` at +218 |
| `EventClass::OutList` | `0x00A802C8` | local outgoing events, `QueueClass<EventClass,128>` |
| `EventClass::DoList` | `0x008B41F8` | events to execute, `QueueClass<EventClass,128*128>` |
| `EventClass::MegaMissionList` | `0x00A83ED0` | queued movement/attack orders |
| EventClass ctors | `0x004C65A0`+ | one per event shape (Target, MegaMission, Production, ...) |
| `LatestFramesCRC[256]` | `0x00B04474` | the engine's per-frame sync CRCs |
| `CurrentFrameCRC` | `0x00AC51FC` | this frame's CRC |

`EventClass` is a fixed 111-byte struct, `#pragma pack(1)`:

```
offset  size  field
0       1     Type         (EventType, unsigned char)
1       1     IsExecuted
2       1     HouseIndex   (-1 = not a valid house)
3       4     Frame        the frame this event executes on
7       104   DataBuffer   union payload (MegaMission, Production, ...)
```

Every event carries its own execute-`Frame` and `HouseIndex`, so the recorded
stream needs no extra framing per event -- it is a list of these structs.

## Architecture

**Record.** Hook the point where events enter `DoList` (or execute from it) --
one choke point that sees local, AI and network events alike. Append each
`EventClass` (111 bytes) to the replay file. This is deterministic input, already
frame-stamped, so no timestamps or de-duplication are needed. Capture the setup
once at game start: map file (or its id + the full INI the game composed),
player slots (house, country, colour, team, AI difficulty), game options, and
the seed at `0x00A8ED94`.

**Replay.** Start a game from the recorded setup and seed, suppress live local
input (do not let the player's orders reach `OutList`), and at the top of each
frame inject the recorded events whose `Frame` equals `CurrentFrame` into
`DoList`. The sim re-executes them identically.

**Verify.** After each frame, compare `CurrentFrameCRC` against the CRC the
recording saw for that frame (record a sparse sample of `LatestFramesCRC`).
A mismatch is a determinism bug, caught immediately rather than as a visibly
wrong replay -- the same signal the engine's own desync detector uses.

**Observer** is a separate layer on top of a working replay: free camera,
reveal the whole map (shroud off), and switch which player's POV/sidebar is
shown. None of it touches the sim, so it can also drive a live MP spectator
later.

## File format (sketch)

```
magic "RA2RPL\0" + version
setup block: target (gamemd/game), map id + composed INI, seed,
             player slots, game options
event stream: repeated 111-byte EventClass records (frame-stamped)
crc samples:  (frame, CurrentFrameCRC) every N frames, for verification
```

## Increments

1. **Recorder + determinism proof.** Pin the DoList execute/add instruction,
   hook it, dump the event stream + setup + CRC samples for a skirmish. Verify
   two recordings of the same scripted skirmish produce the same event stream
   and CRCs. (Skirmish is single-process and all events are local, so it is the
   cleanest first target.)
2. **Playback.** Inject recorded events, suppress live input, play a skirmish
   back and confirm per-frame CRCs match the recording to the last frame.
3. **Observer camera.** Free scroll, reveal-all, POV switch over a replay.
4. **Red Alert 2 target.** Shape-match the gamemd addresses in game.exe.
5. **MP spectator.** Reuse the observer layer on a live network game.

## Open questions

- **Exact hook instruction.** `0x0064CC68` sits in the MegaMission region of a
  large, closure-duplicated function; the generic per-event execution point
  still needs one focused disassembly pass (or hook the EventClass ctors +
  `OutList` for a skirmish-only MVP). `run_lift.py` applies a patch to every
  duplicated copy of an address, so multi-occurrence is not a problem.
- **RA2 (game.exe) addresses** must be found by shape, as the existing
  per-target patch tables do.
- **Map identity.** Standard maps can be referenced by name; custom/random maps
  need the composed scenario stored in the replay so playback loads the same
  one.
- **Version skew.** A replay is only valid against the same game build and the
  same mod; the setup block records enough to refuse a mismatch.
