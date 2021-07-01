# General

## Deployment type

Docker

## Image

Based on a Docker image on Docker Hub:

- https://hub.docker.com/r/itsthenetwork/nfs-server-alpine
- https://github.com/sjiveson/nfs-server-alpine

## Licence

GPL-2.0 License

## Version

NFS utils 2.5.2

## Description

NFS, or Network File System is a distributed file system protocol allows a user on a client computer to access files over a network in the same way they would access a local storage file. Because it is an open standard, anyone can implement the protocol. NFS started in-system as an experiment but the second version was publicly released after the initial success.

# Deployment

General example:

```sh
docker run -d --rm \
        --name nfs \
        --privileged \
        -e SHARED_DIRECTORY=/nfsshare \
        -p 2049:2049 \
        -v $HOME/nfs/data:/nfsshare \
        digitbrain/bb-nfs:latest
```

## Parameters

|Name|Value|Description|
|-|-|-|
|Ports|`-p 2049:2049`|NFS server|
|Env|`-e SHARED_DIRECTORY=/nfsshare`|This image can be used to export and share multiple directories. Be aware that NFSv4 dictates that the additional shared directories are subdirectories of the root share specified by `SHARED_DIRECTORY`.|
|Env|`-e READ_ONLY`|Will cause the exports file to contain ro instead of rw, allowing only read access by clients.|
|Env|`-e SYNC=true`|Will cause the exports file to contain sync instead of async, enabling synchronous mode. Check the exports man page for more information: https://linux.die.net/man/5/exports.|
|Env|`-e PERMITTED="10.11.99.*"`| Will permit only hosts with an IP address starting 10.11.99 to mount the file share.|
|Volumes|`-v $HOME/nfs/data:/nfsshare`|Persist NFS data|


<br>

## Testing

```sh
# Mount storage
sudo mount -v <IP>:/ /mnt/one
sudo mount -v <IP>:/another /mnt/two

# Unmount storage
sudo umount /some/where/here
```

## Miscellaneous

### Privileged Mode

You'll note above with the `docker run` command that privileged mode is required. Yes, this is a security risk but an unavoidable one it seems.

### Multiple Shares

This image can be used to export and share multiple directories with a little modification. Be aware that NFSv4 dictates that the additional shared directories are subdirectories of the root share specified by SHARED_DIRECTORY.

> Note its far easier to volume mount multiple directories as subdirectories of the root/first and share the root.

To share multiple directories you'll need to mount additional volumes and specify additional environment variables in your docker run command. Here's an example:

```sh
docker run -d --rm \
        --name nfs \
        --privileged \
        -v /some/where/fileshare:/nfsshare \
        -v /some/where/else:/nfsshare/another \
        -e SHARED_DIRECTORY=/nfsshare \
        -e SHARED_DIRECTORY_2=/nfsshare/another \
        digitbrain/nfs:latest:latest
```

You should then modify the **nfsd.sh** file to process the extra environment variables and add entries to the exports file. I've already included a working example to get you started:

```
if [ ! -z "${SHARED_DIRECTORY_2}" ]; then
  echo "Writing SHARED_DIRECTORY_2 to /etc/exports file"
  echo "{{SHARED_DIRECTORY_2}} {{PERMITTED}}({{READ_ONLY}},{{SYNC}},no_subtree_check,no_auth_nlm,insecure,no_root_squash)" >> /etc/exports
  /bin/sed -i "s@{{SHARED_DIRECTORY_2}}@${SHARED_DIRECTORY_2}@g" /etc/exports
fi
```

You'll find you can now mount the root share as normal and the second shared directory will be available as a subdirectory. However, you should now be able to mount the second share directly too. In both cases you don't need to specify the root directory name with the mount commands. Using the `docker run` command above to start a container using this image, the two mount commands would be:

```
sudo mount -v 10.11.12.101:/ /mnt/one
sudo mount -v 10.11.12.101:/another /mnt/two
```

You might want to make the root share read only, or even make it inaccessible, to encourage users to only mount the correct, more specific shares directly. To do so you'll need to modify the exports file so the root share doesn't get configured based on the values assigned to the PERMITTED or SYNC environment variables.
