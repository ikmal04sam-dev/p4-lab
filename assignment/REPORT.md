# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **1761**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop

<!-- How did you create the tunnel loop (which entry, on which switch, which commands)?
     What did the switch logs show over time, with numbers from loop-before.txt?
     Then look at the IPv4 path in tunnel.p4: it decrements the TTL. Does that stop an IPv4
     loop in this program? Check the code, and explain what happens when TTL reaches 0. -->
    I created the tunnel loop by modifying entry 0x2 on switch s2 using `p4cli` to forward packets to port 1 back toward s1, while s1 forwarded exit 3 to s2. As recorded in assignment/evidence/loop-before.txt, the switch processing counts escalated uncontrollably over five seconds because no stopped the cycle. The IPv4 TTL decrement inside ipv4_forward does not prevent this loop because tunnelled packets enter the if (hdr.myTunnel.isValid()) branch, completely bypassing `ipv4_lpm` so the inner IPv4 TTL is never touched or checked.

## 2. Design: the hop limit

<!-- Your header change and your pipeline change, and the reason for each choice:
     - the field's size, and where in the header you put it (and why there);
     - the starting value send.py uses, and how you chose it;
     - where it's decremented, and what happens when it runs out;
     - what else had to change for every part of the path to agree (parser, deparser, hosts). -->
     I appended an 8-bit `hop_limit` field at the end of `myTunnel_t` after `dst_id`, allowing for 255 hops without changing the byte positions of the other fields. The default initial value of this field in the sender script `send.py` is `64`, offering plenty of headroom for legitimate paths while preventing any infinite loops. In `MyIngress`, the switch verifies that `hop_limit` is not equal to `0` before forwarding; when it is `0`, the packet is dropped, and when it is non-zero, `hop_limit` is reduced by `1` before using `myTunnel_exact`. In order to maintain consistency on the path, `parse_myTunnel` and deparser have been changed to strip and output the 5-byte header, respectively, and the Python hosts scripts were changed to include `ByteField`.

## 3. Implementation

<!-- Summarise what you changed in tunnel.p4, send.py and receive.py. The pull request shows
     the code itself: here, explain anything a reviewer might not see from the diff alone. -->
     I have extended the `header myTunnel_t` with `bit<8> hop_limit` and modified the `MyIngress` `apply` code to make sure that the zero check and decrementation happens only if there is a valid tunnel header. I have included the `--hops` command line argument in `send.py` and have modified the Scapy `MyTunnel` class with `ByteField("hop_limit", 64)` so that the packet contains the budget. The `receive.py` code has been modified in the similar way to parse the 5 byte header without making any alignment issue in the inner IPv4 packet.

## 4. Evidence: it works

<!-- Show, with your evidence files:
     - tunnelled delivery to h2 and h3, and the hop limit the receivers saw (delivery.txt);
     - plain IPv4 still works (delivery.txt);
     - the same loop as in section 1 now ends, and after how many passes — does that match
       your design? (loop-after.txt) -->
       in `assignment/evidence/delivery.txt`, when using `--hops 10`, the packet with the value of `hop_limit=9` got to `h2` through `s1`, and the packet with `hop_limit=8` got to `h3` through `s1` and `s2` while `h1 ping -c 2 h3` worked successfully with 0% packet loss. In the file `assignment/evidence/loop-after.txt`, running the loop test on `s2` with `--hops 10` led to switch processing counts getting stuck at `logs/s1.txt:1221` and `logs/s2.txt:1220` in 5 seconds.

## 5. Evaluation

<!-- a) The mixed rollout (mixed.txt): what did you predict before running s1 with your new
        program and s2 with tunnel_v1.p4? What actually happened to delivery, and to the loop?
        Explain every difference from your prediction, byte by byte if you need to.
     b) Propose a safe way to roll out a header change across a network of switches that can't
        all be upgraded at once. Compare at least two options (for example a version field, a
        new EtherType, or ordering fields so old switches still parse them).
     c) Loop protection in the data plane (this hop limit) versus the control plane (Week 3's
        check_paths()): what can each catch that the other can't? Why do real networks use both?
     d) What would this change cost on a hardware switch running at line rate? -->
     Before running the mixed rollout, I predicted that `s2` (running the old 4-byte `tunnel_v1.p4`) would treat the 5th byte (`hop_limit`) as the start of the inner IPv4 header. As confirmed in `assignment/evidence/mixed.txt` and `logs/h2cap.txt`, delivery failed because `s2` stripped only 4 bytes during deparsing, leaving the `hop_limit` byte prepended to the IPv4 header and corrupting the IP version/length byte (`0x45`), while any loop would still terminate at `s1` once `s1` decremented the field to zero.

     To safely roll out a header change across switches, operators can either assign a new EtherType (such as `0x1213`) so unupgraded switches cleanly drop unrecognized frames rather than misparsing payloads, or perform an egress-to-ingress phased upgrade so legacy switches never receive new headers from upgraded upstream nodes. Comparing loop protections, data plane hop limits react at line rate to stop transient micro-loops and runtime misconfigurations immediately, whereas control plane checks like Week 3's `check_paths()` detect structural topology loops globally before rules are installed; real networks use both so the control plane prevents bad routes while the data plane caps transient forwarding errors. On a line-rate hardware switch, adding an 8-bit header field and decrement check slightly increases parser extraction buffer width and consumes an extra ALU/match-action stage per packet.

## AI use

<!-- This assignment is Level 3 (Open): AI use is allowed if you declare it. State which tools you
     used (or "None") and what each did — for example "ChatGPT explained the bmv2 log format;
     the first draft of the hop-limit check came from Copilot, which I then changed and tested".
     Be ready to explain any part of your work yourself. -->
     I used Gemini (Google AI) to verify P4 header and Scapy `ByteField` syntax, clarify the byte-shift behavior during the mixed rollout test, and refine the wording of this report. i also use AI for the coding

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
