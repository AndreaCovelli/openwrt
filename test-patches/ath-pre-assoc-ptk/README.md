# ath10k/ath11k/ath12k pre-association PTK experiment

This experiment tests one question: after an AP-side FT station entry exists
but before it reaches `IEEE80211_STA_ASSOC`, can the selected ath driver and
firmware already install the pairwise key?

Use this diagnostic patch exclusively in a temporary firmware image on a
non-critical AP with a known recovery method. The experiment can reveal a
rejected command, a timeout, or a firmware bug.

## Scope

The test selector is `IEEE80211_HW_SW_CRYPTO_CONTROL`. In current wireless-next,
the only in-tree wireless drivers advertising it are ath10k, ath11k, and
ath12k. Stock ath10k is in scope, as are ath10k-ct builds that advertise the
same flag and ath11k/ath12k configurations using normal hardware crypto.

This flag selects each driver's real `set_key()` callback for the experiment.
Interpret every result for the exact driver, device, and firmware tested.

For ath12k, begin with a non-MLO client. The current ath12k MLO loop can return
success from its outer `set_key()` callback after a per-link operation fails.
For an MLO station, combine the tagged return value with all nearby ath12k
warnings and firmware messages.

## Compatibility

The patch was generated against pristine OpenWrt `backports-6.18.26`. Use it as
a standalone replacement for the AP-side FT deferred-key patch: remove
`400-mac80211-defer-ap-side-ft-key-upload.patch`, then place this test patch in
the same patch slot. Keep one patch 400 variant in the build at a time.

## Add it to an OpenWrt build

Copy `400-mac80211-test-ath-pre-assoc-ptk.patch` to:

```text
package/kernel/mac80211/patches/subsys/400-mac80211-test-ath-pre-assoc-ptk.patch
```

Remove the production/deferred patch, then rebuild the mac80211 package and
firmware using the tester's normal OpenWrt procedure. A clean mac80211 package
rebuild avoids reusing source prepared with the other patch:

```sh
make package/kernel/mac80211/clean
make package/kernel/mac80211/compile V=s
```

## Run the test

Record the exact device, radio/chip, driver, firmware version, and OpenWrt
revision. Follow kernel messages while repeating FT roams in both directions:

```sh
logread -f | grep -E 'mac80211 FT key test|key addition failed|failed to install key|firmware'
```

Exercise data traffic immediately after every roam. Repeat enough times to
trigger the original pre-ASSOC race. Tagged lines appear when a roam enters
the pre-ASSOC key-install window.

Interpret the tagged result as follows:

- `calling pre-ASSOC set_key` followed by `returned 0`: the complete driver
  callback accepted this early key request. Record whether roaming and data
  traffic also remained healthy. Collect repeated successes across relevant
  firmware and hardware variants.
- `returned 1`: software crypto was permitted. Classify this result as
  inconclusive for an early-upload capability.
- A negative return, install-key failure, timeout, warning, firmware crash, or
  loss of traffic identifies an unsupported ordering for that combination.
- `STA ... not uploaded; set_key skipped`: mac80211 recorded the station object
  before driver-peer readiness and skipped the callback.
- An absent tagged line usually indicates a roam outside the timing window, an
  image using a different patch, a configuration without
  `SW_CRYPTO_CONTROL`, or earlier interception by another patch.

Please return the complete tagged line pair, nearby driver/hostapd messages,
the status of the visible `key addition failed` message, and explicit
confirmation of post-roam traffic.

## Evidence required before an upstream opt-in

Treat each driver family separately. Before setting a production capability
for ath10k, ath11k, or ath12k, collect:

1. repeated tagged callbacks returning zero on representative hardware and
   firmware;
2. successful bidirectional data traffic immediately after tagged roams;
3. clean logs free of install-key timeouts, datapath errors, firmware warnings,
   and crashes; and
4. a code-path review confirming that the firmware peer and datapath peer exist
   before `IEEE80211_STA_ASSOC` for that driver.

Establish the capability independently for each driver family.
