---
title: Dealing with troublesome OVAs in Virtualbox
date: 2016-06-08T21:03:50Z
tags:
  - howto
  - virtualbox
category: howto
---

Recently I've come across a lot of issues when trying to import _OVA_ files into Virtualbox. Luckily it's not hard to remedy. This post will serve both as a reference to me and hopefully help others

The error I often get is

```sourceCode
Failed to import appliance /path/to/file.ova.

Error reading &quot;/path/to/file.ova&quot;: Host resource of type &quot;Other Storage Device (20)&quot; is supported with SATA AHCI controllers only, line 47.

Result Code: VBOX_E_FILE_ERROR (0x80BB0004)
Component: ApplianceWrap
Interface: IAppliance {8398f026-4add-4474-5bc3-2f9f2140b23e}
```

To remedy this simply extract the _OVA_ file and edit the _OVF_ like so

```sourceCode
tar xvf file.ova
```

You will end up with three files: **.ovf**, **.vmdk**, and **.mf**

Now open the **.ovf** in a text editor and replace all occurances of `ElementName` with `Caption` and then replace the entry `vmware.sata.ahci` with `AHCI`. Save the file and then delete the **.mf** file (it contains a checksum for the _VMDK_ file, and it'll be wrong and we won't need it anymore). Now import the **.ovf** into Virtualbox.

Happy virtualising.
