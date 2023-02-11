# "Vaultlocker" for ZFS datasets

[Vaultlocker](https://github.com/openstack-charmers/vaultlocker) is a tool to manage the encryption keys for LUKS encrypted block devices by storing them in [Hashicorp Vault](https:;//vaultproject.io).  It generates a random encryption key for each new LUKS volume which it stores in Vault, and sets up a oneshot systemd unit to retrieve the key from the Vault at boot time and unlock the block device without requiring human intervention.

This tool extends Vaultlocker to do the same job for ZFS encrypted datasets.  It deliberately re-uses the vaultlocker configuration file for the Vault URL and credentials, so systems with both encrypted LUKS devices and encrypted ZFS datasets can manage both with a single set of credentials.

## Installation

1. Ensure Python 3.7+ and the required dependencies are installed - on Ubuntu you can `apt install python3-hvac python3-tenacity python3-pexpect`, or use `pip install -r requirements.txt`
2. Copy `zfs-vaultlocker` to `/usr/local/sbin` and make it readable and executable only by root
3. Copy `zfs-vaultlocker-decrypt@.service` to `/etc/systemd/system`, and `systemctl daemon-reload`

If the dataset you want to mount includes `/usr/local/sbin` then you will need to place the script somewhere else and modify `zfs-vaultlocker-decrypt@.service` to match.

## Configuration

The `zfs-vaultlocker` tool uses the same configuration file as `vaultlocker`, at `/etc/vaultlocker/vaultlocker.conf`.  The `/etc/vaultlocker` directory and the files within it should be readable only by root, as they include an AppRole secret that is used to authenticate to Vault.

```
[vault]
url = https://vault.example.com:8200
approle = {uuid-of-approle}
secret_id = {corresponding-secret-id}
backend = secret
```

`backend` must be a _version 1_ `kv` backend - `zfs-vaultlocker` uses a relatively old version of the `hvac` Python client library (for compatibility with the original `vaultlocker`) which does not handle version 2 `kv` stores.  The `approle` and `secret_id` must give read-write permission on the relevant secrets if you want to be able to create datasets with `zfs-vaultlocker` - it is possible to use a read-only approle for `zfs-vaultlocker decrypt` but you will have to create the dataset by hand and add the keys at the right location in the secret store.

## Usage

### Creating a new encrypted dataset

```
zfs-vaultlocker create {zpool}/{dataset} [args-to-zfs-create]
```

Creates a new ZFS dataset encrypted with a randomly generated key, stores that key as a secret in Hashicorp Vault at `{prefix}/{hostname}/{zpool}/{dataset}`, and enables a systemd unit at boot that calls `zfs-vaultlocker decrypt` to fetch the key and mounts the datasets that the key has unlocked.

The only required parameter for the `create` command is the name of the dataset, any additional arguments beyond that will be passed through verbatim to `zfs create`, and can be used to set other options such as `mountpoint`.  The `encryption`, `keylocation` and `keyformat` options are set by `zfs-vaultlocker`, it is an error to specify them explicitly.

### Unlocking a dataset

```
zfs-vaultlocker decrypt {zpool}/{dataset}
```

Retrieves the encryption key for the given dataset from the Vault and feeds it to `zfs load-key`.  This command does _not_ itself actually mount the dataset, but you should be able to do that once the key has been loaded.

### Automatic unlocking at boot

When creating a dataset with `zfs-vaultlocker create`, a systemd unit is enabled that will fetch the key using `zfs-vaultlocker decrypt` and if that is successful then call `zfs mount -a`.  This should mount the target dataset as well as any descendent datasets that share the same `encryptionroot`.  Any other units that require files or directories within the encrypted dataset should declare a `Requires` and `After` dependency on the decryption service via a drop-in:

```
[Unit]
Requires=zfs-vaultlocker-decrypt@{zpool}-{dataset}.service
After=zfs-vaultlocker-decrypt@{zpool}-{dataset}.service
```

Notes:

1. since `zfs-vaultlocker` can only access the Vault once the network is available, any other units that depend on the encrypted datasets being available must also be deferred until after `network-online.target` (which should happen automatically with a dependency on `zfs-vaultlocker-decrypt@...`).
2. The part after `@` and before `.service` is the standard systemd "escaped" form of `{zpool}/{dataset}`, with forward slashes converted to hyphens.  If the dataset name itself includes any hyphens then those also need to be escaped as `\x2d` - if in doubt use `systemd-escape {zpool}/{dataset}` to generate the correct escaped form.

## Special cases

### Generating a key without creating the dataset

```
zfs-vaultlocker generate-key {zpool}/{dataset}
```

Occasionally you may need to generate and store a random key separately from creating the dataset in question.  The intended use case for this is where you want to create the dataset using `zfs receive` from an existing dataset (on the same or another machine) sent using `zfs send -R`.

Since the encrypt-on-receive logic cannot use `keylocation=prompt`, you can instead run `zfs-vaultlocker generate-key {zpool}/{dataset}` to generate the key and store it in Vault as normal, but instead of using the key immediately to create a dataset this command will write the key to `/dev/shm/zfs-vaultlocker-{randomstring}.key` in hex format.  You can then specify `-o encryption=aes-256-gcm -o keyformat=hex -o keylocation=file:///dev/shm/{file-that-was-just-created}` as options to `zfs receive` to encrypt the dataset as it is received.

Once received, delete the temporary key file and run

```
zfs set -o keylocation=prompt {zpool}/{dataset}
zfs-vaultlocker enable-unit {zpool}/{dataset}
```

to configure the new dataset to be unlocked at boot by `zfs-vaultlocker`.
