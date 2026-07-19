# groovymame-build

Build automatico di **GroovyMAME** (drop-in per Batocera / RGS_CRT, x64) su GitHub Actions.

## Cosa fa
- Compila il GroovyMAME **ufficiale** (`antonioginer/GroovyMAME`) ogni giorno: appena esce un
  nuovo tag `gm0*sr*`, lo builda e pubblica il binario `mame` come GitHub Release.
- Output: `mame` (~100 MB, x64), copiabile in `rgs15/binaries/mame` di RGS_CRT.

## Trigger
- **Cron giornaliero** → build automatico della nuova versione di GroovyMAME.
- **Manuale** (`workflow_dispatch`, input `groovy_tag`, default `gm0288sr222d`).

## Come lo buildiamo
Build su Ubuntu 24.04, GroovyMAME `gm0288sr222d` (MAME 0.288 + SwitchRes 2.22d).

| Opzione | Valore |
|---|---|
| Target | `mame` (subtarget `mame`) |
| Arch | `PTR64=1` (x64) |
| Ottimizzazione | `OPTIMIZE=3` |
| Precompilati | `PRECOMPILE=1` |
| `REGENIE=1` | skip rigenerazione genie |
| Strip | `SYMBOLS=0 STRIP_SYMBOLS=1` + `strip mame` |
| Audio | `NO_USE_PULSEAUDIO=1`, PortAudio attivo (`-sound part`) |
| SDL | `USE_SDL=1` |
| Link | `LDOPTS="-lasound -lfontconfig"` |
| Lib di sistema | zlib, jpeg, sqlite3, rapidjson, expat, glm, zstd |
| FLAC | bundle (nel binario) |
| Tool | `TOOLS=0` |
| ccache | `OVERRIDE_CC/CXX='ccache gcc/g++'`, cache persistita tra run |

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
