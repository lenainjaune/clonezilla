# Clonezilla
Tout ce qui concerne ce superbe outil

# USB boot Clonezilla personnalisé
Créer une clé bootable en téléchargant la dernière version de CZ (en 32 ou 64 bits, en outrepassant un éventuel certificat Let's encrypt expiré), avec clavier FR, rend le système accessible par SSH et par HOSTNAME (Avahi), installe des outils utiles tel vim, exporte les disques locaux par NBD, exécute un script sur une 2ème partition en lecture/écriture pour permettre de modifier en LIVE session (persistance de données). 

Exporter ses disques locaux avec un serveur NBD permet de pouvoir les exploiter à distance depuis un client NBD. Voir mon travail sur NBD [ici](https://github.com/lenainjaune/nbd_remote_disk/blob/main/README.md) et voir un exemple complet de mise en place d'une installation client/server NBD [là](https://github.com/lenainjaune/nbd_remote_disk/blob/main/README.md#exemple-complet-clientserveur-).

Comment ça marche :
* Formater support : partition 1 pour Clonezilla, partition 2 pour données persistantes
* DL dernier ISO clonezilla stable réel
* Extraction du contenu ISO sur support et modifier la configuration du boot (DHCP et AZERTY)
* Extraction du système interne
* Personnalisation du système interne pour ajouter des outils, définir des réglages et permettre d'exécuter au boot un script externe
* Réintégration du système interne
* Ajout du script externe sur la partition persistante qui permettra d'outrepasser certains réglages et exécutera le script d'export NBD
* Ajout du script d'export de disque NBD
* Rendre support bootable

Astuce : on peut tester la clé sans rebooter avec QEmu/KVM (voir plus bas)

TODO : mettre paramètres longs pour plus explicatif

TODO : créer une clé USB Clonezilla USB pour 32-bits ET 64-bits (idéal : le même menu avec chargement de la LIVE session selon architecture)

TODO : démarrer SSH ailleurs que /etc/profile exécuté à chaque logon

TODO : empêcher de supprimer le script utilisateur (avec chattr ?)

TODO : si script utilisateur absent en recreer un par defaut ; si existe deja et que c'est un bien un fichier texte verifier que shebang en ligne 1

TODO : parfois on doit redémarrer avahi (hostname-2 !) => https://github.com/lathiat/avahi/issues/117 (tester en particulier **cache-entries-max=0** de @gramels qui contournerait mais empêcherait la résolution à son tour)

TODO : encapsuler code changement de mot de passe et changement de hostname (pour empecher erreur et simplifier utilisation)


TODO : bug nbd.conf : | grep /by-id/.*$dev$ | cut -d ' ' -f 1 \ -> grep /by-id/.*$dev$ | sort -r | find -n 1 | cut -d ' ' -f 1

```sh
# Définir la clé (utiliser aussi `dmesg -w` avant de brancher)
lsblk
DEV=/dev/sdb


# Dépendances pour mettre en place (Debian) en SU
apt update && apt install -y rsync libc6-i386 mtools squashfs-tools parted wget dosfstools

# Config
DNS=8.8.8.8
HOSTNAME_USB=CZ-LIVE
PART_RW_NAME=casper-rw
## 'PASS' protège PASS d'une expansion non voulue de * ou ! etc.
PASS='P455w0rd!*'
OUTER_1ST_SCRIPT=ran_first.sh
SCRIPT_NAME_NBD_EXPORT=export_all_local_disk.sh
#ISO_ARCH="(i386|i486|i686)"
ISO_ARCH=64
#USB_PART1_SIZE=1G
USB_PART1_SIZE=90%
USB_PART2_SIZE=100%


# Préparer support

parted -a cylinder $DEV -s "mklabel msdos" \
 -s "mkpart primary fat32 1 $USB_PART1_SIZE" \
 -s "mkpart primary ext4 $USB_PART1_SIZE $USB_PART2_SIZE"
mkfs.vfat -n $HOSTNAME_USB -I -F 32 ${DEV}1
mkfs.ext4 -F -L $PART_RW_NAME ${DEV}2

# Nota : il manque qqc ; lsblk n'actualise pas FSTYPE,FSVER,LABEL de DEV
#  => on le voit si formatage précédent par "dd ISO /dev/$DEV" 


# Dernier ISO stable réel (Nécessite mtools pour nouvelles versions)

ISO_DL_SOURCE=\
https://sourceforge.net/projects/clonezilla/files/clonezilla_live_stable
cz_latest_dl_folder=$(
 wget --no-check-certificate -qO- $ISO_DL_SOURCE \
 | grep -Eo "https?://[^ \"]*sourceforge[^ \"]*/[-0-9.]+/" \
 | sort -rV | head -n1
)
cz_latest_dl_version=$(
 echo $cz_latest_dl_folder | grep -Eo "[-0-9.]{2,}" 
)
cz_latest_iso=$(
 wget --no-check-certificate -qO- $cz_latest_dl_folder \
 | grep -iEo "https?://[^ \"]*sourceforge[^ \"]*\.iso" \
 | grep -E $cz_latest_dl_version | grep -E "$ISO_ARCH[^-]" | sort -u
)
wget --no-check-certificate $cz_latest_iso -O cz_latest.iso


# Extraire ISO dans USB

mkdir -p /mnt/CLONEZILLA/ /mnt/$PART_RW_NAME/ /mnt/ISO
mount -o loop cz_latest.iso /mnt/ISO/
mount ${DEV}1 /mnt/CLONEZILLA/
mount ${DEV}2 /mnt/$PART_RW_NAME/
rsync -aP /mnt/ISO/ /mnt/CLONEZILLA/
umount -l /mnt/ISO
rm cz_latest.iso
rmdir /mnt/ISO


# Booter en FR et pré-exécuter dhclient

sed -i -e '/^ *append initrd=/s/locales=/locales=fr_FR.UTF-8/' \
 -e 's/keyboard-layouts=/keyboard-layouts=fr/' \
 -e 's/enforcing=0 /enforcing=0  ocs_prerun="dhclient" /' \
 /mnt/CLONEZILLA/syslinux/syslinux.cfg
mount -t squashfs -o loop /mnt/CLONEZILLA/live/filesystem.squashfs /mnt
mkdir squashfs
rsync -aP /mnt/ squashfs


# Personnaliser système interne

for f in /proc /sys ; do mount -B $f squashfs$f ; done ;
mount -t devpts none squashfs/dev/pts

cat << EOC | PART_RW_NAME=$PART_RW_NAME DNS=$DNS PASS=$PASS \
  HOSTNAME_USB=$HOSTNAME_USB OUTER_1ST_SCRIPT=$OUTER_1ST_SCRIPT \
  chroot squashfs

 INNER_1ST_SCRIPT=inner_1st_script.sh
 INNER_1ST_SERVICE=run_1st_script.service
 APP="vim avahi-daemon dnsutils winbind libnss-winbind libnss-mdns \
  memtester nbd-server"

 echo \$HOSTNAME_USB > /etc/hostname

 a=\$(
  echo nameserver \$DNS
  cat /etc/resolv.conf \\
   | grep -vE ^[[:space:]]*nameserver[[:space:]]+\$DNS[[:space:]]*$
 )
 echo -e "\$a" > /etc/resolv.conf

 apt update && apt install -y \$APP

 a=\$(
  cat /etc/profile \\
   | grep -vE ^[[:space:]]*systemctl[[:space:]]+restart[[:space:]]+ssh
  echo systemctl restart ssh
 )
 echo -e "\$a" > /etc/profile
 a=\$(
  cat /etc/ssh/sshd_config | grep -vE ^[[:space:]]*PermitRootLogin
  echo PermitRootLogin yes
 )
 echo -e "\$a" > /etc/ssh/sshd_config
 systemctl enable ssh
 

 # Service déclencheur (voir [1])
 
 cat << EOF > /etc/systemd/system/\$INNER_1ST_SERVICE

 [Unit]
 Description=Exécute le premier script interne

 [Service]
 Type=Simple
 ExecStart=/bin/bash /\$INNER_1ST_SCRIPT
 # ExecStart=/usr/bin/sudo /bin/bash /\$INNER_1ST_SCRIPT

 [Install]
 WantedBy=multi-user.target
 
EOF

 chmod 644 /etc/systemd/system/\$INNER_1ST_SERVICE
 systemctl enable \$INNER_1ST_SERVICE


 cat << EOF > \$INNER_1ST_SCRIPT
#!/bin/bash

 # echo -e "127.0.0.1\tlocalhost\n127.0.1.1\t$HOSTNAME_USB" > /etc/hosts

 echo -e "\$PASS\n\$PASS" | passwd user
 echo -e "\$PASS\n\$PASS" | passwd root

 ln -s /usr/lib/live/mount/medium /root/usb_root
 ln -s /usr/lib/live/mount/medium /home/user/usb_root

 part=\\\$(
  lsblk -nlo NAME,LABEL | grep \$PART_RW_NAME$ | cut -d ' ' -f 1
 )
 mount /dev/\\\$part /mnt
 /mnt/\$OUTER_1ST_SCRIPT

EOF

 chmod +x \$INNER_1ST_SCRIPT
 
EOC

for f in /proc /sys /dev/pts ; do umount -lf squashfs$f ; done


# Intégrer système interne personnalisé

mksquashfs squashfs filesystem.squashfs -info
umount -l /mnt/
chroot squashfs dpkg-query -W --showformat='${Package}  ${Version}\n' \
 > /mnt/CLONEZILLA/filesystem.manifest
rm -rf squashfs
rsync -P filesystem.squashfs /mnt/CLONEZILLA/live \
 && rm filesystem.squashfs
chmod 444 /mnt/CLONEZILLA/live/filesystem.squashfs


# Script à exécuter en 1er ; peut être modifié en LIVE session

cat << EOF > /mnt/$PART_RW_NAME/$OUTER_1ST_SCRIPT
#!/bin/bash

 # touch /touched
 # => OK


 # Rédéfinir
 

 # Host Name pour multiple Clonezilla distants (defaut : $HOSTNAME_USB)
 # Laisser vide pour ne pas redéfinir

 NEW_HOSTNAME_USB=


 # Mot de passe de session (defaut : '$PASS')
 # Laisser '' pour ne pas redéfinir
 # Nota : 'PASS' protège PASS d'une expansion non voulue de * ou ! etc.
 
 NEW_PASS=''


 if [[ ! -z \$NEW_HOSTNAME_USB ]] ; then
  sed -i "s/$HOSTNAME_USB/\$NEW_HOSTNAME_USB/g" /etc/hosts
  echo \$NEW_HOSTNAME_USB > /etc/hostname
  hostnamectl set-hostname \$NEW_HOSTNAME_USB
  systemctl restart avahi-daemon
 fi

 if [[ ! \$NEW_PASS == '' ]] ; then
  # echo \$NEW_PASS > /touched
  echo -e "\$NEW_PASS\n\$NEW_PASS" | passwd user
  echo -e "\$NEW_PASS\n\$NEW_PASS" | passwd root
  # echo retour passwd : $?
 fi

 /usr/lib/live/mount/medium/$SCRIPT_NAME_NBD_EXPORT

EOF

chmod +x /mnt/$PART_RW_NAME/$OUTER_1ST_SCRIPT


# Exporter disques locaux
# Nota : heredoc avec variables non expandées

cat << 'EOF' > /mnt/CLONEZILLA/$SCRIPT_NAME_NBD_EXPORT
#!/bin/bash

 if [[ $( netstat -atnp | grep nbd-server | grep ESTABLISHED ) ]] ; then
  msg="Erreur : les disques sont déjà importés à distance"
  msg="$msg\nVoici la liste :"
  msg="$msg\n$( netstat -atnp | grep nbd-server | grep ESTABLISHED )"
  echo -e "$msg"
  exit 1
 fi

 modprobe -r nbd
 pkill nbd-server


 # Configurer disques importables à distance
 
 (
 
  echo -e "[generic]\nuser=root\ngroup=root\nallowlist=true"
  
  for e in $( \
    lsblk -nd -o NAME,SIZE | awk '$2 != "0B" { print $1 "|" $2 }' \
  ) ; do
  
   dev=$( echo $e | cut -d '|' -f 1 )
   id=$(
    find /dev -type l \
         -exec bash -c 'l={} ; echo $l $( readlink $l )' \; \
    | grep /by-id/.*$dev$ | sort -r | find -n 1 | cut -d ' ' -f 1 \
    | rev | cut -d '/' -f 1 | rev
   )
   size=$( echo $e | cut -d '|' -f 2 )
   echo -e "[$dev@$id@$size]\nexportname=/dev/$dev"
  
  done

 ) > /nbd.conf
 
 modprobe nbd

 # Je ne sais pas pourquoi mais il faut le démarrer avec -d (daemon ?)
 nbd-server -C /nbd.conf -d

EOF

chmod +x /mnt/CLONEZILLA/$SCRIPT_NAME_NBD_EXPORT


# Rendre clé bootable
bash /mnt/CLONEZILLA/utils/linux/makeboot.sh -L $HOSTNAME_USB -b ${DEV}1

umount -l /mnt/CLONEZILLA /mnt/$PART_RW_NAME
rmdir /mnt/CLONEZILLA /mnt/$PART_RW_NAME
```
[1] utiliser les services pour exécuter un script au boot ([source](https://unix.stackexchange.com/questions/529111/execute-a-script-before-login-screen/529183#529183))

## Tester avec QEmu/KVM
```sh
DEV=/dev/sdg

root@host:~# kvm -m 4096 -hda $DEV
# WARNING: Image format was not specified for '/dev/sdg' and probing guessed raw.
#         Automatically detecting the format is dangerous for raw images, write operations on block 0 will be restricted.
#         Specify the 'raw' format explicitly to remove the restrictions.
# => la fenêtre d'accueil devrait s'afficher
```
