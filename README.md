# YubiKey 5C login and sudo on macOS

Use a **YubiKey 5C** instead of typing a long password on macOS:

| Where | How you authenticate | Mechanism |
|---|---|---|
| Login window, lock screen, Keychain | Insert key and type a short **PIN** | PIV smart card pairing (built into macOS) |
| `sudo` in Terminal | **Touch** the key (or type the PIN) | `pam_u2f` (FIDO2) plus the built-in `pam_smartcard` |

Your normal password **keeps working as a fallback** everywhere. Nothing here can lock you out
unless you change the `sufficient` lines to `required`, which is not recommended.

Tested on macOS 27 (Apple Silicon, Homebrew in `/opt/homebrew`). On an Intel Mac, replace
`/opt/homebrew` with `/usr/local`.

---

## 0. Before you start

- Keep **one Terminal window with a root shell open** (`sudo -s`) while you change PAM files.
  If something goes wrong you can fix it from there.
- Ideally have a **second YubiKey** as a backup. Every step below has a "backup key" variant.
- Never put keys, certificates, `u2f_keys` or PINs in this repository. `.gitignore` blocks the
  common file types, but check `git status` before every commit anyway.

Install the tools:

```bash
brew install ykman pam-u2f
```

Check that the key is detected:

```bash
ykman info
```

---

## 1. Login, lock screen and Keychain with a PIN (PIV smart card)

macOS natively supports smart card login. The YubiKey's PIV applet acts as the smart card. After
pairing, you insert the key and type a 6 to 8 character PIN instead of your password.

### 1.1 Secure the PIV applet

Factory defaults are PIN `123456`, PUK `12345678` and a well-known management key. Change all three.

> If the key was used before and you do not know its PIV PIN, `ykman piv reset` wipes the PIV
> applet only (FIDO2 and OTP are untouched). This is irreversible for any PIV keys on it.

```bash
ykman piv access change-pin
```

```bash
ykman piv access change-puk
```

```bash
ykman piv access change-management-key --generate --protect
```

The last command generates a random management key and stores it on the YubiKey, protected by your
PIN, so you never need to type it. The PIN may be alphanumeric, 6 to 8 characters. **3 wrong PINs
block it**; unblock with `ykman piv access unblock-pin` and the PUK. 3 wrong PUKs mean a PIV reset.

### 1.2 Create the login (9a) and key-management (9d) identities

macOS needs a certificate in slot **9a** (authentication) and one in slot **9d** (used to unlock
the login Keychain). Public keys are written to a temporary folder, outside the repo:

```bash
ykman piv keys generate --algorithm ECCP256 9a "$TMPDIR/9a.pem"
```

```bash
ykman piv certificates generate --subject "CN=$USER login" 9a "$TMPDIR/9a.pem"
```

```bash
ykman piv keys generate --algorithm ECCP256 9d "$TMPDIR/9d.pem"
```

```bash
ykman piv certificates generate --subject "CN=$USER keychain" 9d "$TMPDIR/9d.pem"
```

Each command asks for the PIN, and the key may blink; touch it if it does.

```bash
rm "$TMPDIR/9a.pem" "$TMPDIR/9d.pem"
```

### 1.3 Pair the YubiKey with your macOS account

**Easiest:** unplug and re-insert the YubiKey. A **"SmartCard Pairing"** notification appears.
Click **Pair**, then enter your account password, the PIV PIN, and your Keychain password.

**Command line alternative:**

```bash
sc_auth identities
```

Copy the hash shown for the key (the line with `CN=<you> login`), then:

```bash
sudo sc_auth pair -u "$USER" -h PASTE_HASH_HERE
```

Confirm:

```bash
sc_auth list -u "$USER"
```

### 1.4 Use it

- **Login window / lock screen:** insert the YubiKey; the password field turns into a PIN field.
  Type the PIN and press Return.
- **sudo:** `/etc/pam.d/sudo` already contains `auth sufficient pam_smartcard.so`, so with the
  paired key inserted `sudo` asks for the **PIN** instead of the password. Step 2 turns this into a
  single touch.
- **FileVault:** after a cold boot or restart, the FileVault unlock screen still asks for your
  **account password** once. PIN login applies from then on (wake, lock, log out, fast user switch).

**Backup key:** repeat 1.1 to 1.3 with the second YubiKey. One account can have several paired
identities.

---

## 2. sudo with a touch (pam_u2f)

### 2.1 Register the YubiKey for your user

```bash
mkdir -p ~/.config/Yubico
```

```bash
pamu2fcfg -o pam://yubikey-mac -i pam://yubikey-mac > ~/.config/Yubico/u2f_keys
```

Touch the key when it blinks. If you set a FIDO2 PIN on the key, it asks for that first.

**Backup key** (appends to the same line; insert the second key first):

```bash
pamu2fcfg -n -o pam://yubikey-mac -i pam://yubikey-mac >> ~/.config/Yubico/u2f_keys
```

```bash
chmod 600 ~/.config/Yubico/u2f_keys
```

The `origin` / `appid` value `pam://yubikey-mac` must match the one in
[config/sudo_local](config/sudo_local). Using a fixed value (not the hostname) means renaming the
Mac does not break sudo.

### 2.2 Install the PAM config

The repository's [config/sudo_local](config/sudo_local) adds one line:

```text
auth  sufficient  /opt/homebrew/lib/pam/pam_u2f.so cue origin=pam://yubikey-mac appid=pam://yubikey-mac
```

`/etc/pam.d/sudo_local` is Apple's supported hook for local changes; it **survives macOS updates**,
unlike edits to `/etc/pam.d/sudo`. Back up the current file, then install:

```bash
mkdir -p backups && cp /etc/pam.d/sudo_local "backups/sudo_local.$(date +%Y%m%d-%H%M%S)"
```

```bash
sudo install -m 444 -o root -g wheel config/sudo_local /etc/pam.d/sudo_local
```

### 2.3 Test (in a NEW Terminal window, keep the root shell open)

```bash
sudo -k && sudo -v
```

You should see `Please touch the device.` Touch the key: done, no password.

Then test the fallbacks:

1. Remove the key and run `sudo -k && sudo -v`: it should ask for the PIN or password.
2. Insert the key, run it again and **don't** touch it: after the timeout it falls back too.

Order of attempts for `sudo`: YubiKey touch, then PIV PIN (if the paired key is inserted), then
your password.

---

## 3. Optional hardening

**Touch ID as well:** uncomment the `pam_tid.so` line in [config/sudo_local](config/sudo_local)
and reinstall it (2.2).

**Require a touch for PIV operations too** (slot 9a policy is set when the key is generated):
regenerate 9a with `--touch-policy cached` added to the `ykman piv keys generate` command, then
generate its certificate and pair again.

**Smart-card-only login** (`enforceSmartCard`) disables password login entirely. Do not enable it
unless you have a paired backup key and understand the recovery path; it is not part of this setup.

---

## 4. Undo everything

Remove touch-for-sudo:

```bash
sudo cp /etc/pam.d/sudo_local.template /etc/pam.d/sudo_local
```

```bash
rm ~/.config/Yubico/u2f_keys
```

Unpair the smart card (get the hash from `sc_auth list -u "$USER"`):

```bash
sudo sc_auth unpair -u "$USER" -h PASTE_HASH_HERE
```

---

## 5. Troubleshooting

| Symptom | Fix |
|---|---|
| sudo never asks for a touch | Check `ls /opt/homebrew/lib/pam/pam_u2f.so` exists and the path in `/etc/pam.d/sudo_local` matches. Add `debug` to the end of the pam_u2f line and rerun `sudo -v` to see why. |
| `Please touch the device` but the touch is rejected | `origin`/`appid` in `u2f_keys` and `sudo_local` differ. Re-run 2.1 with the same `pam://yubikey-mac` value. |
| sudo broken entirely | From the root shell you kept open: `cp /etc/pam.d/sudo_local.template /etc/pam.d/sudo_local`. Without one, boot to Recovery (hold power), open Terminal and restore the file on the system volume. |
| No pairing notification | Run `sc_auth identities`. If nothing is listed, 9a or 9d has no certificate; redo 1.2. Then pair with the CLI (1.3). |
| Login screen still shows the password field | Pairing missing (`sc_auth list -u "$USER"`) or the key was inserted after the field appeared; reinsert it. |
| PIN blocked | `ykman piv access unblock-pin` with the PUK. |
| `brew upgrade` broke sudo touch | The path `/opt/homebrew/lib/pam/pam_u2f.so` is a stable symlink, so this is rare; reinstall with `brew reinstall pam-u2f`. |

---

## Repository layout

```text
.
├── README.md          this guide
├── config/
│   └── sudo_local     template for /etc/pam.d/sudo_local
└── .gitignore         keeps keys, certificates, u2f_keys and backups out of git
```
