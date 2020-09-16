## Περιγραφή

Το [```OpenNMS```](https://www.opennms.com/) είναι μια πλατφόρμα παρακολούθησης και διαχείρησης δικτύων. Θα χρησιμοποιήσουμε τη λειτουργία ανακάλυψης (Discovery Scan) με σκοπό να ανακαλύψουμε κόμβους στο δίκτυό μας και με χρήση του πρωτοκόλλου SNMP να λάβουμε πληροφορίες σχετικά με το κάθε κόμβο που συναντάμε. Το δίκτυο αυτό θα είναι ένα προσομοιωμένο δίκτυο μέσα από το IMUNES.

Το [```IMUNES```](http://imunes.net/) είναι ένας εξομοιωτής/προσομοιωτής δικτύου ο οποίος χρησιμοποιεί το [```Quagga```](https://www.quagga.net/). Το Quagga είναι ένα πακέτο λογισμικού δρομολόγησης. Για κάθε εικονικό κόμβο δικτύου, το IMUNES τρέχει ένα docker ```container``` το οποίο κάνει χρήση ενός έτοιμου docker ```image```, το [```imunes/template```](https://hub.docker.com/r/imunes/template) image.

Το έγγραφο αυτό παρουσιάζει τα απαραίτητα βήματα για την:
- εγκατάσταση IMUNES
- επεξεργασία του ```imunes/template``` docker image
- ρύθμιση και εγκατάσταση Quagga με υποστήριξη πρωτοκόλλου SNMP
- ρύθμιση υπηρεσίας πρωτοκόλλου SNMP
- αποθήκευση αλλαγών στο ```imunes/template``` docker image
- εγκατάσταση και ρύθμιση OpenNMS
- δημιουργία πειραματικού δικτύου στο IMUNES
- χρήση λειτουργίας OpenNMS Single Discovery Scan για την αναζήτηση κόμβων στο πειραματικό δίκτυο

## Περιβάλλον Εγκατάστασης
Το λειτουργικό σύστημα που χρησιμοποιήθηκε για αυτή την εργασία είναι το *Xubuntu 20.04.1 LTS (Focal Fossa) 64-bit* σε εικονικό μηχάνημα *VirtualBox*. Στο σύστημα αυτό ο χρήστης ```user``` έχει δικαιώματα root και το όνομα του τερματικού είναι ```msnlab```. Η πρόσβαση στο docker container θα εμφανίζεται ώς ```root@msnlab``` καθώς το container χρησιμοποιεί το όνομα του συστήματος ως δικό του μιας και μοιραζόνται την ίδια σύνδεση δικτύου.

## Εγκατάσταση IMUNES
Αρχικά, ενημερώνουμε και εγκαθιστούμε τυχόν αναβαθμίσεις στο σύστημα.

```console
user@msnlab:~$ sudo apt update && sudo apt dist-upgrade -y
```

Εγκαθιστούμε τα απαραίτητα πακέτα λογισμικού για το IMUNES.
```console
user@msnlab:~$ sudo apt install git openvswitch-switch docker.io xterm wireshark make imagemagick tk tcllib util-linux
```

Κατεβάζουμε τον πηγαίο κώδικα του IMUNES από το GitHub και μεταβαίνουμε στον φάκελο ```imunes/```
```console
user@msnlab:~$ git clone https://github.com/imunes/imunes.git && cd imunes/
```

Εκτελούμε ```sudo make install``` για να κάνουμε εγκατάσταση του IMUNES στο σύστημά μας.
```console
user@msnlab:~/imunes$ sudo make install
```

Έπειτα αρχικοποιούμε το IMUNES με την παρακάτω εντολή.
Η παράμετρος ```-p``` θα προετοιμάσει το *virtual root file system* και θα κατεβάσει το ```imunes/template``` docker image τοπικά, ένα image το οποίο είναι αναγκαίο και εκτελείται σε κάθε εικονικό κόμβο πειραματικού δικτύου μέσα στο IMUNES.

```console
user@msnlab:~$ sudo imunes -p
```

Πλέον, μπορούμε να το τρέξουμε ως εξής:
```console
user@msnlab:~$ sudo imunes
```

Στο σημείο αυτό έχουμε ολοκληρώσει την εγκατάσταση του IMUNES και το επόμενο βήμα είναι να ρυθμίσουμε το Quagga έτσι ώστε να υποστηρίζει το πρωτόκολλο SNMP.

## Επεξεργασία του ```imunes/template``` docker image
Επιβεβαιώνουμε πως διαθέτουμε τοπικά το ```imunes/template``` docker image
```console
user@msnlab:~$ sudo docker image ls
REPOSITORY          TAG         IMAGE ID            CREATED SIZE
imunes/template     latest      28ab347bef71        934MB
```

Εκτελούμε το image αυτό σε ένα container ως εξής:
```console
user@msnlab:~$ sudo docker run --detach --tty --net='host' imunes/template
```
Το container αυτό θα έχει πρόσβαση στο ίδιο δίκτυο με τον host διότι χρειαζόμαστε πρόσβαση στο διαδίκτυο για την εγκατάσταση επιπλέον λογισμικού μέσα στο container και αυτό το επιτυχγάνουμε με τη παράμετρο ```--net='host'```.

Επιβεβαιώνουμε πως το ```container``` δημιουργήθηκε επιτυχώς.
```console
user@msnlab:~$ sudo docker container ls
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
d3cdf12c8d50        c009531c75fd        "/bin/bash"         10 seconds ago      Up 10 seconds                             clever_babbage
```

Από το αποτέλεσμα της παραπάνω εντολής θα πρέπει να λάβουμε πληροφορίες σχετικά με το ```container``` που δημιουργήθηκε. Ιδιάιτερα, χρειαζόμαστε το αναγνωριστικό ```CONTAINER-ID```, σε αυτή τη περίπτωση αυτό είναι το ```d3cdf12c8d50```.

Αποκτάμε πρόσβαση στο ```container``` ως εξής:
```console
user@msnlab:~$ sudo docker exec -u root -t -i d3cdf12c8d50 /bin/bash
root@msnlab:~#
```
Εκτελώντας τη παραπάνω εντολή έχει ως αποτέλεσμα να αποκτήσουμε ```root``` πρόσβαση στο shell του container.

## Ρύθμιση και εγκατάσταση του Quagga με υποστήριξη πρωτοκόλλου SNMP
Ενημερώνουμε τα τοπικά αποθετήρια και εγκαθιστούμε τυχόν ενημερώσεις στο σύστημα
```console
root@msnlab:~# apt update && apt dist-upgrade -y 
```

Εγκαθιστούμε όλα τα απαραίτητα πακέτα λογισμικού (required package dependencies)
```console
root@msnlab:/# apt install git make snmp snmpd snmptrapd snmp-mibs-downloader automake autoconf libtool texinfo gawk pkg-config libreadline-dev libc-ares-dev libsnmp-dev
```

Κατεβάζουμε τον πηγαίο κώδικα του Quagga από το αποθετήριο και μεταβαίνουμε στον φάκελο ```quagga/```
```console
root@msnlab:/# git clone https://git.savannah.gnu.org/git/quagga.git && cd quagga/
```
Θέτουμε την μεταβλητή περιβάλλοντος ```LD_LIBRARY_PATH``` με σκοπό ο μεταγγλωτιστής να γνωρίζει πως οι βιβλιοθήκες σε αυτό το ```container``` βρίσκονται στο φάκελο ```/usr/local/lib```
```console
root@msnlab:/quagga# export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib
```
Βάσει των οδηγιών από το [INSTALL.quagga.txt](http://git.savannah.gnu.org/cgit/quagga.git/tree/INSTALL.quagga.txt) εκτελούμε τα εξής ώστε να αρχικοποιήσουμε τα απαραίτητα αρχεία για την εγκατάσταση.
```console
root@msnlab:/quagga# automake --add-missing
root@msnlab:/quagga# ./bootstrap.sh
```
Επίσης, πρέπει να προσθέσουμε επιπλέον μια μεταβλητή περιβάλλοντος με σκοπό να ρυθμίσουμε το ```Quagga``` να χρησιμοποιεί τις βιβλιοθήκες που βρίσκονται στο φάκελο ```/usr/local/lib```.
```console
root@msnlab:/quagga# export LDFLAGS='-L/usr/local/lib' && ldconfig
```
Έπειτα ρυθμίζουμε το Quagga να κάνει χρήση του πρωτοκόλλου ```SNMP``` και να τρέχει ως χρήστης ```root``` καθώς και να διαβάζει τις ρυθμίσεις από το φάκελο ```/etc/quagga```.
```console
root@msnlab:/quagga# ./configure --enable-snmp=agentx --enable-user=root --enable-group=root --sysconfdir=/etc/quagga
```
Μετά την επιτυχή ρύθμιση, προχωράμε στην εγκατάσταση.
```console
root@msnlab:/quagga# make install
```

 >Στη περίπτωση που δεν γίνει σωστά η εγκατάσταση του Quagga, σιγουρευόμαστε η μεταβλητές περιβάλλοντος LD_LIBRARY_PATH και LDFLAGS να έχουν λάβει τη σωστή τιμή και έπειτα τρέχουμε ξανά το ```make install```.

## Ρύθμιση υπηρεσίας πρωτοκόλλου SNMP
Μετά την εγκατάσταση του Quagga προχωράμε στις ρυθμίσεις της υπηρεσίας SNMP.

Το πρώτο αρχείο που θα πρέπει να επεξεργαστούμε είναι το ```/etc/snmp/snmpd.conf```, όπου και βρίσκονται οι κύριες ρυθμίσεις της υπηρεσίας SNMP.
Ανοίγουμε το αρχείο αυτό με το ```nano```.
```console
root@msnlab:/quagga# nano /etc/snmp/snmpd.conf
```
Βάζουμε σε σχόλια τα παρακάτω πεδία:
- agentAddress  udp:127.0.0.1:161
- extend    test1
- extend-sh test2

Τροποποιούμε το πεδίο ```sysLocation``` από την προεπιλεγμένη τιμή σε:
- sysLocation       University of Macedonia

Στο τέλος του αρχείου, βρίσκουμε τη γραμμή ```master agentx``` και προσθέτουμε τα παρακάτω:
- agentXSocket /var/agentx/master
- agentXPerms 777 777

Ουσιαστικά προσθέσαμε επιπλέον παραμέτρους ώστε η υπηρεσία SNMP να λειτουργεί στο ```/var/agentx/master``` socket.

Το επόμενο αρχείο ρυθμίσεων που πρέπει να επεξεργαστούμε είναι το ```/etc/default/snmpd```.
```console
root@msnlab:/quagga# nano /etc/default/snmpd
```
Βάζουμε σε σχόλια το παρακάτω:
- export MIBS=

Αλλάζουμε το ```SNMPDOPTS``` ως εξής:
- SNMPDOPTS='-Lsd -Lf /tmp/snmpd.log -u root -g root -I -agentx -p /var/run/snmpd.pid -c /etc/snmp/snmpd.conf'

Ουσιαστικά ενημερώνουμε την υπηρεσία να κρατά logs στο αρχείο ```/tmp/snmpd.log```, να τρέχει με τα δικαιώματα του χρήστη ```root``` και να χρησιμοποιεί ως αρχείο ρυθμίσεων, το ```/etc/snmp/snmpd.conf```.

Μένει να ρυθμίσουμε την υπηρεσία SNMP να ξεκινά κάθε φορά που ξεκινά και το docker container.
Στο αρχείο ```/etc/bash.bashrc``` προσθέτουμε το παρακάτω:
```console
root@msnlab:/quagga# nano /etc/bash.bashrc
```

- \#Enable SNMP service on start-up
- /usr/sbin/service snmpd start

Κάνουμε επανεκκίνηση της υπηρεσίας SNMP
```console
root@msnlab:/quagga# service snmpd restart
```
Εαν όλα λειτουργούν σωστά, με την εντολή ```snmpwalk -v 1 -c public 127.0.0.1``` θα πρέπει να λάβουμε αρκετές πληροφορίες σχετικά με το σύστημα μέσω της υπηρεσίας SNMP.
Όπου:
- το ```-v 1``` είναι η έκδοση του SNMP που χρησιμοποιούμε για αυτή τη δοκιμή
- το ```-c public``` είναι το community string και μπορεί να βρεθεί στο αρχείο ```/etc/snmp/snmpd.conf```
- και τέλος, το ```127.0.0.1``` είναι η IPv4 διέυθυνση του συστήματος του οποίου προσπαθούμε να επικοινωνήσουμε με την SNMP υπηρεσία του. Μπορεί να είναι οποιοσδήποτε άλλος κόμβος που προσφέρει την υπηρεσία SNMP αλλά σε αυτή τη περίπτωση είναι το ```localhost```.

```console
root@msnlab:/quagga# snmpwalk -v 1 -c public 127.0.0.1
iso.3.6.1.2.1.1.1.0 = STRING: "Linux msnlab 5.4.0-42-generic #46-Ubuntu SMP Fri Jul 10 00:24:02 UTC 2020 x86_64"
iso.3.6.1.2.1.1.2.0 = OID: iso.3.6.1.4.1.8072.3.2.10
iso.3.6.1.2.1.1.3.0 = Timeticks: (8074890) 22:25:48.90
iso.3.6.1.2.1.1.4.0 = STRING: "Me <me@example.org>"
iso.3.6.1.2.1.1.5.0 = STRING: "msnlab"
iso.3.6.1.2.1.1.6.0 = STRING: "University of Macedonia"
iso.3.6.1.2.1.1.7.0 = INTEGER: 72
iso.3.6.1.2.1.1.8.0 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.2.1 = OID: iso.3.6.1.6.3.11.3.1.1
iso.3.6.1.2.1.1.9.1.2.2 = OID: iso.3.6.1.6.3.15.2.1.1
iso.3.6.1.2.1.1.9.1.2.3 = OID: iso.3.6.1.6.3.10.3.1.1
iso.3.6.1.2.1.1.9.1.2.4 = OID: iso.3.6.1.6.3.1
iso.3.6.1.2.1.1.9.1.2.5 = OID: iso.3.6.1.6.3.16.2.2.1
iso.3.6.1.2.1.1.9.1.2.6 = OID: iso.3.6.1.2.1.49
iso.3.6.1.2.1.1.9.1.2.7 = OID: iso.3.6.1.2.1.4
iso.3.6.1.2.1.1.9.1.2.8 = OID: iso.3.6.1.2.1.50
iso.3.6.1.2.1.1.9.1.2.9 = OID: iso.3.6.1.6.3.13.3.1.3
iso.3.6.1.2.1.1.9.1.2.10 = OID: iso.3.6.1.2.1.92
iso.3.6.1.2.1.1.9.1.3.1 = STRING: "The MIB for Message Processing and Dispatching."
iso.3.6.1.2.1.1.9.1.3.2 = STRING: "The management information definitions for the SNMP User-based Security Model."
iso.3.6.1.2.1.1.9.1.3.3 = STRING: "The SNMP Management Architecture MIB."
iso.3.6.1.2.1.1.9.1.3.4 = STRING: "The MIB module for SNMPv2 entities"
iso.3.6.1.2.1.1.9.1.3.5 = STRING: "View-based Access Control Model for SNMP."
iso.3.6.1.2.1.1.9.1.3.6 = STRING: "The MIB module for managing TCP implementations"
iso.3.6.1.2.1.1.9.1.3.7 = STRING: "The MIB module for managing IP and ICMP implementations"
iso.3.6.1.2.1.1.9.1.3.8 = STRING: "The MIB module for managing UDP implementations"
iso.3.6.1.2.1.1.9.1.3.9 = STRING: "The MIB modules for managing SNMP Notification, plus filtering."
iso.3.6.1.2.1.1.9.1.3.10 = STRING: "The MIB module for logging SNMP Notifications."
iso.3.6.1.2.1.1.9.1.4.1 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.2 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.3 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.4 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.5 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.6 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.7 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.8 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.9 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.10 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.25.1.1.0 = Timeticks: (42810291) 4 days, 22:55:02.91
iso.3.6.1.2.1.25.1.2.0 = Hex-STRING: 07 E4 08 1A 10 02 1C 00 2B 00 00 
iso.3.6.1.2.1.25.1.3.0 = INTEGER: 393216
iso.3.6.1.2.1.25.1.4.0 = STRING: "BOOT_IMAGE=/boot/vmlinuz-5.4.0-42-generic root=UUID=08550881-62ca-4260-98c1-dad2b7f21ed9 ro quiet splash
"
iso.3.6.1.2.1.25.1.5.0 = Gauge32: 0
iso.3.6.1.2.1.25.1.6.0 = Gauge32: 4
iso.3.6.1.2.1.25.1.7.0 = INTEGER: 0
End of MIB
```

## Αποθήκευση αλλαγών στο ```imunes/template``` docker image
Εφόσον ρυθμίσαμε το Quagga σωστά με υποστήριξη SNMP, το μόνο που απομένει είναι να αποθηκέυσουμε τις αλλαγές που κάναμε σε αυτό το container στο κύριο docker image που χρησιμοποιεί το IMUNES με σκοπό ο κάθε κόμβος στο IMUNES να διαθέτει την υπηρεσία SNMP και τις αλλαγές που μόλις κάναμε.

Τερματίζουμε τη σύνδεση μας με το docker container.
```console
root@msnlab:/quagga# exit
```
Υποβάλλουμε τις αλλαγές του container στο ```imunes/template``` docker image.
```console
user@msnlab:~$ sudo docker commit d3cdf12c8d50 imunes/template:latest
```
Πλέον ο κάθε εικονικός κόμβος στο IMUNES θα εμπεριέχει τις αλλαγές που κάναμε. Δηλαδή, ο κάθε κόμβος θα διαθέτει την υπηρεσία SNMP μέσω της οποίας θα μπορούμε να λαμβάνουμε πληροφορίες συστήματος μέσα από το OpenNMS.

## Εναλλακτική εκγατάστασης ```imunes/template```
Μια εναλλακτική πρόταση για την απόκτηση του ```imunes/template``` image με προρυθμισμένη την υπηρεσία SNMP είναι να την κατεβάσουμε από το αποθετήριο https://github.com/iamaldi/imunes-snmp/packages/373676?version=latest

Το image αυτό έχει δημιουργηθεί με τα ίδια βήματα που περιγράψαμε παραπάνω, απλά υπάρχει ώστε να γλυτώνει χρόνο εγκατάστασης για τους υπόλοιπους πιθανούς χρήστες.

## Εγκατάσταση και ρύθμιση του OpenNMS
Η εγκατάσταση του OpenNMS είναι σχετικά απλή και μπορούμε απλά να ακολουθούμε τα βήματα στον οδηγό εγκατάστασης που βρίσκεται στο https://docs.opennms.org/opennms/branches/develop/guide-install/guide-install.html#_installing_on_debian

Εγκαθιστούμε το Java OpenJDK 11.
```console
user@msnlab:~$ sudo apt install openjdk-11-jdk
```

Προσθέτουμε τα απαραίτητα αποθετήρια.
```console
user@msnlab:~$ sudo cat << EOF | sudo tee /etc/apt/sources.list.d/opennms.list
> deb https://debian.opennms.org stable main
> deb-src https://debian.opennms.org stable main
> EOF
user@msnlab:~$ wget -O - https://debian.opennms.org/OPENNMS-GPG-KEY | sudo apt-key add -
user@msnlab:~$ sudo add-apt-repository ppa:willat8/shepherd
user@msnlab:~$ sudp apt-get update
```

Μέσω του apt-get εγκαθιστούμε το OpenNMS και τα απαραίτητα πακέτα λογισμικού που έρχονται με αυτό.
```console
user@msnlab:~$ sudo apt-get -y install opennms
```
Μετά την επιτυχής εγκατάσταση του OpenNMS, σειρά έχει να ρυθμίσουμε την βάση δεδομένων PostgreSQL.

Ξεκινάμε την υπηρεσία PostgreSQL
```console
user@msnlab:~$ sudo systemctl start postgresql
```
Αλλάζουμε τον χρήστη μας σε ```postgresql```
```console
user@msnlab:~$ sudo su postgress
```

Ως χρήστης ```postgresql```, με τις εντολές ```createuser``` και ```createdb``` προσθέτουμε έναν νέο χρήστη και δημιουργούμε μια νέα βάση ```opennms``` στη βάση δεδομένων PostgreSQL.
```console
postgres@msnlab:/home/user$ createuser -P opennms
Enter password for new role: 
Enter it again: 
postgres@msnlab:/home/user$ createdb -O opennms opennms
```

Έπειτα αλλάζουμε τον default κωδικό του χρήστη postgresql στη βάση και κάνουμε έξοδο από τον λογαριασμό χρήστη ```postgresql``` με ```exit```.
```console
postgres@msnlab:/home/user$ psql -c "ALTER USER postgresql WITH PASSWORD 'msnlabsecretpass';"
postgres@msnlab:/home/user$ exit
user@msnlab:~$
```

Καθώς αλλάξαμε τον κωδικό του χρήστη ```postgresql``` θα πρέπει να ενημερώσουμε το αρχείο ρυθμίσεων ```/etc/opennms/opennms-datasources.xml``` του OpenNMS με τα κατάλληλα δεδομένα.
```console
user@msnlab:~$ sudo nano /etc/opennms/opennms-datasources.xml
```

Σε αυτή τη περίπτωση το αρχείο θα πρέπει να μοιάζει με το παρακάτω.
```xml
<jdbc-data-source name="opennms"
                    database-name="opennms"
                    class-name="org.postgresql.Driver"
                    url="jdbc:postgresql://localhost:5432/opennms"
                    user-name="opennms"
                    password="opennms" />

<jdbc-data-source name="opennms-admin"
                    database-name="template1"
                    class-name="org.postgresql.Driver"
                    url="jdbc:postgresql://localhost:5432/template1"
                    user-name="postgres"
                    password="msnlabsecretpass" />
```

Αναζητούμε το περιβάλλον της Java και το αποθηκεύουμε στις ρυθμίσεις του OpenNMS.
```console
user@msnlab:~$ sudo /usr/share/opennms/bin/runjava -s
```

Αρχικοποιούμε τη βάση δεδομένων και ανιχνεύουμε τυχόν βιβλιοθήκες συστήματος και αποθηκέυουμε τις ρυθμίσεις αυτές στο OpenNMS.
```console
user@msnlab:~$ sudo /usr/share/opennms/bin/install -dis
```
Ενεργοποιούμε την υπηρεσία του OpenNMS να ξεκινά μαζί με το λειτουργικό σύστημα.
```console
user@msnlab:~$ sudo systemctl enable opennms
```

Ξεκινάμε την υπηρεσία OpenNMS χειροκίνητα τη πρώτη φορά.
```console
user@msnlab:~$ sudo systemctl start opennms
```
Ο πίνακας διαχείρισης θα πρέπει να είναι πλέον προσβάσιμος μέσα από τον εξής σύνδεσμο, http://localhost:8980/opennms.

Συνδεόμαστε με τα διαπιστευτήρια ```admin/admin``` και αλλάζουμε τον κωδικό πρόσβασης με έναν πιο ισχυρό.

## Δημιουργία πειραματικού δικτύου στο IMUNES

Σε αυτό το βήμα μένει να δημιουργήσουμε ένα πειραματικό δίκτυο στο IMUNES και να ρυθμίσουμε το OpenNMS να συλλέγει δεδομένα πάνω σε αυτό μέσω του πρωτοκόλλου SNMP.

Τρέχουμε το IMUNES.
```console
user@msnlab:~$ sudo imunes
```

Δημιουργούμε το εξής δίκτυο όπως φαίνεται και στην παρακάτω εικόνα και εκτελούμε το πείραμα (```Experiment``` -> ```Execute```). Το αρχείο αυτού του δικτύου μπορείτε να το βρείτε [εδώ](imunes-test-net.imn).
![IMUNES Test Network](imunes-experimet-network.jpg)

Το εικονικό αυτό δίκτυο περιλαμβάνει 2 τερματικούς υπολογιστές (```office-pc``` & ```home-pc```) και έναν εξυπηρετητή (```WEBSERVER```). Επιπλέον, το δίκτυο αυτό είναι προσβάσιμο μέσω της διέυθυνσης ```10.0.0.0/24```.

## Αναζήτηση κόμβων στο δίκτυο IMUNES μέσω OpenNMS Single Discovery Scan

Για να λάβουμε πληροφορίες σχετικά με αυτό το δίκυτο μέσω του πρωτοκόλλου SNMP ακολουθούμε τα εξής βήματα στον πίνακα διαχείρησης του OpenNMS.

- Μεταβαίνουμε στον πίνακα διαχείρησης του OpenNMS, http://localhost:8980/opennms
- Από τη μπάρα επιλογών πάνω δεξιά, κάνουμε κλίκ στο ```Configure OpenNMS``` εικονίδιο (γραναζάκι) ή μπορούμε να μεταβούμε κατευθείαν μέσω http://localhost:8980/opennms/admin/index.jsp ![OpenNMS Admin Panel](admin-panel.png)
- Στο μενού ```Provisioning``` επιλέγουμε το ```Run Single Discovery Scan``` ή http://localhost:8980/opennms/admin/discovery/edit-scan.jsp ![Run Single Discovery Scan Option](run-single-discovery.png)
- Στις ρυθμίσεις του Single Discovery Scan, στο ```Include Ranges``` κάνουμε κλίκ στο ```Add New```. ![OpenNMS Single Discovery Scan Settings](singe-discovery-settings.png)
- Στο νέο παράθυρο που θα εμφανιστεί, στο πεδίο ```Begin IP Address``` εισάγουμε τη διέυθυνση ```10.0.0.0``` και στο πεδίο ```End IP Address``` εισάγουμε τη διέυθυνση ```10.0.0.254``` και έπειτα κάνουμε κλίκ στο ```Add``` να προσθέσουμε το εύρος των διευθύνσεων. ![Single Discovery Scan IP Range](singe-discovery-ip-range.png)
- Πλέον, το εύρος των διευθύνσεων αυτών θα πρέπει να εμφανιστεί ως εξής. ![IP Ranges Included](include-ip-ranges.png)
- Κάνουμε κλίκ στο κουμπί ```Start Discovery Scan``` ![Start Discovery Scan](start-discovery-scan.png) με σκοπό να ξεκινήσει η αναζήτηση.

Τα αποτελέσματα της αναζήτησης θα εμφανιστούν μετά από λίγα λεπτά μέχρι να ολοκληρωθεί η διαδικασία της ανακάλυψης όλων τον κόμβων δικτύου μέσα στο εύρος το οποίο δηλώσαμε.

Το OpenNMS προσθέτει τους κόμβους που βρίσκει σε κάθε αναζήτηση στο ```Info -> Nodes```, δηλαδή, σε μία λίστα κόμβων όπου και από εκεί μπορούμε να επιλέξουμε τον κάθε κόμβο και να δούμε τις σχετικές πληροφορίες με αυτό. Μπορούμε να μεταβούμε στη λίστα αυτή και μέσω του συνδέσμου http://localhost:8980/opennms/element/nodeList.htm

![Discovery Scan Results](scan-results.png)
Όπως παρατηρούμε και στην παραπάνω εικόνα, το OpenNMS είναι σε θέση να ανακαλύψει όλους τους κόμβους του δικτύου στο IMUNES που δηλώσαμε στο εύρος διευθύνσεων.

Για να δούμε πληροφορίες σχετικά με τον κάθε κόμβο απλά κάνουμε κλίκ πάνω στο όνομα του. Ας εξερευνήσουμε τον κόμβο ```WEBSERVER```.
![Node Info](webserver-node-details.png)

Βλέπουμε πως το OpenNMS έκανε χρήση του πρωτοκόλλου SNMP με σκοπό τη συλλογή δεδομένων σχετικά με τον κόμβο αυτό όπως παρατηρούμε στο ```SNMP Attributes```.

Επίσης, στο ```Availability``` βλέπουμε τις υπηρεσίες που ανίχνευσε το OpenNMS στον κόμβο αυτό αλλά καθώς και τη διαθεσιμότητα της κάθε υπηρεσίας. Σε περίπτωση που κάποια από τις υπηρεσίες αυτές (π.χ. είτε ICMP ή SNMP) δεν είναι διαθέσιμη για κάποιο χρονικό διάστημα, το OpenNMS θα μας ειδοποιήσει και θα σημειώσει το συμβάν ώστε να μπορούμε να το διερευνήσουμε περαιτέρω. 

Το OpenNMS μας επιτρέπει να οργανώσουμε τους κόμβους δικτύων σε ομάδες για την ευκολότερη διαχείρηση τους. Επίσης, προσφέρει έλεγχο διαθεσιμότητας υπηρεσίας για κάθε κόμβο. Περαιτέρω, μας επιτρέπει να ανανεώσουμε και να ενημερώσουμε τις πληροφορίες του κάθε κόμβου. Οι πληροφορίες αυτές περιλαμβάνουν δεδομένα όπως το υλικό αλλά έως και πληροφορίες σχετικά με τη φυσική τοποθεσία ενός κόμβου. Αυτές οι πληροφορίες μας βοηθούν να καταλαβαίνουμε καλύτερα τις τοπολογίες των δικτύων μας και προσφέρουν ευκολότερη διαχείρηση των περιουσιακών στοιχείων ενός οργανισμού.

![Asset Info](asset-info.png)
