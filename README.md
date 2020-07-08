ObjectiveFS FlexVolume driver for Kubernetes
============================================

Driver for [ObjectiveFS](https://objectivefs.com/) S3-based filesystems as [Kubernetes volumes](https://kubernetes.io/docs/concepts/storage/volumes/).

Based off the the [CIFS flexvolume driver](https://github.com/fstab/cifs) by [fstab](https://github.com/fstab).

Background
----------

Docker containers running in Kubernetes have an ephemeral file system: Once a container is terminated, all files are gone. In order to store persistent data in Kubernetes, you need to mount a [Persistent Volume](https://kubernetes.io/docs/concepts/storage/volumes/) into your container. Kubernetes has built-in support for network filesystems found in the most common cloud providers, like [Amazon's EBS](https://aws.amazon.com/ebs), [Microsoft's Azure disk](https://azure.microsoft.com/en-us/services/storage/unmanaged-disks/), etc. [ObjectiveFS](https://objectivefs.com/) is a technology that combines the storage power of [Amazon's S3](https://aws.amazon.com/s3/) with local caching to provide fast read/write throughput. However, ObjectiveFS is not natively supported by Kubernetes.

Fortunately, Kubernetes provides [Flexvolume](https://github.com/kubernetes/community/blob/master/contributors/devel/flexvolume.md), which is a plugin mechanism enabling users to write their own drivers.

Installing
----------

The flexvolume plugin is a single shell script named [ofs](https://github.com/altalang/ofs). This shell script must be available on the Kubernetes master and on each of the Kubernetes nodes. By default, Kubernetes searches for third party volume plugins in `/usr/libexec/kubernetes/kubelet-plugins/volume/exec/`. The plugin directory can be configured with the kubelet's `--volume-plugin-dir` parameter, run `ps aux | grep kubelet` to learn the location of the plugin directory on your system. The `ofs` script must be located in a subdirectory named `altalang~ofs/`. The directory name `altalang~ofs/` will be [mapped](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-storage/flexvolume.md#prerequisites) to the Flexvolume driver name `altalang/ofs`.

The `ofs` script requires a few executables to be available on each host system:

* `mount.objectivefs`, provided by the ObjectiveFS package. You can find the appropriate package for your system on the [ObjectiveFS download page](https://objectivefs.com/install).
* `jq`, on Ubuntu this is in the [jq](https://packages.ubuntu.com/bionic/jq) package.
* `mountpoint`, on Ubuntu this is in the [util-linux](https://packages.ubuntu.com/bionic/util-linux) package.
* `base64`, on Ubuntu this is in the [coreutils](https://packages.ubuntu.com/bionic/coreutils) package.

### **Installing in a cluster with `kops`**

If your Kubernetes cluster is managed through `kops`, you will need to configure your [cluster specification](https://github.com/kubernetes/kops/blob/master/docs/cluster_spec.md) with a [hook](https://github.com/kubernetes/kops/blob/master/docs/cluster_spec.md#hooks) to install the flexvolume driver and all appropriate packages on all nodes that it creates.

First, you will need your download path for your ObjectiveFS pacakge. This can be found on the [ObjectiveFS download page](https://objectivefs.com/install).

Edit your cluster configuration with `kops edit cluster <cluster-name>` and add an appropriate hook to install the necessary packages and flexvolume driver. Here is an example for a cluster based on Ubuntu (be sure to change the plugin paths as needed, described above) using ObjectiveFS 6.7.2 (be sure to change the download link to your own download link):

```yaml
spec:
# sections removed
  hooks:
  - before:
    - kubelet.service
    execContainer:
      command:
      - sh
      - -c
      - chroot /rootfs apt-get update && chroot /rootfs apt-get install -y jq fuse
        && chroot /rootfs curl -L -o /usr/libexec/kubernetes/kubelet-plugins/volume/exec/altalang~ofs/ofs
        --create-dirs https://raw.githubusercontent.com/altalang/ofs/master/ofs
        && chroot /rootfs chmod 755 /usr/libexec/kubernetes/kubelet-plugins/volume/exec/altalang~ofs/ofs
        && chroot /rootfs curl -L -O https://objectivefs.com/user/download/<example>/objectivefs_6.7.2_amd64.deb
        && chroot /rootfs dpkg -i objectivefs_6.7.2_amd64.deb
      image: busybox
```

Then update your cluster with `kops update cluster <cluster-name> --yes`, and perform a rolling update with `kops rolling-update cluster <cluster-name> --yes`.

### **Installing manually**

On the Kubernetes master and on each Kubernetes node, run the following commands:

```bash
VOLUME_PLUGIN_DIR="/usr/libexec/kubernetes/kubelet-plugins/volume/exec"
mkdir -p "$VOLUME_PLUGIN_DIR/altalang~ofs"
cd "$VOLUME_PLUGIN_DIR/altalang~ofs"
curl -L -O https://raw.githubusercontent.com/altalang/ofs/master/ofs
chmod 755 ofs
```

You will also need to ensure all the additional dependencies are installed as described above, including `mount.objectivefs`, `jq`, `mountpoint`, and `base64`.

To check if the installation was successful, run the following command:

```bash
VOLUME_PLUGIN_DIR="/usr/libexec/kubernetes/kubelet-plugins/volume/exec"
$VOLUME_PLUGIN_DIR/altalang~ofs/ofs init
```

It should output a JSON string containing `"status": "Success"`. This command is also run by Kubernetes itself when the ofs plugin is detected on the file system.

Running
-------

The plugin takes your AWS Access Key ID, AWS Secret Access Key, ObjectiveFS License, and ObjectiveFS Passphrase from a [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/). To create the secret, you first have to convert each of theve values to base64 encoding:

```bash
echo -n <AWS access key id> | base64
echo -n <AWS secret access key> | base64
echo -n <ObjectiveFS license> | base64
echo -n <ObjectiveFS passphrase> | base64
```

Then, create a file `secret.yml` and use the ouput of the above commands as username and password:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ofs-secret
  namespace: default
data:
  accesskey: 'QVdTIEFjY2VzcyBLZXkgSUQ='
  secretkey: 'QVdTIFNlY3JldCBBY2Nlc3MgS2V5'
  license: 'WW91ciBPYmplY3RpdmVGUyBsaWNlbnNl'
  passphrase: 'cGFzc3BocmFzZQ=='
```

Apply the secret:

```bash
kubectl apply -f secret.yml
```

You can check if the secret was installed successfully using `kubectl describe secret ofs-secret`.

Next, create a file `pod.yml` with a test pod (replace `s3://my-bucket` with the name of the S3 bucket you are using for your ObjectiveFS share):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - name: busybox
    image: busybox
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: test
      mountPath: /data
  volumes:
  - name: test
    flexVolume:
      driver: "altlang/ofs"
      secretRef:
        name: "ofs-secret"
      options:
        networkPath: "s3://my-bucket"
        mountOptions: "mt"
```

Start the pod:

```yaml
kubectl apply -f pod.yml
```

You can verify that the volume was mounted successfully using `kubectl describe pod busybox`.

Testing
-------

If everything is fine, start a shell inside the container to see if it worked:

```bash
kubectl exec -ti busybox /bin/sh
```

Inside the container, you should see the ObjectiveFS share mounted to `/data`.
