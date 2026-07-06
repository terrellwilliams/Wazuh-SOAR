# wazuh-soar

A lightweight, SOAR-style automated responder for Wazuh, written in Python. It tails the Wazuh alerts feed, matches alerts against playbooks, and fires response actions — notify a webhook, block a source IP at the firewall, or hand off to a human.

Built for and running in my home SOC lab (Wazuh manager on Proxmox, 23 onboarded endpoints). It's the automation layer that sits on top of the detections in [home-soc-detections](https://github.com/terrellwilliams/home-soc-detections).

## Why

Tier-1 work is mostly the same handful of responses to the same handful of alerts. This closes the loop on the noisy, obvious ones (brute-force from a public IP → block + notify) so a human only touches the alerts that need judgment.

## Safety model

This tool can modify firewall state, so it is **safe by default**:

- Runs in `--dry-run` unless you pass `--live` — dry-run logs exactly what it *would* do.
- `block_ip` also requires `enabled: true` in config — two switches must be on before any IP is ever blocked.
- Loopback and (by default) RFC1918 private IPs are never blocked, so you can't lock yourself out of your own lab.

## Quick start

```bash
git clone https://github.com/terrellwilliams/wazuh-soar
cd wazuh-soar
python -m pip install -r requirements.txt        # only PyYAML, optional
cp config.yaml.example config.yaml               # then edit

# Replay the bundled sample alerts (dry-run) to see it work:
python -m src.soar --config config.yaml --replay tests/sample_alerts.json

# Run the tests:
python tests/test_match.py
```

Live, against a real Wazuh manager:

```bash
# point alerts_file at /var/ossec/logs/alerts/alerts.json in config.yaml
python -m src.soar --config config.yaml --live
```

## How it works

```
Wazuh alerts.json ──tail──▶ soar.py ──match playbook──▶ dispatch actions
                                                          ├─ notify  (webhook)
                                                          └─ block_ip (firewall)
```

Playbooks live in `config.yaml`. Each maps a set of alert criteria (rule IDs, minimum level, or rule groups) to an ordered list of actions:

```yaml
playbooks:
  - name: ssh-brute-force
    match:
      rule_ids: [100001, 100002, 100003]
    actions: [notify, block_ip]
  - name: fim-critical-file
    match:
      rule_ids: [100010, 100011, 100012]
    actions: [notify]        # FIM never auto-blocks — human decides
```

## Layout

```
src/soar.py            # watcher + matcher + dispatcher
src/actions/notify.py  # webhook notification
src/actions/block_ip.py# firewall block (dry-run by default)
config.yaml.example    # playbooks + action settings
tests/                 # match-logic tests + sample alerts
```

## Roadmap

- [ ] Add an `isolate_host` action (quarantine VLAN via UniFi/OPNsense API)
- [ ] Enrichment step: OUI + GeoIP lookup before notifying
- [ ] Ship structured audit log of every action taken

---
Maintained by [Terrell Williams](https://github.com/terrellwilliams) · part of my home SOC lab.
