# tapeimgr

**Tapeimgr** is a software application that sequentially reads all files from a data tape. After the extraction is done it also generates a checksum file with SHA-512 hashes of the extracted files. *Tapeimgr* is completely format-agnostic: it only extracts the raw byte streams. It is up to the user to figure out the format of the extracted files (e.g. a TAR archive), and how to further process or open them.

In short, *tapeimgr* tries to read sequential files from a tape until its logical end is reached. For each successive file, it automatically determines its block size using an iterative procedure.

Internally *tapeimgr* wraps around the Linux [*dd*](http://manpages.ubuntu.com/manpages/bionic/man1/dd.1.html) and [*mt*](http://manpages.ubuntu.com/manpages/bionic/man1/mt.1.html) tools. 

## Warnings

At this stage *tapeimgr* has only had limited testing with a small number of DDS-1 and DLT-IV tapes. Use at your own risk, and please [report any unexpected behaviour using the issue tracker](https://github.com/KBNLresearch/tapeimgr/issues).

For now *tapeimgr* can only read tapes that were written in fixed block mode; tapes written in variable block mode may result in unexpected behaviour (if ). Support of variable block mode [may be added to a future release](https://github.com/KBNLresearch/tapeimgr/issues/28). Note that older tapes were most likely written in fixed block mode, as variable block mode is typically only supported by more recent tape drives.

## System requirements

*Tapeimgr* is only available for Linux (but you would probably have a hard time setting up a tape drive on Windows to begin with). So far it has been tested with Ubuntu 18.04 LTS (Bionic) and Linux Mint 18.3, which is based on Ubuntu 16.04 (Xenial). In addition it has the following dependencies (many distros have most or all of these installed by default):

- **Python 3.2 or more recent** (Python 2.x is not supported)

- **Tkinter**. If *tkinter* is not installed already, you need to use the OS's package manager to install (there is no PyInstaller package for *tkinter*). If you're using *apt* this should work:

        sudo apt-get install python3-tk

- **dd** and **mt** (but these are available by default on all Linux platforms)


## Installation

### Preparation: add default user to tape group

By default, Linux requires special permissions to access tape devices. So before proceeding any further, use the command below (replacing `$USER` with the name of the user who will be using tapeimgr):

    sudo adduser $USER tape

The user is now added to the 'tape' [system group](https://wiki.debian.org/SystemGroups). Now log out and then log in again for the changes to take effect.

### Global install

For a global (all-users) installation run the following command:

    sudo pip3 install tapeimgr

Then run:

    sudo tapeimgr-config

If all goes well this should result in the following output:

    INFO: writing configuration file /etc/tapeimgr/tapeimgr.json
    INFO: creating desktop file /home/johan/Desktop/tapeimgr.desktop
    INFO: creating desktop file /usr/share/applications/tapeimgr.desktop
    INFO: tapeimgr configuration completed successfully!

### User install

Use the following command for a single-user installation:

    pip3 install --user tapeimgr

Then run:

    ~/.local/bin/tapeimgr-config

Result:

    INFO: writing configuration file /home/johan/.config/tapeimgr/tapeimgr.json
    INFO: creating desktop file /home/johan/Desktop/tapeimgr.desktop
    INFO: creating desktop file /home/johan/.local/share/applications/tapeimgr.desktop
    INFO: tapeimgr configuration completed successfully!

*Tapeimgr* is now ready to roll!

In the instructions that follow below, it is assumed that you have a functioning tape device attached to your machine, and that a tape is loaded (i.e. inserted in the drive).

## GUI operation

You can start *tapeimgr* from the OS's main menu (in Ubuntu 18.04 the *tapeimgr* item is located under *System Tools*), or by clicking the *tapeimgr* shortcut on the desktop. Depending on your distro, you might get an "Untrusted application launcher" warning the first time you activate the shortcut. You can get rid of this by clicking on "Mark as Trusted". On startup the main *tapeimgr* window appears:

![](./img/tapeimgr-1.png)

Use the *Select Output Directory* button to navigate to an (empty) directory where the extracted files are to be stored. Press the *Start* button to start the extraction. You can monitor the progress of the extraction procedure in the progress window:

![](./img/tapeimgr-2.png)

Note that the screen output is also written to a log file in the output directory. A prompt appears when the extraction is finished:

![](./img/tapeimgr-success.png)

If the extraction finished without any errors, the output directory now contains the following files:

![](./img/tapeimgr-files.png)

Here, **file000001.dd** through **file000003.dd** are the extracted files; **checksums.sha512** contains the SHA512 checksums of the extracted files, **metadata.json** contains some basic metadata and **tapeimgr.log** is the log file.

### Options

If needed you can use the folowing options to customize the behaviour of *tapeimgr*:

|Option|Description|
|:-|:-|
|**Tape Device**|Non-rewind tape device (default: `/dev/nst0`).|
|**Initial Block Size**|Initial block size in bytes (must be a multiple of 512). This is used as a starting value for the iterative block size estimation procedure. Block sizes smaller than 4096 are reported to give poor performance (source: [*forensicswiki*](https://www.forensicswiki.org/wiki/Dd)), and this option can be useful to speed up the extraction process in such cases. Note that the user-specified value of **Initial Block Size** is ignored if the **Fill failed blocks** option (see below) is activated.|
|**Files**|Comma-separated list of files to extract. For example, a value of `2,3` will only extract the 2nd and 3rd files from the tape, and skip everything else. By default this field is empty, which extracts all files).|
|**Prefix**|Output prefix (default: `file`).|
|**Extension**|Output file extension (default: `dd`).|
|**Fill failed blocks**|Fill blocks that give read errors with null bytes. When this option is checked, *tapeimgr* calls *dd* with the flags `conv=noerror,sync`. The use of these flags is often recommended to ensure a forensic image with no missing/offset bytes in case of read errors (source: [*forensicswiki*](https://www.forensicswiki.org/wiki/Dd)), but when used with a block size that is larger than the actual block size it will generate padding bytes that make the extracted data unreadable. Because of this, any user-specified value of  the **Initial Block Size** setting (see above) is ignored when this option is used. **WARNING: this option may result in malformed output if the actual block size is either smaller than 512 bytes, and/or if the block size is not a multiple of 512 bytes! (I have no idea if this is even possible?).**|
|**Identifier**|Unique identifier. You can either enter an existing identifier yourself, or press the *UUID* button to generate a [Universally unique identifier](https://en.wikipedia.org/wiki/Universally_unique_identifier).|
|**Description**|A text string that describes the tape (e.g. the title that is written on its inlay card).|
|**Notes**|Any additional info or notes you want to record with the tape.|

## Command-line operation

It is also possible to invoke *tapeimgr* with command-line arguments. The general syntax is:

    tapeimgr [-h] [--version] [--fill] [--device DEVICE]
                [--blocksize SIZE] [--files FILES] [--prefix PREF]
                [--extension EXT] [--identifier IDENTIFIER]
                [--description DESCRIPTION] [--notes NOTES]
                dirOut

Here `dirOut` is the output directory. So, the command-line equivalent of the first GUI example is:

    tapeimgr /home/bcadmin/test/

This will extract the contents of the tape to directory */home/bcadmin/test/*, using the default options. Note that for a user install, you may need to provide the full path to *tapeimgr*, i.e.:

    ~/.local/bin/tapeimgr /home/bcadmin/test/

### Options

As with the GUI interface you can customize the default behaviour by using one or more of the following optional arguments:

|Argument|Description|
|:-|:-|
|`-h, --help`|show help message and exit|
|`--version, -v`|show program's version number and exit|
|`--device DEVICE, -d DEVICE`|Non-rewind tape device (default: `/dev/nst0`).|
|`--blocksize SIZE, -b SIZE`|Initial block size in bytes (must be a multiple of 512). This is used as a starting value for the iterative block size estimation procedure. Block sizes smaller than 4096 are reported to give poor performance (source: [*forensicswiki*](https://www.forensicswiki.org/wiki/Dd)), and this option can be useful to speed up the extraction process in such cases. Note that the user-specified value of `--blocksize` is ignored if the `--fill` option (see below) is activated.|
|`--files FILES, -s FILES`|Comma-separated list of files to extract. For example, a value of `2,3` will only extract the 2nd and 3rd files from the tape, and skip everything else. By default this field is empty, which extracts all files).|
|`--prefix PREF, -p PREF`|Output prefix (default: `file`).|
|`--extension EXT, -e EXT`|Output file extension (default: `dd`).|
|`--fill, -f`|Fill blocks that give read errors with null bytes. When this option is checked, *tapeimgr* calls *dd* with the flags `conv=noerror,sync`. The use of these flags is often recommended to ensure a forensic image with no missing/offset bytes in case of read errors (source: [*forensicswiki*](https://www.forensicswiki.org/wiki/Dd)), but when used with a block size that is larger than the actual block size it will generate padding bytes that make the extracted data unreadable. Because of this, any user-specified value of the `--blocksize`setting (see above) is ignored when this option is used. **WARNING: this option may result in malformed output if the actual block size is either smaller than 512 bytes, and/or if the block size is not a multiple of 512 bytes! (I have no idea if this is even possible?).**|
|`--identifier IDENTIFIER, -i IDENTIFIER`|Unique identifier. You can either enter an existing identifier yourself, or enter special value `@uuid` to generate a [Universally unique identifier](https://en.wikipedia.org/wiki/Universally_unique_identifier).|
|`--description DESCRIPTION, -c DESCRIPTION `|A text string that describes the tape (e.g. the title that is written on its inlay card).|
|`--notes NOTES, -n NOTES`|Any additional info or notes you want to record with the tape.|

## Metadata file

The file *metadata.json* contains metadata in JSON format. Below is an example:

    {
        "acquisitionEnd": "2019-01-21T13:26:38.813304+01:00",
        "acquisitionStart": "2019-01-21T13:25:53.570770+01:00",
        "checksumType": "SHA-512",
        "checksums": {
            "file000001.dd": "e58279519cd394870f7d39fe59d722bf85c64fb95a9b4c8a893fde0a606f5e270529d17d0598d9e703f9b259a2355c91aaa3721249a64982e580f2f18e6e52f5",
            "file000002.dd": "19f200700afeab90d45d3beec0cdaf5290ca517574b9049feee80d1257c5d11677d9ecd101c31e30b02d58b4d84c8cac0ea7326d10342d1d7ea4ce40dde663ca",
            "file000003.dd": "9f4f5ea7cc3639c07ca88b4dcda9b976d2802d229a26415d960962d4fdc2d920ea1ce554343dfc5f22c3c616cad3977e24ddf33c86417dc0112a8560e2f1e75f"
        },
        "description": "Test tape October 25 2018",
        "extension": "dd",
        "files": "",
        "fillBlocks": false,
        "identifier": "2d257dec-1d77-11e9-9594-2c4138b5272c",
        "initBlockSize": 512,
        "notes": "This tape only contains some old rubbish. Created specifically for testing tapeimgr.",
        "prefix": "file",
        "successFlag": true,
        "tapeDevice": "/dev/nst0",
        "tapeimagrVersion": "0.4.0b1"
    }

## Configuration file

*Tapeimgr*'s internal settings (default values for output file names, tape device, etc.) are defined in a configuration file in Json format. For a global installation it is located at */etc/tapeimgr/tapeimgr.json*; for a user install it can be found at *~/.config/tapeimgr/tapeimgr.json*. The default configuration is show below:

    {
        "checksumFileName": "checksums.sha512",
        "defaultDir": "",
        "extension": "dd",
        "files": "",
        "fillBlocks": "False",
        "initBlockSize": "512",
        "logFileName": "tapeimgr.log",
        "metadataFileName": "metadata.json",
        "prefix": "file",
        "tapeDevice": "/dev/nst0",
        "timeZone": "Europe/Amsterdam"
    }

You can change *tapeimgr*'s default settings by editing this file. Most of the above settings are self-explanatory, with the exception of the following:

- **defaultDir**: this allows you to change the default file path that is opened after pressing *Select Output Directory*. By default *tapeimgr* uses the current user's home directory. However, if *defaultDir* points to a valid directory path, that directory is used instead.

- **timeZone**: time zone string that is used to correctly format the *acquisitionStart* and *acquisitionEnd* date/time strings. You can adapt it to your own location by using the *TZ database name* from [this list of tz database time zones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).

Note that it is *not* recommended to change the value of *initBlockSize*, as it may result in unexpected behaviour. If you accidentally messed up the configuration file, you can always restore the original one by running the *tapeimgr-config* tool again.

## Uninstalling tapeimgr

To remove *tapeimgr*, first run the *tapeimgr-config* with the `--remove` flag to remove the configuration file and the start menu and desktop files. For a global install, run:

    sudo tapeimgr-config --remove

For a user install, run:

    ~/.local/bin/tapeimgr-config --remove

The resulting output (shown below for a user install):

    INFO: removing configuration file /home/johan/.config/tapeimgr/tapeimgr.json
    INFO: removing configuration directory /home/johan/.config/tapeimgr
    INFO: removing desktop file /home/johan/Desktop/tapeimgr.desktop
    INFO: removing desktop file /home/johan/.local/share/applications/tapeimgr.desktop
    INFO: tapeimgr configuration completed successfully!

Then remove the Python package with following command (global install):

    sudo pip3 uninstall tapeimgr

For a user install use this:

    pip3 uninstall tapeimgr

## Testing tapeimgr witout a tape drive

If you want to test *tapeimgr* without having access to a physical tape drive, check out [*mhvtl*, the Linux based Virtual Tape Library](https://github.com/markh794/mhvtl) (and also [these rough notes](https://gist.github.com/bitsgalore/f02e37aba6de988c8ae173a1307c73ff) which explain how to install mhvtl as well as its basic usage).

**Note**: based on some limited tests, it seems that rewinding a virtual tape in *mhvtl* with the *mt* command (which is used by *tapeimgr*) doesn't actually rewind the tape at all! This has the effect that after a tape has been processed by *tapeimgr*, running *tapeimgr* again on the same device will cause *mt* to freeze (and *tapeimgr* with it). A workaround is to unload the tape, and then load it again using something like this:

    mtx -f /dev/sg11 uload 1 0
    mtx -f /dev/sg11 load 1 0

After this the virtual tape device works normally again.

## Contributors

Written by Johan van der Knijff. 

## License

*Tapeimgr* is released under the  Apache License 2.0.
