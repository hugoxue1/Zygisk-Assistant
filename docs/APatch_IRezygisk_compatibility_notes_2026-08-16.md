# APatch / IRezygisk compatibility notes

## Scope

These notes record the verified boundary between APatch exclusion, Zygisk Assistant cleanup, and the Zygisk loader.
They apply to the APatch 19076 + IRezygisk i16/553 test performed on 2026-08-16.

## Exclusion is required but is not physical unloading

For every protected target app, enable `Umount modules/Exclude modifications` in APatch Manager. A verified Hunter
test showed that this creates clean per-process mount namespaces: neither `/data/adb/modules` nor the CkbSmartN ART
APEX dex2oat bind mounts remained in the target processes.

That result does not prove the Zygisk module ELF mappings are gone. In the same processes, both Assistant and
CkbSmartN still appeared as file-backed VMAs because IRezygisk's Android-linker fallback did not honor the modules'
`DLCLOSE_MODULE_LIBRARY` requests with a real `dlclose()`.

The responsibilities are therefore separate:

- APatch exclusion supplies a clean mount namespace;
- Assistant removes mount, property, environment, and linker-list evidence within its supported scope;
- the Zygisk loader owns the actual module handle and must honor opt-in unloading after the post callback returns.

Assistant must not add app-name or module-name exceptions to compensate for a loader lifecycle defect. Cleaning the
linker solist also cannot replace removing the kernel VMA backing visible through `/proc/<pid>/maps` or `map_files`.

## SELinux boundary

APatch exclusion is per-process mount behavior. It does not hide or undo globally loaded SELinux policy. A module such
as CkbSmartN/LSPlant may require `system_server -> system_server:process execmem`; policy scanners can report that rule
even when their app is fully excluded and Assistant cleanup succeeds.

Assistant must not falsify SELinuxFS or weaken system enforcement. Required global policy is a product compatibility
boundary and must be evaluated separately from Zygisk/mount hiding.

## Release verification

After a loader or Assistant change, perform a full reboot and verify all of the following on real target processes:

1. the app and its isolated/app-zygote children remain stable;
2. mountinfo contains neither module paths nor unexpected system/APEX bind mounts;
3. maps and map_files contain no module SO after an explicit unload request;
4. Assistant companion requests complete without namespace errors;
5. no new target tombstone or SELinux denial appears.

Passing only the manager UI, mount check, or linker-list check is insufficient.

## Current build candidate

The v2.1.5-apatch1/215 candidate makes the denylist namespace request explicit through
`FORCE_DENYLIST_UNMOUNT` and replaces the fortified `strncat` property copy with a bounded
`strnlen + memcpy + NUL` sequence. Physical fallback unloading remains the loader's responsibility and is implemented
by the paired IRezygisk i17/554 candidate.

All four ABIs build as valid shared ELF objects and export both Zygisk entry points. The 15-entry ZIP passes compressed
data integrity checks. Rebuild `Zygisk-Assistant-v2.1.5-apatch1-<commit>-release.zip` after the final local commit; the
exact filename, size, and SHA-256 are recorded in workspace handoff 102 to avoid a package-hash/commit-hash cycle. This
is a build candidate, not a device-verified stable release; the reboot verification above is still mandatory.
