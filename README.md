# SeedVault Backup Parser

This is a tool to decrypt and (partially) re-encrypt the android backups make by [Seedvault](https://github.com/stevesoltys/seedvault/).

## Requirements
For the AES decryption, the python dependency `pycryptodome` is needed.
Script only tested on Linux.

## Usage
To decrypt a backup stored in the folder `1601080173780` into `decrypted`, run
```
./parse.py decrypt 1601080173780 decrypted
```

The script will ask for your 12 word mnemonic key at runtime. It has to be lowercase, words separated by a single space.
Example:

```
fish test thing gift mercy siren erode acoustic mango veteran soup bus
```

The files created in the `full` directory are tar files and can be extracted with `tar -tvf`.

Re-encryption is currently only implemented for the key-value part of backups, not for the full app backups.

## Wifi Key Import
You can create a backup, modify it, and restore it back to the device. This allows to bulk-add wifi passwords without root access.

__WARNING__: I have tested this only for wifi passwords and do not entirely understand why the `@pm@` metadata needs to be present. [Googles Documentation](https://developer.android.com/guide/topics/data/testingbackup#TestingBackup) states `This action stops your app and wipes its data before performing the restore operation`. This does not happen for wifi passwords. The new ones simply get added to the store, no old ones are deleted. But things might go wrong!

```sh
# create a 'fake' plaintext backup
mkdir -p toencrypt/kv/com.android.providers.settings
mkdir -p toencrypt/kv/@pm@

# copy package manager metadata from decrypted backup, required for restoring backups
cp decrypted/kv/@pm@/meta_QG1ldGFA toencrypt/kv/@pm@/meta.QG1ldGFA

# wifi passwords live in com.android.providers.settings
# copy metadata and old passwords
cp decrypted/kv/@pm@/com.android.providers.settings.Y29tLmFuZHJvaWQucHJvdmlkZXJzLnNldHRpbmdz \
   toencrypt/kv/@pm@/com.android.providers.settings.Y29tLmFuZHJvaWQucHJvdmlkZXJzLnNldHRpbmdz
cp decrypted/kv/com.android.providers.settings/wifinewconfig.d2lmaV9uZXdfY29uZmln \
   toencrypt/kv/com.android.providers.settings/wifinewconfig.d2lmaV9uZXdfY29uZmln

# modify the old passwords file

# create a fake .backup.metadata file (based on real one?), change token to 1234
# example file shown below

# you know should have the following directory sturcture:
#   toencrypt/.backup.metadata
#   toencrypt/kv/com.android.providers.settings/wifinewconfig.d2lmaV9uZXdfY29uZmln
#   toencrypt/kv/@pm@/com.android.providers.settings.Y29tLmFuZHJvaWQucHJvdmlkZXJzLnNldHRpbmdz
#   toencrypt/kv/@pm@/meta.QG1ldGFA

# encrypt the fake backup with the same key the device uses. Output folder has to be numeric only and match the token
./parse.py encrypt toencrypt 1234

# copy the encrypted folder to somewhere seedvault detects it (usb/internal storage `.SeedVaultAndroidBackup`).
adb push 1234 /storage/emulated/0/.SeedVaultAndroidBackup/

# start the restore process with
adb shell bmgr restore 4d2 com.android.providers.settings
# note that 0x4d2 == 1234

# you might need to reboot if you get error -1000.
# somewhat detailed logs can be seen with
adb logcat
```

Example metadata file:
```json
{
    "@meta@": {
        "version": 0,
        "token": 1234,
        "time": 1601750759994,
        "sdk_int": 29,
        "incremental": "2020.09.11.14",
        "name": "Custom Wifi Restore"
    }
}
```

You can also import an old `wpa_supplicant` config, by saving it in `toencrypt/kv/com.android.providers.settings//WIFI.77-tV0lGSQ==` (filename taken from android source and generated by `base64.urlsafe_b64encode("\uffedWIFI".encode("utf-8"))`)

## Backup Format
The current backup format (as of 2020/10/04) is Version 0. Each file starts with a single byte specifying the used version. After that, a list of segments follows. Each is:

```
2 Bytes Segment Length x | 12 Bytes Encryption IV | x Bytes Encryted Segment Content
```

For Key-Value backups, the first segment contains a VersionHeader, which specifies the app and key.

The file `.backup.metadata` in the root of a backup contains information about which app was backed up when.


## License
This application is available as open source under the terms of the [Apache-2.0 License](https://opensource.org/licenses/Apache-2.0).
