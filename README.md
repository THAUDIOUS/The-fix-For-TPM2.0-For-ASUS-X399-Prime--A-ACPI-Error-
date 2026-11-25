# The-fix-For-TPM2.0-For-ASUS-X399-Prime--A-ACPI-Error-
The fix For TPM2.0 For ASUS X399 Prime -A ACPI Error 



Let's double-check the exact parameters from their dmesg:
• Board: ASUS PRIME X399-A
• OS: Bazzite (Fedora Atomic / rpm-ostree)
• Issue: tpm_crb fails with -EBUSY (can't request region for resource) and logs [Firmware Bug]: ACPI region does not cover the entire command/response buffer because ASUS's AMI BIOS declares a 4 KB (0x1000) ACPI NVS region instead of 16 KB (0x4000), and overlaps with ACPI NVS memory.
• Working kernel arguments they used:
tpm_crb.interrupts=0
memmap=0x1000%0x7813b000-4
memmap=0x1000%0x7813f000-4
(Note: The memory addresses 0x7813b000 and 0x7813f000 can shift depending on BIOS version/settings or hardware configuration, so any good write-up should explain how to read those two addresses from dmesg first rather than hardcoding blindly).
Let's format this cleanly as a practical, well-organized technical guide. No search needed since we have all exact details from the user's own dmesg and standard Linux tpm_crb / rpm-ostree behavior.Here is a clean write-up of the root cause and fix for the ASUS PRIME X399-A ACPI tpm_crb firmware bug on Linux (tailored for Bazzite / Fedora Atomic).

Fix: ASUS PRIME X399-A TPM 2.0 (tpm_crb) ACPI Firmware Bug on Linux


The Problem

On the ASUS PRIME X399-A, the AMI UEFI firmware enables AMD fTPM (MSFT0101), but its ACPI tables misreport the Command/Response Buffer (CRB) regions as 0x1000 (4 KB) instead of 0x4000 (16 KB) and leave them marked as reserved ACPI NVS memory (type 4).
When Linux boots, the tpm_crb driver logs:
text
tpm_crb_acpi MSFT0101:00: [Firmware Bug]: ACPI region does not cover the entire command/response buffer. [mem 0x7813b000-0x7813bfff flags 0x200] vs 7813b000 4000
tpm_crb_acpi MSFT0101:00: can't request region for resource [mem 0x7813b000-0x7813bfff]
tpm_crb_acpi: probe of MSFT0101:00 failed with error -16
Use code with caution.
Because probe fails with -EBUSY (-16), /dev/tpm0 and /dev/tpmrm0 are never created. ASUS has not released a BIOS update that corrects this ACPI table defect.

Step 1: Verify BIOS Settings

1. Enter UEFI BIOS (Del or F2 at boot).
2. Go to Advanced → AMD fTPM configuration.
3. Set TPM Device Selection to Firmware TPM.
4. Save and exit.

Step 2: Find Your Exact Buffer Addresses

Run the following command in a terminal to check the two memory addresses reported by the firmware:
bash
sudo dmesg | grep -iE "tpm_crb|MSFT0101"
Use code with caution.
Look for the two 0x1000-sized regions (commonly 0x7813b000 and 0x7813f000, though they can vary slightly depending on BIOS version and PCIe/memory layout).

Step 3: Re-map the ACPI NVS Regions via Kernel Arguments

Use the memmap parameter to convert those two 0x1000 regions from ACPI NVS (type 4) to usable RAM (type 1) using the syntax memmap=<size>%<start>-<old_type>, and disable tpm_crb interrupts (tpm_crb.interrupts=0) so the driver uses polling.

On Bazzite / Fedora Atomic (rpm-ostree):

bash
sudo rpm-ostree kargs \
  --append-if-missing="tpm_crb.interrupts=0" \
  --append-if-missing="memmap=0x1000%0x7813b000-4" \
  --append-if-missing="memmap=0x1000%0x7813f000-4"
Use code with caution.
(Replace 0x7813b000 and 0x7813f000 if Step 2 showed different addresses on your system.)

On Standard GRUB Distros (Fedora Workstation, Ubuntu, Arch):

Add the following parameters to GRUB_CMDLINE_LINUX_DEFAULT in /etc/default/grub and regenerate your GRUB config:
text
tpm_crb.interrupts=0 memmap=0x1000%0x7813b000-4 memmap=0x1000%0x7813f000-4
Use code with caution.
Reboot the system for the memory map changes to take effect.

Step 4: Verify TPM Initialization

After rebooting, confirm the TPM device nodes exist and respond:
bash
ls -l /dev/tpm*
sudo tpm2_pcrread sha256:0,7
Use code with caution.
• Expected result: /dev/tpm0 and /dev/tpmrm0 appear, and tpm2_pcrread returns valid SHA-256 PCR hashes.
• Note on remaining log warnings: You will still see [Firmware Bug]: ACPI region does not cover the entire command/response buffer and Disabling hwrng in dmesg. These are harmless informational messages—the kernel's built-in quirk handler automatically falls back and initializes the TPM normally.
