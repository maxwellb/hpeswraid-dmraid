# hpeswraid-dmraid

Bash shell scripts to perform discovery and
operations on HPE servers via the iLO5/Redfish
REST API.

## Origin Story

I created these scripts to address one particular
issue: I had locked myself out of Windows and
needed to perform recovery. Naturally, the
_HPE Smart Array S100i SR Gen10 Software RAID_
controller and its disks were not supported in
Linux, the environment I was using for recovery.

Rather than suffer through the rituals required to
make it through the Express OS Install again via
&ldquo;Intelligent Provisioning&rdquo;, I was
determined to mount my &ldquo;Software RAID&rdquo;
(initialized in hardware/UEFI) via, well, software.

## Current Status

Luckily for me, my boot drive was "Logical Drive 1".
I also think that the job was made easier since I
had configured it as "RAID 0" and had no striping
or parity to deal with.

As they are, these scripts are a proof-of-concept
to interface with the iLO5 management interface
via Bash shell. The solution for me, listed below,
could have in retrospect been determined by
piecing together information readily available in
the administrative interface for the storage
controller.

### _lib/functions_

Sourceable Bash script containing functions which
wrap the iLO5/Redfish REST&hairsp;_ish_ API.

[`source_env()`](./lib/functions#LC5)

> Proof-of-concept. Implementation of `.env` file
> loading in Bash. Loads `.env` files from any
> parent directory, with closer ancestors
> overriding settings defined in more distant paths.
> 
> Sanitizes input through `declare -g` and allows
> comments to begin with `#`.

[`bodyfilter()`](./lib/functions#LC19)

> Strips HTTP headers from `curl` responses,
> provided to standard input. Passes the body
> through standard output.

[`ilo_login()`](./lib/functions#LC46)

> Prompts for username and password via `read`, or
> understands `${iLO_USERNAME}` and
> `${iLO_PASSWORD}`. Opens a new session and sets:
> 
> - `${auth_header}` - the entire `X-Auth-Token`
>   header for subsequent calls. To use: `curl -H`
>   ` "${auth_header}" ... `
> - `${session_cid}` - the full (canonical) path
>   of the logged-in session. To log out, send a
>   `DELETE` to this path.

[`ilo_discovery()`](./lib/functions#LC91)

> Right now, just a smattering of wrapped API
> calls. Ideally, loads a bunch of OData ID paths
> into some sort of structured variable list.

## Initial Solution

The key information for my original problem came
down to understanding two values: `CapacityMiB`
and `StripeSizeBytes`. Here's the output from when
I ran `ilo_discovery sda`:

<pre style="background-color: black; color: silver">
<code style="user-select: none; font-family: monospace; font-weight: 700; color: lightgreen">$ </code><code
>ilo_discovery sda</code><code>
{
  "@odata.context": "/redfish/v1/$metadata#HpeSmartStorageLogicalDrive.HpeSmartStorageLogicalDrive",
  "@odata.etag": "W/\"6D9E0576\"",
  "@odata.id": "/redfish/v1/Systems/1/SmartStorage/ArrayControllers/31/LogicalDrives/1/",
  "@odata.type": "#HpeSmartStorageLogicalDrive.v2_3_0.HpeSmartStorageLogicalDrive",
  "Id": "1",
  "AccelerationMethod": "ControllerCache",
  <mark style="color: salmon;background-color: inherit">"CapacityMiB": 263168,</mark>
  "Description": "HPE Smart Storage Logical Drive View",
  "InterfaceType": "SATA",
  "LegacyBootPriority": "None",
  "Links": {
    "DataDrives": {
      "@odata.id": "/redfish/v1/Systems/1/SmartStorage/ArrayControllers/31/LogicalDrives/1/DataDrives/"
    }
  },
  "LogicalDriveEncryption": false,
  "LogicalDriveName": "LogicalDrive1",
  "LogicalDriveNumber": 1,
  "LogicalDriveStatusReasons": [
    "Ok"
  ],
  "LogicalDriveType": "Data",
  "MediaType": "HDD",
  "Name": "HpeSmartStorageLogicalDrive",
  "Raid": "1",
  "Status": {
    "Health": "OK",
    "State": "Enabled"
  },
  <mark style="color: mediumseagreen;background-color: inherit">"StripeSizeBytes": 32768,</mark>
  "VolumeUniqueIdentifier": "600508B1001C18D86340708F3FB2858A"
}
</code></pre>

Through some guess work, I determined that Linux
was seeing my two data disks for this array as
`sda` and `sdb`. Given a 512-byte sector size, the
following command established the logical volume
(hopefully).

<pre style="background-color: black; color: silver"><code style="user-select: none; font-family: monospace; font-weight: 700; color: red"># </code
>dmsetup create foobar <span style="color:gold">--table="</span> \
  0 $(( <mark style="color: salmon;background-color: inherit">263168</mark> *1024*1024 /512 )) \
  mirror core 1 $(( <mark style="color: mediumseagreen;background-color: inherit">32768</mark> /512 )) \
  2 /dev/sda 0 /dev/sdb 0<span style="color:gold">"</span>
</pre>

Among other things, I validated that the regions
of interest on the disks were probably correct by
the presence of what appeared to be a GPT
partition table. (Before running `dmsetup create`)

<pre style="background-color: black; color: silver"><code style="user-select: none; font-family: monospace; font-weight: 700; color: red"># </code
>dd if=/dev/sdb bs=512 count=1 \
  | xxd
<code style="user-select: none; font-family: monospace; font-weight: 700; color: red"># </code
>dd if=/dev/sdb bs=512 count=1 \
  skip=$(( 263168 *1024*1024 /512 <mark style="color: gold;background-color: inherit">- 1</mark>)) \
  | xxd
</pre>

In the second command, I also saw a structure that
may have been a partition table. This suggests
that the size of logical disk fills the entire
reported space, and leads right into the next
logical disk, without any metadata inter-leaved
before the start of the next disk.

I didn't have to use this knowledge this time,
since I was working with offsets of zero, and only
cared about the one logical disk. But validation
and further exploration in this area may have been
necessary if I had been working on other disks.
Before running `dmsetup` and mounting your disk, I
definitely recommend some sort of dry run in read
only mode.

### Partition-ception

One "gotcha" - After creating the `foobar` device,
I attempted to `mount /dev/mapper/foobar` straight
away. After all, I finally found the piece of the
disk that contained my **C:** drive!

Logical drives, as slices of a physical drive,
sure feel a lot like partitions. They are in the
language-sense of the word. But first, I needed to
treat my new `foobar` device as a complete hard
disk!

<pre style="background-color: black; color: silver">
<code style="user-select: none; font-family: monospace; font-weight: 700; color: red"># </code
>fdisk -l /dev/mapper/foobar
Disk /dev/mapper/foobar: 257 GiB, 275951648768 bytes, 538968064 sectors    
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: D7E6AB38-3AAB-42F7-B06C-0D0FAE7B8750

Device                    Start        End   Sectors   Size Type
/dev/mapper/foobar-part1   2048     206847    204800   100M EFI System
/dev/mapper/foobar-part2 206848     468991   4194304   128M Microsoft reserved
/dev/mapper/foobar-part3 <span style="color: skyblue">468992</span>  538968030 <span style="color: orchid">538499039</span> 256.8G Microsoft basic data
</pre>

Aha! I'll just `mount /dev/mapper/foobar-part3
/mnt` and be on my way! Not so fast. `fdisk`
likes to talk the talk, but `foobar-part3` is
not a device that actually exists yet. Once more
to `dmsetup`!

This time, I created a straightforward `linear`
device on top of the `foobar` device already
created. If I had a more complicated logical
disk setup, then I would be especially thankful
that mapping the partition winds up being easy.

<pre style="background-color: black; color: silver"><code style="user-select: none; font-family: monospace; font-weight: 700; color: red"># </code
>dmsetup create foobar-part3 <span style="color:gold">--table="</span> \
  0 <mark style="color: orchid;background-color: inherit">538499039</mark> \
  linear /dev/mapper/foobar <mark style="color: skyblue;background-color: inherit">468992</mark><span style="color:gold">"</span>
<code style="user-select: none; font-family: monospace; font-weight: 700; color: red"># </code
>mount /dev/mapper/foobar-part3 /mnt
</pre>

## To-Do

- I would like to script the entire
  discovery of logical disk boundaries.
  This should include some sort of sanity
  check in the output for expected markers,
  such as partition tables.
- Validate RAID-0 (striped) logical disks.
  This would be the `striped` device type,
  or `raid` with a `raid0`.
- Apparently (per [HPE blog][
  hpe-blog-storage]), some metadata
  and configuration information exists at
  the end of data disks. I'll have to
  check that out.

## Resources

**[dmsetup man page][dmsetup]**

**[Gentoo Linux Wiki][gentoo-wiki]**

**[HPE iLO5 API Reference][hpe-apidoc]**


## Comments?

Please feel free to create a
[new issue][issue-create] and share any
information or feedback you might have.
Thank you for reading this far.

[dmsetup]: https://man7.org/linux/man-pages/man8/dmsetup.8.html#TABLE_FORMAT
[gentoo-wiki]: https://wiki.gentoo.org/wiki/Device-mapper#Mirror_and_RAID1
[hpe-apidoc]: https://hewlettpackard.github.io/ilo-rest-api-docs/ilo5/
[hpe-blog-storage]: https://developer.hpe.com/blog/storage-management-with-redfish
[issue-create]: https://github.com/maxwellb/hpeswraid-dmraid/issues/new/choose
