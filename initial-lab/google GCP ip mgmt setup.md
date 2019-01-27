## Resources for setting up EVE-NG in the Goolge Compute Platform

These resources are good ideal to do the setup - nothing for me to add - everything here is to confgure the detail beyond the linked guides.

Comprehensive post by [ithitman](http://ithitman.blogspot.com/2018/04/configuring-eve-ng-on-google-compute.html)

Tony Es similar but basic blog [post](https://showipintbri.blogspot.com/2018/08/eve-ng-in-cloud.html) which points to his comprehensive video on [YouTube](https://www.youtube.com/watch?v=HDHsMgCs0XU)

I setup my VM in the europe-west1 region due to the need for more cores not availiable in the UK DCs.

10.132.0.0/20 is the europe-west1 region VPC

#### Cloud9 setup on EVE-NG server (allows Internet and mgmt)

    ip address add 10.132.0.10/20 dev pnet9

    echo 1 > /proc/sys/net/ipv4/ip_forward

    iptables -t nat -A POSTROUTING -o pnet0 -s 10.132.0.0/20 -j MASQUERADE

##### To make pnet9 IP persistent

1- Edit the interfaces file

    sudo nano /etc/network/interfaces

2- edit eth9 for your preferred IP

    iface eth9 inet manual
    auto pnet9
    iface pnet9 inet static
        address 10.132.0.10
        netmask 255.255.240.0
        bridge_ports eth9
        bridge_stp off

##### To make iptables persistent

1-

    sudo apt-get install iptables-persistent

2-

    sudo netfilter-persistent save

3-

    sudo netfilter-persistent reload

4- reboot the system and do the following command:

    sudo iptables -t nat -L

it should show the rule as persistent

##### To make ip_forward persistent

1-

    sudo nano /etc/sysctl.conf

2- Uncomment 'net.ipv4.ip_forward=1' and save

3- issue the following command:

    sudo sysctl -p /etc/sysctl.conf

4- reboot the system and do the following command:

    cat /proc/sys/net/ipv4/ip_forward

it should show 1 in the output

Check routing on the EVE-NG server:

    root@sipart-eve:~# route
    Kernel IP routing table
    Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
    default         10.132.0.1      0.0.0.0         UG    0      0        0 pnet0
    10.132.0.0      *               255.255.240.0   U     0      0        0 pnet9
    10.132.0.1      *               255.255.255.255 UH    0      0        0 pnet0

    root@sipart-eve:~# ip route list
    default via 10.132.0.1 dev pnet0
    10.132.0.0/20 dev pnet9  proto kernel  scope link  src 10.132.0.10
    10.132.0.1 dev pnet0  scope link

#### Setup of network device mgmt. and Linux box mgmt.

Connect devices mgmt. ints (fxp0 on vMX) to cloud9 network object

Set IP on mgmt. interfaces -

10.132.0.201/20 and up for PEs

10.132.0.221/20 and up for CEs

#### Add automation Linux box image ([18.04 Ubuntu server](https://ipnet.xyz/2018/06/ubuntu-image-for-eve-ng-python-for-network-engineers/)) to EVE-NG and add to any topology

** Edit netplan file to get Linux box networked

    sudo nano /etc/netplan/01-netcfg.yaml

    This file describes the network interfaces available on your system
    For more information, see netplan(5).
    network:
     version: 2
      renderer: networkd
      ethernets:
        ens3:
         addresses: [10.132.0.200/20]
         gateway4: 10.132.0.2
         nameservers:
           addresses: [8.8.8.8, 1.1.1.1]
         dhcp4: no

Apply netplan changes
    
    sudo netplan apply

Check if you can ping 8.8.8.8 :-)

Extras to install on the Linux box:

        sudo ansible-galaxy install Juniper.junos

        sudo pip2 install git+https://github.com/Juniper/py-junos-eznc.git

        sudo pip install git+https://github.com/networkop/ssh-copy-net.git

Create SSH key when logged in as pfne:

        ssh-keygen -t rsa -b 2048

Use SSH copy util to copy SSH key of current logged in Linux host account to Juniper boxes in this case 'pfne' (make sure a superuser account already exists on the Juniper device - in this case the 'lab' user)

    pfne@ubuntu1804-pfne:~$ ssh-copy-net 10.132.0.201 juniper
    Username: lab
    Password: lab123
    All Done!
    pfne@ubuntu1804-pfne:~$

##### [TMUX](https://linuxize.com/post/getting-started-with-tmux/) - multi window access from EVE-NG SSH session - 
    Ctrl+b c Create a new window (with shell)

    Ctrl+b % Split current pane horizontally into two panes
    Ctrl+b " Split current pane vertically into two panes

    Ctrl+b o Go to the next pane
    Ctrl+b x Close the current pane

    Ctrl+b ; Toggle between current and previous pane
    Ctrl+b w Choose window from a list
    Ctrl+b 0 Switch to window 0 (by number )
    Ctrl+b , Rename the current window

#### Ansible

All tools should be setup for Ansible on the Linux host

Lets check netconf to R1 (this assumes SSH RSA key access is working - see above)

    pfne@ubuntu1804-pfne:~$ ssh -s pfne@10.132.0.201 netconf
    <!-- No zombies were killed during the creation of this user interface -->
    <!-- user pfne, class j-super-user -->
    <hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
      <capabilities>
        <capability>urn:ietf:params:netconf:base:1.0</capability>
        <capability>urn:ietf:params:netconf:capability:candidate:1.0</capability>
        <capability>urn:ietf:params:netconf:capability:confirmed-commit:1.0</capability>
        <capability>urn:ietf:params:netconf:capability:validate:1.0</capability>
        <capability>urn:ietf:params:netconf:capability:url:1.0?scheme=http,ftp,file</capability>
        <capability>urn:ietf:params:xml:ns:netconf:base:1.0</capability>
        <capability>urn:ietf:params:xml:ns:netconf:capability:candidate:1.0</capability>
        <capability>urn:ietf:params:xml:ns:netconf:capability:confirmed-commit:1.0</capability>
        <capability>urn:ietf:params:xml:ns:netconf:capability:validate:1.0</capability>
        <capability>urn:ietf:params:xml:ns:netconf:capability:url:1.0?protocol=http,ftp,file</capability>
        <capability>urn:ietf:params:xml:ns:yang:ietf-netconf-monitoring</capability>
        <capability>http://xml.juniper.net/netconf/junos/1.0</capability>
        <capability>http://xml.juniper.net/dmi/system/1.0</capability>
      </capabilities>
      <session-id>5363</session-id>
    </hello>



Update the /etc/hosts file for name resoloution

Update the /etc/ansible/hosts for Ansible inventory use

```
pfne@ubuntu1804-pfne:~$ cat /etc/ansible/hosts
# This is the default ansible 'hosts' file.
#
# It should live in /etc/ansible/hosts
#
#   - Comments begin with the '#' character
#   - Blank lines are ignored
#   - Groups of hosts are delimited by [header] elements
#   - You can enter hostnames or ip addresses
#   - A hostname/ip can be a member of multiple groups

[MX]
r1 ansible_connection=local
r2 ansible_connection=local
r3 ansible_connection=local
r4 ansible_connection=local
r5 ansible_connection=local
r6 ansible_connection=local
r7 ansible_connection=local
r8 ansible_connection=local
r9 ansible_connection=local

[SRX]
ce1 ansible_connection=local
ce2 ansible_connection=local
ce3 ansible_connection=local
ce4 ansible_connection=local
ce5 ansible_connection=local
ce6 ansible_connection=local

[JUNIPER:children]
MX
SRX

[JUNIPER:vars]
juniper_user=pfne
os=junos
```

```
pfne@ubuntu1804-pfne:~$ tree
.
├── ansible-automation
│   └── juniper_core
│       ├── conf
│       ├── group_vars
│       ├── host_vars
│       ├── info
│       │   └── interfaces
│       ├── logs
│       └── playbooks
```

```
pfne@ubuntu1804-pfne:~$ cat ansible-automation/juniper_core/playbooks/get_conf_and_int.yml
---
- name: GET
  hosts: JUNIPER
  roles:
  - Juniper.junos
  connection: local
  gather_facts: no

  # Execute tasks (plays) this way "ansible-playbook <path>/GET.yml --tags <tag-name>"
  tasks:

### JUNOS_GET_CONFIG SECTION
  # Execute "show configuration" and save output to a file
  - name: Pull Down The Configs
    juniper_junos_command:
      commands:
          - "show configuration | display set"
      host: "{{ inventory_hostname }}"
      logfile: ../logs/get_config.log
      format: text
      dest: "../conf/{{ inventory_hostname }}.conf"

### JUNOS_CLI SECTION
  # Get list of interfaces and save output to a file
  - name: GET-INTERFACES
    juniper_junos_command:
      commands:
          - "show interfaces terse"
          - "show interfaces descriptions"
      host: "{{ inventory_hostname }}"
      logfile: ../logs/get_interfaces.log
      format: text
      dest: "../info/interfaces/{{ inventory_hostname }}_get-interfaces.output"

### EOF ###
```

```
pfne@ubuntu1804-pfne:~$ ansible-playbook ansible-automation/juniper_core/playbooks/get_conf_and_int.yml --limit=MX

PLAY [GET] *******************************************************************************************************************************************************************************

TASK [Pull Down The Configs] *************************************************************************************************************************************************************
ok: [r5]
ok: [r2]
ok: [r1]
ok: [r3]
ok: [r4]
ok: [r7]
ok: [r9]
ok: [r8]
ok: [r6]

TASK [GET-INTERFACES] ********************************************************************************************************************************************************************
ok: [r1]
ok: [r2]
ok: [r5]
ok: [r3]
ok: [r4]
ok: [r8]
ok: [r6]
ok: [r7]
ok: [r9]

PLAY RECAP *******************************************************************************************************************************************************************************
r1                         : ok=2    changed=0    unreachable=0    failed=0
r2                         : ok=2    changed=0    unreachable=0    failed=0
r3                         : ok=2    changed=0    unreachable=0    failed=0
r4                         : ok=2    changed=0    unreachable=0    failed=0
r5                         : ok=2    changed=0    unreachable=0    failed=0
r6                         : ok=2    changed=0    unreachable=0    failed=0
r7                         : ok=2    changed=0    unreachable=0    failed=0
r8                         : ok=2    changed=0    unreachable=0    failed=0
r9                         : ok=2    changed=0    unreachable=0    failed=0
```

```
pfne@ubuntu1804-pfne:~$ tree
.
├── ansible-automation
│   └── juniper_core
│       ├── conf
│       │   ├── r1.conf
│       │   ├── r2.conf
│       │   ├── r3.conf
│       │   ├── r4.conf
│       │   ├── r5.conf
│       │   ├── r6.conf
│       │   ├── r7.conf
│       │   ├── r8.conf
│       │   └── r9.conf
│       ├── group_vars
│       ├── host_vars
│       ├── info
│       │   └── interfaces
│       │       ├── r1_get-interfaces.output
│       │       ├── r2_get-interfaces.output
│       │       ├── r3_get-interfaces.output
│       │       ├── r4_get-interfaces.output
│       │       ├── r5_get-interfaces.output
│       │       ├── r6_get-interfaces.output
│       │       ├── r7_get-interfaces.output
│       │       ├── r8_get-interfaces.output
│       │       └── r9_get-interfaces.output
│       ├── logs
│       │   ├── get_config.log
│       │   └── get_interfaces.log
│       └── playbooks
│           ├── get_conf_and_int.retry
│           └── get_conf_and_int.yml
```
