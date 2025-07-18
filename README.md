# pikaur

[![Tests](https://github.com/actionless/pikaur/actions/workflows/tests.yml/badge.svg)](https://github.com/actionless/pikaur/actions/workflows/tests.yml) [![Coverage Status](https://coveralls.io/repos/github/actionless/pikaur/badge.svg?branch=master)](https://coveralls.io/github/actionless/pikaur?branch=master) [![Commit Activity](https://img.shields.io/github/commit-activity/m/actionless/pikaur)](https://github.com/actionless/pikaur/graphs/commit-activity) [![Support Project](https://img.shields.io/badge/support%20project-pink?logo=buymeacoffee&logoColor=black)](https://www.paypal.me/actionless)

AUR helper with minimal dependencies. Review PKGBUILDs all in once, next build them all without user interaction.

Inspired by `pacaur`, `yaourt` and `yay`.

Instead of trying to be smarter than pacman (by using `--nodeps`, `--force`, `--ask`, `--noconfirm` and so) it just interactively tells pacman what to do. If pacman asks some unexpected question, the user will be just able to answer it manually.

Notable features:

* build local PKGBUILDs with AUR deps (`-P`/`--pkgbuild`)
* retrieve PKGBUILDs from AUR and ABS (`-G`/`--getpkgbuild`)
* interactively handle common build problems (like untrusted GPG key or checksum mismatch, wrong architecture)
* show unread [Arch news](https://www.archlinux.org/news/ "") before sysupgrade
* [m]anual package selection in [install prompt](#screenshot "") using text editor (ignore unwanted updates or select package provider)
* show AUR package diff and review PKGBUILD and .install files
* [upgrade](#how-to-upgrade-all-the-dev--git-packages-at-once "") `-git`, `-svn` and other dev packages
* AUR package names in shell completion (bash, fish, zsh)
* quickly search&install package by `pikaur <search-query>`

The following pacman operations are extended with AUR capabilities:

* `-S` (build AUR packages, `--needed`, `--ignore` and `--noconfirm` are supported as in pacman, other args are just bypassed to it)
* `-Sw` (build AUR packages but don't install)
* `-Ss` (search or list all AUR packages, `-q` also supported)
* `-Si` (package info)
* `-Su` / `-Syu` (sysupgrade)
* `-Sc` / `-Scc` (build dir/built packages cache clean)
* `-Qu` (query upgradeable, `-q` supported)

Also see `pikaur -Sh`, `-Qh`, `-Ph`, `-Gh` and `-Xh` for pikaur-specific flags.

Pikaur wraps all the pacman options accurately except for `-Syu` which is being split into `-Sy` (to refresh package list first) and `-Su` (to install upgrades after user confirmed the package list or altered it via [M]anual package selection).


* [Installation](#installation "")
* [Run without installation](#run-without-installation "")
* - [Pikaur-Static](#pikaur-static "")
* [File locations](#file-locations "")
* [Config file](#configuration "")
* [FAQ](#faq "")
* [Contributing](#contributing "")
* - [Code](#code "")
* - [Translations](#translations "")
* [Authors](#authors "")


## Installation

```sh
sudo pacman -S --needed base-devel git
git clone https://aur.archlinux.org/pikaur.git
cd pikaur
makepkg -fsri
```

## Screenshot

![Screenshot](https://raw.githubusercontent.com/actionless/pikaur/master/screenshots/package_update.png "Screenshot")



## Run without installation

### Pikaur-Static

To avoid situations during upgrading the system when you can't run Pikaur anymore (for example due breaking changes in Python, Pyalpm or other system dep) it's recommended to have [pikaur-static ⚡️](https://aur.archlinux.org/packages?O=0&SeB=nd&K=pikaur-static&outdated=&SB=p&SO=d&PP=50&submit=Go) installed, which doesn't depend on Python (or Pyalpm) and doesn't conflict with the regular pikaur installation.

You can download it from the [Releases Page](https://github.com/actionless/pikaur/releases)
(or downgrade Python/[other pkg which broke the update] to the previous version if it broke due to update, build+install `pikaur-static` from aur, upgrade python/[that pkg] again).

### Running directly from git repo for development purposes/etc

```sh
git clone https://github.com/actionless/pikaur.git
cd pikaur
python3 ./pikaur.py -S AUR_PACKAGE_NAME
```



## File locations

```sh
~/.cache/pikaur/
├── build/  # build directory (removed after successful build)
├── pkg/  # built packages directory
~/.config/pikaur.conf  # config file
~/.local/share/pikaur/
└── aur_repos/  # keep aur repos there; show diff when updating
    └── last_installed.txt  # aur repo hash of last successfully installed package
```



## Configuration

`~/.config/pikaur.conf`


#### [sync]

##### DevelPkgsExpiration (default: -1)
When doing sysupgrade, count all devel (-git, -svn, -bzr, -hg, -cvs) packages older than N days as being upgradeable.
-1 disables this.
0 means always upgrade.
Passing `--devel` argument will override this option to 0.

##### AlwaysShowPkgOrigin (default: no)
When installing new packages, show their repository name, even if they are coming from one of the official Arch Linux repositories.

##### UpgradeSorting (default: versiondiff)
When upgrading packages, sort them by `versiondiff`, `pkgname` or `repo`.

##### ShowDownloadSize (default: no)
When installing repository packages, show their download size.

##### IgnoreOutofdateAURUpgrades (default: no)
When doing sysupgrade ignore AUR packages which have `outofdate` mark.


#### [build]

##### GpgDir (default: ) (root default: /etc/pacman.d/gnupg)
Provides an override path for the GPG home directory used when validating aur package sources.
See explanations of `--homedir` and `${GNUPGHOME}` in the gpg man pages for more details.
Will be overridden by `--build_gpgdir` argument.

##### KeepBuildDir (default: no)
Don't remove `~/.cache/pikaur/build/${PACKAGE_NAME}` directory between the builds.
Will be overridden by `-k/--keepbuild` flag.

##### KeepDevBuildDir (default: yes)
When building dev packages (`-git`, `-svn`, etc),
don't remove `~/.cache/pikaur/build/${PACKAGE_NAME}` directory between the builds.
`No` value will be overridden by `KeepBuildDir` option and `-k/--keepbuild` flag.

##### KeepBuildDeps (default: no)
Don't remove build dependencies between and after the builds.
Will be overridden by `--keepbuilddeps` flag.

##### SkipFailedBuild (default: no)
Always skip the build if it fails and don't show recovery prompt.

##### DynamicUsers (default: never) [root|never|always]
When to isolate the build using systemd dynamic users.
(`root` - only when running as root)
Will be overridden by `--dynamic-users` flag.

##### IgnoreArch (default: no)
Ignore specified architectures (`arch`-array) in PKGBUILDs.


#### [review]

##### DontEditByDefault (default: no)
Always default to no when prompting to edit PKGBUILD and install files.

##### NoEdit (default: no)
Don't prompt to edit PKGBUILD and install files.
Will be overridden by `--noedit` and `--edit` flags.

##### NoDiff (default: no)
Don't prompt to show the build files diff.
Will be overridden by `--nodiff` flag.

##### GitDiffArgs (default: --ignore-space-change,--ignore-all-space)
Flags to be passed to `git diff` command when reviewing build files.
Should be separated by commas (`,`).

##### DiffPager (default: auto)
Wherever to use `less` pager when viewing AUR packages diff. Choices are `always`, `auto` or `never`.

##### HideDiffFiles (default: .SRCINFO)
Hide `git diff` for file paths, separated by commas (`,`).


#### [colors]

Terminal colors, from 0 to 15:

##### Version (default: 10)
##### VersionDiffOld (default: 11)
##### VersionDiffNew (default: 9)


#### [ui]

##### RequireEnterConfirm (default: yes)
Require enter key to be pressed when answering questions.

##### PrintCommands (default: no)
Print each command which pikaur is currently spawning.

##### GroupByRepo (default: yes)
Groups official packages by repository when using commands like `pikaur -Ss <query>` or `pikaur <query>`.

##### AurSearchSorting (default: hottest)
Sorting key for AUR packages when using commands like `pikaur -Ss <query>` or `pikaur <query>`. Accepts `hottest`, `numvotes`, `lastmodified`, `popularity`, `pkgname`. Only `pkgname` is sorted ascendingly. The metric for `hottest` is weighted by both `numvotes` and `popularity`.

##### DisplayLastUpdated (default: no)
Display the date a package is last updated on search results when using commands like `pikaur -Ss <query>` or `pikaur <query>`.

##### ReverseSearchSorting (default: no)
Reverse search results of the commands like `pikaur -Ss <query>` or `pikaur <query>`.

##### WarnAboutPackageUpdates (default: )
Comma-separated list of packages names or globs, which upgrade should have additional warning message in the UI.

##### WarnAboutNonDefaultPrivilegeEscalationTool (default: yes)
Print warning when using privilege escalation tool other than `sudo`.


#### [misc]

##### PacmanPath (default: pacman)
Path to pacman executable.
Will be overriden by `--pacman-path` flag.

##### PreserveEnv (default: `PKGDEST,VISUAL,EDITOR,http_proxy,https_proxy,ftp_proxy,HTTP_PROXY,HTTPS_PROXY,FTP_PROXY,ALL_PROXY`)
Preserve environment variables of current user when running pikaur as root (comma-separated).
Will be overriden by `--preserve-env` flag.

##### PrivilegeEscalationTool (default: sudo)
A tool used to escalate user privileges. Currently supported options are `sudo` and `doas`.

##### PrivilegeEscalationTarget (default: pikaur)
Choices: pikaur, pacman.
In case of elevating privilege for pacman - pikaur would ask for password every time pacman runs.

##### UserId (default: 0)
User ID to run makepkg if pikaur started from root.
0 - means disabled, not that it will use uid=0.
Setting this option would override DynamicUsers settings and force changing to this UID instead of a dynamic one.
The config value would be overriden by `--user-id` flag.

##### GroupId (default: 0)
Group ID to run makepkg if pikaur started from root.
0 - means disabled, not that it will use gid=0.
The config value would be overriden by `--group-id` flag.

##### CachePath (default: ~/.cache)
Path to package cache location.
Will be overridden by `--xdg-cache-home` argument
or environment variable `XDG_CACHE_HOME`, if set.

##### DataPath (default: ~/.local/share)
Path to database location.
Will be overridden by `--xdg-data-home` argument
or environment variable `XDG_DATA_HOME`, if set.

#### [network]

##### AurUrl (default: https://aur.archlinux.org)
AUR Host.

##### NewsUrl (default: https://archlinux.org/feeds/news/)
Arch Linux News URL, useful for users of Parabola or other Arch derivatives.

##### Socks5Proxy (default: )
Specify a socks5 proxy which is used to get AUR package information.

The format is `[host[:port]]`, and the default port is 1080.
PySocks module (`python-pysocks` package) should be installed in order to use this option.

Note that any downloads by `pacman`, `git` or `makepkg` will NOT use this proxy.
If that's needed, setting proxy options in their own config files will take effect
(such as `~/.gitconfig`, `~/.curlrc`).

##### AurHttpProxy (default: )
Specify a HTTP proxy which is used to get AUR package information and to `git`-clone from AUR.

Note that any downloads by `pacman`, `git` (inside the build) or `makepkg` will NOT use this proxy.
If that's needed, setting proxy options in their own config files will take effect
(such as `env HTTP_PROXY=`, `~/.gitconfig`, `~/.curlrc`).

##### AurHttpsProxy (default: )
Specify a HTTPS proxy which is used to get AUR package information and to `git`-clone from AUR.

Note that any downloads by `pacman`, `git` (inside the build) or `makepkg` will NOT use this proxy.
If that's needed, setting proxy options in their own config files will take effect
(such as `env HTTPS_PROXY=`, `~/.gitconfig`, `~/.curlrc`).



## FAQ


##### How to upgrade all the dev (-git) packages at once?

`pikaur -Sua --devel --needed`

(`--needed` option will make sure what the same package version won't be rebuilt again and `-a/--aur` will ensure what only AUR packages will be upgraded)


##### How to manually remove unneeded dependencies?

Pikaur is not needed for that, use just Pacman itself:

`sudo pacman -Rs $(pacman -Qtdq)` (however `pikaur -Rs ...` would work as well if you lazy to type `sudo` :) )


##### How to override default source directory, build directory or built package destination?

Set `SRCDEST`, `BUILDDIR` or `PKGDEST` accordingly in `makepkg.conf`.

For more info see `makepkg` documentation.


##### How to clean old or uninstalled AUR packages in ~/.cache/pikaur/pkg?

Use `paccache(8)` with the `--cachedir` option.

To clean them up automatically, you may:

- use a pacman hook.  Start with the provided
  Copy `/usr/share/pikaur/examples/pikaur-cache.hook` to
  `/usr/share/libalpm/hooks/pikaur-cache-cleanup.hook`,
  remember to update the cache's path.

- use a systemd service & timer (provided `pikaur-cache.service` and
  `pikaur-cache.timer`).  Configure it with `systemctl --user edit
  --full pikaur-cache.service` and activate it with `systemctl --user
  enable --now pikaur-cache.timer`.


##### How to restore original PKGBUILD after editing?

Go to the package's directory, `cd ~/.local/share/pikaur/aur_repos/${PACKAGE_NAME}`.
Review the current PKGBUILD file changes with `git diff` and then reset with  `git checkout -- '*'`.


##### How to see upgrade list without syncing the database? (like "checkupdates" tool from pacman)

Actually use `checkupdates` tool to check the repo updates and use pikaur only for AUR (`-a`/`--aur` switch):

```
checkupdates ; pikaur -Qua 2>/dev/null
```

##### Pikaur slow when running it as root user (or via sudo)

If you find the command takes a long time to initialize, make sure to periodically clear your cache: `pikaur -Scc`. Root pikaur is using SystemD Dynamic Users to isolate build process from the root, and it takes some time to change the owner of build cache to dynamic temporary user.

##### How to migrate from Yay?

This will migrate the cache of what AUR packages have been installed, so you can still see a Git diff for the next update of each package:

```sh
mv ~/.cache/yay/* ~/.local/share/pikaur/aur_repos/
find ~/.local/share/pikaur/aur_repos -mindepth 1 -maxdepth 1 -type d | xargs -r -I '{}' -- sh -c 'cd "{}" && git rev-parse HEAD > last_installed.txt'
```

##### How to downgrade a package?

This will show a list of commits to choose one to downgrade to.

```sh
pikaur -G <package>  # retrieve package sources
cd <package>
git log  # choose <commit> from the list
git checkout <commit>
pikaur -Pi  # build and install older version
```

##### How to add additional trusted keys when building with systemd dynamic users?
When using systemd dynamic users, by default, there is not a persistent user or gpg home directory. You can set the path to a persistent gpg home directory using the cli argument `--build_gpgdir`. Alternatively, you can set a permanent default with the configuration option `[build] gpgdir` in the root pikaur configuration file `/root/.config/pikaur.conf` The below example configures makepkg to use a hypothetical gpg home directory at `/etc/pikaur.d/gnupg` when validating source files.

```ini
[build]
gpgdir=/etc/pikaur.d/gnupg
```

You can initialize a minimal gnupghome at the example path by executing the below commands as root.

```sh
export GNUPGHOME="/etc/pikaur.d/gnupg"
mkdir -p "${GNUPGHOME}"
gpg --batch --passphrase '' --quick-gen-key "pikaur@localhost" rsa sign 0
```

## Contributing


### Code

You can start from [this list of issues](https://github.com/actionless/pikaur/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22 ""). Grep-ing `@TODO` comments also useful if you're itching to write something.

To install development deps, run:

```sh
pikaur -Pi ./PKGBUILD_dev_deps
```

#### Running CI locally

##### Linters (code quality check)

```sh
make -j lint
```

##### Tests

```sh
./maintenance_scripts/docker_test.sh
```

See also `./maintenance_scripts/docker_test.sh --help` for more options.

For example to run a single test inside docker:

```sh
./maintenance_scripts/docker_test.sh --local 1 pikaur_test.test_sysupgrade.SysupgradeTest.test_devel_upgrade
```


### Translations

To start working on a new language, say `hi_IN` (Indian Hindi):
1) add it to the `Makefile` `LANGS` variable and run `make`.
2) Then translate `locale/hi_IN.po` using your favorite PO editor (for example `gtranslator`).
3) Run `make` every time the Python code strings change or the `.po` is modified.



## Authors

To see the list of authors, use this command inside pikaur git repository directory:

```sh
git log --pretty=tformat:"%an <%ae>" | sort -u
```

### Special thanks

@AladW ([aurutils](https://github.com/AladW/aurutils)), @morganamilo ([yay](https://github.com/Jguer/yay)) during the early stages of Pikaur development.
And [all the other issue contributors](https://github.com/actionless/pikaur/issues?utf8=%E2%9C%93&q=is%3Aissue+-author%3Aactionless) for helping in triaging the bugs and clearing up feature requirements.
