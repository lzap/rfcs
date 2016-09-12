- Feature Name: UEFI HTTP Boot
- Start Date: 2016-09-12
- RFC PR:
- Issue:

# Summary
[summary]: #summary

UEFI supports new kind of boot via HTTP instead of traditional (and slow) TFTP. Implementing this in Foreman is possible with relatively small amount of changes.

# Motivation
[motivation]: #motivation

Todays firmware is complicated and implementing a HTTP stack is possible, therefore we will see less TFTP in the future. Foreman already supports iPXE HTTP booting which is used for booting virtual machines with iPXE firmware built-in and UEFI HTTP Boot feature will be essentially the same. The feature can enable both UEFI HTTP and iPXE HTTP booting as the implementation requires only two main steps described below.

# Detailed design
[design]: #detailed-design

In order to support HTTP booting, we need to have an HTTP service deployed. Until now, users were deploying their own services becaue foreman-proxy only provides TFTP service and HTTP API. As part of this RFC, I propose to implement a simple HTTP/HTTPS service as a foreman-proxy plugin serving all files from TFTP directory. Only regular files are supported, listing of directories must be disabled.

Second change is to modify our `dhcpd.conf` file deployed by our puppet installer to hand over HTTPS/HTTP URLs when `HTTPClient` vendor class and "HTTP Boot x64/ia32" architecture flags are detected:

    option arch code 93 = unsigned integer 16; # RFC4578

    # support for UEFI HTTP Boot on Intel architectures
    class "httpclients" {
      match if substring (option vendor-class-identifier, 0, 9) = "HTTPClient";
      if option arch = 00:0F {
        filename "https://foreman.proxy.example.com:8443/grub2/bootia32.efi";
      } else if option arch = 00:10 {
        filename "https://foreman.proxy.example.com:8443/grub2/bootx64.efi";
      }
    }

    # the rest as implemented by http://projects.theforeman.org/issues/14920
    class "pxeclients" {
      match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
      next-server 10.0.0.1;
      if exists user-class and option user-class = "iPXE" {
        filename "https://foreman.example.com:443/unattended/iPXE";
      } else if option arch = 00:06 {
        filename "grub2/bootia32.efi";
      } else if option arch = 00:07 {
        filename "grub2/bootx64.efi";
      } else {
        filename "pxelinux.0";
      }
    }

In the Foreman UI user selects "Grub2" TFTP loader option.

# Drawbacks
[drawbacks]: #drawbacks

New service for handling HTTP/HTTPS file downloads is a new attack vector, we need to make sure we only read regular files (not symlinks) and only read from the root directory.

Exposing PXE configurations via HTTP/HTTPS adds new attack vector, but since the files are already readable by TFTP and we won't provide directory listings, the attacker chances to read existing loader configuration are the same.

# Unresolved questions
[unresolved]: #unresolved-questions

## Grub configuration loading

It wasn't yet tested if Grub2 loads its configuration from HTTP directory or uses DHCP filename option to get it. In the initial version of the design, I assume it will load from HTTP directly, otherwise additional if statements would be required similarly to iPXE user-class above (assuming that Foreman will render Grub2 unattended template just fine).

## Serving from TFTP or HTTP directory

Creating new directory for HTTP service seems to be clean idea, but if Grub loads its configuration directly from HTTP (which was assumed, see above unresolved item), the configuration file must be present.

# Links
[links]: #links

* http://www.uefi.org/specifications
* http://projects.theforeman.org/projects/foreman/wiki/Fetch_boot_files_via_http_instead_of_TFTP#C-Chainbooting-virtual-machines
* http://projects.theforeman.org/issues/14920
