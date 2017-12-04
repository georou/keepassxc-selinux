# keepassxc-selinux

This policy is designed to work on Fedora 26 with i3wm to confine keepassxc-2.2.2-1. Since it concerns passwords, use at your own risk.

The primary goals of this policy module is:
* Deny network access to the program
* Isolate the password file (.kdbx) from other programs on the system

A key note to know is because private types are used for the password file, it breaks the convention of SELinux that unconfined has access to everything.

The policy works as inteded to achieve the above goals as a GUI application. However, see below

### Current caveats:

* Policy is designed with the assumption that the user is using unconfined_u (which is standard on Fedora out-of-the-box)
* Moving the .kdbx away from the home directory needs to have `restorecon` run to restore its private label: keepassxc_db_t
* Using another window manager apart from i3wm will possibly need other permissions and is untested
* Auto-type under i3wm doesn't work, keepassxc bug see here: https://github.com/keepassxreboot/keepassxc/issues/457
* keepassxc-cli is untested
* Challenge-Response with Yubikey works. Untested with other brands - should still work
* Importing from keepass(original) is untested

### Booleans needed for specific functions:
```
Challenge-Response requires: keepassxc_can_access_usb_devices --> off
Reading a password file on a USB drive requires: keepassxc_can_access_usb_devices --> off
```

## Installation
```sh
# Clone the repo
git clone https://github.com/georou/keepassxc-selinux.git

# Compile the selinux module (see below)

# Install the SELinux policy module. Compile it before hand to ensure proper compatibility (see below)
semodule -i keepassxc.pp

# Restore all the correct context labels
restorecon -Rv /usr/share/keepassxc
restorecon -v /usr/bin/keepassxc
restorecon -v /usr/bin/keepassxc-cli

# Start keepassxc

# Ensure it's working
ps -eZ | grep keepassxc
```

## How To Compile The Module Locally(Recommended before installing)
Ensure you have the `selinux-policy-devel` package installed.
```sh
# Ensure you have the devel packages
yum install selinux-policy-devel
# Change to the directory containing the .if, .fc & .te files
cd keepassxc-selinux
make -f /usr/share/selinux/devel/Makefile keepassxc.pp
semodule -i keepassxc.pp
```

## Debugging and Troubleshooting

* If you're getting permission errors, uncomment permissive in the .te file and try again. Re-check logs for any issues. Or `semanage permissive -a keepassxc_t`
* Easy way to add in allow rules is the below command, then copy or redirect into the .te module. Rebuild and re-install:
* Don't forget to actually look at what is suggested. audit2allow will most likely go for a coarse grained permission!

```sh
ausearch -m avc,user_avc,selinux_err -ts recent | audit2allow -R
# If you get a could not open interface info [/var/lib/sepolgen/interface_info] error, install:
yum install policycoreutils-devel
```
