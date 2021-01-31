

# UEFI shim bootloader secure boot life-cycle improvements
## Background
In the PC ecosystem, [UEFI Secure Boot](https://docs.microsoft.com/en-us/windows-hardware/design/device-experiences/oem-secure-boot) is typically configured to trust 2 authorities for signing UEFI boot code, the Microsoft UEFI Certificate Authority (CA) and Windows CA. When malicious or security compromised code is detected, 2 revocation mechanisms are provided by compatible UEFI implementations, signing certificate or image hash. The UEFI Specification does not provides any well tested additional revocation mechanisms.

Signing certificate revocation is not practical for the Windows and Microsoft UEFI CAs because it would revoke too many UEFI applications and drivers, especially for Option ROMs. This is true even for the UEFI CA leaf certificates as they generally sign 1 entire year of UEFI images. For this reason UEFI revocations have, until recently, been performed via image hash.

The UEFI shim bootloader provides a level of digital signature indirection, enabling more authorities to participate in UEFI Secure Boot. Shims' certificates typically sign targeted UEFI applications, enabling certificate-based revocation where it makes sense.
As part of the recent "BootHole" security incident [CVE-2020-10713](https://nvd.nist.gov/vuln/detail/CVE-2020-10713), 3 certificates and 150 image hashes were added to the UEFI Secure Boot revocation database `dbx` on the popular x64 architecture. This single revocation event consumes 10kB of the 32kB, or roughly one third, of revocation storage typically available on UEFI platforms. Due to the way that UEFI merges revocation lists, this plus prior revocation events can result in a `dbx` that is almost 15kB in size, approaching 50% capacity.

The large size of the BootHole revocation event is due to the inefficiency of revocation by image hash when there is a security vulnerability in a popular component signed by many authorities, sometimes with many versions.

Coordinating the BootHole revocation has required numerous person months of planning, implementation, and testing multiplied by the number of authorities, deployments, & devices. It is not yet complete, and we anticipate many months of upgrades and testing with a long tail that may last years.

Additionally, when bugs or features require updates to UEFI shim, the number of images signed are multiplied by the number of authorities.

## Summary
Given the tremendous cost and disruption of a revocation event like BootHole, and increased activity by security researchers in the secure boot space, we should take action to greatly improve this process. Updating revocation capabilities in the UEFI specification and system firmware implementations will take years to deploy into the ecosystem. As such, the focus of this document is on improvements that can be made to the UEFI shim, which are compatible with existing UEFI implementations. Shim can move faster than the UEFI system BIOS ecosystem while providing large impact to the in-market Secure Boot ecosystem.

The background section identified 2 opportunities for improvement:

1. Improving the efficiency of revocation when a number of versions have a vulnerability
  
   * For example, a vulnerability spans some number of versions, it might be more efficient to be able to revoke by version, and simply modify the revocation entry to modify the version each time a vulnerability is detected.
2. Improving the efficiency of revocation when there are many shim variations
  
   * For example, a new shim is released to address bugs or adding features. In the current model, the number of images signed are multiplied by the number of authorities as they sign shims to gain the fixes and features.

Microsoft has brainstormed with partners possible solutions for evaluation and feedback:

1. To improve revocation when there are many versions of vulnerable boot images, shim, grub, or otherwise, investigate methods of revoking by image metadata that includes generation numbers. Once targeting data is established (e.g. Company foo, product bar, boot component zed), each revocation event ideally edits an existing entry, increasing the trusted minimum security generation.
2. To improve revocation when there is a shim vulnerability, and there are many shim images, standardize on a single image shared by authorities. Each release of bug fixes and features result in 1 shim being signed, compressing the number by dozens. This has the stellar additional benefit of reducing the number of shim reviews, which should result in much rejoicing. The certificates used by a vendor to sign individual boot components would be picked up from additional PE files that are signed either by a shim specific key controlled by Microsoft, or controlled by a vendor, but used only to sign additional key files. This key built into shim is functionally similar so a CA certificate.
The certificates built into shim can be revoked by placing the image hash into dbx, similar to the many shim solution we have today.

## Proposals
This document focuses on the shim bootloader, not the UEFI specification or updates to UEFI firmware.

### Generation Number Based Revocation
Microsoft may refer to this as a form of Secure Boot Advanced Targeting (SBAT), perhaps to be named EFI_CERT_SBAT. This introduces a mechanism to require a
specific level of resistance to secure boot bypasses.

#### Generation-Based Revocation Overview
Metadata that includes the vendor, product family, product, component,
version and generation are added to artifacts. This metadata is
protected by the digital signature. New image authorization data
structures, akin to the EFI_CERT_foo EFI_SIGNATURE_DATA structure (see
Signature Database in the UEFI specification), describe how this
metadata can be incorporated into allow or deny lists. In a simple
implementation, 1 SBAT entry with security generations could be used
for each revocable boot module, replacing many image hashes with 1
entry with security generations. To minimize the size of
EFI_CERT_SBAT, the signature owner field might be omitted, and
recommend that either metadata use shortened names, or perhaps the
EFI_CERT_SBAT contains a hash of the non-generation metadata instead
of the metadata itself.

Ideally, servicing of the image authorization databases would be updated to support replacement of individual EFI_SIGNATURE_DATA items. However, if we assume that new UEFI variable(s) are used, to be serviced by 1 entity per variable (no sharing), then the existing, in-market SetVariable(), without the APPEND attribute, could be used. Microsoft currently issues dbx updates exclusively with the APPEND attribute under the assumption that multiple entities might be servicing dbx. When a new revocation event takes place, rather than increasing the size of variables with image hashes, existing variables can simply be updated with new security generations, consuming no additional space. This constrains the number of entries to the number of unique boot components revoked, independent of generations revoked. The solution may support several major/minor versions, limiting revocation to build/security generations, perhaps via wildcards.

While previously the APPEND attribute guaranteed that it would not be possible to downgrade the set of revocations on a system using a previously signed variable update, this guarantee can also be accomplished by setting the EFI_VARIABLE_TIME_BASED_AUTHENTICATED_WRITE_ACCESS attribute. This will verify that the
timestamp value of the signed data is later than the current timestamp value associated with the data currently stored in that variable.

#### Generation-Based Revocation Scenarios

 Products (**not** vendors, a vendor can have multiple products or even
pass a product from one vendor to another over time) are assigned a
name. Product names can specify a specific version or refer to the
entire product family. For example mydistro and mydistro-12.

 Components that are used as a link in the UEFI secure boot chain of
trust are also assigned names. Examples of components are shim, GRUB,
kernel, hypervisors, etc.

 We could conceivably support sub-components, but it's hard to
conceive of a scenario that would trigger a UEFI variable update that
wouldn't justify a hypervisor or kernel re-release to enforce that
sub-component level from there. Something like a "level 1.5
hypervisor" that can exist between different kernel generations can be
considered its own component.

 Each component is assigned a minimum global generation number. Vendors
signing component binary artifacts with a specific global generation
number are required to include fixes for any public or pre-disclosed
issue required for that generation. Additionally, in the event that a
bypass only manifests in a specific product's component, vendors may
ask for a product-specific generation number to be published for one
of their product's components. This avoids triggering an industry wide
re-publishing of otherwise safe components.

 A product-specific minimum generation number only applies to the
instance of that component that is signed with that product
name. Another product's instance of the same component may be
installed on the same system and would not be subject to the other
product's product-specific minimum generation number. However, both of
those components will need to meet the global minimum generation
number for that component. A very likely scenario would be that a
product is shipped with an incomplete fix required for a specific
minimum generation number, but is labeled with that number. Rather
than having the entire industry that uses that component re-release,
just that product's minimum generation number would be incremented and
that product's component re-released along with a UEFI variable update
specifying that requirement.

 The global and product-specific generation number name spaces are not
tied to each other. The global number is managed externally, and the
vast majority of products will never publish a minimum
product-specific generation number for any of their components. In
that case, these components shall be signed with a product-specific
generation number of 0.

 A minimum feature set, for example enforced kernel lock down, may be
required as well to sign and label a component with a specific
generation number. As time goes on, it is likely that the minimum
feature set required for the currently valid generation number will
expand. (For example, hypervisors supporting secure boot guests may
at some point require memory encryption or similar protection
mechanism.)

 The footprint of the UEFI variable payload will expand as
product-specific generation numbers ahead of the global number are
added. However, it will shrink again as the global number for that
component is incremented again.  The expectation is that a product or
vendor specific generation number is a rare event, and that the
generation number for the upstream code base will suffice in most
cases.

A product-specific generation number is needed if a CVE is fixed in
code that **only** exists in a specific product's branch. This would
either be something like product-specific patches, or a mis-merge that
only occurred in that product. Setting a product-specific generation
number for such an event eliminates the need to increment the global
number (and then subsequent re-release by other vendors to match that
incremented global number).

However, once the global number is bumped for the next upstream CVE
fix there will be no further need to carry that product-specific
generation number; satisfying the check of the global number will also
exclude any of the older product-specific binaries.

For example: There is a global CVE disclosure and all vendors
coordinate to release fixed components on the disclosure date. This
releases bumps the global generation number for GRUB to 4.

 SBAT revocation data would then require a grub with a global
 generation number of 4.

However, Vendor C mis-merges the patches into one of their products and
does not become aware of the fact that this mis-merge created an
additional vulnerability until after they have published a signed
binary in that vulnerable state.

 Vendor C's grub binary can now be used to compromise anyone's system.

To remedy this, Vendor C will release a fixed binary with the same
global generation number and the product-specific generation number
set to 1.

 SBAT revocation data would then require a grub with a global
 generation number of 4, as well as a product-specific generation
 number of 1 for C's product that included the vulnerable binary.

If and when there is another upstream fix for a CVE that would bump
the global number, this product-specific number can be dropped from
the UEFI revocation variable.

If this same Vendor C has a similar event after the global number is
incremented, they would again set their product or version specific
number to 1. If they have a second event on with the same component,
they would set their product- or version-specific number to 2.

In such an event, a vendor would set the product- or product
version-specific generation number based on whether the mis-merge
occurred in all of their branches or in just a subset of them. The
goal is generally to limit end customer impact with as few re-releases
as possible, while not creating an unnecessarily large UEFI revocation
variable payload.

The variable payload will be stored publicly in the shim source base
and identify the global generation associated with a product- or
version-specific one. The payload is also built into shim to
additionally limit exposure.

#### Retiring Signed Releases

Products that have reached the end of their support life (EoSL) will,
by definition, no longer receive patches. They are also generally not
going to be examined for CVEs. Allowing such unsupported products to
continue to participate in UEFI Secure Boot is, at the very least,
questionable. If an EoSL product is made up of commonly used
components, such as the GRUB and the Linux kernel, it is reasonable to
assume that the global generation numbers will eventually move forward
and exclude those products from booting on a UEFI Secure Boot enabled
system. However, a product made up of GRUB and a closed source kernel
is just as conceivable. In that case the kernel version may never move
forward once the product reaches its end of support. Therefore it is
recommended that the product-specific generation number be incremented
past the latest one shown in any binary for that product, effectively
disabling that product on UEFI Secure Boot enabled systems.

An equivalent of this case would be a beta-release that may contain
eventually abandoned, experimental, kernel code. Such releases should
also have their product-specific generation numbers incremented past
the latest one shown in any, released or unreleased, binary signed
with a production key.

Until a release is retired in this manner, vendors will be responsible
for keeping up with fixes for CVEs and ensuring that any known signed
binaries containing known CVEs are denied from booting on UEFI Secure
Boot enabled systems via the most up to date UEFI metadata.

#### Key Revocations
Since Vendor Product keys are brought into Shim as signed binaries,
generation numbering can and should be used to revoke them in case of
a private key compromise.

#### Kernel support for SBAT

The initial SBAT implementation will add SBAT metadata to Shim and
GRUB and enforce SBAT on all components labeled with it. Until a given
component (like, say, the Linux kernel) gains SBAT metadata it cannot
be revoked via SBAT, but only by revoking the keys signing that
component. These keys should live in separate, product-specific,
signed PE files that contain **only** the certificate and SBAT
metadata for the key files. This means these key files can then be
revoked via SBAT in order to invalidate and replace a specific
key. While certificates built into Shim can be revoked via SBAT and
Shim introspection, this practice would still result in a
proliferation of Shim binaries that would need to be revoked via dbx
in the event of an early Shim code bug. Therefore, SBAT must be used
in conjunction with separate Vendor Product Key binaries.

At the time of this writing, revoking a Linux kernel with a lockdown
compromise is not spelled out as a requirement for shim signing. In
fact, with limited dbx space and the size of the attack surface for
lockdown it would be impractical do so without SBAT. By using SBAT it
should be possible to raise the bar here, and treat lockdown bugs that
would allow a kexec of a tampered kernel as revocations.


#### Kernels execing other kernels (aka kexec, fast reboot)

It is expected that kexec and other similar implementations of kernels
spawning other kernels will eventually consume and honor SBAT
metadata. Until they do, the same Vendor Product Key binary based
revocation will need to be used for them.


#### Version-Based Revocation Metadata

This depends on adding a .sbat section containing the SBAT metadata
structure to PE images.

Each component carries a metadata payload within the signed binary.
This metadata contains the component name, the name of the product
with which that component is released, and the version of that
product, along with a global generation number that is in sync with
the patch level of that build. If applicable, it may also contain
non-zero product- and product version-specific generation numbers.

The .sbat section is made up of comma-separated values UTF-8 encoded
strings:

sbat_data_version,component_name,component_generation,product_name,product_generation,product_version,version_generation

For example:

```
1,GRUB2,1,Oracle Linux,0,7.9,0
```

Components that do not have special code to construct the final PE
files can simply add this section using objcopy(1):

```
objcopy --add-section .sbat=sbat.csv foo.efi

```

This will then be used to populate the following data structure:

```
struct sbat_metadata {
       char *sbat_data_version          // version of this structure, 1 at initial release
       char *component_name; 	      	  // for example "GRUB2"
       char *component_generation;      // 1 at initial release then incrementing
       char *product_name;   	      	  // for example: "Oracle Linux"
       char *product_generation;     	  // generally 0 unless	needed
       char *product_version;	      	  // for example: "7.9"
       char *version_generation;     	  // generally 0 unless needed
};

```

#### UEFI SBAT Variable content
The SBAT UEFI variable contains a descriptive form of all components
used by all UEFI signed Operating Systems, along with a minimum
generation number for each one. It may also contain a product-specific
generation number, which in turn may also specify version-specific
generation numbers. It is expected that specific generation numbers
will be exceptions that will be obsoleted if and when the global
number for a component is incremented.

```
// XXX evolving
COMPONENTS
  InternalName                   "SHIM"
    MinComponentGeneration       0
  InternalName                   "GRUB2"
    MinComponentGeneration       3
  InternalName                   "Linux Kernel"
    MinComponentGeneration       73
    PRODUCTS
      ProductName                "Some Linux"
        MinProductGeneration     1
      ProductName                "Other Linux"
        MinProductGeneration     0
        VERSIONS
          ProductVersion         "32"
            MinVersionGeneration 2
          ProductVersion         "33"
            MinVersionGeneration 1
        /VERSIONS
    /PRODUCTS
  InternalName                   "Some Vendor Cert"
    MinComponentGeneration       22
  InternalName                   "Other Vendor Cert"
    MinComponentGeneration       4
/COMPONENTS

```

An EDK2 example demonstrating a method of image authorization, based
upon version metadata, should be available soon. We expect the
structures and methods to evolve as feedback is incorporated.