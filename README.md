# groovymame-build

Build automatico di **GroovyMAME** (drop-in per Batocera / RGS_CRT, x64) su GitHub Actions.

## Cosa fa
- Controlla ogni giorno il repo ufficiale GroovyMAME (`antonioginer/GroovyMAME`): quando esce un
  nuovo tag `gm0*sr*`, lo builda e pubblica il binario `mame` come GitHub Release. Se quel tag è
  già stato buildato (Release esistente), il build viene saltato.
- Output: `mame` (~91 MB UPX, x64, dynamic), copiabile in `rgs15/binaries/mame` di RGS_CRT.

## Trigger
- **Cron giornaliero** → controllo automatico dei nuovi tag `gm0*sr*`: compila e pubblica la
  Release solo al primo rilascio di un tag nuovo (se già buildato, salta).
- **Manuale** (`workflow_dispatch`, input `groovy_tag`, default `gm0288sr222d`).

## Come lo buildiamo
Build su Ubuntu 24.04, GroovyMAME `gm0288sr222d` (MAME 0.288 + SwitchRes 2.22d).

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

## Patch
9 patch ufficiali Batocera (set completo **meno `001`/`004`/`007`**):

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
| `013` | fix-qt-buildoptions |

## Download
- **Release** (stabile): https://github.com/frezeen/groovymame-build/releases → `mame`.
  Repo **public**: chiunque può scaricare.
- **Artefatto per-run** (debug): Actions → run → sezione *Artifacts*.

## Deploy su RGS_CRT
1. Scarica `mame` dalla Release.
2. Copia in `rgs15/binaries/mame` (mai scrivere su `/usr/bin/mame/mame`).
3. `./crt-install.sh`.
4. Test: `./mame -showconfig`; poi un rom reale → `display.log` deve segnare il modo video nativo.
