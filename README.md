# keepassxc-selinux

This policy is designed to work on Fedora 29 confine keepassxc-2.2.2-1. Since it concerns passwords, use at your own risk!. This is more an exercise to containing a user application. The Flatpak sandboxing might be more practical.

The primary goals of this policy module is:
* Deny network access to the program. (Within reason. DBUS is denied but keepassxc could potentially talk through that if enabled via the boolean)
* Isolate the password file (.kdbx) from other programs on the system (see private-db branch for this feature)


### Current caveats:

* Access to dbus is disabled by default. But can be enabled through the boolean. This is intentional to try and contain the program as much as possible. Keepassxc needs dbus to monitor for events like lid closing and logout to lock the database. Nothing too important but that might change in the future with development
* Policy is designed with the assumption that the user is using unconfined_u (which is standard on Fedora out-of-the-box). Staff_u is also supported, remove unconfined from the policy if desired since it goes against convention.
* Moving the .kdbx away from the home directory needs to have its private label `keepassxc_db_t` added
* Auto-type is hit and miss
* Challenge-Response with Yubikey works. Untested with other brands
* Importing from keepass(original) is untested


### Booleans needed for specific functions:
```
Allow keepassxc to use dbus: keepassxc_can_use_dbus --> off
Challenge-Response requires: keepassxc_can_access_usb_devices --> off
Reading a password file on a USB drive requires: keepassxc_can_access_usb_devices --> off
```

## Installation
```sh
# Clone the repo
git clone https://github.com/georou/keepassxc-selinux.git

# Copy relevant .if interface file to /usr/share/selinux/devel/include to expose them when building and for future modules
install -Dp -m 0664 -o root -g root keepassxc.if /usr/share/selinux/devel/include/myapplications/keepassxc.if

# Compile the selinux module (see below)

# Install the SELinux policy module. Compile it before hand to ensure proper compatibility (see below)
semodule -i keepassxc.pp

# Restore all the correct context labels
restorecon -RvF /usr/bin/{keepassxc,keepassxc-cli} $HOME/.config/keepassxc

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

## Compatibility Notes
Built on Fedora 29 Gnome at the time with:
```
selinux-policy-3.13.1-260.20.fc26.noarch
selinux-policy-targeted-3.13.1-260.20.fc26.noarch
selinux-policy-devel-3.13.1-260.20.fc26.noarch
```
