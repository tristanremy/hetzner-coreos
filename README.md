# hetzner-coreos
Fedora CoreOS as a snapshot in Hetzner Cloud by major.io (updated)

Wanted a quick way to deploy Fedora CoreOS in Hetzner cloud, as they don't have CoreOS in the cloud base images.
Found this solution : [Deploy Fedora CoreOS in Hetzner cloud](https://major.io/p/deploy-fedora-coreos-in-hetzner-cloud/) by [Major Hayden](https://major.io/)

Create the butane configuration :

```yaml
# hetzner-coreos.butane
variant: fcos
version: 1.5.0
passwd:
  users:
    - name: core
      groups:
        - wheel
      ssh_authorized_keys:
        - [MY_RSA_KEY]
```

Compile into and ignition file:
```bash
butane hetzner-coreos.butane > config.ign
```

Create the deployer server (to be snapshot)
```bash
hcloud server create --datacenter nbg1-dc3 --image fedora-39 \
    --type cx11 --name coreos-deployer
```

Put it in rescue mode to be able to replace the entire root disk of the instance
```bash
hcloud server enable-rescue coreos-deployer
```

Reboot :
```bash
hcloud server reboot coreos-deployer
```

SSH into it (using the rescue IP) :
```bash
ssh root@[MY_SERVER_RESCUE_IP]
```

Stream the filesystem from Fedora CoreOS repo to the disk :
```bash
    export COREOS_DISK="https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/39.20240112.3.0/x86_64/fedora-coreos-39.20240112.3.0-metal.x86_64.raw.xz"
curl -sL $COREOS_DISK | xz -d | dd of=/dev/sda status=progress
```


```bash
fdisk -l /dev/sda
mount /dev/sda3 /mnt
mkdir /mnt/ignition
vi /mnt/ignition/config.ign
# paste the ignition file from earlier
umount /mnt
poweroff
````

Save as snapshot in your hetzner cloud account :
```bash
hcloud server create-image --description fedora-39-coreos-cx11 \
    --type snapshot coreos-deployer
```

Get the snapshot id :
```bash
hcloud image list | grep fedora-39-coreos-cx11
```

Anytime you'll want to deploy a new server, you'll need to create a new server with the same image id :
```bash
hcloud server create --datacenter nbg1-dc3 --image [MY_IMAGE_ID] --type cx11 \
    --ssh-key mmtr --name [MY_SERVER_NAME]
```    