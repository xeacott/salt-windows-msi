# Windows MSI installer build toolkit

This project creates a Salt Minion msi installer using [WiX][WiX_link].

The focus is on 64bit, unattended install, Python 2.

## Introduction

[Introduction on Windows installers](http://unattended.sourceforge.net/installers.php)

An msi installer allows to unattended/silent installations, meaning without opening any window, while still providing
customized values for e.g. master hostname, minion id, installation path, using the following generic command:

> msiexec /i *.msi /qb! PROPERTY1=VALUE1 PROPERTY2=VALUE2

Values may be quotes (example: `PROPERTY3="VALUE3"`)

Example Set a Master to salt2, the old master key, if present, wil be reused:

> msiexec /i YOUR.msi MASTER=salt2

Example Set a Master to salt2 with a new key:

> msiexec /i YOUR.msi MASTER=salt2 MASTER_KEY=MIIBIjA...2QIDAQAB

Example Remove including configuration

> MsiExec.exe /X{3478B5D9-B494-438F-97A4-160ECE1CF345} KEEP_CONFIG=0

## Features

- Uninstall leaves configuration by default, optionally removes configuration with `msiexec /x KEEP_CONFIG=0`
- Creates a very verbose log file, by default named %TEMP%\MSIxxxxx.LOG, where xxxxx are 5 random lowercase letters and numbers. The name of the log can be specified with `msiexec /log example.log`
- Upgrades NSIS installations
- Change installation directory __BLOCKED BY__ [issue#38430](https://github.com/saltstack/salt/issues/38430)

Minion-specific msi-properties:

  Property              |  Default                | Comment
 ---------------------- | ----------------------- | ------
 `MASTER`               | `salt`                  | The master (name or IP). Only a single master. May also be read from kept config.
 `MASTER_KEY`           |                         | The master public key. See below.
 `ZMQ_filtering`        | `False`                 | Set to `True` if the master requires zmq_filtering
 `MINION_ID`            | Hostname                | May also be read from kept config.
 `START_MINION`         | `1`                     | Set to `""` to prevent the start of the salt-minion service. (`START_MINION=""`)
 `KEEP_CONFIG`          | `1`                     | Set to `0` to remove configuration on uninstall.
 `MINION_CONFIGFILE`    | `C:\salt\conf\minion`   | The minion config file and directory (minion.d)      __DO NOT CHANGE (yet)__
 `INSTALLFOLDER`        | `c:\salt\`              | Where to install the Minion  __DO NOT CHANGE (yet)__

These files and directories are regarded as config and kept:

- C:\salt\conf\minion
- C:\salt\conf\minion.d\
- c:\salt\var\cache\salt\minion\extmods\
- c:\salt\var\cache\salt\minion\files\

Master and id are read from kept configuration from file `C:\salt\conf\minion`
and then from all files `C:\salt\conf\minion.d\*.conf` (except `_schedule.conf`) in alphabetical order.

You can set a new master with `MASTER`. This will overrule the master in a kept configuration.

You can set a new master public key with `MASTER_KEY`, but you must convert it into one line:

- Remove the first and the last line (`-----BEGIN PUBLIC KEY-----` and `-----END PUBLIC KEY-----`).
- Remove linebreaks.
- From the default public key file (458 bytes), the one-line key has 394 characters.
- Example below.

## Target client requirements

The target client is where the installer is deployed.

- 64bit
- Windows 7 (workstation), Server 2012 (domain controller), or higher.

## Build client requirements

The build client is where the msi installer is built.

- 64bit Windows 10
- Salt clone in `c:\git\salt\`
- This clone in `c:\git\salt-windows-msi\`
- .Net 3.5 SDK (for WiX)<sup>*</sup>
- Microsoft_VC90_CRT_x86_x64.msm from Visual Studio 2008 SP2 in `c:\salt_msi_resources\`
- [Wix 3.11](http://wixtoolset.org/releases/)<sup>**</sup>
- [Build tools 2015](https://www.microsoft.com/en-US/download/confirmation.aspx?id=48159)<sup>**</sup>

<sup>*</sup> `build_env.cmd` will open `optionalfeatures` if necessary.

<sup>**</sup> downloaded and installed by `build_env.cmd` if necessary.

Optionally: [Visual Studio Extension](https://marketplace.visualstudio.com/items?itemName=WixToolset.WiXToolset)

### Step 1: build the exe installer

[Building and Developing on Windows](https://docs.saltstack.com/en/latest/topics/installation/windows.html#building-and-developing-on-windows)

Prepare

    cd c:\git\salt\pkg\windows
    git checkout v2018.3.4
    git checkout .
    git clean -fd

until `git status` returns

    HEAD detached at v2018.3.4
    nothing to commit, working tree clean

then execute both commands (always)

    clean_env.bat
    build.bat

### Step 2: build the msi installer

You need the exe installer.

    cd c:\git\salt-windows-msi
    build_env.cmd
    build.cmd

You should see screen output containing:

    Build succeeded
      warning CNDL1150
      warning CNDL1150
        2 Warning(s)
        0 Error(s)

## How to program the msi builder toolkit

The remainder is documentation how to program the msi build toolkit.

### Directory structure

- msbuild.proj: main MSbuild file.
- msbuild.d/: contains MSbuild resource files:
  - BuildDistFragment.targets: find files (from the extracted distribution?).
  - DownloadVCRedist.targets: (ORPHANED) download Visual C++ redistributable for bundle.
  - Minion.Common.targets: set version and platform parameters.
- salt-windows-msi.sln: Visual Studio solution file, included in msbuild.proj.
- wix.d/: installer sources:
  - MinionConfigurationExtension/: C# for custom actions:
    - MinionConfiguration.cs
  - MinionEXE/: (ORPHANED) create a bundle.
  - MinionMSI/: create a msi:
    - dist-$(TargetPlatform).wxs: found files (from the distribution zip file?).
    - MinionConfigurationExtensionCA.wxs: custom actions boilerplate.
    - MinionMSI.wixproj: msbuild boilerplate.
    - Product.wxs: main file, that e.g. includes the UI description.
    - service.wxs: salt-minion Windows Service using ssm.exe, the Salt Service Manager.
    - servicePython.wxs: (EXPERIMENTAL) salt-minion Windows Service
      - requires [saltminionservice](https://github.com/saltstack/salt/blob/167cdb344732a6b85e6421115dd21956b71ba25a/salt/utils/saltminionservice.py) or [winservice](https://github.com/saltstack/salt/blob/3fb24929c6ebc3bfbe2a06554367f8b7ea980f5e/salt/utils/winservice.py) [Removed](https://github.com/saltstack/salt/commit/8c01aacd9b4d6be2e8cf991e3309e2a378737ea0)
    - SettingsCustomizationDlg.wxs: Dialog for the master/minion properties.
    - WixUI_Minion.wxs: UI description, that includes the dialog.

### Naming conventions

Postfix  | Example                            | Meaning
-------- | ---------------------------------- | -------
`_IMCAC` | `ReadConfig_IMCAC`                 | Immediate custom action written in C#
`_IMCAX` | `SetMinionIdToHostname_IMCAX`      | Immediate custom action written in XML
`_DECAC` | `Uninstall_excl_Config_DECAC`      | Deferred custom action written in C#
`_CADH`  | `Uninstall_excl_Config_CADH`       | Custom action data helper (only for deferred custom action)
`COMP_` | `COMP_NukeBin`          | Component
`DIR_`  | `DIR_conf`              | Directory

### Extending

Additional configuration manipulations may be able to use the existing
MinionConfigurationExtension project. Current manipulations read the
value of a particular property (e.g. MASTER\_HOSTNAME) and apply to the
existing configuration file using a regular expression replace. Each new
manipulation will require changes to the following files:

- MinionConfiguration.cs: a new method (i.e. new custom action).
- MinionConfigurationExtensionCA.wxs: a &lt;CustomAction /&gt; entry to
  make the new method available.
- Product.wxs: a &lt;Custom /&gt; entry in the &lt;InstallSequence /&gt;
  to make the configuration change.
- Product.wxs: a &lt;Property /&gt; entry containing a default for the
  manipulated configuration setting.
- README.md: alter this file to explain the new configuration option and
  log the property name.

If the new custom action should be exposed to the UI, additional changes
are required:

- SettingsCustomizatonDlg.wxs: There is room to add 1-2 more properties to this dialog.
- WixUI_Minion.wxs: A &lt;ProgressText /&gt; entry providing a brief description of what the new action is doing.

If the new custom action requires its own dialog, these additional changes are required:

- The new dialog file.
- WixUI_Minion.wxs: &lt;Publish /&gt; entries hooking up the dialog buttons to other dialogs.
  Other dialogs will also have to be adjusted to maintain correct sequencing.
- MinionMSI.wixproj: The new dialog must be added as a &lt;Compile /&gt; item to be included in the build.

### MSBuild

General command line:

> msbuild msbuild.proj \[/t:target[,target2,..]] \[/p:property=value [ .. /p:... ] ]

A 'help' target is available which prints out all the targets, customizable
properties, and the current value of those properties:

> msbuild msbuild.proj /t:help

### Other Notes

[Wix-Setup-Samples](https://github.com/deepak-rathi/Wix-Setup-Samples)

[Which Python version uses which MS VC CRT version](https://wiki.python.org/moin/WindowsCompilers)

- Python 2.7 = VC CRT 9.0 = VS 2008  
- Python 3.6 = VC CRT 14.0 = VS 2017

Distutils contains bin/Lib/distutils/command/bdist_msi.py, which probably does not work.

[WiX_link]: http://wixtoolset.org
[MSBuild_link]: http://msdn.microsoft.com/en-us/library/0k6kkbsd(v=vs.120).aspx
[MSBuild2015_link]: https://www.microsoft.com/en-US/download/details.aspx?id=48159
[SALT_versions_link]:https://docs.saltstack.com/en/develop/topics/releases/version_numbers.html
[salt_versions_py_link]: https://github.com/saltstack/salt/blob/develop/salt/version.py
[WindowsInstaller4.5_link]:https://www.microsoft.com/en-us/download/details.aspx?id=8483
[MSDN_ProductVersion_link]:https://msdn.microsoft.com/en-us/library/windows/desktop/aa370859
