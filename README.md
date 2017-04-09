# Vagrant Consul Cluster

A 5-node [Consul][a01] cluster with 2-node compute resources to run services on to demonstrate how Consul works, and to learn about how it operates.

## Prerequisites

1. An ISO creation utility (`sudo apt-get install genisoimage` on Debian/Ubuntu or `brew install cdrtools` on Mac with [Homebrew][a12]) so that we can use [Cloud Init][a11] to provision the virtual machines.
1. [Vagrant][a13] 1.9.1 or newer.
1. [VirtualBox][a14] 5.1.x.  Older versions may work, but I have not tested it.

## Cluster Architecture

The `Vagrantfile` is set up to create 8 hosts of various types as described below.

| Hostname | Description |
|----------|-------------|
| `bootstrap` | This is the first Consul server that is started in bootstrap mode to expect 2 more Consul servers to join the server cluster.  Its sole purpose is to bootstrap the Consul cluster, after which it can be destroyed. |
| `consul-0[1-5]` | This is a 5-node Consul server cluster that is bootstrapped by the bootstrap host, and has quorum as a Consul cluster even after the `bootstrap` host is destroyed. |
| `compute-0[1-2]` | A 2-node compute cluster, each running a Consul client, Docker and [Registrator][a06] to update the location of services running on them. |

The diagram below shows the setup from the server level.

![Consul cluster architecture](consul-cluster-architecture.png "Figure 1: Consul cluster architecture")

## Usage

### Cluster Creation

```
# Create the first Consul server to bootstrap the Consul cluster
vagrant up bootstrap

# Create the rest of the Consul servers
vagrant up /consul-0[1-5]/

# As the rest of the Consul servers come online, open up another terminal
# and log on to the bootstrap server to see the Consul cluster converge.
vagrant ssh bootstrap

# On the Consul bootstrap server, see the membership list grow by running
# "consul members" repeatedly.
consul members

# OPTIONAL: Destroy the bootstrap Consul server (since it's outlived it usefulness in this setup)
vagrant destroy -f bootstrap

# Create compute VMs
vagrant up /compute-0[1-2]/
```

### Registering Services

```
# Log on to the compute-01 node
vagrant ssh compute-01

# Run the Hello World container
docker run \
  --detach \
  --publish 80 \
  --env SERVICE_80_NAME=hello-world \
  --env SERVICE_80_CHECK_SCRIPT='curl --silent --fail http://0.0.0.0:$SERVICE_PORT/health' \
  --env SERVICE_80_CHECK_INTERVAL=5s \
  --env SERVICE_80_CHECK_TIMEOUT=3s \
  --env SERVICE_TAGS=http \
  infrastructureascode/hello-world

# Look up its Consul DNS name with the SRV record
dig @0.0.0.0 -p 8600 -t SRV hello-world.service.dc1.consul

# Log on to the compute-02 node
vagrant ssh compute-02

# Run another copy of the Hello World container
docker run \
  --detach \
  --publish 80 \
  --env SERVICE_80_NAME=hello-world \
  --env SERVICE_80_CHECK_SCRIPT='curl --silent --fail http://0.0.0.0:$SERVICE_PORT/health' \
  --env SERVICE_80_CHECK_INTERVAL=5s \
  --env SERVICE_80_CHECK_TIMEOUT=3s \
  --env SERVICE_TAGS=http \
  infrastructureascode/hello-world

# Look up its Consul DNS again and see the SRV record resolve to two instances,
# with one instance on each compute node.
dig @0.0.0.0 -p 8600 -t SRV hello-world.service.dc1.consul
```

### Simulating Consul Outages

```
# Destroy any Consul server, or the Consul leader, and watch the cluster react.
vagrant destroy consul-03
```

## FAQ

### What does the bootstrap server do, and why can it be destroyed after the rest of the Consul servers come online?

The Consul start-up command is hard-coded to [bootstrap][a08] the Consul cluster, while the rest of the 5-node Consul servers are told to [join][a09] an agent in an existing Consul cluster.  Since the bootstrap server is hard-coded to bootstrap, it has outlived its function after the bootstrap process unless its hard-coded command is updated.  However, since I try to build only [immutable infrastructure][a10], updating the command on the bootstrap host would be less than ideal, so I just destroy it instead since the rest of the servers are already bootstrapped, and can come and go without the operations of the cluster getting impacted as long as we maintain a quorum online.

### Why do you list a bunch of IP addresses in the `-join` parameter for the Consul servers in `user-data.consul-server.txt`?

The expectation here is that all Consul servers will have their IP addresses allocated in a predetermined IP block, or will have static IP addresses.  By listing every possible IP allocated for Consul servers in a data center, we can avoid having to update this list as servers come and go over time.  Again, in the spirit of building [immutable infrastructure][a10], we do not update the list once the server is created.

### Why do you use [Cloud Init][a11] to build the servers when there are already recipes for doing so that you can reuse?

Everyone has their favorite provision tool, and instead of forcing anyone to install Ansible, Puppet or Chef, going with Cloud Init avoids adding dependencies that not everyone is comfortable with, and it allows me to use the same Cloud Init configuration (with minimal changes) in AWS if the need arises.

### Should I use this setup in my production environment?

Each production environment is slightly different since needs vary widely.  This Vagrant setup is created as a means to create aConsul cluster that can be used as a starting point to learn about how to operate such a cluster, and may or may not be appropriate for your production environment as is.  Make sure you factor in your needs!


## References

1. [Consul documentation][a01]
1. [Wicked Awesome Tech: Setting up Consul Service Discovery for Mesos in 10 Minutes][a02]
1. [Get Docker for Ubuntu][a03]
1. [kelseyhightower/setup-network-environment][a04]
1. [AWS Compute Blog: Service Discovery via Consul with Amazon ECS][a05]
1. [gliderlabs/registrator][a06]
1. [Sreenivas Makam's Blog: Service Discovery with Consul][a07]
1. [tomkins/cloud-init-vagrant][a15]

[a01]: https://www.consul.io/
[a02]: http://www.wickedawesometech.us/2016/04/setting-up-consul-service-discovery-in.html
[a03]: https://docs.docker.com/engine/installation/linux/ubuntu/
[a04]: https://github.com/kelseyhightower/setup-network-environment
[a05]: https://aws.amazon.com/blogs/compute/service-discovery-via-consul-with-amazon-ecs/
[a06]: http://gliderlabs.com/registrator/latest/
[a07]: https://sreeninet.wordpress.com/2016/04/17/service-discovery-with-consul/
[a08]: https://www.consul.io/docs/guides/bootstrapping.html
[a09]: https://www.consul.io/docs/agent/options.html#_join
[a10]: https://www.oreilly.com/ideas/an-introduction-to-immutable-infrastructure
[a11]: https://cloudinit.readthedocs.io/en/latest/
[a12]: https://brew.sh/
[a13]: https://www.vagrantup.com/
[a14]: https://www.virtualbox.org/
[a15]: https://github.com/tomkins/cloud-init-vagrant
