layout: page
title: "System Upgrade Controller"

## System Upgrade Controller

### Introduction
This project aims to provide a general-purpose, Kubernetes-native upgrade controller (for nodes). It introduces a new CRD, the Plan, for defining any and all of your upgrade policies/requirements. A Plan is an outstanding intent to mutate nodes in your cluster

### Github
https://github.com/rancher/system-upgrade-controller

### Installation (with kustomize)
```
kustomize build github.com/rancher/system-upgrade-controller | kubectl apply -f - 
```

### Example Upgrade OpenSUSE Leap Controlplane nodes.
```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: leap-update-script
  namespace: system-upgrade
type: Opaque
stringData:
  update.sh: |
    #!/bin/bash
    set -e
    zypper up -y
    # It is important to check if reboot if needed otherwise you will get in a reboot loop.
    zypper needs-rebooting; REBOOT=$?
    zypper ps | grep "You may wish to restart these processes"; REBOOT_PS=$? 
    if [ "$REBOOT" == "1" ] || [ "$REBOOT_PS" == "1" ]; then
      reboot
    fi
---
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: leap-update
  namespace: system-upgrade
spec:
  concurrency: 1
  nodeSelector:
    matchExpressions:
      - {key: node-role.kubernetes.io/control-plane, operator: Exists}
  tolerations:
    - {key: node-role.kubernetes.io/control-plane, effect: NoSchedule, operator: Exists}
    - {key: node-role.kubernetes.io/etcd, effect: NoExecute, operator: Exists}
  serviceAccountName: system-upgrade
  secrets:
    - name: leap-update-script
      path: /host/run/system-upgrade/secrets/leap-update-script
  drain:
    force: true
  version: "1"
  upgrade:
    image: registry.opensuse.org/opensuse/leap:latest
    command: ["chroot", "/host"]
    args: ["sh", "/run/system-upgrade/secrets/leap-update-script/update.sh"]
```

### Example Upgrade Ubuntu Worker Node
```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: ubuntu-update-script
  namespace: system-upgrade
type: Opaque
stringData:
  update.sh: |
    #!/bin/sh
    set -e
    apt-get update
    apt-get upgrade -y
    # It is important to check if reboot if needed otherwise you will get in a reboot loop.
    if [ -f /var/run/reboot-required ]; then
      reboot
    fi
---
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: ubuntu-update
  namespace: system-upgrade
spec:
  concurrency: 1
  nodeSelector:
    matchExpressions:
      - {key: node-role.kubernetes.io/worker, operator: Exists}
  serviceAccountName: system-upgrade
  secrets:
    - name: ubuntu-update-script
      path: /host/run/system-upgrade/secrets/ubuntu-update-script
  drain:
    force: true
  version: focal
  upgrade:
    image: ubuntu
    command: ["chroot", "/host"]
    args: ["sh", "/run/system-upgrade/secrets/ubuntu-update-script/update.sh"]
```
