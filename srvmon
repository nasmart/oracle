#!/usr/bin/perl -w
# --------------------------------------------------------------------------
# Description: Script to establish if a service is running on its preferred
#              instances and will optionally ouput the command required to
#              relocate the service from the available instances where it is
#              currently running to the preferred instance(s).
# --------------------------------------------------------------------------
#
# --------------------------------------------------------------------------
# Change Log:
# ------------------- ---- -------------------------------------------------
# Yong Huang           1.0 Initial version
# Coskan Gundogar      2.0 Modified for 11.2
# Martin Nash          3.0 Changed to crsctl and runs for all databases on
#                          the cluster that it is run on.
# Martin Nash          3.1 Added the option to output commands and code to
#                          handle instances being down.
# --------------------------------------------------------------------------
#
# --------------------------------------------------------------------------
# ToDo Section:
#    + Add option to filter on SERVICE_NAME
#    + Add code to check for service being disabled and report on this
#    + Need to exit if srvctl commands return an unrecognised format as this
#      will generally indicate an error in communication with CRSD
# --------------------------------------------------------------------------

use strict;
use Getopt::Std;
use File::Basename;

my $scriptname = &File::Basename::fileparse($0);
my $version_no = "3.1";

# Below stops execution when using --help or --version options
$Getopt::Std::STANDARD_HELP_VERSION = 1;

getopts('cvd:');

our($opt_c, $opt_d, $opt_v);

# Making sure that the database name is lowercase
$opt_d = lc $opt_d;

$ENV{PATH}="/usr/bin:/bin";
chomp(my $CRS_HOME = `grep "^+ASM[1-8]" /etc/oratab | cut -d":" -f2`);

# Get all the services that are registered in OCR (filtering on database name if supplied vi -d option)
$_ = `$CRS_HOME/bin/crsctl stat res -w "TYPE = ora.service.type" | grep NAME= | grep ora.$opt_d | sed 's/NAME=//'`;

# Add all services to @line
my @line = split /\n/;
foreach (@line) {
  my @fields = split /\./;
  my $db = $fields[1];
  my $srv = $fields[2];
  verbose("Examining service $srv in database $db");
  # Need to use srvctl from the RDBMS Home as srvctl version must match the resource version to avoid error
  chomp(my $DB_HOME = `grep -i "^$db:" /etc/oratab | cut -d":" -f2`);
  # Set ORACLE_HOME environment variable in OS shell to corect value for database being queried
  $ENV{ORACLE_HOME}=$DB_HOME;
  # Get PREFERRED and AVAILABLE instances lists
  $_ = `$DB_HOME/bin/srvctl config service -d $db -s $srv | egrep "(Available|Preferred) instances" | awk -F": " '{print \$NF}'`;
  my @srvctl_output = split /\n/; 
  # PREFFERED is first line
  my @pref = split (/,/, shift @srvctl_output);
  # Do not sort so that ordering of priority is retained from srvctl output
  chomp (my $pref = join(",",@pref));
  verbose("PREFERRED instance(s) list for $srv is: $pref");
  # AVAILABLE is second line
  my @avail = ();
  if (@srvctl_output) { @avail = split (/,/, shift @srvctl_output); }
  # Do not sort so that ordering of priority is retained from srvctl output
  chomp (my $avail = join(",",@avail));
  verbose("AVAILABLE instance(s) list for $srv is: $avail");
  # print "$pref\n"; # DEBUG
  # Get output of service status command
  $_ = `$DB_HOME/bin/srvctl status service -d $db -s $srv`;
  my @srv_stat = split;
  my @curr = ();
  my $curr = "";
  my $srv_down = "N";
  if ( $srv_stat[3] eq "not" ) { # Check to see if the service id down
    $srv_down = "Y";
    warning("Service $srv is DOWN!");
  } else { # If it's not down then the 7th token is list of instances
    # Sorting just in case this isn't guaranteed by srvctl
    @curr = split (/,/, $srv_stat[6]);
    @curr = sort @curr;
    chomp ($curr = join(",",@curr));
    verbose("CURRENT instance(s) list for $srv is: $curr");
  }
  # Need to have the list of preferred instances ordered for comparion with order current instances list
  my @ordered_pref = sort @pref;
  chomp (my $ordered_pref = join(",",@ordered_pref));
  if ($ordered_pref ne $curr) { # If the sorted preferred and current instances don't match
    verbose("PREFERRED and CURRENT instance(s) do not match");
    if ( $srv_down eq "Y" ) { # If the service is down
      warning("$srv (service) for $db (database) has preferred instance(s) $pref and is DOWN!");
    } else {    
      warning("$srv (service) for $db (database) has preferred instance(s) $pref and is running on instance(s) $curr");
    }
    if ( $opt_c && $opt_c == 1 ) { # If the "-c" option was provided on the command line
      verbose("Displaying commands as \"-c\" option was supplied on the command line.");
      # Generate commands to start/relocate the service using srvctl 
      if ( $srv_down eq "Y" ) { # If the service is down
        command("Start $srv USING: srvctl start service -d $db -s $srv\n");
      } 
      else {    
        # Put the current instances into a hash
        my %curr_inst = ();
        foreach (@curr) {
          chomp ($curr = $_);
          $curr_inst{"$curr"} = 1;
        }
        # Initialise hash for recording which of the AVAILABLE instances have been used to relocate from
        my %used_inst = ();
        # Iterate through list (@pref), checking hash (%curr_inst) to find preferred instances that are not running
        foreach (@pref) {
          chomp (my $pref = $_);
          unless ( $curr_inst{"$pref"} ) { # We've found a preferred instance that is up, but does not have the service on it
            verbose("Evaluating relocation for $srv to $pref - checking status of instance.");
            # Need to check if the instance is actually up
            $_ = `$DB_HOME/bin/srvctl status instance -d $db -i $pref | awk '{ print \$4 }'`;
            if ( /not/ ) { # Instances is down
              warning("Preferred instance $pref for service $srv in database $db is down, so can't relocate service to it!");
            } else {
              # Scenario 1 - The service has been stopped manually on the preferred instance.
              # Scenario 2 - The service has failed over to an available instance.
              # Need to find a current instance that is AVAILABLE and output relocation command
              verbose("Instance $pref is up - selecting AVAILABLE instance to relocate from.");
              # If there are available instances in where the service is running then the services
              # should be relocated from one of these to the "current preferred instance"
              foreach (reverse @avail) { # Reversing the available instance list in order to honor the priority when relocating
                chomp (my $avail = $_);
                if ( $curr_inst{"$avail"} ) { 
                  command("Relocate $srv from $curr to $pref USING: srvctl relocate service -d $db -s $srv -i $curr -t $pref");
                  $used_inst{"$curr"} = 1;
                }  
              }
              if ( scalar keys %used_inst == 0 ) {
                # None of the current instances are available, so it must have been stopped manually
                verbose("No AVAILABLE instances to relocate from - service must have been stopped manually.");
                command("Start $srv on $pref USING: srvctl start service -d $db -s $srv -i $pref");
              }
            }
          }
        }
      }
    }
  }
  verbose("Completed examination of service $srv in database $db");
}

sub verbose {
  if ( $opt_v ) {
    print "INFO: @_\n";
  }
}

sub command {
  if ( $opt_c ) { # SAFETY NET - Not really required as function only called after testing for command option
    print "CMD : @_\n";
  }
}

sub warning {
  print "WARN: @_\n";
}

sub HELP_MESSAGE {
print "\nUsage: $scriptname [-d <database name>] [-c] [-v]\n\n" .
      "Where:" .
      "\t-d\tAllows specification of a single database to check\n" .
      "\t-c\tPrints command(s) required to start/relocate services to their preferred instances\n" .
      "\t-v\tVerbose output so you can see what it's doing\n\n";
}

sub VERSION_MESSAGE {
print "$scriptname - version $version_no\n"
}  
