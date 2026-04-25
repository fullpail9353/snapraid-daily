# Detailed Operation

As optional reading, a detailed description of all the steps carried out by **SnapRAID-DAILY** are outlined below:

## Step 1 - Initial Checks

The script carries out an initial check of everything before it will even attempt to interact with **SnapRAID**.

Initially the config file **snapraid-daily.conf** is read, and its contents are checked. If anything is undefined, a default
value is assumed.

If the main config file for SnapRAID itself is not present in the default location or doesn't exist, the script will continue with all defaults.

The script will then check for an updated version on the Github page if the parameter to disable the update check is not used in the config file.

Next checks are carried out for the script dependencies: awk, grep, sed, mktemp, tee and SnapRAID itself. It will exit if any of these
are not present.

Next, a check is performed on all of the config files that are defined in the SnapRAID config (**/etc/snapraid.conf**). If any of them are
not found or not writable the script will exit with an error and notify the user. This is useful to check if something unexpected has gone on like one
of the disks dropping offline etc. Note that the parity files are not checked for, the reasoning being not to wake the parity disks
from standby if no changes are found and thus a sync is not required if the sync-only argument is used. Given that the script always
runs **snapraid diff**, this will wake all of the data disks anyway.

One could check all config files exist,  but given that there is usually a copy on each disk, that would mean potentially
waking all the disks from standby which would be undesirable if a sync/scrub is not ran because of a subsequent issue.

Likewise, it was decided not to check if the parity files exist to again prevent waking the disks unless a sync or a
scrub operation are **actually** going to be carried out.

If there is an issue with not being able to access the parity files or content files, the status command will flag this in the case of
a missing content file, and the sync or scrub will fail in the case of a missing parity file, and the user will be notified anyway.

## Step 2 - Check SnapRAID Array Current Status

Checks the Array for Errors or if Touch is required using `snapraid status`

If errors are encountered at this point, the script will exit and send a notification email to the user. It will continue to
do this each time the script is invoked, and stop here until the user intervenes to address whatever the error is.

If a sync was found to be in progress (that is - the last sync was interrupted), and **\--scrub-only** is NOT specified, the
script will continue as the solution to this is usually to let the sync complete. If **\--scrub-only** is specified, the script
will exit here.

A check is also carried out if **SnapRAID** is already in use, ie. whether a **sync** or **scrub** etc. is currently running.
SnapRAID will not allow multiple instances of itself processing the same array.

The script will exit in this case and the user is also notified explicity of this.

## Step 3 - Run Start Hooks If Given

After all checks are completed, and SnapRAID's status looks okay, the start hooks are called here one-by-one if specified in the config file.
If any of the start hooks report an error, the script will exit here and notify the user via email or notification hooks.

## Step 4 - Run Touch if Required

If it is determined from the initial check that 1 or more files do not have sub-zero timestamps, **snapraid touch** is ran to
add the sub-zero timestamps.

Touch of files added to the array by default will run the next time the script is executed after the files have been added.

SnapRAID is invoked with **-v** & **-l** switches to turn on verbose mode and logging, this is to aid in quick debugging if
errors are detected during the touch.

The output is monitored for errors and if any are detected, the script will exit, send a notification email to the user or call
the notification hooks and attach the output of the log created with the "-l" argument to SnapRAID.

The idea here is that one can quickly know what the error is via the notification email alone.

If it is determined that the touch operation is not required, this step will be skipped entirely.

This step is also skipped if the **\--scrub-only** argument to the script is used.

## Step 5 - Check Array for Changes

Runs **snapraid diff** to check the array for changes to determine if a sync is required or not.

If no changes are detected, the script will skip the sync entirely. If changes are detected, the sync will proceed. In the event that
the **-s, --sync-only** is used, the script will exit here if no changes are detected.

However if either the threshold for deletions, moves or updates that are defined in the config file **snapraid-daily.conf** are found to be
exceeded, the script will exit and notify the user via email and/or call the notification hook(s) if present.

The theory is that if excess deletions, moves or updates are detected, it could very well be accidental, and a subsequent sync could prevent
the recovery of that data.

For subsequent runs, the script will continue to do this and stop at this point until the user intervenes. This step is skipped if the
**\--scrub-only** argument is used.

## Step 6 - Run Sync to Update the Array

Runs **snapraid sync** to update the array. The start-time and finish time are monitored to compute the duration so it can be added to the
main log that will form the email body.

The **-v** and **-l** switches are used to turn on verbose mode and logging, this is to aid in quick debugging if errors are detected
during the sync operation.

The **-h** option is also used to read data a 2nd time during the sync. This is to serve as an additional safeguard against silent errors
during what is an extreme condition for the machine whereby all disks are spinning at the same time. See the SnapRAID documentation here for
more information:     
* [https://www.snapraid.it](https://www.snapraid.it)

The output is monitored for errors and if any are detected, the script will exit, send a notification email to the user and/or call the notification
hook and attach the output of the log created with the **-l** argument to SnapRAID.

The idea here is that one can quickly know what the error is via the notification email/other method alone.

If it is determined that the sync operation is **NOT** required from the above Step of checking for changes, this step will be skipped
entirely.

This step is also skipped if **\--scrub-only** is active.

## Step 7 - Run Scrub to Check for Silent Corruption

Runs Scrub using the **scrub_percent** & **scrub_age** input parameters specified in the config file **snapraid-daily.conf**. The start-time
and finish time are computed such that is can be added to the main log that will form the email body.

Before the scrub is carried out, the array is once again checked for changes since the last sync so the scrub can correctly run.

In the default setup where **SnapRAID-DAILY** is called with no arguments, this check should not find any changes since a sync was carried
out moments ago. However when one is using the **-c, --scrub-only** option this may not be the case.

This is required as **SnapRAID** will exit with an error during a scrub if it finds files that have been modified and not synced.

In this case, the script will exit and the user will be notified via email/notification hook(s) explicitly that a sync is required.

The scrub is then called with the **-v** & **-l** switches to turn on verbose mode and logging, this is to aid in quick debugging if errors
are detected during the scrub operation.

The output is monitored for errors and if any are detected, the script will exit, send a notification email to the user and/or call the notification hook(s),
and attach the logfile from the output of the log created with the **-l** argument to SnapRAID.

Once again, this is to allow for quick debugging to know exactly what the error is from the notification email alone.

If the scrub does encounter issues such as silent corruption, the status check at the start of the script when ran the next time will flag them,
and the script will subsequently exit each time it is invoked until the user intervenes to attempt a manual fix.

## Step 8 - Run End Hooks If Given

The end hooks are now called here one-by-one if specified in the config file. If the any of the end hooks exit in error, the script will print a warning and
still continue so that the final notification(s) are sent.

The end hooks are also ran in the event of error conditions for SnapRAID touch/diff/sync/scrub operations above. If for example one stops a list of services
with the start hook(s), this means that the services are always restarted via the end hook(s) regardless of the outcome of the script.

## Step 9 - Generate SMART Report and Spin Down Disks if Enabled

Next, the script will call **snapraid smart** and **snapraid down** to generate a SMART Report and spin the disks down if these parameters are
enabled in the configuration file. If the script is not ran as root, it will attempt to call those commands using sudo.

## Step 10 - Send Final Notification Email

If no errors are detected during the touch, sync and scrub, or just touch & sync, or just scrub (depending on whether the **-s, \--sync-only**
or **-c, \--scrub-only** arguments are used):

If emails on success are not disabled, the final condensed log file for the email is sent to the user. This will contain an concise output of all the operations carried
out and what the result was (an example is shown below).

## Step 11 - Run Notification Hooks if Given

Lastly the notification hooks are called here one-by-one if specified in the config file.
