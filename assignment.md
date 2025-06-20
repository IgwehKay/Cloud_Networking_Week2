# 🛠️ Use Ansible to Create a Windows Server and Add 5 Users

## ✅ Goal

Use Ansible to:

1. Create a Windows EC2 instance
2. Create 5 local admin users on the instance

---

## 🔹 Step 1: Launch a Windows EC2 Instance using Ansible.
After signing in to AWS via CLI 

On terminal, Create a file name: **launch_windows.yaml**

nano launch_windows.yaml

Input the code below
```
---
- name: Launch Windows EC2 instance with WinRM pre-configured
  hosts: localhost
  connection: local
  gather_facts: no

  vars:
    region: eu-north-1
    key_name: yourkeypairfile
    instance_type: t3.micro
    ami_id: ami-0954c42276330704a  # Use actual Windows AMI in your region
    security_group: sg-0*************
    subnet_id: subnet-0**************

#Setting up WinRM
    user_data_script: |
      <powershell>
      winrm quickconfig -q
      winrm set winrm/config/service/auth '@{Basic="true"}'
      winrm set winrm/config/service '@{AllowUnencrypted="true"}'
      netsh advfirewall firewall set rule group="Windows Remote Management" new enable=yes
      net user ansibleadmin "P@ssw0rd123!" /add
      net localgroup administrators ansibleadmin /add
      Set-NetConnectionProfile -InterfaceAlias "Ethernet" -NetworkCategory Private
      </powershell>

  tasks:
    - name: Launch EC2 Windows instance
      community.aws.ec2_instance:
        name: AnsibleWindowsServer
        key_name: "{{ key_name }}"
        instance_type: "{{ instance_type }}"
        image_id: "{{ ami_id }}"
        region: "{{ region }}"
        wait: yes
        state: present
        count: 1
        tags:
          Name: AnsibleWindows
        vpc_subnet_id: "{{ subnet_id }}"
        security_group: "{{ security_group }}"
        user_data: "{{ user_data_script }}"
```

  * In Security Group of the Windows instance:

    * ✅ Allow TCP port 3389 (RDP) Source - Custom 0.0.0.0/0
    * ✅ Allow TCP port 5985 (WinRM HTTP) — Source: Custom, your IP `/32` or 0.0.0.0/0

Launch the instance.
```
ansible-playbook launch_windows.yaml
```
---

## 🔹 Step 2: Enable WinRM on the EC2 Instance (via RDP)

1. Use RDP to connect: Click on the instance ID, click on connect, select RDP  (download `.rdp` file and get admin password from EC2 Console)
2. Open the .rdp file, enter the password. 
3. In the RDP terminal (as Administrator, or PowerShell (as Administrator), run:

```
winrm quickconfig -q
winrm set winrm/config/service/auth '@{Basic="true"}'
winrm set winrm/config/service '@{AllowUnencrypted="true"}'
netsh advfirewall firewall set rule group="Windows Remote Management" new enable=yes
```

3. Create an Ansible user:

```
net user ansibleadmin "P@ssw0rd123!" /add
net localgroup administrators ansibleadmin /add
```

---

## 🔹 Step 3: Confirm Port 5985 Is Open

From local (VS code) terminal:

```
nmap -Pn -p 5985 <ec2-public-ip>
```

Expected output:

```
5985/tcp open  http
```

---

## 🔹 Step 4: Create `inventory.yml`

```
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

```
ansible windows_server -i inventory.yml -m win_ping
```

Expected result:

```
windows_server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

## 🔹 Step 6: Create a Playbook to Add 5 Users

Save as `create_windows_users.yml`:

```
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

```
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

```
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

![image](https://github.com/user-attachments/assets/8fcc0952-1f03-4201-b3eb-d4166b062056)

