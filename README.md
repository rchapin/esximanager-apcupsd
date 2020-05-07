# apcupsd Integration with the esximanager shutdown utility

This project integrates apcupsd with esximanager shutdown utility to cleanly shutdown all vms and the esxihost during a power outage.

One of the first things that I wanted to get sorted out after getting an esxi server up and running was integration with my ups so that I could cleanly shutdown vms and the esxi host itself when I lost power longer then my UPS could handle it.  I had existing APC UPSes on hand and was familiar with apcupsd.  After some searching around it seemed that the route to directly integrate the esxi host with a UPS was frought with some brittle complexities so I wrote [esximanager](https://github.com/rchapin/esximanager) and then this project to integrate it with apcupsd.

## Architecture

It is quite simple.

1. Install and configure this utility and esximanager on a bare metal host (or one on which you can pass through the USB or serial connection) on which you have apcupsd installed and configured.

2. apcupsd executes the scripts defined in this project when the event is raised by the UPS device to indicate that the UPS is out of power and that the machines connected to it need to power off.

3. The script then executes the ```esximanager shutdown``` call to the exsi host to shudown all of the vms and the esxi host blocking until that call returns.

3. The apcupsd daemon then finishes it's job of shutting down the host on which this utility is installed.

## Installation

The first version is a manual install.  If anyone actually uses this other than myself, just get in touch with me and I can automate it sooner vs. later.

1.  Install and configure [apcupsd](http://www.apcupsd.org/)on a host that you will connect to the UPS.

2.  See [esximanager](https://github.com/rchapin/esximanager) for details on installing and configuring it.

3.  Create config and log dirs

```
mkdir -p /etc/esximanager-apcupsd
mkdir -p /var/log/esximanager-apcupsd
```

4.  Copy the config template from the etc dir to ```/etc/esximanager-apcupsd``` and update based on your installation of esximanager.

5.  Copy the scripts in the ```apcupsd``` dir to ```/etc/apcupsd``` and set permissions to ```755``` and owner to ```root:```.

6.  Set the DRYRUN config to 1 and run as follows to test.

### For SELinux Enabled Systems

You will need to make the following config updates and install selinux rules.

1. Ensure you have the following packages installed (names may vary on systems other than RHEL/CentOS)

```
yum install -y checkpolicy policycoreutils policycoreutils-python
```

2. Set the following booleans

```
setsebool -P authlogin_nsswitch_use_ldap on
setsebool -P nis_enabled on
```

3. Compile the Type Enforcement (te) file in the selinux dir to enable the esximanager program the required accesses.
```
cp selinux/esximanager-apcupsd.te /var/tmp/esximanager-apcupsd.te
```

4. Check and compile the mod file (assuming the .te file in in /var/tmp)
```
checkmodule -M -m -o /var/tmp/esximanager-apcupsd.mod /var/tmp/esximanager-apcupsd.te
```

5. Create the SELinux Policy Module Packet (pp) File From the .mod file
```
semodule_package -o /var/tmp/esximanager-apcupsd.pp -m /var/tmp/esximanager-apcupsd.mod
```

6. Install the SELinux Policy Module
```
semodule -i /var/tmp/esximanager-apcupsd.pp
```

7. Set the selinux policies on any of the ```/etc/apcupsd``` scripts that you installed as follows

```
chcon -u system_u -t bin_t /etc/apcupsd/<file>
chmod 755 /etc/apcupsd/<file>
```
