# Update SlurmDB, Controller, and Client

This Ansible playbook automates the process of updating SlurmDB, Controller, and Client nodes. The playbook performs tasks such as creating backup directories, verifying configurations, downloading and extracting new releases from GitHub, and restarting Slurm services. https://confluence.dug.com/pages/viewpage.action?spaceKey=DUGIT&title=20240122+Update+London%27s+SLURM+to+slurm-23-02-7-1-DUG-5

## ✨ Requirements

- Ansible installed on the control machine.
- SSH access to the Slurm nodes (SlurmDB, Controller, and Client).
- GitHub personal access token for downloading releases.
- SQL key for database access.

## ❔ Variables

- `get_url_args`: Dictionary containing the destination path for downloaded files.
- `asset_name`: The name of the GitHub release asset.
- `github_pat`: GitHub personal access token.
- `sql_key`: SQL key for database access.
- `backup_dir`: Directory path for backups.
- `gh_release_api`: GitHub API URL for the latest release.

## 💻 Hosts

- `lslurmdb`: Hosts related to SlurmDB.
- `lslurmcontroller`: Hosts related to the Slurm Controller.
- `lslurmclient`: Hosts related to the Slurm Client.

## 🚀 Tasks

1. **Ensure the backup directory exists**
   - Creates backup directories on the SlurmDB and SlurmController nodes.

```yaml
    - name: Ensure the backup directory exists
      ansible.builtin.file:
        path: "{{ backup_dir }}"
        mode: '0755'
        state: directory

```

2. **Run the sanity check script**
   - Runs the `sanity_slurm_conf.sh` script to verify the Slurm configuration on the Controller.
```yaml
      ansible.builtin.shell: DEBUG=1 /d/sw/slurm/etc/sanity_slurm_conf.sh
```

3. **Extract and compare configuration files**
   - Compares the current Slurm configuration file and failed if it doesn't match (ensure all dynamic changes have been written)
```yaml
      failed_when: diff_result.rc != 0 
      when: inventory_hostname in groups['lslurmcontroller']

```

4. **Obtain GitHub release details**
   - Fetches the latest release details from the GitHub API
```yaml
      ansible.builtin.set_fact:
        gh_release_asset_url: "{{ gh_release.json.assets | selectattr('name', 'contains', asset_name) | map(attribute='url') | first }}"
        asset_dir_name: "{{ gh_release.json.assets | selectattr('name', 'contains', asset_name) | map(attribute='name') | first | regex_replace('.tar.bz2', '') }}"

```
5. **Download and extract the release**
   - Downloads and extracts the specified release asset from GitHub.
```yaml
      ansible.builtin.unarchive:
        src: "{{ download_dest.dest }}"
        dest: "{{ get_url_args.dest }}"
        remote_src: yes
```
6. **Stop Slurm services**
   - Stops the SlurmDB, SlurmController, and SlurmClient services and failed if it still on active state
```yaml
      failed_when: "'restarted' in slurmdbd_status.state"
```
7. **Backup the configuration**
   - Copies the existing configuration to a backup directory.
```yaml
      ansible.builtin.copy:
        src: "/var/pool/slurm"
        dest: "{{ get_url_args.dest }}/backup_var_spool_slurm_{{ lookup('pipe', 'date +%Y%m%d') }}"
```
8. **Create relative symlinks**
   - Creates symbolic links for the new Slurm version.
```yaml
      ansible.builtin.file:
        src: "{{ asset_dir_name }}"
        dest: "{{ get_url_args.dest }}/d/sw/slurm/latest"
        state: link
```
9. **Restart Slurm services**
   - Restarts the Slurm services on the SlurmDB, SlurmController, and SlurmClient nodes and failed if it stopped
```yaml
      failed_when: "'stopped' in slurmdbd_status.state"
```
## 📣 Usage

1. **Prepare the environment**
   - Ensure you have the required access tokens and keys in `~/.ssh/github_token.txt` and `~/.ssh/sql_token.txt`.

2. **Run the playbook**
   ```sh
    ansible-playbook -i inventory/hosts.yaml playbooks/slurm_rollout.yaml 
   ```


## 🐛 Debugging   
1. **Run the playbook**
   ```sh
    ansible-playbook -i inventory/hosts.yaml playbooks/slurm_rollout.yaml -vv
   ```

## 📜 Notes

- This is being done in the tested environment. Please change the `FastX3` services to the relevant services, and ensure the paths are correct as per the configuration.
- The playbook assumes that the GitHub token and SQL key files are stored in the `~/.ssh` directory.
- Adjust file paths and variables according to your specific setup.

## 👉 Pending
- [x] to move into /d/sw/slurm from its localData
- [x] update services to  relevant services
- [ ] update `host.yaml` to  relevant host
- [ ] test the key for dumping sql database into backup dir
- [ ] sanity check the size of dumped sql database
