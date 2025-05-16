## Preamble

This repository along with the instructions in this README can be used to extract HomeKit Pairing Identity Keys along with Paired HomeKit Accessory Keys from macOS. These keys can be used in order to interact with your already paired HomeKit devices through third-party software, such as the [Home Assistant HomeKit Device Integration](https://www.home-assistant.io/integrations/homekit_controller/), without needing to reset the device. This is especially useful when trying to have Home Assistant interface with Apple HomePods, as they can not be paired using the officially supported approach.

When following this guide for use with Home Assistant, please note that it is easily possible **to fully break your Home Assistant instance** by installing an invalid configuration file into your Home Assistant configuration's `.storage` directory, as these configuration files are normally **not intended to be edited manually**. Therefore, please make absolutely sure to backup your instance before proceeding, or at the very least to keep a copy of any unmodified files you extract from your `.storage` directory.

This guide uses a fork of the [KeychainKit](https://github.com/pvieito/KeychainKit) project, which contains some minor tweaks for it work on Apple Silicon machines running the most recent version of macOS.

## Disclaimer of Warranties

YOU EXPRESSLY AGREE THAT YOU ARE FOLLOWING THE BELOW INSTRUCTIONS AT YOUR SOLE RISK.\
THIS DOCUMENT IS PROVIDED "AS-IS", AND UNLESS OTHERWISE SPECIFIED IN WRITING,\
NO WARRANTIES OF ANY KIND ARE PROVIDED, NEITHER EXPRESS NOR IMPLIED.\
I WILL NOT BE HELD LIABLE FOR ANY POTENTIAL DATA LOSS OR DAMAGE DIRECTLY\
OR INDIRECTLY RESULTING FROM ANY OF THE DIRECTIONS GIVEN IN THIS DOCUMENT.

## Extract HomeKit Pairing Identity and Paired Accessory Keys

### Disable System Protections

* Boot your Mac into recoeryOS by shutting it down, holding the power button until "Loading Startup Options..." appears and choose the "Startup Options" entry.
* Within recoveryOS, open the Terminal via the "Utilities" menu bar section, disable System Integrity Protection and reboot.
```bash
$ csrutil disable
$ reboot
```
* Once back within macOS, disable Apple Mobile File Integrity and reboot.
```bash
$ sudo nvram boot-args="amfi_get_out_of_my_way=0x1"
$ sudo reboot
```

### Extract your Keys
* Ensure that the latest version of Xcode is installed and switch your active developer directory to your current installation.
```bash
$ sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer
```
* Fetch your code-signing certificate via Xcode account settings if it is not already present.
```
Xcode -> Settings... -> Accounts -> Manage Certificates -> (+) -> Apple Development
```
* Find your code-signing identity via Keychain Access, e.g. "Apple Development: appleid@example.com (FFFFFFFFFF)" and check its validity status.
  * If your certificate is shown as being invalid or not trusted, you will need to install the following two missing intermediate CA certificates:
    * Apple Worldwide Developer Relations Certificate Authority: https://developer.apple.com/certificationauthority/AppleWWDRCA.cer
    * Apple Worldwide Developer Relations Certificate Authority G3: https://www.apple.com/certificateauthority/AppleWWDRCAG3.cer
* Set the environment variable `CODESIGNKIT_DEFAULT_IDENTITY` to the name of your code-signing identity.
```bash
$ export CODESIGNKIT_DEFAULT_IDENTITY="Apple Development: appleid@example.com (FFFFFFFFFF)"
```
* Run KeychainTool to dump your keys:
```bash
$ git clone https://github.com/pseudorandomuser/KeychainKit.git
$ cd KeychainKit
$ mkdir dump
$ swift run KeychainTool -g "com.apple.hap.pairing" 1> dump/dump.txt 2>&1
```
* Proceed to the following sections only if the dump was successful and contains your keys, otherwise troubleshoot and try again.

### Restore System Protections
* Re-enable Apple Mobile File Integrity while still within macOS.
```bash
$ sudo nvram boot-args=""
$ sudo shutdown -h now
```
* Reboot into recoveryOS as outlined in the first step of the [Disable System Protections](#disable-system-protections) section.
* From within recoveryOS, re-enable System Integrity Protection and reboot.
  * Note: **It is required to connect to a network with a working Internet connection prior to re-enabling full system security.**
```bash
$ csrutil enable
$ reboot
```

### Install Python Dependencies Safely
Because macOS’s Python environment is externally managed (e.g., by Homebrew), direct system-wide pip installs are disabled to protect your system. To safely install the homekit[IP] package, use a Python virtual environment:

> **Important:** The `homekit[IP]` package depends on the `ed25519` library, which currently is incompatible with Python 3.13 due to deprecated API usage. Attempting to install on Python 3.13 results in build errors.

To avoid this, use Python 3.11 or 3.10 to create your virtual environment:

```bash
# Install Python 3.11.8 if you haven't already
pyenv install 3.11.8

# Set the local Python version for this project directory
pyenv local 3.11.8

# Create a new virtual environment with the selected Python version
python -m venv venv

# Activate the new virtual environment
source venv/bin/activate

# Confirm the Python version in the virtual environment
python3 -V  # Should output Python 3.11.8

# Install dependencies
python3 -m pip install --upgrade pip
python3 -m pip install "homekit[IP]"
```

### Discover HomeKit Devices on Network

* Run device discovery.
```bash
$ python3 -m homekit.discover
```
* Note down the Device ID values and find the correlation between the Device IDs in the discovery output and the Paired HomeKit Accessory Account values in [`dump/dump.txt`](dump/dump.txt).
  * Label the Paired HomeKit Accessory blocks in [`dump/dump.txt`](dump/dump.txt) by name and IP address for convenience to facilitate the following steps.

### Extract Information from Dump

* Extract the relevant information about your HomeKit Pairing Identity from [`dump/dump.txt`](dump/dump.txt).
* If there is more than one HomeKit Pairing Identity, determine which one is in use with your accessories.
  * It is most likely going to be the one with the most recent creation date prior to the creation date of the accessory in question.
  * If this is not the case, you will need to go about this in a trial-and-error manner and try to establish a connection with all of the identities until one of them allows you to connect.
  * Note: **Different accessories may be associated with different HomeKit Pairing Identities.**
```
[*] HomeKit Pairing Identity (com.apple.hap.pairing)
[ ] Class: Generic Password
[ ] Label: HomeKit Pairing Identity
[ ] Creation Date: XXXX-XX-XX XX:XX:XX AM +0000
[ ] Modification Date: XXXX-XX-XX XX:XX:XX AM +0000
[ ] Synchronizable: true
[ ] Access Group: com.apple.hap.pairing
[ ] Service: HomeKit Pairing Identity
[ ] Account: <iOSPairingId>
[ ] Key: “<iOSDeviceLTPK>+<iOSDeviceLTSK>”
```
* Note down `iOSPairingId`, `iOSDeviceLTPK`, `iOSDeviceLTSK`.
  * `iOSDeviceLTPK` and `iOSDeviceLTSK` are contained within the `Key` field and separated by a `+` sign.
* For each of the discovered accessories, extract the relevant information from [`dump/dump.txt`](dump/dump.txt).
```
[*] Paired HomeKit Accessory: XX:XX:XX:XX:XX:XX (com.apple.hap.pairing)
[ ] Class: Generic Password
[ ] Label: Paired HomeKit Accessory: XX:XX:XX:XX:XX:XX
[ ] Creation Date: XXXX-XX-XX XX:XX:XX AM +0000
[ ] Modification Date: XXXX-XX-XX XX:XX:XX AM +0000
[ ] Synchronizable: false
[ ] Access Group: com.apple.hap.pairing
[ ] Service: Paired HomeKit Accessory: XX:XX:XX:XX:XX:XX
[ ] Account: <AccessoryPairingID>
[ ] Key: “<AccessoryLTPK>”
```
* Note down the `AccessoryPairingID` and `AccessoryLTPK` values for each accessory.

### Populate Python HomeKit Pairing Configuration

* Create and populate [`dump/pairing.json`](dump/pairing.json) with the previously gathered information.
  * Replace all placeholders wrapped in angle brackets with their respective values.
```json
{
  "<DeviceName>": {
    "AccessoryPairingID": "<AccessoryPairingID>",
    "AccessoryLTPK": "<AccessoryLTPK>",
    "iOSPairingId": "<iOSPairingId>",
    "iOSDeviceLTSK": "<iOSDeviceLTSK>",
    "iOSDeviceLTPK": "<iOSDeviceLTPK>",
    "AccessoryIP": "<IPv4Address>",
    "AccessoryPort": <Port>,
    "Connection": "IP"
  },
  <...>
}
```

### Test the Connection to the Devices

* Run the following command for each configured device to verify the correctness of your configuration.
  * Replace `<DeviceName>` with the device name you defined in [`dump/pairing.json`](dump/pairing.json).
```bash
$ python3 -m homekit.get_accessories -f dump/pairing.json -a <DeviceName>
```
* If the connection fails, it is possible that you are using the wrong HomeKit Pairing Identity. In this case, you need to do the following:
  * Return to the [`Extract Information from Dump`](#extract-information-from-dump) section.
  * Repeat the first and second steps to extract `iOSPairingId`, `iOSDeviceLTPK` and `iOSDeviceLTSK` from a different identity in the dump.
  * [Update your `dump/pairing.json`](#populate-python-homekit-pairing-configuration) with the new values and try again.

### Bonus: Configure Home Assistant Integration

* On the device hosting your Home Assistant instance:
  * Stop your Home Assistant instance.
  * Extract the `.storage/core.config_entries` file from your configuration directory and copy it to [`dump/core.config_entries`](dump/core.config_entries).

* For each device, add an entry to [`dump/core.config_entries`](dump/core.config_entries) following the data structure below.
  * Replace all placeholders wrapped in angle brackets with their respective values.
  * `<AccessoryPairingID | lower>` represents the lower-case equivalent of the `<AccessoryPairingID>` value.
  * Replace <IPv4Address> and <Port> with the actual IP address and port number of your accessory, which you can find in the output of python3 -m homekit.discover. For example, Apple HomePods typically use port 55523, HomePod minis use 55313, and other devices might use ports like 8080.
  * Create <DateTime>
```python
python3 -c 'from datetime import datetime, timezone; print(datetime.now(timezone.utc).isoformat(timespec="microseconds"))'
```
```json
{
    "entry_id": "<Random128BitHexString>",
    "version": 1,
    "domain": "homekit_controller",
    "title": "<DeviceName>",
    "data": {
        "AccessoryPairingID": "<AccessoryPairingID>",
        "AccessoryLTPK": "<AccessoryLTPK>",
        "iOSPairingId": "<iOSPairingId>",
        "iOSDeviceLTSK": "<iOSDeviceLTSK>",
        "iOSDeviceLTPK": "<iOSDeviceLTPK>",
        "AccessoryIP": "<IPv4Address>",
        "AccessoryPort": "<Port>",
        "Connection": "IP"
    },
    "options": {},
    "pref_disable_new_entities": false,
    "pref_disable_polling": false,
    "source": "zeroconf",
    "unique_id": "<AccessoryPairingID | lower>",
    "disabled_by": null,
    "created_at": "<DateTime>",
    "discovery_keys": {},
    "minor_version": 1,
    "modified_at": "<DateTime>",
    "subentries": []
}
```
* For convenience, you can generate random 128-bit hex strings for use as `entry_id` by running the following:
```bash
$ openssl rand -hex 16
```
* As it is easily possible to **break your Home Assistant instance** by installing an invalid configuration file:
  * Make sure to backup your instance, or at the very least the unmodified file you extracted from `.storage/core.config_entries`.
  * Validate your modified [`dump/core.config_entries`](dump/core.config_entries) file using a JSON validator.
    * You may want to **refrain from using online JSON validators**, as this file contains a multitude of sensitive values such as the keys you just extracted.
    * One simple way to validate the modified file locally is to run the following command:
```bash
$ python3 -m json.tool dump/core.config_entries
```
  * Only if the JSON validation was successful, overwrite the `.storage/core.config_entries` file in your Home Assistant instance's configuration directory with your modified [`dump/core.config_entries`](dump/core.config_entries) file and start your Home Assistant instance.
