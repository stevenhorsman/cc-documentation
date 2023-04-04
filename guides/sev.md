# SEV Guide

## Creating a CoCo workload using a pre-existing encrypted image on SEV

### Platform Setup

To enable SEV on the host platform, first ensure that it is supported. Then follow these instructions to enable SEV:

[AMD SEV - Prepare Host OS](https://github.com/AMDESE/AMDSEV#prepare-host-os)

### Install sevctl and Export SEV Certificate Chain

[sevctl](https://github.com/virtee/sevctl) is the SEV command line utility and is needed to export the SEV certificate chain.

Follow these steps to install `sevctl`:

* Debian / Ubuntu:

  ```
  # Rust must be installed to build
  curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
  source $HOME/.cargo/env
  sudo apt install -y musl-dev musl-tools

  # Additional packages are required to build
  sudo apt install -y pkg-config libssl-dev

  # Clone the repository
  git clone https://github.com/virtee/sevctl.git

  # Build
  (cd sevctl && cargo build)
  ```

* CentOS / Fedora / RHEL:

  ```
  sudo dnf install sevctl
  ```

> **Note** Due to [this bug](https://bugzilla.redhat.com/show_bug.cgi?id=2037963) on sevctl for RHEL and Fedora you might need to build the tool from sources to pick the fix up.

If using the SEV kata configuration template file, the SEV certificate chain must be placed in `/opt/sev`. Export the SEV certificate chain using the following commands:

```
sudo mkdir -p /opt/sev
sudo ./sevctl/target/debug/sevctl export --full /opt/sev/cert_chain.cert
```

### Setup and Run the simple-kbs

The [simple-kbs](https://github.com/confidential-containers/simple-kbs) is a basic key broker service that hosts secret storage and provides secret release policies configurable by container workload creators or users.

The `simple-kbs` is a prototype implementation under development and is not intended for production use at this time.

For the SEV encrypted image use case, it is required to host the key used to encrypt the container image from the `simple-kbs`.

The CoCo project has created a sample encrypted container image ([encrypted-image-tests](ghcr.io/fitzthum/encrypted-image-tests:encrypted)). This image is encrypted using a key that comes already provisioned inside the `simple-kbs` for ease of testing. No `simple-kbs` policy is required to get things running.

The image encryption key and key for SSH access have been attached to the CoCo sample encrypted container image as docker labels. This image is meant for TEST purposes only as these keys are published publicly. In a production use case, these keys would be generated by the workload administrator and kept secret. For further details, see the section how to [Create an Encrypted Image](#create-an-encrypted-image).

To learn more about creating custom policies, see the section on [Creating a simple-kbs Policy to Verify the SEV Firmware Measurement](#creating-a-simple-kbs-policy-to-verify-the-sev-firmware-measurement).

A KBS is not required to run unencrypted containers.
Instead, disable pre-attestation by editing the Kata config file located at `/opt/confidential-containers/share/defaults/kata-containers/configuration-qemu-sev.toml`.
```
guest_pre_attestation = false
```
Image decryption and signature validation will not work if pre-attestation is disabled.

> **Note** It is not recommended to edit the Kata configuration file manually.
These changes might be overwritten by the operator.

If you are using attestation, you will need to update the above Kata configuration file to point
to the URI of the KBS.

For example, set
`guest_pre_attestation_proxy = "<KBS IP>:44444"`

You will also need to update the Kata configuration to add an extra kernel parameter specifying KBS information.
For example, add `agent.aa_kbc_params=online_sev_kbc::<KBS IP>:44444`
to the `kernel_params` field in the configuration file.

The KBS IP must be accesible from inside the guest. Port 44444 is the default port per the directions below, but it can be configured.

`docker-compose` is required to run the `simple-kbs` and its database in docker containers:

* Debian / Ubuntu:

  ```
  sudo apt install docker-compose
  ```

* CentOS / Fedora / RHEL:

  ```
  sudo dnf install docker-compose-plugin
  ```

Clone the repository for specified tag:
```
simple_kbs_tag="0.1.1"
git clone https://github.com/confidential-containers/simple-kbs.git
(cd simple-kbs && git checkout -b "branch_${simple_kbs_tag}" "${simple_kbs_tag}")
```

Run the service with `docker-compose`:

* Debian / Ubuntu:

  ```
  (cd simple-kbs && sudo docker-compose up -d)
  ```

* CentOS / Fedora / RHEL:

  ```
  (cd simple-kbs && sudo docker compose up -d)
  ```

### Launch the Pod and Verify SEV Encryption

Here is a sample kubernetes service yaml for an encrypted image:

```
kind: Service
apiVersion: v1
metadata:
  name: encrypted-image-tests
spec:
  selector:
    app: encrypted-image-tests
  ports:
  - port: 22
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: encrypted-image-tests
spec:
  selector:
    matchLabels:
      app: encrypted-image-tests
  template:
    metadata:
      labels:
        app: encrypted-image-tests
    spec:
      runtimeClassName: kata-qemu-sev
      containers:
      - name: encrypted-image-tests
        image: ghcr.io/fitzthum/encrypted-image-tests:encrypted
        imagePullPolicy: Always
```

Save this service yaml to a file named `encrypted-image-tests.yaml`. Notice the image URL specified points to the previously described CoCo sample encrypted container image. `kata-qemu-sev` must also be specified as the `runtimeClassName`.

Start the service:

```
kubectl apply -f encrypted-image-tests.yaml
```

Check for pod errors:

```
pod_name=$(kubectl get pod -o wide | grep encrypted-image-tests | awk '{print $1;}')
kubectl describe pod ${pod_name}
```

If there are no errors, a CoCo encrypted container with SEV has been successfully launched!

### Verify SEV Memory Encryption

The container `dmesg` report can be parsed to verify SEV memory encryption.

Get pod IP:

```
pod_ip=$(kubectl get pod -o wide | grep encrypted-image-tests | awk '{print $6;}')
```

Get the CoCo sample encrypted container image SSH access key from docker image label and save it to a file.
Currently the docker client cannot pull encrypted images. We can inspect the unencrypted image instead,
which has the same labels. You could also use `skopeo inspect` to get the labels from the encrypted image.

```
docker pull ghcr.io/fitzthum/encrypted-image-tests:unencrypted
docker inspect ghcr.io/fitzthum/encrypted-image-tests:unencrypted | \
  jq -r '.[0].Config.Labels.ssh_key' \
  | sed "s|\(-----BEGIN OPENSSH PRIVATE KEY-----\)|\1\n|g" \
  | sed "s|\(-----END OPENSSH PRIVATE KEY-----\)|\n\1|g" \
  > encrypted-image-tests
```

Set permissions on the SSH private key file:

```
chmod 600 encrypted-image-tests
```

Run a SSH command to parse the container `dmesg` output for SEV enabled messages:

```
ssh -i encrypted-image-tests \
  -o "StrictHostKeyChecking no" \
  -t root@${pod_ip} \
  'dmesg | grep SEV'
```

The output should look something like this:
```
[    0.150045] Memory Encryption Features active: AMD SEV
```

## Create an Encrypted Image

If SSH access to the container is desired, create a keypair:

```
ssh-keygen -t ed25519 -f encrypted-image-tests -P "" -C "" <<< y
```

The above command will save the keypair in a file named `encrypted-image-tests`.

Here is a sample Dockerfile to create a docker image:

```
FROM alpine:3.16

# Update and install openssh-server
RUN apk update && apk upgrade && apk add openssh-server

# Generate container ssh key
RUN ssh-keygen -t ed25519 -f /etc/ssh/ssh_host_ed25519_key -P ""

# A password needs to be set for login to work. An empty password is
# unproblematic as password-based login to root is not allowed.
RUN passwd -d root

# Copy the remote generated public key to the container authorized_keys
# Generate with 'ssh-keygen -t ed25519 -f encrypted-image-tests -P "" -C ""'
COPY encrypted-image-tests.pub /root/.ssh/authorized_keys

# Entry point - run sshd
ENTRYPOINT /usr/sbin/sshd -D
```

Store this `Dockerfile` in the same directory as the `encrypted-image-tests` ssh keypair.

Build image:

```
docker build -t encrypted-image-tests .
```

Tag and upload this unencrypted docker image to a registry:

```
docker tag encrypted-image-tests:latest [REGISTRY_URL]:unencrypted
docker push [REGISTRY_URL]:unencrypted
```

Be sure to replace `[REGISTRY_URL]` with the desired registry URL.

[skopeo](https://github.com/containers/skopeo) is required to encrypt the container image. Follow the instructions here to install `skopeo`:

[skopeo Installation](https://github.com/containers/skopeo/blob/main/install.md)

The Attestation Agent hosts a grpc service to support encrypting the image. Clone the repository:

```
attestation_agent_tag="v0.1.0"
git clone https://github.com/confidential-containers/attestation-agent.git
(cd simple-kbs && git checkout -b "branch_${attestation_agent_tag}" "${attestation_agent_tag}")
```

Run the offline_fs_kbs:

```
(cd attestation-agent/sample_keyprovider/src/enc_mods/offline_fs_kbs \
&& cargo run --release --features offline_fs_kbs -- --keyprovider_sock 127.0.0.1:50001 &)
```

Create the Attestation Agent keyprovider:

```
cat > attestation-agent/sample_keyprovider/src/enc_mods/offline_fs_kbs/ocicrypt.conf <<EOF
{
  "key-providers": {
    "attestation-agent": {
      "grpc": "127.0.0.1:50001"
}}}
EOF
```

Set a desired value for the encryption key that should be a 32-bytes and base64 encoded value:

```
enc_key="RcHGava52DPvj1uoIk/NVDYlwxi0A6yyIZ8ilhEX3X4="
```

Create a Key file:

```
cat > keys.json <<EOF
{
    "key_id1":"${enc_key}"
}
EOF
```

Run skopeo to encrypt the image created in the previous section:

```
sudo OCICRYPT_KEYPROVIDER_CONFIG=$(pwd)/attestation-agent/sample_keyprovider/src/enc_mods/offline_fs_kbs/ocicrypt.conf \
skopeo copy --insecure-policy \
docker:[REGISTRY_URL]:unencrypted \
docker:[REGISTRY_URL]:encrypted \
--encryption-key provider:attestation-agent:$(pwd)/keys.json:key_id1
```

Again, be sure to replace `[REGISTRY_URL]` with the desired registry URL.
`--insecure-policy` flag is used to connect to the attestation agent and will not impact the security of the project.

Make sure to use the `docker` prefix in the source and destination URL when running the `skopeo copy` command as demonstrated above. 
Utilizing images via the local `docker-daemon` is known to have issues, and the `skopeo copy` command does not return an adequate error 
response. A remote registry known to support encrypted images like GitHub Container Registry (GHCR) is required.

At this point it is a good idea to inspect the image was really encrypted as skopeo can silently leave it unencrypted. Use
`skopeo inspect` as shown below to check that the layers MIME types are **application/vnd.oci.image.layer.v1.tar+gzip+encrypted**:

```
skopeo inspect docker-daemon:[REGISTRY_URL]:encrypted
```

Push the encrypted image to the registry:

```
docker push [REGISTRY_URL]:encrypted
```

`mysql-client` is required to insert the key into the `simple-kbs` database. `jq` is required to json parse responses on the command line.

* Debian / Ubuntu:

  ```
  sudo apt install mysql-client jq
  ```

* CentOS / Fedora / RHEL:

  ```
  sudo dnf install [ mysql | mariadb | community-mysql ] jq
  ```

The `mysql-client` package name may differ depending on OS flavor and version.

The `simple-kbs` uses default settings and credentials for the MySQL database. These settings can be changed by the `simple-kbs` administrator and saved into a credential file. For the purposes of this quick start, set them in the environment for use with the MySQL client command line:

```
KBS_DB_USER="kbsuser"
KBS_DB_PW="kbspassword"
KBS_DB="simple_kbs"
KBS_DB_TYPE="mysql"
```

Retrieve the host address of the MySQL database container:

```
KBS_DB_HOST=$(docker network inspect simple-kbs_default \
  | jq -r '.[].Containers[] | select(.Name | test("simple-kbs[_-]db.*")).IPv4Address' \
  | sed "s|/.*$||g")
```

Add the key to the `simple-kbs` database without any verification policy:

```
mysql -u${KBS_DB_USER} -p${KBS_DB_PW} -h ${KBS_DB_HOST} -D ${KBS_DB} <<EOF
  REPLACE INTO secrets VALUES (10, 'key_id1', '${enc_key}', NULL);
  REPLACE INTO keysets VALUES (10, 'KEYSET-1', '["key_id1"]', NULL);
EOF
```

The second value in the keysets table (`KEYSET-1`) must match the `guest_pre_attestation_keyset` value specified in the SEV kata configuration file located here:

`/opt/confidential-containers/share/defaults/kata-containers/configuration-qemu-sev.toml`

Return to step [Launch the Pod and Verify SEV Encryption](#launch-the-pod-and-verify-sev-encryption) and finish the remaining process. Make sure to change the `encrypted-image-tests.yaml` to reflect the new `[REGISTRY_URL]`.

To learn more about creating custom policies, see the section on [Creating a simple-kbs Policy to Verify the SEV Firmware Measurement](#creating-a-simple-kbs-policy-to-verify-the-sev-guest-firmware-measurement).


## Creating a simple-kbs Policy to Verify the SEV Guest Firmware Measurement

The `simple-kbs` can be configured with a policy that requires the kata shim to provide a matching SEV guest firmware measurement to release the key for decrypting the image. At launch time, the kata shim will collect the SEV guest firmware measurement and forward it in a key request to the `simple-kbs`.

These steps will use the CoCo sample encrypted container image, but the image URL can be replaced with a user created image registry URL.

To create the policy, the value of the SEV guest firmware measurement must be calculated. 

`pip` is required to install the `sev-snp-measure` utility.

* Debian / Ubuntu:

  ```
  sudo apt install python3-pip
  ```

* CentOS / Fedora / RHEL:

  ```
  sudo dnf install python3
  ```

[sev-snp-measure](https://github.com/IBM/sev-snp-measure) is a utility used to calculate the SEV guest firmware measurement with provided ovmf, initrd, kernel and kernel append input parameters. Install it using the following command:

```
sudo pip install sev-snp-measure
```

The path to the guest binaries required for measurement is specified in the kata configuration. Set them:

```
ovmf_path="/opt/confidential-containers/share/ovmf/OVMF.fd"
kernel_path="/opt/confidential-containers/share/kata-containers/vmlinuz-sev.container"
initrd_path="/opt/confidential-containers/share/kata-containers/kata-containers-initrd.img"
```

The kernel append line parameters are included in the SEV guest firmware measurement. A placeholder will be initially set, and the actual value will be retrieved later from the qemu command line:

```
append="PLACEHOLDER"
```

Use the `sev-snp-measure` utility to calculate the SEV guest firmware measurement using the binary variables previously set:

```
measurement=$(sev-snp-measure --mode=sev --output-format=base64 \
  --ovmf="${ovmf_path}" \
  --kernel="${kernel_path}" \
  --initrd="${initrd_path}" \
  --append="${append}" \
)
```

If the container image is not already present, pull it:

```
encrypted_image_url="ghcr.io/fitzthum/encrypted-image-tests:unencrypted"
docker pull "${encrypted_image_url}"
```

Retrieve the encryption key from docker image label:

```
enc_key=$(docker inspect ${encrypted_image_url} \
  | jq -r '.[0].Config.Labels.enc_key')
```

Add the key, keyset and policy with measurement to the `simple-kbs` database:

```
mysql -u${KBS_DB_USER} -p${KBS_DB_PW} -h ${KBS_DB_HOST} -D ${KBS_DB} <<EOF
  REPLACE INTO secrets VALUES (10, 'key_id1', '${enc_key}', 10);
  REPLACE INTO keysets VALUES (10, 'KEYSET-1', '["key_id1"]', 10);
  REPLACE INTO policy VALUES (10, '["${measurement}"]', '[]', 0, 0, '[]', now(), NULL, 1);
EOF
```

Using the same service yaml from the section on [Launch the Pod and Verify SEV Encryption](#launch-the-pod-and-verify-sev-encryption), launch the service:

```
kubectl apply -f encrypted-image-tests.yaml
```

Check for pod errors:

```
pod_name=$(kubectl get pod -o wide | grep encrypted-image-tests | awk '{print $1;}')
kubectl describe pod ${pod_name}
```

The pod will error out on the key retrieval request to the `simple-kbs` because the policy verification failed due to a mismatch in the SEV guest firmware measurement. This is the error message that should display:

```
Policy validation failed: fw digest not valid
```

The `PLACEHOLDER` value that was set for the kernel append line when the SEV guest firmware measurement was calculated does not match what was measured by the kata shim. The kernel append line parameters can be retrieved from the qemu command line using the following scripting commands, as long as kubernetes is still trying to launch the pod:

```
duration=$((SECONDS+30))
set append

while [ $SECONDS -lt $duration ]; do
  qemu_process=$(ps aux | grep qemu | grep append || true)
  if [ -n "${qemu_process}" ]; then
    append=$(echo ${qemu_process} \
      | sed "s|.*-append \(.*$\)|\1|g" \
      | sed "s| -.*$||")
    break
  fi
  sleep 1
done

echo "${append}"
```

The above check will only work if the `encrypted-image-tests` guest launch is the only consuming qemu process running.

Now, recalculate the SEV guest firmware measurement and store the `simple-kbs` policy in the database:

```
measurement=$(sev-snp-measure --mode=sev --output-format=base64 \
  --ovmf="${ovmf_path}" \
  --kernel="${kernel_path}" \
  --initrd="${initrd_path}" \
  --append="${append}" \
)

mysql -u${KBS_DB_USER} -p${KBS_DB_PW} -h ${KBS_DB_HOST} -D ${KBS_DB} <<EOF
  REPLACE INTO secrets VALUES (10, 'key_id1', '${enc_key}', 10);
  REPLACE INTO keysets VALUES (10, 'KEYSET-1', '["key_id1"]', 10);
  REPLACE INTO policy VALUES (10, '["${measurement}"]', '[]', 0, 0, '[]', now(), NULL, 1);
EOF
```

The pod should now show a successful launch:

```
kubectl describe pod ${pod_name}
```

If the service is hung up, delete the pod and try to launch again:

```
# Delete
kubectl delete -f encrypted-image-tests.yaml

# Verify pod cleaned up
kubectl describe pod ${pod_name}

# Relaunch
kubectl apply -f encrypted-image-tests.yaml
```

Testing the SEV encrypted container launch can be completed by returning to the section on how to [Verify SEV Memory Encryption](#verify-sev-memory-encryption).