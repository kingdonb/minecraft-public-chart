Based heavily on:

* [gilesknap/k3s-minecraft](https://github.com/gilesknap/k3s-minecraft#intro)
* [itzg/minecraft-server-charts](https://github.com/itzg/minecraft-server-charts)

This chart has a few small alterations compared with upstream:

* We do not call the `volume.beta.kubernetes.io/storage-class` annotation at all, as it is deprecated in favor of `storageClassName` now, and it was causing some errors about immutability on Sync in our cluster.
* I have enabled `--bonusChest`
* I have enabled some ops by default

Feel free to override these as in `minecraft-helm.yaml` from the [k3s-minecraft guide](https://github.com/gilesknap/k3s-minecraft#deploy-a-minecraft-server-to-your-cluster), adjusting these values to suit your own needs.

A sample of my own configuration (remember to replace `<YOUR_RCON_PWD>` with your own password):

```bash
helm upgrade --install my-first-minecraft -f minecraft-helm.yaml \
  --set minecraftServer.eula=true \
  --set rcon.password=<YOUR_RCON_PWD> \
  --set persistence.storageClass=local-path ./minecraft-public-chart/charts/minecraft
```

```yaml
# These are value overrides for the helm chart defined here
# https://github.com/itzg/minecraft-server-charts

# It contains the most common parameters, but for even more settings, see
# https://github.com/itzg/minecraft-server-charts/blob/master/charts/minecraft/values.yaml

# to use this file execute the following two commands
# helm repo add minecraft-server-charts https://itzg.github.io/minecraft-server-charts/
# helm upgrade --install RELEASE_NAME -f THIS_FILE.yaml --set minecraftServer.eula=true,rcon.password=YOUR_PWD minecraft-server-charts/minecraft

# Common minecraft server settings
minecraftServer:
  # may be SNAPSHOT or a specific version number
  version: LATEST
  # allowed players' names comma separated, leave blank to allow any players
  whitelist:
  # comma separated admin players' names
  ops: kingdonb,juicegg2077
  # always set game mode when a player connects
  forcegameMode: true
  # choose a seed for the starting world
  levelSeed:
  # size of the world in blocks radius from spawn point (default is 29999984)
  maxWorldSize: 10000
  # default game mode for players creative,adventure,survival or spectator
  gameMode: creative
  # message in the launcher
  motd: "Welcome to Minecraft on Kubernetes"
  # player vs player enabled
  pvp: false
  # REQUIRED this means you get nodePort exposed on all nodes and will get
  # directed to the correct node if you connect to any
  serviceType: LoadBalancer
  loadBalancerIP: 10.17.12.203
  # restore the world from a backup on first execution (requires NFS volume - see below)
  downloadWorldUrl:
  # external port must be unique across all mc server instances on a given node
  # nodePort: 30100
  # how far from the player are chunks rendered on the server
  # 10 is default, larger is nice on high spec hardware
  # I believe 25 is the client max so larger is wasted
  viewDistance: 10
  # memory to allocate to the jvm - if you change this also change the K8s resource request
  memory: 2g

  rcon:
    # enable rcon for remote control - set to false if not required
    enabled: true
    # external port must be unique across all mc server instances
    # nodePort: 30101
    # required
    serviceType: LoadBalancer
    loadBalancerIP: 10.17.12.204
    password: "TODO - supply this as an override on the command line "

# UNCOMMENT and supply an NFS mounted folder if you have backups of minecraft server
# worlds that you would like to use as the starting point for the new server
# extraVolumes:
#   # define where the volume mounts in the container
#   - volumeMounts:
#       - name: bigdisk
#         mountPath: /share/bigdisk
#         readOnly: true
#     # define the remote nfs mount to attach to above
#     volumes:
#       - name: bigdisk
#         nfs:
#           server: 192.168.86.32
#           path: /share/bigdisk

extraEnv:
  # recommended rolling logs for saving disk
  ENABLE_ROLLING_LOGS: true
  # if this is true then minecraftServer properties above are always applied
  # on every restart, if false they only apply at first run
  OVERRIDE_SERVER_PROPERTIES: true
  # if this is true then downloadWorldUrl is always loaded on restart
  # useful for resetting the world but should be reset to false afterwards
  # WARNING: setting this to true deletes your previous world data
  FORCE_WORLD_COPY: false

# required to save the world in persistent local-path storage (k3s default Storage Class)
# by default this goes in /var/lib/rancher/k3s/storage on the executing node.
# NOTE this ties the deployment to the first node it spins up on,
# we could perhaps use an NFS mount to avoid this
persistence:
  annotations: {}
  dataDir:
    enabled: true
    Size: 1Gi

# K8S resources to grant the pod. memory must be the same as java memory above
# I've also found that limit should be a little higher.
# For processors - if you want to run lots of servers but only use one or two at
# a time then it is OK to set a low request and a higher limit. Minecraft servers
# can make use of quite a few cpus if available and this is especially helpful
# at startup time
resources:
  requests:
    memory: 2Gi
    cpu: 1
  limits:
    memory: 3Gi
    cpu: 6

# UNCOMMENT to choose a single node to run on
nodeSelector:
  kubernetes.io/hostname: hpworker01

# UNCOMMENT to choose a range of nodes to run on
# (change 'key' for other criteria e.g. processor architecture 
#  use 'kubectl get nodes --show-labels' for full set of criteria )
# affinity:
#   nodeAffinity:
#     preferredDuringSchedulingIgnoredDuringExecution:
#     - weight: 1
#       preference:
#         matchExpressions:
#         - key: kubernetes.io/hostname
#           operator: In
#           values:
#           - pi2
#           - pi3
#           - pi4
```
