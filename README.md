# Yealink desk phone call log for Home Assistant

[![Validate](https://github.com/SudoManaged/ha-yealink-call-log/actions/workflows/validate.yml/badge.svg)](https://github.com/SudoManaged/ha-yealink-call-log/actions/workflows/validate.yml)
![Home Assistant 2024.11+](https://img.shields.io/badge/Home%20Assistant-2024.11%2B-41BDF5)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

A call history for your Yealink desk phones inside Home Assistant: incoming, missed, answered and outgoing calls,
per phone and per line.

A free, open-source Home Assistant blueprint by SudoManaged. It runs entirely on your own Home Assistant: no
add-ons, no custom integrations, no cloud services, no accounts. It started as a call log we use every day on our
own Home Assistant, and was then generalized and tested for release.

## Contents

- [Requirements and tested versions](#requirements-and-tested-versions)
- [Installation](#installation)
- [What it is for](#what-it-is-for)
- [How it works](#how-it-works)
- [1. Add the sensor](#1-add-the-sensor)
- [2. Set up each phone](#2-set-up-each-phone)
- [3. Show it on a dashboard](#3-show-it-on-a-dashboard)
- [4. Use it in automations](#4-use-it-in-automations)
- [Maintenance commands](#maintenance-commands)
- [Tested phones](#tested-phones)
- [Troubleshooting](#troubleshooting)
- [Limitations](#limitations)
- [Choosing a webhook ID](#choosing-a-webhook-id)
- [Updating and removing](#updating-and-removing)
- [Security](#security)
- [Reporting problems and contributing](#reporting-problems-and-contributing)
- [More free Home Assistant tools from SudoManaged](#more-free-home-assistant-tools-from-sudomanaged)
- [Changelog and license](#changelog-and-license)

## Requirements and tested versions

- **Home Assistant 2024.11 or newer.** The blueprint declares this.
- **Yealink phones with Action URL settings** (on the phone's web page under **Features, Action URL**).
- **The phones must reach Home Assistant over plain HTTP** at the address you put in the URLs, normally
  `http://<Home Assistant's IP>:8123`. If your Home Assistant only answers HTTPS (you set `ssl_certificate` under
  `http:`), the phones would have to use `https://` and trust Home Assistant's certificate; that depends on the
  phone's firmware and we have not tested it.
- It passes Home Assistant's configuration check on **2024.11.0**, **2026.3.3** and **2026.11.0**. It was also
  run live on all three with simulated phone requests: answered, missed, outgoing, the `+` fix, two phones, a
  missed call that is not the newest, an unknown event, a call number reused after a reboot, `drop`, a reload of
  template entities, a Home Assistant restart, and the missed-call automation example below. The 50-call limit and
  unsupported variables were checked in a simulation of the template.
- Every change to this repository is checked automatically against Home Assistant 2024.11.0 and the current
  release (see [`.github/workflows/validate.yml`](.github/workflows/validate.yml)).

## Installation

This is a *template* blueprint: it creates a sensor and is used from `configuration.yaml`, as shown in
[1. Add the sensor](#1-add-the-sensor). First copy
[`blueprints/template/sudomanaged/yealink_call_log.yaml`](blueprints/template/sudomanaged/yealink_call_log.yaml)
to `/config/blueprints/template/sudomanaged/yealink_call_log.yaml` in your Home Assistant configuration folder.

## What it is for

A call history for your desk phones inside Home Assistant: who called, when, on which phone and which line, and
whether the call was answered or missed, plus the calls you placed. Useful on a dashboard, for "missed call while
I was out" automations, or just to see the history without walking to the phone.

## How it works

Yealink phones can call a web address on phone events, through their **Action URL** settings. Each phone calls a
local Home Assistant webhook when a call comes in, goes out, is missed and ends. This blueprint creates one
sensor that keeps the **50 most recent calls** in its `calls` attribute, newest first:

```yaml
calls:
  - t: "2026-11-01T12:22:08.512345-04:00"   # when the call started
    phone: Frontdesk                         # the ph= name of the phone, capitalized
    line: "8885550100"                       # the phone account (line) that took or placed the call
    number: "+15555550123"
    name: "Jane Doe"                         # caller ID name when the network sends one
    status: answered                         # ringing, answered, missed or outgoing
    key: frontdesk-32774                     # phone + the phone's call number, used to match later events
```

The sensor's state is the time of the last update, so you can trigger automations on it.

| Event the phone sends | What happens |
|---|---|
| Incoming call | A new entry with status `ringing` |
| Missed call | That entry becomes `missed` |
| Call ended | A `ringing` entry becomes `answered`; other statuses stay as they are |
| Outgoing call | A new entry with status `outgoing` (it stays `outgoing`; see [Limitations](#limitations)) |

"Answered" means the call rang and then ended without the phone reporting it as missed. A call you decline at the
phone, or one picked up on another phone that rang at the same time, may therefore also show as `answered`.

Several phones can share one sensor: each phone names itself with `ph=` in its URLs. Each call is matched by the
phone's name and the phone's own number for the call (`c=`). A phone may number its calls from the start again,
for example after a reboot. When a new call reuses the number of an older one, the older entry stays in the log,
with its start time (in Unix seconds) added to its key (for example `frontdesk-32774-1793550128`), so later events
only change the new call.

## 1. Add the sensor

Add this to `configuration.yaml` (if you already have a `template:` section, add the list item under it):

```yaml
template:
  - use_blueprint:
      path: sudomanaged/yealink_call_log.yaml
      input:
        webhook_id: YOUR_LONG_RANDOM_ID
        local_only: true
    name: Phone calls
    unique_id: phone_calls
```

| Setting | Default | Explanation |
|---|---|---|
| `webhook_id` | (required) | The secret part of the URL the phones call. Use a long random value (see [Choosing a webhook ID](#choosing-a-webhook-id)). |
| `local_only` | `true` | Accept reports only from private network addresses (`10.x`, `172.16.x` to `172.31.x`, `192.168.x`, IPv6 `fd00::/8`, link-local and loopback). Keep it on: phones are on your network. |

Check the configuration (**Developer tools, YAML, Check configuration**) and restart Home Assistant. You now have
`sensor.phone_calls`.

## 2. Set up each phone

1. Open the phone's web page (type its IP address into a browser; the phone shows it under **Menu, Status**) and
   sign in as admin.
2. Go to **Features, Action URL**.
3. Fill in these four fields. Replace `YOUR_HA` with your Home Assistant's address (an IP address is the most
   reliable), `YOUR_LONG_RANDOM_ID` with the webhook ID, and `frontdesk` with a short name for this phone (letters
   and digits only; it is shown capitalized, for example "Frontdesk").

| Field | URL |
|---|---|
| Incoming Call | `http://YOUR_HA:8123/api/webhook/YOUR_LONG_RANDOM_ID?ev=incoming&ph=frontdesk&c=$call_id&lu=$active_user&n=$remote&nm=$display_remote` |
| Outgoing Call | `http://YOUR_HA:8123/api/webhook/YOUR_LONG_RANDOM_ID?ev=outgoing&ph=frontdesk&c=$call_id&lu=$active_user&n=$remote&nm=$display_remote` |
| Missed Call | `http://YOUR_HA:8123/api/webhook/YOUR_LONG_RANDOM_ID?ev=missed&ph=frontdesk&c=$call_id` |
| Terminated (on some firmware: Call Terminated) | `http://YOUR_HA:8123/api/webhook/YOUR_LONG_RANDOM_ID?ev=ended&ph=frontdesk&c=$call_id` |

4. Click **Confirm**. Leave every other Action URL field as it is; other systems (call recording, hold music) may
   use them.
5. Repeat on each phone with its own `ph=` name.

**What the parts of the URL mean**

| Parameter | Filled in by | Meaning |
|---|---|---|
| `ev` | you | The event: `incoming`, `outgoing`, `missed` or `ended`. Any other value changes nothing (apart from the maintenance commands `clear` and `drop`). |
| `ph` | you | This phone's name |
| `c` | phone (`$call_id`) | The phone's own number for the call, which ties the events of one call together. Needed: without it, events cannot be matched to the right call. |
| `lu` | phone (`$active_user`) | The SIP user of the account that took or placed the call: which line it was |
| `n` | phone (`$remote`) | The other side's number or SIP address (cleaned to the number) |
| `nm` | phone (`$display_remote`) | The caller ID name when the network sends one, otherwise the number |

The phone sends `+` unencoded, which arrives as a space; the blueprint turns it back into `+`. If your firmware
does not support one of the variables, it sends the variable's name literally (for example `$display_remote`);
the blueprint then leaves that field empty. Without `$call_id` (`c`), calls cannot be told apart reliably (see
[Troubleshooting](#troubleshooting)).

**Test it:** call the phone and let it ring out, then call again and answer. Open **Developer tools, States**,
find `sensor.phone_calls` and look at the `calls` attribute: you should see one `missed` and one `answered` call.

## 3. Show it on a dashboard

Add a **Markdown** card with this content for a table of the last 25 calls:

```yaml
type: markdown
content: >-
  {%- set calls = state_attr('sensor.phone_calls', 'calls') or [] -%}
  {%- if not calls -%}No calls logged yet.{%- else -%}
  <table width="100%">
  <tr><th align="left">When</th><th align="left">Phone</th><th align="left">Line</th><th align="left">Caller</th><th align="left">Number</th><th align="left">Status</th></tr>
  {%- for c in calls[:25] %}{%- set t = c.t | as_datetime(none) %}
  <tr><td>{{ (t | as_local).strftime('%b %-d, %-I:%M %p') if t else '' }}</td><td>{{ c.phone }}</td><td>{{ c.line or '' }}</td><td>{{ c.name if c.name and c.name != c.number else '' }}</td><td>{{ c.number }}</td><td>{{ c.status }}</td></tr>
  {%- endfor %}
  </table>
  {%- endif %}
```

## 4. Use it in automations

Example: a phone notification for every missed call. It compares the log before and after each update, so each
missed call is announced once, even when it is not the newest call in the log or the phone sends more than one
event for it. Updates where the log was not there before (Home Assistant starting, or template entities being
reloaded) are skipped, so old missed calls are not announced again.

```yaml
alias: Missed call alert
mode: queued
triggers:
  - trigger: state
    entity_id: sensor.phone_calls
variables:
  before: >-
    {{ (trigger.from_state.attributes.get('calls', []) if trigger.from_state else [])
       | selectattr('status', 'eq', 'missed') | map(attribute='t') | list }}
  new_missed: >-
    {{ (trigger.to_state.attributes.get('calls', []) if trigger.to_state else [])
       | selectattr('status', 'eq', 'missed') | rejectattr('t', 'in', before) | list }}
conditions:
  - condition: template
    value_template: >-
      {{ trigger.from_state is not none and 'calls' in trigger.from_state.attributes
         and new_missed | count > 0 }}
actions:
  - repeat:
      for_each: "{{ new_missed }}"
      sequence:
        - action: notify.mobile_app_your_phone
          data:
            title: Missed call
            message: "{{ repeat.item.name or repeat.item.number or 'Unknown caller' }} on {{ repeat.item.phone }}"
```

Calls are compared by their start time (`t`), which never changes, rather than their key, which changes when a
reused call number moves an old entry aside.

## Maintenance commands

Open these in a browser on your network:

| Action | URL |
|---|---|
| Empty the log | `http://YOUR_HA:8123/api/webhook/YOUR_LONG_RANDOM_ID?ev=clear` |
| Remove one call | `http://YOUR_HA:8123/api/webhook/YOUR_LONG_RANDOM_ID?ev=drop&ph=frontdesk&c=CALL` where `CALL` is everything after the first `-` in that call's `key` (for `frontdesk-32774-1793880035` that is `32774-1793880035`) |

## Tested phones

- Yealink SIP-T42G, firmware 29.83.0.120.

Other Yealink T- and SIP-series phones have a similar Action URL page, but we have not tested them. If yours works
(or does not), please open an issue with the model and firmware so we can list it here.

## Troubleshooting

- **Nothing is logged.** Open this in a browser on your network, with your address and ID:
  `http://YOUR_HA:8123/api/webhook/YOUR_LONG_RANDOM_ID?ev=incoming&ph=test&c=1&n=5555550123`. If that adds a
  call, the phone is not calling the URL: check that the phone can reach Home Assistant's address and that Home
  Assistant answers plain `http` there (see [Requirements](#requirements-and-tested-versions)). If it does not,
  check the webhook ID and the configuration, and look in **Settings, System, Logs** for "Received remote request
  for local webhook" (the browser was not on a private address). Remove the test entry afterwards with
  `...?ev=drop&ph=test&c=1` (if you opened the test URL more than once, drop each entry by its key, as in
  [Maintenance commands](#maintenance-commands)).
- **Unanswered calls stay "ringing".** The Missed Call URL is missing or in the wrong field. Some firmware only
  reports missed calls when call history is enabled on the phone.
- **The line column is empty.** Your firmware does not support `$active_user` in Action URLs.
- **Calls less than 2 minutes apart replace each other, and keys end in `-`.** The phone does not fill in
  `$call_id`; check the `c=` part of the URLs.

## Limitations

- Outgoing calls stay `outgoing`, and "answered" is worked out from "rang, then ended": the blueprint does not use
  Yealink's **Established** and **Reject Incoming Call** events, which some firmware also offers. So a call
  declined at the phone, or picked up on another phone that rang at the same time, may show as `answered`.
- The phone does not encode special characters in names: a caller ID name containing `&` is cut short at it
  (`Smith & Sons` shows as `Smith`).
- The log lives in the sensor's attributes: it survives restarts, but the recorder may skip storing attribute
  history when it grows large. 50 calls stay well within the limit.
- The webhook is unauthenticated apart from its ID, like all Home Assistant webhooks; keep it local-only.

## Choosing a webhook ID

The webhook ID works like a password: anyone on your network who knows it could add fake calls to your log or
clear it. Generate a long random one, for example:

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
# or
openssl rand -hex 32
```

To change it later, change `webhook_id` in `configuration.yaml`, restart Home Assistant, and update the four Action
URLs on every phone.

## Updating and removing

- **Updating:** replace the blueprint file with the new version and restart Home Assistant. The sensor keeps
  its call history.
- **Removing:** delete the `template:` entry that uses the blueprint and restart. Because the sensor had a
  `unique_id`, its entity stays listed under **Settings, Devices & services, Entities** as unavailable; delete it
  there. Then delete the blueprint file and clear the Action URL fields on the phones.

## Security

- The blueprint is **local-only by default**; the webhook ID works like a password, so use a long random ID and do
  not publish it.
- Nothing is sent outside your Home Assistant.
- To report a security problem privately, use **Report a vulnerability** on this repository's
  [Security tab](../../security). Please do not open a public issue for security problems. See
  [SECURITY.md](SECURITY.md).

## Reporting problems and contributing

- **Something does not work:** open an [issue](../../issues) with your Home Assistant version, your phone model
  and firmware, what you expected and what happened. The `calls` attribute around the problem helps a lot; remove
  phone numbers and names from it first.
- **Improvements:** pull requests are welcome. Keep the blueprint self-contained (no custom integrations),
  local-only by default, and documented in this README.

## More free Home Assistant tools from SudoManaged

- [Webhook alerts](https://github.com/SudoManaged/ha-webhook-alerts)
- [Bluetooth self-heal](https://github.com/SudoManaged/ha-bluetooth-self-heal)
- [Dehumidifier kit](https://github.com/SudoManaged/ha-dehumidifier-kit)

## Changelog and license

See [CHANGELOG.md](CHANGELOG.md). Released under the [MIT License](LICENSE). Copyright (c) 2026 SudoManaged.
