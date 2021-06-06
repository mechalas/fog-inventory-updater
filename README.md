# Fog Inventory Updater

This is a Python command-line utility that can create, update, and edit inventory data for hosts on a FOG server.

FOG only updates the inventory data for a host when it's PXE explicitly booted to FOS to run an inventory task. This script provides a method of updating inventory without having to do that, allowing you to:

* Update inventory on a running host on demand
* Update an arbitrary host's inventory, provided it has been collected in advance or manually provided

It follows the same methodology as FOS in collecting invtentory (with one exception). Specifically it:

1. Executes `dmidecode -s string` for each of the strings supported by the command. See dmicecode(1) for a full list. These provide information about the system, CPU, motherboard, chassis, and BIOS.
1. Executes `dmidecode -t 4` to obtain the CPU's base and maximum frequency
1. Executes `dmidecode -t 3` to obtain the system type
1. Reads total system memory from /proc/meminfo
1. Executes `hdparm -i` to get the model, serial number, and firmware of the primary disk device

Unlike FOS, the primary disk device is assumed to be the one where / is mounted. This methodology doesn't work well on logical volumes (e.g. lvm and RAID volumes).

**This script must be run as root when collecting inventory data on a system.** This is necessary for running `dmidecode` and `hdparm`, but the script lowers its priviledges to the user nobody and only elevates them back to root when it needs to execute then (and then immediately lowers them again).

## Prerequisites

### Python dependancies

fog-inventory requires python v3 with urllib3 which can be installed using pip:

`python -m pip install urllib3`

Alternatively, you can use your Linux distribution's package manager.

### Ubuntu/Debian

`apt install python3-urllib3`

### CentOS/Redhat

`dnf install python3-urllib3`

### FOG API system

The API system must be enabled on your FOG server (FOG Configuration -> FOG Settings -> API System), and you must have a user API token (enable the target user for the API system in Users -> _username_ -> API Settings).

**To avoid exposing API tokens, the use of TLS on the FOG server (https:// URLs instead of http://) is strongly recommended.**

## Usage

```
usage: fog-inventory [-h] [-A TOKEN] [-U TOKEN] [-c] [-f FILE] [-n] [-u URL]
                     [-x] [-p USER] [-o1 INFO] [-o2 INFO]
                     [-H HOSTNAME | -i HOSTID]

optional arguments:
  -h, --help            show this help message and exit
  -A TOKEN, --fog-api-token TOKEN
                        The FOG API token
  -U TOKEN, --user-api-token TOKEN
                        The user API token
  -c, --fetch-current   Fetch the current inventory data for the host rather
                        than updating it.
  -f FILE, --file FILE  Update the inventory using the JSON data from FILE.
                        Useful for updating arbitrary systems, individual
                        inventory fields, or systems where root access is not
                        available.
  -n, --dryrun          Show the inventory data but don't submit it to the FOG
                        server.
  -u URL, --fog-server-url URL
                        The FOG server name
  -x, --inventory-only  Only collect and show the new inventory data. Do not
                        attempt to find the host in FOG or submit the data.
                        You do not need to provide API tokens with this
                        option.
  -p USER, --primary-user USER
                        Set the "Primary User" field in the inventory data
  -o1 INFO, --other1 INFO
                        Set the "Other Tag #1" field in the inventory data
  -o2 INFO, --other2 INFO
                        Set the "Other Tag #2" field in the inventory data
  -H HOSTNAME, --hostname HOSTNAME
                        Use the hostname HOSTNAME instead of the local system
                        name.
  -i HOSTID, --host-id HOSTID
                        Use the host with id HOSTID instead of the local
                        system name.
```

fog-inventory has four primary modes of operation.

### Collect inventory data and upload it to FOG

This is the default operating mode. The script will gather inventory data on the current host, and attempt to upload it to FOG. It assumes that the hostname of the local system matches host's name in the FOG database. If for some reason these differ, the host's name can be overridden with the `-H` option, or a FOG host ID can be provided with `-i`. If the host can't be found in FOG, it will exit without taking action and print an error. If the hosts exists but does not yet have an inventory record, a new one will be created.

The primary user and "other" tags can be set with `-p`, `-o1` and `-o2`, respectively. These are the only FOG inventory items that cannot be automatically determined.

The `-n` option performs a dry run: the host is validated and inventory data is collected, but it is not uploaded to FOG. Instead, the inventory data that _would_ be sent is printed to stdout.

The location of the FOG server is determined from the default FOG client settings in /opt/fog-service/settings.json, and can be overridden with `-u`.

This mode requires both the FOG API token and a user API token, and the script must be run as root to collect the complete inventory data.

### Fetch the current inventory data for a host

When the `-c` option is supplied, the inventory information for the local system is fetched from the FOG server and printed to stdout. It assumes the hostname of th local system matches the host name in the FOG database. You can also fetch data for an arbitrary host by specifying a host name with `-H` or a FOG host ID with `-i`.  If the host can't be found in FOG, it will exit and print an error.

This mode requires both the FOG API token and a user API token. The script does not need to be run as root.

### Upload inventory data from a file

In this mode, inventory data is read from the file specified by the `-f` option, and is uploaded to the FOG server. It's assumed that you are updating the inventory for the local machine, and that its hostname matches the host name in the FOG database. You can updating inventory for an arbitrary machine by specifying a hostname with `-H` or a FOG host ID with `-i`. If the host can't be found in FOG, it will exit without taking action and print an error. If the hosts exists but does not yet have an inventory record, a new one will be created.

The input file must be in JSON format, and the field data must be strings. See the output from the `-c` or `-x` option for the format. _Only the fields to be edited need to be provided._

The allowed fields are:

| Field | Description |
|-------|-------------|
| biosdate | The date the BIOS was published by the BIOS manufacturer |
| biosvendor | The BIOS manufacturer/vendor name |
| biosversion | The BIOS version string |
| caseasset | The asset number for the chassis/case |
| caseman | The manufacturer of the chassis/case |
| caseserial | The chassis/case serial number | 
| cpucurrent | The CPU's base speed |
| cpuman | The CPU manufactuer |
| cpumax | The CPU's maximum speed, for CPU's where the CPU speed can exceed the base speed (e.g. Intel® Turbo Boost Technology or AMD AMD Turbo Core Technology). |
| cpuversion | The CPU model name |
| mbasset | The motherboard's asset number |
| mbman | The motherboard manufacturer |
| mbproductname | The motherboard's product or model name |
| mbserial | The motherboard's serial number |
| mbversion | The motherboard version string |
| mem | The total system memory |
| other1 | The data for "Other Tag #1" |
| other2 | The date for "Other Tag #2" |
| primaryUser | The primary user of the system |
| sysman | The system manufacuter |
| sysproduct | The system's product or model name |
| sysserial | The system's serial number |
| systype | The system type |
| sysuid | The UUID for the system |

Not all systems have data for every field.

This mode requires both the FOG API token and a user API token. The script does not need to be run as root.

### Collect and dump inventory info on the local system

When the `-x` paremeter is supplied, fog-inventory collects the FOG inventory data and prints it to stdout.

This mode does not require an API tokens, but the script must be run as root.

