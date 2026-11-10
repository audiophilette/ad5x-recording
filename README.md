# Automatically record Flashforge AD5X prints with Home Assistant, SSH, systemd, and FFmpeg

This is the setup I used to automatically record video from a **Flashforge AD5X** whenever a print begins, and stop recording when the print completes.

The recording happens on a separate Linux server (Fedora in my case), not on Home Assistant OS. Home Assistant watches the printer's status through the [Flashforge Home Assistant integration](https://github.com/GhostTypes/ff-5mp-hass), and runs remote `systemctl` commands over SSH. FFmpeg reads the camera stream from **Home Assistant's camera proxy**, which also avoids competing with the Home Assistant camera view for the printer's direct MJPEG connection.

Home Assistant monitors the printer and starts or stops the Linux service over SSH. FFmpeg pulls video through the authenticated camera proxy and writes timestamped MKV files.

## Prerequisites

- A Flashforge AD5X integrated with Home Assistant, including a machine-status sensor and camera entity.
- A Linux host with `ffmpeg`, `systemd`, SSH server, and a writable recording directory.
- An SSH identity Home Assistant can use to run **only** the relevant service start/stop commands on that host.
- A Home Assistant **long-lived access token** for reading the camera proxy stream.
- Home Assistant's `shell_command` integration and YAML automations/packages.

Replace each `/REPLACE_WITH_…_DIR` placeholder with a full directory path on the appropriate host. The command examples assume paths without spaces. Use the same replacement wherever a placeholder appears. The systemd unit and sudoers examples use the standard Linux locations shown below.

| Directory placeholder | Purpose |
| --- | --- |
| `/REPLACE_WITH_RECORDINGS_DIR` | MKV output directory on the recording host |
| `/REPLACE_WITH_RECORDER_SCRIPT_DIR` | Directory containing the recorder script on the recording host |
| `/REPLACE_WITH_RECORDER_ENV_DIR` | Directory containing the token environment file on the recording host |
| `/REPLACE_WITH_HA_CONFIG_DIR` | Home Assistant configuration directory, typically `/config` on HAOS |
| `/REPLACE_WITH_HA_SSH_DIR` | Directory for the SSH key and `known_hosts` that Home Assistant Core can read |

Adjust these example settings for your installation:

| Setting | Example |
| --- | --- |
| Recorder host | `recorder_host.example.com` |
| SSH service user | `ha-ad5x` |
| Recorder service | `ad5x-record.service` |
| Camera entity | `camera.dining_room_flashforge_printer_camera` |
| Machine-status entity | `sensor.dining_room_flashforge_printer_machine_status` |
| Recording directory | `/REPLACE_WITH_RECORDINGS_DIR` |
| HA base URL | `http://HOME_ASSISTANT:8123` (placeholder) |


## 1. Create the recorder script on the Linux host

Copy [the recorder script](examples/ad5x-record) to `/REPLACE_WITH_RECORDER_SCRIPT_DIR/ad5x-record` and set its recordings directory and camera URL.

Replace `HOME_ASSISTANT` with your Home Assistant hostname or address. If you use HTTPS, update the URL accordingly. Ensure the recording directory exists and is writable by the service's Unix user (in this example, `cjg`):

```bash
sudo install -d -o cjg -g cjg /REPLACE_WITH_RECORDINGS_DIR
sudo chmod 755 /REPLACE_WITH_RECORDER_SCRIPT_DIR/ad5x-record
```

The `-use_wallclock_as_timestamps 1`, `-fflags +genpts`, and `-fps_mode vfr` options are intentional: the AD5X camera stream doesn't necessarily deliver frames at a perfectly constant rate. MKV is a useful container for long-running recordings that might be interrupted unexpectedly. `-nostdin` keeps FFmpeg from trying to read interactive commands from systemd.

### Home Assistant camera token

Create a long-lived access token in Home Assistant under your user profile. On the **Linux recording host**, store it in `/REPLACE_WITH_RECORDER_ENV_DIR/ad5x-record.env`:

```ini
HA_TOKEN=REPLACE_WITH_YOUR_LONG_LIVED_ACCESS_TOKEN
```

Protect the file:

```bash
sudo chown root:root /REPLACE_WITH_RECORDER_ENV_DIR/ad5x-record.env
sudo chmod 600 /REPLACE_WITH_RECORDER_ENV_DIR/ad5x-record.env
```

The service uses `EnvironmentFile=` to load this secret. Keep the actual file out of Git and out of public Gists. The camera proxy URL is authenticated; this token must be authorized to access the camera.

You can check the proxy separately (without sharing the token) using `curl` from the recording host:

```bash
curl -I \
  -H "Authorization: Bearer $HA_TOKEN" \
  'http://HOME_ASSISTANT:8123/api/camera_proxy_stream/camera.dining_room_flashforge_printer_camera'
```

**Note:** For a streaming endpoint, a GET test is more reliable than `HEAD` if the latter is unsupported. A successful GET should return a multipart MJPEG stream (`multipart/x-mixed-replace`).

## 2. Create the systemd service

Copy [the systemd unit](examples/ad5x-record.service) to `/etc/systemd/system/ad5x-record.service`. Set the paths for the recorder script and token file. The example runs as `cjg`.

**Do not add an `[Install]` section or enable this service at boot.** Home Assistant controls when it starts. `KillSignal=SIGINT` tells FFmpeg to shut down cleanly and finalize its output when the service is stopped.

In this setup, FFmpeg logged `Exiting normally, received signal 2` but returned exit status `255`, causing systemd to mark an otherwise successful stop as `failed`. `SuccessExitStatus=255` makes systemd treat that code as success. **Tradeoff:** a genuine FFmpeg error returning `255` will also be treated as success; inspect the journal when troubleshooting.

Load the unit:

```bash
sudo systemctl daemon-reload
sudo systemctl start ad5x-record.service
systemctl is-active ad5x-record.service
sudo systemctl stop ad5x-record.service
systemctl show ad5x-record.service \
  -p ActiveState -p SubState -p Result -p ExecMainStatus
```

After a clean stop, expect `ActiveState=inactive` and `Result=success`. Check the resulting MKV in your recordings directory. **Remember `daemon-reload` after editing the unit!**

## 3. Give Home Assistant restricted SSH access

Generate an SSH key pair in a location Home Assistant Core can read, using `/REPLACE_WITH_HA_SSH_DIR/ad5x_recorder` and `/REPLACE_WITH_HA_SSH_DIR/ad5x_recorder.pub`. The exact key-generation command depends on where you run it; for example, from an HAOS terminal add-on:

```bash
mkdir -p /REPLACE_WITH_HA_SSH_DIR
ssh-keygen -t ed25519 -f /REPLACE_WITH_HA_SSH_DIR/ad5x_recorder -N ''
chmod 600 /REPLACE_WITH_HA_SSH_DIR/ad5x_recorder
```

Install the **public** key for the remote service account (`ha-ad5x` in my setup). I used **FreeIPA** for the account, SSH public key, and access controls, but a local Linux account with an `authorized_keys` entry works too.

Configure passwordless sudo narrowly, allowing only:

```text
/usr/bin/systemctl start ad5x-record.service
/usr/bin/systemctl stop ad5x-record.service
```

For a conventional `/etc/sudoers.d/` setup, an equivalent rule is:

```sudoers
ha-ad5x ALL=(root) NOPASSWD: /usr/bin/systemctl start ad5x-record.service, /usr/bin/systemctl stop ad5x-record.service
```

Validate sudoers changes with `visudo`. **Do not grant blanket passwordless sudo.**

### FreeIPA-specific notes

In my FreeIPA deployment, I created:

- A service user named `ha-ad5x` with its SSH public key registered in IPA.
- A sudo rule allowing that user on `recorder_host.example.com` to run only the two `systemctl` commands, with authentication disabled for those commands.
- An HBAC rule permitting the service user to access **both** `sshd` and `sudo` on that host.

The second HBAC service is easy to miss: SSH may work while `sudo -n` fails with `PAM account management error: Permission denied`. Check both:

```bash
ipa hbactest --user=ha-ad5x --host=recorder host.example.com --service=sshd
ipa hbactest --user=ha-ad5x --host=recorder host.example.com --service=sudo
```

SSSD caching may delay policy changes; I needed `sudo sss_cache -E` on the host after adjusting the HBAC rule.

For host-key verification, put the **verified** SSH host key in `/REPLACE_WITH_HA_SSH_DIR/known_hosts`. If you initially connected from the HAOS Terminal & SSH add-on, its `/root/.ssh/known_hosts` may be separate from the files visible to Home Assistant Core. Do not turn off strict host-key checking to work around that.

Test from the HA environment where the key is accessible:

```bash
ssh -i /REPLACE_WITH_HA_SSH_DIR/ad5x_recorder \
  -o BatchMode=yes \
  -o StrictHostKeyChecking=yes \
  -o UserKnownHostsFile=/REPLACE_WITH_HA_SSH_DIR/known_hosts \
  ha-ad5x@recorder host.example.com \
  'sudo -n /usr/bin/systemctl start ad5x-record.service'
```

The SSH client inside a terminal add-on is **not necessarily the same execution environment** as Home Assistant Core. The final proof is a successful Home Assistant `shell_command` action.

## 4. Configure Home Assistant

In `/REPLACE_WITH_HA_CONFIG_DIR/configuration.yaml`, set `packages` under the existing `homeassistant` key to `!include_dir_named packages/`. I already had this configured.

Copy [the Home Assistant package](examples/ad5x.yaml) to `/REPLACE_WITH_HA_CONFIG_DIR/packages/ad5x.yaml`. Set the SSH directory, remote account and host, and machine-status entity ID for your installation.

**Important: raw state values are case-sensitive.** The Home Assistant UI may display `Printing` and `Completed`, while the actual states used in automations are `printing` and `completed`. In my case, the [Spook integration](https://spook.boo/) flagged the incorrect uppercase `Printing` trigger. Check **Developer Tools → States** for your own entity's raw values rather than trusting the friendly display text.

I deliberately trigger on the **machine-status** sensor instead of the binary `printing` sensor, because a temporary pause should not automatically end a recording.

### Load or reload the configuration

Validate the YAML under **Developer Tools → YAML**. When adding the package or shell commands for the first time, do a **full Home Assistant restart**. For subsequent changes to the `automation:` section alone, **Reload Automations** or run `automation.reload` from **Developer Tools → Actions**.

Changes to `shell_command:` generally need a Home Assistant restart.

## 5. Test it

In **Developer Tools → Actions**, run `shell_command.ad5x_record_start`.

On the recorder host:

```bash
systemctl is-active ad5x-record.service
ls -lht /REPLACE_WITH_RECORDINGS_DIR/ | head
```

Then run `shell_command.ad5x_record_stop` in **Developer Tools → Actions**.

Verify the service becomes inactive and the MKV is playable. FFmpeg can take a moment to finalize the file after the stop action.

To exercise the **start automation’s actions** during an existing print, run `automation.trigger` with `automation.ad5x_start_recording` as the target.

Check the actual automation entity ID in your instance. Manually triggering the automation bypasses its state trigger; it tests the action, **not** whether the state-change trigger works. For a true end-to-end test, start a short print and confirm the service starts automatically and stops after the sensor transitions to `completed`.

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| No AD5X automations appear | Verify `packages:` is enabled, check YAML, and fully restart Home Assistant after first adding the package. |
| Automation never fires | Check its trace and the **raw** machine-status state (`printing`, not display text `Printing`). |
| SSH action returns `0` but prints home-directory warnings | A service account with home `/dev/null` may print `Could not chdir ...` or `.bashrc` warnings. The command can still succeed; choose a valid home directory if desired. |
| SSH key accepted but connection closes | Check FreeIPA HBAC permission for the `sshd` service. |
| SSH works but sudo fails with PAM permission error | Check HBAC permission for the `sudo` service, the sudo rule, and SSSD cache. |
| HA terminal SSH works but `shell_command` fails | Check SSH executable, private-key access, and `known_hosts` from **Home Assistant Core**, not just the terminal add-on. |
| Recording stops but service says `failed` | Check journal for `Exiting normally, received signal 2`, FFmpeg exit code, `SuccessExitStatus=255`, and whether `systemctl daemon-reload` was run. |
| Camera view fails while recording | Use the authenticated HA camera proxy instead of opening another direct MJPEG connection to the printer. |
| Recording does not begin after HA restarts mid-print | The basic state-change automation does not perform startup reconciliation; see limitations below. |

Useful host-side diagnostics:

```bash
journalctl -u ad5x-record.service -n 100 --no-pager
systemctl show ad5x-record.service \
  -p ActiveState -p SubState -p Result -p ExecMainStatus
```

## Limitations and next steps

This is the **working basic setup**, not a complete print-job lifecycle manager:

- It starts on `printing` and stops on `completed`; it does **not yet** handle every cancellation/failure state, because those raw values weren't verified.
- It does not automatically reconcile state after a Home Assistant restart in the middle of a print.
- If the recorder host reboots or FFmpeg fails mid-print, it does not automatically restart recording until another matching trigger occurs.
- Each invocation creates a timestamped MKV, but filenames aren't yet tied to the print filename or job ID.
- A conventional speed-up timelapse can look jerky. A future improvement is a **one-frame-per-layer** timelapse using `sensor.dining_room_flashforge_printer_current_layer`. That sensor sometimes reports brief backward jumps, so a robust implementation should track the highest observed layer per print rather than capture on every change.

Despite those limitations, this setup automatically captures the full print video without requiring the printer's cloud service or a separate streaming relay.
