---
layout: post
title: 	Triggering Immediate Check-ins with Fleet Automations
---

A common pattern in endpoint configuration management is to audit a system for a desired state, and if a system deviates from this state, trigger remediation to bring it back into compliance.

This is how we handle almost all of our endpoint configuration management, powered through use of Jamf's Extension Attributes, Smart Groups, and Policies. The moving parts look something like this:

1. Update Inventory job runs recurringly on computers, triggering Extension Attribute evaluation.
2. An Extension Attribute script runs on the computer to evaluate for desired state, and returns a Pass / Fail value to Jamf.
3. Computers with a Fail value in that Extension Attribute are automatically added to a corresponding Smart Group.
4. A Policy is scoped to the Smart Group, which attempts to remediate the problem, and then executes the "Update Inventory" payload.


While this works well enough, the mean time-to-detect and time-to-remediate tends to be pretty delayed, since both the execution of the Extension Attribute script and Policy will not trigger until an "Update Inventory" job is run, which we currently only run once a day, save for manual `jamf recon` invocations. Further compounding the delay, Smart Group membership updates may happen after an Update Inventory job, meaning remediation policies often require an additional inventory cycle before they execute.

In practice, it can end up taking up to two days before a given deviation is detected and remediated.


# Doing things with Fleet

We've been using [Fleet](https://fleetdm.com/), and have begun evaluating their Premium offering. One of the features in Premium is the ability to run [automations](https://fleetdm.com/guides/automations#policy-automations) upon the detection of a failing policy. Of particular interest to us, is the ability for it to run a script automatically. Much like how Jamf's policy works, we can use this to remediate deviations.

The cool thing about this is that unlike Jamf, which requires you to orchestrate Jamf to trigger the remediation as described above, Fleet automatically executes the policy automation basically immediately. Assuming the computer is online, this means that detection to remediation is roughly 30 minutes, the default check in cadence for the Fleet agent.

However, while the remediation may actually be applied within those 30 minutes, the policy will continue to show as unhealthy until the *next* time the Fleet agent checks in. Unfortunately, there is currently no built-in way to make Fleet check for policy state upon a policy automation being triggered.

Fortunately, there *is* a workaround!

## Automatically triggering Fleet check-ins after remediation


While Fleet offers no built-in way to trigger a check in after a policy automation, we *can* do this programmatically, by appending check in logic at the end of every script triggered by a policy automation. But there is no command-line equivalent to `jamf recon`.

Instead, we can easily accomplish this via Fleet's API, which provides a [refetch device's host route](https://github.com/fleetdm/fleet/blob/004027cca26546a112ba8cede25019500a8d1ea8/docs/Contributing/API-for-contributors.md#refetch-devices-host).

This request is authenticated using a device's token, which means most of our concerns around how we'll handle authentication are addressed. However, the device's token is only made available on the device if the agent was installed with [fleet-desktop](https://fleetdm.com/guides/fleet-desktop), so if your fleet agent was not installed with this, you'll need to rebuild your installer and reinstall.

Once the device token is available (by default, it should exist at `/opt/orbit/identifier`), you should be able to simply make a POST request to the endpoint using curl at the end of any given script.

```bash
DEVICE_TOKEN=$(cat /opt/orbit/identifier)
FLEET_URL='https://your-fleet-url.com'
curl -X POST "${FLEET_URL}/api/latest/fleet/device/${DEVICE_TOKEN}/refetch" \
  -H "Content-Type: application/json"

exit 0
```

This way, the Fleet agent will automatically check in after a policy automation runs. 🎉

It would be great if Fleet did this natively! To that end, I submit a feature request [here](https://github.com/fleetdm/fleet/issues/28523).


## Considerations

### Fleet Desktop Dependency:
Unfortunately, this depends on Fleet Desktop being installed on the computer, which ensures the device token is populated. Without this, you'd need to generate an API token and expose it on the endpoint. I don't really recommend this.

### Potential for multiple refetch calls:
If you did use this pattern across all your policies, and you had a lot of policies that fail at once, it's possible you could end up triggering the `refetch` job *a lot*. I don't know if this'll have any negative repercussions, but if Fleet were to implement this capability natively, I'd recommend they call a single `refetch` only after all policy automation jobs are completed.

### Increasing Jamf Inventory Update frequency as an alternative:

You could! Unfortunately, Jamf's Update Inventory job is triggered via a Policy, and Policy scheduling is not very granular. Outside of setting this to "Ongoing", where it'll run every time it checks in, the shortest recurring interval is "Once every day".

You *could* shorten the interval by orchestrating the Update Inventory job outside of Jamf; for example, using a custom event trigger or running `jamf recon` via cron. But personally, I don't think the added complexity is worth it.