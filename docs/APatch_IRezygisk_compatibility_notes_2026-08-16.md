# APatch / IRezygisk compatibility notes

## Scope

These notes record the verified boundary between APatch exclusion, Zygisk Assistant cleanup, and the Zygisk loader.
The i16/i17 results below are historical steps. The current device-verified baseline is APatch 19096/3215d2a,
KernelPatch d23, IRezygisk i21/558, CkbSmartN 0.101.20, and Assistant v2.1.5-apatch1/215.

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

## Historical transition and current verified release

The v2.1.5-apatch1/215 candidate makes the denylist namespace request explicit through
`FORCE_DENYLIST_UNMOUNT` and replaces the fortified `strncat` property copy with a bounded
`strnlen + memcpy + NUL` sequence. Physical fallback unloading remains the loader's responsibility and is implemented
by the paired IRezygisk i17/554 candidate.

The Assistant-specific change was finalized in local commit `5ec463e`: it explicitly requests denylist namespace
cleanup with `FORCE_DENYLIST_UNMOUNT` and replaces the fortified property-string append with bounded copy logic. It
does not contain Hunter, CkbSmartN, package-name, ABI, or device-model exceptions.

The final package is:

```text
Zygisk-Assistant-v2.1.5-apatch1-5ec463e-release.zip
size   = 394,588 bytes
sha256 = D6D6C16776DDC0E9BD596525514CFE22D7598CFFC7F4D9A47FCAC7199A59AB4A
```

It has been reboot-tested together with IRezygisk i21 and CkbSmartN 0.101.20 on APatch 19096/d23. Both ABI module
lists contained Assistant and CkbSmartN, the APatch V10 transactions all returned successfully, the CkbSmartN lspd
and serial Binder forwarding remained healthy, and no new target AVC or tombstone appeared. This verifies module
coexistence and the shared loading/startup contracts. A detector application's UI remains a separate end-to-end
acceptance test; Assistant must not fake SELinuxFS or compensate for another module's policy/mount behavior.

Future standard Zygisk modules do not copy these Assistant changes. They only need module-side changes if their own
business behavior exceeds the standard Zygisk module/companion lifecycle.
