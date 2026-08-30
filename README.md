# ansible
My simple Ansible playbooks

## users
Creates, updates, and deletes Linux user accounts from per-user directories.

### Layout
Each user gets a directory next to `deploy.yml`. The directory name is only a
label; the account name comes from `username` inside `env.yaml`.

```
users/
├── ansible.cfg
├── deploy.yml
├── inventory
├── requirements.yml
├── tasks/
│   └── manage_user.yml
└── exampleuser/
    ├── env.yaml
    └── id_rsa.pub
```

Any directory containing an `env.yaml` is discovered automatically, so adding a
user means adding a directory. Discovery runs on the control node and ignores an
`env.yaml` sitting directly in the playbook directory.

### User definition
`env.yaml` requires `username`. Everything else is optional:

```yaml
---
username: exampleuser
passwd: testpassword

# Optional:
# state: absent          # delete instead of create
# remove_home: true      # only used with state: absent
# shell: /bin/zsh
# home: /home/exampleuser
# groups: [wheel]        # supplementary groups, appended
```

Every `*.pub` file in the directory is added to the user's
`~/.ssh/authorized_keys`. Existing keys added outside this repo are left alone
(`exclusive: false`).

### Setup
```bash
ansible-galaxy collection install -r requirements.yml
```

Put target hosts in `inventory`.

### Usage
```bash
ansible-playbook deploy.yml                          # all discovered users
ansible-playbook deploy.yml -e target_users=alice    # one user
ansible-playbook deploy.yml -e target_users=alice,bob
ansible-playbook deploy.yml --check --diff           # dry run
```

`target_users` accepts a comma-separated string as well as a real list, because
`-e key=value` extra vars always arrive as strings.

### Deleting users
Deletion is driven by state, not by removing the directory, so a definition and
its keys can stay in git after the account is gone.

```bash
# Delete, keep the home directory
ansible-playbook deploy.yml -e target_users=alice -e user_state=absent

# Delete, and wipe the home directory
ansible-playbook deploy.yml -e target_users=alice -e user_state=absent \
                            -e remove_home=true
```

State resolves in this order, first match wins:

1. `-e user_state=absent` — run-wide override
2. `state: absent` in the user's `env.yaml` — per-user intent, kept in git
3. `default_user_state` in `deploy.yml` — `present`

Home directories are **preserved by default**, since deletion is
unrecoverable. The playbook prints a reminder when it leaves one behind.
Because `userdel` refuses to remove an account that still owns live processes,
`pkill -KILL -u <user>` runs first; its exit code 1 ("nothing to kill") is
treated as success.

### Privilege escalation
`ansible.cfg` sets `become_ask_pass = True`, so runs prompt once for
`BECOME password`. If the sudo user is passwordless, that prompt does not
appear. Add `-k` when the SSH login itself needs a password instead of a key.

### Passwords
`passwd` is stored in clear text in `env.yaml` and hashed at run time with
`password_hash('sha512')`, because Linux needs a crypt hash in `/etc/shadow`.
The salt is derived from the username so repeated runs stay idempotent.
`update_password: on_create` means the password is only written when the account
is created — later manual password changes on the host are not reverted.

Clear-text credentials in git are not safe for anything real. Use
`ansible-vault` before putting actual passwords here.

### Verified behavior
Checked against a live host (`ansible-core 2.21.3`, `ansible.posix 2.2.2`):

- Default run predicts three changes for a new user: create primary group,
  create account, write `authorized_keys`.
- `-e user_state=absent` skips the whole create/update block, and vice versa.
- Deleting an account that does not exist is a clean no-op and reports
  `nothing to remove`.
- Existence detection relies on `getent`, which returns the passwd entry for an
  existing user and `null` for a missing one.

Two limits of `--check` on the delete path:

- The `pkill` task is a `command`, so it is always skipped in check mode; its
  real behavior only runs live.
- Check mode cannot predict `userdel` failures caused by processes or open files
  appearing between the check and the real run.
