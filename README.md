# check_lsi_raid - Nagios/Icinga plugin to check LSI RAID Controllers

## General

* The LSI RAID Monitoring Plugin is a Nagios/Icinga to check the LSI RAID Controller for warnings or critical errors.
* It is written in Perl and uses the storage command line tool storcli to interact with the RAID Controller.

## Current Version

* The current version of the check_lsi_raid plugin is available at the Thomas-Krenn github account:
  https://github.com/thomas-krenn/check_lsi_raid

## Further information

* A wiki article can be found at:
  http://www.thomas-krenn.com/de/wiki/LSI_RAID_Monitoring_Plugin or
  http://www.thomas-krenn.com/en/wiki/LSI_RAID_Monitoring_Plugin
* Windows user should read:
  http://www.thomas-krenn.com/de/wiki/LSI_RAID_Monitoring_Plugin_unter_Windows_Server_2012_einrichten

## Functionalities

* Controller Status
  * ROC temperature
  * Degraded drives
  * Offline drives
  * Critical disks
  * Failed Disks
  * Memory correctable errors
  * Memory uncorrectable errors

* Logical Device Status
  * State
  * Consistency
  * Initialization

* Physical Device Status
  * State
  * Shield counter
  * Media, Other, BBM, Predictive error counts
  * SMART alert flag
  * Drive temperature
  * Initialization
  * Rebuild

* If BBU or CV is present (unless disabled)

* BBU
  * Battery State
  * Temperature
  * Firmware temperature
  * Voltage
  * I2C errors
  * Replacement required
  * Capacity low
  * Is about to fail
  * GasGauge discharged
  * GasGauge over temperature
  * GasGauge over charged

* CV
  * State
  * Temperature
  * Replacement required

## Used storcli commands

* storcli /c0 /bbu show
* storcli /c0 /bbu show status
* storcli /c0 /cv show
* storcli /c0 /cv show status
* storcli adpallinfo a0
* storcli /c0 show time
* storcli /c0/vall show all
* storcli /c0/vall show init
* storcli /c0/eall/sall
* storcli /c0/eall/sall
* storcli /c0/eall/sall

## Installation Requirements

* libfile-which-perl
* storcli

## Requirements for Icinga/Nagios

* On the system to be monitored:
  * check_lsi_raid plugin
  * storcli
  * sudoers entry for user nagios and storcli
   (example: nagios ALL=(root) NOPASSWD:/usr/sbin/storcli)
* NRPE (optional): command definition
* On the Icinga-server:
  * command definition
  * service definition

## Installation hints for CentOS

* It is possible that the NRPE daemon runs as user 'nrpe'. If that's the case
  you have to change the user in the sudoers entry.
* It may be necessary to specify the perl interpretor explicitly in the NRPE
  command definition, e.g.:
  command[check_lsi_raid]=/usr/lib/nagios/plugins/check_lsi_raid -C 0 -p /usr/sbin/storcli

## Parameter usage (example)

./check_lsi_raid -Tw 40 -Tc 50 -LD 0,1 -PD 1 -b 0

## License

Copyright (C) 2013-2018 Thomas-Krenn.AG

This program is free software; you can redistribute it and/or modify it under
the terms of the GNU General Public License as published by the Free Software
Foundation; either version 3 of the License, or (at your option) any later version.

This program is distributed in the hope that it will be useful, but WITHOUT
ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS
FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details.

You should have received a copy of the GNU General Public License along with
this program; if not, see <http://www.gnu.org/licenses/>.
