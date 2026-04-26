# Audio playback investigation (macOS / Linux OpenAL)

User-reported symptoms (macOS arm64, native build, fresh runtime dir, retail
ZH assets):

- Background music does not play.
- Some loop sounds (water, helicopter rotor) loop incorrectly — they
  re-trigger or stack instead of being a single sustained source.
- Voice lines (unit responses, EVA) are missing.

## What is *not* the bug

Two ODR-looking duplicates that turned out to be fine:

1. `GeneralsMD/Code/GameEngineDevice/Include/OpenALAudioManager.h` — looks
   like a stub but is actually a `#include "OpenALAudioDevice/OpenALAudioManager.h"`
   forwarder when `SAGE_USE_OPENAL` is defined. So all translation units end
   up with the real class layout from `Core/.../OpenALAudioDevice/`.
2. `GeneralsMD/Code/GameEngineDevice/Source/OpenALAudioManager.cpp` — is a
   stub with `addAudioEvent` returning `AHSV_Error` and music as no-op, but
   the `GeneralsMD/.../CMakeLists.txt` only adds it to the build when
   `SAGE_USE_OPENAL=OFF`. With `OPENAL=ON` (current macOS preset), the real
   implementation in `Core/.../OpenALAudioDevice/OpenALAudioManager.cpp`
   (~3.2 kloc) is the one linked.

Init also looks healthy in launch logs:

```
INFO: SDL3GameEngine::createAudioManager()
INFO: Creating OpenAL audio backend
[SUBSYS] initSubsystem('TheAudio') START
... (AudioSettings.ini, Default/Music.ini, Music.ini all load with filesRead=1)
[SUBSYS] initSubsystem('TheAudio') END
```

No `alc-error`, no FFmpeg complaints during init.

## Where to look next

### Music silent
- Music streaming in `Core/.../OpenALAudioDevice/OpenALAudioManager.cpp:777`
  uses `FFmpegFile::decodePacket()` to feed an `OpenALAudioStream`. Hypothesis
  is that the music file the engine asks for cannot be opened by FFmpeg
  (codec mismatch) and the failure is logged via `DEBUG_LOG`, which is a
  no-op in release builds — the music silently never plays.
- Verify by attaching a printf to the `Failed to open FFmpeg file` branch
  and checking if it fires. If it does, dump the resolved track filename and
  check whether it points into a `.big` or onto disk.
- Music tracks are referenced from `Music.ini` / `Default/Music.ini`; both
  load cleanly, so the issue is later when a specific track is requested.

### Loop sounds re-triggering
- `addAudioEvent` in the real backend should call `alSourcei(source, AL_LOOPING, AL_TRUE)`
  for sustained ambients. If looping is not flagged on the source, the
  sample plays once, finishes, and the dispatcher re-issues it the next
  game tick — producing the "stutter loop" symptom.
- Cross-check `playing->m_audioEventRTS->getAudioEventInfo()->m_isLooping`
  vs. the actual `AL_LOOPING` flag on the source.

### Voices missing
- Voice samples are streamed from `SpeechZH.big` / `SpeechEnglishZH.big`.
  Confirm the ArchiveFileSystem mounts both at startup. If voices are
  routed through a separate `playStream` path (FFmpeg-decoded), same root
  cause as music applies — the codec might fail silently.

## Suggested first patch

Add lightweight stderr logging to:
1. `OpenALAudioManager::addAudioEvent` — log every event name + audio type +
   whether the source was created, whether the file resolved on disk, and
   whether `alSourcePlay` was called.
2. `FFmpegFile::open` failure path — currently `DEBUG_LOG` only.
3. The music dispatcher (`nextMusicTrack` / `playMusic` analogues).

Then a single `./run.sh -fullscreen ... 2>&1 | tee audio.log` with the user
playing for ~30 s should reveal which path (sample, stream, music) is
breaking. From there we can write a real fix.

## Related upstream notes

`docs/BUILD/MACOS.md` lists `Audio (OpenAL) | In progress (Phase 2)` so the
backend is known-incomplete. Whatever we land here should slot into that
phase rather than introducing a parallel audio system.
