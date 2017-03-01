# weave-guide
Guide for installing Weave onto DC/OS - Experimental

REVISION 2-28-17	

##OVERVIEW

This is how to install weave on DC/OS 1.8. The install directions differ for the DC/OS masters and agents. Weave is preconfigured via systemd to use DC/OS's DNS to reach seed nodes, which are the weave instances running on the DC/OS master(s). Weave on the DC/OS agents is configured via systemd to be a weave dynamic peer (to use the --ipalloc-init=observer option), which allows DC/OS agent nodes to be transient and to not affect weave's mesh architecture. These directions do not work on coreos, and have only been tested on centos 7.2

weave-dns is not utilized in this configuration, since it currently conflicts with DC/OS's built-in DNS. However for the use case this was deleloped for, services discovery via DNS is not necessary, only IP connectivity. 

Each container must use host mode networking, and will be multi-homed on DC/OS, thus both DC/OS's network and the weave network interfaces will exist within a container. To have a container communicate with another container via  weave only, the executables must be set to bind to only the weave network. For example, a database needs to be configured to only use the weave network in the container, but an app/web tier container would use both; DC/OS's network would be used for the north/south traffic into and out of the cluster, such as to the web service, and weave would be used for the web/app to communicate with the database tier. 

Weaves encryption has not been enabled yet in this configuration.  

##1. INSTALL ON EACH DC/OS MASTER NODE

Login as root and download the systemd unit:

curl -f -o /etc/systemd/system/weave-master.service -O https://raw.githubusercontent.com/mesosphere/weave-guide/master/weave-master.service

systemctl daemon-reload && systemctl enable weave-master

Download weave:

curl -L git.io/weave -o /usr/local/bin/weave

chmod a+x /usr/local/bin/weave

For DC/OS CNI (container network interface) will be setup for weave, but will not be used at this time:

mkdir -p /etc/cni/net.d

Start weave via systemd:

systemctl start weave-master

Run weave setup. It will create a basic weave config file of /etc/cni/net.d/10-weave.conf  that we don’t put to use, the systemd unit contains the configuration. It will also copy weave files into /opt.cni/bin

weave setup

Weave will have added a systemd unit, but we don't need it since this systemd unit runs the weave proxy:

systemctl disable weaveproxy

Setup DC/OS for weave’s CNI module:

ln /etc/cni/net.d/10-weave.conf /opt/mesosphere/etc/dcos/network/cni/weave.conf
cp /opt/cni/bin/* /opt/mesosphere/active/cni 

Check status:

weave status

##2. INITIALIZE WEAVE ON LAST DC/OS MASTER

Now that #1 has been accomplished on each DC/OS master node, it's time to initialize weave. On the last master run this command:

weave prime

##3. INSTALL ON EACH DC/OS AGENT NODE

Login as root and download the systemd unit:

curl -f -o /etc/systemd/system/weave-agent.service -O https://raw.githubusercontent.com/mesosphere/weave-guide/master/weave-agent.service

systemctl daemon-reload && systemctl enable weave-agent

Download weave:

curl -L git.io/weave -o /usr/local/bin/weave

chmod a+x /usr/local/bin/weave

For DC/OS CNI (container network interface) will be setup for weave, but will not be used at this time:

mkdir -p /etc/cni/net.d

Start weave via systemd:

systemctl start weave-agent

Run weave setup. It will create a basic weave config file of /etc/cni/net.d/10-weave.conf  that we don’t put to use, the systemd unit contains the configuration. It will also copy weave files into /opt.cni/bin

weave setup

Weave will have added a systemd unit, but we don't need it since this systemd unit runs the weave proxy:

systemctl disable weaveproxy

Setup DC/OS for weave’s CNI module:

ln /etc/cni/net.d/10-weave.conf /opt/mesosphere/etc/dcos/network/cni/weave.conf
cp /opt/cni/bin/* /opt/mesosphere/active/cni

Check weave status:

weave status


##4. SETUP DC/OS CONTAINER TO USE WEAVE

To enable a container, in the DC/OS app defiinition set the container to use host networking. Also add a docker parameter (in Container Settings screen) of 'net' with a value of 'weave'. The container will now be dual homed.

