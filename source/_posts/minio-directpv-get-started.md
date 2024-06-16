---
title: 在 Kubernetes 中使用 DirectPV 管理 MinIO 存储 - 入门
date: 2024-06-13 12:57:16
tags:
  - Linux
  - Storage
  - MinIO
  - DirectPV
  - Kubernetes
  - CSI
  - StorageClass
---
`DriectPV` 是一种用于 `Direct Attached Storage`（直连存储）的 `CSI` 磁盘。简单来说，`DirectPV` 是一个分布式持久卷管理器，而非 `SAN` 或 `NAS` 这样的存储系统，可以检测 `Kubernetes` 集群中各个节点上的未挂载。

## 术语

- `drive`：通常指的是物理设备，如机械硬盘、固态硬盘等，是 `volume` 的物理载体。
- `volume`：通常指的是逻辑概念，是 `drive` 的逻辑抽象，可以实现容错、镜像等高级功能。一个物理 `drive` 可以包含一个或多个逻辑 `volume`，而一个 `volume` 通常对应一个磁盘分区，但也可能跨越多个分区或磁盘。文件系统构建在逻辑 `volume` 之上，而不是直接建立在物理 `drive` 上。

## 上手

1. 使用 [`Krew`](https://krew.sigs.k8s.io/docs/user-guide/setup/install/) 安装 `DirectPV` 插件：

   ```Shell
   kubectl krew install directpv
   ```
2. 在 `Kubernetes` 集群中安装 `DirectPV`：

   ```Shell
   kubectl label node storage-1.aihub.aistar.info usage=storage
   kubectl label node storage-2.aihub.aistar.info usage=storage

   # 通过 --node-selector 参数可以根据 label 选择 DirectPV 所部署的节点
   kubectl directpv install --node-selector usage=storage
   Installing on unsupported Kubernetes v1.29

    ███████████████████████████████████████████████████████████████████████████ 100%

   ┌──────────────────────────────────────┬──────────────────────────┐
   │ NAME                                 │ KIND                     │
   ├──────────────────────────────────────┼──────────────────────────┤
   │ directpv                             │ Namespace                │
   │ directpv-min-io                      │ ServiceAccount           │
   │ directpv-min-io                      │ ClusterRole              │
   │ directpv-min-io                      │ ClusterRoleBinding       │
   │ directpv-min-io                      │ Role                     │
   │ directpv-min-io                      │ RoleBinding              │
   │ directpvdrives.directpv.min.io       │ CustomResourceDefinition │
   │ directpvvolumes.directpv.min.io      │ CustomResourceDefinition │
   │ directpvnodes.directpv.min.io        │ CustomResourceDefinition │
   │ directpvinitrequests.directpv.min.io │ CustomResourceDefinition │
   │ directpv-min-io                      │ CSIDriver                │
   │ directpv-min-io                      │ StorageClass             │
   │ node-server                          │ Daemonset                │
   │ controller                           │ Deployment               │
   └──────────────────────────────────────┴──────────────────────────┘

   DirectPV installed successfully

   # 可以看到仅在选定的节点上部署了 node-server
   kubectl -n directpv get pods -owide
   NAME                          READY   STATUS    RESTARTS   AGE   IP             NODE                           NOMINATED NODE   READINESS GATES
   controller-5f544cbd8c-dklpv   2/3     Running   0          21s   10.216.3.14    storage-2.ai-hub.aistar.info   <none>           <none>
   controller-5f544cbd8c-fq4fp   2/3     Running   0          21s   10.216.1.253   a800-1.ai-hub.aistar.info      <none>           <none>
   controller-5f544cbd8c-nzvn4   2/3     Running   0          21s   10.216.2.11    storage-1.ai-hub.aistar.info   <none>           <none>
   node-server-rfhj6             3/4     Running   0          21s   10.216.2.12    storage-1.ai-hub.aistar.info   <none>           <none>
   node-server-vsktt             3/4     Running   0          21s   10.216.3.13    storage-2.ai-hub.aistar.info   <none>           <none>

   # node-server 为 DaemonSet，会在选定的每个节点上部署一个实例，用于挂载 `drive` 为 `Local PV`
   kubectl -n directpv get all
   NAME                              READY   STATUS    RESTARTS   AGE
   pod/controller-5f544cbd8c-dklpv   2/3     Running   0          27s
   pod/controller-5f544cbd8c-fq4fp   2/3     Running   0          27s
   pod/controller-5f544cbd8c-nzvn4   2/3     Running   0          27s
   pod/node-server-rfhj6             3/4     Running   0          27s
   pod/node-server-vsktt             3/4     Running   0          27s

   NAME                         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
   daemonset.apps/node-server   2         2         0       2            0           <none>          28s

   NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/controller   0/3     3            0           28s

   NAME                                    DESIRED   CURRENT   READY   AGE
   replicaset.apps/controller-5f544cbd8c   3         3         0       28s

   # 默认会创建名为 directpv-min-io 的 StorageClass，后续会介绍如何根据存储介质类型来创建相应 StorageClass
   kubectl -n directpv get sc | grep directpv
   directpv-min-io             directpv-min-io         Delete          WaitForFirstConsumer   true                   81s

   kubectl -n directpv get sc directpv-min-io -o yaml
   allowVolumeExpansion: true
   allowedTopologies:
   - matchLabelExpressions:
     - key: directpv.min.io/identity
       values:
       - directpv-min-io
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     annotations:
       directpv.min.io/plugin-version: v4.0.10
     creationTimestamp: "2024-06-13T06:02:35Z"
     finalizers:
     - foregroundDeletion
     labels:
       application-name: directpv.min.io
       application-type: CSIDriver
       directpv.min.io/created-by: kubectl-directpv
       directpv.min.io/version: v1beta1
     name: directpv-min-io
     resourceVersion: "3333324"
     uid: 006dc208-8181-4dad-83c9-a848f67d5c39
   parameters:
     csi.storage.k8s.io/fstype: xfs
   provisioner: directpv-min-io
   reclaimPolicy: Delete
   volumeBindingMode: WaitForFirstConsumer
   ```
3. 获取安装信息：

   ```Shell
   kubectl directpv info
   ```
4. 检测 `Kubernetes` 集群中的可用 `drives`，集群各节点上当前没有挂载的 `drives` 都会展示出来：

   ```Shell
   # Discover drives to check the available devices in the cluster to initialize
   # The following command will create an init config file (default: drives.yaml) which will be used for initialization
   kubectl directpv discover

   # 如果不加任何限制，会罗列如下内容，并在当前目录下创建 drives.yaml 文件
   kubectl directpv discover

    Discovered node 'storage-1.ai-hub.aistar.info' ✔
    Discovered node 'storage-2.ai-hub.aistar.info' ✔

   ┌─────────────────────┬──────────────────────────────┬─────────┬─────────┬────────────┬───────────────────────────┬───────────┬─────────────┐
   │ ID                  │ NODE                         │ DRIVE   │ SIZE    │ FILESYSTEM │ MAKE                      │ AVAILABLE │ DESCRIPTION │
   ├─────────────────────┼──────────────────────────────┼─────────┼─────────┼────────────┼───────────────────────────┼───────────┼─────────────┤
   │ 259:1$CD/B7hOzmP... │ storage-1.ai-hub.aistar.info │ nvme0n1 │ 7.0 TiB │ -          │ XFSP4157T60000N           │ YES       │ -           │
   │ 259:0$wcrBtilTtI... │ storage-1.ai-hub.aistar.info │ nvme1n1 │ 7.0 TiB │ -          │ XFSP4157T60000N           │ YES       │ -           │
   │ 8:0$BKKArjtxaZ/i... │ storage-1.ai-hub.aistar.info │ sda     │ 13 TiB  │ -          │ WUH721414ALE6L0           │ YES       │ -           │
   │ 8:16$eP1pOGWAy58... │ storage-1.ai-hub.aistar.info │ sdb     │ 13 TiB  │ -          │ WUH721414ALE6L0           │ YES       │ -           │
   │ 8:32$eh+MtHw0Xsq... │ storage-1.ai-hub.aistar.info │ sdc     │ 13 TiB  │ -          │ WUH721414ALE6L0           │ YES       │ -           │
   │ 8:48$76U9WbYRvQj... │ storage-1.ai-hub.aistar.info │ sdd     │ 13 TiB  │ -          │ WUH721414ALE6L0           │ YES       │ -           │
   │ 8:68$JL37EZHVfW4... │ storage-1.ai-hub.aistar.info │ sde4    │ 977 MiB │ swap       │ AVAGO XF-SAS3408 (Part 4) │ YES       │ -           │
   │ 259:1$g8XDt0LYk2... │ storage-2.ai-hub.aistar.info │ nvme0n1 │ 7.0 TiB │ -          │ XFSP4157T60000N           │ YES       │ -           │
   │ 259:0$d+mW0S9iaw... │ storage-2.ai-hub.aistar.info │ nvme1n1 │ 7.0 TiB │ -          │ XFSP4157T60000N           │ YES       │ -           │
   │ 8:0$uPotqaQII0Wp... │ storage-2.ai-hub.aistar.info │ sda     │ 13 TiB  │ -          │ WUH721414ALE6L0           │ YES       │ -           │
   │ 8:16$gpv2Pgulvvt... │ storage-2.ai-hub.aistar.info │ sdb     │ 13 TiB  │ -          │ WUH721414ALE6L0           │ YES       │ -           │
   │ 8:32$L1kjwMRcl3f... │ storage-2.ai-hub.aistar.info │ sdc     │ 13 TiB  │ -          │ WUH721414ALE6L0           │ YES       │ -           │
   │ 8:52$sMKOhae3sff... │ storage-2.ai-hub.aistar.info │ sdd4    │ 977 MiB │ swap       │ AVAGO XF-SAS3408 (Part 4) │ YES       │ -           │
   │ 8:64$RhqrV/1oOSF... │ storage-2.ai-hub.aistar.info │ sde     │ 13 TiB  │ -          │ WUH721414ALE6L0           │ YES       │ -           │
   └─────────────────────┴──────────────────────────────┴─────────┴─────────┴────────────┴───────────────────────────┴───────────┴─────────────┘

   Generated 'drives.yaml' successfully.

   # 我们可以按照存储类型进行分类
   kubectl directpv discover --nodes=storage-{1...2}.ai-hub.aistar.info --drives=nvme{0...3}n1 --drives=sd{a...f}
   # 归类 hdd 存储（一般没有必要）
   kubectl directpv discover --nodes=storage-{1...2}.ai-hub.aistar.info --drives=sd{a...f} --output-file hdd-drives.yaml

    Discovered node 'storage-1.ai-hub.aistar.info' ✔
    Discovered node 'storage-2.ai-hub.aistar.info' ✔

   ┌─────────────────────┬──────────────────────────────┬───────┬────────┬────────────┬─────────────────┬───────────┬─────────────┐
   │ ID                  │ NODE                         │ DRIVE │ SIZE   │ FILESYSTEM │ MAKE            │ AVAILABLE │ DESCRIPTION │
   ├─────────────────────┼──────────────────────────────┼───────┼────────┼────────────┼─────────────────┼───────────┼─────────────┤
   │ 8:0$BKKArjtxaZ/i... │ storage-1.ai-hub.aistar.info │ sda   │ 13 TiB │ -          │ WUH721414ALE6L0 │ YES       │ -           │
   │ 8:16$eP1pOGWAy58... │ storage-1.ai-hub.aistar.info │ sdb   │ 13 TiB │ -          │ WUH721414ALE6L0 │ YES       │ -           │
   │ 8:32$eh+MtHw0Xsq... │ storage-1.ai-hub.aistar.info │ sdc   │ 13 TiB │ -          │ WUH721414ALE6L0 │ YES       │ -           │
   │ 8:48$76U9WbYRvQj... │ storage-1.ai-hub.aistar.info │ sdd   │ 13 TiB │ -          │ WUH721414ALE6L0 │ YES       │ -           │
   │ 8:0$uPotqaQII0Wp... │ storage-2.ai-hub.aistar.info │ sda   │ 13 TiB │ -          │ WUH721414ALE6L0 │ YES       │ -           │
   │ 8:16$gpv2Pgulvvt... │ storage-2.ai-hub.aistar.info │ sdb   │ 13 TiB │ -          │ WUH721414ALE6L0 │ YES       │ -           │
   │ 8:32$L1kjwMRcl3f... │ storage-2.ai-hub.aistar.info │ sdc   │ 13 TiB │ -          │ WUH721414ALE6L0 │ YES       │ -           │
   │ 8:64$RhqrV/1oOSF... │ storage-2.ai-hub.aistar.info │ sde   │ 13 TiB │ -          │ WUH721414ALE6L0 │ YES       │ -           │
   └─────────────────────┴──────────────────────────────┴───────┴────────┴────────────┴─────────────────┴───────────┴─────────────┘

   Generated 'hdd-drives.yaml' successfully.

   cat hdd-drives.yaml 
   version: v1
   nodes:
       - name: storage-1.ai-hub.aistar.info
         drives:
           - id: 8:48$76U9WbYRvQjMfj8QgtoXqiw8VI7FNkoG3qQbH6VR+SM=
             name: sdd
             size: 14000519643136
             make: WUH721414ALE6L0
             select: "yes"
           - id: 8:32$eh+MtHw0Xsqhw/p9RIbkvZERHuOVoPqVz1LfneQEr4U=
             name: sdc
             size: 14000519643136
             make: WUH721414ALE6L0
             select: "yes"
           - id: 8:16$eP1pOGWAy58AmEajzEU4wBCkjgAK/y5fcSCMzum7OaM=
             name: sdb
             size: 14000519643136
             make: WUH721414ALE6L0
             select: "yes"
           - id: 8:0$BKKArjtxaZ/iAmDOOkY6qQ1YpoFuIiwe05Z88lWav5g=
             name: sda
             size: 14000519643136
             make: WUH721414ALE6L0
             select: "yes"
       - name: storage-2.ai-hub.aistar.info
         drives:
           - id: 8:64$RhqrV/1oOSF67MUsGZ66U0X0AnYlaQtt9aRrzeHicwE=
             name: sde
             size: 14000519643136
             make: WUH721414ALE6L0
             select: "yes"
           - id: 8:0$uPotqaQII0Wp1D2zVHyJwFyPiwRS7MW+zlA+Bprb2pY=
             name: sda
             size: 14000519643136
             make: WUH721414ALE6L0
             select: "yes"
           - id: 8:32$L1kjwMRcl3fVekYtzQpr3cxeTlEzqUTpZx5b4EPnQPg=
             name: sdc
             size: 14000519643136
             make: WUH721414ALE6L0
             select: "yes"
           - id: 8:16$gpv2Pgulvvtjz/R54mNbOzLiCxwHj5uinPPZ8MySBDU=
             name: sdb
             size: 14000519643136
             make: WUH721414ALE6L0
             select: "yes"

   # 归类 nvme 存储（一般没有必要）
   kubectl directpv discover --nodes=storage-{1...2}.ai-hub.aistar.info --drives=nvme{0...3}n1 --output-file nvme-drives.yaml

    Discovered node 'storage-1.ai-hub.aistar.info' ✔
    Discovered node 'storage-2.ai-hub.aistar.info' ✔

   ┌─────────────────────┬──────────────────────────────┬─────────┬─────────┬────────────┬─────────────────┬───────────┬─────────────┐
   │ ID                  │ NODE                         │ DRIVE   │ SIZE    │ FILESYSTEM │ MAKE            │ AVAILABLE │ DESCRIPTION │
   ├─────────────────────┼──────────────────────────────┼─────────┼─────────┼────────────┼─────────────────┼───────────┼─────────────┤
   │ 259:1$CD/B7hOzmP... │ storage-1.ai-hub.aistar.info │ nvme0n1 │ 7.0 TiB │ -          │ XFSP4157T60000N │ YES       │ -           │
   │ 259:0$wcrBtilTtI... │ storage-1.ai-hub.aistar.info │ nvme1n1 │ 7.0 TiB │ -          │ XFSP4157T60000N │ YES       │ -           │
   │ 259:1$g8XDt0LYk2... │ storage-2.ai-hub.aistar.info │ nvme0n1 │ 7.0 TiB │ -          │ XFSP4157T60000N │ YES       │ -           │
   │ 259:0$d+mW0S9iaw... │ storage-2.ai-hub.aistar.info │ nvme1n1 │ 7.0 TiB │ -          │ XFSP4157T60000N │ YES       │ -           │
   └─────────────────────┴──────────────────────────────┴─────────┴─────────┴────────────┴─────────────────┴───────────┴─────────────┘

   Generated 'nvme-drives.yaml' successfully.

   cat nvme-drives.yaml
   version: v1
   nodes:
       - name: storage-2.ai-hub.aistar.info
         drives:
           - id: 259:0$d+mW0S9iawSC8In5/tXQ3shQahmlquO1V9GEVaK8eDQ=
             name: nvme1n1
             size: 7681501126656
             make: XFSP4157T60000N
             select: "yes"
           - id: 259:1$g8XDt0LYk2vvCl2ZVZ31LNyRvokZA+j7u7AbQm/hjYs=
             name: nvme0n1
             size: 7681501126656
             make: XFSP4157T60000N
             select: "yes"
       - name: storage-1.ai-hub.aistar.info
         drives:
           - id: 259:0$wcrBtilTtIJKmkTanwRzPsMAOG4/k/tpfXDm1QoOa9M=
             name: nvme1n1
             size: 7681501126656
             make: XFSP4157T60000N
             select: "yes"
           - id: 259:1$CD/B7hOzmPls2XCHBKROndOTRaluWkeWeGMB8a+KFR0=
             name: nvme0n1
             size: 7681501126656
             make: XFSP4157T60000N
             select: "yes"
   ```
5. 添加 `drives` 用于管理持久卷并提供调度：

   ```Shell
   kubectl directpv init drives.yaml

   # 统一初始化 hdd drives
   kubectl directpv init hdd-drives.yaml
   ERROR Initializing the drives will permanently erase existing data. Please review carefully before performing this *DANGEROUS* operation and retry this command with --dangerous flag.

   kubectl directpv init hdd-drives.yaml --dangerous

    ███████████████████████████████████████████████████████████████████████████ 100%

    Processed initialization request 'af4d2375-0363-4e40-b260-dbe87ac0e10b' for node 'storage-1.ai-hub.aistar.info' ✔
    Processed initialization request 'ad732117-7bd9-46bf-900d-9b28978d804a' for node 'storage-2.ai-hub.aistar.info' ✔

   ┌──────────────────────────────────────┬──────────────────────────────┬───────┬─────────┐
   │ REQUEST_ID                           │ NODE                         │ DRIVE │ MESSAGE │
   ├──────────────────────────────────────┼──────────────────────────────┼───────┼─────────┤
   │ af4d2375-0363-4e40-b260-dbe87ac0e10b │ storage-1.ai-hub.aistar.info │ sda   │ Success │
   │ af4d2375-0363-4e40-b260-dbe87ac0e10b │ storage-1.ai-hub.aistar.info │ sdb   │ Success │
   │ af4d2375-0363-4e40-b260-dbe87ac0e10b │ storage-1.ai-hub.aistar.info │ sdc   │ Success │
   │ af4d2375-0363-4e40-b260-dbe87ac0e10b │ storage-1.ai-hub.aistar.info │ sdd   │ Success │
   │ ad732117-7bd9-46bf-900d-9b28978d804a │ storage-2.ai-hub.aistar.info │ sda   │ Success │
   │ ad732117-7bd9-46bf-900d-9b28978d804a │ storage-2.ai-hub.aistar.info │ sdb   │ Success │
   │ ad732117-7bd9-46bf-900d-9b28978d804a │ storage-2.ai-hub.aistar.info │ sdc   │ Success │
   │ ad732117-7bd9-46bf-900d-9b28978d804a │ storage-2.ai-hub.aistar.info │ sde   │ Success │
   └──────────────────────────────────────┴──────────────────────────────┴───────┴─────────┘

   # 统一初始化 nvme drives
   kubectl directpv init nvme-drives.yaml --dangerous

    ███████████████████████████████████████████████████████████████████████████ 100%

    Processed initialization request 'ceda7628-dabc-4958-9e7c-eccb5597fa20' for node 'storage-2.ai-hub.aistar.info' ✔
    Processed initialization request '7a653d5b-7cd9-47d0-8949-3abc3619eb75' for node 'storage-1.ai-hub.aistar.info' ✔

   ┌──────────────────────────────────────┬──────────────────────────────┬─────────┬─────────┐
   │ REQUEST_ID                           │ NODE                         │ DRIVE   │ MESSAGE │
   ├──────────────────────────────────────┼──────────────────────────────┼─────────┼─────────┤
   │ 7a653d5b-7cd9-47d0-8949-3abc3619eb75 │ storage-1.ai-hub.aistar.info │ nvme0n1 │ Success │
   │ 7a653d5b-7cd9-47d0-8949-3abc3619eb75 │ storage-1.ai-hub.aistar.info │ nvme1n1 │ Success │
   │ ceda7628-dabc-4958-9e7c-eccb5597fa20 │ storage-2.ai-hub.aistar.info │ nvme0n1 │ Success │
   │ ceda7628-dabc-4958-9e7c-eccb5597fa20 │ storage-2.ai-hub.aistar.info │ nvme1n1 │ Success │
   └──────────────────────────────────────┴──────────────────────────────┴─────────┴─────────┘
   ```
6. 给 `drives` 打上标签：

   ```Shell
   kubectl directpv label drives directpv.min.io/tier=hot directpv.min.io/type=nvme  --drives=nvme{0...3}n1
   Label 'directpv.min.io/tier:hot' successfully set on storage-1.ai-hub.aistar.info/nvme0n1
   Label 'directpv.min.io/type:nvme' successfully set on storage-1.ai-hub.aistar.info/nvme0n1
   Label 'directpv.min.io/tier:hot' successfully set on storage-2.ai-hub.aistar.info/nvme1n1
   Label 'directpv.min.io/type:nvme' successfully set on storage-2.ai-hub.aistar.info/nvme1n1
   Label 'directpv.min.io/tier:hot' successfully set on storage-2.ai-hub.aistar.info/nvme0n1
   Label 'directpv.min.io/type:nvme' successfully set on storage-2.ai-hub.aistar.info/nvme0n1
   Label 'directpv.min.io/tier:hot' successfully set on storage-1.ai-hub.aistar.info/nvme1n1
   Label 'directpv.min.io/type:nvme' successfully set on storage-1.ai-hub.aistar.info/nvme1n1

   kubectl directpv label drives directpv.min.io/tier=cold directpv.min.io/type=hdd --drives=sd{a...f}
   Label 'directpv.min.io/tier:cold' successfully set on storage-2.ai-hub.aistar.info/sdb
   Label 'directpv.min.io/type:hdd' successfully set on storage-2.ai-hub.aistar.info/sdb
   Label 'directpv.min.io/tier:cold' successfully set on storage-1.ai-hub.aistar.info/sdd
   Label 'directpv.min.io/type:hdd' successfully set on storage-1.ai-hub.aistar.info/sdd
   Label 'directpv.min.io/tier:cold' successfully set on storage-2.ai-hub.aistar.info/sda
   Label 'directpv.min.io/type:hdd' successfully set on storage-2.ai-hub.aistar.info/sda
   Label 'directpv.min.io/tier:cold' successfully set on storage-1.ai-hub.aistar.info/sdb
   Label 'directpv.min.io/type:hdd' successfully set on storage-1.ai-hub.aistar.info/sdb
   Label 'directpv.min.io/tier:cold' successfully set on storage-2.ai-hub.aistar.info/sdc
   Label 'directpv.min.io/type:hdd' successfully set on storage-2.ai-hub.aistar.info/sdc
   Label 'directpv.min.io/tier:cold' successfully set on storage-1.ai-hub.aistar.info/sdc
   Label 'directpv.min.io/type:hdd' successfully set on storage-1.ai-hub.aistar.info/sdc
   Label 'directpv.min.io/tier:cold' successfully set on storage-2.ai-hub.aistar.info/sde
   Label 'directpv.min.io/type:hdd' successfully set on storage-2.ai-hub.aistar.info/sde
   Label 'directpv.min.io/tier:cold' successfully set on storage-1.ai-hub.aistar.info/sda
   Label 'directpv.min.io/type:hdd' successfully set on storage-1.ai-hub.aistar.info/sda
   ```
7. 列出所有已添加的 `drives`：

   ```Shell
   kubectl directpv list drives
   xxx

   # 列出所有包含 tier=cold 和 type=hdd 标签的 drives
   kubectl directpv list drives --labels directpv.min.io/tier=cold --labels directpv.min.io/type=hdd
   ┌──────────────────────────────┬──────┬─────────────────┬────────┬────────┬─────────┬────────┐
   │ NODE                         │ NAME │ MAKE            │ SIZE   │ FREE   │ VOLUMES │ STATUS │
   ├──────────────────────────────┼──────┼─────────────────┼────────┼────────┼─────────┼────────┤
   │ storage-1.ai-hub.aistar.info │ sda  │ WUH721414ALE6L0 │ 13 TiB │ 13 TiB │ -       │ Ready  │
   │ storage-1.ai-hub.aistar.info │ sdb  │ WUH721414ALE6L0 │ 13 TiB │ 13 TiB │ -       │ Ready  │
   │ storage-1.ai-hub.aistar.info │ sdc  │ WUH721414ALE6L0 │ 13 TiB │ 13 TiB │ -       │ Ready  │
   │ storage-1.ai-hub.aistar.info │ sdd  │ WUH721414ALE6L0 │ 13 TiB │ 13 TiB │ -       │ Ready  │
   │ storage-2.ai-hub.aistar.info │ sda  │ WUH721414ALE6L0 │ 13 TiB │ 13 TiB │ -       │ Ready  │
   │ storage-2.ai-hub.aistar.info │ sdb  │ WUH721414ALE6L0 │ 13 TiB │ 13 TiB │ -       │ Ready  │
   │ storage-2.ai-hub.aistar.info │ sdc  │ WUH721414ALE6L0 │ 13 TiB │ 13 TiB │ -       │ Ready  │
   │ storage-2.ai-hub.aistar.info │ sde  │ WUH721414ALE6L0 │ 13 TiB │ 13 TiB │ -       │ Ready  │
   └──────────────────────────────┴──────┴─────────────────┴────────┴────────┴─────────┴────────┘

   # 列出所有包含 tier=hot 和 type=nvme 标签的 drives
   kubectl directpv list drives --labels directpv.min.io/tier=hot --labels directpv.min.io/type=nvme
   ┌──────────────────────────────┬─────────┬─────────────────┬─────────┬─────────┬─────────┬────────┐
   │ NODE                         │ NAME    │ MAKE            │ SIZE    │ FREE    │ VOLUMES │ STATUS │
   ├──────────────────────────────┼─────────┼─────────────────┼─────────┼─────────┼─────────┼────────┤
   │ storage-1.ai-hub.aistar.info │ nvme0n1 │ XFSP4157T60000N │ 7.0 TiB │ 7.0 TiB │ -       │ Ready  │
   │ storage-1.ai-hub.aistar.info │ nvme1n1 │ XFSP4157T60000N │ 7.0 TiB │ 7.0 TiB │ -       │ Ready  │
   │ storage-2.ai-hub.aistar.info │ nvme0n1 │ XFSP4157T60000N │ 7.0 TiB │ 7.0 TiB │ -       │ Ready  │
   │ storage-2.ai-hub.aistar.info │ nvme1n1 │ XFSP4157T60000N │ 7.0 TiB │ 7.0 TiB │ -       │ Ready  │
   └──────────────────────────────┴─────────┴─────────────────┴─────────┴─────────┴─────────┴────────┘

   # 登录到 storage-1.ai-hub.aistar.info 节点上，我们可以查看 DirectPV 已经帮我们格式化并进行了挂载（默认文件系统为 XFS 类型）
   sudo lsblk -f
   NAME    FSTYPE FSVER LABEL    UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
   sda     xfs          DIRECTPV f3e32e35-d1f4-4b90-9acd-015fbc367be3   12.6T     1% /var/lib/directpv/mnt/f3e32e35-d1f4-4b90-9acd-015fbc367be3
   sdb     xfs          DIRECTPV 720c3302-c3dd-4853-8604-852469524e10   12.6T     1% /var/lib/directpv/mnt/720c3302-c3dd-4853-8604-852469524e10
   sdc     xfs          DIRECTPV cc2cb268-1383-4ecf-8d13-e2d81815897a   12.6T     1% /var/lib/directpv/mnt/cc2cb268-1383-4ecf-8d13-e2d81815897a
   sdd     xfs          DIRECTPV 651c1b8e-df75-415b-9095-cd5eaeac53be   12.6T     1% /var/lib/directpv/mnt/651c1b8e-df75-415b-9095-cd5eaeac53be
   nvme1n1 xfs          DIRECTPV fe04ab16-127d-4636-94a1-add08c8d3f15    6.9T     1% /var/lib/directpv/mnt/fe04ab16-127d-4636-94a1-add08c8d3f15
   nvme0n1 xfs          DIRECTPV 475b112b-5032-4184-80bd-7a0af49c5dc1    6.9T     1% /var/lib/directpv/mnt/475b112b-5032-4184-80bd-7a0af49c5dc1

   # 根据如下信息可以初步判断 `mount` 信息应该被 `DirectPV` 存储在 `etcd` 中，每次启动节点时都会进行挂载
   sudo systemctl list-units | grep directpv
   [sudo] password for yobol: 
     var-lib-directpv-mnt-f3e32e35\x2dd1f4\x2d4b90\x2d9acd\x2d015fbc367be3.mount                                                                   loaded active mounted   /var/lib/directpv/mnt/f3e32e35-d1f4-4b90-9acd-015fbc367be3
     ...
   yobol@storage-1:~$ sudo systemctl status 'var-lib-directpv-mnt-f3e32e35\x2dd1f4\x2d4b90\x2d9acd\x2d015fbc367be3.mount'
   ● var-lib-directpv-mnt-f3e32e35\x2dd1f4\x2d4b90\x2d9acd\x2d015fbc367be3.mount - /var/lib/directpv/mnt/f3e32e35-d1f4-4b90-9acd-015fbc367be3
        Loaded: loaded (/proc/self/mountinfo)
        Active: active (mounted) since Thu 2024-06-13 15:43:31 CST; 1h 14min ago
         Where: /var/lib/directpv/mnt/f3e32e35-d1f4-4b90-9acd-015fbc367be3
          What: /dev/sda
   ```
8. 根据 `label` 选择 `drives` 创建对应的 `StorageClass`：

   ```Shell
   # https://github.com/minio/directpv/blob/master/docs/tools/create-storage-class.sh
   ./create-storage-class.sh hot-tier-storage 'directpv.min.io/tier: hot'
   ./create-storage-class.sh cold-tier-storage 'directpv.min.io/tier: cold'
   ```
9. 创建 `cold-tier.yaml` 进行测试，理论上应该在 `hdd` 类型的 `drives` 创建 `volumes`：

   ```YAML
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: cold-tier-pvc
   spec:
     volumeMode: Filesystem
     storageClassName: cold-tier-storage
     accessModes: [ "ReadWriteOnce" ]
     resources:
       requests:
         storage: 8Mi

   ---

   apiVersion: v1
   kind: Pod
   metadata:
     name: cold-tier-pv-pod
   spec:
     volumes:
       - name: cold-tier-pv-storage
         persistentVolumeClaim:
           claimName: cold-tier-pvc
     containers:
       - name: cold-tier-pv-container
         image: nginx
         ports:
           - containerPort: 80
             name: "http-server"
         volumeMounts:
           - mountPath: "/usr/share/nginx/html"
             name: cold-tier-pv-storage
   ```
10. 验证：

    ```Shell
    # 如下命令会指出各个 drive 上分别创建了多少个 volume
    kubectl directpv list drives

    # 如下命令会展示所创建的 volume
    kubectl directpv list volumes

    # 登录到 storage-1.ai-hub.aistar.info 节点上，也会看到创建了相应的 pvc 目录
    tree -L 2 /var/lib/directpv/mnt
    ```

## 参考

1. [MinIO - DirectPV](https://min.io/directpv)
