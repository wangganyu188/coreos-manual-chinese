These documents were localized into Esperanto by cloudcube@outlook.com
---
layout: docs
title: OpenStack
category: running_coreos
sub_category: platforms
supported: true
weight: 5
---

# Running CoreOS on OpenStack  
# 在[OpenStack][openstack_link]上运行[CoreOS][coreos_link]  

CoreOS现阶段正处在重大开发并且进行积极的测试，这些操作说明将引导你完成下载[OpenStack][openstack_link]版本的[CoreOS][coreos_link]，然后使用`glance`工具导入该镜像，最后使用`nova`工具运行你的第一个集群.

CoreOS is currently in heavy development and actively being tested.  These
instructions will walk you through downloading CoreOS for OpenStack, importing
it with the `glance` tool and running your first cluster with the `nova` tool.




## Import the Image

## 导入镜像  

这些步骤将会下载镜像，解压然后导入[glance][glance_link]镜像存储  

These steps will download the CoreOS image, uncompress it and then import it
into the glance image store.

### Choosing a Channel
### 选择一个通道  


CoreOS is released into alpha and beta channels. Releases to each channel serve as a release-candidate for the next channel. For example, a bug-free alpha release is promoted bit-for-bit to the beta channel.
[CoreOS][coreos_link]被分发到`alpha`与`beta`两个通道，发布到每一个通道的[CoreOS][coreos_link]版本，作为一个释放候选的下一个通道.例如:一个没有缺陷`alpha`版本将被提升位对位到`beta`通道.  


基于下面的URL选择通道，使用`beta`替换`alpha`非常简单.阅读[发布说明]({{site.url}}/releases)为在每一个通道指定的特性与缺陷修复。  
The channel is selected based on the URL below. Simply replace `alpha` with `beta`. Read the [release notes]({{site.url}}/releases) for specific features and bug fixes in each channel.

```sh
$ wget http://alpha.release.core-os.net/amd64-usr/current/coreos_production_openstack_image.img.bz2
$ bunzip2 coreos_production_openstack_image.img.bz2
$ glance image-create --name CoreOS \
  --container-format bare \
  --disk-format qcow2 \
  --file coreos_production_openstack_image.img \
  --is-public True
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | 4742f3c30bd2dcbaf3990ac338bd8e8c     |
| container_format | ovf                                  |
| created_at       | 2013-08-29T22:21:22                  |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | qcow2                                |
| id               | cdf3874c-c27f-4816-bc8c-046b240e0edd |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | coreos                               |
| owner            | 8e662c811b184482adaa34c89a9c33ae     |
| protected        | False                                |
| size             | 363660800                            |
| status           | active                               |
| updated_at       | 2013-08-29T22:22:04                  |
+------------------+--------------------------------------+
```

## Cloud-Config

## Cloud-Config  

CoreOS allows you to configure machine parameters, launch systemd units on startup and more via cloud-config. Jump over to the [docs to learn about the supported features][cloud-config].

[CoreOS][coreos_link]允许你配置机器参数,通过[cloud-config][cloud-config_link]在启动的时候运行[systemd][systemd_link]单元.  跳转到[cloud-config的更多特性][cloud-config_link]


We're going to provide our cloud-config to OpenStack via the user-data flag. Our cloud-config will also contain SSH keys that will be used to connect to the instance.
In order for this to work your OpenStack cloud provider must support [config drive][config-drive] or the OpenStack metadata service.

[cloud-config]: {{site.url}}/docs/cluster-management/setup/cloudinit-cloud-config
[config-drive]: http://docs.openstack.org/user-guide/content/config-drive.html

The most common cloud-config for OpenStack looks like:

```yaml
#cloud-config

coreos:
  etcd:
    # generate a new token for each unique cluster from https://discovery.etcd.io/new
    discovery: https://discovery.etcd.io/<token>
    # multi-region and multi-cloud deployments need to use $public_ipv4
    addr: $private_ipv4:4001
    peer-addr: $private_ipv4:7001
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
ssh_authorized_keys:
  # include one or more SSH public keys
  - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC0g+ZTxC7weoIJLUafOgrm+h...
```

The `$private_ipv4` and `$public_ipv4` substitution variables are fully supported in cloud-config on most OpenStack deployments. Unfortunately some systems relying on config drive may leave these values undefined.

## Launch Cluster

Boot the machines with the `nova` CLI, referencing the image ID from the import step above and your `cloud-config.yaml`:

```sh
nova boot \
--user-data ./cloud-config.yaml \
--image cdf3874c-c27f-4816-bc8c-046b240e0edd \
--key-name coreos \
--flavor m1.medium \
--num-instances 3 \
--security-groups default coreos
```

To use config drive you may need to add `--config-drive=true` to command above.

Your first CoreOS cluster should now be running. The only thing left to do is
find an IP and SSH in.

```sh
$ nova list
+--------------------------------------+-----------------+--------+------------+-------------+-------------------+
| ID                                   | Name            | Status | Task State | Power State | Networks          |
+--------------------------------------+-----------------+--------+------------+-------------+-------------------+
| a1df1d98-622f-4f3b-adef-cb32f3e2a94d | coreos-a1df1d98 | ACTIVE | None       | Running     | private=10.0.0.3  |
| db13c6a7-a474-40ff-906e-2447cbf89440 | coreos-db13c6a7 | ACTIVE | None       | Running     | private=10.0.0.4  |
| f70b739d-9ad8-4b0b-bb74-4d715205ff0b | coreos-f70b739d | ACTIVE | None       | Running     | private=10.0.0.5  |
+--------------------------------------+-----------------+--------+------------+-------------+-------------------+
```

Finally SSH into an instance, note that the user is `core`:

```sh
$ chmod 400 core.pem
$ ssh -i core.pem core@10.0.0.3
   ______                ____  _____
  / ____/___  ________  / __ \/ ___/
 / /   / __ \/ ___/ _ \/ / / /\__ \
/ /___/ /_/ / /  /  __/ /_/ /___/ /
\____/\____/_/   \___/\____//____/

core@10-0-0-3 ~ $
```

## Adding More Machines

Adding new instances to the cluster is as easy as launching more with the same 
cloud-config. New instances will join the cluster assuming they can communicate 
with the others.

Example:

```sh
nova boot \
--user-data ./cloud-config.yaml \
--image cdf3874c-c27f-4816-bc8c-046b240e0edd \
--key-name coreos \
--flavor m1.medium \
--security-groups default coreos
```

## Multiple Clusters

If you would like to create multiple clusters you'll need to generate and use a
new discovery token. Change the token value on the etcd discovery parameter in the cloud-config, and boot new instances.

## Using CoreOS

Now that you have instances booted it is time to play around.
Check out the [CoreOS Quickstart]({{site.url}}/docs/quickstart) guide or dig into [more specific topics]({{site.url}}/docs).




[coreos_link]:http://www.coreos.com  
[openstack_link]:http://www.openstack.org

