# Lab: Bash Script — Media Archive Zip + Copy to Storage Server

Create `/scripts/media_archive.sh` on App Server 3 (stapp03) that zips `/var/www/html/media` into `xfusioncorp_media.zip`, saves it in `/archives/`, copies it to the Nautilus Storage Server's `/archives/` over passwordless SSH, is runnable by the server user (banner), and uses no sudo inside the script.

## How It Works

**The script is two sequenced jobs — zip then scp.** The zip must finish before scp runs, and that ordering only holds when you run the *script*. Running `scp` by hand before the zip exists fails with "No such file or directory" because there is nothing to copy yet.

**Passwordless copy = SSH key trust, not sudo.** The `scp` runs as the local user (`banner`) connecting to `natasha@ststor01`. It won't prompt once (a) banner has an SSH key, (b) `natasha`'s `authorized_keys` holds banner's public key, and (c) the storage-side `/archives` is owned by `natasha` so the simple destination `/archives/` is writable. This is why the only password you ever type is during setup (`ssh-copy-id`), never during the script run.

**Permissions are prepared ahead of time, not inside the script.** Banner can write the zip because `/archives` is `777`, and banner can execute the script only after `chmod +x`. Nothing in the script needs root, which is exactly the "no sudo inside" requirement.

### Commands

```sh
# get on App Server 3
ssh banner@stapp03 
# 1. set up passwordless SSH from stapp03 to the storage server
ls ~/.ssh/id_rsa.pub || ssh-keygen -t rsa -b 2048 -N "" -f ~/.ssh/id_rsa   # key if absent
ssh-keyscan ststor01 >> ~/.ssh/known_hosts                                  # host key, no prompt
ssh-copy-id -i ~/.ssh/id_rsa.pub natasha@ststor01                           # asks natasha password once (Bl@kW)

# 2. make the storage-side target exist and be writable by natasha
ssh natasha@ststor01 "sudo mkdir -p /archives && sudo chown natasha:natasha /archives"

# 3. write the script (sudo vi /scripts/media_archive.sh)
#!/bin/bash
zip -r /archives/xfusioncorp_media.zip /var/www/html/media
scp /archives/xfusioncorp_media.zip natasha@ststor01:/archives/

# 4. make it executable, then run it as banner (no password, no sudo inside)
sudo chmod +x /scripts/media_archive.sh
/scripts/media_archive.sh
```

### Flag Reference
* `-b 2048` — key size; `-N ""` — empty passphrase so the script's scp never prompts; `-f <path>` — where to write the key
* `-i <keyfile>` — explicitly use banner's public key during `ssh-copy-id`
* `-r` — recursive zip (include the directory and everything under it)
* `ssh-keyscan <host>` — fetch a host's SSH key and add it to `known_hosts` so the first scp won't ask to trust the host
* `chown natasha:natasha /archives` — transfer ownership so the storage-side scp destination is writable

### Verification

Confirm the archive exists on both the app server and the storage server, and that its contents are the media files.

```sh
ls -l /archives/xfusioncorp_media.zip                # on stapp03
ssh natasha@ststor01 "ls -l /archives/"              # on ststor01
unzip -l /archives/xfusioncorp_media.zip | head      # contents = media dir files
```

### Outputs

```
[banner@stapp03 ~]$ /scripts/media_archive.sh
  adding: var/www/html/media/ (stored 0%)
  adding: var/www/html/media/.gitkeep (stored 0%)
  adding: var/www/html/media/index.html (stored 0%)
xfusioncorp_media.zip                                             100%  595     1.1MB/s   00:00
[banner@stapp03 ~]$ ls -l /archives/xfusioncorp_media.zip
-rw-r--r-- 1 banner banner 595 Oct  6 09:43 /archives/xfusioncorp_media.zip
[banner@stapp03 ~]$ ssh natasha@ststor01 "ls -l /archives/"
total 4
-rw-r--r-- 1 natasha natasha 595 Oct  6 09:43 xfusioncorp_media.zip
```

> **Note:** Do not run `scp` by hand to "test" it — it depends on the zip that only the script creates. Run the whole `/scripts/media_archive.sh` and let it copy after zipping.

### Also Useful

```sh
ssh natasha@ststor01 "ls -ld /archives"                 # confirm storage-side dir + owner
unzip -t /archives/xfusioncorp_media.zip                # test archive integrity
rm -f /archives/xfusioncorp_media.zip && /scripts/media_archive.sh   # clean re-run, verify end-to-end
crontab -e   # 0 2 * * 0  /scripts/media_archive.sh     # weekly re-archive, if desired
```

### Errors
| Error | Why it happened | Fix |
| --- | --- | --- |
| `scp: stat local "/archives/xfusioncorp_media.zip": No such file or directory` | Ran `scp` before the script created the zip; the zip and scp are ordered steps inside the script | Run `/scripts/media_archive.sh` instead of scp by hand |
| `ls: cannot access '/archives/xfusioncorp_media.zip': No such file or directory` | Script had been made executable but not yet executed, so the archive didn't exist | Run `/scripts/media_archive.sh`, then re-check |
| `Permission denied, please try again` (during `ssh-copy-id`) | Typed natasha's password incorrectly in a couple of attempts before the correct one | Retry with the right password (`Bl@kW`); key is only added once the password authenticates |
| Script `scp` would prompt for a password | No SSH key pair or the public key wasn't in natasha's `authorized_keys` | `ssh-keygen` + `ssh-copy-id -i ~/.ssh/id_rsa.pub natasha@ststor01` |
| First `scp` would ask "yes/no" host authenticity | `ststor01` not in known_hosts on the fresh server | `ssh-keyscan ststor01 >> ~/.ssh/known_hosts` before running the script |
| `chmod +x` as banner fails on the script | Script created with `sudo vi` → root-owned; non-owner can't chmod | `sudo chmod +x /scripts/media_archive.sh` |
| scp to `/archives/` fails even with the key | Storage target not writable by natasha | `ssh natasha@ststor01 "sudo mkdir -p /archives && sudo chown natasha:natasha /archives"` |

### Quick Context
**"Passwordless scp" is an SSH key + pre-made target, not a script trick**: the script just runs `scp`; what stops the prompt is banner's public key sitting in natasha's `authorized_keys` and the destination being writable. Setup work (key + known_hosts + chown) is done once, outside the script.

**zip-then-scp is a two-step pipeline, not two independent commands**: because they live inside the script, running the script guarantees ordering. That's why a manual scp isn't a valid "test" — and why the exact script content you were asked for (zip on line 1, scp on line 2) is the whole implementation.

**No-sudo-inside = prep permissions beforehand**: every writable path (`/scripts` 777, `/archives` 777, storage `/archives` chown'd to natasha) is fixed during setup so the everyday run needs neither password nor root.
