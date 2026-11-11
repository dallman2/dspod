# macdsp — Architecture (draft 2)

`macdsp` is a command-line toolbox that lets an AI agent (or a person) turn a Mac
into a simple loudspeaker processor using CamillaDSP. An agent should be able
to install it, run one init command, and then put together whatever routing and
crossover the user asks for, as long as the hardware supports it.

It is meant to replace a hardware speaker processor (e.g. a DriveRack PA2) for
the common case where that is overkill.

---

## 0. Decisions so far

| Topic | Decision |
| :--- | :--- |
| Target use | Real speakers. Not critical listening, not high SPL (for now). |
| Clock sync | **CamillaDSP rate adjust + async resampler** (BlackHole is *not* put inside an aggregate). |
| Built-in headphone jack | **Excluded by default.** It only exists while something is plugged in (Apple Silicon), which breaks aggregates. Opt in with `--allow-builtin`. |
| Natural-language `prompt` command | **Removed.** The calling agent turns the user's words into structure itself. macdsp contains no LLM. |
| BlackHole | Not installed by macdsp. `doctor` detects it and returns a "needs human" result with instructions. |
| Microphone permission | Assumed to be needed. Detected, explained, and checked end to end. |
| CamillaDSP binary | Downloaded by `macdsp init` at a **pinned version with a checksum**. The pin is bumped deliberately over time. |
| Aggregate devices | Created and removed by macdsp through the **CoreAudio API**, not Audio MIDI Setup. |
| System output | Controlled by the CLI (`macdsp system-output set …`), with a remembered "previous" device to go back to. |
| Calibration | Pragmatic: distance-based delay, mic-assisted level and polarity checks, all optional. |
| Shape of the tool | **Toolbox of small composable commands**, plus one high-level golden-path command (`crossover`). |

---

## 1. Golden use case

> "I have a subwoofer and a pair of stereo speakers. Each takes its own
> line-level input. I want to run cables from my computer to both and set a
> crossover frequency."

What the agent should be able to do, start to finish:

```bash
macdsp init --json                       # fetch pinned CamillaDSP, create state dir
macdsp doctor --json                     # BlackHole? mic permission? engine? -> tells agent what's missing
macdsp devices list --json               # see real output devices and channel counts
macdsp outputs assign mains --device "<Interface>" --out 1,2
macdsp outputs assign sub   --device "<Interface>" --out 3
macdsp system-output set blackhole --json
macdsp engine start --json
macdsp crossover set --freq 80 --json    # 2.1 crossover, safe defaults, starts at low volume
macdsp volume set -20 --json
# later
macdsp crossover set --freq 95 --json    # just move the crossover point
macdsp crossover set --sub-gain -3 --sub-polarity inverted --json
```

If the mains and the sub are on **different** devices, the agent runs one
extra step before assigning outputs:

```bash
macdsp aggregate create --name "macdsp-out" \
  --device "<Interface A>" --device "<Interface B>" --clock "<Interface A>" --json
```

---

## 2. Non-goals (for now)

- Room correction / automatic EQ from measurements.
- High-SPL protection beyond basic limiting and gain caps.
- Controlling hardware DSPs (DriveRack etc.).
- A GUI, or an LLM inside the tool.
- Linux / Windows.

---

## 3. Principles

1. **Toolbox, not wizard.** Each command does one thing and can be combined
   with the others. `crossover` is just a convenience that writes a graph built
   from the same pieces an agent could put together by hand with `graph apply`.
2. **Deterministic.** Exact names only (no fuzzy matching). The same input
   always gives the same compiled config. Filters are named by **role**
   (`sub_lowpass`), not by value (`sub_lp_110hz`), so changing a frequency only
   changes parameters and CamillaDSP can update it in place without restarting.
3. **Fail closed.** If something is unclear, missing or unhealthy, nothing new
   is applied, and the last known-good config stays (or is restored).
4. **Machine-readable everything.** `--json` on every command, `schema` for
   every input model, documented exit codes, and an explicit "a human needs to
   do X" result for steps macOS won't let software do.
5. **Undoable.** `uninstall` and `system-output restore` always leave the Mac
   producing sound through its original output.

---

## 4. Signal path

```
macOS apps
   │  (system output = BlackHole 2ch, set by `macdsp system-output set blackhole`)
   ▼
BlackHole 2ch  ── virtual device, host-clocked
   │  CoreAudio capture, 2 ch
   ▼
CamillaDSP (launchd user agent, pinned binary)
   ├─ async resampler + enable_rate_adjust  ← absorbs the BlackHole-vs-hardware clock difference
   ├─ mixer: stereo → N outputs (sub = mono sum L+R by default)
   ├─ per-output filter chain: crossover → calibration (gain/delay/polarity) → limiter
   └─ main volume (with ramp)
   │  CoreAudio playback
   ▼
Output device
   ├─ preferred: ONE interface with ≥3 outputs  (no aggregate, single clock)
   └─ otherwise: macdsp-created aggregate (clock = one sub-device,
                 drift correction on the others)
```

### Why rate adjust instead of putting BlackHole in the aggregate

- No feedback path: BlackHole's outputs are never part of the playback device.
  The compiler also refuses any playback device that contains BlackHole.
- The sync is visible. `status` reports CamillaDSP's rate-adjust value and
  buffer level, so an agent can tell whether sync is healthy.
- Costs a little CPU and about a millisecond of latency. Acceptable here.

The output aggregate (if one is used) still relies on macOS drift correction
between its hardware sub-devices. That can't be avoided with two separate
pieces of hardware.

### Compiled `devices` section (shape)

```yaml
devices:
  samplerate: 48000
  chunksize: 1024            # generous default; latency isn't critical for music
  enable_rate_adjust: true
  resampler:
    type: AsyncSinc
    profile: Balanced
  capture:
    type: CoreAudio
    device: "BlackHole 2ch"
    channels: 2
  playback:
    type: CoreAudio
    device: "<interface or macdsp aggregate>"
    channels: <device output count>
```

(Exact keys must be checked against the pinned CamillaDSP version. The config
schema has changed between major versions.)

---

## 5. Command surface

All commands accept `--json`. Commands that change anything accept `--dry-run`.

### Setup and health
| Command | Purpose |
| :--- | :--- |
| `macdsp init` | Create the state dir. Download the pinned CamillaDSP for this CPU, verify its SHA-256, install it to `bin/`. Write the launchd agent plist (not loaded yet). Safe to re-run. `--upgrade-engine` moves to the package's current pin. |
| `macdsp doctor` | Run every check (§9) and report each as `ok` / `fail` / `needs_human`, with a fix for each. |
| `macdsp uninstall` | Stop and unload the engine, restore the previous system output, remove macdsp-created aggregates, remove the plist. `--purge` also deletes the state dir. |

### Devices and routing targets
| Command | Purpose |
| :--- | :--- |
| `macdsp devices list` | CoreAudio devices: name, UID, input/output channel counts, supported rates, whether it is built-in or virtual, whether it is alive. |
| `macdsp aggregate create / list / remove` | Make or remove a **public** (persistent) aggregate through CoreAudio, with an explicit sub-device order, clock device and per-device drift correction. Only touches aggregates macdsp created (tagged by UID prefix). |
| `macdsp outputs assign <name> --device <exact name or UID> --out <1-based list>` | Give physical outputs a name (`mains`, `sub`, `left_top` …). Stored in the hardware profile. |
| `macdsp outputs list / remove` | Show or remove assignments. |

Channel numbers in the CLI and JSON are **1-based** to match the labels on
interface panels. The compiler converts them to CamillaDSP's 0-based indices.

### System audio
| Command | Purpose |
| :--- | :--- |
| `macdsp system-output get` | Current default output (and default system-sound output). |
| `macdsp system-output set <device or "blackhole">` | Change the default output. The first change saves the previous device. |
| `macdsp system-output restore` | Go back to the saved previous device. |

### Engine
| Command | Purpose |
| :--- | :--- |
| `macdsp engine start / stop / restart` | Load or unload the launchd agent. On start, the engine comes up **muted** with the last-good config. |
| `macdsp status` | Engine state, stop reason, sample rate, rate-adjust value, buffer level, peak/RMS levels per channel, active graph summary, volume and mute. |

### Processing
| Command | Purpose |
| :--- | :--- |
| `macdsp crossover set [--freq] [--slope] [--mains-highpass on/off] [--sub-gain] [--sub-delay-ms] [--sub-polarity] [--subsonic-hz] [--mains-gain] …` | Golden path: builds a 2.1 graph from the `mains` and `sub` assignments. Settings left out keep their current values, so `--freq 95` changes only the frequency. |
| `macdsp crossover show` | Current crossover settings. |
| `macdsp graph apply --file/--stdin` | Low-level: apply any `GraphSpec` (§6). This is how an agent builds anything other than 2.1 (3-way, stereo subs, delay-only, EQ-only …). |
| `macdsp graph show / diff --file` | Show the active graph, or what a candidate would change. |
| `macdsp schema <model>` | JSON Schema for `GraphSpec`, `CrossoverArgs`, `HardwareProfile`, and the response envelope. |

### Level and safety
| Command | Purpose |
| :--- | :--- |
| `macdsp volume set <dB> / get` | CamillaDSP main volume (ramped). |
| `macdsp mute / unmute [<output name>]` | Mute everything or one output. |
| `macdsp panic` | Mute **only** (never bypass to full range). If the engine can't be reached: kill the engine process and `system-output restore`. |

### Calibration (pragmatic, optional — §8)
| Command | Purpose |
| :--- | :--- |
| `macdsp calibrate delay --distance <output>=<meters> …` | Turn speaker distances into per-output delay. |
| `macdsp calibrate level [--mic <device>]` | Play band-limited noise per output and suggest gain trims. |
| `macdsp calibrate polarity [--mic <device>]` | Compare sub normal vs inverted at the crossover and suggest the louder one. |
| `macdsp test-tone <output> [--type pink/sine] [--freq] [--level]` | Hearing check for one output (always band-limited to what that output is allowed to play). |

---

## 6. Data model

There are three separate layers, stored as separate files, so that changing
one never quietly changes another.

1. **HardwareProfile** — what is plugged in: playback device UID, named outputs → physical channels, `allow_builtin`.
2. **GraphSpec** — the processing: what each named output receives and which filters it gets.
3. **Calibration** — per named output: trim (dB), delay (ms), polarity. Applied after the graph, so changing the crossover never resets calibration.

```python
class Source(BaseModel):
    input: Literal["L", "R"]
    gain_db: float = Field(0.0, ge=-60, le=0)

class Filter(BaseModel):            # discriminated union in practice
    kind: Literal["highpass", "lowpass", "peaking", "lowshelf", "highshelf", "gain", "delay"]
    freq_hz: float | None = Field(None, ge=10, le=20000)
    topology: Literal["LinkwitzRiley", "Butterworth"] | None = None
    order: Literal[2, 4, 6, 8] | None = None   # LR must be even
    q: float | None = None
    gain_db: float | None = None
    delay_ms: float | None = Field(None, ge=0, le=50)

class OutputGraph(BaseModel):
    output: str                     # must exist in HardwareProfile.outputs
    sources: list[Source]           # e.g. sub = [L -6 dB, R -6 dB]
    filters: list[Filter] = []

class GraphSpec(BaseModel):
    outputs: list[OutputGraph]
    # cross-output validation:
    #  - every output name exists and maps to a distinct physical channel
    #  - each output appears only once
    #  - the playback device does not contain BlackHole
    #  - net gain input→output ≤ 0 dB unless allow_boost
```

`crossover set` is a pure function `CrossoverArgs → GraphSpec`, so an agent
can see exactly what it produced (`--dry-run --json` returns the GraphSpec and
the compiled CamillaDSP YAML).

### Crossover defaults

- LR4 (24 dB/oct) at the requested frequency.
- Sub: mono sum (L+R, each −6 dB), lowpass at `freq`, subsonic highpass
  Butterworth 2nd-order at 20 Hz (configurable, can be turned off).
- Mains: highpass at `freq` (`--mains-highpass on`, the default). `off` leaves
  the mains full range and the sub only adds low end. This is a common
  pragmatic setup when the mains are decent bookshelf or PA speakers.
- Every output: a limiter as the last filter (default −1 dBFS).

---

## 7. Apply lifecycle

Every change (`crossover set`, `graph apply`, `calibrate … --apply`, `outputs assign`) goes through the same steps:

1. **Lock** (`state/lock`, so two agents can't apply at once).
2. **Validate** the model (Pydantic + cross-output rules) → exit 3 on failure.
3. **Resolve hardware**: the device is present and alive, and the channels exist → exit 4.
4. **Compile** to a CamillaDSP config. Filter names are by role.
5. **Offline check** with the pinned binary: `camilladsp --check <tmpfile>`. Works with no engine running. `--dry-run` stops here and returns the compiled config plus a diff.
6. **Classify the change**: parameters only, pipeline/structure change, or `devices` change (stream restart). Reported as `"reload": "params" | "pipeline" | "restart"`.
7. **Apply** over the WebSocket (`config.set_active`).
8. **Health check**: poll for up to ~3 s. Healthy means the engine is running (or paused only because of silence), with no stop reason and no capture/playback errors.
9. Healthy → write `last_good.yml` and return exit 0. Not healthy → restore `last_good.yml`, check again, return **exit 6** with the engine's stop reason.

---

## 8. Calibration (pragmatic)

This is for alignment that's good enough, not for measurement-grade work. Every
step also works by ear without a mic.

- **Delay** — `calibrate delay --distance mains=3.0 --distance sub=2.2`
  computes `Δd / 343 m/s` and delays whichever output is closer. It also adds a
  known fixed latency difference between sub-devices when an aggregate is used,
  if one has been measured (below).
- **Level** — plays in-band pink noise on one output at a time (others muted
  through a temporary calibration graph). With `--mic`, macdsp records the mic,
  compares band RMS, and suggests trims so the sub matches the mains in the
  overlap region. Without a mic it returns the steps for the user to judge by
  ear plus the command to apply a trim.
- **Polarity** — plays band-limited noise around the crossover on sub + mains,
  once normal and once with the sub inverted. The louder one wins (mic RMS, or
  the user's ear).
- **Inter-device latency (experimental, mic only)** — impulse on each output,
  cross-correlate the arrivals, suggest delay. Only worth doing when an
  aggregate spans two interfaces.

Suggestions are returned as data (`"suggested": {...}`) and applied only with
`--apply`, so the agent stays in control.

Test signals are played by macdsp into BlackHole through the normal path, so
calibration exercises the real chain. The mic is recorded by the macdsp process
itself (which will also need mic permission for its host app).

---

## 9. `doctor` checks

| id | Check | On failure |
| :--- | :--- | :--- |
| `engine_binary` | Pinned CamillaDSP present, checksum matches, runs `--version` | `fail` → run `macdsp init` |
| `blackhole` | A "BlackHole 2ch" device exists | `needs_human` → install instructions and link |
| `mic_permission` | End-to-end: play a short tone into BlackHole, confirm CamillaDSP's capture peak is above silence | `needs_human` → which app/binary to allow in System Settings › Privacy & Security › Microphone |
| `launchd_agent` | Plist installed and loadable | `fail` → `macdsp init` |
| `engine_reachable` | WebSocket responds on 127.0.0.1:1234 | `fail` → `macdsp engine start` |
| `system_output` | Default output is BlackHole (only when a graph is active) | `fail` → `macdsp system-output set blackhole` |
| `playback_device` | Device in the hardware profile is present and alive, with enough channels | `fail` / `needs_human` (plug it in) |
| `aggregate_health` | macdsp aggregates have every sub-device present | `fail` → recreate, or `needs_human` |
| `rate_sync` | Rate adjust within normal range, no buffer under/overruns recently | `fail` → report details |
| `builtin_policy` | No output assigned to built-in hardware unless `allow_builtin` | `fail` |

---

## 10. Safety rails

- Engine **starts muted**, and the first apply of a new graph ramps up from a
  low volume (default −30 dB). The agent raises the volume on purpose.
- **Limiter** at the end of every output chain.
- **Net gain ≤ 0 dB** from input to any output unless `--allow-boost` (still capped, e.g. +6 dB).
- **Subsonic highpass** on sub outputs by default.
- **`panic` mutes; it never bypasses** to full range.
- **No fuzzy matching** of device or output names. Ambiguity is an error that lists the candidates.
- **Built-in output excluded** unless `--allow-builtin`.
- **Playback device may never contain BlackHole** (feedback-loop guard).
- Unassigned physical channels get no mixer mapping, so they are silent.

---

## 11. JSON envelope and exit codes

```json
{
  "ok": false,
  "command": "crossover set",
  "dry_run": false,
  "result": { },
  "reload": "params",
  "engine": { "state": "RUNNING", "stop_reason": null, "samplerate": 48000, "rate_adjust": 1.00002 },
  "warnings": [],
  "errors": [ { "code": "output_unknown", "message": "No output named 'subwoofer'", "candidates": ["sub", "mains"] } ],
  "human_action": null
}
```

When `human_action` is set, it looks like
`{"kind": "install_blackhole" | "grant_mic_permission" | "connect_device", "instructions": "...", "url": "..."}`
so an agent can pass it straight to the user.

| Code | Meaning |
| :--- | :--- |
| 0 | Success, applied and healthy |
| 1 | Internal error (bug) |
| 2 | CLI usage error (reserved by Typer/Click; no JSON envelope) |
| 3 | Validation failed (`errors[]`) |
| 4 | Hardware missing, unknown or unhealthy |
| 5 | Engine not reachable |
| 6 | Applied but unhealthy → rolled back to last-good |
| 7 | Needs a human (`human_action`) |
| 8 | Locked by another macdsp process |

The models are the single source of truth. The envelope schema is produced by
`macdsp schema envelope`, and the docs examples are generated from it.

---

## 12. Packaging and on-disk layout

- Python package, installed with `uv tool install macdsp` or `pipx install macdsp`.
- CoreAudio access (devices, aggregates, default output, device rates) through
  **PyObjC** (`pyobjc-framework-CoreAudio`) or `ctypes`, so no compiler or
  Xcode is needed on the user's machine.
- Engine pin shipped inside the package (`engine_pin.toml`: version, per-arch URL, SHA-256).

```
~/Library/Application Support/macdsp/
  bin/camilladsp              # pinned engine
  hardware.yml                # HardwareProfile
  graph.yml                   # GraphSpec (or crossover.yml + derived graph)
  calibration.yml
  active.yml / last_good.yml  # compiled CamillaDSP configs
  engine_state.yml            # CamillaDSP statefile (volume/mute/config path)
  previous_output.json        # for system-output restore
  lock
~/Library/LaunchAgents/com.macdsp.engine.plist
~/Library/Logs/macdsp/engine.log
```

Source layout:

```
src/macdsp/
  cli/              # Typer apps per command group, envelope + exit codes
  models/           # HardwareProfile, GraphSpec, CrossoverArgs, Calibration, Envelope
  coreaudio/        # device list, aggregate create/remove, default output, rates
  engine/           # binary install/pin, launchd, websocket client, apply lifecycle
  compile/          # GraphSpec + profile + calibration -> CamillaDSP config
  presets/          # crossover (2.1) -> GraphSpec; future: 3-way, stereo subs
  calibrate/        # signals, mic capture, analysis
  doctor/
```

---

## 13. Risks to prototype first

These could change the design, so test them before building the rest:

1. **Mic permission for a launchd-run engine.** macOS decides permission by
   "responsible process". A background CamillaDSP may never show a prompt and
   simply capture silence. We need a reliable way to get it allowed (and the
   end-to-end `mic_permission` check to detect it).
2. **Creating a public aggregate through PyObjC**: getting a stable channel
   order, and what happens when a sub-device is unplugged and plugged back in.
3. **Long-running rate adjust** with BlackHole capture. Run for hours and watch
   for drift, xruns and CPU use.
4. **Reload behaviour**: confirm that changing only parameters (role-named
   filters) updates without clicks or a stream restart on the pinned version.
5. **Volume keys**: with BlackHole as system output, the Mac's volume keys may
   not do anything useful. Decide whether `macdsp volume` is enough.

---

## 14. Open questions

1. **Hardware for the golden case.** 2.1 needs 3 outputs (L, R, sub). The
   AudioBox USB 96 has 2. What will feed the sub? Either a ≥3-output
   interface (preferred: one clock, no aggregate) or a second device in an
   aggregate. If the second device would be the built-in jack, that conflicts
   with the "exclude built-in" rule.
2. **Mains highpass default**: on (true crossover) or off (sub just adds low end)?
3. Does the sub have its own lowpass knob or an LFE/bypass input? Docs should
   tell users to set it to max/bypass so the two lowpasses don't stack.
4. Is **Python + PyObjC** acceptable, or should the CoreAudio parts be a small
   prebuilt Swift helper?
5. Is losing the Mac volume keys acceptable for v1?
