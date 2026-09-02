# ESPHome — voice satellite builder

Compiles and flashes the ESPHome devices. Today that is one device: a
**reSpeaker XVF3800 with an onboard XIAO ESP32-S3**, acting as a Home Assistant
voice satellite with the wake word **"Patrick"**.

## Why this stack exists at all

Every guide for this board says *Settings → Add-ons → ESPHome Device Builder*.
**This HA install has no add-on store** — it is the plain
`ghcr.io/home-assistant/home-assistant:stable` container, not HAOS or
Supervised. So the builder runs as its own container here instead.

The *ESPHome integration* that actually talks to the finished device is built
into HA core and needs none of this. The builder is only a compiler; once the
device is flashed and adopted, this container can be stopped without affecting
the satellite.

UI: **https://esphome.tedraykov.me**, or `http://192.168.1.100:6052` directly.

**Private, not public** — the name resolves to `192.168.1.100` for LAN clients
via AdGuard's `*.tedraykov.me` wildcard rewrite, and the route is a file-provider
entry in `traefik/dynamic/homelab.yml` (host networking means Traefik's docker
provider cannot see the container, so labels are not an option). No Cloudflare
record was needed: the zone holds one wildcard `A` already. No DMZ firewall hole
exists for port 6052, so it is not reachable from the internet.

**It must stay that way unless auth is verified working**, because the builder
ships with **no authentication at all**. `ESPHOME_USERNAME` / `ESPHOME_PASSWORD`
are set as Portainer stack variables — never committed, this repo is public —
and the compose uses a `:?` guard so a missing value fails the deploy instead of
quietly starting an open dashboard. Credentials are at
`/opt/stacks/esphome/.admin-credentials` on VM 100, mode 600.

`ESPHOME_TRUSTED_DOMAINS` pins the websocket Origin/Host allowlist that the
whole UI rides on. `normalize_host()` strips the port, so
`esphome.tedraykov.me,192.168.1.100` covers both the proxied name and direct
`:6052` access. Leaving it unset disables the check entirely.

## The voice pipeline — ElevenLabs, not Whisper/Piper

**No STT or TTS container is needed.** The official HA ElevenLabs integration
does *both* speech-to-text and text-to-speech, and it is already configured in
HA. That removes `wyoming-whisper` and `wyoming-piper` from this design
entirely — nothing local, no GPU allocation, no extra RAM on a VM that has
about 3.7 GiB free.

Wake word is the piece that stays local: `micro_wake_word` runs **on the
ESP32-S3 itself**, so no `openWakeWord` container either.

The trade-off to be aware of: every utterance now leaves the house and hits
ElevenLabs, and each one costs quota. If that becomes a problem, the swap is a
local `faster-whisper` on the 3060 for STT while keeping ElevenLabs for TTS —
STT is the expensive half by volume.

## Files

| File | Goes where |
|---|---|
| `docker-compose.yml` | the Portainer stack |
| `respeaker-patrick.yaml` | **copy to `/opt/stacks/esphome/config/`** |
| `secrets.yaml.example` | copy to `/opt/stacks/esphome/config/secrets.yaml`, fill in |

The device YAML is *not* deployed by the stack. Portainer clones this repo to
its own directory; the builder reads `/opt/stacks/esphome/config`. Those are
different places, so the YAML is copied there by hand.

## ⚠ The wake word has to be trained first

**There is no pre-built "Patrick" model.** The stock micro_wake_word catalogue
is `okay_nabu`, `hey_jarvis`, `hey_mycroft` and `stop` — that is the whole list.
A build referencing `models/patrick.json` fails until the file exists.

To produce it:

1. Check <https://microwakeword.com/library> first, in case someone has already
   trained and published one.
2. Otherwise train it at <https://microwakeword.com/train> (browser, synthetic
   samples, minutes) or with the Colab notebook in
   [`alfiedennen/microwakeword-trainer`](https://github.com/alfiedennen/microwakeword-trainer).
3. It emits `patrick.json` + `patrick.tflite`. Put **both** in
   `/opt/stacks/esphome/config/models/`.

Note "Patrick" is a two-syllable, plosive-initial word — short wake words draw
more false accepts than three-syllable ones like "okay nabu". Expect to tune
`probability_cutoff` upward if it fires at the television.

**Meanwhile the satellite ships with the package's own wake words** —
`okay_nabu`, `kenobi`, `hey_jarvis`, `hey_mycroft`. Use **"hey jarvis"**. The
`micro_wake_word` block at the bottom of `respeaker-patrick.yaml` is commented
out; uncomment it once `patrick.json` exists and push it over OTA — no USB, no
re-adoption in HA.

**Do not bridge the gap by pointing that block at a stock model.** The package
already defines `hey_jarvis` under its own id, so a second entry loading the
same model as `patrick` runs the detector twice: wasted PSRAM, and both fire on
one utterance, which surfaces as `duplicate_wake_up_detected`.

The device keeps the name "Patrick" throughout. The wake word is separate from
the device identity, and renaming after adoption would churn every HA entity
id.

## First flash

1. Copy `respeaker-patrick.yaml` and a filled-in `secrets.yaml` into
   `/opt/stacks/esphome/config/`.
2. In the ESPHome UI, open the device → **Install** → **Manual download**.
3. Flash the resulting `.bin` from <https://web.esphome.io> in Chrome or Edge,
   with the USB-C cable in the **XIAO's own port** — not the array's power port.
   The data lines only run through the XIAO.
4. Everything after that is OTA.
5. The XVF3800's own firmware is flashed by the ESP32 over I2C on first boot.
   Leave it alone; the first boot after a firmware change takes longer.

Then in HA: Settings → Devices & Services → the ESPHome device is discovered,
Configure, and assign it a pipeline whose STT and TTS are both ElevenLabs.

## Gotchas

- **Do not enable GPIO9 MCLK.** The package deliberately leaves it off; the
  XVF3800 is the I2S master and generates its own clock.
- I2S input and output share GPIO7/GPIO8 with `allow_other_uses: true`. That is
  correct, not a mistake.
- The stock ESPHome `i2s_audio` component does not work with this board — the
  package pulls a patched fork. Do not "fix" it back to upstream.
- ESP-IDF builds are heavy. VM 100 has ~3.7 GiB free; a first build will take a
  while and may need the container to be the only thing working hard.
