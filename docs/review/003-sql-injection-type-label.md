# Finding 3: SQL injection via inverter-controlled strings in `type_label()`

## Metadata

| | |
|---|---|
| **Severity** | HIGH |
| **CWE** | [CWE-89](https://cwe.mitre.org/data/definitions/89.html) — Improper Neutralization of Special Elements used in an SQL Command; secondary [CWE-20](https://cwe.mitre.org/data/definitions/20.html) |
| **CVSS 3.1 Vector** | `AV:A/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N` — **8.1 (HIGH)** |
| **Affected version** | 3.10.0 (commit `b77f650`, branch `master`) |
| **Affected build** | Any build with database export enabled (MySQL/MariaDB or SQLite) |
| **Confidence** | 8–9/10 (independently reported by two reviewers) |
| **Verification** | Static analysis only; no PoC NameplateLocation record sent to a running build |

## Summary

`db_SQL_Base::type_label()` builds `INSERT IGNORE INTO Inverters` and `UPDATE
Inverters` statements by string-concatenating inverter-supplied fields
(`DeviceName`, `DeviceType`, `SWVersion`) through a helper named `s_quoted()`
that performs **no escaping** — it merely surrounds the value with single
quotes. A single quote byte in any of the three fields closes the literal
and the rest of the field is parsed as SQL.

This is particularly notable because the rest of the same classes (day data,
month data, spot data, events, battery) correctly uses parameterized prepared
statements — only these two methods regressed to concatenation.

**Per-field taint note.** Of the three fields fed to `s_quoted()` in
`type_label()`, only `DeviceName` is a direct copy of inverter-supplied
bytes (`SBFspot/SBFspot.cpp:2595`). `DeviceType` comes from
`tagdefs.getDesc(attr.front())` at `SBFspot.cpp:2609`, and `SWVersion` is
BCD-formatted from 4 bytes by `version_tostring()` at `SBFspot.cpp:2274`.
Neither of those paths lets an unmodified-tagfile attacker plant a `'` byte
on a stock build. The SQL injection therefore hinges on `DeviceName`; the
other two columns still want a parameterized binding for consistency and
for the case where an attacker can also plant a malicious tagfile.

## Affected code

### The unsafe quoting helper

`SBFspot/db_MySQL.h:93-94` (identical in `SBFspot/db_SQLite.h:87-88`):

```cpp
std::string s_quoted(std::string str) { return "'" + str + "'"; }
std::string s_quoted(char *str)       { return "'" + std::string(str) + "'"; }
```

No escaping whatsoever. A single `'` in `str` closes the string literal and
leaves everything after it as SQL.

### The vulnerable builders

`SBFspot/db_MySQL.cpp:126-160` — `type_label` (SQLite mirror at
`SBFspot/db_SQLite.cpp:140-174` uses `INSERT OR IGNORE` instead of
`INSERT IGNORE`, but is otherwise structurally identical). The `s_quoted`
call sites to reference directly: `db_MySQL.cpp:139-141`, `:150-152` and
`db_SQLite.cpp:153-155`, `:164-166`:

```cpp
sql << "INSERT IGNORE INTO Inverters VALUES(" <<
    inverters[inv]->Serial << ',' <<
    s_quoted(inverters[inv]->DeviceName) << ',' <<     // (*) attacker-controlled
    s_quoted(inverters[inv]->DeviceType) << ',' <<
    s_quoted(inverters[inv]->SWVersion) << ',' <<
    "0,0,0,0,0,0,'','',0)";

if ((rc = exec_query(sql.str())) != SQL_OK)
    print_error("exec_query() returned", sql.str());

sql.str("");

sql << "UPDATE Inverters SET" <<
    " Name=" << s_quoted(inverters[inv]->DeviceName) <<     // (*) again
    ",Type=" << s_quoted(inverters[inv]->DeviceType) <<
    ",SW_Version=" << s_quoted(inverters[inv]->SWVersion) <<
    " WHERE Serial=" << inverters[inv]->Serial;
```

### Where the tainted strings come from (shared with finding #2)

`SBFspot/SBFspot.cpp:2595` for `DeviceName`:

```cpp
device->DeviceName = std::string((char *)recptr + 8,
                                 strnlen((char *)recptr + 8, recordsize - 8));
```

Whatever bytes the peer sends become the field value; no character
validation. `DeviceType` and `SWVersion` come from the indirect paths
described in the Summary (`SBFspot.cpp:2609` and `:2600`/`:2274`
respectively) — still worth parameterizing, but not the primary attack
surface on a stock build.

### Execution path

#### MySQL

`SBFspot/db_MySQL.cpp:101-104`:

```cpp
int db_SQL_Base::exec_query(const std::string &qry)
{
    return mysql_real_query(m_dbHandle, qry.c_str(), qry.size());
}
```

`mysql_real_query` does **not** execute multiple semicolon-separated
statements by default — that requires either the `CLIENT_MULTI_STATEMENTS`
connect flag or a prior
`mysql_set_server_option(MYSQL_OPTION_MULTI_STATEMENTS_ON)`. `exec_query()`
uses neither, so an injected `;` followed by a second statement will
generally fail to parse.

However, single-statement injection is still dangerous: the attacker can:

- extend the `VALUES(...)` list to overwrite any other column (including columns they should never influence),
- in the `UPDATE`, inject additional `SET` clauses or widen the `WHERE` clause to rewrite every row in the `Inverters` table,
- use a MySQL error-based exfiltration payload to read other tables (e.g. `Config`, which typically contains the PVOutput API key) via error-message leakage,
- use subqueries like `' || (SELECT password FROM mysql.user LIMIT 1) || '` to splice other data into the stored `Name` column, exfiltrated on the next `SELECT` by the victim's PVO uploader or web UI.

Under MySQL the granted role is `DELETE,INSERT,SELECT,UPDATE ON SBFspot.*`
(per `CreateMySQLUser.sql`), so the attacker can read or modify any row in
the SBFspot schema.

#### SQLite

`SBFspot/db_SQLite.cpp:108-131`:

```cpp
int db_SQL_Base::exec_query(const std::string &qry)
{
    ...
    result = sqlite3_exec(m_dbHandle, qry.c_str(), NULL, NULL, NULL);
    ...
}
```

`sqlite3_exec` **does** execute multiple semicolon-separated statements. On
SQLite the injection is therefore a classic stacked-query exploit. Also
relevant: `ATTACH DATABASE '/some/writable/path' AS x;` allows the attacker
to cause SQLite to create a file at a path of their choosing as the SBFspot
user, which has been used as a stepping stone to code execution in other
contexts. The entire DB file is writable by the SBFspot process.

### Other sites audited

A targeted audit of all `s_quoted()` and string-concatenation sites in
`db_MySQL.cpp`, `db_MySQL_Export.cpp`, `db_SQLite.cpp`, `db_SQLite_Export.cpp`,
`db_update.cpp`, and `SQLselect.h`:

- **`type_label()` (MySQL + SQLite): VULNERABLE** — direct concatenation of three inverter-supplied fields. **This finding.**
- `device_status()` (`db_MySQL.cpp:181-182`, `db_SQLite.cpp:195-196`) — uses `s_quoted(status_text(...))`. `status_text()` is a 7-element whitelist switch (`"Open"`, `"Closed"`, `"OK"`, `"Warning"`, `"Fault"`, `"N/A"`, `"?"`). Less directly attacker-controlled today, but it depends on `tagdefs` loaded from the on-disk lookup table — parameterize for consistency.
- `exportSpotData()` (`db_MySQL_Export.cpp:265-266`, `db_SQLite_Export.cpp:219-220`) — same `status_text()` whitelist. Safe.
- `exportEventData()` paths — already use parameter binding (`mysql_stmt_bind_param`, `sqlite3_bind_text`). Safe.
- `set_config()`, `get_config()`, `batch_set_pvoflag()` — concatenate config-/HTTP-derived data. Not reachable from inverter input.

The codebase contains **zero** calls to `mysql_real_escape_string` or
`sqlite3_mprintf("%q"/"%Q", …)`. The only safe DB code path is parameter
binding, used for events/day data. Inverter metadata writes were never
converted.

## Preconditions

1. SBFspot built with `USE_MYSQL` or `USE_SQLITE`.
2. `SBFspot.cfg` has `SQL_Database` and credentials set so `type_label()` is invoked (this is the standard configuration when persistence is desired).
3. At least one inverter responds (the standard data-collection flow).
4. Attacker can influence inverter response bytes: compromised inverter, LAN attacker spoofing Speedwire frames (UDP 9522, unauthenticated), or attacker on the Bluetooth link.

## Exploit scenario

### SQLite (stacked query)

The attacker sets the inverter's NameplateLocation name to:

```
x','0','0',0,0,0,0,0,0,0,'','',0);ATTACH DATABASE '/home/pi/.ssh/authorized_keys' AS x;--
```

On the next SBFspot run, `type_label()` concatenates this into the `INSERT`,
`sqlite3_exec` runs both statements, and the ATTACH side-effect causes
SQLite to touch the attacker's chosen path as the SBFspot user.

### MySQL (single statement)

The attacker sets the device name to:

```
x',(select user_pass from wp_users limit 1),'',0,0,0,0,0,0,'','',0
```

The injected subquery runs in the context of the `INSERT`, copying data from
another table into the `Inverters.Type` column, which is then exposed to
anyone viewing inverter details (the uploader, the web UI, or subsequent
dumps).

Alternatively, an `UPDATE` payload like:

```
x',Type='pwn',SW_Version='1' WHERE 1=1 OR Serial='
```

produces:

```sql
UPDATE Inverters SET
  Name='x',Type='pwn',SW_Version='1' WHERE 1=1 OR Serial=''
  ,Type='...',SW_Version='...' WHERE Serial=12345
```

— every row in the `Inverters` table is rewritten, and the trailing portion
of the original statement becomes a no-op.

Either way, the attacker gains the ability to read from or, in many cases,
write to the database as the SBFspot SQL user — whose credentials are
typically high-privilege against the `SBFspotDB`/`sbfspot` database.
Outcome: corruption or exfiltration of historical inverter data and
configuration, with no audit trail.

## Recommended fix

**Primary recommendation: use parameterized statements**, consistently with
how the rest of this class already handles day/month/spot/event data.

Sketch (MySQL):

```diff
--- a/SBFspot/db_MySQL.cpp
+++ b/SBFspot/db_MySQL.cpp
@@
 int db_SQL_Base::type_label(InverterData *inverters[])
 {
-    std::stringstream sql;
     int rc = SQL_OK;

+    const char* sql_insert =
+        "INSERT IGNORE INTO Inverters VALUES(?,?,?,?,0,0,0,0,0,0,'','',0)";
+    const char* sql_update =
+        "UPDATE Inverters SET Name=?,Type=?,SW_Version=? WHERE Serial=?";
+
     for (uint32_t inv = 0; inverters[inv] != NULL && inv < MAX_INVERTERS; inv++)
     {
-        sql.str("");
-        sql << "INSERT IGNORE INTO Inverters VALUES(" <<
-            inverters[inv]->Serial << ',' <<
-            s_quoted(inverters[inv]->DeviceName) << ',' <<
-            s_quoted(inverters[inv]->DeviceType) << ',' <<
-            s_quoted(inverters[inv]->SWVersion) << ',' <<
-            "0,0,0,0,0,0,'','',0)";
-
-        if ((rc = exec_query(sql.str())) != SQL_OK)
-            print_error("exec_query() returned", sql.str());
-
-        sql.str("");
-        sql << "UPDATE Inverters SET" <<
-            " Name=" << s_quoted(inverters[inv]->DeviceName) <<
-            ",Type=" << s_quoted(inverters[inv]->DeviceType) <<
-            ",SW_Version=" << s_quoted(inverters[inv]->SWVersion) <<
-            " WHERE Serial=" << inverters[inv]->Serial;
-
-        if ((rc = exec_query(sql.str())) != SQL_OK)
-            print_error("exec_query() returned", sql.str());
+        // INSERT IGNORE
+        MYSQL_STMT* stmt = mysql_stmt_init(m_dbHandle);
+        mysql_stmt_prepare(stmt, sql_insert, strlen(sql_insert));
+        MYSQL_BIND b[4] = {};
+        unsigned long serial = inverters[inv]->Serial;
+        unsigned long n_len  = inverters[inv]->DeviceName.size();
+        unsigned long t_len  = inverters[inv]->DeviceType.size();
+        unsigned long v_len  = inverters[inv]->SWVersion.size();
+        b[0].buffer_type = MYSQL_TYPE_LONG;   b[0].buffer = &serial;
+        b[1].buffer_type = MYSQL_TYPE_STRING; b[1].buffer = (void*)inverters[inv]->DeviceName.c_str(); b[1].buffer_length = n_len; b[1].length = &n_len;
+        b[2].buffer_type = MYSQL_TYPE_STRING; b[2].buffer = (void*)inverters[inv]->DeviceType.c_str(); b[2].buffer_length = t_len; b[2].length = &t_len;
+        b[3].buffer_type = MYSQL_TYPE_STRING; b[3].buffer = (void*)inverters[inv]->SWVersion.c_str();  b[3].buffer_length = v_len; b[3].length = &v_len;
+        mysql_stmt_bind_param(stmt, b);
+        if (mysql_stmt_execute(stmt) != 0) { rc = SQL_ERROR; print_error("mysql_stmt_execute() returned", sql_insert); }
+        mysql_stmt_close(stmt);
+
+        // UPDATE (analogous: bind Name/Type/SW_Version then Serial; omitted for brevity)
     }
     return rc;
 }
```

Apply the parallel change in `db_SQLite.cpp` using `sqlite3_prepare_v2` /
`sqlite3_bind_text` (with `SQLITE_TRANSIENT`) / `sqlite3_step` /
`sqlite3_finalize`, mirroring the existing pattern in `exportEventData`.

**Minimum viable patch** (if a full refactor is out of scope short-term):
replace `s_quoted()` with a driver-provided escaper:

```cpp
// MySQL
std::string s_quoted(const std::string& str) const
{
    std::string buf(str.size() * 2 + 1, '\0');
    unsigned long n = mysql_real_escape_string(
        m_dbHandle, &buf[0], str.data(), (unsigned long)str.size());
    buf.resize(n);
    return "'" + buf + "'";
}

// SQLite
std::string s_quoted(const std::string& str) const
{
    char *q = sqlite3_mprintf("%Q", str.c_str());    // %Q adds quotes + escape
    std::string out = q ? q : "''";
    sqlite3_free(q);
    return out;
}
```

`s_quoted` is already a protected non-static member of `db_SQL_Base` (see
`db_MySQL.h:92-94` / `db_SQLite.h:86-88`), so `m_dbHandle` is in scope.
`mysql_real_escape_string` requires a live connection handle — ensure
`s_quoted` is not called before `open()` succeeds. `sqlite3_mprintf("%Q", ...)`
handles both escaping and quoting and does not require a handle.

**Defense in depth.** Restrict `DeviceName` (and all other inverter-sourced
strings) to a safe character set at ingest — see finding #2's sample patch
at `SBFspot/SBFspot.cpp:2595`. This protects against any callsite that is
missed today or added tomorrow, and shares its root cause with finding #2.

## Cross-reference

Findings #2 (MQTT command injection) and #3 (this) both stem from treating
inverter-sourced strings as trusted, and the primary tainted field in both
cases is `DeviceName`. A single sanitization pass at the ingest point
(`SBFspot/SBFspot.cpp:2595`) is the most economical one-line defense while
the per-site fixes are rolled out. Finding #1 independently demonstrates
that the same transport layer cannot be trusted for *length* either.

## Disclosure guidance

File as a private GitHub Security Advisory; do not open a public issue.
Pre-auth network-reachable; coordinate a release before public disclosure.

## References

- CWE-89: Improper Neutralization of Special Elements used in an SQL Command
- CWE-20: Improper Input Validation
- SQLite documentation: ["ATTACH DATABASE" — security implications](https://www.sqlite.org/lang_attach.html)
- MySQL documentation: [`mysql_real_escape_string()`](https://dev.mysql.com/doc/c-api/8.0/en/mysql-real-escape-string.html)
