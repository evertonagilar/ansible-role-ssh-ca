
Ansible role to manage SSH Certificate Authority (CA) and issue user certificates.

This role allows you to:

* Generate a local SSH CA (only once)
* Configure remote servers to trust the CA
* Issue SSH certificates for users with defined principals and validity

---

## Requirements

* Ansible >= 2.10
* OpenSSL >= 3.0.2

---

## Role Variables

Available variables are listed below, along with default values:

```yaml
ssh_ca_dir: "{{ playbook_dir }}/ssh-ca"
ssh_ca_private_key: "{{ ssh_ca_dir }}/ssh_ca"
ssh_ca_public_key: "{{ ssh_ca_dir }}/ssh_ca.pub"

# List of users to issue SSH certificates
ssh_ca_users: []
# Example:
# ssh_ca_users:
#   - name: devops
#     validity: "+1825d"
#     principals: "evertonagilar,vagrant,ubuntu"
```

* `ssh_ca_dir`: Directory on the controller to store CA keys.
* `ssh_ca_private_key`: Private key of the SSH CA.
* `ssh_ca_public_key`: Public key of the SSH CA.
* `ssh_ca_users`: List of users for whom certificates will be issued.

  * `name`: Profile name for the user. This will be used as the base filename for both the private key and the issued certificate.
  * `validity`: Duration for which the issued certificate is valid. Can be expressed in days (`d`), weeks (`w`), or months (`m`), e.g., `+30d` for 30 days, `+4w` for 4 weeks, `+6m` for 6 months.
  * `principals`: Comma-separated list of usernames on the server who are allowed to authenticate using this certificate. These usernames **must exist on the target server**.

---

## Example Playbook

```yaml
- hosts: all
  become: true
  roles:
    - role: evertonagilar.ssh-ca
      vars:
        ssh_ca_users:
          - name: devops
            validity: "+1825d"
            principals: "evertonagilar,vagrant,ubuntu"
```

---

## How it works

1. **Generate CA**
   The role will generate the CA keys on the Ansible controller if they do not exist.

2. **Distribute CA public key**
   The public key is copied to remote servers and added to `sshd_config` via `TrustedUserCAKeys`.
   A handler restarts `sshd` only if configuration changes.

3. **Issue user certificates**
   For each user in `ssh_ca_users`, the role generates a user key pair (if missing) and issues a certificate signed by the CA.

4. **Connect using certificates**
   Users authenticate to the server using:

   ```bash
   ssh -i devops -i devops-cert.pub username@server
   ```

   Or configure your `~/.ssh/config`:

   ```ssh
   Host server
       HostName server.example.com
       User evertonagilar
       IdentityFile ~/.ssh/devops
       CertificateFile ~/.ssh/devops-cert.pub
   ```

---

## Manual CA Generation (for understanding)

```bash
# 1. Generate SSH CA keys
ssh-keygen -t ed25519 -f ~/ssh-ca/ssh_ca -C "SSH CA" -N ''

# 2. Copy public key to server
scp ~/ssh-ca/ssh_ca.pub user@server:/etc/ssh/ca.pub

# 3. Configure server to trust CA
echo "TrustedUserCAKeys /etc/ssh/ca.pub" | sudo tee -a /etc/ssh/sshd_config
sudo systemctl restart sshd

# 4. Generate user key pair
ssh-keygen -t ed25519 -f ~/.ssh/devops -N ''

# 5. Issue user certificate
ssh-keygen -s ~/ssh-ca/ssh_ca -I devops -n "evertonagilar,vagrant" -V +4w ~/.ssh/devops.pub

# 6. Connect using key + certificate
ssh -i ~/.ssh/devops -i ~/.ssh/devops-cert.pub evertonagilar@server
```

### SSH Key Generation Parameters Explained

When generating the SSH CA or user keys, the role uses `ssh-keygen` with the following options:

| Parameter | Explanation |
|-----------|-------------|
| `-t ed25519` | Specifies the key type. `ed25519` is a modern elliptic curve algorithm, offering high security and performance. |
| `-f <path>` | File path where the private key will be saved. The public key will be created automatically with `.pub` appended. Example: `~/ssh-ca/ssh_ca` → creates `ssh_ca` (private) and `ssh_ca.pub` (public). |
| `-C "<comment>"` | Comment associated with the key, useful for identification. Example: `"SSH CA"` for the Certificate Authority key. |
| `-N '<passphrase>'` | Passphrase for the private key. Empty (`''`) means no passphrase, allowing automated use without prompts. |
| `-s <ca_key>` | Sign a user key with the specified CA key. Used when issuing certificates. |
| `-I <identity>` | Identity string for the certificate, usually the username or profile name. |
| `-n <principals>` | Comma-separated list of usernames allowed to authenticate using this certificate. Must exist on the target server. |
| `-V <validity>` | Validity period for the issued certificate. Can be expressed in days (`d`), weeks (`w`), or months (`m`). Example: `+4w` for 4 weeks. |


---

## Notes

* The CA should be generated **once** and reused for all hosts.
* Certificates are **time-limited** (`validity`) and define which server users (`principals`) can authenticate.
* Principals specified must exist on the target server.
* This approach is safer and more scalable than copying public keys to `authorized_keys` manually.

---

## License

MIT

---

## Author Information

Created by Everton Agilar ([evertonagilar@gmail.com](mailto:evertonagilar@gmail.com))
