# IBM Elastic Storage Server Upgrade Notes (v3.5 to v4.0)

This page describes upgrade procedure for IBM Elastic Storage Server (ESS) 4.0 for Power. The official documentation can be found [here](https://www-01.ibm.com/support/knowledgecenter/SSYSP8_4.0.0/sts40_welcome.html)
[ESS Quick Deployment Guide](https://www-01.ibm.com/support/knowledgecenter/SSYSP8_4.0.0/c2785801.pdf?lang=en-us) doesn't have a lot of context so these notes may be helpful to some users, but are in no way a replacement for the official documentation.

This document **should not be used instead of the official ESS Quick Deployment Guide 4.0**. 
They are mere technical notes. If in doubt, contact IBM Support or seek advice on the IBM Spectrum Scale discussion forum.

## Overview of the Procedure

We'll follow the section *Upgrading from ESS 3.x.y* that begins on page 5 of this document. The procedure has three main and several follow-up steps:

* Upgrade the Management Node (commonly named `ems1`; we'll use the "default" hostnames in this note)
* Upgrade all I/O nodes (`gss[1,2]` in each I/O pair)
* Upgrade storage (JBOD f/w and disk drive firmware)
* Check the installed software and system health
* Upgrade the GUI
* Complete GPFS-related tasks

We begin with two files, `gss_install-4.0.0_ppc64_*_20160128T230543Z.tgz` (the `*` denotes Spectrum Scale version and can be express, standard, or advanced) and `gpfs.gss.firmware-4.2.0-1.ppc64.rpm` which contains latest firmware upgrades.

* We follow the three steps in *Install the management server software* (page 1) to extract the archive and deploy the installer and install  (`rpm -i`) the RPM on the management node.
* Then we move on to page 5 to perform steps from the bulleted list above. 

**NOTE**: be patient when running these commands. If you want to see what's going on, open another shell and watch `top` output, or tail the right log (e.g. `tail -f /var/log/xcat/xcat.log`).
The node upgrade and OFED upgrade command can take 10-15 minutes to run.

## Upgrade the Management Node

We first want to upgrade the management node, `ems1`, as described on page 5 of the Guide.

```
cd /opt/ibm/gss/install
installer/gssinstall -m manifest -u
```

NOTE: do not run `gssinstall` from the installer subdirectory.

Next, stop the GUI on `ems1`: `systemctl stop gpfsgui`.

You can make a backup of these config files below, but as we shall see later, for ESS 3.5 to 4.0, this doesn't appear to be necessary:

```
cp /opt/IBM/zimon/ZIMonSensors.cfg /tmp
cp /opt/IBM/zimon/ZIMonCollector.cfg /tmp
```

At this point the document suggests to shutdown Spectrum Scale on this node. It is probably better to disable it first, and shutdown later.
Remember, we're on `ems1`.

```
mmchconfig autoload=no -N ems1
mmshutdown
```

We need to **remember** to re-enable it later after this section is completed.

Now let's update ESS software on `ems1` (run Step 5, because Step 4 would fail):

`updatenode ems1 -P gss_updatenode`

Reboot, and then from `ems1` run these three commands from Step 4.

```
/opt/ibm/gss/tools/samples/gssupg400.sh -b ems1
/opt/ibm/gss/tools/samples/gssupg400.sh -c ems1
/opt/ibm/gss/tools/samples/gssupg400.sh -p ems1
```

Now the cluster has the latest information about software on `ems1`. We need to update two more things; the network stack and firmware of the RAID card used for internal RAID on `ems1`.
This Step 7 usually fails because the command doesn't know what to upgrade, which is why we will first run the suggested workaround from *Appendix. Known Issues* (page 9). 

```
updatenode ems1 -P gss_ofed
mmlsfirmware --type host-adapter -N ems1
updatenode ems1 -P gss_ipraid
```

NOTES: 

* The manual contains a mistake here - it suggests that the `mmlsfirmware` command should be executed against `gss_ppc64`, which is wrong.
`gss_ppc64` is a node class (see output of `mmlsnodeclass`) and denotes all ESS I/O nodes (in our case, `gssio1` and `gssio2`, while `ems1` does not belong to this class).
* When you move on to repeat this upgrade procedure on the I/O nodes, you could use `gss_ppc64` to refer to the both of them, or individual host names (which is going to run slightly faster and may be better since you will be upgrading them in a rolling fashion anyway).
* If the file `/usr/lpp/mmfs/updates/gnrFirmware.tar.untarred` exist from a previous upgrade, the `gss_ipraid` update might fail with message about "FW rpm not found, trying to untar Firmware.tar FW rpm not found. Exiting...". If that happens, remove the `/usr/lpp/mmfs/updates/gnrFirmware.tar.untarred` and re-run the updatenode command for gss_ipraid.

Assuming everything goes well, we can reboot `ems1`, enable GPFS on it and start service: 

```
mmchconfig autoload=yes -N ems1
mmstartup -N ems1
```

You can verify new RPMs have been installed (`rpm -qa | grep gpfs`, for example), but we assume that we wouldn't continue if any of the major steps had serious errors.

## Upgrade all I/O nodes (`gssio[1,2]` in the first I/O pair)

We need to repeat this procedure (upgrade I/O nodes) twice: once on `gssio1` and once on `gssio2`. This can't be executed in parallel, because we need at least one of the nodes to serve data.
This section has 11 steps and we'll first execute steps 1 to 11 on `gssio1`, and then on `gssio2`.

### What's the difference between upgrading ESS s/w on ESS management and I/O nodes?

This is just for the context: the management node (or nodes, if you hvve more than one) normally doesn't serve data. Other than that, the procedure is almost the same.
To account for that, we'll repeat the same procedure as for the management node, but with some additional steps that account for this difference.

We won't repeat all 11 steps here - it should be enough to clarify the concept:

* An I/O node pair likely contains a filesystem manager(s) and each of the I/O nodes is the preferred "owner" of "left" or "right" Recovery Group of disks in JBOD enclosures.
* When we upgrade one of the two nodes, we need to preemptively unassign any roles to it, to make the upgrade smoother in terms of failover and failback.

NOTE: The Guide mentions `CurrentServer` (and variants thereof), which is the server we're currently working on. This is to prevent accidental copy-and-paste that could be executed against the wrong server, so we need to pay attention to the hostnames.

All right, so we'll start with `gssio1` and to accmmodate for the first the difference (filesystem manager) vs. `ems1`, we need to remove its manager role during this stage.

```
mmlsmgr
```

You may see output like this which shows that `gssio1` manages our filesystem `fs1`:

```
# mmlsmgr
file system      manager node
---------------- ------------------
fs1              192.168.45.22 (gssio1)
fs2              192.168.45.22 (gssio2)
```

For all such filesystems where `$CURRENT_IO_SERVER` is the manager, reassign the role to the other node (pay attention to node and filesystem names!).

```
mmchmgr fs1 gssio2
```

After this is done, we need to account for the second difference compared to `ems1`, namely the I/O nodes' ownership of Recovery Groups.

```
mmlsrecoverygroup
```

We expect to see something like this:

```
# mmlsrecoverygroup

                     declustered
                     arrays with
 recovery group        vdisks     vdisks  servers
 ------------------  -----------  ------  -------
 rg_gssio1                     3       9  gssio1.gpfs.net,gssio2.gpfs.net
 rg_gssio2                     3       9  gssio2.gpfs.net,gssio1.gpfs.net
```

If you deal with a large cluster you may want to keep this output for later when you need to revert these temporary changes.

We need to demote the "current node" (the first time we run this `$CURRENT_IO_SERVER` is `gssio1`) to Secondary role, so let's revert the preferred server order for the group owned by `gssio1`:

```
mmchrecoverygroup rg_gssio1 --servers gssio2,gssio1
```

This will take a minute. Repeat it for all RG's where `$CURRENT_IO_SERVER` is Primary.

Later, when we move on to `gssio2`, we'll do the opposite (demote `gssio2` from Primary to Secondary in all RG's). And when we are done with this section, we need to revert the role back to the default we had when we first executed `mmlsrecoverygroup` above. Again, we should not simply copy and paste without paying attention to RG and server names. Even in that case nothing catastrophic should happen, but longer I/O server failovers with more performance impact could be possible.

Starting with Step 3 in this section, the procedure is almost identical to `ems1`, but remember to use the correct I/O server name!

Follow Steps 3 to 7 in this section while paying attention to our suggestions and workarounds outlined above for `ems1`.
For example, we suggest to not only shutdown, but also disable GPFS service on the nodes, etc.

This can be beneficial in Step 8 (`mmchfirmware --type host-adapter`), for example, because Step 8 **won't work** if GPFS service is running on the node, and it will be running if you didn't disable it and rebooted as instructed in the official guide's Step 7. (Normally GPFS service is set to autostart (`autoload=yes`), so there may be exceptions.)

NOTE: Unlike with `ems1`, for I/O nodes only `gssupg400.sh -s $CURRENT_IO_SERVER` needs to be executed in this section.

Step 10 contains a typo, it's not `gssinstallcheck -N CurrentIo server --phy-mapping`, but `gssinstallcheck -N $CURRENT_IO_SERVER --phy-mapping` which means you'd have something like `gssinstallcheck -N gssio1 --phy-mapping` the first time you go through this section.

After Step 11, remember to set `gssio1` to `autoload=yes`, as we did with `ems1`. 

---------------------------

Then repeat this section related to I/O nodes one more time, but with `gssio2`, and after you're done verify:

* The both I/O servers are set to `autoload=yes` (if you want GPFS to auto-start)
* GPFS service is running (`mmgetstate -aLs`) on all the nodes which have been successfully upgraded
* Recovery Groups are balanced and filesystems are mounted (`mmlsrecoverygroup`)
* Filesystem managers are "distributed" (a best practice, if you have more than one filesystem; `mmlsmgr`) or at least the way they were before the upgrade

## Update the enclosure and drive firmware

We also need to upgrade I/O components that aren't part of the two I/O nodes, so from `ems1` we'd now upgrade storage firmware (GPFS service can be running):

```
mmchfirmware --type storage-enclosure -N gss_ppc64
mmchfirmware --type drive -N gss_ppc64
```

You can and need to run this only once - as we mentioned above, `gss_ppc64` is a Spectrum Scale node class that contains all I/O nodes so the updates will run on the both `gssio1` and `gssio2` (and other servers if you have more units in the cluster).
If you need to upgrade just one pair, refer to the individual nodes in that pair (i.e. in ESS unit 3, they could be named `gssio[5,6]`.

For reference, here's a list of node classes on our ESS cluster:

```
# mmlsnodeclass
Node Class Name       Members
--------------------- -----------------------------------------------------------
gss_ppc64             gssio1.gpfs.net,gssio2.gpfs.net
ems                   ems1.gpfs.net
GUI_RG_SERVERS        gssio1.gpfs.net,gssio2.gpfs.net
GUI_SERVERS           ems1.gpfs.net

```

## Check the installed software and system health

Check the state of each node and the Spectrum Scale Native RAID:

```
gssinstallcheck -N ems1
gssinstallcheck -N gssio1
gssinstallcheck -N gssio2
gnrhealthcheck
```

You may also want to once again verify that GPFS on all the nodes is set to `autoload=yes` if that's what we want.

## Upgrade the GUI

Execute the upgrade procedure for ESS 3.5 (not for ESS 3.0!) on `ems1`, where the GUI runs and where you made a backup of those configuration files.
As you can see on page 8, it is not necessary to restore the config files when upgrading from ESS v3.5.

You should be able to access the new ESS GUI at https://EMS1/login (User: `admin`/ Password: `admin001`).

## Completing GPFS-related Tasks

If we want to take advantage of the latest filesystem and other features in Spectrum Scale 4.2, we may (or not) want to execute these commands.
You need to be aware that older versions of Scale may not be able to participate in the cluster after this, so we may want to read the documentation for more details on this.

NOTE: Generally speaking if you have older clients (older than Spectrum Scale 4.2), you may not want to execute these before reading Scale documentation.

```
mmchconfig release=LATEST
mmchfs fs1 -V full
```

Additional details about these commands can be found [on this page](https://www-01.ibm.com/support/knowledgecenter/STXKQY_4.2.0/com.ibm.spectrum.scale.v4r2.ins.doc/bl1ins_migratl.htm?lang=en) and elsewhere in Spectrum Scale 4.2 documentation.
