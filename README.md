# Zimbra Daily Mail Report — Truncation Bug Analysis & Fix

---

## Problem Statement

The standard Zimbra Daily Mail Report generation script (`zmdailyreport`) produces a truncated email on days when the **"message reject detail — RCPT"** section is very long. The section starts but never completes, and all subsequent sections of the report are missing.

---

## Script Architecture

The script uses **two pipes and a fork** to pass log data through `pflogsumm.pl`:

| Pipe | Direction | Purpose |
|------|-----------|---------|
| `PFIN` / `PFOUT` | Parent writes → Child reads as STDIN | Feed today's log lines to `pflogsumm.pl` |
| `MIN` / `MOUT` | Child writes → Parent reads | Capture `pflogsumm.pl` report output |

**Parent sequence** (after fork):

1. Write all matching log lines to `PFOUT`
2. `close(PFOUT)`
3. Read all output from `MIN` into `@Report`
4. Send or print `@Report`

**Child sequence** (after fork):

1. Redirect STDOUT to `MOUT`
2. Run `pflogsumm.pl` via `do` (reads STDIN = `PFIN`, writes STDOUT = `MOUT`)
3. `close(STDOUT)`

---

## Root Cause: Classic Two-Pipe Deadlock

The deadlock requires **both pipe buffers to be full simultaneously**:

| Condition | Effect |
|-----------|--------|
| `PFOUT` buffer full | Parent **blocks** writing log data |
| `MOUT` buffer full | `pflogsumm.pl` **blocks** writing report output |

If `pflogsumm.pl` writes **any output to `MOUT` while still reading from `PFIN`** (e.g., it flushes a section header, or the Zimbra-modified version emits some sections before finishing input), both processes deadlock:

- `pflogsumm.pl` is blocked trying to write to `MOUT` (full — parent not reading yet)
- Parent is blocked trying to write to `PFOUT` (full — `pflogsumm.pl` not reading)
- Neither can proceed → **report is truncated at exactly that point**

This matches the symptom: the "message reject detail RCPT" section is the largest output section. On days with heavy rejection activity, this section alone can push `MOUT`'s kernel pipe buffer (typically 64 KB) to its limit while the parent is still mid-write to `PFOUT`.

---

## The Fix

Eliminate the input pipe (`PFIN` / `PFOUT`) entirely by writing the log data to a **temporary file** instead. `pflogsumm.pl` then reads from the temp file (no pipe pressure on writes), and only the one-directional output pipe (`MIN` / `MOUT`) remains — which has no deadlock risk because the parent is always reading it by the time `pflogsumm.pl` writes.

### Changes summary

1. **Add `use File::Temp qw(tempfile)`** — the only new `use` statement.
2. **Remove `pipe(PFIN, PFOUT)`** and the three lines that redirected STDIN to PFIN (`fileno`, `close(STDIN)`, `open(STDIN, "<&=$fhin")`).
3. **Add `my ($tmpfh, $tmpfile) = tempfile("zmdailyreport.XXXXXX", DIR => "/tmp", UNLINK => 0)`** in their place. `UNLINK => 0` disables auto-cleanup so the file is explicitly removed after use.
4. **Move all log writing before the fork** — the parent writes log lines to the tempfile and closes it before calling `fork()`, ensuring `pflogsumm.pl` always reads a complete dataset with no race condition.
5. **Parent block**: replace all `print PFOUT $line` with `print $tmpfh $line`; replace `close(PFOUT)` with `close($tmpfh)`. After draining `MIN`, call `unlink $tmpfile` to clean up.
6. **Child block**: remove `close(PFOUT)`; replace the old STDIN dup with `open(STDIN, "<", $tmpfile)`; add `exit 0` after `close(STDOUT)`.

---

## Updated Script

See `zmdailyreport-updated_v2` in this repository.

```perl
#!/usr/bin/perl
#
# ***** BEGIN LICENSE BLOCK *****
# Zimbra Collaboration Suite Server
# Copyright (C) 2006, 2007, 2008, 2009, 2010, 2012, 2013, 2014, 2015, 2016 Synacor, Inc.
#
# This program is free software: you can redistribute it and/or modify it under
# the terms of the GNU General Public License as published by the Free Software Foundation,
# version 2 of the License.
#
# This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY;
# without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
# See the GNU General Public License for more details.
# You should have received a copy of the GNU General Public License along with this program.
# If not, see <https://www.gnu.org/licenses/>.
# ***** END LICENSE BLOCK *****
# Modified 23 April 2026 by Mission Critical Email LLC  to support very long Daily Mail Reports
# and prevent such reports from otherwise getting truncated.
=head1 NAME

zmdailyreport - report on email usage

=head1 SYNOPSIS

  zmdailyreport [options]

  Options:
    --mail                 Send report via email, default is stdout
    --help                 This help message
    --user                 User to deliver report to, default is localconfig smtp_destination

=cut
use warnings;
use strict;
use Pod::Usage qw(pod2usage);
use lib "/opt/zimbra/common/lib/perl5";
use Zimbra::Util::Common;
use Zimbra::Mon::Logger;
use Getopt::Long qw(GetOptions);
use Mail::Mailer ();
use POSIX qw(strftime setlocale LC_TIME);
use Time::Local qw(timelocal);
use File::Temp qw(tempfile);

my ( %Opt, $Debug );

GetOptions( \%Opt, "mailreport", "debug", "user=s", "help" )
  or pod2usage( -exitval => 2 );

pod2usage( -exitval => 1, -verbose => 0 ) if ( $Opt{help} );

$Debug = $Opt{debug};

setlocale(LC_TIME, "C");

my $stdout;
open($stdout, ">&STDOUT") || die $!;

my ($tmpfh, $tmpfile) = tempfile(
    "zmdailyreport.XXXXXX",
    DIR    => "/tmp",
    UNLINK => 0
);

my $today = strftime("%b %e ", localtime);
my $datestring = strftime("%F", localtime);
my $ymd = strftime("%Y%m%d", localtime);

my $found = 0;
my $old_log;

if (-f "/var/log/zimbra.log.1.gz") {
    $old_log = "gzip -dc /var/log/zimbra.log.1.gz |";
} elsif (-f "/var/log/zimbra.log-$ymd.gz") {
    $old_log = "gzip -dc /var/log/zimbra.log-$ymd.gz |";
} elsif (-f "/var/log/zimbra.log.1.bz2") {
    $old_log = "bzip2 -dc /var/log/zimbra.log.1.bz2 |";
} elsif (-f "/var/log/zimbra.log-$ymd.bz2") {
    $old_log = "bzip2 -dc /var/log/zimbra.log-$ymd.bz2 |";
}

if (defined($old_log)) {
    if (open(LOG, $old_log)) {
        while (<LOG>) {
            my $line = $_;
            if ($line =~ /$today/o) {
                print $tmpfh $line;
                $found = 1;
            } elsif ($found) {
                last; # don't scan any further, we've exhausted today's logs
            }
        }
        close(LOG);
    }
}

open(LOG, "</var/log/zimbra.log") || die $!;
$found = 0;
while (<LOG>) {
    my $line = $_;
    if ($line =~ /$today/o) {
        print $tmpfh $line;
        $found = 1;
    } elsif ($found) {
        last; # don't scan any further, we've exhausted today's logs
    }
}
close(LOG);

close($tmpfh) || die "Could not close temp file $tmpfile: $!";

pipe(MIN, MOUT) || die $!;

if (fork()) {
    close(MOUT);

    my @Report;
    while (<MIN>) {
        push(@Report, $_);
    }
    close(MIN);

    unlink $tmpfile;

    if ($Opt{mailreport}) {
        sendEmailReport(day => $datestring, data => \@Report);
    } else {
        print @Report;
    }
} else {
    close(MIN);

    open(STDIN, "<", $tmpfile) || die "Could not open temp file $tmpfile: $!";
    open(STDOUT, ">&MOUT") || die $!;

    my $user_limit = getLocalConfig('zimbra_mtareport_max_users') || 50;
    my $host_limit = getLocalConfig('zimbra_mtareport_max_hosts') || 50;

    $user_limit = 50 if $user_limit !~ /^\d+/;
    $host_limit = 50 if $host_limit !~ /^\d+/;

    @ARGV = ('-u', $user_limit, '-h', $host_limit);
    do '/opt/zimbra/common/bin/pflogsumm.pl';

    close(STDOUT);
    exit 0;
}

{    # avoid making $LocalConfig global
    my $LocalConfig;

    sub getLocalConfig {
        my $key = shift;
        $LocalConfig = loadLocalConfig() unless ($LocalConfig);
        return $LocalConfig->{$key};
    }
}

sub loadLocalConfig {
    my %conf;

    open( CONF, "/opt/zimbra/bin/zmlocalconfig -s -q -m shell |" )
      or die("Open local config failed: $!\n");

    while (<CONF>) {
        chomp;
        my ( $key, $val ) = split( /=/, $_, 2 );
        $val =~ s/';$//;
        $val =~ s/^'//;
        $conf{$key} = $val;
    }
    die("No data returned from local config") unless ( keys %conf );
    return \%conf;
}

sub getLdapConfigValue {
    my $attrib = shift;

    return (undef) unless ($attrib);

    my $val = getLdapServerConfigValue($attrib);
    print "Server value for $attrib: $val\n" if $Debug;
    if ($val eq "") {
      $val = getLdapGlobalConfigValue($attrib);
      print "Global value for $attrib: $val\n" if $Debug;
    }
    return $val;
}

sub getLdapServerConfigValue {
    my $attrib = shift;

    return (undef) unless ($attrib);
    my $server = getLocalConfig("zimbra_server_hostname");

    open( CONF, "/opt/zimbra/bin/zmprov -l gs $server '$attrib' |" )
      or die("Open global config failed: $!");

    my ( $key, $val );
    while (<CONF>) {
        chomp;
        next if (/^#/);
        ( $key, $val ) = split( /:\s*/, $_, 2 );
        last if ($val);
    }
    return $val;
}

sub getLdapGlobalConfigValue {
    my $attrib = shift;

    return (undef) unless ($attrib);

    open( CONF, "/opt/zimbra/bin/zmprov -l gcf '$attrib' |" )
      or die("Open global config failed: $!");

    my ( $key, $val );
    while (<CONF>) {
        chomp;
        ( $key, $val ) = split( /:\s*/, $_, 2 );
        last if ($val);
    }
    return $val;
}

sub sendEmailReport {
    my %arg = @_;

    my $data = $arg{data};

    my $subject      = "Daily mail report for $arg{day}";
    my $from_address = getLocalConfig("smtp_source");
    my $to_address;

    if ($Opt{user}) {
      $to_address = $Opt{user};
    } else {
      $to_address = getLocalConfig("smtp_destination");
    }

    my $smtphost = getLdapConfigValue("zimbraSmtpHostname") || "localhost";
    my $smtpport = getLdapConfigValue("zimbraSmtpPort") || 25;

    warn("DEBUG: Sending daily report to $to_address via $smtphost\n")
      if ($Debug);

    eval {
        $ENV{MAILADDRESS} = $from_address;
        my $mailer = Mail::Mailer->new( "smtp", Server => $smtphost, Port => $smtpport );
        $mailer->open(
            {
                From    => $from_address,
                To      => $to_address,
                Subject => $subject,
            }
        );
        print $mailer @$data;
        $mailer->close();
    };

    if ($@) {
        logError("Failed to email report using SMTP via '$smtphost': $@\n");
    } else {
        warn("DEBUG: Email report sent to $to_address\n") if ($Debug);
    }
}

sub logError {
    my $msg = shift;
    print STDERR $msg;
    return;
}
```
