# RAID (Redundant Array Of Independence Disk)
Raid (Redundant Array Of Independent Disk) : est une technique de Virtualisation de stockage permettant de **regrouper plusieurs disques** afin _d'ameliorer_ les **performances ou la Tolerance aux pannes**.

| RAID 0 | RAID 1 | RAID 5 |
| --------- | --------- | --------- |
| Combine au moins deux disques :<br/> <br/> <ul><li>Les deux Disques Travaillent simultanement.</li> <br/> <li>Repartissant 50% des donnees sur un disque et 50% sur l'autre.</li> <br/> <li>performances deux fois plus élevée.</li></ul> | protection des données robuste  <br/>  <br/>  <ul><li>Le disque 1 contient les mêmes données que le disque 2.</li> <br/> <li>Ameliorer la securite de données</li></ul> |  l'équilibre entre protection des données et vitesse <br/><br/> <ul><li>Tolérance aux pannes</li><br/><li>Une partie pour le calcule</li><br/><li>performance elevee</li></ul> |

## Les etapes pour Creer  RAID 0 sous Linux
Lister les partitions disponibles :

```bash
$ lsblk -o NAME,SIZE,TYPE
NAME      SIZE    TYPE
sdb       931.5G  disk
└─sdb1    4G      part
sdc       931.5G  disk
└─sdc1    4G      part
```
Utilise la commande mdadm (multi-disk administrator) pour creer un RAID
```bash
# pour creer RAID 0 "--level=0"
$ mdadm --verbos --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb1 /dev/sdc1
mdadm: chunk size defaults to 512K
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```
Pour Verifier la creation du RAID
```bash
lsblk -o NAME,SIZE,TYPE
NAME      SIZE    TYPE
sdb       931.5G  disk
└─sdb1    4G      part
 └─md0    8G      raid0
sdc       931.5G  disk
└─sdc1    4G      part
 └─md0    8G      raid0

# On retrouve également des informations utiles dans /proc/mdstat :

$ cat /proc/mdstat
Personalities : [raid0]
md0 : active raid0 sdc1[1] sdb1[0]
      1952448512 blocks super 1.2 512k chunks

unused devices: <none>
```
**Pour utiliser le RAID, nous devons le formater avec un système de fichiers et le monter** <br/><br/>
_Formater avec le système de fichier ext4_
```bash
$ mkfs.ext4 /dev/md0
# ou
$ mkfs -t ext4 /dev/md0
```
 _Monter_
```bash
$ mkdir /mnt/raid0
$ mount /dev/md0 /mnt/raid0

# verifier
$ df -h /mnt/myraid
Filesystem      Size  Used Avail Use% Mounted on
/dev/md0        7.8G  24K  7.4G  1%   /mnt/raid0

# Ajouter une ligne à /etc/fstab pour rendre ce point de montage permanent
$ vim /etc/fstab
/dev/md0  /mnt/raid0  ext4  defaults  0 0
```
_Et si nous voulons supprimer notre raid_
```bash
# option -S = --Stop
$ mdadm -S /dev/md0
mdadm: stopped /dev/md0
```
## Creer RAID 5

```bash
$ mdadm --verbose --create /dev/md1 --level=5 --raid-devices=5 /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1 /dev/sdf1
mdadm: layout defaults to left-symmetric
mdadm: layout defaults to left-symmetric
mdadm: chunk size defaults to 512K
mdadm: size set to 4189184K
Continue creating array? y
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md1 started.
```
> Ensuite, nous pouvons `mkfs` et `mount` notre dernier RAID.

Mettre le disque sdc1 en panne <br/> <br/>
_Cela pourrait être utile si vous souhaitez simuler une panne ou si vous devez retirer temporairement un disque pour maintenance._
```bash
$ mdadm /dev/md1 -f /dev/sdc1
mdadm: set /dev/sdc1 faulty in /dev/md1
```
verifier sur le fichier /proc/mdstat
```bash
$ cat /proc/mdstat
Personalities : [raid0] [raid6] [raid5] [raid4]
md1 : active raid5 sdf1[5] sde1[3] sdd1[2] sdb1[1] sdc1[0](F)
      16756736 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/4] [_UUUU]

unused devices: <none>
```
> **Ici, nous voyons la partition que nous avons sélectionnée, marquée `(F)` pour échec.**

Utilise la commande commande `mdadm --detail`
```bash
$ mdadm --detail /dev/md1
/dev/md1:
           Version : 1.2
     Creation Time : Tue Aug 10 14:52:59 2021
        Raid Level : raid5
        Array Size : 16756736 (15.98 GiB 17.16 GB)
     Used Dev Size : 4189184 (4.00 GiB 4.29 GB)
      Raid Devices : 5
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Tue Aug 10 14:59:20 2021
             State : clean, degraded
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 1
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : salvage:1  (local to host salvage)
              UUID : 0c32834c:e5491814:94a4aa96:32d87024
            Events : 24

    Number   Major   Minor   RaidDevice State
       -       0        0        0      removed
       1       8       34        1      active sync   /dev/sdb1
       2       8       35        2      active sync   /dev/sdd1
       3       8       36        3      active sync   /dev/sde1
       5       8       37        4      active sync   /dev/sdf1

       0       8       33        -      faulty   /dev/sdc1
```

Nous pouvons supprimer notre lecteur défectueux et en ajouter un nouveau. <br/>

Tout d'abord, nous supprimons notre disque défectueux:
```bash
$ mdadm /dev/md1 --remove /dev/sdc1
mdadm: hot removed /dev/sdc1 from /dev/md1
```
Ensuite, nous remplaçons physiquement notre disque et ajoutons le nouveau
```bash
$ mdadm /dev/md1 -add /dev/sdc1
$ cat /proc/mdstat
Personalities : [raid0] [raid6] [raid5] [raid4]
md1 : active raid5 sdc1[6] sdf1[5] sde1[3] sdd1[2] sdb1[1]
      16756736 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/4] [_UUUU]
      [==>..................]  recovery = 10.7% (452572/4189184) finish=3.4min speed=18102K/sec
```
nous pouvons ajouter un `Spare Drive` dédié pour permettre à mdadm de basculer automatiquement vers
```bash
$ mdadm /dev/md1 --add-spare /dev/sdg1
mdadm: added /dev/sdg1

# Verifier
$ mdadm --detail /dev/md1 | grep spare
#       7       8       38        -      spare   /dev/sdg1
```
## supprimer un ensemble RAID sous Linux, vous pouvez suivre ces étapes
Démonter le système de fichiers RAID
```bash
$ umount /dev/md1
```
Arrêter l'ensemble RAID
```bash
$ mdadm --stop /dev/md1
```
Supprimer l'ensemble RAID
```bash
$ mdadm --remove /dev/md1
```
Supprimer les informations de configuration :
```bash
$ mdadm --zero-superblock /dev/sdb1
$ mdadm --zero-superblock /dev/sdc1
$ mdadm --zero-superblock /dev/sdd1
$ mdadm --zero-superblock /dev/sde1
$ mdadm --zero-superblock /dev/sdf1
$ mdadm --zero-superblock /dev/sdg1   
```
_Répétez cette commande pour chaque périphérique membre de l'ensemble RAID_
