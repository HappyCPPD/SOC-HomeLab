# SOC homelab: Wazuh on a single laptop

I wiped an old laptop, put Linux on it, and built a working Security Operations Center on it: a real SIEM collecting logs, catching attacks, and raising alerts. Then I wrote my own detection rule. This repo is the whole thing: how it's built, four attacks it catches, and honest notes on what it can and can't do.

**Start here:** [I wiped my laptop to build a SOC in my bedroom](posts/00-setup.md) — how the lab was built, from a wiped laptop to a running dashboard, and the one kernel setting that cost me half an hour.

## The lab

One laptop, running a single-node Wazuh stack in Docker, monitoring itself and generating its own attacks.

```
          Ubuntu host (Ryzen 7 5800H, 16 GB)
  ┌────────────────────────────────────────────────┐
  │  Docker: Wazuh single-node stack               │
  │    ├─ wazuh.indexer    (stores + searches)     │
  │    ├─ wazuh.manager    (analyzes, holds rules) │
  │    └─ wazuh.dashboard  (web UI, port 443)      │
  │                                                │
  │  Wazuh agent on the host        -- endpoint    │
  │  DVWA container (vulnerable web app) -- target │
  └────────────────────────────────────────────────┘
```

Stack: Wazuh 4.14.6, Docker, Ubuntu 26.04, `docker compose ps` confirms all three containers (`single-node-wazuh.dashboard-1`, `.indexer-1`, `.manager-1`) up.

16 GB of RAM won't run a multi-node cluster, so I ran one of each component and monitored the host itself as a free first endpoint.

## Detections

Four attacks, each caught end to end: the attack, the log it left, the rule that fired, and the MITRE technique it maps to.

| # | Detection | What fires | MITRE | Write-up |
|---|---|---|---|---|
| 1 | SSH brute-force | 5763 / 5712 | T1110 | [The two ways Wazuh sees a brute-force](posts/01-ssh-bruteforce.md) |
| 2 | File Integrity Monitoring | 550 / 553 / 554 | T1565 | [Detection without an attacker: watching files change](posts/02-fim.md) |
| 3 | Web attack (SQLi / XSS / traversal) | 31103 / 31105 / 31106 | T1190 | [Attempt, 200, or actually owned](posts/03-web-attack.md) |
| 4 | Custom rule, `/etc/passwd` access | 100020 (mine) | T1083 | [Writing my own detection rule](posts/04-custom-rule.md) · [rule XML](evidence/custom-rule/local_rule_100020.xml) |

Each write-up covers its own setup steps inline.

## The part I'm proudest of

Detections 1 through 3 use Wazuh's built-in rules: anyone can trigger those by following a guide. Detection 4 is mine. When my web-attack test tried to read the system password file through DVWA's file-inclusion page, the default rules only flagged it as a generic level-6 web attack, the same alert as someone poking at a search box. So I wrote a rule that escalates that specific act to critical:

```xml
<rule id="100020" level="12">
  <if_sid>31100</if_sid>
  <url>etc/passwd</url>
  <description>Critical web request attempting to read /etc/passwd (path traversal to system password file)</description>
  <mitre><id>T1083</id></mitre>
</rule>
```

`if_sid` is the idea that made it click: my rule runs on top of work the built-in rules already did, instead of re-parsing raw logs. I tested it with `wazuh-logtest`, fed it a log line and it told me exactly which rule fires, then watched 100020 fire live where the generic alert used to be. Full walkthrough in [Writing my own detection rule](posts/04-custom-rule.md).

## What the whole thing taught me

A brute-force is two things, not one. Wazuh forks on whether the username exists: guessing at a real account (5763) is a different rule than guessing at accounts that aren't there (5712). Same attack, different detection path.

Detection isn't only about attackers. File Integrity Monitoring never looks at who, it watches state and reports change. Half the job is knowing what normal looks like precisely enough to notice when it stops.

One alert can hide three claims. A web-attack alert can mean attempted, the server returned 200, or actually compromised, and a SIEM reading logs can only ever see the first two. The distance between "attack attempted" and "attacker got in" is the distance an analyst closes by hand.

Built-in rules are a foundation, not a product. The real skill is knowing your environment well enough to say "this specific thing matters more than the tool thinks," and encoding it.

## Limits and honesty

This is one laptop, so it's a single point of everything: if the box is off, there's no monitoring. There's no passive network capture yet, only host and web logs. Every detection has real gaps, documented in its write-up: the brute-force rule ignores a slow attacker, FIM can't tell my edits from an intruder's, the web rules match strings and miss encoded payloads, and my custom rule only covers one literal file path. None of that is hidden. Knowing the gaps is most of the point.

## Ethics

Every attack here ran against my own lab, on `127.0.0.1` or a local Docker container. I'd never point these tools at a system I don't own and don't have explicit permission to test.
