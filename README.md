# How to make an offline portable repository for Ubuntu
Ubuntu Offline Complete Portable Repo How to </br>

Old method: debmirror </br> 
New method: apt-mirror </br>

Old: </br>
Pre-Ubuntu 16 </br>
New: </br>
Pre-Rust </br>
Pre-Ubuntu 25 </br> 

[Old method](https://web.archive.org/web/20160320113042/https://ubuntuforums.org/showthread.php?t=352460) includes a tutorial to split downloaded repository into several [DVD size images](https://web.archive.org/web/20160320113042/https://ubuntuforums.org/showthread.php?t=352460) </br>
could work as a guide if using [Blu-ray](https://en.wikipedia.org/wiki/Blu-ray_Disc_recordable) 25 GB (1-layer), 50 / 66 GB (2-layer) & 100 / 128 GB (4-Layer BDXL) </br>

Ubuntu 20.04.x LTS Complete Repository is 242,453 items, 708.3 GB </br>
including sources & backports, Not proposed. </br>

In [Web Archive](https://archive.org/details/@chris85?query=ubuntu+repo) there are 25 repos Ubuntu 17 and older, Not all, Not 14.04 (Trusty Tar) </br>
most have [.torrent](https://www.qbittorrent.org/download) but Not all. </br>
Unknown if include sources & backports. </br>

There are other mirrors like: </br>
[packages.ubuntu.com](https://packages.ubuntu.com/) </br>
[pkgs.org](https://pkgs.org/) </br>

personally: </br>
i prefer .AppImages when possible, </br>
but Not everything is available as .Appimages </br>
Repos are a necesary Evil. </br>
Snap is ok </br>
Flatpak, avoid when possible, </br>
better compiling from source when possible. </br>
Flatpak follows [Wirth's_law](https://en.wikipedia.org/wiki/Wirth's_law) No mercy. </br>

# New Method

Portable Offline Ubuntu 20.04.x LTS Repository </br>

```
install:
Ubuntu 20.04.x LTS

$ sudo apt install apt-mirror

edit:
$ sudo tea /usr/bin/apt-mirror
$ sudo tea /etc/apt/mirror.list

unless you have a >2TB NVMe/SSD as /dev/sda1
change:
 base_path
to external HDD.
Not needed:
/var/spool/apt-mirror
$ ls -l
drwxr-xr-x 5 apt-mirror apt-mirror 4096 May 14 15:36 apt-mirror

Architecture:
$ dpkg --print-architecture 2>/dev/null
amd64

pre-downloaded packages "if Not clean install" stored in:
$ cd /var/cache/apt/archives/
$ ls >> ~/Downloads/var-cache-apt-archives.txt

IF using a different Ubuntu version,
requires to change deb sources in configuration files.
or install a different Ubuntu version.
```

$ tea /etc/apt/mirror.list </br>

```
############# config ##################
#
set base_path    /media/user/ext4-1.7TB/var/spool/apt-mirror
#
# set mirror_path  $base_path/mirror
# set skel_path    $base_path/skel
# set var_path     $base_path/var
# set cleanscript $var_path/clean.sh
# set defaultarch  <running host architecture>
# set postmirror_script $var_path/postmirror.sh
# set run_postmirror 0
set nthreads     20
set _tilde 0
#
############# end config ##############

deb http://archive.ubuntu.com/ubuntu focal main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu focal-security main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu focal-updates main restricted universe multiverse
#deb http://archive.ubuntu.com/ubuntu focal-proposed main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu focal-backports main restricted universe multiverse

deb-src http://archive.ubuntu.com/ubuntu focal main restricted universe multiverse
deb-src http://archive.ubuntu.com/ubuntu focal-security main restricted universe multiverse
deb-src http://archive.ubuntu.com/ubuntu focal-updates main restricted universe multiverse
#deb-src http://archive.ubuntu.com/ubuntu focal-proposed main restricted universe multiverse
deb-src http://archive.ubuntu.com/ubuntu focal-backports main restricted universe multiverse

clean http://archive.ubuntu.com/ubuntu
```

$ tea /usr/bin/apt-mirror </br>

```
#!/usr/bin/perl

=pod

=head1 NAME

apt-mirror - apt sources mirroring tool

=head1 SYNOPSIS

apt-mirror [configfile]

=head1 DESCRIPTION

A small and efficient tool that lets you mirror a part of or
the whole Debian GNU/Linux distribution or any other apt sources.

Main features:
 * It uses a config similar to APT's F<sources.list>
 * It's fully pool compliant
 * It supports multithreaded downloading
 * It supports multiple architectures at the same time
 * It can automatically remove unneeded files
 * It works well on an overloaded Internet connection
 * It never produces an inconsistent mirror including while mirroring
 * It works on all POSIX compliant systems with Perl and wget

=head1 COMMENTS

apt-mirror uses F</etc/apt/mirror.list> as a configuration file.
By default it is tuned to official Debian or Ubuntu mirrors. Change
it for your needs.

After you setup the configuration file you may run as root:

    # su - apt-mirror -c apt-mirror

Or uncomment the line in F</etc/cron.d/apt-mirror> to enable daily mirror updates.

=head1 FILES

F</etc/apt/mirror.list>
        Main configuration file

F</etc/cron.d/apt-mirror>
        Cron configuration template

F</var/spool/apt-mirror/mirror>
        Mirror places here

F</var/spool/apt-mirror/skel>
        Place for temporarily downloaded indexes

F</var/spool/apt-mirror/var>
        Log files placed here. URLs and MD5 checksums also here.

=head1 CONFIGURATION EXAMPLES

The mirror.list configuration supports many options, the file is well commented explaining each option.
Here are some sample mirror configuration lines showing the various supported ways:

Normal:
deb http://example.com/debian stable main contrib non-free

Arch Specific: (many other architectures are supported)
deb-powerpc http://example.com/debian stable main contrib non-free

HTTP and FTP Auth or non-standard port:
deb http://user:pass@example.com:8080/debian stable main contrib non-free

HTTPS with sending Basic HTTP authentication information (plaintext username and password) for all requests:
(this was default behaviour of Wget 1.10.2 and prior and is needed for some servers with new version of Wget)
set auth_no_challenge 1
deb https://user:pass@example.com:443/debian stable main contrib non-free

HTTPS without checking certificate:
set no_check_certificate 1
deb https://example.com:443/debian stable main contrib non-free

Source Mirroring:
deb-src http://example.com/debian stable main contrib non-free

=head1 AUTHORS

Dmitry N. Hramtsov E<lt>hdn@nsu.ruE<gt>
Brandon Holtsclaw E<lt>me@brandonholtsclaw.comE<gt>

=cut

use warnings;
use strict;
use File::Copy;
use File::Compare;
use File::Path qw(make_path);
use File::Basename;
use Fcntl qw(:flock);

my $config_file;

my %config_variables = (
    "defaultarch" => `dpkg --print-architecture 2>/dev/null` || 'i386',
    "nthreads"    => 20,
    "base_path"   => '/media/user/ext4-1.7TB/var/spool/apt-mirror',
    "mirror_path" => '$base_path/mirror',
    "skel_path"   => '$base_path/skel',
    "var_path"    => '$base_path/var',
    "cleanscript" => '$var_path/clean.sh',
    "_contents"   => 1,
    "_autoclean"  => 0,
    "_tilde"      => 0,
    "limit_rate"  => '100m',
    "run_postmirror"       => 1,
    "auth_no_challenge"    => 0,
    "no_check_certificate" => 0,
    "unlink"               => 0,
    "postmirror_script"    => '$var_path/postmirror.sh',
    "use_proxy"            => 'off',
    "http_proxy"           => '',
    "https_proxy"          => '',
    "proxy_user"           => '',
    "proxy_password"       => ''
);

my @config_binaries = ();
my @config_sources  = ();

my @index_urls;
my @childrens       = ();
my %skipclean       = ();
my %clean_directory = ();

######################################################################################
## Setting up $config_file variable

$config_file = "/etc/apt/mirror.list";    # Default value
if ( $_ = shift )
{
    die("apt-mirror: invalid config file specified") unless -e $_;
    $config_file = $_;
}

chomp $config_variables{"defaultarch"};

######################################################################################
## Common subroutines

sub round_number
{
    my $n = shift;
    my $minus = $n < 0 ? '-' : '';
    $n = abs($n);
    $n = int( ( $n + .05 ) * 10 ) / 10;
    $n .= '.0' unless $n =~ /\./;
    $n .= '0' if substr( $n, ( length($n) - 1 ), 1 ) eq '.';
    chop $n if $n =~ /\.\d\d0$/;
    return "$minus$n";
}

sub format_bytes
{
    my $bytes     = shift;
    my $bytes_out = '0';
    my $size_name = 'bytes';
    my $KiB       = 1024;
    my $MiB       = 1024 * 1024;
    my $GiB       = 1024 * 1024 * 1024;

    if ( $bytes >= $KiB )
    {
        $bytes_out = $bytes / $KiB;
        $size_name = 'KiB';
        if ( $bytes >= $MiB )
        {
            $bytes_out = $bytes / $MiB;
            $size_name = 'MiB';
            if ( $bytes >= $GiB )
            {
                $bytes_out = $bytes / $GiB;
                $size_name = 'GiB';
            }
        }
        $bytes_out = round_number($bytes_out);
    }
    else
    {
        $bytes_out = $bytes;
        $size_name = 'bytes';
    }

    return "$bytes_out $size_name";
}

sub get_variable
{
    my $value = $config_variables{ shift @_ };
    my $count = 16;
    while ( $value =~ s/\$(\w+)/$config_variables{$1}/xg )
    {
        die("apt-mirror: too many substitution while evaluating variable") if ( $count-- ) < 0;
    }
    return $value;
}

sub quoted_path
{
    my $path = shift;
    $path =~ s/'/'\\''/g;
    return "'" . $path . "'";
}

sub lock_aptmirror
{
    open( LOCK_FILE, '>', get_variable("var_path") . "/apt-mirror.lock" );
    my $lock = flock( LOCK_FILE, LOCK_EX | LOCK_NB );
    if ( !$lock )
    {
        die("apt-mirror is already running, exiting");
    }
}

sub unlock_aptmirror
{
    close(LOCK_FILE);
    unlink( get_variable("var_path") . "/apt-mirror.lock" );
}

sub download_urls
{
    my $stage = shift;
    my @urls;
    my $i = 0;
    my $pid;
    my $nthreads = get_variable("nthreads");
    my @args     = ();
    local $| = 1;

    @urls = @_;
    $nthreads = @urls if @urls < $nthreads;

    if ( get_variable("auth_no_challenge") == 1 )    { push( @args, "--auth-no-challenge" ); }
    if ( get_variable("no_check_certificate") == 1 ) { push( @args, "--no-check-certificate" ); }
    if ( get_variable("unlink") == 1 )               { push( @args, "--unlink" ); }
    if ( length( get_variable("use_proxy") ) && ( get_variable("use_proxy") eq 'yes' || get_variable("use_proxy") eq 'on' ) )
    {
        if ( length( get_variable("http_proxy") ) || length( get_variable("https_proxy") ) ) { push( @args, "-e use_proxy=yes" ); }
        if ( length( get_variable("http_proxy") ) ) { push( @args, "-e http_proxy=" . get_variable("http_proxy") ); }
        if ( length( get_variable("https_proxy") ) ) { push( @args, "-e https_proxy=" . get_variable("https_proxy") ); }
        if ( length( get_variable("proxy_user") ) ) { push( @args, "-e proxy_user=" . get_variable("proxy_user") ); }
        if ( length( get_variable("proxy_password") ) ) { push( @args, "-e proxy_password=" . get_variable("proxy_password") ); }
    }
    print "Downloading " . scalar(@urls) . " $stage files using $nthreads threads...\n";

    while ( scalar @urls )
    {
        my @part = splice( @urls, 0, int( @urls / $nthreads ) );
        open URLS, ">" . get_variable("var_path") . "/$stage-urls.$i" or die("apt-mirror: can't write to intermediate file ($stage-urls.$i)");
        foreach (@part) { print URLS "$_\n"; }
        close URLS or die("apt-mirror: can't close intermediate file ($stage-urls.$i)");

        $pid = fork();

        die("apt-mirror: can't do fork in download_urls") if !defined($pid);

        if ( $pid == 0 )
        {
            exec 'wget', '--no-cache', '--limit-rate=' . get_variable("limit_rate"), '-t', '5', '-r', '-N', '-l', 'inf', '-o', get_variable("var_path") . "/$stage-log.$i", '-i', get_variable("var_path") . "/$stage-urls.$i", @args;

            # shouldn't reach this unless exec fails
            die("\n\nCould not run wget, please make sure its installed and in your path\n\n");
        }

        push @childrens, $pid;
        $i++;
        $nthreads--;
    }

    print "Begin time: " . localtime() . "\n[" . scalar(@childrens) . "]... ";
    while ( scalar @childrens )
    {
        my $dead = wait();
        @childrens = grep { $_ != $dead } @childrens;
        print "[" . scalar(@childrens) . "]... ";
    }
    print "\nEnd time: " . localtime() . "\n\n";
}

## Parse config

sub parse_config_line
{
    my $pattern_deb_line = qr/^[\t ]*(?<type>deb-src|deb)(?:-(?<arch>[\w\-]+))?[\t ]+(?:\[(?<options>[^\]]+)\][\t ]+)?(?<uri>[^\s]+)[\t ]+(?<components>.+)$/;
    my $line = $_;
    my %config;
    if ( $line =~ $pattern_deb_line ) {
        $config{'type'} = $+{type};
        $config{'arch'} = $+{arch};
        $config{'options'} = $+{options} ? $+{options} : "";
        $config{'uri'} = $+{uri};
        $config{'components'} = $+{components};
        if ( $config{'options'} =~ /arch=((?<arch>[\w\-]+)[,]*)/g ) {
            $config{'arch'} = $+{arch};
        }
        $config{'components'} = [ split /\s+/, $config{'components'} ];
    } elsif ( $line =~ /set[\t ]+(?<key>[^\s]+)[\t ]+(?<value>"[^"]+"|'[^']+'|[^\s]+)/ ) {
        $config{'type'} = 'set';
        $config{'key'} = $+{key};
        $config{'value'} = $+{value};
        $config{'value'} =~ s/^'(.*)'$/$1/;
        $config{'value'} =~ s/^"(.*)"$/$1/;
    } elsif ( $line =~ /(?<type>clean|skip-clean)[\t ]+(?<uri>[^\s]+)/ ) {
        $config{'type'} = $+{type};
        $config{'uri'} = $+{uri};
    }

    return %config;
}

open CONFIG, "<$config_file" or die("apt-mirror: can't open config file ($config_file)");
while (<CONFIG>)
{
    next if /^\s*#/;
    next unless /\S/;
    my $line = $_;
    my %config_line = parse_config_line;

    if ( $config_line{'type'} eq "set" ) {
        $config_variables{ $config_line{'key'} } = $config_line{'value'};
        next;
    } elsif ( $config_line{'type'} eq "deb" ) {
        my $arch = $config_line{'arch'};
        $arch = get_variable("defaultarch") if ! defined $config_line{'arch'};
        push @config_binaries, [ $arch, $config_line{'uri'}, @{$config_line{'components'}} ];
        next;
    } elsif ( $config_line{'type'} eq "deb-src" ) {
        push @config_sources, [ $config_line{'uri'}, @{$config_line{'components'}} ];
        next;
    } elsif ( $config_line{'type'} =~ /(skip-clean|clean)/ ) {
        my $link = $config_line{'uri'};
        $link =~ s[^(\w+)://][];
        $link =~ s[/$][];
        $link =~ s[~][%7E]g if get_variable("_tilde");
        if ( $config_line{'type'} eq "skip-clean" ) {
            $skipclean{ $link } = 1;
        } elsif ( $config_line{'type'} eq "clean" ) {
            $clean_directory{ $link } = 1;
        }
        next;
    }

    die("apt-mirror: invalid line in config file ($.: $line ...)");
}
close CONFIG;

die("Please explicitly specify 'defaultarch' in mirror.list") unless get_variable("defaultarch");

######################################################################################
## Create the 3 needed directories if they don't exist yet
my @needed_directories = ( get_variable("mirror_path"), get_variable("skel_path"), get_variable("var_path") );
foreach my $needed_directory (@needed_directories)
{
    unless ( -d $needed_directory )
    {
        make_path($needed_directory) or die("apt-mirror: can't create $needed_directory directory");
    }
}
#
#######################################################################################

lock_aptmirror();

######################################################################################
## Skel download

my %urls_to_download = ();
my ( $url, $arch );

sub remove_double_slashes
{
    local $_ = shift;
    while (s[/\./][/]g)                { }
    while (s[(?<!:)//][/]g)            { }
    while (s[(?<!:/)/[^/]+/\.\./][/]g) { }
    s/~/\%7E/g if get_variable("_tilde");
    return $_;
}

sub add_url_to_download
{
    my $url = remove_double_slashes(shift);
    $urls_to_download{$url} = shift;
}

foreach (@config_sources)
{
    my ( $uri, $distribution, @components ) = @{$_};

    if (@components)
    {
        $url = $uri . "/dists/" . $distribution . "/";

        add_url_to_download( $url . "InRelease" );
        add_url_to_download( $url . "Release" );
        add_url_to_download( $url . "Release.gpg" );
        foreach (@components)
        {
            add_url_to_download( $url . $_ . "/source/Release" );
            add_url_to_download( $url . $_ . "/source/Sources.gz" );
            add_url_to_download( $url . $_ . "/source/Sources.bz2" );
            add_url_to_download( $url . $_ . "/source/Sources.xz" );
        }
    }
    else
    {
        add_url_to_download( $uri . "/$distribution/Release" );
        add_url_to_download( $uri . "/$distribution/Release.gpg" );
        add_url_to_download( $uri . "/$distribution/Sources.gz" );
        add_url_to_download( $uri . "/$distribution/Sources.bz2" );
        add_url_to_download( $uri . "/$distribution/Sources.xz" );
    }
}

foreach (@config_binaries)
{
    my ( $arch, $uri, $distribution, @components ) = @{$_};

    if (@components)
    {
        $url = $uri . "/dists/" . $distribution . "/";

        add_url_to_download( $url . "InRelease" );
        add_url_to_download( $url . "Release" );
        add_url_to_download( $url . "Release.gpg" );
        if ( get_variable("_contents") )
        {
            add_url_to_download( $url . "Contents-" . $arch . ".gz" );
            add_url_to_download( $url . "Contents-" . $arch . ".bz2" );
            add_url_to_download( $url . "Contents-" . $arch . ".xz" );
        }
        foreach (@components)
        {
            if ( get_variable("_contents") )
            {
                add_url_to_download( $url . $_ . "/Contents-" . $arch . ".gz" );
                add_url_to_download( $url . $_ . "/Contents-" . $arch . ".bz2" );
                add_url_to_download( $url . $_ . "/Contents-" . $arch . ".xz" );
            }
            add_url_to_download( $url . $_ . "/binary-" . $arch . "/Release" );
            add_url_to_download( $url . $_ . "/binary-" . $arch . "/Packages.gz" );
            add_url_to_download( $url . $_ . "/binary-" . $arch . "/Packages.bz2" );
            add_url_to_download( $url . $_ . "/binary-" . $arch . "/Packages.xz" );
            add_url_to_download( $url . $_ . "/i18n/Index" );
        }
    }
    else
    {
        add_url_to_download( $uri . "/$distribution/Release" );
        add_url_to_download( $uri . "/$distribution/Release.gpg" );
        add_url_to_download( $uri . "/$distribution/Packages.gz" );
        add_url_to_download( $uri . "/$distribution/Packages.bz2" );
        add_url_to_download( $uri . "/$distribution/Packages.xz" );
    }
}

chdir get_variable("skel_path") or die("apt-mirror: can't chdir to skel");
@index_urls = sort keys %urls_to_download;
download_urls( "index", @index_urls );

foreach ( keys %urls_to_download )
{
    s[^(\w+)://][];
    s[~][%7E]g if get_variable("_tilde");
    $skipclean{$_} = 1;
    $skipclean{$_} = 1 if s[\.gz$][];
    $skipclean{$_} = 1 if s[\.bz2$][];
    $skipclean{$_} = 1 if s[\.xz$][];
}

######################################################################################
## Translation index download

%urls_to_download = ();

sub sanitise_uri
{
    my $uri = shift;
    $uri =~ s[^(\w+)://][];
    $uri =~ s/^([^@]+)?@?// if $uri =~ /@/;
    $uri =~ s&:\d+/&/&;                       # and port information
    $uri =~ s/~/\%7E/g if get_variable("_tilde");
    return $uri;
}

sub find_translation_files_in_release
{
    # Look in the dists/$DIST/Release file for the translation files that belong
    # to the given component.

    my $dist_uri  = shift;
    my $component = shift;
    my ( $release_uri, $release_path, $line ) = '';

    $release_uri  = $dist_uri . "Release";
    $release_path = get_variable("skel_path") . "/" . sanitise_uri($release_uri);

    unless ( open STREAM, "<$release_path" )
    {
        warn( "Failed to open Release file from " . $release_uri );
        return;
    }

    my $checksums = 0;
    while ( $line = <STREAM> )
    {
        chomp $line;
        if ($checksums)
        {
            if ( $line =~ /^ +(.*)$/ )
            {
                my @parts = split( / +/, $1 );
                if ( @parts == 3 )
                {
                    my ( $sha1, $size, $filename ) = @parts;
                    if ( $filename =~ m{^$component/i18n/Translation-[^./]*\.(bz2|xz)$} )
                    {
                        add_url_to_download( $dist_uri . $filename, $size );
                    }
                }
                else
                {
                    warn("Malformed checksum line \"$1\" in $release_uri");
                }
            }
            else
            {
                $checksums = 0;
            }
        }
        if ( not $checksums )
        {
            if ( $line eq "SHA256:" )
            {
                $checksums = 1;
            }
        }
    }
}

sub process_translation_index
{
    # Extract all translation files from the dists/$DIST/$COMPONENT/i18n/Index
    # file. Fall back to parsing dists/$DIST/Release if i18n/Index is not found.

    my $dist_uri  = remove_double_slashes(shift);
    my $component = shift;
    my ( $base_uri, $index_uri, $index_path, $line ) = '';

    $base_uri   = $dist_uri . $component . "/i18n/";
    $index_uri  = $base_uri . "Index";
    $index_path = get_variable("skel_path") . "/" . sanitise_uri($index_uri);

    unless ( open STREAM, "<$index_path" )
    {
        find_translation_files_in_release( $dist_uri, $component );
        return;
    }

    my $checksums = 0;
    while ( $line = <STREAM> )
    {
        chomp $line;
        if ($checksums)
        {
            if ( $line =~ /^ +(.*)$/ )
            {
                my @parts = split( / +/, $1 );
                if ( @parts == 3 )
                {
                    my ( $sha1, $size, $filename ) = @parts;
                    add_url_to_download( $base_uri . $filename, $size );
                }
                else
                {
                    warn("Malformed checksum line \"$1\" in $index_uri");
                }
            }
            else
            {
                $checksums = 0;
            }
        }
        if ( not $checksums )
        {
            if ( $line eq "SHA256:" or $line eq "SHA1:" or $line eq "MD5Sum:" )
            {
                $checksums = 1;
            }
        }
    }

    close STREAM;
}

print "Processing translation indexes: [";

foreach (@config_binaries)
{
    my ( $arch, $uri, $distribution, @components ) = @{$_};
    print "T";
    if (@components)
    {
        $url = $uri . "/dists/" . $distribution . "/";

        my $component;
        foreach $component (@components)
        {
            process_translation_index( $url, $component );
        }
    }
}

print "]\n\n";

push( @index_urls, sort keys %urls_to_download );
download_urls( "translation", sort keys %urls_to_download );

foreach ( keys %urls_to_download )
{
    s[^(\w+)://][];
    s[~][%7E]g if get_variable("_tilde");
    $skipclean{$_} = 1;
}

######################################################################################
## DEP-11 index download

%urls_to_download = ();

sub find_dep11_files_in_release
{
    # Look in the dists/$DIST/Release file for the DEP-11 files that belong
    # to the given component and architecture.

    my $dist_uri  = shift;
    my $component = shift;
    my $arch      = shift;
    my ( $release_uri, $release_path, $line ) = '';

    $release_uri  = $dist_uri . "Release";
    $release_path = get_variable("skel_path") . "/" . sanitise_uri($release_uri);

    unless ( open STREAM, "<$release_path" )
    {
        warn( "Failed to open Release file from " . $release_uri );
        return;
    }

    my $checksums = 0;
    while ( $line = <STREAM> )
    {
        chomp $line;
        if ($checksums)
        {
            if ( $line =~ /^ +(.*)$/ )
            {
                my @parts = split( / +/, $1 );
                if ( @parts == 3 )
                {
                    my ( $sha1, $size, $filename ) = @parts;
                    if ( $filename =~ m{^$component/dep11/(Components-${arch}\.yml|icons-[^./]+\.tar)\.(gz|bz2|xz)$} )
                    {
                        add_url_to_download( $dist_uri . $filename, $size );
                    }
                }
                else
                {
                    warn("Malformed checksum line \"$1\" in $release_uri");
                }
            }
            else
            {
                $checksums = 0;
            }
        }
        if ( not $checksums )
        {
            if ( $line eq "SHA256:" )
            {
                $checksums = 1;
            }
        }
    }
}

print "Processing DEP-11 indexes: [";

foreach (@config_binaries)
{
    my ( $arch, $uri, $distribution, @components ) = @{$_};
    print "D";
    if (@components)
    {
        $url = $uri . "/dists/" . $distribution . "/";

        my $component;
        foreach $component (@components)
        {
            find_dep11_files_in_release( $url, $component, $arch );
        }
    }
}

print "]\n\n";

push( @index_urls, sort keys %urls_to_download );
download_urls( "dep11", sort keys %urls_to_download );

foreach ( keys %urls_to_download )
{
    s[^(\w+)://][];
    s[~][%7E]g if get_variable("_tilde");
    $skipclean{$_} = 1;
}

######################################################################################
## cnf directory download

%urls_to_download = ();

sub find_cnf_files_in_release
{
    # Look in the dists/$DIST/Release file for the cnf files that belong
    # to the given component and architecture.

    my $dist_uri  = shift;
    my $component = shift;
    my $arch      = shift;
    my ( $release_uri, $release_path, $line ) = '';

    $release_uri  = $dist_uri . "Release";
    $release_path = get_variable("skel_path") . "/" . sanitise_uri($release_uri);

    unless ( open STREAM, "<$release_path" )
    {
        warn( "Failed to open Release file from " . $release_uri );
        return;
    }

    my $checksums = 0;
    while ( $line = <STREAM> )
    {
        chomp $line;
        if ($checksums)
        {
            if ( $line =~ /^ +(.*)$/ )
            {
                my @parts = split( / +/, $1 );
                if ( @parts == 3 )
                {
                    my ( $sha1, $size, $filename ) = @parts;
                    if ( $filename =~ m{^$component/cnf/(Commands-${arch}(\.(gz|bz2|xz))?)$} )
                    {
                        add_url_to_download( $dist_uri . $filename, $size );
                    }
                }
                else
                {
                    warn("Malformed checksum line \"$1\" in $release_uri");
                }
            }
            else
            {
                $checksums = 0;
            }
        }
        if ( not $checksums )
        {
            if ( $line eq "SHA256:" )
            {
                $checksums = 1;
            }
        }
    }
}

print "Processing cnf indexes: [";

foreach (@config_binaries)
{
    my ( $arch, $uri, $distribution, @components ) = @{$_};
    print "C";
    if (@components)
    {
        $url = $uri . "/dists/" . $distribution . "/";

        my $component;
        foreach $component (@components)
        {
            find_cnf_files_in_release( $url, $component, $arch );
        }
    }
}

print "]\n\n";

push( @index_urls, sort keys %urls_to_download );
download_urls( "cnf", sort keys %urls_to_download );

foreach ( keys %urls_to_download )
{
    s[^(\w+)://][];
    s[~][%7E]g if get_variable("_tilde");
    $skipclean{$_} = 1;
}

######################################################################################
## Main download preparations

%urls_to_download = ();

open FILES_ALL, ">" . get_variable("var_path") . "/ALL" or die("apt-mirror: can't write to intermediate file (ALL)");
open FILES_NEW, ">" . get_variable("var_path") . "/NEW" or die("apt-mirror: can't write to intermediate file (NEW)");
open FILES_MD5, ">" . get_variable("var_path") . "/MD5" or die("apt-mirror: can't write to intermediate file (MD5)");
open FILES_SHA1, ">" . get_variable("var_path") . "/SHA1" or die("apt-mirror: can't write to intermediate file (SHA1)");
open FILES_SHA256, ">" . get_variable("var_path") . "/SHA256" or die("apt-mirror: can't write to intermediate file (SHA256)");

my %stat_cache = ();

sub _stat
{
    my ($filename) = shift;
    return @{ $stat_cache{$filename} } if exists $stat_cache{$filename};
    my @res = stat($filename);
    $stat_cache{$filename} = \@res;
    return @res;
}

sub clear_stat_cache
{
    %stat_cache = ();
}

sub need_update
{
    my $filename       = shift;
    my $size_on_server = shift;

    my ( undef, undef, undef, undef, undef, undef, undef, $size ) = _stat($filename);

    return 1 unless ($size);
    return 0 if $size_on_server == $size;
    return 1;
}

sub remove_spaces($)
{
    my $hashref = shift;
    foreach ( keys %{$hashref} )
    {
        while ( substr( $hashref->{$_}, 0, 1 ) eq ' ' )
        {
            substr( $hashref->{$_}, 0, 1 ) = '';
        }
    }
}

sub process_index
{
    my $uri   = shift;
    my $index = shift;
    my ( $path, $package, $mirror, $files ) = '';

    $path = sanitise_uri($uri);
    local $/ = "\n\n";
    $mirror = get_variable("mirror_path") . "/" . $path;

    if (-e "$path/$index.gz" )
    {
        system("gunzip < $path/$index.gz > $path/$index");
    }
    elsif (-e "$path/$index.xz" )
    {
        system("xz -d < $path/$index.xz > $path/$index");
    }
    elsif (-e "$path/$index.bz2" )
    {
        system("bzip2 -d < $path/$index.bz2 > $path/$index");
    }

    unless ( open STREAM, "<$path/$index" )
    {
        warn("apt-mirror: can't open index $path/$index in process_index");
        return;
    }

    while ( $package = <STREAM> )
    {
        local $/ = "\n";
        chomp $package;
        my ( undef, %lines ) = split( /^([\w\-]+:)/m, $package );

        $lines{"Directory:"} = "" unless defined $lines{"Directory:"};
        chomp(%lines);
        remove_spaces( \%lines );

        if ( exists $lines{"Filename:"} )
        {    # Packages index
            $skipclean{ remove_double_slashes( $path . "/" . $lines{"Filename:"} ) } = 1;
            print FILES_ALL remove_double_slashes( $path . "/" . $lines{"Filename:"} ) . "\n";
            print FILES_MD5 $lines{"MD5sum:"} . "  " . remove_double_slashes( $path . "/" . $lines{"Filename:"} ) . "\n" if defined $lines{"MD5sum:"};
            print FILES_SHA1 $lines{"SHA1:"} . "  " . remove_double_slashes( $path . "/" . $lines{"Filename:"} ) . "\n" if defined $lines{"SHA1:"};
            print FILES_SHA256 $lines{"SHA256:"} . "  " . remove_double_slashes( $path . "/" . $lines{"Filename:"} ) . "\n" if defined $lines{"SHA256:"};
            if ( need_update( $mirror . "/" . $lines{"Filename:"}, $lines{"Size:"} ) )
            {
                print FILES_NEW remove_double_slashes( $uri . "/" . $lines{"Filename:"} ) . "\n";
                add_url_to_download( $uri . "/" . $lines{"Filename:"}, $lines{"Size:"} );
            }
        }
        else
        {    # Sources index
            foreach ( split( /\n/, $lines{"Files:"} ) )
            {
                next if $_ eq '';
                my @file = split;
                die("apt-mirror: invalid Sources format") if @file != 3;
                $skipclean{ remove_double_slashes( $path . "/" . $lines{"Directory:"} . "/" . $file[2] ) } = 1;
                print FILES_ALL remove_double_slashes( $path . "/" . $lines{"Directory:"} . "/" . $file[2] ) . "\n";
                print FILES_MD5 $file[0] . "  " . remove_double_slashes( $path . "/" . $lines{"Directory:"} . "/" . $file[2] ) . "\n";
                if ( need_update( $mirror . "/" . $lines{"Directory:"} . "/" . $file[2], $file[1] ) )
                {
                    print FILES_NEW remove_double_slashes( $uri . "/" . $lines{"Directory:"} . "/" . $file[2] ) . "\n";
                    add_url_to_download( $uri . "/" . $lines{"Directory:"} . "/" . $file[2], $file[1] );
                }
            }
        }
    }

    close STREAM;
}

print "Processing indexes: [";

foreach (@config_sources)
{
    my ( $uri, $distribution, @components ) = @{$_};
    print "S";
    if (@components)
    {
        my $component;
        foreach $component (@components)
        {
            process_index( $uri, "/dists/$distribution/$component/source/Sources" );
        }
    }
    else
    {
        process_index( $uri, "/$distribution/Sources" );
    }
}

foreach (@config_binaries)
{
    my ( $arch, $uri, $distribution, @components ) = @{$_};
    print "P";
    if (@components)
    {
        my $component;
        foreach $component (@components)
        {
            process_index( $uri, "/dists/$distribution/$component/binary-$arch/Packages" );
        }
    }
    else
    {
        process_index( $uri, "/$distribution/Packages" );
    }
}

clear_stat_cache();

print "]\n\n";

close FILES_ALL;
close FILES_NEW;
close FILES_MD5;
close FILES_SHA1;
close FILES_SHA256;

######################################################################################
## Main download

chdir get_variable("mirror_path") or die("apt-mirror: can't chdir to mirror");

my $need_bytes = 0;
foreach ( values %urls_to_download )
{
    $need_bytes += $_;
}

my $size_output = format_bytes($need_bytes);

print "$size_output will be downloaded into archive.\n";

download_urls( "archive", sort keys %urls_to_download );

######################################################################################
## Copy skel to main archive

sub copy_file
{
    my ( $from, $to ) = @_;
    my $dir = dirname($to);
    return unless -f $from;
    make_path($dir) unless -d $dir;
    if ( get_variable("unlink") == 1 )
    {
        if ( compare( $from, $to ) != 0 ) { unlink($to); }
    }
    unless ( copy( $from, $to ) )
    {
        warn("apt-mirror: can't copy $from to $to");
        return;
    }
    my ( $atime, $mtime ) = ( stat($from) )[ 8, 9 ];
    utime( $atime, $mtime, $to ) or die("apt-mirror: can't utime $to");
}

foreach (@index_urls)
{
    die("apt-mirror: invalid url in index_urls") unless s[^(\w+)://][];
    copy_file( get_variable("skel_path") . "/" . sanitise_uri("$_"), get_variable("mirror_path") . "/" . sanitise_uri("$_") );
    copy_file( get_variable("skel_path") . "/" . sanitise_uri("$_"), get_variable("mirror_path") . "/" . sanitise_uri("$_") ) if (s/\.gz$//);
    copy_file( get_variable("skel_path") . "/" . sanitise_uri("$_"), get_variable("mirror_path") . "/" . sanitise_uri("$_") ) if (s/\.bz2$//);
    copy_file( get_variable("skel_path") . "/" . sanitise_uri("$_"), get_variable("mirror_path") . "/" . sanitise_uri("$_") ) if (s/\.xz$//);
}

######################################################################################
## Make cleaning script

my ( @rm_dirs, @rm_files ) = ();
my $unnecessary_bytes = 0;

sub process_symlink
{
    return 1;    # symlinks are always needed
}

sub process_file
{
    my $file = shift;
    $file =~ s[~][%7E]g if get_variable("_tilde");
    return 1 if $skipclean{$file};
    push @rm_files, sanitise_uri($file);
    my ( undef, undef, undef, undef, undef, undef, undef, $size, undef, undef, undef, undef, $blocks ) = stat($file);
    $unnecessary_bytes += $blocks * 512;
    return 0;
}

sub process_directory
{
    my $dir       = shift;
    my $is_needed = 0;
    return 1 if $skipclean{$dir};
    opendir( my $dir_h, $dir ) or die "apt-mirror: can't opendir $dir: $!";
    foreach ( grep { !/^\.$/ && !/^\.\.$/ } readdir($dir_h) )
    {
        my $item = $dir . "/" . $_;
        $is_needed |= process_directory($item) if -d $item && !-l $item;
        $is_needed |= process_file($item)      if -f $item;
        $is_needed |= process_symlink($item)   if -l $item;
    }
    closedir $dir_h;
    push @rm_dirs, $dir unless $is_needed;
    return $is_needed;
}

chdir get_variable("mirror_path") or die("apt-mirror: can't chdir to mirror");

foreach ( keys %clean_directory )
{
    process_directory($_) if -d $_ && !-l $_;
}

open CLEAN, ">" . get_variable("cleanscript") or die("apt-mirror: can't open clean script file");

my ( $i, $total ) = ( 0, scalar @rm_files );

if ( get_variable("_autoclean") )
{

    my $size_output = format_bytes($unnecessary_bytes);
    print "$size_output in $total files and " . scalar(@rm_dirs) . " directories will be freed...";

    chdir get_variable("mirror_path") or die("apt-mirror: can't chdir to mirror");

    foreach (@rm_files) { unlink $_; }
    foreach (@rm_dirs)  { rmdir $_; }

}
else
{

    my $size_output = format_bytes($unnecessary_bytes);
    print "$size_output in $total files and " . scalar(@rm_dirs) . " directories can be freed.\n";
    print "Run " . get_variable("cleanscript") . " for this purpose.\n\n";

    print CLEAN "#!/bin/sh\n";
    print CLEAN "set -e\n\n";
    print CLEAN "cd " . quoted_path(get_variable("mirror_path")) . "\n\n";
    print CLEAN "echo 'Removing $total unnecessary files [$size_output]...'\n";
    foreach (@rm_files)
    {
        print CLEAN "rm -f '$_'\n";
        print CLEAN "echo -n '[" . int( 100 * $i / $total ) . "\%]'\n" unless $i % 500;
        print CLEAN "echo -n .\n" unless $i % 10;
        $i++;
    }
    print CLEAN "echo 'done.'\n";
    print CLEAN "echo\n\n";

    $i     = 0;
    $total = scalar @rm_dirs;
    print CLEAN "echo 'Removing $total unnecessary directories...'\n";
    foreach (@rm_dirs)
    {
        print CLEAN "if test -d '$_'; then rmdir '$_'; fi\n";
        print CLEAN "echo -n '[" . int( 100 * $i / $total ) . "\%]'\n" unless $i % 50;
        print CLEAN "echo -n .\n";
        $i++;
    }
    print CLEAN "echo 'done.'\n";
    print CLEAN "echo\n";

    close CLEAN;

}

# Make clean script executable
my $perm = ( stat get_variable("cleanscript") )[2] & 07777;
chmod( $perm | 0111, get_variable("cleanscript") );

if ( get_variable("run_postmirror") )
{
    print "Running the Post Mirror script ...\n";
    print "(" . get_variable("postmirror_script") . ")\n\n";
    if ( -x get_variable("postmirror_script") )
    {
        system( get_variable("postmirror_script"), '' );
    }
    else
    {
        system( '/bin/sh', get_variable("postmirror_script") );
    }
    print "\nPost Mirror script has completed. See above output for any possible errors.\n\n";
}

unlock_aptmirror();
```

----------------------------------------------------

run: </br>
$ sudo apt-mirror </br>

```
Downloading 216 index files using 20 threads...
Begin time: Wed May 14 16:15:52 2025
[20]... [19]... [18]... [17]... [16]... [15]... [14]... [13]... [12]... [11]... [10]... [9]... [8]... [7]... [6]... [5]... [4]... [3]... [2]... [1]... [0]...
End time: Wed May 14 16:16:56 2025

Processing translation indexes: [TTTT]

Downloading 567 translation files using 20 threads...
Begin time: Wed May 14 16:16:56 2025
[20]... [19]... [18]... [17]... [16]... [15]... [14]... [13]... [12]... [11]... [10]... [9]... [8]... [7]... [6]... [5]... [4]... [3]... [2]... [1]... [0]...
End time: Wed May 14 16:17:06 2025

Processing DEP-11 indexes: [DDDD]

Downloading 120 dep11 files using 20 threads...
Begin time: Wed May 14 16:17:06 2025
[20]... [19]... [18]... [17]... [16]... [15]... [14]... [13]... [12]... [11]... [10]... [9]... [8]... [7]... [6]... [5]... [4]... [3]... [2]... [1]... [0]...
End time: Wed May 14 16:17:14 2025

Processing cnf indexes: [CCCC]

Downloading 32 cnf files using 20 threads...
Begin time: Wed May 14 16:17:14 2025
[20]... [19]... [18]... [17]... [16]... [15]... [14]... [13]... [12]... [11]... [10]... [9]... [8]... [7]... [6]... [5]... [4]... [3]... [2]... [1]... [0]...
End time: Wed May 14 16:17:19 2025

Processing indexes: [SSSSPPPP]

655.9 GiB will be downloaded into archive.
Downloading 208237 archive files using 20 threads...
Begin time: Wed May 14 16:17:37 2025
[20]... [19]... [18]... [17]... [16]... [15]... [14]... [13]... [12]... [11]... [10]... [9]... [8]... [7]... [6]... [5]... [4]... [3]... [2]... [1]... [0]...
End time: Wed May 14 19:48:11 2025

0 bytes in 0 files and 0 directories can be freed.
Run /media/user/ext4-1.7TB/var/spool/apt-mirror/var/clean.sh for this purpose.

Running the Post Mirror script ...
(/media/user/ext4-1.7TB/var/spool/apt-mirror/var/postmirror.sh)

/bin/sh: 0: Can't open /media/user/ext4-1.7TB/var/spool/apt-mirror/var/postmirror.sh

Post Mirror script has completed. See above output for any possible errors.
```

----------------------------------------------------------------------


# Old Method

```
    How to make your own Ubuntu Repository DVDs

    preamble
    Corrections and fixes submitted by the community are added and credited within the main tutorial. However, many of the posts that follow add great ideas. Please feel free to browse. But this first post should have all you need in terms of accomplishing the task at hand. And a big thanks to the brilliant souls who've added so many detailed and beautifully formatted ideas!

    EDIT: This tutorial has fallen behind a bit. A clean up will begin shortly.



    note
    Sensiva reports that 9.10 Karmic Koala has an issue with this, stemming from a bug in apt:

    Let me describe it in details. If I want to install a program that needs more than one package, and those packages are located on different DVDs, APT used to fetch then install then ask for the next disc. But with Karmic, after fetching it doesn't install, it prompts for the next discs... etc and then starts installation, the packages currently in the last disc in the drive are installed, then giving an error about the other packages which are located in the previous discs because they are not there. Its not a problem in the discs, I think its APT behavior itself that should be changed. Unfortunately I don't know how to adjust this way of APT in Karmic. but a workaround for this is pointing APT to the local repos already downloaded on the harddrive and not using the created DVDs. I have tested it and worked fine.

    ! What's The Point Of This Tutorial?

    A local set of repositories proves useful for friends and family who have slow or no bandwidth. Assembled from various sources (see the references below) these steps should create Ubuntu repository DVDs to share or keep in storage, however the need arises.

    What level experience do you need?

    I've tried to conform to the tutorial rules. However, the very nature of this tutorial requires a bit more thinking than simply copying and pasting. Feel free to post a question about some step that may not work right for you. But the idea is to reduce the amount of text in the tutorial within reason by not detailing each step.

    Hard Disk Requirements (Please Read This)

    Roughly 45 gigabytes free hard drive space: to store the complete set of .deb files Ubuntu offers; to create four DVD ISOs; left-over space for the function of the PC. Now, in a day and age of the 1 terabyte drive... this may not be a huge consideration any more. But if your drive is smaller than 60 Gb, this download will take its toll. Download sizes alone (not including space required to make DVD ISOs):
    4.10 - Warty Warthog: 9.72 GB
    5.04 - Hoary Hedgehog: 11.53 GB
    5.10 - Breezy Badger: 12.17 GB
    6.06 - Dapper Drake: 15.71 GB
    6.10 - Edgy Eft: 17.46 GB
    7.04 - Feisty Fawn: 19.90 GB
    7.10 - Gutsy Gibbon: 24.81 GB
    8.04 - Hardy Heron: 28.85 GB
    8.10 - Intrepid Ibex: 26.72 GB
    9.04 - Jaunty Jackalope: 27.53 GB
    9.10 - Karmic Koala: 32.11 GB
    10.04 - Lucid Lynx: 32.29 GB
    10.10 - Maverick Meerkat: 39.38 GB
    11.04 - Natty Narwhal: 40.00 GB
    11.10 - Oneiric Ocelot: 42.79 GB
    Well, you get the point. From Warty to Oneiric we went from roughly 10 Gb to 43 Gb.
    Index
    01. Install the necessary tools covers what you'll need to get started.
    02. Extract debcopy is not necessary to do first, but we get it out of the way at the beginning.
    03. The Big Download has the command necessary to download the desired repositories. Make your changes according to the type you need.
    04. Divide into DVD-sized portions prepares the files to make the DVDs.
    05. Create ISOs finishes what began in step 4.
    06. Burning the ISO files covers how to write the files to blank discs.
    07. Getting It All To Work is the instructions on how to tell Ubuntu to use the DVDs as a source for installing new software.
    08. Updating the local repositories shows how to keep what you've downloaded up-to-date.
    09. Pointing Apt locally shows you how to use what you've downloaded to your own advantage (should you decide to keep it around).
    10. You only want a setup DVD? is a link to Ubuntu setup DVDs for download.
    11. FAQs answers some frequently asked questions.
    Q1: How would I do this through DOS/Windows?
    Q2: Why don't I skip this mess and download pre-made DVD ISOs?
    Q3: What if I wanted to combine or add in more repositories? Something like Canonical Commercial? Or MediBuntu?
    Q4: How to get "debmirror" to download and validate the "Release.gpg" file.
    Q5: How do I setup a local Ubuntu mirror from a set of Repository DVDs?
    12. Classic Ubuntu Repository Downloads. Warty? Hoary? Breezy? Feisty, anyone? Don't fret about older editions disappearing. Help is right here.
    13. References gives credit where credit is due - and links
    14. Mission, Vision & Values (very boring)
    1. Install the necessary tools

    Open a terminal (from the top-left side of the Ubuntu screen: Applications → Accessories → Terminal). Please do not close this application throughout the length of this tutorial.

    Copy the following code and paste it into the terminal.
    Code:

    sudo apt-get install debmirror liblockfile-simple-perl liblog-agent-perl ruby mkisofs dpkg-dev libdigest-sha1-perl libruby libzlib-ruby

    Hit the Enter key after pasting. (A quick way to "paste" text into the terminal is to hold down the following keys Ctrl+Shift+V, or right click in the terminal and click Paste in the context menu.)

    While debpartial is still in the repositories (big fancy phrase that means: this is an official and valid Ubuntu file), starting with Hardy Heron, it is not considered part of Hardy Heron or newer -- it is 100% compatible. Provided below is the link to the necessary software:
    >> CLICK HERE << to download DebPartial from Canonical.
    Once saved on the desktop, double-click the file and extract debpartial to your home directory/folder.


    2. Extract debcopy

    The debcopy file is included in debpartial (downloaded and installed in Step 1). Let's extract it now for later use. Please paste the following code into the Terminal:
    Code:

    cp /usr/share/doc/debpartial/examples/debcopy.gz ~

    Code:

    gunzip ~/debcopy.gz

    3. The Big Download

    Use this code to start downloading Lucid Lynx's repository files:
    Code:

    debmirror --nosource -m --passive --host=archive.ubuntu.com --root=ubuntu/ --method=http --progress --dist=lucid,lucid-security,lucid-updates,lucid-backports, --section=main,restricted,universe,multiverse --arch=i386 ~/UbuntuRepos --ignore-release-gpg

    To see the whole code with some brief explanations:
    Code:

    debmirror \
        --nosource -m --passive \
        --host=archive.ubuntu.com \ (select a faster repository if need be)
        --root=ubuntu/ --method=http --progress \
        --dist=lucid,lucid-security,lucid-updates,lucid-backports, \
        --section=main,restricted,universe,multiverse \
        --arch=i386 ~/UbuntuRepos \ (this puts our files in /home/USER/UbuntuRepos)
        --ignore-release-gpg

    Some variables in blue.


        Want a detailed explanation of this command? CLICK HERE. (links to a post in this thread.)
        Be forewarned: this can take a long time, perhaps 24 or so hours. Downloading individual files is slow.


    Did the download stop midway?
    Press the up cursor key on your keyboard and the command should re-appear. Hit Enter and the download will pick up where it last left off.

    Regular network glitches?
    Try adding one of the following argument(s) to the command (the number following "-t" is the length of time -- in seconds -- before the retry begins) :

    Code:

    --timeout=seconds -t 120

    The "120" means 120 seconds or 2 minutes. Increase it to 4 minutes, for example, like this:
    Code:

    --timeout=seconds -t 240

    4. Divide into DVD-sized portions

    Part 1:

        Note: If you changed repositories in Step 3 then the following command will require editing:


    Code:

    debpartial --nosource --dirprefix=ubuntu --section=main,restricted,universe,multiverse --dist=lucid,lucid-security,lucid-updates,lucid-backports --size=DVD ~/UbuntuRepos ~/UbuntuDVDs

    1. Replace the red lucid in the code above with your repository choice (i.e., intrepid, jaunty or karmic).

    2. If you plan on burning CDs instead of DVDs, then replace --size=DVD with --size=CD74 (for 650 megabyte CD-Rs), or --size=CD80 (for 700 megabyte CD-Rs)

    Part 2:
    How many CDs will we need to create? How do we find out? Paste this code in the Terminal and hit Enter:
    Code:

    ls -l ~/UbuntuDVDs

    If the last folder is ubuntu3 you need to create 4 DVDs. If the last folder is ubuntu4 then it will be 5 DVDs, and if the last folder is ubuntu31, you'll need to create 32 CDs, and so on. NOTE: Oneiric's repository size = 42.79 Gb. You're going to need a large stack of CDs.

    Having determined the number of physical discs you'll need, use the following as your pattern until all discs are complete (the only change from one to the next is the final number). These represent Lucid Lynx's requirements:

    Code:

    ruby ~/debcopy -l ~/UbuntuRepos ~/UbuntuDVDs/ubuntu0

    Code:

    ruby ~/debcopy -l ~/UbuntuRepos ~/UbuntuDVDs/ubuntu1

    Code:

    ruby ~/debcopy -l ~/UbuntuRepos ~/UbuntuDVDs/ubuntu2

    Code:

    ruby ~/debcopy -l ~/UbuntuRepos ~/UbuntuDVDs/ubuntu3

    Code:

    ruby ~/debcopy -l ~/UbuntuRepos ~/UbuntuDVDs/ubuntu4

    Code:

    ruby ~/debcopy -l ~/UbuntuRepos ~/UbuntuDVDs/ubuntu5

    Code:

    ruby ~/debcopy -l ~/UbuntuRepos ~/UbuntuDVDs/ubuntu6

    Code:

    ruby ~/debcopy -l ~/UbuntuRepos ~/UbuntuDVDs/ubuntu7

    5. Create ISOs

    These instructions assume you've got enough data to make 8 DVDs. If you've downloaded repositories that do not need an 8th DVD, none will be created.

    Code:

    mkisofs -f -J -r -V "Ubuntu 10.04 1/8" -o ubuntu-10.04-$(date -I)-complete-i386-dvd1.iso ~/UbuntuDVDs/ubuntu0

    Code:

    mkisofs -f -J -r -V "Ubuntu 10.04 2/8" -o ubuntu-10.04-$(date -I)-complete-i386-dvd2.iso ~/UbuntuDVDs/ubuntu1

    Code:

    mkisofs -f -J -r -V "Ubuntu 10.04 3/8" -o ubuntu-10.04-$(date -I)-complete-i386-dvd3.iso ~/UbuntuDVDs/ubuntu2

    Code:

    mkisofs -f -J -r -V "Ubuntu 10.04 4/8" -o ubuntu-10.04-$(date -I)-complete-i386-dvd4.iso ~/UbuntuDVDs/ubuntu3

    Code:

    mkisofs -f -J -r -V "Ubuntu 10.04 5/8" -o ubuntu-10.04-$(date -I)-complete-i386-dvd5.iso ~/UbuntuDVDs/ubuntu4

    Code:

    mkisofs -f -J -r -V "Ubuntu 10.04 6/8" -o ubuntu-10.04-$(date -I)-complete-i386-dvd6.iso ~/UbuntuDVDs/ubuntu5

    Code:

    mkisofs -f -J -r -V "Ubuntu 10.04 7/8" -o ubuntu-10.04-$(date -I)-complete-i386-dvd7.iso ~/UbuntuDVDs/ubuntu6

    Code:

    mkisofs -f -J -r -V "Ubuntu 10.04 8/8" -o ubuntu-10.04-$(date -I)-complete-i386-dvd8.iso ~/UbuntuDVDs/ubuntu7

    Note: If you changed repositories in Step 3 then each command above will require editing. Replace the version number of 10.04 with:

        9.04 for jaunty and 9.10 for karmic, 10.10 for maverick
        i386 for amd64, powerpc or sparc



        If you're committed to creating CD-R ISOs (detailed in Step 4) then make the following changes:
            replace -dvd1.iso with -cd1.iso
            replace -dvd2.iso with -cd2.iso (and so on until all 50 or so CD-R ISOs are completed)



    6. Burning the ISO files

    Using Ubuntu (Link)

        Insert a blank CD into your burner. A "CD/DVD Creator" or "Choose Disc Type" window will pop up. Close this, as we will not be using it.
        Browse to the created ISO image in the file browser
        Right click on the ISO image file and choose Write to Disc.
        Select the write speed. It is recommended that you write at the lowest possible speed.
        Start the burning process and repeat these steps until all ISOs are done.



    Using Kubuntu (Link)

        Find the ISO image in the file browser (available at System Menu > Home Folder on bottom of the screen next to KMenu.)
        Right click on the ISO → Actions → Write CD Image with K3b...
        K3b will now automatically verify the md5sum, make sure these match.
        Place Blank CD in burner and click on start.



    Using Xubuntu (Link)

        Launch the burning tool, xfburn (available at Applications → Accessories → xfburn.)
        Choose from the main toolbar, or from the menu Actions the option "Burn CD Image",
        From the Burn CD Image dialog box, click over (None) from "image to burn", and find the ISO image in the file browser
        In the dialog, click 'Burn Image'.



    7. Getting it all to work

    There are two ways to add a DVD/CD to your repository list:
    "KPackageKit" and "Adept"

    Kubuntu's built-in package managers seem to have difficulties with adding CD/DVD ROMs to the list of available system packages. Try using the "Terminal" solution.

    Synaptic:

    Open Synaptic Package Manager (System → Administration → Synaptic Package Manager). Enter your password. From the menu: Edit → Add CD-ROM.... and follow the clearly written instructions in each dialog box. Repeat until all your media (DVDs, CDs) have been entered. Should this fail, try the following.

    Terminal:

    Open a Terminal (Applications → Accessories → Terminal) and enter the following command:
    Code:

    sudo apt-cdrom add

    Insert the first disc and follow the instructions (it will ask you to name each disc). When apt-cdrom is finished with the disc, eject it with
    Code:

    eject

    (yeah, just type it in and hit Enter) and insert the next and use the same command. Repeat the process per disc.

    Once the last disc is done, enter this code in the Terminal:
    Code:

    sudo apt-get update

    followed by
    Code:

    sudo apt-get upgrade

    which will update your system if any newer files are available. You can now open Synaptic Package Manager to use tens of thousands of packages.

    (To be added: how to remove previously added DVD/CD entries in apt when old discs are replaced with new, as well as correcting any goofs that occur when incorrectly labeling the discs, etc.)
    8. Updating the local repositories (i.e., "Now what!?")

    Now that you have gigabytes of .deb files all tucked neatly into the ~/UbuntuRepos folder... what on earth do you do to keep it updated? Start again? Sort of. Read on...

    Open the Terminal (Applications → Accessories → Terminal) and go to the folder where you initially ran debmirror (by default: your home folder) and run the command in Step 3 again. This only makes an incremental update consuming roughly 10 minutes instead of 25 hours. Run this daily and ISO creation won't have the added bore of a long download.

    If new DVDs are required or desired, delete your original .iso files (in your home folder) and run steps 5 through 7 to recreate them.

    9. Pointing Apt locally

    luvr has a tutorial entitled "Creating a Trusted Local Repository from which Software Updates can be installed" that is more detailed than my addition. CLICK HERE for this Ubuntu Forums HOWTO. It begins with:
    "If you manage multiple PCs running Ubuntu, you will likely want to keep them all updated. Thus, you will want to install the Ubuntu updates to each of them as they become available, and you will have each PC individually download all of the updates from the Ubuntu repositories on the internet. This may, however, be impractical to you—particularly if, e.g., you are on a rather slow internet connection, or if your monthly data transfer volume is severely limited, or if you simply prefer to save the bandwidth."
    Credit=Geochelone

    It is possible to use the files you've downloaded as your own local repository. It can be used for updates and installing new packages. But this will only work well if you keep these files up-to-date. If not, you will fall behind in security updates.

    With that out of the way let's proceed. Enter the following code in the terminal:

    Code:

    cd ~/UbuntuRepos && dpkg-scanpackages . /dev/null > Packages && gzip Packages && cd ~

    This creates the necessary files for apt to know what is in your local repositories.

    Once completed insert the following into your sources.list file:
    Code:

    deb file:/home/[USER NAME HERE]/UbuntuRepos/ lucid main multiverse restricted universe

    Code:

    deb file:/home/[USER NAME HERE]/UbuntuRepos/ lucid-security main multiverse restricted universe

    (CREDIT: xfile087)

    NOTE: Don't forget these two things:

        modify these lines for jaunty or karmic
        put hash marks (#) before each Internet repository line you no longer require. Example:

        Code:

        # deb http://archive.ubuntu.com/ubuntu hardy-backports main restricted universe multiverse



    10. You only want a setup DVD?

    Head over to this link and select the corresponding Ubuntu suite you desire.
    The difference between the Ubuntu setup CD and the Ubuntu setup DVD is how much of the repositories are on the setup disc. The DVD contains quite a bit more, most of "Main" if I'm not mistaken. However, if you have the repository DVDs handy, a setup DVD is redundant and not recommended. Save your bandwidth and don't bother. However, if you want a snazzy setup DVD, try this on for size:

    Click here: Ultimate Edition

    Click here: SuperOS


    11. F.A.Qs

    Question 1: How would I do this through DOS/Windows?

    Answer 1. I... honestly... don't... know. I can't imagine how tedious it would be to try to assemble all this file by file. If you're a dual-booter or you've got faster bandwidth on a Windows machine elsewhere then I understand your dilemma. debmirror is the binary that does all the work. I cannot imagine that anyone has made a port for Windows. Until someone shows me a blog that tells Windows users how to accomplish this I suggest looking at the solutions in Q2.

    Answer 2. This is a "round about" way of using a Windows PC where the bandwidth is fast. Try using the Ubuntu setup CD in "Live User Mode". Follow the instructions in this tutorial and use the Windows Hard Drive as your destination. Well, look. I didn't think it would be easy, just... sorta... possible.

    __________________________________

    Question 2: Why don't I skip this mess and download pre-made DVD ISOs?

    Answer 1. Pre-made DVD ISOs are fine for a single, never-to-be-repeated download (if time is an issue). Here's the difficulty: to get updates means downloading the ISOs again (sorta time consuming). If a single ISO download is right for you -- use this link: ftp://ftp.leg.uct.ac.za/pub/linux/ub...-packages-dvd/

    Answer 2. Modem connections make this tutorial impossible regardless the O/S. Consider purchasing inexpensive DVD ISOs from an online vendor. Examples are:

        On-Disk.com
        OSDisc.com


    They will provide exactly what you need: Ubuntu Repository DVDs with setup disks - perfect for that off-line PC. These companies will ship these discs right to your door for a small fee (no affiliation with the author).

    __________________________________

    Question 3: What if I wanted to combine or add in more repositories? Something like Canonical Commercial? Or MediBuntu?

    Answer. Don't do it!

    OK. Now that I've got your attention... let me say that jocose says you can. But you must follow his instructions here. Please note: these instructions have not been tested in our labs at Muppet Central. Advance at your own risk.

    For those of you who want to mess around and don't mind burning an extra CD or two, continue reading:
    debmirror will only erase your original files. You must change the name of the download folder(s). Let me explain and illustrate.
    This is what debmirror appears to do when run:

        debmirror receives the file list from the server within the command line (archive.ubuntu.com, for example).
        debmirror then looks for the "destination" folder. If it doesn't exist, it creates it.
        if debmirror finds that the destination folder already exists it compares the files it finds with the remote server's file listing.
        debmirror then carefully erases everything in the destination folder that is not an exact match with that list.
        debmirror downloads files it deems are missing in the destination folder.



    If you are updating a local set of files, this is a good thing. But if you choose a different server, this is a bad thing.

    The quick way to add files from different servers (such as MediBuntu and Canonical) is to put the additional files in unique folders. Examples: ~/MediBuntuRepos and/or ~/CanonicalRepos (your folder names may vary). Then replace ~/UbuntuRepos in each command.

    If you're feeling confused by the above description and you just want to start downloading, the following steps show how to create Canonical and Medibuntu CD sets:

    Canonical Repos
    Code:

    debmirror --nosource -m --passive --host=archive.canonical.com --root=/ --method=http --progress --dist=lucid,lucid-backports,lucid-proposed,lucid-security,lucid-updates --section=partner --arch=i386 ~/CanonicalRepos --ignore-release-gpg

    Code:

    debpartial --nosource --dirprefix=ubuntu --section=partner --dist=lucid,lucid-backports,lucid-proposed,lucid-security,lucid-updates --size=CD80 ~/CanonicalRepos ~/CanonicalRepos/CD

    Code:

    ruby debcopy -l ~/CanonicalRepos ~/CanonicalRepos/CD/ubuntu0

    Code:

    mkisofs -f -J -r -V "Canonical" -o ubuntu-7.10-$(date -I)-Canonical.iso ~/CanonicalRepos/CD/ubuntu0

    Replace lucid with intrepid, jaunty or karmic.

    Architecture options = amd64, powerpc or sparc

    MediBuntu Repos
    Code:

    debmirror --nosource -m --passive --host=packages.medibuntu.org --root=/ --method=http --progress --dist=lucid --section=free,non-free --arch=i386 ~/MediBuntuRepos --ignore-release-gpg

    Code:

    ruby debcopy -l ~/MediBuntuRepos ~/MediBuntuRepos/CD/ubuntu0

    Code:

    mkisofs -f -J -r -V "MediBuntu" -o ubuntu-7.10-$(date -I)-MediBuntu.iso ~/MediBuntuRepos/CD/ubuntu0

    Replace lucid with intrepid, jaunty or karmic.

    Architecture options = i386, amd64, powerpc

    (To add MediBuntu to your personal repositories: https://help.ubuntu.com/community/Me...07a38a00d70e9f)

    Google Repos
    Code:

    debmirror --nosource -m --passive --host=dl.google.com --root=/linux/deb/ --method=http --progress --dist=stable --section=non-free --arch=i386 ~/GoogleRepos --ignore-release-gpg

    The Google directories appear... then disappear. This is the last directory address I have.

    __________________________________

    Question 4: How to get "debmirror" to download and validate the "Release.gpg" file.

    Answer. I don't know!!! But luvr does. Thanks for the addition.

    luvr tackles this question with this patient and well laid out post: Click Here.

    __________________________________

    Q5: How do I setup a local Ubuntu mirror from a set of Repository DVDs?

    Answer. I just don't know!!! But luvr does. Thanks for the addition.

    luvr tackles this question with this patient and well laid out post: Click Here.


    __________________________________

    12. Classic Ubuntu Repository Downloads

    For anyone who is interested for whatever reason... be it nostalgia, business, just plain techno geek, or reasons you would probably not want everyone to know, here are example codes for the classic repositories.

    Warty Warthog on the i386 architecture:

    Code:

    debmirror --nosource -m --passive --host=old-releases.ubuntu.com --root=ubuntu/ --method=http --progress --dist=warty,warty-security,warty-updates,warty-backports, --section=main,restricted,universe,multiverse --arch=i386 ~/WartyRepos_i386 --ignore-release-gpg

    Warty Warthog on the 64-bit AMD architecture:

    Code:

    debmirror --nosource -m --passive --host=old-releases.ubuntu.com --root=ubuntu/ --method=http --progress --dist=warty,warty-security,warty-updates,warty-backports, --section=main,restricted,universe,multiverse --arch=amd64 ~/WartyRepos_amd64 --ignore-release-gpg

    Doing it this way clearly defines a different folder for each repository set. Note the coloured text can be changed to suit the version, architecture, and output folder as per your original tutorial method.

    Please note a few points:

    1. Don't mix repositories. If you have enough free hard drive space to keep these repositories around then put each version's files in a different folder (fear not: they're not being updated any more: what you download never needs to be updated). Here we've used "~/UbuntuReposWartyi386" and "~/UbuntuReposWartyAMD64" and not "~/UbuntuRepos". All contents of ~/UbuntuRepos will be wiped to accommodate the new download. We encourage a folder for each version, e.g. Wartyi386, WartyAMD64, Hoaryi386, HoaryAMD64, etc.

    2. the architectures currently available in each (old.release) version are:

        4.10 (Warty Warthog); i386, amd64, ppc.
        5.04 (Hoary Hedgehog); i386, amd64, ppc, ia64, sparc.
        5.10 (Breezy Badger); i386, amd64, ppc, ia64, sparc.
        6.10 (Edgy Eft); i386, amd64, ppc, ia64, sparc.
        7.04 (Feisty Fawn); i386, amd64, ppc, sparc.
        7.10 (Gutsy Gibbon); amd64, armel, hppa, i386, ia64, lpia, powerpc, sparc.
        8.04 (Hardy Heron)


    CREDIT: k3lt01

    If you're still running a classic copy of Ubuntu and want to install software, change your sources.list file to reflect that release. If it would be Warty Warthog, 4.10, you'd use the following:

    Code:

    ## EOL upgrade sources.list
    # Required
    deb http://old-releases.ubuntu.com/ubuntu/ warty main restricted universe multiverse
    deb http://old-releases.ubuntu.com/ubuntu/ warty-updates main restricted universe multiverse
    deb http://old-releases.ubuntu.com/ubuntu/ warty-security main restricted universe multiverse

    # Optional
    #deb http://old-releases.ubuntu.com/ubuntu/ warty-backports main restricted universe multiverse
    #deb http://old-releases.ubuntu.com/ubuntu/ warty-proposed main restricted universe multiverse

    Source: https://help.ubuntu.com/community/EOLUpgrades

    13. References

    This tutorial was pieced together from:

        Ramon's original tutorial has vanished. It was last seen at http://cargol.net/~ramon/ubuntu-dvd-en. It was very basic and designed for Warty Warthog. Then I found...
        How to Make Ubuntu DVDs by Burt (significant modifications to Ramon's tutorial. Bert's instructions were brought to this forum for a local reference. Thanks Burt for improving on Ramon's original.)



    The tutorial was updated with the help of the thoughtful comments by forum friends ... and I may have polished it up a bit. I take responsibility for all errors and give credit to contributors for all accuracies.

    Other related links:

        Apt on CD
        Creating a Trusted Local Repository from which Software Updates can be installed
        HowToCreateYourOwnGNULinuxDistribution
        HowTo Forge
        List of Ubuntu Releases



    14. Mission, Vision & Values

        My mission was to put the repository tutorial here in the Ubuntu Forums, to include extra elements and ensure that those with basic Ubuntu experience could accomplish this. This tutorial was added to this Forum for these reasons:
            The original tutorial disappeared.
            The original tutorial did not reflect Ubuntu's 6-month updates or the changes the tutorial needed due to the difference in the size of the growing repositories, etc.
            The original tutorial tutorial only focused on i386.

        My vision is to keep it maintained (the repositories are forever changing, making this necessary), improve its wording, answer questions that arise due to the lack of clarity that perpetually plagues the text.
        My values cover only active versions, with a focus on the most recent Long Term Support distribution. Support for abandoned ("unsupported") editions or future alpha/beta ("unstable") editions must be maintained by forum members who feel the need to have these. Please do not perceive my refusal to work on these as a personal affront. Send me the codes. I'll add them and credit the author.
        Note: k3lt01 has stepped up to the plate and has sent me the codes for the classic (i.e., discontinued) distros. Should you encounter problems with these codes, let me know and I'll fix 'em.


