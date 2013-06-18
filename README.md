# remote_syslog Ruby daemon & sender

Lightweight Ruby daemon to tail one or more log files and transmit UDP syslog
messages to a remote syslog host (centralized log aggregation).

remote_syslog generates UDP packets itself instead of depending on a system
syslog daemon, so its configuration doesn't affect system-wide
logging - syslog is just the transport.

Uses:

* collecting logs from servers & daemons which don't natively support syslog
* when reconfiguring the system logger is less convenient than a
  purpose-built daemon (e.g., automated app deployments)
* aggregating files not generated by daemons (e.g., package manager logs)

The library can also be used to generate one-off log messages from Ruby code.

Tested with the hosted log management service [Papertrail] and should work for
transmitting to any syslog server.


## Installation

Install the gem, which includes a binary called "remote_syslog":

    $ [sudo] gem install remote_syslog

Optionally, create a log_files.yml with the log file paths to read and the
host/port to log to (see examples/[log_files.yml.example][sample config]). These can also be
specified as command-line arguments (below).


## Usage

    Usage: remote_syslog [OPTION]... <FILE>...

    Options:
        -c, --configfile PATH            Path to config (/etc/log_files.yml)
        -d, --dest-host HOSTNAME         Destination syslog hostname or IP (logs.papertrailapp.com)
        -p, --dest-port PORT             Destination syslog port (514)
        -D, --no-detach                  Don't daemonize and detach from the terminal
        -f, --facility FACILITY          Facility (user)
            --hostname HOST              Local hostname to send from
        -P, --pid-dir DIRECTORY          DEPRECATED: Directory to write .pid file in
            --pid-file FILENAME          Location of the PID file (default /var/run/remote_syslog.pid)
            --parse-syslog               Parse file as syslog-formatted file
        -s, --severity SEVERITY          Severity (notice)
            --strip-color                Strip color codes
            --tls                        Connect via TCP with TLS
            --tcp                        Connect via TCP (no TLS)
            --new-file-check-interval INTERVAL
                                         Time between checks for new files

    Advanced options:
            --[no-]eventmachine-tail     Enable or disable using eventmachine-tail
            --debug-log FILE             Log internal debug messages
            --debug-level LEVEL          Log internal debug messages at level

    Common options:
        -h, --help                       Show this message
            --version                    Show version

    Example:
        $ remote_syslog -c configs/logs.yml -p 12345 /var/log/mysqld.log



## Example

Typical:

    $ remote_syslog

Daemonize and collect messages from files listed in `./config/logs.yml` as
well as the file `/var/log/mysqld.log`. Send to port `logs.papertrailapp.com:12345`:

    $ remote_syslog -c configs/logs.yml -p 12345 /var/log/mysqld.log

Stay attached to the terminal, look for and use `/etc/log_files.yml` if it
exists, write PID to `/tmp/remote_syslog.pid`, and send with facility local0
to `a.example.com:514`:

    $ remote_syslog -D -d a.example.com -f local0 --pid-file /tmp/remote_syslog.pid /var/log/mysqld.log

### Windows

Windows is not currently supported, though in certain situations it may work.

## Auto-starting at boot

The gem includes sample init files, also [available here]. You may be able to:

    $ cp examples/remote_syslog.init.d /etc/init.d/remote_syslog
    $ chmod 755 /etc/init.d/remote_syslog

And then ensure it's started at boot, either by using:

    $ sudo update-rc.d remote_syslog defaults
	
or by creating a link manually:

    $ sudo ln -s /etc/init.d/remote_syslog /etc/rc3.d/S30remote_syslog

remote_syslog will daemonize by default.

Init files: [remote_syslog.init.d] (init.d), OS X [launchd], [supervisor], Ubuntu [upstart]

## Sending messages securely ##

If the receiving system supports sending syslog over TCP with TLS, you can
pass the `--tls` option when running `remote_syslog`:

    $ remote_syslog --tls -p 1234 /var/log/mysqld.log

## Configuration

By default, the gem looks for a configuration in /etc/log_files.yml.

The gem comes with a [sample config].  Optionally:

    $ cp examples/log_files.yml.example /etc/log_files.yml

log_files.yml has filenames to log from (as an array) and hostname and port
to log to (as a hash). Wildcards are supported using * and standard shell
globbing. Filenames given on the command line are additive to those in
the config file.

Only 1 destination server is supported; the command-line argument wins.

    files:
     - /var/log/httpd/access_log
     - /var/log/httpd/error_log
     - /var/log/mysqld.log
     - /var/run/mysqld/mysqld-slow.log
    destination:
      host: logs.papertrailapp.com
      port: 12345

remote_syslog sends the name of the file without a path ("mysqld.log") as
the syslog tag (program name). RFCs 3164 and 5424 limit the tag to 32
characters. Longer filenames are truncated to 32 characters.

After changing the configuration file, restart `remote_syslog` using the 
init script or by manually killing and restarting the process. For example:

    /etc/init.d/remote_syslog restart


## Advanced Configuration (Optional)

Here's an [advanced config] which uses all options.

### Override hostname

Provide `--hostname somehostname` or use the `hostname` configuration option:

    hostname: somehostname


### Verify server certificate

Provide the public key for the remote host when using TLS:

    ssl_server_cert: syslog.crt


### Use a client certificate

Provide a client certificate when connecting via TLS:

    ssl_client_cert_chain: syslog_client.crt
    ssl_client_private_key: syslog_client.key


### Detecting new files

remote_syslog automatically detects and activates new log files that match 
its file specifiers. For example, `*.log` may be provided as a file specifier, 
and remote_syslog will detect a `some.log` file created after it was started. 
Globs are re-checked every 10 seconds. Ruby's `Dir.glob` is used.

Note: messages may be written to files in the 0-10 seconds between when the 
file is created and when the periodic glob check detects it. This data is not 
currently acted on, though the default behavior may change in the future.

Also, explicitly-provided filenames need not exist when `remote_syslog` is 
started. `remote_syslog` can be pre-configured to monitor log files which are 
created later (or may never be created).

If globs are specified on the command-line, enclose each one in single-quotes 
(`'*.log'`) so the shell passes the raw glob string to remote_syslog (rather 
than the current set of matches). This is not necessary for globs defined in 
the config file.


### Log rotation

External log rotation scripts often move or remove an existing log file
and replace it with a new one (at a new inode). The Linux standard script
[logrotate](http://iain.cx/src/logrotate/) supports a `copytruncate` config 
option.  With that option, `logrotate` will copy files, operate on the copies, 
and truncate the original so that the inode remains the same.

This comes closest to ensuring that programs watching these files (including 
`remote_syslog`) will not be affected by, or need to be notified of, the 
rotation. The only tradeoff of `copytruncate` is slightly higher disk usage 
during rotation, so we recommend this option whether or not you use 
`remote_syslog`.


### Excluding files from being sent

Provide one or more regular expressions to prevent certain files from being
matched.

    exclude_files:
	  - \.\d$
	  - .bz2
	  - .gz


### Multiple instances

Run multiple instances to support more than one message-specific file format
or to specify unique syslog hostnames.

To do that, provide an alternate PID path as a command-line option to the 
additional instance(s). For example:

    --pid-file /var/run/remote_syslog_2.pid


### Parse fields from log messages

Rarely needed. Usually only used when remote_syslog is watching files
generated by syslogd (rather than by apps), like ``/var/log/messages``.

remote_syslog can parse the program and hostname from the log line. When one
file contains logs from multiple programs (like with syslog), the log line
may include text that is not part of the log message, like a timestamp,
hostname, or program name. remote_syslog will extract those and use them in
the corresponding syslog packet fields.

To do that, use the config file option `parse_fields` with the name of a
format supported by remote_syslog, or your own regex. Included format names
are `syslog` and `rfc3339`. For example:

    parse_fields: syslog

The included `syslog` format uses the regex `(\w+ \d+ \S+) (\S+) ([^:]+): (.*)`
to parse standard syslog lines like this:

    Jul 18 08:25:08 hostname programname[1234]: The log message

The included `rfc3339` format uses the regex `(\S+) (\S+) ([^: ]+):? (.*)` to
parse syslog lines with high-precision RFC 3339 timestamps, like this:

    2011-07-16T08:25:08.651413-07:00 hostname programname[1234]: The log message

To parse a format other than those, provide your own regex. It should include
4 backreferences to parse, in order: timestamp, system name, program name,
message.

Match and return empty strings for any empty positions where the log line
doesn't provide a value. For example, given the log message:

    something-meaningless The log message

One could use a regex to ignore "something-meaningless" (and not to extract
a program or hostname). To ignore that prefix and return 3 empty values
then the log message, use parse_fields with this regex:

    parse_fields: "something-meaningless ()()()(.*)"

Per-file regexes are not supported. Run multiple instances with different
config files.


### Excluding lines matching a pattern

There may be certain log messages that you do not want to be sent.  These may
repetitive log lines that are "noise" that you might not be able to filter out
easily from the respective application.  To filter these lines, use the
exclude_patterns with an array or regexes:

    exclude_patterns:
     - exclude this
     - \d+ things

### Prepending a string to log messages

Use `prepend` to prepend a string to every log message before
transmitting.  The string is prepended to the log message body, as if it
occurred at the start of every log file line. Include a trailing space
if desired.

Examples:

    prepend: important: 

or:

    prepend: cafebabe-1024-4096-badd-1234abcd1234 

### Choosing app name

remote_syslog uses the log file name (like "access_log") as the syslog 
program name, or what the syslog RFCs call the "tag." This is ideal unless 
remote_syslog watches many files that have the same name.

In that case, tell remote_syslog to set another program name by creating 
symbolic link to the generically-named file:

    cd /path/to/logs
    ln -s generic_name.log unique_name.log

Point remote_syslog at unique_name.log. It will use that as the program name.


## Troubleshooting

Two commands are particularly useful for observing `remote_syslog`
behavior.  First, its own debugging:

    remote_syslog --debug-level DEBUG --debug-log remote_syslog.log

This will write internal operations to the file `remote_syslog.log`.

Second, strace or ktrace shows the interaction between `remote_syslog`
and the OS. To run `strace` against an existing `remote_syslog` instance
(process ID 12345):

     strace -fp 12345 -s 500

Feel free to ask questions or report bugs.


## Reporting bugs

1. See whether the issue has already been reported: <https://github.com/papertrail/remote_syslog/issues/>
2. If you don't find one, create an issue with a repro case.


## Contributing

Once you've made your great commits:

1. [Fork][fk] remote_syslog
2. Create a topic branch - `git checkout -b my_branch`
3. Commit the changes without changing the Rakefile or other files unrelated to your enhancement.
4. Push to your branch - `git push origin my_branch`
5. Create a Pull Request or an [Issue][is] with a link to your branch
6. That's it!

[sample config]: https://github.com/papertrail/remote_syslog/blob/master/examples/log_files.yml.example
[init files]: https://github.com/papertrail/remote_syslog/blob/master/examples/
[remote_syslog.init.d]: https://github.com/papertrail/remote_syslog/blob/master/examples/remote_syslog.init.d
[launchd]: https://github.com/papertrail/remote_syslog/blob/master/examples/com.papertrailapp.remote_syslog.plist
[supervisor]: https://github.com/papertrail/remote_syslog/blob/master/examples/remote_syslog.supervisor.conf
[upstart]: https://github.com/papertrail/remote_syslog/blob/master/examples/remote_syslog.upstart.conf
[advanced config]: https://github.com/papertrail/remote_syslog/blob/master/examples/log_files.yml.example.advanced
[fk]: http://help.github.com/forking/
[is]: https://github.com/papertrail/remote_syslog/issues/
[Papertrail]: http://papertrailapp.com/
