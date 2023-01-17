# Configuration

1. Disable SELinux
   ```
   setenforce 0
   ```
1. Install rsyslog
   ```
   yum install rsyslog
   ```
1. Verify the rsyslog version
   ```
	# rsyslogd -v
	rsyslogd 5.8.10, compiled with:
			FEATURE_REGEXP:                         Yes
			FEATURE_LARGEFILE:                      No
			GSSAPI Kerberos 5 support:              Yes
			FEATURE_DEBUG (debug build, slow code): No
			32bit Atomic operations supported:      Yes
			64bit Atomic operations supported:      Yes
			Runtime Instrumentation (slow code):    No

	See http://www.rsyslog.com for more information.
   ```
1. Create spool directory
   ```
   mkdir /var/spool/rsyslog
   chown syslog:syslog /var/spool/rsyslog
   chmod 770 /var/spool/rsyslog
   ```
1. Rsyslog configuration `/etc/rsyslog.conf` as below
    ```
    # rsyslog v5 configuration file

    # For more information see /usr/share/doc/rsyslog-*/rsyslog_conf.html
    # If you experience problems, see http://www.rsyslog.com/doc/troubleshoot.html

    #### MODULES ####

    $ModLoad imuxsock # provides support for local system logging (e.g. via logger command)
    $ModLoad imklog   # provides kernel logging support (previously done by rklogd)
    #$ModLoad immark  # provides --MARK-- message capability

    # Provides UDP syslog reception
    #$ModLoad imudp
    #$UDPServerRun 514

    # Provides TCP syslog reception
    #$ModLoad imtcp
    #$InputTCPServerRun 514


    #### GLOBAL DIRECTIVES ####

    # Use default timestamp format
    $ActionFileDefaultTemplate RSYSLOG_TraditionalFileFormat

    # File syncing capability is disabled by default. This feature is usually not required,
    # not useful and an extreme performance hit
    #$ActionFileEnableSync on

    # Include all config files in /etc/rsyslog.d/
    $IncludeConfig /etc/rsyslog.d/*.conf


    #### RULES ####

    # Log all kernel messages to the console.
    # Logging much else clutters up the screen.
    #kern.*                                                 /dev/console

    # Log anything (except mail) of level info or higher.
    # Don't log private authentication messages!
    *.info;mail.none;authpriv.none;cron.none                /var/log/messages

    # The authpriv file has restricted access.
    authpriv.*                                              /var/log/secure

    # Log all the mail messages in one place.
    mail.*                                                  -/var/log/maillog


    # Log cron stuff
    cron.*                                                  /var/log/cron

    # Everybody gets emergency messages
    *.emerg                                                 *

    # Save news errors of level crit and higher in a special file.
    uucp,news.crit                                          /var/log/spooler

    # Save boot messages also to boot.log
    local7.*                                                /var/log/boot.log


    # ### begin forwarding rule ###
    # The statement between the begin ... end define a SINGLE forwarding
    # rule. They belong together, do NOT split them. If you create multiple
    # forwarding rules, duplicate the whole block!
    # Remote Logging (we use TCP for reliable delivery)
    #
    # An on-disk queue is created for this action. If the remote host is
    # down, messages are spooled to disk and sent when it is up again.
    #$WorkDirectory /var/lib/rsyslog # where to place spool files
    #$ActionQueueFileName fwdRule1 # unique name prefix for spool files
    #$ActionQueueMaxDiskSpace 1g   # 1gb space limit (use as much as possible)
    #$ActionQueueSaveOnShutdown on # save messages to disk on shutdown
    #$ActionQueueType LinkedList   # run asynchronously
    #$ActionResumeRetryCount -1    # infinite retries if host is down
    # remote host is: name/ip:port, e.g. 192.168.0.1:514, port optional
    #*.* @@remote-host:514
    # ### end of the forwarding rule ###
    ```
1. Edit Rsyslog startup file
    ```
    # Options for rsyslogd
    # Syslogd options are deprecated since rsyslog v3.
    # If you want to use them, switch to compatibility mode 2 by "-c 2"
    # See rsyslogd(8) for more details
    SYSLOGD_OPTIONS="-c5"
    ```

1. Create `00-devo.conf` file in `/etc/rsyslog.d/00-devo.conf` with the following content.

    ```
	$ModLoad imfile
	$ModLoad immark

	$MarkMessagePeriod 60

	$WorkDirectory /var/spool/rsyslog

	$RepeatedMsgReduction off

	#Disable imuxsock rate limit
	$IMUXSockRateLimitInterval 0
	$SystemLogRateLimitInterval 0
    ```

1. Create `49-devo.conf` file in `/etc/rsyslog.d/49-devo.conf` with the following content.

	```
	$template boxunix,"<%PRI%>%timegenerated% %HOSTNAME% box.unix.%syslogtag%%msg%"

	#ActionQueue section
	$ActionQueueType                LinkedList
	$ActionQueueFileName            ltboxq1
	$ActionResumeRetryCount         -1
	$ActionQueueSaveOnShutdown      on

	*.*    @@[RELAY 1 IP ADDRESS]:13006;boxunix
	```
1. Create `45-apache.conf` file in `/etc/rsyslog.d/49-devo.conf` with the following content.
    ```
    $template apache,"<%PRI%>%timegenerated% %HOSTNAME% %syslogtag%.%HOSTNAME%: %msg%"

    # Define the input of access log
    $InputFileName /var/log/httpd/access_log
    $InputFileTag web.apache.access-clf.production.apache
    $InputFileStateFile stat-file1-ApacheAccess
    $InputFileSeverity info
    $InputFileFacility local7
    $InputFilePollInterval 1
    $InputFilePersistStateInterval 1
    $InputRunFileMonitor

    # Define the input of error log
    $InputFileName /var/log/httpd/error_log
    $InputFileTag web.apache.error.production.apache
    $InputFileStateFile stat-file1-ApacheError
    $InputFileSeverity info
    $InputFileFacility local7
    $InputFilePollInterval 1
    $InputFilePersistStateInterval 1
    $InputRunFileMonitor

    if $syslogtag contains 'web.apache.' and $syslogfacility-text == 'local7' then @@[RELAY 1 IP ADDRESS]:13006;apache
    :syslogtag, contains, "web.apache."
    
    if $syslogtag contains 'web.apache.' and $syslogfacility-text == 'local7' then @@[RELAY 2 IP ADDRESS]:13006;apache
    :syslogtag, contains, "web.apache." ~
    ```

1. Restart Rsyslog service
	```
	# service rsyslog restart
	```
  
# Verification
1. Access log

   ![photo1673943998](https://user-images.githubusercontent.com/125193/212846851-961a9661-fa13-4125-82fa-3a6b856db8dd.jpeg)
   
   ![photo1673944023](https://user-images.githubusercontent.com/125193/212846843-d9b6b153-27c5-4a7f-973e-eff0d6b6700a.jpeg)

1. Error log
  
   ![photo1673943748](https://user-images.githubusercontent.com/125193/212846349-6893ada0-3556-46a6-97fe-fc1c14485d6b.jpeg)
    
   ![photo1673943774](https://user-images.githubusercontent.com/125193/212846300-d7192ea1-2b26-4392-a1ef-bc93af1d345c.jpeg)
1. Syntax verification
    ````
    # /sbin/rsyslogd -f /etc/rsyslog.conf -N1
    rsyslogd: version 5.8.10, config validation run (level 1), master config /etc/rsyslog.conf
    rsyslogd: WARNING: rsyslogd is running in compatibility mode. Automatically generated config directives may interfer with your rsyslog.conf settings. We suggest upgrading your config and adding -c5 as the first rsyslogd option.
    rsyslogd: error -2185 skipping to action part - ignoring selector [try http://www.rsyslog.com/e/2185 ]
    rsyslogd: the last error occured in /etc/rsyslog.d/45-apache.conf, line 44:":syslogtag, contains, "web.apache.""
    rsyslogd: warning: selector line without actions will be discarded
    rsyslogd: Warning: backward compatibility layer added to following directive to rsyslog.conf: ModLoad immark
    rsyslogd: Warning: backward compatibility layer added to following directive to rsyslog.conf: MarkMessagePeriod 1200
    rsyslogd: Warning: backward compatibility layer added to following directive to rsyslog.conf: ModLoad imuxsock
    rsyslogd: End of config validation run. Bye.
    ````
1. `logger test` output
![Vivaldi_2023-01-17 17-09-55@2x](https://user-images.githubusercontent.com/125193/212856550-69e83758-1427-4874-a17d-788840406dc1.jpg)

