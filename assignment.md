# 🛠️ Use Ansible to Create a Windows Server and Add 5 Users

## ✅ Goal

Use Ansible to:

1. Create a Windows EC2 instance
2. Create 5 local admin users on the instance

---

## 🔹 Step 1: Launch a Windows EC2 Instance Manually

Since the Ansible EC2 module caused issues, do this manually:

* Go to **AWS Console > EC2 > Launch Instance**
* Choose:

  * **AMI**: Windows Server 2019/2022 Base
  * **Instance type**: t2.micro (or higher)
* Configure:

  * Create/select key pair (.pem)
  * In Security Group:

    * ✅ Allow TCP **3389** (RDP)
    * ✅ Allow TCP **5985** (WinRM HTTP) — Source: your IP `/32`

Launch the instance.

---

## 🔹 Step 2: Enable WinRM on the EC2 Instance (via RDP)

1. Use RDP to connect (download `.rdp` file and get admin password from EC2 Console)
2. In PowerShell (as Administrator), run:

```powershell
winrm quickconfig -q
winrm set winrm/config/service/auth '@{Basic="true"}'
winrm set winrm/config/service '@{AllowUnencrypted="true"}'
netsh advfirewall firewall set rule group="Windows Remote Management" new enable=yes
```

3. Create an Ansible user:

```powershell
net user ansibleadmin "P@ssw0rd123!" /add
net localgroup administrators ansibleadmin /add
```

---

## 🔹 Step 3: Confirm Port 5985 Is Open

From local terminal:

```bash
nmap -Pn -p 5985 <ec2-public-ip>
```

Expected output:

```
5985/tcp open  http
```

---

## 🔹 Step 4: Create `inventory.yml`

```yaml
all:
  hosts:
    windows_server:
      ansible_host: <your-ec2-public-ip>
      ansible_user: ansibleadmin
      ansible_password: P@ssw0rd123!
      ansible_connection: winrm
      ansible_port: 5985
      ansible_winrm_transport: basic
      ansible_winrm_server_cert_validation: ignore
```

---

## 🔹 Step 5: Test the Ansible Connection

```bash
ansible windows_server -i inventory.yml -m win_ping
```

Expected result:

```json
windows_server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

## 🔹 Step 6: Create a Playbook to Add 5 Users

Save as `create_windows_users.yml`:

```yaml
---
- name: Create 5 local users on Windows Server
  hosts: windows_server
  gather_facts: no
  tasks:
    - name: Create local users
      win_user:
        name: "{{ item.name }}"
        password: "{{ item.password }}"
        state: present
        groups:
          - Administrators
      loop:
        - { name: 'user1', password: 'P@ssw0rd123!' }
        - { name: 'user2', password: 'P@ssw0rd123!' }
        - { name: 'user3', password: 'P@ssw0rd123!' }
        - { name: 'user4', password: 'P@ssw0rd123!' }
        - { name: 'user5', password: 'P@ssw0rd123!' }
```

---

## 🔹 Step 7: Run the Playbook

```bash
ansible-playbook -i inventory.yml create_windows_users.yml
```

Expected output:

```
ok: [windows_server] => (item=user1)
ok: [windows_server] => (item=user2)
...
```

---

## 🔹 Step 8: Verify Users via RDP

In PowerShell (on the instance):

```powershell
net user
```

Should show:

```
user1
user2
user3
user4
user5
```

---

## ✅ Summary Table

| Step | Action                       | Status |
| ---- | ---------------------------- | ------ |
| 1    | Launch Windows Server EC2    | ✅      |
| 2    | Enable WinRM and create user | ✅      |
| 3    | Open port 5985               | ✅      |
| 4    | Configure `inventory.yml`    | ✅      |
| 5    | Test WinRM with `win_ping`   | ✅      |
| 6    | Create playbook to add users | ✅      |
| 7    | Run playbook                 | ✅      |
| 8    | Verify users                 | ✅      |

---

**Optional Next Steps:**

* Use Ansible Vault to secure passwords
* Add users to other groups like `Remote Desktop Users`
* Configure WinRM over HTTPS (port 5986) for production
