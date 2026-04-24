# Finding 1: Unvalidated Bluetooth packet length causes global buffer overflow in `getPacket()`

## Metadata

| | |
|---|---|
| **Severity** | HIGH |
| **CWE** | [CWE-120](https://cwe.mitre.org/data/definitions/120.html) — Buffer Copy without Checking Size of Input; secondary [CWE-130](https://cwe.mitre.org/data/definitions/130.html), [CWE-787](https://cwe.mitre.org/data/definitions/787.html) |
| **CVSS 3.1 Vector** | `AV:A/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H` — **8.8 (HIGH)** |
| **Affected version** | 3.10.0 (commit `b77f650`, branch `master`) |
| **Affected build** | Bluetooth transport only — see "Scope" |
| **Confidence** | 9–10/10 (independently reported by two reviewers) |
| **Verification** | Static analysis only; no PoC frame constructed against a running build |

## Summary

`getPacket()` in the Bluetooth read path trusts the 16-bit `pkLength` field
from the on-wire SMAdata2 header and passes it unchecked to a second `recv()`
call into the 2048-byte global `CommBuf`. Because `pkLength` is a `uint16_t`
and no upper bound is enforced, a peer can cause up to ~63 KB of
attacker-chosen bytes to be written past the end of `CommBuf`, corrupting
adjacent BSS globals (including `pcktBuf`, the Bluetooth `sock`, and other
state). This is a classic linear overflow with a fully attacker-controlled
length and contents, reachable from any device able to send data on the
paired RFCOMM link.

## Affected code

### The read itself

`SBFspot/SBFspot.cpp:95-164` — `getPacket()`. The vulnerable sequence:

```cpp
E_SBFSPOT getPacket(uint8_t senderaddr[6], int wait4Command)
{
    ...
    int bib = bthRead(CommBuf, sizeof(pkHeader));        // (1) 18-byte header
    if (bib <= 0) { ... return E_NODATA; }

    //More data after header?
    if (btohs(pkHdr->pkLength) > sizeof(pkHeader))
    {
        bib += bthRead(CommBuf + sizeof(pkHeader),
                       btohs(pkHdr->pkLength) - sizeof(pkHeader));   // (2) OVERFLOW (line 114)
        ...
        if (hasL2pckt == 1)
        {
            for (int i = sizeof(pkHeader); i < btohs(pkHdr->pkLength); i++)
            {
                pcktBuf[index] = CommBuf[bufptr++];      // line 137 — bufptr unbounded vs CommBuf
                ...
            }
        }
        else
        {
            memcpy(pcktBuf, CommBuf, bib);               // line 162 — bib unbounded vs pcktBuf
        }
    }
}
```

1. At (1), 18 bytes are read into `CommBuf`. `pkHdr` aliases `CommBuf`, so
   `pkHdr->pkLength` is now attacker-controlled.
2. At (2), the code reads `pkLength - 18` more bytes starting at
   `CommBuf + 18`. An attacker can set `pkLength` to `0xFFFF` (65535),
   causing a read of up to 65517 bytes into a buffer whose total capacity is
   2048.

### The buffer

- `CommBuf`: `uint8_t CommBuf[COMMBUFSIZE]` declared at `SBFspot/Bluetooth.cpp:38`.
- `COMMBUFSIZE = 2048` in `SBFspot/misc.h:44`.
- `pcktBuf`: also 2048 bytes (`maxpcktBufsize`).

`CommBuf` is a global in BSS, adjacent to other globals. Exact layout is
determined by the linker, but `bytes_in_buffer`, `bufptr`, `sock`, `addr_in`,
and `addr_out` (declared at `SBFspot/Bluetooth.cpp:40-43`) and `pcktBuf`,
`pcktID`, `FCSChecksum`, `packetposition` (declared at `SBFspot/SBFNet.cpp:52-57`)
are all reachable.

### The read helper — no additional bounds check

`SBFspot/Bluetooth.cpp:428-469` — `bthRead`:

```cpp
int bthRead(uint8_t *buf, unsigned int bufsize)
{
    ...
    if (FD_ISSET(sock, &readfds))
        bytes_read = recv(sock, (char *)buf, bufsize, 0);   // bufsize == attacker-controlled
    ...
}
```

`bthRead` trusts its `bufsize` argument and passes it unmodified to `recv()`.

### Secondary corruption in the de-escape loop

Even if the initial overflow is mitigated, the subsequent copy loop at
`SBFspot/SBFspot.cpp:135-157` iterates up to `pkHdr->pkLength` into
`pcktBuf` with only a post-hoc check against `maxpcktBufsize`, so with a
large `pkLength` it can still corrupt `pcktBuf` (2048 bytes) from attacker
input. The `memcpy` at line 162 (the non-L2 path) similarly uses `bib`
unconstrained.

## Scope

- **Bluetooth path: VULNERABLE.** Transport is RFCOMM (`SOCK_STREAM` over BT), so `recv()` returns as much data as the peer supplies up to the requested size.
- **Ethernet / Speedwire path: NOT directly affected, but warrants audit.** `ethGetPacket()` (`SBFspot/SBFspot.cpp:244-298`) reads via `ethRead(CommBuf, sizeof(CommBuf))` (`SBFspot/Ethernet.cpp:113`), which is bounded by the buffer size and naturally capped by UDP datagram semantics — no second `recv()` driven by the header length. However, the subsequent `memcpy(pcktBuf+1, CommBuf + sizeof(ethPacketHeaderL1), bib - sizeof(ethPacketHeaderL1))` uses `bib - sizeof(ethPacketHeaderL1)` without checking against `maxpcktBufsize`. If `pcktBuf < CommBuf` this is an overflow; if equal, it is safe-but-fragile. Add a bounds check regardless.

## Preconditions

- SBFspot built with `BT_SBFSPOT` (Bluetooth) support enabled.
- Operator runs SBFspot connected to an inverter over Bluetooth.
- Attacker can either (a) bring a Bluetooth radio within range and impersonate the paired inverter, (b) compromise the inverter firmware, or (c) MITM the RFCOMM stream.
- Reachable **pre-authentication** — `getPacket()` runs during the initial handshake, before the inverter password challenge completes. The `isValidSender` check only compares the source address; it does *not* validate the length, and address spoofing on Bluetooth is feasible.

## Exploit scenario

SBFspot is typically deployed as a long-running background process on a
Raspberry Pi or home server, paired once with an SMA inverter over Bluetooth
(legacy SMA models), and run on a schedule — commonly as `root` on SBC/Pi/Docker
deployments.

**Attacker model.** Any of:

1. A malicious or compromised SMA inverter — the paired-inverter model assumes integrity from the peer, but SMA Bluetooth pairing only authenticates *who* the peer is, not that the peer is benign.
2. An attacker who has previously compromised the paired inverter's firmware.
3. An attacker within BT range who can MITM or impersonate the inverter (SMA Bluetooth uses legacy pairing).

**Exploit.**

1. Attacker, in Bluetooth range of the SBFspot host, masquerades as the paired SMA inverter (RFCOMM SDP advertisement + matching device address).
2. SBFspot opens RFCOMM, calls `getPacket()`. The first `bthRead` returns the 18-byte header.
3. Attacker's header has `pkLength = 0xFFFF`. Code at line 114 calls `bthRead(CommBuf+18, 65517)` → `recv(sock, CommBuf+18, 65517, 0)`.
4. Attacker streams ~65 KiB of payload. `recv()` writes them past `CommBuf`, corrupting adjacent globals.
5. Subsequent `memcpy(pcktBuf, CommBuf, bib)` (line 162 / 187) compounds the corruption with `bib > 2048`.

Because the attacker chooses every byte of the overflow, they control:

- function-pointer-like state (e.g. the `packetposition`, flags, and any C++ vtable pointers on globals placed after `CommBuf`),
- in-memory credentials the process holds (MySQL password and PVOutput API key loaded from the config file at startup).

Depending on the linker-chosen layout, this yields either arbitrary memory
disclosure (next BSS globals are read later and sent back to the inverter or
logged) or arbitrary code execution as the SBFspot user. At minimum, DoS and
corruption of the socket descriptor. Given that the binary is typically built
without stack canaries or PIE on small-board distributions, this is an **RCE
primitive**.

No user interaction is required after the inverter is paired. The process
typically runs on a quiet network segment, so detection is unlikely.

## Recommended fix

Validate the length before the second read, clamp the destination on every
`memcpy`, and bound the for-loop. Suggested patch:

```diff
--- a/SBFspot/SBFspot.cpp
+++ b/SBFspot/SBFspot.cpp
@@ -109,11 +109,18 @@ E_SBFSPOT getPacket(uint8_t senderaddr[6], int wait4Command)
             return E_NODATA;
         }

         //More data after header?
         if (btohs(pkHdr->pkLength) > sizeof(pkHeader))
         {
-            bib += bthRead(CommBuf + sizeof(pkHeader), btohs(pkHdr->pkLength) - sizeof(pkHeader));
+            // Reject frames whose declared length exceeds our buffer (CWE-120).
+            const unsigned int pkLen = btohs(pkHdr->pkLength);
+            if (pkLen > sizeof(CommBuf) || pkLen < sizeof(pkHeader))
+            {
+                if (DEBUG_NORMAL)
+                    printf("Invalid packet length %u (max %zu)\n",
+                           pkLen, sizeof(CommBuf));
+                return E_BUFOVRFLW;
+            }
+            bib += bthRead(CommBuf + sizeof(pkHeader),
+                           pkLen - sizeof(pkHeader));

             //Check if data is coming from the right inverter
             if (isValidSender(senderaddr, pkHdr->SourceAddr))
@@ -132,7 +139,9 @@ E_SBFSPOT getPacket(uint8_t senderaddr[6], int wait4Command)

                     if (DEBUG_HIGHEST) printf("PacketLength=%d\n", btohs(pkHdr->pkLength));

-                    for (int i = sizeof(pkHeader); i < btohs(pkHdr->pkLength); i++)
+                    for (int i = sizeof(pkHeader);
+                         i < btohs(pkHdr->pkLength) && bufptr < bib && bufptr < (int)sizeof(CommBuf);
+                         i++)
                     {
                         pcktBuf[index] = CommBuf[bufptr++];
                         //Keep 1st byte raw unescaped 0x7E
@@ -158,7 +167,12 @@ E_SBFSPOT getPacket(uint8_t senderaddr[6], int wait4Command)
                 }
                 else
                 {
-                    memcpy(pcktBuf, CommBuf, bib);
+                    if (bib > (int)maxpcktBufsize)
+                    {
+                        printf("Warning: bib=%d > pcktBuf size\n", bib);
+                        return E_BUFOVRFLW;
+                    }
+                    memcpy(pcktBuf, CommBuf, bib);
                     packetposition = bib;
                 }
             } // isValidSender()
```

(Apply an equivalent guard to the second `memcpy(pcktBuf, CommBuf, bib)` at
line 187.)

**Defense-in-depth** — add a clamp inside `bthRead` itself so any future
caller is also protected:

```diff
--- a/SBFspot/Bluetooth.cpp
+++ b/SBFspot/Bluetooth.cpp
@@
 int bthRead(unsigned char *buf, unsigned int bufsize)
 {
+    if (bufsize > COMMBUFSIZE) bufsize = COMMBUFSIZE;
     int bytes_read = recv(sock, (char *)buf, bufsize, 0);
```

**Longer term:**

- Consider removing the global `CommBuf`/`pcktBuf` and allocating per-call buffers with checked sizes.
- Add a fuzz harness around `getPacket()` and `ethGetPacket()` that feeds random `pkHeader` + payload combinations — this class of bug is well-suited to coverage-guided fuzzing (AFL++/libFuzzer).

## Disclosure guidance

File as a private GitHub Security Advisory; do not open a public issue.
Pre-auth network-reachable; coordinate a release before public disclosure.

## References

- CWE-120: Buffer Copy without Checking Size of Input
- CWE-130: Improper Handling of Length Parameter Inconsistency
- CWE-787: Out-of-bounds Write
