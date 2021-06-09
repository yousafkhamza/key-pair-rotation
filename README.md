# SSH Key Rotation. (AWS Cloud)
[![Builds](https://travis-ci.org/joemccann/dillinger.svg?branch=master)](https://travis-ci.org/joemccann/dillinger)

---
## Description 

It's an ansible playbook for SSH-Keypair rotation. I write a short brief when we use this: if a customer using a key for which he used all instances in a single key pair. But he needs to change his key periodically with all the instances used in AWS (Also, change the key fingerprint in AWS Console).  

---

## Feature
- Key pair Rotation easily applicable for multiple instances in single
- Easy to configure and use.
- No need to mention any hosts (Inventory) Because it's generated Dynamic Inventory appending your <old_key.pem>

---
## Pre-Requests 
- Install Ansible on your Master Machine (_localhost_)
- Create an IAM user role under your AWS account and please enter the values once the playbook running time
##### Installation
[Ansible2](https://docs.ansible.com/ansible/2.3/index.html) (For your reference visit [How to install Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html))
##### IAM Role Creation
[IAM Role Creation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create.html)
##### Ansible Modules used
- [ec2_instance_info](https://docs.ansible.com/ansible/latest/collections/community/aws/ec2_instance_info_module.html) 
- [ec2-key](https://docs.ansible.com/ansible/latest/collections/amazon/aws/ec2_key_module.html)
- [openssh_keypair](https://docs.ansible.com/ansible/latest/collections/community/crypto/openssh_keypair_module.html)
- [authorized_key](https://docs.ansible.com/ansible/2.4/authorized_key_module.html)
- [shell](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/shell_module.html)
---

# Architacture
![alt text](https://i.ibb.co/4jPnV0C/rotation.jpg)

---

### How To Use
Ansible Installation article is in the pre-request section so please check out the pre-request section.
```sh
amazon-linux-extras install -y ansible2
yum install git -y
git clone https://github.com/yousafkhamza/deploy-key.git
cd deploy-key
# --------------------------------------
# --- Please-Change-Your-Credentials ---
# --------------------------------------
ansible-playbook main.yml
```
---
## Behind the Playbook.
I just pasted the key.yml
```sh
- name: "Generating Inventory Of Key To Be Rotated"
  hosts: localhost
  vars_files:
    - key.vars  
  tasks:
    
    # ---------------------------------------------------------------
    # Getting Information About Of Ec2's Which Need key To be Rotated
    # ---------------------------------------------------------------
    
    - name: "Fetching Detail About Ec2 Instance"
      ec2_instance_info:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "ap-south-1"
        filters:
          "key-name": "{{ old_key }}"
          instance-state-name: [ "running"]
      register: ec2
    
    
    # ------------------------------------------------------------
    # Creating Inventory Of Ec2  With Old Ssh-keyPair
    # ------------------------------------------------------------    
    - name: "Creating Inventory "
      add_host:
        name: "{{ item.public_ip_address }}"
        groups: "aws"
        ansible_host: "{{ item.public_ip_address }}"
        ansible_port: 22
        ansible_user: "ec2-user"
        ansible_ssh_private_key_file: "{{ old_key }}.pem"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
      with_items:
        - "{{ ec2.instances }}"
      no_log: true     
                             
- name: "Updating SshKey Meterial"
  hosts: aws
  become: true
  gather_facts: false
  vars_files:
    - key.vars
  vars:
    old_key: "mumbai-production"
    tmp_key: "keypair-tmp"
  tasks:
    
    - name: "Creating New SSH-Kay Meterial"
      delegate_to: localhost
      run_once: True
      openssh_keypair:
        path: "{{ tmp_key }}"
        type: rsa
        size: 4096
        state: present
        
    - name: "Adding New SshKey Meterial"
      authorized_key:
       user: ec2-user
       state: present
       key: "{{ lookup('file', '{{ tmp_key }}.pub')  }}"
        
    - name: "Creating Ssh Connection Command"
      set_fact:
        ssh_connection: "ssh -o StrictHostKeyChecking=no -i {{ tmp_key }} {{ansible_ssh_user}}@{{ ansible_ssh_host }} 'uptime'" 

    - name: "Checking Connectivity To Ec2 Using Newly Added Key"
      ignore_errors: true
      delegate_to: localhost
      shell: "{{ ssh_connection }}" 
    
    - name: "Removing Old KeyMeterial"
      authorized_key:
       user: ec2-user
       state: present
       key: "{{ lookup('file', '{{ tmp_key }}.pub')  }}"
       exclusive: true
        
    - name: "Removing Old Ssh public From Aws Account "
      delegate_to: localhost
      run_once: True
      ec2_key:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        name: "{{ old_key }}"
        key_material: "{{ lookup('file', '{{ tmp_key }}.pub') }}"
        force: true
        state: present
            
    - name: "Renaming Local Pulbic Key"
      run_once: True
      delegate_to: localhost
      shell: "mv {{  tmp_key }}.pub  {{ old_key }}.pub"

    - name: "Renaming Local Private Key" 
      run_once: True
      delegate_to: localhost
      shell: "mv {{  tmp_key }}  {{ old_key }}.pem"
```

> key.vars are store variables actual values so Please change your values with the same and Please note that the key name doesn't need extension and please store the old key.pem file under the working directory

```sh
access_key: "Your access key"
secret_key: "<Your secret key>"
region: "<region which one you use>"       #----------> Please let me know if you have using one key in multiple regions then I will help you to change the playbook. 
old_key: "<KeyName which you need to change>"   #----> please past the old key name without pem extension and store private pem file on the same directory with 0400 permission
tmp_key: "keypair-tmp"  
```
---

# Conclusion

It's used for ssh key rotation on your AWS Cloud and which region and key you selected the playbook sort that key used instances and changed the SSH-Key inside the server and AWS Console at the same time

_By_
_Yousaf K Hamza_
