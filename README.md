# BOSH release for dcos

This BOSH release and deployment manifest deploy a [DC/OS](https://dcos.io) cluster.

## Install

Start by configuring your [cloud config](https://bosh.io/docs/update-cloud-config.html) for the infrastructure you would like to deploy to and then deploy:

```
export BOSH_ENVIRONMENT=<bosh-alias>
export BOSH_DEPLOYMENT=dcos
bosh deploy manifests/dcos.yml
```

There currently are no credentials required you can omit the `--vars-store` flag. 


## Create a Development Release

If you would like to test out local modifications of this release you can create, upload and redeploy a development release:

```
bosh create-release --force && \
  bosh -n upload-release && \
  bosh deploy manifests/dcos.yml
```

## Architecture Overview

The goal of this BOSH Release is to make it as simple as possible to maintain the DC/OS bootstrap scripting while leveraging the features of BOSH to create and monitor vms.

Here is an example of a running BOSH deployment of DC/OS:
```
+-------------------------------------------------------+---------+-----+---------+-------------+
| VM                                                    | State   | AZ  | VM Type | IPs         |
+-------------------------------------------------------+---------+-----+---------+-------------+
| bootstrap/0 (23c74258-727c-4257-b2ad-cc241d105c6a)    | running | n/a | default | 10.33.7.151 |
| mesos-agent/0 (55cabe46-3a56-42ad-a226-beb86fe715ee)  | running | n/a | default | 10.33.7.155 |
| mesos-agent/1 (67c31e25-0227-446c-a1a3-c4c50bee8167)  | running | n/a | default | 10.33.7.157 |
| mesos-agent/2 (982f85bb-90e9-4e53-bb04-4cbc234a6438)  | running | n/a | default | 10.33.7.158 |
| mesos-agent/3 (651cb173-5df5-489d-b35e-a41e4e15e9b7)  | running | n/a | default | 10.33.7.156 |
| mesos-master/0 (250192b5-a057-40ab-bc4b-165e479ae7c7) | running | n/a | default | 10.33.7.152 |
| mesos-master/1 (89b25625-88f0-40ab-b7bc-1b85190b60bf) | running | n/a | default | 10.33.7.154 |
| mesos-master/2 (77703369-08ba-4a3d-b02f-a487271cb4e3) | running | n/a | default | 10.33.7.153 |
+-------------------------------------------------------+---------+-----+---------+-------------+

VMs total: 8
```

![vm diagram](/docs/images/dcos-boshrelease1.png)


### Bootstrap VM

This is the only vm which is not native to DC/OS.  This server has two purposes:

 - The first is to run `nginx` as to serve up the DC/OS install scripts for the master and agent servers to consume.  
 - The second is to run an instance of the Docker service for the `dcos_generate_config.sh`

### Mesos Agent & Server VMs

These are the vms which the DC/OS script will perform the installs on.  There are `post-start` scripts in each of the job definitions in this release ( [mesos-agent](/jobs/mesos-agent/templates/bin/post-start) and [mesos-server](/jobs/mesos-server/templates/bin/post-start) ) which perform the following tasks:

 - Create symbolic links back to the BOSH persistent mount point at `/var/vcap/store`
 - Pull the install files from the Bootstrap VM
 - Perform the DC/OS install declaring the vm to either be a master or slave