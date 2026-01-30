# Lab 07 — Ansible Vault (intro)

## Objective

- Store secrets in an encrypted file
- Use secrets in a playbook safely

## Step 1 — Prepare folder structure

```bash
mkdir -p ~/ansible-labs/lab07/group_vars/all
cd ~/ansible-labs/lab07
cp ~/ansible-labs/lab01/inventory.ini .
```

## Step 2 — Create a vault file

Create an encrypted file:

```bash
ansible-vault create group_vars/all/vault.yml
```

Put this inside (example):

```yaml
demo_password: "SuperSecret123!"
```

Save and exit.

## Step 3 — Create playbook that uses the secret

Create `07-vault-demo.yml`:

```yaml
- name: Vault demo (do not print secrets in real projects)
  hosts: nodes
  become: false
  vars_files:
    - group_vars/all/vault.yml
  tasks:
    - name: Show that we can read a vault variable (training only)
      ansible.builtin.debug:
        msg: "Vault variable demo_password length: {{ demo_password | length }}"
```

Run:

```bash
ansible-playbook -i inventory.ini 07-vault-demo.yml --ask-vault-pass
```

## Step 4 — Best practices

- Do not commit real secrets to Git.
- Prefer external secret managers for production.
- Avoid printing secrets using `debug`.

