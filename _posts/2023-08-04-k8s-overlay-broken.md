---
layout:       post
title:        "Containerd Overlayfs 文件系统损坏导致Pod无法启动"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - Containerd
    - CloudNative
---

首先，在描述问题之前，我们来看下containerd镜像文件系统组织。

## Overlayfs介绍

overlayfs是containerd默认使用的镜像文件系统。

overlayfs是一种联合文件系统，它可以将两个文件系统合并为一个文件系统，其中两个文件系统包括：

- upperdir：上层文件系统，可写。
- lowerdir：下层文件系统，只读。

overlayfs的读写规则是：

- 读取时，从上往下读。
- 写入时，写到下层。如果文件在下层不存在，会先将文件从下层copy到上层后进行修改。
- 删除时，只会删除上层的文件。

我们可以简单测试下：

```bash
mkdir upper lower
echo "I am from upper" > upper/in_upper.txt
echo "I am from upper" > upper/both.txt
echo "I am from lower" > lower/in_lower.txt
echo "I am from lower" > lower/both.txt
mkdir work merged
mount -t overlay overlay -olowerdir=./lower,upperdir=./upper,workdir=./work ./merged
```

现在`merged`即为合并后的目录，我们可以看下它的结构并验证内容：

```bash
tree ./merged/
# ./merged/
# ├── both.txt
# ├── in_lower.txt
# └── in_upper.txt

# 0 directories, 3 files

cat ./merged/both.txt
# I am from upper

cat ./merged/in_lower.txt
# I am from lower

cat ./merged/in_upper.txt
# I am from upper
```

你可以尝试修改merged目录中的文件，看下是否符合overlayfs的规则。

## containerd目录组织

containerd的元数据在默认情况下是存在`/var/lib/containerd`下面的，你也可以通过配置文件来更改它。

下面来介绍一下其中一些重要的目录：

- `io.containerd.metadata.v1.bolt/meta.db`：boltdb文件，存储镜像的元数据。
- `io.containerd.content.v1.content/blobs/sha256`：被拉取的实际镜像以及它们的配置文件储存在这里。其中配置文件可以直接用cat命令查看，layer可以用tar命令进行解压。
- `io.containerd.snapshotter.v1.overlayfs/snapshots`：如果containerd的储存引擎为overlayfs，这里将储存镜像的fs数据。包括解压后的镜像只读layer以及容器的读写layer，容器的最终rootfs将由这个目录下的多个snapshots联合挂载为mergedir实现。
- `io.containerd.runtime.v2.task/default/<container-id>/rootfs`：联合挂载overlayfs后提供给容器的rootfs。

也就是说，containerd拉取镜像并启动容器的过程如下：

- 拉取镜像，将原始数据和配置文件储存在`io.containerd.content.v1.content/blobs/sha256`。
- 将镜像的元数据写入BoltDB：`io.containerd.metadata.v1.bolt/meta.db`。
- 解压容器的镜像数据，将数据保存到overlayfs目录`io.containerd.snapshotter.v1.overlayfs/snapshots`下面。
- 容器启动前，将镜像layer数据作为lowerdir，创建一个新的目录作为upperdir，联合挂载为mergedir设置为容器的rootfs。这样就能让多个容器共享镜像layers了。

了解了这些以后，我们就能更容器排查容器运行时overlayfs的问题了。

## 问题排查

某次，客户的集群启动Pod报错：

```text
Warning  FailedCreatePodSandBox  3m23s (x44 over 13m)  kubelet    Failed to create pod sandbox: rpc error: code = Unknown desc = failed to create containerd task: OCI runtime create failed: container_linux.go:380: starting container process caused: exec: "/pause": stat /pause: no such file or directory: unknown
```

这一看就是创建容器失败了，创建容器的步骤基本上为：kubelet --(grpc)--> containerd --> containerd-shim-v2 --> runc。

我们看到containerd返回给kubelet了错误，描述为`OCI runtime create failed`，解释一下，OCI的全称是"Open Container Initiative"，开放容器标准，它描述了容器最底层的运行时。所以在我们的场景下，OCI runtime错误就表示containerd调用runc出现了问题。

所以基本可以认为，后面的报错`container_linux.go:380: starting container process caused: exec: "/pause": stat /pause: no such file or directory: unknown`是runc返回的。对应的代码位置在[container_linux:380](https://github.com/opencontainers/runc/blob/v1.0.2/libcontainer/container_linux.go#L380)。可以看出，这里是runc在启动容器时，没有找到`/pause`二进制文件。

那后面的`/pause`又是什么呢？这是Kubernetes的机制，所有Pod在启动时都会启动一个init容器，作为其他容器的父进程，以防止同一个Pod内容器进程终止会影响别的容器。

所以到这里，基本可以确定问题了，Pod在启动init容器时，容器中rootfs的/pause不见了。

## 定位问题

那我们第一个怀疑的点就是，拉取的pause镜像是否有问题？上面我们介绍过，拉取的镜像全部存在content目录下。我们可以手动解压一下看看镜像有没有问题。

那么，如何手动解压镜像数据呢？

首先，查看pause镜像的元数据：

```bash
crictl inspecti uhub.service.ucloud.cn/google_containers/pause-amd64:3.2
```

在输出中找到`repoDigests`字段：

```json
{
  "status": {
    "id": "sha256:80d28bedfe5dec59da9ebf8e6260224ac9008ab5c11dbbe16ee3ba3e4439ac2c",
    "repoTags": [
      "uhub.service.ucloud.cn/google_containers/pause-amd64:3.2"
    ],
    "repoDigests": [
      "uhub.service.ucloud.cn/google_containers/pause-amd64@sha256:4a1c4b21597c1b4415bdbecb28a3296c6b5e23ca4f9feeb599860a1dac6a0108"
    ],
  }
  ...
}
```

pause是一个非常简单的镜像，`repoDigests`记录了其镜像的SHA256，我们可以到content中查看该镜像的元数据：

```bash
cat /var/lib/containerd/io.containerd.content.v1.content/blobs/sha256/4a1c4b21597c1b4415bdbecb28a3296c6b5e23ca4f9feeb599860a1dac6a0108
```

输出为：

```json
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/vnd.docker.container.image.v1+json",
      "size": 759,
      "digest": "sha256:80d28bedfe5dec59da9ebf8e6260224ac9008ab5c11dbbe16ee3ba3e4439ac2c"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 296534,
         "digest": "sha256:c74f8866df097496217c9f15efe8f8d3db05d19d678a02d01cc7eaed520bb136"
      }
   ]
}
```

该镜像只有一个layer，其实际SHA256记录在`digest`字段中，我们可以根据这个SHA256将镜像文件解压到临时目录中：

```bash
mkdir /tmp/pause-data
tar -xzvf /var/lib/containerd/io.containerd.content.v1.content/blobs/sha256/c74f8866df097496217c9f15efe8f8d3db05d19d678a02d01cc7eaed520bb136 -C /tmp/pause-data
```

进入临时目录，就可以看到镜像文件：

```bash
ls /tmp/pause-data
# pause
```

可以看到pause二进制并没有丢失。

那就说明overlayfs可能出现问题了，那么如何找到pause镜像对应的解压后的overlayfs呢？其实pause的overlayfs id固定为1，如果你的机器上面有运行中的镜像，可以通过`findmnt`很轻松地验证：

```bash
findmnt  | grep overlayfs
```

找一个pause容器，其挂载为：

```
overlay rw,relatime,lowerdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/1/fs,upperdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/38/fs,workdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/38/work
```

可以看到它的lowerdir为`/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/1/fs`，这对于pause容器来说是固定的。

进入这个目录，会发现pause文件丢失了。

因此就可以定位到问题了：

- 容器镜像原始数据解压后，没有问题，pause二进制文件存在。
- 容器镜像数据解压为snapshot之后，pause二进制文件丢失了。

而储存pause文件的snapshot是作为lowerdir存在的，根据overlayfs的规则，容器是绝无可能去更改它的，即使容器内部删除了pause文件，最多也只会影响upperdir，因此pause文件丢失不可能是容器造成的。

在了解这一点后，我们就可以从Linux系统来排查问题了，看有没有什么组件会去操作这个目录，最终客户排查到是他们的一个清理脚本出现了bug，误删了/var/lib/containerd中的一些数据，导致了这个问题。

虽然这个问题最终原因跟containerd、k8s无关，但是排查过程作为了解containerd底层元数据分布、overlayfs机制，都是很有用的。
