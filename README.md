# mkschroot #


A simple script for making `schroot` environments from a JSON configuration file. The idea is to let you set up multiple `schroot` images simply and quickly and to be able to check in the configuration that makes this repeatable across many machines easily.

This first version isn't so smart. It assumes that you're using a 64 bit host machine, and doesn't check that you're on Debian as it should (it only uses `debootstrap` to build the chroot environment). It's probably also highly Ubuntu specific right now. To add to the fun it's not been very thoroughly tested yet either.

To use it you must install a couple of things:

    apt-get install debootstrap schroot
    pip install mkschroot


## Using mkschroot ##

You just need to pass `mkschroot` a configuration file:

    mkschroot ~/chroots/example.json

Don't run with root privileges. If root access is required then `sudo` rights will be requested.

If the `schroot` configuration has been changed then a new configuration file will be generated. If the image doesn't exist it will be created and have the packages installed on it, if it does exist it will be updated to the latest package list and package versions.

You really want to have a local `apt-cacher` if you're going to be making a lot of images.


### Configuring mkschroot ###

The configuration file needs to be JSON. A configuration file might look like the below:

    {
        "root": "/mnt/files2/chroot",
        "source": "http://th.archive.ubuntu.com/ubuntu/",
        "http-proxy": "http://angelo:3142/",
        "base-packages": ["lsb-release", "openssh-client"],
        "defaults": {
            "sources": {
                "universe": {}
            },
            "conf": {
                "root-users": ["kirit"],
                "users": ["kirit"]
            }
        },
        "schroot": {
            "build-lucid64": {
                "release": "lucid",
                "packages": [
                    "g++", "libbz2-dev", "libssl-dev", "python-dev", "uuid-dev",
                    "libboost-dev", "subversion", "git-core"
                ]
            },
            "root-ca-kirit": {
                "release": "precise",
                "packages": ["openssl"],
                "conf": {
                    "personality": "linux32"
                }
            }
        }
    }

The base options are:

* `base-packages`: Packages that are to be installed in all chroots.
* `defaults`: Default values for individual chroot configurations.
* `http-proxy`: A HTTP proxy (probably an apt-cache) that should be used by `debootstrap` to fetch packages.
* `root`: The directory where you want the chroots to be created in by default (override this using the `directory` setting within a chroot).
* `schroot`: The schroot environments to be created.
* `source`: Where the packages can be installed from. This is required.

A chroot configuration is described by a structure like the following:

    {
        "release": "lucid",
        "variant": "buildd",
        "packages": ["g++"],
        "sources": {
            "universe": {}
        },
        "conf": {
            "root-users": ["kirit'"],
            "users": ["kirit"]
        }
    }

* `release`: The operating system version you wish to make use of.
* `variant`: If specified then the variant is passed to `debootstrap` so that the right base image options are used. The variant name is also used in the `schroot` configuration file so that the right start up options are used when the chroot is started. Note that some `schroot` options (notably fstab for buildd) won't work until they're configured to match your system.
* `conf`: The fields used for the schroot configuration file (in `/etc/schroot/chroot.d/`). The following fields are required: `root-users`, `users`, and the following are optional: `description`, `type`, `personality`, `directory`. Do read the part about common fields though.
* `sources`: Extra sources that are to be added to the chroot. See sources below.
* `packages`: Packages that need to be installed into the chroot using `apt-get`. These are combined with the base-packages.

Note that if you change the variant of an existing image no attempt is made to correct the packages that are installed.


#### Common configuration items ####

Generally many of the chroots that you want will share a good deal of configuration between machines. To help with this a default `schroot` configuration can be given which will then have values overridden by the specific `schroot`s that you request be made.

This means that the values in the `defaults` key will be used, then any values in the specific `schroots` key will be added in, and finally a few defaults will be generated by `mkschroot`.

* `description`: The release name together with personality name.
* `directory`: The `root` option will have the schroot name added to it.
* `type`: Always `directory`.
* `personality`: The same as the host personality (currently hard coded to 64 bits)


#### /etc/apt/apt.conf ####

If the host environment has an `/etc/apt/apt.conf` file then it is assumed that this should also be in the schroot environments. If the file content differ then the host file is copied into the schroot and `apt-get update` is run within the chroot.


#### Sources ####

If other sources are needed then they can be specified at either the defaults level or for an individual schroot. The name is used as the component name, and this will generate a file in `/etc/apt/sources.list.d/` with the component pointing at the source. If a source field is given than that is used, otherwise the global source is used.

For example:

    sources: {
        "universe": {},
        "private-example": {"source": "http://example.com/"}
    }

Will generate two files, `universe.list`:

    deb http://th.archive.ubuntu.com/ubuntu/ universe

And private-example.list`:

    deb http://example.com/ private-example

Note that the value for a source component must be a JSON object, even if empty.
