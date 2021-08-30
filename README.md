# Ansible Bootstrap

Get a remote host ready to be managed with Ansible.

## Requirements

Assumes a pre-existing user account is available on the target with sufficient
privilege and that the host is accessible via ssh public key authentication.

Uses the pre-existing user on the target host to create an `ansible` user and
copies the ssh public key, sshd_config file, and custom sudoers file to the
target host.

On first run, the `ansible.cfg` file should only specify the inventory file
path. the `ansible` remote user and private key file can be added after the
playbook runs once and successfully creates the `ansible` account on the remote
host.

Alternatively, you can specify the user for the initial run at the commandline
with:

```
ansible-playbook -i=pilot, --user=lido -K bootstrap.yml
```

Note: When targeting a single hostname (pilot) the name must be followed by a
trailing comma. Without the comma `ansible-playbook` barfs on being "unable to
parse the single hostname as an inventory source" since it think the single
name is a file in the present working directory:

```
[WARNING]: Unable to parse /home/lido/projects/ansible/ansible-bootstrap/pilot as an inventory source
[WARNING]: No inventory was parsed, only implicit localhost is available
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not
match 'all'
```

## Demo Environment

Demo environment consists of three nodes:

1. tower: this is the control host / management workstation.
1. pilot: this is a target host running Ubuntu "focal".
1. gunner: this is a target host running Debian "bullseye".

## SSH

I generated an `ed25519` SSH keypair for my user (`lido`) on the control host.
I specified a passphrase for this key and copied it to the remote hosts that
will be managed with ansible using `ssh-copy-id`.

```
ssh-keygen -t ed25519 -C "lido@tower"
ssh-copy-id pilot
```

I also created an `ed25519` keypair for the `ansible` account that will be used
for all future administration once bootstrapped. I did not specify a passphrase
and also made sure to save this key with its own unique name of `ansible`.

```
ssh-keygen -t ed25519 -C "ansible"
```

The target hosts have an account named `lido` and I have already copied my ssh
public key to the remote hosts using the `ssh-copy-id` command. I also have an
inventory file in the current directory that includes the hostnames. With this
I should be able to successfully run an ansible ad-hoc command using the ping
module to connect to the target node.

```
lido@tower:~/projects/ansible/ansible-bootstrap$ ansible all -m ping
gunner | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
pilot | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

## Usage

Now I can run `ansible-playbook main.yml --ask-become-pass` providing my
password to elevate privileges on the target hosts and finally see the play
run successfully.

```
lido@tower:~/projects/ansible/ansible-bootstrap$ ansible-playbook main.yml --ask-become-pass
BECOME password:

PLAY [all] *******************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************
ok: [pilot]

TASK [update repo cache] *****************************************************************************************
ok: [pilot]

PLAY [all] *******************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************
ok: [pilot]

TASK [~/projects/ansible/ansible-bootstrap/ : create the ansible user] **************
changed: [pilot]

TASK [~/projects/ansible/ansible-bootstrap/ : add ssh key for the ansible user] *****
changed: [pilot]

TASK [~/projects/ansible/ansible-bootstrap/ : add custom sudoers file] **************
changed: [pilot]

TASK [~/projects/ansible/ansible-bootstrap/ : add custom sshd_config file] **********
changed: [pilot]

RUNNING HANDLER [~/projects/ansible/ansible-bootstrap/ : restart_sshd] **************
changed: [pilot]

PLAY RECAP *******************************************************************************************************
pilot                     : ok=8    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

After the play completes and adds the necessary configuration for the `ansible`
user account, we can edit the `ansible.cfg` file in the present working
directory to direct ansible to use the updated `ansible` user account and it's
corresponding key.

```
lido@tower:~/projects/ansible/ansible-bootstrap$ cat ansible.cfg
[defaults]
inventory = inventory
private_key_file = ~/.ssh/ansible
remote_user = ansible
```

A note on the Debian host: I installed from the netinst image and I had to make
sure that the following packages were installed in order for this playbook to
work:

```
python3
python3-apt
gnupg
sudo
```

I found some bootstrap roles on Galaxy to acheive this but this requires more
learning/testing on my part.
