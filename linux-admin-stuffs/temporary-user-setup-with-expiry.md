# Lab: Temporary User Setup with Expiry

Create user `mark` on App Server 3 with an account expiry date of `2027-03-28`.

### Commands

```sh
sudo useradd -e 2027-03-28 mark
```

### Flag Reference
- `-e YYYY-MM-DD` — set account expiration date (ISO 8601 format required)

### Verification

```sh
grep mark /etc/passwd
sudo chage -l mark
```

### Outputs

```
[banner@stapp03 ~]$ sudo useradd -e 2027-03-28 mark
[banner@stapp03 ~]$ grep mark /etc/passwd
mark:x:1001:1001::/home/mark:/bin/bash
[banner@stapp03 ~]$ sudo chage -l mark
Last password change                                    : Jul 02, 2026
Password expires                                        : never
Password inactive                                       : never
Account expires                                         : Mar 28, 2027
Minimum number of days between password change          : 0
Maximum number of days between password change          : 99999
Number of days of warning before password expires       : 7
```

> **Note:** `chage -l mark` is the single best command for viewing all expiry details — account, password, and inactivity periods — all in one view.

### Also Useful

```sh
sudo usermod -e 2027-12-31 mark         # change expiry on existing user
sudo usermod -e "" mark                 # remove expiry (never expires)
sudo chage -E 2027-03-28 mark           # set expiry via chage instead
sudo usermod -L mark                    # lock account (manual disable)
sudo userdel -r mark                    # delete when no longer needed
```

### Errors
| Error | Why it happened | Fix |
| --- | --- | --- |
| `useradd: invalid date format '03-28-2027'` | Used MM-DD-YYYY or DD-MM-YYYY. `useradd` only accepts ISO 8601 (`YYYY-MM-DD`). | Re-run with `useradd -e 2027-03-28 mark` |
| `Account expires` shows `never` | Forgot `-e` flag at creation. The user exists but has no expiry. | `sudo usermod -e 2027-03-28 mark` — no need to delete and recreate |
| `useradd: user 'mark' already exists` | User already exists. `useradd` can't create duplicates. | Use `usermod -e` to add expiry instead |

### Quick Context
**Account expiry vs password expiry**: These are separate mechanisms. `-e` disables the account entirely (no SSH, console, or su). Password expiry (from `chage`) only forces a password change on next login — the user can still log in. Account expiry is a hard block; password expiry is a soft reminder. Use each for different purposes.

**Why ISO 8601 (`YYYY-MM-DD`)**: MM-DD-YYYY and DD-MM-YYYY are ambiguous between US and European conventions. `YYYY-MM-DD` is unambiguous, sorts correctly as text, and is locale-independent. Linux enforces this across all date-related flags to prevent international mistakes.

**`useradd -e` vs `chage -E` vs `usermod -e`**: All three write to the same field in `/etc/shadow`. `useradd -e` is creation-time. `usermod -e` and `chage -E` modify existing users. Use whichever you remember — they're interchangeable.

**What happens after expiry**: The account cannot log in via SSH, console, or `su`. But cron jobs owned by that user keep running, and existing processes continue. Files remain accessible by root. Account expiry is a reversible disable — not user deletion.
