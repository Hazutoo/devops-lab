# Case Study: Removing Duplicate Zigbee2MQTT Action Triggers and Bounding Docker Logs

**Evidence date:** 8 November 2026  
**State:** Duplicate MQTT publication mechanism verified; automation change deployed and two physical double-click tests passed. Docker logging limits deployed and inspected; long-term rollover pending.

## Symptoms and boundaries

A five-service Docker homelab included Home Assistant, Zigbee2MQTT and an MQTT broker. Home Assistant repeatedly emitted `Already running` warnings for three light-switch automations. Zigbee2MQTT's Docker `json-file` log had reached **1,414,492,578 bytes (~1.35 GiB)** without rotation.

This is an engineering case study, not a live service dashboard. No private endpoint, unique device ID, backup path, account, hostname, hardware serial, or credential is included.

## 1. Trace MQTT actions before changing automation mode

The affected HA automations were configured with `mode: single`. Each double-click was subscribed to twice:

- `zigbee2mqtt/<switch>`: device JSON payload containing an `action` field
- `zigbee2mqtt/<switch>/action`: a dedicated single-action payload

A bounded Zigbee2MQTT application-log review confirmed that the same physical events were published on **both** topics, with **31, 16 and 6** matched publication pairs across the three respective switch actions.

The pre-fix HA log sample included **31, 16 and 6** `Already running` warnings for the corresponding automations. Because both subscriptions could fire for one gesture, increasing the automation concurrency or switching to `queued` could conceal the symptom while creating duplicate actions.

### Narrow fix

- Backed up the existing Home Assistant automation YAML.
- Removed only each redundant JSON/base-topic double-click trigger.
- Retained one `<switch>/action` trigger per double-click, each physical single-click trigger, and all existing action sequences.
- Compared parsed YAML before/after to ensure that unrelated automations remained unchanged.
- Validated the complete Home Assistant configuration and restarted **Home Assistant only**.
- Confirmed HTTP 200 after restart.

**Physical verification after change:** Double-click in two rooms still performed the intended lamp actions. The third switch's post-change gesture was **not** independently tested; later passive state history suggested functioning manual hold but is not a substitute for the full physical acceptance cycle.

**Still pending:** Verify whether any **new** `Already running` warnings occur after the patch. Historical log entries remain and must not be treated as new incidents.

## 2. Restore log-size discipline to Zigbee2MQTT

The application's high-volume informational messages were dominating an unbounded Docker `json-file` log. A guarded preflight confirmed that the live Zigbee2MQTT container matched its Docker Compose service configuration.

The scoped deployment added the following to the existing Zigbee2MQTT service (not all services):

```yaml
logging:
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"
```

Global Zigbee2MQTT logging remained at `info`; a `z2m:mqtt` namespace filter was persisted at `warning` to reduce routine MQTT publish noise.

### Verification

- YAML and Compose preflight validated.
- Only **Zigbee2MQTT** was recreated; other home services were not targeted.
- Zigbee2MQTT HTTP endpoint returned **200**.
- Docker inspect confirmed `json-file` with `max-size=10m` and `max-file=3`.
- The namespace-specific logging filter was still configured.
- A rollback copy and a limited tail of the old container log were retained locally by the operator.

The same successful deployment report was submitted twice, but the matching pre/post container identities and backup timestamp showed **one** deployment, not two.

**Still pending:** Observe an actual rotation cycle after the new log reaches its threshold and confirm sustained volume reduction. The configured maximum does not establish that real rollover has already occurred.

## 3. Adjacent reliability observations

Read-only checks recorded five running containers, no failed systemd units, healthy HTTP responses from three applications, low RAM pressure and no used swap during that snapshot.

Recent Zigbee2MQTT log samples included intermittent ping timeouts for two light devices and a single failed command for a smart outlet. These were not treated as proof of a whole Zigbee mesh outage.

Home Assistant Recorder had recorded light entities without brightness or color-temperature attributes; therefore the operator did not claim that a bulb does or does not remember brightness after manual/app control solely from Recorder history. A live-attribute check is a separate follow-up.

## Lessons

1. **Observe physical-event duplication at its source** before changing trigger behavior or concurrency.
2. **Retain only one canonical action subscription** when a publisher emits the same event on multiple topics.
3. **Protect every persistent config edit with preflight, backup, full validation and rollback.**
4. **Scope container recreation narrowly.** A logging change need not interrupt unrelated services.
5. **Distinguish configuration verification from behavior verification.** Real log rollover and ongoing quiet operation require a later observation.
6. **Keep private operational details out of public portfolio documentation.**

For related MQTT Discovery problems and Recorder metadata repairs, see the [earlier Home Assistant case study](2026-11-06-home-assistant-mqtt-discovery-and-energy-statistics.md). For encrypted off-host backup evidence, see the [Restic/SFTPGo case study](../backup/encrypted-restic-sftpgo-windows-backup-and-restore.md).
