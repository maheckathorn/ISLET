# Security Recommendations

Once ISET is installed, users may want to consider reconfiguring their systems to enhance security.  The list below is for manually configuring security controls and documenting recommendations.
Most of these are satisified by `make` targets.

#### SSH: _/etc/ssh/sshd_config_

The following command will configure sshd_config to match the next example, with the exception of modifying LoginGraceTime.

```shell
make security-config
```

```shell
LoginGraceTime 30s
ClientAliveInterval 15
ClientAliveCountMax 10

#Subsystem       sftp    /usr/libexec/openssh/sftp-server

Match User training
	ForceCommand /opt/islet/bin/islet_shell
	X11Forwarding no
	AllowTcpForwarding no
	PermitTunnel no
	PermitOpen none
	MaxAuthTries 3
	MaxSessions 1
	AllowAgentForwarding no
	PermitEmptyPasswords no
```

#### Drop capabilities in containers

ISLET includes security options in `$CONFIG_DIR/modules/docker.conf`, which can be used to easily add or drop kernel capabilities(7)
(and apply ulimit values to containers) to all Docker environments. They can be set in any ISLET configuration file.
Only add what you need to run the software in the container.

#### System limitation contraints

**Note:** ISLET with Docker 1.6 supports passing ulimit settings in config files.

The following command will configure decent ulimit settings for docker processes.
These have the effect of restricting the user's environment inside the container.

Adjust as necessary: _/etc/init/docker.conf_
```
# BEGIN ISLET Additions
limit nofile 1000 2000		 # Limit number of open files
limit nproc  1000 2000		 # Prevent fork bombs
limit fsize  100000000 200000000 # Limit file sizes to max of 200MB
# END
```

#### Separate storage for containers

```
service docker stop
rm -rf /var/lib/docker/*
mkfs.ext2 /dev/sdb1
mount -o defaults,noatime,nodiratime /dev/sdb1 /var/lib/docker
tail -1 /etc/fstab
	/dev/sdb1	/var/lib/docker	    ext2     defaults,noatime,nodiratime,nobootwait 0 1
service docker start
```

#### Limit container storage size to prevent DoS or resource abuse

Switching storage backends to devicemapper allows for disk quotas.
Set dm.basesize to the maximum size the container can grow to (def: 10G)

**Note:** Currently unstable, and all existing container and image data will be lost.

```
service docker stop
rm -rf /var/lib/docker/*
docker -d --storage-driver=devicemapper --storage-opt dm.basesize=3G &
sleep 3 && pkill docker
tail -1 /etc/default/docker
	DOCKER_OPTS="--storage-driver=devicemapper --storage-opt dm.basesize=3G"
start docker
```

**Note:** There's currently a bug in devicemapper that may cause docker to fail to run containers [more info](https://github.com/docker/docker/issues/4036).

#### Rate limiting

Use iptables for protection of the SSH service.
```
make iptables-config
```

#### GRSecurity kernel patches

To aid in protecting the host system, it's recommended to patch the Linux kernel [more info](https://grsecurity.net/)

