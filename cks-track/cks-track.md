[DRAFT]
Hello K8s community members! We at Spectro Cloud are excited to share a a tutorial following our CKS webinar (if you haven't watched it, here is the [link to the recording](https://www.spectrocloud.com/webinars/certified-kubernetes-security-specialist/)). Some of aspects of this tutorial is modeled after the webinar content, but there are new contents as well.

# General Setup

Here are a series of steps to prepare for the rest of the tutorial.

Note that if you're using one of the managed Kubernetes services (e.g. EKS, AKS, GKE), you will not have access to the master nodes of your cluster and you can’t try on the master nodes.


* Create your cluster with at least one worker node. For convenience, feel free to sign up [Spectro Cloud Palette](https://www.spectrocloud.com/free-trial/) and use the `CKS Tool Box` cluster profile.
* After your cluster is launched, create a pod to exercise the CKS scenarios, using the following command:
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu18
  labels:
    app: ubuntu18
spec:
  hostPID: true
  volumes:
  - name: sec-ctx-vol
    emptyDir: {}
  containers:
  - name: ubuntu18
    image: ubuntu:18.04
    command: ["/bin/sleep", "3650d"]
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: sec-ctx-vol
      mountPath: /
  restartPolicy: Always
EOF
```

## APP ARMOR

App Armor is a security enhancement to Linux kernel that can limit software programs to a set of resources.

### Steps

* `kubectl exec -it ubuntu18 -- bash   # get into the pod`
* `cd /host; chroot .; bash # chroot to host filesystem`
* `apt update && apt install apparmor-utils # update the utilties for below use`

The following should output "Y" to indicate enablement:
* `cat /sys/module/apparmor/parameters/enabled # boolean flag`

```
sudo apparmor_parser -r -q <<EOF
#include <tunables/global>

profile k8s-apparmor-example-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>

  file,

  # Deny all file writes.
  deny /** w,
}
EOF
```
* `sudo apparmor_status # new policy will show up`
* Next is applying the app armor profile to a test pod
* `kubectl apply -f tools/apparmor/pod.yaml # cat tools/apparmor/pod.yaml`
* by now all file writes will be blocked to the pod
* `kubectl exec hello-apparmor -- touch /tmp/test # this will fail due to apparmor policy`

Feel free to research more how App Armor can harden your pods.

## TRIVY

[Trivy](https://aquasecurity.github.io/trivy/) is a vulnerability scanner for containers. Trivy detects vulnerabilities in OS as well as package independencies.

For ubuntu here are the instructions:
```
wget https://github.com/aquasecurity/trivy/releases/download/v0.24.1/trivy_0.24.1_Linux-64bit.deb
sudo dpkg -i ./trivy_0.24.1_Linux-64bit.deb 
```
Installation instructions for other Linux flavors can be found [here](
https://aquasecurity.github.io/trivy/v0.24.1/getting-started/installation/).

Scan images (the process will includ image download):
`sudo trivy image nginx # image-name `

The output will be quite extensive and long
```

+------------------+------------------+----------+--------------------+---------------+-----------------------------------------+
|     LIBRARY      | VULNERABILITY ID | SEVERITY | INSTALLED VERSION  | FIXED VERSION |                  TITLE                  |
+------------------+------------------+----------+--------------------+---------------+-----------------------------------------+
| apt              | CVE-2011-3374    | LOW      | 2.2.4              |               | It was found that apt-key in apt,       |
|                  |                  |          |                    |               | all versions, do not correctly...       |
|                  |                  |          |                    |               | -->avd.aquasec.com/nvd/cve-2011-3374    |
+------------------+------------------+          +--------------------+---------------+-----------------------------------------+
...
```

To look for CRITICAL items use the following flag:
`sudo trivy image --severity CRITICAL nginx`

Contrast this with output of the Alpine version (smaller footprint as well as scrubbed regularyly):
`sudo trivy image nginx:alpine`

One may also scan configuration directory:
`sudo trivy conf --severity MEDIUM /etc/kubernetes # do this on control plane`

## KUBE-BENCH

[kube-bench](https://github.com/aquasecurity/kube-bench) can check whether kubernetes is deployed securely accordingly CNCF standards.

```
curl -L https://github.com/aquasecurity/kube-bench/releases/download/v0.6.6/kube-bench_0.6.6_linux_amd64.deb -o kube-bench_0.6.6_linux_amd64.deb # let's stick with at least 0.6.6 as there could be installation issue such as 0.6.2
sudo dpkg -i ./kube-bench_0.6.6_linux_amd64.deb
```

Let's do a scan on the master node:
`kube-bench run --targets=master`

```
== Summary master ==
50 checks PASS
3 checks FAIL
12 checks WARN
0 checks INFO

== Summary total ==
50 checks PASS
3 checks FAIL
12 checks WARN
0 checks INFO
```

And do the same on the woker node
`kube-bench run --targets=node`

Check the result and fixing failures are left as an exercise to the readers :)
