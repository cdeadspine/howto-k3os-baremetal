# howto-k3os-baremetal

Either https://github.com/rancher/k3os is not really designed to run on a bare metal cluster, or it is just too bleeding edge right now


## k3os config.yaml
- i dont think mDNS is a part of k3os, so you have to use a hardcoded ip address i think? or else have an external DNS system outside the cluster maybe?
- need to create 2 different installation ISOs to set up the "Embedded HA using etcd" , the other options seem equally strange in that you need an external database not part of the cluster. This is weird to my brain, in that i thought the whole point of a cluster is without single point of failure. but the options are "external database single point of failure" or "i need multiple clusters to make a cluster"
- the first computer to make needs
```
k3os:
  k3s_args:
  - server  
  - "--cluster-init"
  token: token
```
- subsequent computers need a different configuration / installation ISO . This hardcoded ip seems very strange, and i don't entirely know yet if this breaks metalLB when certain/first computer goes down. Also not sure if I need the first master fully running before I can add any more computers to the cluster like to fix a problem? Do I really need to make ANOTHER ISO image (with a new server_url) if the 1/3 computers that experiences a hardware problem happens to be the first master i spun up?
```
k3os:
  k3s_args:
  token: token
  server_url: https://192.168.10.43:6443
```
- metalLB seems like the simplest way to route a tiny corporate router to potentially partial cluster. All config.yaml require
```
k3os:
  k3s_args:
  - server
  - "--disable=servicelb"
```




## Embedded High availabilty using ETCD

Documentation says you need 3 or more, and odd number of servers only. Nothing explains why this is the case. I think it is specifically from the way ETCD works.

From trial and error testing:
### 2 servers reacts like this 
- shutting one down, none of kubectl works
- Can ssh into each cluster computer to see which one seems to be running and which one is not, then troubleshoot any hardware problems on a per computer basis
- Turning second server back on, everything started running again
- pods on each surviving server are "suppose to stay running" (but this didn't seem true, not everything definitely), but anything running on the downed server was not automaticaly scaled/rescheduled
- you can mostly trust that multiple pods (replicaset >= 2) will run on different nodes (it wont blindly run replicate: 2 on the same node when no nodeaffinity is defined)
- metalLB doesnt seem to work when one of 2 servers goes down, so multiple replicas isnt enough

### 3 servers
- mostly works as expected
- 1 down : leaves enough nodes running to keep the website up, metallb works enough to get to the website
- a few seconds of no response from kubectl, eventually works ok
- something feels wrong with setting an ip address inside of /user/.kube/k3s.yaml to connect to cluster, if the first server goes down are we just suppose to manually edit the file to connect to one of the other 2?
- 1 down : eventually it will reschedule some new pods on the 2 remaining nodes
- 2 down seems to completely fail, see how 2 servers react (completely down)


## Uninterruptible Power Supply

I wish I didn't have to think much about this, but I was forced to try and re-invent the wheel here
No operating system support exists yet (hopefully someday) : https://github.com/rancher/k3os/issues/548

Basic operating system support requires addition to the k3os config.yaml
```
write_files:
- encoding: ""
  content: |-
    #!/bin/bash
    DIR=/usr/local/killpower
    LOG=${DIR}/killpower.log
    MARK=${DIR}/killpower
    if test -f "$MARK"; then
      now=$(date +"%Y %m %d %T")
      echo "${now} : killpowerwatch boot found marker, cleared" >> $LOG
      rm $MARK
    fi
    for (( ; ; ))
    do
      if test -f "$MARK"; then
        now=$(date +"%Y %m %d %T")
        echo "${now} : killpowerwatch, found marker, shutting down" >> $LOG
        poweroff &
        break
      fi
      sleep 3
    done
  owner: root
  path: /home/killpowerwatch
  permissions: '0700'
run_cmd:
- "echo launching killpowerwatch"
- "start-stop-daemon --background --start --exec /home/killpowerwatch --make-pidfile --pidfile /run/killpowerwatch.pid"  
```

This is a super naive attempt at being minimal, probably a ton of problems doing it this way. A flag file is created and detected on the hostPath to poweroff the computer.

### network-ups-tools
The meat of the logic was hopefully supplied in the helm chart: 
- https://github.com/k8s-at-home/charts/tree/master/charts/network-ups-tools
- There was a reference to this on artifacthub.io, but it says something about not maintained, even though fresh builds are still being done to the docker container by someone
```
#setup 
    helm repo add k8s-at-home https://k8s-at-home.com/charts/
#download to use values.yaml as example file
    helm pull k8s-at-home/network-ups-tools --untar
#also check their "common values" for all k8s at home charts 
    #   https://github.com/k8s-at-home/charts/blob/master/charts/common/values.yaml 
#see physical configuration https://networkupstools.org/docs/man/usbhid-ups.html

#the master NODE that is physically connected to UPS is marked with a label
    kubectl label nodes k3os-1351 ups-device=true
#test
    kubectl get node --selector='ups-device'

```
- This is hard to find, but I think the actual docker container source is found at https://github.com/k8s-at-home/container-images/tree/main/network-ups-tools
- I don't know how anyone actually got this working, because the docker image wasn't actually running "upsmon" as part of the entrypoint
- pull request created at https://github.com/cdeadspine/container-images 
- docker image that does run upsmon temporarily found at cdeadspine/nut:test
- This setup assumes only 1 UPS and other slaves on the same single UPS
- Master configuration
```
image:
  repository: cdeadspine/nut #check todo waiting pull request k8sathome/network-ups-tools
  tag: test #check todo waiting pull request v2.7.4-2124-g9defa7a9
  pullPolicy: Always #todo IfNotPresent

controllerType: daemonset #https://github.com/k8s-at-home/charts/blob/master/charts/common/values.yaml

affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: ups-device
          operator: Exists

securityContext: #requird to access hardware
  privileged: true

service:
  port:
    name: server
    port: 3493

env: {}
  # TZ: UTC

additionalVolumes:
  - name: ups
    hostPath:
      path: /dev/bus/usb
  - name: kp
    hostPath:
      path: /usr/local/killpower #k3os specific path
additionalVolumeMounts:
  - mountPath: /dev/bus/usb
    name: ups
  - mountPath: /killpower
    name: kp
  
# The main configuration files for network-ups-tools
config:
  # If set to 'values', the configuration will be read from these values.
  # Otherwise you have to mount a volume to /etc/nut containing the configuration files.
  mode: values

  # See https://github.com/networkupstools/nut/tree/master/conf for config sample files
  files:
    nut.conf: |
      MODE=netserver

    upsd.conf: |
      LISTEN 0.0.0.0

    ups.conf: |
      [ups1]
        driver = usbhid-ups
        port = auto
        desc = "apc on usb"
    
    upsd.users: |
      [monmaster]
        actions = SET
        instcmds = ALL
        password  = secret
        upsmon master
      [monslave]
        password  = secret
        upsmon slave

    upsmon.conf: |
      MONITOR ups1@localhost 1 monmaster secret master
      SHUTDOWNCMD "`/usr/local/ups/sbin/upsdrvctl -u root shutdown && touch /killpower/killpower`"
```
- Slave configuration
```
image:
  repository: cdeadspine/nut #check todo waiting pull request k8sathome/network-ups-tools
  tag: test #check todo waiting pull request v2.7.4-2124-g9defa7a9
  pullPolicy: Always #todo IfNotPresent

controllerType: daemonset #https://github.com/k8s-at-home/charts/blob/master/charts/common/values.yaml

affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: ups-device
          operator: DoesNotExist

securityContext: #requird to access /usr/local/killpower
  privileged: true

env: {}
  # TZ: UTC

additionalVolumes:
  - name: kp
    hostPath:
      path: /usr/local/killpower #k3os specific path
additionalVolumeMounts:
  - mountPath: /killpower
    name: kp
  
# The main configuration files for network-ups-tools
config:
  # If set to 'values', the configuration will be read from these values.
  # Otherwise you have to mount a volume to /etc/nut containing the configuration files.
  mode: values

  # See https://github.com/networkupstools/nut/tree/master/conf for config sample files
  files:
    nut.conf: |
      MODE=netclient
      ALLOW_NO_DEVICE=true
      export ALLOW_NO_DEVICE
    
    dummy-ups.dev: | #required if we dont set up a upsd configuration
      #nothing

    upsmon.conf: |
      MONITOR ups1@each-master-ups-network-ups-tools.default.svc.cluster.local:3493 1 monslave secret slave
      SHUTDOWNCMD "touch /killpower/killpower" 
```
- This ends up being a failure, slave shutdown works properly. But then the master pod screws up( 1 of 3 cluster computers running, apparently pods stop working) before it can shut itself and the UPS down. Perhaps just really short timer configuration would work but seems flaky
- The real fix might be some kind of time delayed flag created on the master created during the NOTIFY stage of upsmon shut down? But this feels super hacky and gross