#!/usr/bin/perl -w
# =============================================================================
# check_lsi_raid: Nagios/Icinga plugin to check LSI Raid Controller status
# -----------------------------------------------------------------------------
# Created as part of a semester project at the University of Applied Sciences
# Hagenberg (http://www.fh-ooe.at/en/hagenberg-campus/)
#
# Copyright (c) 2013-2016:
#   Georg Schoenberger  (gschoenberger@thomas-krenn.com)
#   Grubhofer Martin    (s1110239013@students.fh-hagenberg.at)
#   Scheipner Alexander (s1110239032@students.fh-hagenberg.at)
#   Werner Sebastian    (s1110239038@students.fh-hagenberg.at)
#   Jonas Meurer        (jmeurer@inet.de)
#
# This program is free software; you can redistribute it and/or modify it under
# the terms of the GNU General Public License as published by the Free Software
# Foundation; either version 3 of the License, or (at your option) any later
# version.
#
# This program is distributed in the hope that it will be useful, but WITHOUT
# ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS
# FOR A PARTICULAR PURPOSE. See the GNU General Public License for more
# details.
#
# You should have received a copy of the GNU General Public License along with
# this program; if not, see <http://www.gnu.org/licenses/>.
# ==============================================================================
use strict;
use warnings;
use Getopt::Long qw(:config no_ignore_case);
use File::Which;

our $VERBOSITY = 0;
our $VERSION = "2.5";
our $NAME = "check_lsi_raid: Nagios/Icinga plugin to check LSI Raid Controller status";
our $C_TEMP_WARNING = 85;
our $C_TEMP_CRITICAL = 95;
our $PD_TEMP_WARNING = 40;
our $PD_TEMP_CRITICAL = 45;
our $BBU_TEMP_WARNING = 50;
our $BBU_TEMP_CRITICAL = 60;
our $CV_TEMP_WARNING = 70;
our $CV_TEMP_CRITICAL = 85;
our ($IGNERR_M, $IGNERR_O, $IGNERR_P, $IGNERR_S, $IGNERR_B) = (0, 0, 0, 0, 0);
our $NOENCLOSURES = 0;
our $CONTROLLER = 0;

use constant {
	STATE_OK => 0,
	STATE_WARNING => 1,
	STATE_CRITICAL => 2,
	STATE_UNKNOWN => 3,
 };

# Header maps to parse logical and physical devices
our $LDMAP;
our @map_a = ('DG/VD','TYPE','State','Access','Consist','Cache','sCC','Size');
our @map_cc_a = ('DG/VD','TYPE','State','Access','Consist','Cache','Cac','sCC','Size');
our @pdmap_a = ('EID:Slt','DID','State','DG','Size','Intf','Med','SED','PI','SeSz','Model','Sp');

# Print command line usage to stdout.
sub displayUsage {
	print "Usage: \n";
	print "  [ -h | --help ]
    Display this help page\n";
	print "  [ -v | -vv | -vvv | --verbose ]
    Sets the verbosity level.
    No -v is the normal single line output for Nagios/Icinga, -v is a
    more detailed version but still usable in Nagios. -vv is a
    multiline output for debugging configuration errors or more
    detailed information. -vvv is for plugin problem diagnosis.
    For further information please visit:
        http://nagiosplug.sourceforge.net/developer-guidelines.html#AEN39\n";
	print "  [ -V --version ]
    Displays the plugin and, if available, the version of StorCLI.\n";
	print "  [ -C <num> | --controller <num> ]
    Specifies a controller number, defaults to 0.\n";
	print "  [ -EID <ids> | --enclosure <ids> ]
    Specifies one or more enclosure numbers, per default all enclosures. Takes either
    an integer as additional argument or a commaseperated list,
    e.g. '0,1,2'. With --noenclosures enclosures can be disabled.\n";
	print "  [ -LD <ids> | --logicaldevice <ids>]
    Specifies one or more logical devices, defaults to all. Takes either an
    integer as additional argument or a comma seperated list e.g. '0,1,2'.\n";
	print "  [ -PD <ids> | --physicaldevice <ids> ]
    Specifies one or more physical devices, defaults to all. Takes either an
    integer as additional argument or a comma seperated list e.g. '0,1,2'.\n";
	print "  [ -Tw <temp> | --temperature-warn <temp> ]
    Specifies the RAID controller temperature warning threshold, the default
    threshold is ${C_TEMP_WARNING}C.\n";
	print "  [ -Tc <temp> | --temperature-critical <temp> ]
    Specifies the RAID controller temperature critical threshold, the default
    threshold is ${C_TEMP_CRITICAL}C.\n";
	print "  [ -PDTw <temp> | --physicaldevicetemperature-warn <temp> ]
    Specifies the disk temperature warning threshold, the default threshold
    is ${PD_TEMP_WARNING}C.\n";
	print "  [ -PDTc <temp> | --physicaldevicetemperature-critical <temp> ]
    Specifies the disk temperature critical threshold, the default threshold
    is ${PD_TEMP_CRITICAL}C.\n";
	print "  [ -BBUTw <temp> | --bbutemperature-warning <temp> ]
    Specifies the BBU temperature warning threshold, default threshold
    is ${BBU_TEMP_WARNING}C.\n";
	print "  [ -BBUTc <temp> | --bbutemperature-critical <temp> ]
    Specifies the BBU temperature critical threshold, default threshold
    is ${BBU_TEMP_CRITICAL}C.\n";
    print "  [ -CVTw <temp> | --cvtemperature-warning <temp> ]
    Specifies the CV temperature warning threshold, default threshold
    is ${CV_TEMP_WARNING}C.\n";
    print "  [ -CVTc <temp> | --cvtemperature-critical <temp> ]
    Specifies the CV temperature critical threshold, default threshold
    is ${CV_TEMP_CRITICAL}C.\n";
	print "  [ -Im <count> | --ignore-media-errors <count> ]
    Specifies the warning threshold for media errors per disk, the default
    threshold is $IGNERR_M.\n";
	print "  [ -Io <count> | --ignore-other-errors <count> ]
    Specifies the warning threshold for media errors per disk, the default
    threshold is $IGNERR_O.\n";
	print "  [ -Ip <count> | --ignore-predictive-fail-count <count> ]
    Specifies the warning threshold for media errors per disk, the default
    threshold is $IGNERR_P.\n";
	print "  [ -Is <count> | --ignore-shield-counter <count> ]
    Specifies the warning threshold for media errors per disk, the default
    threshold is $IGNERR_S.\n";
    print "  [ -Ib <count> | --ignore-bbm-counter <count> ]
    Specifies the warning threshold for bbm errors per disk, the default
    threshold is $IGNERR_B.\n";
	print "  [ -p <path> | --path <path>]
    Specifies the path to StorCLI, per default uses the tool 'which' to get
    the StorCLI path.\n";
	print "  [ -b <0/1> | --BBU <0/1> ]
    Check if a BBU or a CacheVault module is present. One must be present unless
    '-b 0' is defined. This ensures that for a given controller a BBU/CV must be
    present per default.\n";
	print "  [ --noenclosures <0/1> ]
    Specifies if enclosures are present or not. 0 means enclosures are
    present (default), 1 states no enclosures are used (no 'eall' in
    storcli commands).\n";
    print "  [ --nosudo ]
    Turn off using sudo.\n";
    print "  [ --nocleanlogs ]
    Do not clean storcli logs after running storcli commands.\n";
}

# Displays a short Help text for the user
sub displayHelp {
	print $NAME."\n";
	print "Pulgin version: " . $VERSION ."\n";
	print "Copyright (C) 2013-2015 Thomas-Krenn.AG\n";
	print "Current updates available at
    https://github.com/thomas-krenn/check_lsi_raid.git\n";
	print "This Nagios/Icinga Plugin checks LSI RAID controllers for controller,
physical device, logical device, BBU and CV warnings and errors.\n";
	print "In order for this plugin to work properly you need to add the nagios
user to your sudoers file (or create a new one in /etc/sudoers.d/).\n";
	displayUsage();
	print "Further information about this plugin can be found at:
    http://www.thomas-krenn.com/de/wiki/LSI_RAID_Monitoring_Plugin and
    http://www.thomas-krenn.com/de/wiki/LSI_RAID_Monitoring_Plugin
Please send an email to the tk-monitoring plugin-user mailing list:
    tk-monitoring-plugins-user\@lists.thomas-krenn.com
if you have questions regarding use of this software, to submit patches, or
suggest improvements.
Example usage:
* check_lsi_raid -p /opt/MegaRAID/storcli/storcli64
* check_lsi_raid -p /opt/MegaRAID/storcli/storcli64 -C 1\n";
	exit(STATE_UNKNOWN);
}

# Prints the name and the version of check_lsi_raid. If storcli is available,
# the version of it is printed also.
# @param storcli The path to storcli command utility
sub displayVersion {
	my $storcli = shift;
	if(defined($storcli)){
		my @storcliVersion = `$storcli -v`;
		foreach my $line (@storcliVersion){
			if($line =~ /^\s+StorCli.*/) {
				$line =~ s/^\s+|\s+$//g;
				print $line;
			}
		}
		print "\n";
	}
	exit(STATE_OK);
}

# Checks if a storcli call was successfull, i.e. if the line 'Status = Sucess'
# is present in the command output.
# @param output The output of the storcli command as array
# @return 1 on success, 0 if not
sub checkCommandStatus{
	my @output = @{(shift)};
	foreach my $line (@output){
		if($line =~ /^Status/){
			if($line eq "Status = Success\n"){
				return 1;
			}
			elsif (grep { /Failure\s+46/i } @output){
				# Return 46 means a drive is not attached, this is a valid failure
				return 1;
			}
			else{
				return 0;
			}
		}
	}
}

# Shows the time the controller is using. Can be used to check if the
# controller number is a correct one.
# @param storcli The path to storcli command utility, followed by the controller
# number, e.g. 'storcli64 /c0'.
# @return 1 on success, 0 if not
sub getControllerTime{
	my $storcli = shift;
	my @output = `$storcli show time`;
	return (checkCommandStatus(\@output));
}

# Get the status of the raid controller
# @param storcli The path to storcli command utility, followed by the controller
# number, e.g. 'storcli64 /c0'.
# @param logDevices If given, a list of desired logical device numbers
# @param commands_a An array to push the used command to
# @return A hash, each key a value of the raid controller info
sub getControllerInfo{
	my $storcli = shift;
	my $commands_a = shift;
	my $command = '';

	$storcli =~ /^(.*)\/c[0-9]+/;
	$command = $1.'adpallinfo a'.$CONTROLLER;
	push @{$commands_a}, $command;
	my @output = `$command`;
	if($? >> 8 != 0){
		print "Invalid StorCLI command! ($command)\n";
		exit(STATE_UNKNOWN);
	}
	my %foundController_h;
	foreach my $line(@output){
		if($line =~ /\:/){
			my @lineVals = split(':', $line);
			$lineVals[0] =~ s/^\s+|\s+$//g;
			$lineVals[1] =~ s/^\s+|\s+$//g;
			$foundController_h{$lineVals[0]} = $lineVals[1];
		}
	}
	return \%foundController_h;
}

# Checks the status of the raid controller
# @param statusLevel_a The status level array, elem 0 is the current status,
# elem 1 the warning sensors, elem 2 the critical sensors, elem 3 the verbose
# information for the sensors.
# @param foundController The hash of controller infos, created by getControllerInfo
sub getControllerStatus{
	my @statusLevel_a = @{(shift)};
	my %foundController = %{(shift)};
	my $status = '';
	foreach my $key (%foundController){
		if($key eq 'ROC temperature'){
			$foundController{$key} =~ /^([0-9]+\.?[0-9]+).*$/;
			if(defined($1)){
				if(!(checkThreshs($1, $C_TEMP_CRITICAL))){
					$status = 'Critical';
					push @{$statusLevel_a[2]}, 'ROC_Temperature';
				}
				elsif(!(checkThreshs($1, $C_TEMP_WARNING))){
					$status = 'Warning' unless $status eq 'Critical';
					push @{$statusLevel_a[1]}, 'ROC_Temperature';
				}
				$statusLevel_a[3]->{'ROC_Temperature'} = $1;
			}
		}
		elsif($key eq 'Degraded'){
			if($foundController{$key} != 0){
				$status = 'Warning' unless $status eq 'Critical';
				push @{$statusLevel_a[1]}, 'CTR_Degraded_drives';
				$statusLevel_a[3]->{'CTR_Degraded_drives'} = $foundController{$key};
			}
		}
		elsif($key eq 'Offline'){
			if($foundController{$key} != 0){
				$status = 'Warning' unless $status eq 'Critical';
				push @{$statusLevel_a[1]}, 'CTR_Offline_drives';
				$statusLevel_a[3]->{'CTR_Offline_drives'} = $foundController{$key};
			}
		}
		elsif($key eq 'Critical Disks'){
			if($foundController{$key} != 0){
				$status = 'Critical';
				push @{$statusLevel_a[2]}, 'CTR_Critical_disks';
				$statusLevel_a[3]->{'CTR_Critical_disks'} = $foundController{$key};
			}
		}
		elsif($key eq 'Failed Disks'){
			if($foundController{$key} != 0){
				$status = 'Critical';
				push @{$statusLevel_a[2]}, 'CTR_Failed_disks';
				$statusLevel_a[3]->{'CTR_Failed_disks'} = $foundController{$key};
			}
		}
		elsif($key eq 'Memory Correctable Errors'){
			if($foundController{$key} != 0){
				$status = 'Warning' unless $status eq 'Critical';
				push @{$statusLevel_a[1]}, 'CTR_Memory_correctable_errors';
				$statusLevel_a[3]->{'CTR_Memory_correctable_errors'} = $foundController{$key};
			}
		}
		elsif($key eq 'Memory Uncorrectable Errors'){
			if($foundController{$key} != 0){
				$status = 'Critical';
				push @{$statusLevel_a[2]}, 'CTR_Memory_Uncorrectable_errors';
				$statusLevel_a[3]->{'CTR_Memory_Uncorrectable_errors'} = $foundController{$key};
			}
		}
	}
	if($status ne ''){
		if($status eq 'Warning'){
			if(${$statusLevel_a[0]} ne 'Critical'){
				${$statusLevel_a[0]} = 'Warning';
			}
		}
		else{
			${$statusLevel_a[0]} = 'Critical';
		}
		$statusLevel_a[3]->{'CTR_Status'} = $status;
	}
	else{
		$statusLevel_a[3]->{'CTR_Status'} = 'OK';
	}
}

# Checks which logical devices are present for the given controller and parses
# the logical devices to a list of hashes. Each hash represents a logical device
# with its values from the output.
# @param storcli The path to storcli command utility, followed by the controller
# number, e.g. 'storcli64 /c0'.
# @param logDevices If given, a list of desired logical device numbers
# @param action The storcli action to check, 'all' or 'init'
# @param commands_a An array to push the used command to
# @return A list of hashes, each hash is one logical device. Check ldmap_a for valid
# hash keys.
sub getLogicalDevices{
	my $storcli = shift;
	my @logDevices = @{(shift)};
	my $action = shift;
	my $commands_a = shift;

	my $command = $storcli;
	if(scalar(@logDevices) == 0) { $command .= "/vall"; }
	elsif(scalar(@logDevices) == 1) { $command .= "/v$logDevices[0]"; }
	else { $command .= "/v".join(",", @logDevices); }
	$command .= " show $action";
	push @{$commands_a}, $command;

	my @output = `$command`;
	my @foundDevs;
	if(checkCommandStatus(\@output)) {
		if($action eq "all") {
			my $currBlock;
			foreach my $line(@output){
				my @splittedLine;
				if($line =~ /^\/(c[0-9]*\/v[0-9]*).*/){
					$currBlock = $1;
					next;
				}
				if(defined($currBlock)){
					if($line =~ /^DG\/VD TYPE.*/){
						@splittedLine = split(' ', $line);
						if(scalar(@splittedLine)== 9){
							$LDMAP = \@map_a;
						}
						if(scalar(@splittedLine)== 10){
							$LDMAP = \@map_cc_a;
						}
					}
					if($line =~ /^\d+\/\d+\s+\w+\d\s+\w+.*/){
						@splittedLine = map { s/^\s*//; s/\s*$//; $_; } split(/\s+/,$line);
						my %lineValues_h;
						# The current block is the c0/v0 name
						$lineValues_h{'ld'} = $currBlock;
						for(my $i = 0; $i < @{$LDMAP}; $i++){
								$lineValues_h{$LDMAP->[$i]} = $splittedLine[$i];
							}
						push @foundDevs, \%lineValues_h;
					}
				}
			}
		}
		elsif($action eq "init") {
			foreach my $line(@output){
				$line =~ s/^\s+|\s+$//g;#trim line
				if($line =~ /^([0-9]+)\s+INIT.*$/){
					my $vdNum = 'c'.$CONTROLLER.'/v'.$1;
					if($line !~ /Not in progress/i){
						my %lineValues_h;
						my @vals = split('\s+',$line);
						$lineValues_h{'ld'} = $vdNum;
						$lineValues_h{'init'} = $vals[2];
						push @foundDevs, \%lineValues_h;
					}
				}
			}
		}
	}
	else {
		print "Invalid StorCLI command! ($command)\n";
		exit(STATE_UNKNOWN);
	}
	return \@foundDevs;
}

# Checks the status of the logical devices.
# @param statusLevel_a The status level array, elem 0 is the current status,
# elem 1 the warning sensors, elem 2 the critical sensors, elem 3 the verbose
# information for the sensors.
# @param foundLDs The array of logical devices, created by getLogicalDevices
sub getLDStatus{
	my @statusLevel_a = @{(shift)};
	my @foundLDs = @{(shift)};
	my $status = '';
	foreach my $LD (@foundLDs){
		if(exists($LD->{'State'})){
			if($LD->{'State'} ne 'Optl'){
				$status = 'Critical';
				push @{$statusLevel_a[2]}, $LD->{'ld'}.'_State';
				$statusLevel_a[3]->{$LD->{'ld'}.'_State'} = $LD->{'State'};
			}
		}
		if(exists($LD->{'Consist'})){
			if($LD->{'Consist'} ne 'Yes' && $LD->{'TYPE'} ne 'Cac1'){
				$status = 'Warning' unless $status eq 'Critical';
				push @{$statusLevel_a[1]}, $LD->{'ld'}.'_Consist';
				$statusLevel_a[3]->{$LD->{'ld'}.'_Consist'} = $LD->{'Consist'};
			}
		}
		if(exists($LD->{'init'})){
			$status = 'Warning' unless $status eq 'Critical';
			push @{$statusLevel_a[1]}, $LD->{'ld'}.'_Init';
			$statusLevel_a[3]->{$LD->{'ld'}.'_Init'} = $LD->{'init'};
		}
	}
	if($status ne ''){
		if($status eq 'Warning'){
			if(${$statusLevel_a[0]} ne 'Critical'){
				${$statusLevel_a[0]} = 'Warning';
			}
		}
		else{
			${$statusLevel_a[0]} = 'Critical';
		}
		$statusLevel_a[3]->{'LD_Status'} = $status;
	}
	else{
		if(!exists($statusLevel_a[3]->{'LD_Status'})){
			$statusLevel_a[3]->{'LD_Status'} = 'OK';
		}
	}
}

# Checks which physical devices are present for the given controller and parses
# the physical devices to a list of hashes. Each hash represents a physical device
# with its values from the output.
# @param storcli The path to storcli command utility, followed by the controller
# number, e.g. 'storcli64 /c0'.
# @param physDevices If given, a list of desired physical device numbers
# @param action The storcli action to check, 'all', 'initialization' or 'rebuild'
# @param commands_a An array to push the used command to
# @return A list of hashes, each hash is one physical device. Check pdmap_a for valid
# hash keys.
sub getPhysicalDevices{
	my $storcli = shift;
	my @enclosures = @{(shift)};
	my @physDevices = @{(shift)};
	my $action = shift;
	my $commands_a = shift;

	my $command = $storcli;
	if(!$NOENCLOSURES){
		if(scalar(@enclosures) == 0) { $command .= "/eall"; }
		elsif(scalar(@enclosures) == 1) { $command .= "/e$enclosures[0]"; }
		else { $command .= "/e".join(",", @enclosures); }
	}
	if(scalar(@physDevices) == 0) { $command .= "/sall"; }
	elsif(scalar(@physDevices) == 1) { $command .= "/s$physDevices[0]"; }
	else { $command .= "/s".join(",", @physDevices); }
	$command .= " show $action";
	push @{$commands_a}, $command;

	my @output = `$command`;
	my @foundDevs;
	if(checkCommandStatus(\@output)){
		if($action eq "all") {
			my $currBlock;
			my $line_ref;
			foreach my $line(@output){
				my @splittedLine;
				if($line =~ /^Drive \/(c[0-9]*\/e[0-9]*\/s[0-9]*) \:$/){
					$currBlock = $1;
					$line_ref = {};
					next;
				}
				if(defined($currBlock)){
					# If a drive is not in a group, a - is at the DG column
					if($line =~ /^\d+\:\d+\s+\d+\s+\w+\s+[0-9-F]+.*/){
						@splittedLine = map { s/^\s*//; s/\s*$//; $_; } split(/\s+/,$line);
						# The current block is the c0/e252/s0 name
						$line_ref->{'pd'} = $currBlock;
						my $j = 0;
						for(my $i = 0; $i < @pdmap_a; $i++){
							if($pdmap_a[$i] eq 'Size'){
								my $size = $splittedLine[$j];
								if($splittedLine[$j+1] eq 'GB' || $splittedLine[$j+1] eq 'TB'){
									$size .= ''.$splittedLine[$j+1];
									$j++;
								}
								$line_ref->{$pdmap_a[$i]} = $size;
								$j++;
							}
							elsif($pdmap_a[$i] eq 'Model'){
								my $model = $splittedLine[$j];
								# Model should be the next last element, j starts at 0
								if(($j+2) != scalar(@splittedLine)){
									$model .= ' '.$splittedLine[$j+1];
									$j++;
								}
								$line_ref->{$pdmap_a[$i]} = $model;
								$j++;
							}
							else{
								$line_ref->{$pdmap_a[$i]} = $splittedLine[$j];
								$j++;
							}
						}
					}
					if($line =~ /^(Shield Counter|Media Error Count|Other Error Count|BBM Error Count|Drive Temperature|Predictive Failure Count|S\.M\.A\.R\.T alert flagged by drive)\s\=\s+(.*)$/){
						$line_ref->{$1} = $2;
					}
					# If the last value is parsed, set up for the next device
					if(exists($line_ref->{'S.M.A.R.T alert flagged by drive'})){
						push @foundDevs, $line_ref;
						undef $currBlock;
						undef $line_ref;
					}
				}
			}
		}
		elsif($action eq 'rebuild' || $action eq 'initialization') {
				foreach my $line(@output){
				$line =~ s/^\s+|\s+$//g;#trim line
				if($line =~ /^\/c$CONTROLLER\/.*/){
					if($line !~ /Not in progress/i){
						my %lineValues_h;
						my @vals = split('\s+',$line);
						my $key;
						if($action eq 'rebuild'){ $key = 'rebuild'; }
						if($action eq 'initialization'){ $key = 'init'; }
						$lineValues_h{'pd'} = substr($vals[0], 1);
						$lineValues_h{$key} = $vals[1];
						push @foundDevs, \%lineValues_h;
					}
				}
			}
		}
		# Now we check if a drive is not attached, error code 46
		my $failed_pattern = 'c[0-9]*\/e[0-9]*\/s[0-9]*\s+Failure\s+46';
		if(my @match = grep { /$failed_pattern/ } @output){
			$match[0] =~ /(c[0-9]*\/e[0-9]*\/s[0-9]*)/;
			my $dev = {};
			$dev->{'pd'} = $1;
			$dev->{'Detailed Status'} = 'Failure-46';
			push @foundDevs, $dev;
		}
	}
	else {
		if(grep { /No drive found/i } @output){
			print "Warning (CTR Warn) [No storage attached] ($command)\n";
			exit(STATE_WARNING);
		}
		print "Invalid StorCLI command! ($command)\n";
		exit(STATE_UNKNOWN);
	}
	return \@foundDevs;
}

# Checks the status of the physical devices.
# @param statusLevel_a The status level array, elem 0 is the current status,
# elem 1 the warning sensors, elem 2 the critical sensors, elem 3 the vebose
# information for the sensors.
# @param foundPDs The array of physical devices, created by getPhysicalDevices
sub getPDStatus{
	my @statusLevel_a = @{(shift)};
	my @foundPDs = @{(shift)};
	my $status = '';
	foreach my $PD (@foundPDs){
		if(exists($PD->{'State'})){
			if($PD->{'State'} ne 'Onln' && $PD->{'State'} ne 'UGood' && $PD->{'State'} ne 'GHS' && $PD->{'State'} ne 'DHS' && $PD->{'State'} ne 'JBOD'){
				$status = 'Critical';
				push @{$statusLevel_a[2]}, $PD->{'pd'}.'_State';
				$statusLevel_a[3]->{$PD->{'pd'}.'_State'} = $PD->{'State'};
			}
		}
		if(exists($PD->{'Shield Counter'})){
			if($PD->{'Shield Counter'} > $IGNERR_S){
				$status = 'Warning' unless $status eq 'Critical';
				push @{$statusLevel_a[1]}, $PD->{'pd'}.'_Shield_counter';
				$statusLevel_a[3]->{$PD->{'pd'}.'_Shield_counter'} = $PD->{'Shield Counter'};
			}
		}
		if(exists($PD->{'Media Error Count'})){
			if($PD->{'Media Error Count'} > $IGNERR_M){
				$status = 'Warning' unless $status eq 'Critical';
				push @{$statusLevel_a[1]}, $PD->{'pd'}.'_Media_error_count';
				$statusLevel_a[3]->{$PD->{'pd'}.'_Media_error_count'} = $PD->{'Media Error Count'};
			}
		}
		if(exists($PD->{'Other Error Count'})){
			if($PD->{'Other Error Count'} > $IGNERR_O){
				$status = 'Warning' unless $status eq 'Critical';
				push @{$statusLevel_a[1]}, $PD->{'pd'}.'_Other_error_count';
				$statusLevel_a[3]->{$PD->{'pd'}.'_Other_error_count'} = $PD->{'Other Error Count'};
			}
		}
		if(exists($PD->{'BBM Error Count'})){
			if($PD->{'BBM Error Count'} > $IGNERR_B){
				$status = 'Warning' unless $status eq 'Critical';
				push @{$statusLevel_a[1]}, $PD->{'pd'}.'_BBM_error_count';
				$statusLevel_a[3]->{$PD->{'pd'}.'_BBM_error_count'} = $PD->{'BBM Error Count'};
			}
		}
		if(exists($PD->{'Predictive Failure Count'})){
			if($PD->{'Predictive Failure Count'} > $IGNERR_P){
				$status = 'Warning' unless $status eq 'Critical';
				push @{$statusLevel_a[1]}, $PD->{'pd'}.'_Predictive_failure_count';
				$statusLevel_a[3]->{$PD->{'pd'}.'_Predictive_failure_count'} = $PD->{'Predictive Failure Count'};
			}
		}
		if(exists($PD->{'S.M.A.R.T alert flagged by drive'})){
			if($PD->{'S.M.A.R.T alert flagged by drive'} ne 'No'){
				$status = 'Warning' unless $status eq 'Critical';
				push @{$statusLevel_a[1]}, $PD->{'pd'}.'_SMART_flag';
			}
		}
		if(exists($PD->{'Detailed Status'})){
			if($PD->{'Detailed Status'} eq 'Failure-46'){
				$status = 'Warning' unless $status eq 'Critical';
				push @{$statusLevel_a[1]}, $PD->{'pd'}.'_Detailed_status';
			}
		}
		if(exists($PD->{'DG'})){
			if($PD->{'DG'} eq 'F'){
				$status = 'Warning' unless $status eq 'Critical';
				push @{$statusLevel_a[1]}, $PD->{'pd'}.'_DG';
				$statusLevel_a[3]->{$PD->{'pd'}.'_DG'} = $PD->{'DG'};
			}
		}
		if(exists($PD->{'Drive Temperature'})){
			my $temp = $PD->{'Drive Temperature'};
			if($temp ne 'N/A' && $temp ne '0C (32.00 F)'){
				$temp =~ /^([0-9]+)C/;
				if(!(checkThreshs($1, $PD_TEMP_CRITICAL))){
					$status = 'Critical';
					push @{$statusLevel_a[2]}, $PD->{'pd'}.'_Drive_Temperature';
				}
				elsif(!(checkThreshs($1, $PD_TEMP_WARNING))){
					$status = 'Warning' unless $status eq 'Critical';
					push @{$statusLevel_a[1]}, $PD->{'pd'}.'_Drive_Temperature';
				}
				$statusLevel_a[3]->{$PD->{'pd'}.'_Drive_Temperature'} = $1;
			}
		}
		if(exists($PD->{'init'})){
			$status = 'Warning' unless $status eq 'Critical';
			push @{$statusLevel_a[1]}, $PD->{'pd'}.'_Init';
			$statusLevel_a[3]->{$PD->{'pd'}.'_Init'} = $PD->{'init'};
		}
		if(exists($PD->{'rebuild'})){
			$status = 'Warning' unless $status eq 'Critical';
			push @{$statusLevel_a[1]}, $PD->{'pd'}.'_Rebuild';
			$statusLevel_a[3]->{$PD->{'pd'}.'_Rebuild'} = $PD->{'rebuild'};
		}
	}
	if($status ne ''){
		if($status eq 'Warning'){
			if(${$statusLevel_a[0]} ne 'Critical'){
				${$statusLevel_a[0]} = 'Warning';
			}
		}
		else{
			${$statusLevel_a[0]} = 'Critical';
		}
		$statusLevel_a[3]->{'PD_Status'} = $status;
	}
	else{
		if(!exists($statusLevel_a[3]->{'PD_Status'})){
			$statusLevel_a[3]->{'PD_Status'} = 'OK';
		}
	}
}

# Checks the status of the BBU, parses 'bbu show status' for the given controller.
# @param storcli The path to storcli command utility, followed by the controller
# number, e.g. 'storcli64 /c0'.
# @param statusLevel_a The status level array, elem 0 is the current status,
# elem 1 the warning sensors, elem 2 the critical sensors, elem 3 the verbose
# information for the sensors.
# @param commands_a An array to push the used command to
sub getBBUStatus {
	my $storcli = shift;
	my @statusLevel_a = @{(shift)};
	my $commands_a = shift;

	my $command = "$storcli /bbu show status";
	push @{$commands_a}, $command;

	my $status = '';
	my @output = `$command`;
	if(checkCommandStatus(\@output)) {
		my $currBlock;
		foreach my $line (@output) {
			if($line =~ /^(BBU_Info|BBU_Firmware_Status|GasGaugeStatus)/){
				$currBlock = $1;
				next;
			}
			if(defined($currBlock)){
				$line =~ s/^\s+|\s+$//g;#trim line
				if($currBlock eq 'BBU_Info'){
					if ($line =~ /^Battery State/){
						$line =~ /([a-zA-Z]*)$/;
							if($1 ne 'Optimal'){
								$status = 'Warning' unless $status eq 'Critical';
								push @{$statusLevel_a[1]}, 'BBU_State';
								$statusLevel_a[3]->{'BBU_State'} = $1
							}
						}
					elsif($line =~ /^Temperature/){
						$line =~ /([0-9]+) C$/;
						if(!(checkThreshs($1, $BBU_TEMP_CRITICAL))){
							$status = 'Critical';
							push @{$statusLevel_a[2]}, 'BBU_Temperature';
						}
						elsif(!(checkThreshs($1, $BBU_TEMP_WARNING))){
							$status = 'Warning' unless $status eq 'Critical';
							push @{$statusLevel_a[1]}, 'BBU_Temperature';
						}
						$statusLevel_a[3]->{'BBU_Temperature'} = $1;
					}
				}
				elsif($currBlock eq 'BBU_Firmware_Status'){
					if($line =~ /^Temperature/){
						$line =~ /([a-zA-Z]*)$/;
						if($1 ne "OK") {
							$status = 'Critical';
							push @{$statusLevel_a[2]},'BBU_Firmware_temperature';
							$statusLevel_a[3]->{'BBU_Firmware_temperature'} = $1;
						}
					}
					elsif($line =~ /^Voltage/){
						$line =~ /([a-zA-Z]*)$/;
						if($1 ne "OK") {
							$status = 'Warning' unless $status eq 'Critical';
							push @{$statusLevel_a[1]},'BBU_Voltage';
							$statusLevel_a[3]->{'BBU_Voltage'} = $1;
						}
					}
					elsif($line =~ /^I2C Errors Detected/){
						$line =~ /([a-zA-Z]*)$/;
						if($1 ne "No") {
							$status = 'Critical';
							push @{$statusLevel_a[2]},'BBU_Firmware_I2C_errors';
							$statusLevel_a[3]->{'BBU_Firmware_I2C_Errors'} = $1;
						}
					}
					elsif($line =~ /^Battery Pack Missing/){
						$line =~ /([a-zA-Z]*)$/;
						if($1 ne "No") {
							$status = 'Critical';
							push @{$statusLevel_a[2]},'BBU_Pack_missing';
							$statusLevel_a[3]->{'BBU_Pack_missing'} = $1;
						}
					}
					elsif($line =~ /^Replacement required/){
						$line =~ /([a-zA-Z]*)$/;
						if($1 ne "No") {
							$status = 'Critical';
							push @{$statusLevel_a[2]},'BBU_Replacement_required';
							$statusLevel_a[3]->{'BBU_Replacement_required'} = $1;
						}
					}
					elsif($line =~ /^Remaining Capacity Low/){
						$line =~ /([a-zA-Z]*)$/;
						if($1 ne "No") {
							$status = 'Warning' unless $status eq 'Critical';
							push @{$statusLevel_a[1]},'BBU_Remaining_capacity_low';
							$statusLevel_a[3]->{'BBU_Remaining_capacity_low'} = $1;
						}
					}
					elsif($line =~ /^Pack is about to fail \& should be replaced/){
						$line =~ /([a-zA-Z]*)$/;
						if($1 ne "No") {
							$status = 'Critical';
							push @{$statusLevel_a[2]},'BBU_Should_be_replaced';
							$statusLevel_a[3]->{'BBU_Should_be_replaced'} = $1;
						}
					}
				}
				elsif($currBlock eq 'GasGaugeStatus'){
					if($line =~ /^Fully Discharged/){
						$line =~ /([a-zA-Z\/]*)$/;
						if($1 ne "No" && $1 ne "N/A") {
							$status = 'Critical';
							push @{$statusLevel_a[2]},'BBU_GasGauge_discharged';
							$statusLevel_a[3]->{'BBU_GasGauge_discharged'} = $1;
						}
					}
					elsif($line =~ /^Over Temperature/){
						$line =~ /([a-zA-Z\/]*)$/;
						if($1 ne "No" && $1 ne "N/A") {
							$status = 'Warning' unless $status eq 'Critical';
							push @{$statusLevel_a[1]},'BBU_GasGauge_over_temperature';
							$statusLevel_a[3]->{'BBU_GasGauge_over_temperature'} = $1;
						}
					}
					elsif($line =~ /^Over Charged/){
						$line =~ /([a-zA-Z\/]*)$/;
						if($1 ne "No" && $1 ne "N/A") {
							$status = 'Critical';
							push @{$statusLevel_a[2]},'BBU_GasGauge_over_charged';
							$statusLevel_a[3]->{'BBU_GasGauge_over_charged'} = $1;
						}
					}
				}
			}
			if($status ne ''){
				if($status eq 'Warning'){
					if(${$statusLevel_a[0]} ne 'Critical'){
						${$statusLevel_a[0]} = 'Warning';
					}
				}
				else{
					${$statusLevel_a[0]} = 'Critical';
				}
				$statusLevel_a[3]->{'BBU_Status'} = $status;
			}
			else{
				$statusLevel_a[3]->{'BBU_Status'} = 'OK';
			}
		}
	}
	else {
		print "Invalid StorCLI command! ($command)\n";
		exit(STATE_UNKNOWN);
	}
}

# Checks the status of the CV module, parses 'cv show status' for the given
# controller.
# @param storcli The path to storcli command utility, followed by the controller
# number, e.g. 'storcli64 /c0'.
# @param statusLevel_a The status level array, elem 0 is the current status,
# elem 1 the warning sensors, elem 2 the critical sensors, elem 3 the verbose
# information for the sensors.
# @param commands_a An array to push the used command to
sub getCVStatus {
	my $storcli = shift;
	my @statusLevel_a = @{(shift)};
	my $commands_a = shift;

	my $command = "$storcli /cv show status";
	push @{$commands_a}, $command;

	my $status = '';
	my @output = `$command`;
	if(checkCommandStatus(\@output)) {
		my $currBlock;
		foreach my $line (@output) {
			if($line =~ /^(Cachevault_Info|Firmware_Status)/){
				$currBlock = $1;
				next;
			}
			if(defined($currBlock)){
				$line =~ s/^\s+|\s+$//g;#trim line
				if($currBlock eq 'Cachevault_Info' && $line =~ /^State/){
					my @vals = split('\s{2,}',$line);
					if($vals[1] ne "Optimal") {
						$status = 'Warning' unless $status eq 'Critical';
						push @{$statusLevel_a[1]}, 'CV_State';
						$statusLevel_a[3]->{'CV_State'} = $vals[1]
					}
				}
				elsif($currBlock eq 'Cachevault_Info' && $line =~ /^Temperature/){
					$line =~ /([0-9]+) C$/;
					if(!(checkThreshs($1, $CV_TEMP_CRITICAL))){
						$status = 'Critical';
						push @{$statusLevel_a[2]}, 'CV_Temperature';
					}
					elsif(!(checkThreshs($1, $CV_TEMP_WARNING))){
						$status = 'Warning' unless $status eq 'Critical';
						push @{$statusLevel_a[1]}, 'CV_Temperature';
					}
					$statusLevel_a[3]->{'CV_Temperature'} = $1;
				}
				elsif($currBlock eq 'Firmware_Status' && $line =~ /^Replacement required/){
					$line =~ /([a-zA-Z0-9]*)$/;
					if($1 ne "No") {
						$status = 'Critical';
						push @{$statusLevel_a[2]},'CV_Replacement_required';
					}
					$statusLevel_a[3]->{'CV_Replacement_required'} = $1;
				}
			}
			if($status ne ''){
				if($status eq 'Warning'){
					if(${$statusLevel_a[0]} ne 'Critical'){
						${$statusLevel_a[0]} = 'Warning';
					}
				}
				else{
					${$statusLevel_a[0]} = 'Critical';
				}
				$statusLevel_a[3]->{'CV_Status'} = $status;
			}
			else{
				$statusLevel_a[3]->{'CV_Status'} = 'OK';
			}
		}
	}
	else {
		print "Invalid StorCLI command! ($command)\n";
		exit(STATE_UNKNOWN);
	}
}

# Checks if wheter BBU or CV is present
# @param storcli The path to storcli command utility, followed by the controller
# number, e.g. 'storcli64 /c0'.
# @return A tuple, e.g. (0,0), where 0 means module is not present, 1 present
sub checkBBUorCVIsPresent{
	my $storcli = shift;
	my ($bbu,$cv);
	my @output = `$storcli /bbu show`;
	if(checkCommandStatus(\@output)){ $bbu = 1; }
	else{ $bbu = 0 };
	@output = `$storcli /cv show`;
	if(checkCommandStatus(\@output)) { $cv = 1; }
	else{ $cv = 0 };
	return ($bbu, $cv);
}

# Checks if a given value is in a specified range, the range must follow the
# nagios development guidelines:
# http://nagiosplug.sourceforge.net/developer-guidelines.html#THRESHOLDFORMAT
# @param value The given value to check the pattern for
# @param pattern The pattern specifying the threshold range, e.g. '10:', '@10:20'
# @return 0 if the value is outside the range, 1 if the value satisfies the range
sub checkThreshs{
	my $value = shift;
	my $pattern = shift;
	if($pattern =~ /(^[0-9]+$)/){
		if($value < 0 || $value > $1){
			return 0;
		}
	}
	elsif($pattern =~ /(^[0-9]+)\:$/){
		if($value < $1){
			return 0;
		}
	}
	elsif($pattern =~ /^\~\:([0-9]+)$/){
		if($value > $1){
			return 0;
		}
	}
	elsif($pattern =~ /^([0-9]+)\:([0-9]+)$/){
		if($value < $1 || $value > $2){
			return 0;
		}
	}
	elsif($pattern =~ /^\@([0-9]+)\:([0-9]+)$/){
		if($value >= $1 and $value <= $2){
			return 0;
		}
	}
	else{
		print "Invalid temperature parameter! ($pattern)\n";
		exit(STATE_UNKNOWN);
	}
	return 1;
}

# Get the status string as plugin output
# @param level The desired level to get the status string for. Either 'Warning'
# or 'Critical'.
# @param statusLevel_a The status level array, elem 0 is the current status,
# elem 1 the warning sensors, elem 2 the critical sensors, elem 3 the verbose
# information for the sensors, elem 4 the used storcli commands.
# @return The created status string
sub getStatusString{
	my $level = shift;
	my @statusLevel_a = @{(shift)};
	my @sensors_a;
	my $status_str = "";
	if($level eq "Warning"){
		@sensors_a = @{$statusLevel_a[1]};
	}
	if($level eq "Critical"){
		@sensors_a = @{$statusLevel_a[2]};
	}
	# Add the controller parts only once
	my $parts = '';
	# level comes from the method call, not the real status level
	if($level eq "Critical"){
		my @keys = ('CTR_Status','LD_Status','PD_Status','BBU_Status','CV_Status');
		# Check which parts where checked
		foreach my $key (@keys){
			$key =~ /^([A-Z]+)\_.*$/;
			my $part = $1;
			if(${$statusLevel_a[0]} eq 'OK'){
				if(exists($statusLevel_a[3]->{$key}) && $statusLevel_a[3]->{$key} eq 'OK'){
					$parts .= ", " unless $parts eq '';
					$parts .= $part;
				}
			}
			else{
				if(exists($statusLevel_a[3]->{$key}) && $statusLevel_a[3]->{$key} ne 'OK'){
					$parts .= ", " unless $parts eq '';
					$parts .= $part;
					$parts .= ' '.substr($statusLevel_a[3]->{$key}, 0, 4);
				}
			}
		}
		$status_str.= '(';
		$status_str .= $parts unless !defined($parts);
		$status_str.= ')';
	}
	if($level eq 'Critical'){
		$status_str.= ' ' unless !(@sensors_a);
	}
	if($level eq 'Warning' && !@{$statusLevel_a[2]}){
		$status_str.= ' ' unless !(@sensors_a);
	}
	if($level eq "Warning" || $level eq "Critical"){
		if(@sensors_a){
			# Print which sensors are Warn or Crit
			foreach my $sensor (@sensors_a){
				$status_str .= "[".$sensor." = ".$level;
				if($VERBOSITY){
					if(exists($statusLevel_a[3]->{$sensor})){
						$status_str .= " (".$statusLevel_a[3]->{$sensor}.")";
					}
				}
				$status_str .= "]";
			}
		}
	}
	return $status_str;
}

# Get the verbose string if a higher verbose level is used
# @param statusLevel_a The status level array, elem 0 is the current status,
# elem 1 the warning sensors, elem 2 the critical sensors, elem 3 the verbose
# information for the sensors, elem 4 the used storcli commands.
# @param controllerToCheck Controller parsed by getControllerInfo
# @param LDDevicesToCheck LDs parsed by getLogicalDevices
# @param LDInitToCheck LDs parsed by getLogicalDevices init
# @param PDDevicesToCheck PDs parsed by getPhysicalDevices
# @param PDInitToCheck PDs parsed by getPhysicalDevices init
# @param PDRebuildToCheck PDs parsed by getPhysicalDevices rebuild
# @return The created verbosity string
sub getVerboseString{
	my @statusLevel_a = @{(shift)};
	my %controllerToCheck = %{(shift)};
	my @LDDevicesToCheck = @{(shift)};
	my @LDInitToCheck = @{(shift)};
	my @PDDevicesToCheck = @{(shift)};
	my @PDInitToCheck = @{(shift)};
	my @PDRebuildToCheck = @{(shift)};
	my @sensors_a;
	my $verb_str;

	$verb_str .= "Used storcli commands:\n";
	foreach my $cmd (@{$statusLevel_a[4]}){
		$verb_str .= '- '.$cmd."\n";
	}
	if(${$statusLevel_a[0]} eq 'Critical'){
		$verb_str .= "Critical sensors:\n";
		foreach my $sensor (@{$statusLevel_a[2]}){
			$verb_str .= "\t- ".$sensor;
			if(exists($statusLevel_a[3]->{$sensor})){
				$verb_str .= ' ('.$statusLevel_a[3]->{$sensor}.')';
			}
			$verb_str .= "\n";
		}

	}
	if( ${$statusLevel_a[0]} ne 'OK'){
		$verb_str .= "Warning sensors:\n";
		foreach my $sensor (@{$statusLevel_a[1]}){
			$verb_str .= "\t- ".$sensor;
			if(exists($statusLevel_a[3]->{$sensor})){
				$verb_str .= ' ('.$statusLevel_a[3]->{$sensor}.')';
			}
			$verb_str .= "\n";
		}

	}
	if($VERBOSITY == 3){
		$verb_str .= "CTR information:\n";
		$verb_str .= "\t- ".$controllerToCheck{'Product Name'}.":\n";
		$verb_str .= "\t\t- ".'Serial No='.$controllerToCheck{'Serial No'}."\n";
		$verb_str .= "\t\t- ".'FW Package Build='.$controllerToCheck{'FW Package Build'}."\n";
		$verb_str .= "\t\t- ".'Mfg. Date='.$controllerToCheck{'Mfg. Date'}."\n";
		$verb_str .= "\t\t- ".'Revision No='.$controllerToCheck{'Revision No'}."\n";
		$verb_str .= "\t\t- ".'BIOS Version='.$controllerToCheck{'BIOS Version'}."\n";
		$verb_str .= "\t\t- ".'FW Version='.$controllerToCheck{'FW Version'}."\n";
		$verb_str .= "\t\t- ".'ROC temperature='.$controllerToCheck{'ROC temperature'}."\n";
		$verb_str .= "LD information:\n";
		foreach my $LD (@LDDevicesToCheck){
			$verb_str .= "\t- ".$LD->{'ld'}.":\n";
			foreach my $key (sort (keys(%{$LD}))){
				$verb_str .= "\t\t- ".$key.'='.$LD->{$key}."\n";
			}
			foreach my $LDinit (@LDInitToCheck){
				if($LDinit->{'ld'} eq $LD->{'ld'}){
					$verb_str .= "\t\t- init=".$LDinit->{'init'}."\n";
				}
			}
		}
		$verb_str .= "PD information:\n";
		foreach my $PD (@PDDevicesToCheck){
			$verb_str .= "\t- ".$PD->{'pd'}.":\n";
			foreach my $key (sort (keys(%{$PD}))){
				$verb_str .= "\t\t- ".$key.'='.$PD->{$key}."\n";
			}
			foreach my $PDinit (@PDInitToCheck){
				if($PDinit->{'pd'} eq $PD->{'pd'}){
					$verb_str .= "\t\t- init=".$PDinit->{'init'}."\n";
				}
			}
			foreach my $PDrebuild (@PDRebuildToCheck){
				if($PDrebuild->{'pd'} eq $PD->{'pd'}){
					$verb_str .= "\t\t- rebuild=".$PDrebuild->{'rebuild'}."\n";
				}
			}
		}
		my @keys = ('BBU_Status','CV_Status');
		foreach my $key(@keys){
			if(exists($statusLevel_a[3]->{$key})){
				$key =~ /^(\w+)_\w+$/;
				my $type = $1;
				$verb_str .= $type." information:\n";
				foreach my $stat (sort (keys(%{$statusLevel_a[3]}))){
					if($stat =~ /^$type.+$/){
						$verb_str .= "\t\t- $stat=".$statusLevel_a[3]->{$stat}."\n";
					}
				}
			}
		}
	}
	return $verb_str;
}

# Get the performance string for the current check. The values are taken from
# the varbose hash in the status level array.
# @param statusLevel_a The current status level array
# @return The created performance string
sub getPerfString{
	my @statusLevel_a = @{(shift)};
	my %verboseValues_h = %{$statusLevel_a[3]};
	my $perf_str;
	foreach my $key (sort (keys(%verboseValues_h))){
		if($key =~ /temperature/i){
			$perf_str .= ' ' unless !defined($perf_str);
			$perf_str .= $key.'='.$verboseValues_h{$key};
		}
		if($key =~ /ROC_Temperature$/){
			$perf_str .= ';'.$C_TEMP_WARNING.';'.$C_TEMP_CRITICAL;
		}
		elsif($key =~ /Drive_Temperature$/){
			$perf_str .= ';'.$PD_TEMP_WARNING.';'.$PD_TEMP_CRITICAL;
		}
		elsif($key eq 'BBU_Temperature'){
			$perf_str .= ';'.$BBU_TEMP_WARNING.';'.$BBU_TEMP_CRITICAL;
		}
		elsif($key eq 'CV_Temperature'){
			$perf_str .= ';'.$CV_TEMP_WARNING.';'.$CV_TEMP_CRITICAL;
		}
	}
	return $perf_str;
}

MAIN: {
	my ($storcli, $sudo, $noSudo, $noCleanlogs, $version, $exitCode);
	# Create default sensor arrays and push them to status level
	my @statusLevel_a ;
	my $status_str = 'OK';
	my $warnings_a = [];
	my $criticals_a = [];
	my $verboseValues_h = {};
	my $verboseCommands_a = [];
	push @statusLevel_a, \$status_str;
	push @statusLevel_a, $warnings_a;
	push @statusLevel_a, $criticals_a;
	push @statusLevel_a, $verboseValues_h;
	push @statusLevel_a, $verboseCommands_a;
	# Per default use a BBU
	my $bbu = 1;
	my @enclosures;
	my @logDevices;
	my @physDevices;
	my $platform = $^O;

	if( !(GetOptions(
		'h|help' => sub {displayHelp();},
		'v|verbose' => sub {$VERBOSITY = 1 },
		'vv' => sub {$VERBOSITY = 2},
		'vvv' => sub {$VERBOSITY = 3},
		'V|version' => \$version,
		'C|controller=i' => \$CONTROLLER,
		'EID|enclosure=s' => \@enclosures,
		'LD|logicaldevice=s' => \@logDevices,
		'PD|physicaldevice=s' => \@physDevices,
		'Tw|temperature-warn=s' => \$C_TEMP_WARNING,
		'Tc|temperature-critical=s' => \$C_TEMP_CRITICAL,
		'PDTw|physicaldevicetemperature-warn=s' => \$PD_TEMP_WARNING,
		'PDTc|physicaldevicetemperature-critical=s' => \$PD_TEMP_CRITICAL,
		'BBUTw|bbutemperature-warning=s' => \$BBU_TEMP_WARNING,
		'BBUTc|bbutemperature-critical=s' => \$BBU_TEMP_CRITICAL,
		'CVTw|cvtemperature-warning=s' => \$CV_TEMP_WARNING,
		'CVTc|cvtemperature-critical=s' => \$CV_TEMP_CRITICAL,
		'Im|ignore-media-errors=i' => \$IGNERR_M,
		'Io|ignore-other-errors=i' => \$IGNERR_O,
		'Ip|ignore-predictive-fail-count=i' => \$IGNERR_P,
		'Is|ignore-shield-counter=i' => \$IGNERR_S,
		'Ib|ignore-bbm-counter=i' => \$IGNERR_B,
		'p|path=s' => \$storcli,
		'b|BBU=i' => \$bbu,
		'noenclosures=i' => \$NOENCLOSURES,
		'nosudo' => \$noSudo,
		'nocleanlogs' => \$noCleanlogs
	))){
		print $NAME . " Version: " . $VERSION ."\n";
		displayUsage();
		exit(STATE_UNKNOWN);
	}
	if(defined($version)){ print $NAME . "\nVersion: ". $VERSION . "\n"; }
	# Check storcli tool
	if(!defined($storcli)){
		if($platform eq 'linux' || $platform eq 'solaris'){
			$storcli = which('storcli');
			if(!defined($storcli)){
				$storcli = which('storcli64');
			}
		}
		else{
			$storcli = which('storcli.exe');
			if(!defined($storcli)){
				$storcli = which('storcli64.exe');
			}
		}
	}
	if(!defined($storcli)){
		print "Error: cannot find storcli executable.\n";
		print "Ensure storcli is in your path, or use the '-p <storcli path>' switch!\n";
		exit(STATE_UNKNOWN);
	}
	if($platform eq 'linux' || $platform eq 'solaris') {
		if(!defined($noSudo)){
			my $sudo;
			chomp($sudo = `which sudo`);
			if(!defined($sudo)){
				print "Error: cannot find sudo executable.\n";
				exit(STATE_UNKNOWN);
			}
			if($> != 0){
				$storcli = $sudo.' '.$storcli;
			}
		}
	}
	# Print storcli version if available
	if(defined($version)){ displayVersion($storcli) }
	# Prepare storcli command
	$storcli .= " /c$CONTROLLER";
	# Check if the controller number can be used
	if(!getControllerTime($storcli)){
		print "Error: invalid controller number, controller not found!\n";
		exit(STATE_UNKNOWN);
	}
	# Prepare command line arrays
	@enclosures = split(/,/,join(',', @enclosures));
	@logDevices = split(/,/,join(',', @logDevices));
	@physDevices = split(/,/,join(',', @physDevices));
	# Check if the BBU param is correct
	if(($bbu != 1) && ($bbu != 0)) {
		print "Error: invalid BBU/CV parameter, must be 0 or 1!\n";
		exit(STATE_UNKNOWN);
	}
	my ($bbuPresent,$cvPresent) = (0,0);
	if($bbu == 1){
		($bbuPresent,$cvPresent) = checkBBUorCVIsPresent($storcli);
		if($bbuPresent == 0 && $cvPresent == 0){
			${$statusLevel_a[0]} = 'Critical';
			push @{$criticals_a}, 'BBU/CV_Present';
			$statusLevel_a[3]->{'BBU_Status'} = 'Critical';
			$statusLevel_a[3]->{'CV_Status'} = 'Critical';
		}
	}
	if($bbuPresent == 1){getBBUStatus($storcli, \@statusLevel_a, $verboseCommands_a); }
	if($cvPresent == 1){ getCVStatus($storcli, \@statusLevel_a, $verboseCommands_a); }

	my $controllerToCheck = getControllerInfo($storcli, $verboseCommands_a);
	my $LDDevicesToCheck = getLogicalDevices($storcli, \@logDevices, 'all', $verboseCommands_a);
	my $LDInitToCheck = getLogicalDevices($storcli, \@logDevices, 'init', $verboseCommands_a);
	my $PDDevicesToCheck = getPhysicalDevices($storcli, \@enclosures, \@physDevices, 'all', $verboseCommands_a);
	my $PDInitToCheck = getPhysicalDevices($storcli, \@enclosures, \@physDevices, 'initialization', $verboseCommands_a);
	my $PDRebuildToCheck = getPhysicalDevices($storcli, \@enclosures, \@physDevices, 'rebuild', $verboseCommands_a);

	getControllerStatus(\@statusLevel_a, $controllerToCheck);
	getLDStatus(\@statusLevel_a, $LDDevicesToCheck);
	getLDStatus(\@statusLevel_a, $LDInitToCheck);
	getPDStatus(\@statusLevel_a, $PDDevicesToCheck);
	getPDStatus(\@statusLevel_a, $PDInitToCheck);
	getPDStatus(\@statusLevel_a, $PDRebuildToCheck);

	# If desired, clean up the logs created by storcli
	# This is done per default
	if(!defined($noCleanlogs)){
		if(-f 'MegaSAS.log'){ unlink 'MegaSAS.log'}
		if(-f 'CmdTool.log'){ unlink 'CmdTool.log'}
	}

	print ${$statusLevel_a[0]}." ";
	print getStatusString("Critical",\@statusLevel_a);
	print getStatusString("Warning",\@statusLevel_a);
	my $perf_str = getPerfString(\@statusLevel_a);
	if($perf_str){
		print "|".$perf_str;
	}
	if($VERBOSITY == 2 || $VERBOSITY == 3){
		print "\n".getVerboseString(\@statusLevel_a, $controllerToCheck, $LDDevicesToCheck, $LDInitToCheck,
		$PDDevicesToCheck, $PDInitToCheck, $PDRebuildToCheck)
	}
	$exitCode = STATE_OK;
	if(${$statusLevel_a[0]} eq "Critical"){
		$exitCode = STATE_CRITICAL;
	}
	if(${$statusLevel_a[0]} eq "Warning"){
		$exitCode = STATE_WARNING;
	}
	exit($exitCode);
}
