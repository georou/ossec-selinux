# ossec-selinux

This policy is written to contain the wide and varried access of [OSSEC](https://ossec.github.io).

This is a fork from the original policy written by Psi-Jack at [Linux-Help.org - Git](https://git.linux-help.org/Linux-Help/selinux-ossec/src/bugfix/style-organization/ossec.te). Many thanks to his hours of work on the original policy.

I have altered it with improvements, changing the unconfined domain of ossec_ar to confined and also added booleans to help limit unintentional access as described below. It should now be the correct style according to the [Tresys Style Guide](https://github.com/TresysTechnology/refpolicy/wiki/StyleGuide). The original commit of the policy is [here](https://github.com/georou/ossec-selinux/commit/ef2a8d9ba92b315943baf4a4ed941b3c4395a0cc) and then my subsequent [changes](https://github.com/georou/ossec-selinux/commit/475d3138dc8571d8d539572d7a07bc25635348df).

Since the Active Response function of OSSEC is now confined, using the scripts listed in the "Not Tested/Broken" section will most likely not work. At the moment this is because I don't use them and haven't had time to implement them. 
As listed below, what works and what isn't tested. Please note this policy is complex and is in a beta state. However, it is somewhat production ready but the usual monitoring of AVC denails needs to be observed. I currently use it for my systems and will continually commit changes here as they arise.


### Working:
* All basic functionality - monitoring of logs and reporting via email etc.
* Using Active-Response to block IP addresses for iptables and firewalld in `/var/ossec/active-response/bin/firewall{d}-drop.sh`


### Not Tested/Broken:
* All of the following Active-Reponse scripts are untested: (`disable-account.sh  host-deny.sh  ip-customblock.sh  ossec-slack.sh  ossec-tweeter.sh  restart-ossec.sh  route-null.sh`)
* Using remote agents is untested and partially implemented with remoted_t. You will need to enable it's boolean. remoted uses ports 1514 and 514. ossec_remoted_port_t has been defined.
* csyslogd is untested
* dbd is untested

### Needed Tweaks:
The firewall{d}-drop.sh scrip has a variable $PWD which as OSSEC execd does not chroot, returns as the / directory. This needs to be changed so that the pid and lock files it creates are done so in `/var/ossec/var/run`. You WILL need to make this change:
```
firewalld{d}-drop.sh:
#PWD=`pwd`
PWD="/var/ossec/var/run"
```

### Booleans needed for specific functions:
```
ossec_ar_can_edit_iptables --> off
ossec_ar_can_exec_system_bin --> off
ossec_remoted_can_network_connect --> off
ossec_can_read_all_system_logs --> off
```

### Suggestions:
* If you don't enable the read_all_system_logs boolean, you will need to add the logs you want OSSEC to read. Otherwise if you're fine with the access, enable: `setsebool -P ossec_can_read_all_system_lods on`



## Installation:
```sh
# Clone the repo
git clone https://github.com/georou/ossec-selinux.git

# Optional - Copy relevant .if interface file to /usr/share/selinux/devel/include to expose them when building future modules. Safe to ignore duplicate warnings when using make -f
install -Dp -m 0664 -o root -g root ossec.if /usr/share/selinux/devel/include/myapplications/ossec.if

# Compile the selinux module (see below)

# Install the SELinux policy module. Compile it before hand to ensure proper compatibility (see below)
semodule -i ossec.pp

# Restore all the correct context labels
restorecon -Rv /var/ossec
restorecon -v /etc/init.d/ossec-hids

# Start ossec-hids
systemctl enable ossec-hids.service
systemctl start ossec-hids.service

# Ensure it's working
ps -eZ | grep ossec
```

## How To Compile The Module Locally (Needed before installing)
Ensure you have the `selinux-policy-devel` package installed.
```sh
# Ensure you have the devel packages
yum install selinux-policy-devel setools-console
# Change to the directory containing the .if, .fc & .te files
cd ossec-selinux
make -f /usr/share/selinux/devel/Makefile ossec.pp
semodule -i ossec.pp
```

## Debugging and Troubleshooting

* If you're getting permission errors, uncomment permissive in the .te file and try again. Re-check logs for any issues. Or `semanage permissive -a ossec_t`
* Easy way to add in allow rules is the below command, then copy or redirect into the .te module. Rebuild and re-install:
* Don't forget to actually look at what is suggested. audit2allow will most likely go for a coarse grained permission!

```sh
ausearch -m avc,user_avc,selinux_err -ts recent | audit2allow -R
```
If you get a could not open interface info [/var/lib/sepolgen/interface_info] error. 
Ensure policycoreutils-devel is installed and/or run: `sepolgen-ifgen`


## Compatibility Notes
Built on CentOS 7.4 at the time with:
```
selinux-policy-devel-3.13.1-166.el7_4.5.noarch
policycoreutils-devel-2.5-17.1.el7.x86_64
policycoreutils-2.5-17.1.el7.x86_64
selinux-policy-targeted-3.13.1-166.el7_4.5.noarch
policycoreutils-python-2.5-17.1.el7.x86_64
selinux-policy-3.13.1-166.el7_4.5.noarch
```
