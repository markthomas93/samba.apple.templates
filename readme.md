System Administration tools                                       VFS_FRUIT(8)



NAME
       vfs_fruit - Enhanced OS X and Netatalk interoperability

SYNOPSIS
       vfs objects = fruit

DESCRIPTION
       This VFS module is part of the samba(7) suite.

       The vfs_fruit module provides enhanced compatibility with Apple SMB
       clients and interoperability with a Netatalk 3 AFP fileserver.

       The module should be stacked with vfs_catia if enabling character
       conversion and must be stacked with vfs_streams_xattr, see the example
       section for the correct config.

       The module enables alternate data streams (ADS) support for a share,
       intercepts the OS X special streams "AFP_AfpInfo" and "AFP_Resource"
       and handles them in a special way. All other named streams are deferred
       to vfs_streams_xattr which must be loaded together with vfs_fruit.

       Be careful when mixing shares with and without vfs_fruit. OS X clients
       negotiate SMB2 AAPL protocol extensions on the first tcon, so mixing
       shares with and without fruit will globally disable AAPL if the first
       tcon is without fruit.

       Having shares with ADS support enabled for OS X client is worthwhile
       because it resembles the behaviour of Apple's own SMB server
       implementation and it avoids certain severe performance degradations
       caused by Samba's case sensitivity semantics.

       The OS X metadata and resource fork stream can be stored in a way
       compatible with Netatalk 3 by setting fruit:resource = file and
       fruit:metadata = netatalk.

       OS X maps NTFS illegal characters to the Unicode private range in SMB
       requests. By setting fruit:encoding = native, all mapped characters are
       converted to native ASCII characters.

       Finally, share access modes are optionally checked against Netatalk AFP
       sharing modes by setting fruit:locking = netatalk.

       This module is not stackable other than described in this manpage.

GLOBAL OPTIONS
       The following options must be set in the global smb.conf section and
       won't take effect when set per share.

       fruit:aapl = yes | no
           A global option whether to enable Apple's SMB2+ extension codenamed
           AAPL. Default yes. This extension enhances several deficiencies
           when connecting from Macs:

                  o   directory enumeration is enriched with Mac relevant
                      filesystem metadata (UNIX mode, FinderInfo, resource
                      fork size and effective permission), as a result the Mac
                      client doesn't need to fetch this metadata individually
                      per directory entry resulting in an often tremendous
                      performance increase.

                  o   The ability to query and modify the UNIX mode of
                      directory entries.

           There's a set of per share options that come into play when
           fruit:aapl is enabled. These options, listed below, can be used to
           disable the computation of specific Mac metadata in the directory
           enumeration context, all are enabled by default:

                  o   readdir_attr:aapl_rsize = yes | no

                  o   readdir_attr:aapl_finder_info = yes | no

                  o   readdir_attr:aapl_max_access = yes | no

           See below for a description of these options.

       fruit:nfs_aces = yes | no
           A global option whether support for querying and modifying the UNIX
           mode of directory entries via NFS ACEs is enabled, default yes.

       fruit:copyfile = yes | no
           A global option whether to enable OS X specific copychunk ioctl
           that requests a copy of a whole file along with all attached
           metadata.

           WARNING: the copyfile request is blocking the client while the
           server does the copy.

           The default is no.

       fruit:model = MacSamba
           This option defines the model string inside the AAPL extension and
           will determine the appearance of the icon representing the Samba
           server in the Finder window.

           The default is MacSamba.

OPTIONS
       The following options can be set either in the global smb.conf section
       or per share.

       fruit:resource = [ file | xattr | stream ]
           Controls where the OS X resource fork is stored.

           Due to a spelling bug in all Samba versions older then 4.6.0, this
           option can also be given as fruit:ressource, ie with two s.

           Settings:

                  o   file (default) - use a ._ AppleDouble file compatible
                      with OS X and Netatalk

                  o   xattr - use a xattr, requires a filesystem with large
                      xattr support and a file IO API compatible with xattrs,
                      this boils down to Solaris and derived platforms and ZFS

                  o   stream (experimental) - pass the stream on to the next
                      module in the VFS stack.  Warning: this option should
                      not be used with the streams_xattr module due to the
                      extended attributes size limitations of most
                      filesystems.


       fruit:time machine = [ yes | no ]
           Controls if Time Machine support via the FULLSYNC volume capability
           is advertised to clients.

                  o   yes - Enables Time Machine support for this share. Also
                      registers the share with mDNS in case Samba is built
                      with mDNS support.

                  o   no (default) Disables advertising Time Machine support.

           This option enforces the following settings per share (or for all
           shares if enabled globally):

                  o   durable handles = yes

                  o   kernel oplocks = no

                  o   kernel share modes = no

                  o   posix locking = no


       fruit:time machine max size = SIZE [K|M|G|T|P]
           Useful for Time Machine: limits the reported disksize, thus
           preventing Time Machine from using the whole real disk space for
           backup. The option takes a number plus an optional unit.

           IMPORTANT: This is an approximated calculation that only takes into
           account the contents of Time Machine sparsebundle images. Therefore
           you MUST NOT use this volume to store other content when using this
           option, because it would NOT be accounted.

           The calculation works by reading the band size from the Info.plist
           XML file of the sparsebundle, reading the bands/ directory counting
           the number of band files, and then multiplying one with the other.

       fruit:metadata = [ stream | netatalk ]
           Controls where the OS X metadata stream is stored:

                  o   netatalk (default) - use Netatalk compatible xattr

                  o   stream - pass the stream on to the next module in the
                      VFS stack


       fruit:locking = [ netatalk | none ]


                  o   none (default) - no cross protocol locking

                  o   netatalk - use cross protocol locking with Netatalk


       fruit:encoding = [ native | private ]
           Controls how the set of illegal NTFS ASCII character, commonly used
           by OS X clients, are stored in the filesystem.

           Important: this is known to not fully work with
           fruit:metadata=stream or fruit:resource=stream.

                  o   private (default) - store characters as encoded by the
                      OS X client: mapped to the Unicode private range

                  o   native - store characters with their native ASCII value.
                      Important: this option requires the use of vfs_catia in
                      the VFS module stack as shown in the examples section.


       fruit:veto_appledouble = yes | no
           Note: this option only applies when fruit:resource is set to file
           (the default).

           When fruit:resource is set to file, vfs_fruit may create ._
           AppleDouble files. This options controls whether these ._
           AppleDouble files are vetoed which prevents the client from
           accessing them.

           Vetoing ._ files may break some applications, e.g. extracting Mac
           ZIP archives from Mac clients fails, because they contain ._ files.
           rsync will also be unable to sync files beginning with underscores,
           as the temporary files it uses for these will start with ._ and so
           cannot be created.

           Setting this option to false will fix this, but the abstraction
           leak of exposing the internally created ._ files may have other
           unknown side effects.

           The default is yes.

       fruit:posix_rename = yes | no
           Whether to enable POSIX directory rename behaviour for OS X
           clients. Without this, directories can't be renamed if any client
           has any file inside it (recursive!) open.

           The default is yes.

       readdir_attr:aapl_rsize = yes | no
           Return resource fork size in SMB2 FIND responses.

           The default is yes.

       readdir_attr:aapl_finder_info = yes | no
           Return FinderInfo in SMB2 FIND responses.

           The default is yes.

       readdir_attr:aapl_max_access = yes | no
           Return the user's effective maximum permissions in SMB2 FIND
           responses. This is an expensive computation, setting this to off
           pretends the use has maximum effective permissions.

           The default is yes.

       fruit:wipe_intentionally_left_blank_rfork = yes | no
           Whether to wipe Resource Fork data that matches the special 286
           bytes sized placeholder blob that macOS client create on occasion.
           The blob contains a string "This resource fork intentionally left
           blank", the remaining bytes being mostly zero. There being no one
           use of this data, it is probably safe to discard it. When this
           option is enabled, this module truncates the Resource Fork stream
           to 0 bytes.

           The default is no.

       fruit:delete_empty_adfiles = yes | no
           Whether to delete empty AppleDouble files. Empty means that the
           resource fork entry in the AppleDouble files is of size 0, or the
           size is exactly 286 bytes and the content matches a special
           boilerplate resource fork created my macOS.

           The default is no.

       fruit:zero_file_id = yes | no
           Whether to return zero to queries of on-disk file identifier if the
           client has negotiated AAPL.

           Mac applications and / or the Mac SMB client code expect the
           on-disk file identifier to have the semantics of HFS+ Catalog Node
           Identifier (CNID). Samba provides File-IDs based on a file's
           initial creation date if the option store dos attributes is
           enabled. Returning a file identifier of zero causes the Mac client
           to stop using and trusting the file id returned from the server.

           The default is yes.

EXAMPLES
                   [share]
                vfs objects = catia fruit streams_xattr
                fruit:resource = file
                fruit:metadata = netatalk
                fruit:locking = netatalk
                fruit:encoding = native

AUTHOR
       The original Samba software and related utilities were created by
       Andrew Tridgell. Samba is now developed by the Samba Team as an Open
       Source project similar to the way the Linux kernel is developed.



ATTRIBUTES
       See attributes(7) for descriptions of the following attributes:


       +---------------+-----------------------+
       |ATTRIBUTE TYPE |   ATTRIBUTE VALUE     |
       +---------------+-----------------------+
       |Availability   | service/network/samba |
       +---------------+-----------------------+
       |Stability      | Volatile              |
       +---------------+-----------------------+

NOTES
       Source code for open source software components in Oracle Solaris can
       be found at https://www.oracle.com/downloads/opensource/solaris-source-
       code-downloads.html.

       This software was built from source available at
       https://github.com/oracle/solaris-userland.  The original community
       source was downloaded from
       https://download.samba.org/pub/samba/stable/samba-4.13.14.tar.gz.

       Further information about this software can be found on the open source
       community website at http://www.samba.org/.



Samba 4.13.14                     12/22/2021                      VFS_FRUIT(8)
