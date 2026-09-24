# groovymame-build

Build automatici di **GroovyMAME** (drop-in per Batocera / RGS_CRT, x64) e **libretro-mame**
(core RetroArch per arcade su Batocera) su GitHub Actions.

## Cosa fa
Due workflow indipendenti, ciascuno col suo cron e la sua Release:

### GroovyMAME — `build-mame.yml`
- Controlla ogni giorno il repo ufficiale (`antonioginer/GroovyMAME`): quando esce un
  nuovo tag `gm0*sr*`, lo builda e pubblica il binario `mame` come GitHub Release. Se quel
  tag è già stato buildato (Release esistente), il build viene saltato.
- Output: `mame` (~91 MB UPX, x64, dynamic), copiabile in `rgs15/binaries/mame` di RGS_CRT.

### libretro-mame — `build-core.yml`
- Controlla ogni giorno il repo ufficiale (`libretro/mame`): quando esce un nuovo tag
  `lrmame0*`, lo builda e pubblica il core `mame_libretro.so` come GitHub Release. Se quel
  tag è già stato buildato, il build viene saltato.
- Output: `mame_libretro.so` (~120 MB UPX, x64), copiabile in `rgs15/binaries/` di RGS_CRT.

## Trigger
- **Cron giornaliero** → auto-detect dell'ultimo tag disponibile (gm0*sr* / lrmame0*):
  builda e pubblica la Release solo se non esiste già una release per quel tag.
- **Manuale** (`workflow_dispatch`): input `groovy_tag` / `libretro_tag` (vuoto = ultimo
  tag trovato) + input `force` (rebuild anche se la release esiste già).

---

## GroovyMAME

Build su Ubuntu 24.04. Tag deciso dinamicamente (ultimo `gm0*sr*` disponibile,
oppure input manuale). Ultimo buildato con successo: `gm0288sr222d`
(MAME 0.288 + SwitchRes 2.22d); per 0.289+ serve la fix delle patch (vedi sotto).

| Opzione | Valore |
|---|---|
| Target | `mame` (subtarget `mame`, full) |
| Arch | `PTR64=1` (x64) |
| Ottimizzazione | `OPTIMIZE=2` |
| Precompilati (PCH) | `PRECOMPILE=1` |
| `REGENIE=1` | forza rigenerazione genie (applica CXXFLAGS/LDOPTS) |
| Strip | `SYMBOLS=0 STRIP_SYMBOLS=1` + `strip mame` |
| DCE (gc-sections) | `CXXFLAGS="-ffunction-sections -fdata-sections"` |
| Link | `LDOPTS="-Wl,--gc-sections -Wl,-z,pack-relative-relocs"` |
| Compressione | `upx mame` (~200-250 MB → ~91 MB) |
| Audio | `NO_USE_PULSEAUDIO=1`, PortAudio attivo (`-sound part`) |
| SDL | `USE_SDL=1` |
| Lib di sistema | zlib, jpeg, sqlite3, rapidjson, expat, glm, zstd |
| FLAC | bundle (nel binario) |
| Tool | `TOOLS=0` |
| ccache | `OVERRIDE_CC/CXX='ccache gcc/g++'`, cache persistita tra run |

Il binario è **dynamic** (linka le lib di sistema: libGL, SDL2, X11… presenti su Batocera).
Le patch vengono applicate prima del build (`git apply` → fallback `patch -p1 --fuzz=3`).

### Patch
8 patch ufficiali Batocera obbligatorie (set completo **meno `001`/`004`/`007`**):

| Patch | Scopo |
|---|---|
| `002` | no-nag screens |
| `005` | lightgun udev driver |
| `006` | fix-sliver |
| `008` | default-gun-config |
| `009` | offscreenreload autoconfig |
| `010` | fix-gun-aiming (jpark / opwolf3) |
| `011` | fix-compilation (obbligatoria per 0.288) |
| `012` | fix-largefile64 |

In più **`013` (fix-qt-buildoptions)**, in `patches/mame-obsolete/`: applicata in
best effort, se non attacca viene saltata. Non attacca più da **0.289** perché
upstream ha spostato il blocco Qt in `scripts/src/osd/modules.lua` racchiuso in
`if _OPTIONS["USE_QTDEBUG"]=="1"` — bug già fixato a monte.

---

## libretro-mame

Build su Ubuntu 24.04, libretro/mame tag `lrmame0289` (MAME 0.289).

| Opzione | Valore |
|---|---|
| Target | `mame` (subtarget `mame`, core libretro, x64) |
| Arch | `PTR64=1 LIBRETRO_CPU=x86_64 PLATFORM=x86` |
| Config | `CONFIG=libretro LIBRETRO_OS=unix RETRO=1 OSD=retro` |
| Ottimizzazione | `OPTIMIZE=2` |
| Precompilati (PCH) | `PRECOMPILE=1` |
| `REGENIE=1` | forza rigenerazione genie (applica CXXFLAGS/LDOPTS) |
| Strip | `strip mame_libretro.so` |
| DCE (gc-sections) | `CXXFLAGS="-ffunction-sections -fdata-sections"` |
| Link | `LDOPTS="-Wl,--gc-sections -Wl,-z,pack-relative-relocs"` |
| Compressione | `upx mame_libretro.so` (~300-400 MB → ~120 MB) |
| Lib di sistema | zlib, jpeg, sqlite3, expat, zstd |
| ccache | `OVERRIDE_CC/CXX='ccache gcc/g++'`, cache persistita tra run |

Il core è **dynamic** (linka le lib di sistema: libGL, SDL2, X11… presenti su Batocera).
Nessuna patch applicata — il tag `lrmame0*` di libretro/mame è già pronto per libretro.

---

## Download
- **Release** (stabile): https://github.com/frezeen/groovymame-build/releases → `mame`
  (GroovyMAME) e `mame_libretro.so` (libretro-mame). Repo **public**: chiunque può scaricare.
- **Artefatto per-run** (debug): Actions → run → sezione *Artifacts*.

## Deploy su RGS_CRT
### GroovyMAME
1. Scarica `mame` dalla Release.
2. Copia in `rgs15/binaries/mame` (mai scrivere su `/usr/bin/mame/mame`).
3. `./crt-install.sh`.
4. Test: `./mame -showconfig`; poi un rom reale → `display.log` deve segnare il modo video nativo.

### libretro-mame
1. Scarica `mame_libretro.so` dalla Release.
2. Sostituisci in `/usr/lib/libretro/mame_libretro.so` (attenzione: overlay Batocera).
3. Test: `retroarch --libretro /usr/lib/libretro/mame_libretro.so /path/rom.zip`.
