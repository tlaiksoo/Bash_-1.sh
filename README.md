#1/bin/bash
#Autor: Tarmo Laiksoo
#Kirjeldus: Esimene kodutöö Bash'is

#Kontrollib, kas kasutaja on administratori õigustega
if [ $UID -ne 0 ]
then
	echo "käivita skript $(basename $0) administraatori õigustega"
	exit 1
fi

#Kontrollib parameetrite arvu (2 või 3)
if [ $# -eq 2 ]
then 
	SHARE=$(basename $1)
else if [ $# -eq 3 ]
	then	
		SHARE=$3	
	else
		echo "Käivita skript $0 KAUST GRUPP [SHARE]"
		exit 1
	fi
fi	

KAUST=$1
GRUPP=$2

#Ütleb kasutajale mida skript teeb
echo "Jaga kausta $KAUST grupile $GRUPP shaerina $SHARE"

#Kontrollib, kas failijagamisteenus Samba on paigaldatud (vajadusel paigaldab)
type smbd > /dev/null 2>&1 || apt-get update > /dev/null 2>&1 && apt-get install samba -y > /dev/null 2>&1

#Kontrollib, kas kaust on olemas (vajadusel loob)
test -d $KAUST || mkdir -p $KAUST || exit

# chgrop
chgrp $GRUPP $KAUST

# chmod g+w
chmod g+w $KAUST

#Kontrollib, kas grupp on olemas (vajadusel loob)
getent group $GRUPP > /dev/null || addgroup $GRUPP > /dev/null

#Teeb varukoopia confi failist
cp -n /etc/samba/smb.conf /etc/samba/samba.conf

#Kontrollib, kas share on juba loodud
grep "\[$SHARE\]" /etc/samba/samba.conf && echo "Share $SHARE on juba olemas!" && exit 1

#Kirjutab konfifaili
cat >> /etc/samba/samba.conf << LÕPP
[$SHARE]
 path=$KAUST
 writable=yes
 valid users=@$GRUPP
 force group=$GRUPP
 read only = no
 create mask = 700
 directory mask = 775

LÕPP

#Kontroll, kas konfiguratsioon on OK
testparm -s /etc/samba/samba.conf > /dev/null
if [ $? -ne 0 ]
then 
	echo "Skript vigane!"
exit 1
#Kui on õige siis konfiguratsioon olemasoleva asemele
else
  cp /etc/samba/samba.conf /etc/samba/smb.conf
fi

#Samaba teenusele teha reload
service smbd reload > /dev/null

exit 0
