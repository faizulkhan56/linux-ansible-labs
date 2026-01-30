# Lab 05 — Variables, facts, and conditions

## Objective

- Use variables and facts
- Apply conditional logic (`when`)
- Practice running on groups/hosts

## Step 1 — Prepare lab folder

```bash
mkdir -p ~/ansible-labs/lab05
cd ~/ansible-labs/lab05
cp ~/ansible-labs/lab01/inventory.ini .
```

## Step 2 — Create playbook

Create `05-vars-facts-conditions.yml`:

```yaml
- name: Vars, facts, and conditions
  hosts: nodes
  become: true
  vars:
    install_tools: true
    tools:
      - curl
      - unzip
      - htop
  tasks:
    - name: Show OS facts
      ansible.builtin.debug:
        msg:
          - "Hostname: {{ ansible_hostname }}"
          - "OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"
          - "IP: {{ ansible_default_ipv4.address | default('n/a') }}"

    - name: Install tools only on Ubuntu (and only if install_tools=true)
      ansible.builtin.apt:
        name: "{{ tools }}"
        state: present
        update_cache: true
      when:
        - install_tools
        - ansible_os_family == "Debian"

    - name: Create a marker file only on node1
      ansible.builtin.copy:
        dest: /tmp/only-node1.txt
        content: "Hello from node1 at {{ ansible_date_time.iso8601 }}\n"
        mode: "0644"
      when: inventory_hostname == "node1"
```

## Step 3 — Run

```bash
ansible-playbook -i inventory.ini 05-vars-facts-conditions.yml
```

## Step 4 — Verify marker file

```bash
ansible -i inventory.ini nodes -m shell -a "ls -l /tmp/only-node1.txt || true"
```

