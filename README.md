# FreeIPA user workspace

Manage accounts on an existing FreeIPA server with `users.yml` and the local
`ipa_users` role. It uses `freeipa.ansible_freeipa.ipauser`, the collection's
user-management module. Upstream roles install servers and clients; there is no
upstream user-management role. This play does not install or enroll machines.

## Setup

Use Ansible Core 2.14+ and FreeIPA 4.7+ (for password expiration). Install:

```sh
ansible-galaxy collection install -r collections/requirements.yml
```

Edit `inventory/all.yml`: `ipa_server` must contain your existing IPA server.
Tasks execute via SSH on its first host only, using its IPA Python libraries.
The current inventory targets `ipa.lab.shared` and holds a Vault-encrypted SSH
password. Ensure DNS/network access, SSH access for `local_admin` (or override
`ansible_user`), and privilege escalation work.

The existing `ansible.cfg` enables become and reads the Vault password from
`~/.vault/.vault`; override with `--vault-password-file PATH`. It also disables
SSH host-key checking; adjust for your environment.

The play defaults to principal `admin` and reuses `ansible_password` as
`ipaadmin_password`. Override directory credentials in
`inventory/group_vars/ipa_server.yml` or an encrypted extra-vars file if they
differ from SSH credentials. Use a principal authorized to manage users.
Client group variables do not apply to this play. Account tasks use `no_log`
to suppress credentials and account details.

## Regular accounts

Copy `vars/users.example.yml` to `vars/users.yml`, then edit `ipa_users`:

```yaml
ipa_users:
  - name: alex.smith
    first: Alex
    last: Smith
    email: [alex.smith@example.com]
    password: '{{ vault_alex_password }}'
    shell: /bin/bash
```

Store secrets with Ansible Vault:

```sh
ansible-vault create vars/secrets.yml
# Define vault_alex_password and optionally ipaadmin_password in that file.
ansible-playbook users.yml -e @vars/users.yml -e @vars/secrets.yml
```

New users require `name`, `first`, and `last`. Add other supported
[ipauser attributes](https://github.com/freeipa/ansible-freeipa/blob/v1.17.0/README-user.md)
as needed. Passwords are optional: without one the play does not assign a
password. Normal accounts follow the server's password and first-login rules.
The default account list is empty.

Repeated runs reconcile supplied attributes with `update_password: on_create`,
so existing passwords are retained. Removing a user from the list does not
delete it. The ten fictional usernames in `roles/ipa_users/vars/main.yml` are
reserved and cannot also appear in `ipa_users`.

## Fictional accounts

Creation defaults to off:

```sh
ansible-playbook users.yml -e ipa_create_test_users=on
ansible-playbook users.yml -e ipa_create_test_users=off
```

Alternatively set `ipa_create_test_users: true` or `false` in
`inventory/group_vars/ipa_server.yml`. Command-line on/off strings are converted
to booleans. Regular and fictional users may be managed in one run.

Enabling creates exactly ten accounts, from `test-avery-stone` through
`test-parker-moss`, with initial password **`password`**. This shared weak
password is for lab accounts. Existing passwords are not reset on reruns.
After creation a separate task sets expiration to `9999-12-31T23:59:59Z`,
avoiding immediate expiry of administrator-created passwords. This is effectively
non-expiring through a fixed far-future date, rather than an infinite lifetime.
No global password policy is changed. The expiration task repairs drift;
password changes outside this play may reapply the server's expiration policy.

FreeIPA and ipauser have no separate middle-name attribute. `test-user` is
appended to the given name and placed between first and last in the full and
display names. Each fictional user also gets `userclass: test-user` for exact
selection. With an authenticated IPA CLI session:

```sh
ipa user-find --userclass=test-user --all
ipa user-find test-user --all
ipa user-show test-avery-stone --all
```

Turning creation off leaves existing users alone. To explicitly and permanently
delete the ten reserved fictional accounts:

```sh
ansible-playbook users.yml -e ipa_delete_test_users=on
```

Cleanup targets only the ten known usernames, not every marked account.
Repeated cleanup is idempotent. Enabling creation and deletion together fails
validation.

## Validation

```sh
ansible-playbook users.yml --syntax-check
ansible-playbook users.yml --list-hosts
ansible-playbook users.yml -e @vars/users.yml --check
```

Check mode requires a reachable server and credentials. Verify live idempotency
by running the same command twice in your lab: the second run should report no
changes. Syntax and host-selection checks passed during workspace creation;
no live FreeIPA changes or integration tests were performed.
