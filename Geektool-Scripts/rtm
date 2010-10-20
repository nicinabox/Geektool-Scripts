#! /usr/bin/perl -w

# rtm: Command-line interface to www.rememberthemilk.com
# Copyright (C) 2007  Yves Rutschle
# 
# This program is free software; you can redistribute it
# and/or modify it under the terms of the GNU General Public
# License as published by the Free Software Foundation; either
# version 2 of the License, or (at your option) any later
# version.
# 
# This program is distributed in the hope that it will be
# useful, but WITHOUT ANY WARRANTY; without even the implied
# warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR
# PURPOSE.  See the GNU General Public License for more
# details.
# 
# The full text for the General Public License is here:
# http://www.gnu.org/licenses/gpl.html

=head1 NAME

 rtm - access rememberthemilk.com

=head1 SYNOPSIS

 rtm [--filter <filter>]
     [--verbose|-v]
      --show [list|task]
      --list <listnum>
      --add <taskname>
      --complete <tasklist>
      --delete <tasklist>
      --uncomplete <tasklist>
      --undo [<action>]

 rtm --help

=head1 DESCRIPTION

B<rtm> provides a command line interface to the online TODO
list service found at C<http://www.rememberthemilk.com>
(called I<rtm> in the rest of this page).

You need to allow B<rtm> to access your I<rtm> account.
First you should call B<rtm --authorise>, which will give
you an authentication URL on C<rememberthemilk.com>. You
should then direct your browser to that URL, and allow
access. Next time you call B<rtm>, it will finish the
authentication procedure, which you should then never have
to do again.

=cut

use WebService::RTMAgent;
use Pod::Usage;
use Getopt::Long;
use Date::Manip;

use Data::Dumper;

use strict;

my $ua = new WebService::RTMAgent;
$ua->verbose('');  # /netout|netin/

# These are mine! Please request your own if you're going to build an
# application on top of the RTM module. Otherwise (if you use my own rtm script)
# just use them
$ua->api_key("b9a13da4288b2dafa17163990743e352");
$ua->api_secret("23c2132663bdd601");

$ua->no_proxy();
$ua->env_proxy;

my $res;

my ($param_getauth, $param_complete, $param_list, 
    $param_filter, $param_delete, $param_uncomplete,
    $param_show, $param_add, $param_undo,
    $help, $verbose);
GetOptions(
    'authorise'         => \$param_getauth,

    'add=s'             => \$param_add,

    'complete=s'        => \$param_complete,
    'uncomplete=s'      => \$param_uncomplete,
    'delete=s'          => \$param_delete,

    'undo:s'            => \$param_undo,

    'list=i'            => \$param_list,
    'filter=s'          => \$param_filter,
    'verbose'           => \$verbose,
    'show:s'            => \$param_show,
    'help'              => \$help,
) or die pod2usage();

die pod2usage(-verbose=>2) if defined $help;

# if no action defined, make 'show list' the default
$param_show = '' unless defined $param_add or defined
$param_complete or defined $param_uncomplete or defined
$param_delete or defined $param_show or defined $param_undo;

$param_filter ||= 'status:incomplete';
$param_filter = "filter=$param_filter";

# Returns a list of tasks depending on filter
sub task_list {
    my (@params) = @_;
    $res = $ua->tasks_getList(@params);
    die $ua->error unless defined $res;

    my @out;
    foreach my $list (@{$res->{tasks}}) {
        foreach my $taskseries (@{$list->{list}}) {
            my $list_id = $taskseries->{id};
            next unless exists $taskseries->{taskseries};
            foreach my $task (@{$taskseries->{taskseries}}) {
                my $taskseries_id = $task->{id};
                my $task_id = $task->{task}->[0]->{id};
                push @out, {
                    list_id =>$list_id, 
                    taskseries_id => $taskseries_id, 
                    task_id => $task_id,
                    name => $task->{name},
                    task => $task,  # Keep a reference to the whole data
                };
            }
        }
    } 
    my $cnt = 0;
    return map { $cnt++, $_ } sort { $a->{task_id} <=> $b->{task_id} } @out;
}

# returns a hash of
#    number => list hash
# with number garanteed to be consistent (sorted by list_id)
sub list_list {
    my $res = $ua->lists_getList;
    die $ua->error if not defined $res;

    my $cnt = 0;                
    return map { $cnt++, $_ } sort { $a->{id} <=> $b->{id} }
                @{$res->{lists}->[0]->{list}};
}

# Turns "1,4,8-12" into a list [1, 4, 8, 9, 11, 12];
sub expand_list {
    return map {  /(\d+)-(\d+)/ ? ($1..$2): $_; } split ',',$_[0];
}

=head1 OPTIONS

=over 4

=item B<--verbose|-v>

explain what is being done

=item B<--authorise>

Prints an authorisation URL. This is the first thing you
should do before using this program. You should then go to
the specified URL, log in using your username and password,
and authorise this program to use your data.

=cut

if (defined $param_getauth) {
    print($ua->get_auth_url."\n");
    exit 0;
}

$ua->init;

=item B<--filter> I<filter>

Specifies a filter to apply to the list of tasks.
See
L<http://www.rememberthemilk.com/help/answers/search/advanced.rtm>
for details on the filters.

=item B<--list> I<list_num>

Specifies a list on which to apply actions. I<list_num> is
the number returned by C<rtm --show list>.

=cut

my $list_id;
if (defined $param_list) {
    my %lists = list_list;
    die "list $param_list does not exist\n" if not exists $lists{$param_list};

    $list_id = $lists{$param_list}->{id};
    warn "working on list $list_id `$lists{$param_list}->{name}'\n" if $verbose;
}

=item B<--add> I<taskname>

Adds a task called I<taskname>. If not list is specified
with B<--list>, the task is added to the Inbox.

=cut
if (defined $param_add) {
    my $res = $ua->tasks_add(
        "name=$param_add", "parse=1",
        $list_id ? "list_id=$list_id" : "" );
    die $ua->error unless defined $res;
    print "added `$param_add'\n" if $verbose;
}

=item B<--delete> I<tasklist>

=item B<--complete> I<tasklist>

=item B<--uncomplete> I<tasklist>

Mark tasks as deleted, complete or uncomplete. I<tasklist>
is a list of task numbers as returned by --list, with the
same filter.  Several tasks can be specified separated with
comas, and ranges with dashes.

 rtm --list 2 --filter tag:bananas

I<rtm outputs a list of numbered tasks>

 rtm --list 2 --filter tag:bananas --complete 3,6,9-12

I<in that previous list, mark tasks 3, 6 and 9 to 12 as completed>

=cut

sub act_on_tasklist {
    my ($tasks, $method, $message, $list) = @_;
    foreach my $tnum (expand_list $list) {
        warn "$0: task $tnum does not exist\n" if not exists $tasks->{$tnum};
        my %task = %{$tasks->{$tnum}};
        my ($lid, $tsid, $id, $name) = 
            @task{'list_id', 'taskseries_id', 'task_id', 'name'};

        no strict 'refs';
        my $res = $ua->$method("list_id=$lid","taskseries_id=$tsid","task_id=$id");
        warn $ua->error if not defined $res;
        print "$message `$name'\n" if $verbose;
    }
}

if (defined $param_delete or defined $param_complete or
    defined $param_uncomplete) {
    my %tasks = task_list($param_filter, $list_id ?  "list_id=$list_id" : "");
    act_on_tasklist \%tasks, 'tasks_delete', 'deleted', $param_delete if defined $param_delete;
    act_on_tasklist \%tasks, 'tasks_complete', 'completed', $param_complete if defined $param_complete;
    act_on_tasklist \%tasks, 'tasks_uncomplete', 'uncompleted', $param_uncomplete if defined $param_uncomplete;
}

=item B<--show> I<list|task>

If the parameter is 'list', prints all the lists available.
If the parameter is a number, it is taken to be the task
number and that task's details are printed.
If no parameter is present, all the tasks are printed
(taking in account the filter, active list and so on, that
is).

=cut
if (defined $param_show) {
    if ($param_show eq 'list') {
        my %lists = list_list;
        foreach my $i (sort {$a <=> $b} keys %lists) {
            print "$i: $lists{$i}->{name}\n";
        }
    } else {
        my %tasks = task_list($param_filter, $list_id ?
                "list_id=$list_id" : "" );
        if ($param_show =~ /(\d+)/) {
            # There is a number in the parameter -> display
            # that task
            die "task $1 does not exist\n" unless exists $tasks{$1};
            my $task = $tasks{$1};
            my $tstruct = $task->{task}->{task}->[0];
            print "Name: $task->{name}\n";
            if ($tstruct->{due}) {
                my $date = ParseDate($tstruct->{due});
                print "Due : ".UnixDate($date,"%u")."\n";
            }
            print "Est : $tstruct->{estimate}\n" if $tstruct->{estimate};
            $tstruct = $task->{task};
            print "URL : $tstruct->{url}\n" if $tstruct->{url};
            if ($tstruct->{rrule}) {
                my $rec = $tstruct->{rrule}->[0];
                my $content = $rec->{content};
                my ($freq, $interval) = $content =~ /FREQ=(\w+);INTERVAL=(\w+)/;
                print "Rec : \L$freq, every $interval\n";
            }
            print "tags: ".(join ", ", @{$tstruct->{tags}->[0]->{tag}})."\n" 
                if $tstruct->{tags}->[0]->{tag};
            print "location: $tstruct->{location}\n" if $tstruct->{location};
            foreach my $note (@{$task->{task}->{notes}->[0]->{note}}) {
                print '#'x79;
                print "Note created $note->{created}, ".
                      "last modified $note->{modified}\n".
                print "--- $note->{title}\n".
                      "$note->{content}\n";
            }
        } else {
            foreach my $i (sort {$a <=> $b} keys %tasks) {
                print "$i: $tasks{$i}->{name}\n";
            }
        }
    }
}

=item B<--undo> [I<action>]

If no parameter is given, prints a list of the actions that
can be undone. If a parameter is given, it is the number of
the action to be undone, as found in that list. (just try
it, it's quite intuitive really).

=cut
if (defined $param_undo) {
    my $list = $ua->get_undoable;

    if ($param_undo) {
        my $t = $list->[$param_undo];
        die "action $param_undo not found\n" unless $t->{id};
        $ua->transactions_undo("transaction_id=$t->{id}") or die $ua->error;
        $ua->clear_undo($param_undo);
    } else {
        my $cnt = 0;
        map { print($cnt++, ": ", $_->{op}, "(",
                  (join ", ", 
                      grep { /=/  }  @{$_->{params}}) 
                  .")\n"); 
          } @$list;
    }

}


=back

=head1 CONFIGURATION

B<rtm> can be configured to use a proxy server. Simply
define environment variables 'https_proxy' and 'http_proxy'.
As this is pretty standard, your system might already be
configured accordingly.

=head1 EXAMPLES

=over 4

=item rtm --show

=item rtm --delete 2,5,6-9 --complete 3,12-15

List uncompleted tasks; in that list, delete tasks 2, 5, and
6 to 9, and complete tasks 3, and 12 to 15.

=item rtm --show --filter status:completed

List all completed tasks

=item rtm --filter status:completed --uncomplete 5-10

Mark previously completed tasks 5 to 10 as uncomplete.

=item rtm --show list

=item rtm --list 3 --add "Do great things"

Prints all available lists; add a task to list 3.

=head1 NOTES

"Release early, release often!"

This is work in progress. 

Bug reports and feature requests are accepted. Patches are
even better.

=head1 SEE ALSO

B<WebService::RTMAgent.pm>, which implements the B<rtm> API.

http://www.rutschle.net/rtm

=head1 AUTHOR

Written by Yves Rutschle <rtm@rutschle.net>

