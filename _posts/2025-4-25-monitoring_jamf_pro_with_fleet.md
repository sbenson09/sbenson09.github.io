---
layout: post
title: Monitoring Jamf Pro with Fleet
---

We use Jamf Pro as our MDM, and our main means of configuration management for our macOS endpoints. Most of the time, this works very well, but occasionally we've observed failures between the jamf agent installed on the endpoint, and the Jamf server, resulting in the agent failing to check in for prolonged periods of time. This has significant knock-on effects: if the jamf agent isn't working on a system, other things start to fall apart.

One of the challenges in fixing this was that we were wholly dependent on Jamf for measuring and managing the states of our endpoints. To monitor Santa, our EDR, our zero-trust VPN, or any of the other various things we'd monitor on a system, Jamf is enough! But Jamf can't monitor itself; if the jamf agent is broken on a system, it can’t report anything back.

This was one of the biggest reasons we wanted to deploy Fleet in our environment: Fleet would give us the ability to automatically identify and (potentially) remediate broken jamf agents.

Fleet has a vast number of osquery tables at its disposal, but unfortunately, none of them are directly relevant to understanding the state of the jamf agent and specifically, whether or not the jamf agent has recently checked in. Fleet *does* come with some capabilities to read certain filetypes or content from disk (e.g. [plist](https://fleetdm.com/tables/plist#apple), [json](https://fleetdm.com/tables/parse_json#apple), etc.), but as far as I know, there's nothing on the filesystem that will give us the information we need in a way Fleet will natively understand. *However*, there is something of use on the filesystem we'll be able to use!

## Parsing the /var/log/jamf.log

By default, the jamf agent writes its logs to the `/var/log/jamf.log`. To figure out what log entries were most common, I put a quick script together to parse and sort them:
```bash
#!/bin/bash

# Usage: ./script.sh logfile.log

if [ -z "$1" ]; then
    echo "Usage: $0 [logfile]"
    exit 1
fi

sed -E 's/^.*jamf\[[0-9]+\]: //' "$1" | sort | uniq -c | sort -nr
```

This returns output like this:
```
6407 Checking for policies triggered by "recurring check-in" for user "sbenson"...
6358 Checking for patches...
6352 No patch policies were found.
 556 Removing existing launchd task /Library/LaunchDaemons/com.jamfsoftware.task.bgrecon.plist...
 184 Executing Policy Update Jamf Inventory
 184 Executing Policy Assign User Information to Device
 [etc.]
```
This gives us a great sense of what would be a reliable indicator for a healthy jamf agent. In particular, looking for lines containing `No patch policies were found` or `Executing Policy` will be great for our purposes, because it suggests the agent is communicating with the server, and they are frequently represented in our log. This means that if we have logs where these lines don't show up, it's a good indicator the jamf agent isn't working.

However, in its current form, Fleet won't be able to parse this log file. To work around that, we can leverage a Jamf extension attribute to deploy a script that periodically runs on the endpoint, to parse the log, identify the latest timestamp associated with a log entry that corresponds with successful communication between the jamf agent and Jamf server, and then write that timestamp to disk in a format Fleet will be able to use.

The following script should suffice:
```bash
#!/bin/bash
# Jamf EA: last_successful_checkin 
#
#  Returns one of:
#    • 2025-04-25T09:21:38Z  (success stamp)
#    • no_success
#    • no_log
#
#  On success: writes JSON file for Fleet.
#  On failure: leaves the prior JSON untouched.

# config
LOGS=(/private/var/log/jamf.log*) # includes rotated logs
DIR="/opt/telemetry/jamf"
FILE="${DIR}/last_checkin"
SUCCESS='Executing Policy|No patch policies were found|No policies were found|Submitting log to' # Log items indicating success

if [[ ! -d "$DIR" ]]; then
  /usr/bin/install -d -o root -g wheel -m 700 "$DIR"
fi

shopt -s nullglob # glob silently to an empty list if no match
if (( ${#LOGS[@]} == 0 )); then
  echo "<result>no_log</result>"
  exit 0
fi

# locate most‑recent successful line
ts_line=$(grep -aE "$SUCCESS" "${LOGS[@]}" | tail -1)

if [[ -z "$ts_line" ]]; then
  echo "<result>no_success</result>"
  exit 0
fi

# convert timestamp
ts=$(awk '{print $2" "$3" "$4}' <<<"$ts_line") # e.g. Apr 25 09:21:38
epoch=$(date -j -f "%b %d %T" "$ts" "+%s") || exit 0
iso=$(date -u -r "$epoch" "+%Y-%m-%dT%H:%M:%SZ")

# write JSON for Fleet
printf '{"last_successful_checkin":"%s"}\n' "$iso" > "$FILE"
chown root:wheel "$FILE"
chmod 600 "$FILE"

# return ISO stamp to Jamf
echo "<result>$iso</result>"
```

This should result in a file at `/opt/telemetry/jamf/last_checkin` with the latest timestamp stored in json.

Extension attributes are generally used to capture data from the device in Jamf Pro, but in this case we don't really need to do that. Jamf already does capture a device's last check-in time anyway. Instead, I opt to capture whether or not the script ran into errors, for easier troubleshooting in the future.

## Using Fleet to Monitor

Now that the data we need is available and in a format Fleet can work with, using it is pretty simple. Here's a sample query and policy:

Query:

```SQL
SELECT *
FROM parse_json
WHERE
    path = '/opt/telemetry/jamf/last_checkin' AND
    key = 'last_successful_checkin'
```

Policy (Pass if last checkin is less than 14 days ago):

```SQL
SELECT 1
FROM parse_json
WHERE
    path = '/opt/telemetry/jamf/last_checkin' AND
    key = 'last_successful_checkin' AND
    datetime(value) > datetime('now', '-14 days')
```

In our case, we leverage something like this as a component in a much larger query & policy, which also looks at other data points relevant to the health of Jamf on our endpoints (e.g. presence of relevant .Apps, Configuration Profiles, LaunchDaemons, etc.)

## Remediation

Once we detect a broken jamf agent, we need to actually fix it. To do this, we've been leveraging [a self-heal script I made](https://github.com/sbenson09/jamf-self-heal), inspired by a post I read by a `Dr. K` at [modtitan.com](https://www.modtitan.com/2022/02/jamf-binary-self-heal-with-jamf-api.html). The TL;DR is the script makes a request to the Jamf API to use MDM to redeploy the Jamf management framework on the computer specified.

We run this script manually, as running it automatically would mean exposure of our API credentials on our endpoints, which we want to avoid.

# Conclusion

This approach has been working well for us as a lightweight safety net around Jamf’s self-awareness gap. It’s simple, low-risk, and integrates seamlessly with our existing observability in Fleet.

## Other Considerations

#### Isn't last check-in already present in Jamf Pro's web UI?

It is! But the issue is that Jamf has no way of knowing if the computer is actually on or not.

So, sometimes a jamf agent that hasn't checked in for 20 days means the agent is broken, while other times, it means the owner has been on vacation.

The above approach works because if Jamf log doesn't show recent check in, but Fleet is able to communicate with the device, then it means the device is online, and something is wrong with the jamf agent.

#### What about false positives?

Our strategy with Fleet is to really dial in our policies, such that we can alert on them when they fail and action needs to be taken. To that end, minimizing false positives is vital. 

In its current state, we may run into a race condition, where if a computer that has been powered off for a long period of time is powered on, and Fleet evaluates its policy before the jamf agent has had the chance to check in, the device will fail our Fleet policy, even though technically the jamf agent is healthy. This has been rare, and ultimately will sort itself out by the next time Fleet re-runs the policy.

## Future Improvements

#### Parse `/var/log/jamf.log` for explicit errors

When I developed this process, it wasn't clear to me that there was a common root cause across our broken jamf agents. After observing for the last couple months, I've seen `Device Signature Error - A valid device signature is required to perform the action.` this consistently, so in a future revision, it might make sense for us to take this approach. This should be more reliable and address the race condition described above.

#### Automate remediation

Using Fleet's policy automations, we could automatically deploy this script locally, and accept the risk of an exposed API credentials. If we scope the permissions of the credentials to a very limited set, the risk is pretty low.

As an alternative, we could leverage Fleet's policy automations to instead fire a webhook, which could trigger a cloud function to run the self-heal script instead. Doing things this way would help us keep the API token from being exposed on endpoints.