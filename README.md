RO en attente de la fin de migration



















































**hdd-diagnostic**
How to

TODO : ajouter un guide sur badblocks et notamment la possibilité de reprendre une opération interrompue (voir [ici](https://www.qworqs.com/2022/07/26/resume-bad-blocks-where-it-was-stopped/))

Avant propos : cette version en vrac, non structurée et peu lisible est basé sur le fichier /media/user/DATA/projets/linux/hdd/user-tosh_sda1_trouble peu lisible aussi. A terme cette version se voudra claire, structurée, détaillée et compréhensible.

Voir (fusionner par la suite) aussi l'utilisation de smartmontools [ici](https://www.thomas-krenn.com/en/wiki/SMART_tests_with_smartctl)

# Contexte
Dans le cadre d'une p2v de user-tosh vers une VM QEmu/KVM, la sauvegarde de la partition système sda1 avec Clonezilla a été interrompue 
par une erreur bloquante car des secteurs étaient défectueux (de mémoire).

Selon la proposition j'avais la possibilité de recommencer la sauvegarde en ignorant (je ne sais plus le paramètre, mais il fallait aller en mode expert)
=> la sauvegarde se fait sans s'interrompre cette fois et au final j'ai réussi à booter la VM.

Voir mixxx/p2v_user-tosh_avec_mixxx_et_controleur

Mais je souhaiterais corriger le problème et ne plus avoir à ignorer des secteurs défectueux !

C'est à cela que servira ce document qui se veut précis, le plus universel possible 
et qu'il livre clé en main une solution face à un problème de secteur de disque défectueux.

Remarque : Quand on parle de secteur défectueux on est au niveau physique, il n'est pas question de partitions, ni de système de fichiers.

## Outil
pour la sauvegarde, la vérification, la réparation, etc. j'ai utilisé Clonezilla personnalisé (voir Admin_reseaux/au_cas_ou/clonezilla/install_clonezilla_avec_donnees_perso)
A noter qu'il embarque nativement (en dehors de la personnalisation) : dd, smartctl, ddrescue, fdisk, lsblk, hdparm, sg_verify, etc.

Nota : 
- les commentaires #" ... sont des citations du post (https://www.linuxquestions.org/questions/linux-hardware-18/smartctl-long-selftest-shows-read-error-4175488145-print/)
  qui est le fil principal que je suivrais entre OP (joe_2000) et AIDANT (rknichols)
- les commentaires #( ... sont les commandes depuis le howto de smartmontools (https://www.smartmontools.org/wiki/BadBlockHowto)

Aussi, les pages sont sauvegardées localement au cas où elles seraient supprimées

# Tester le disque
<details>
  <summary>Cliquer pour étendre !</summary>

    # En solo d'après mes précédentes investigations.
  
<details>
  <summary>Afficher les partitions !</summary>  

    root@CZ-LIVE:~# lsblk
    NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
    loop0    7:0    0 418,9M  1 loop /usr/lib/live/mount/rootfs/filesystem.squashfs
    sda      8:0    0 465,8G  0 disk 
    ├─sda1   8:1    0  37,3G  0 part 
    ├─sda2   8:2    0 411,6G  0 part 
    ├─sda3   8:3    0   7,9G  0 part 
    └─sda4   8:4    0   8,9G  0 part 
    sdb      8:16   1   3,7G  0 disk 
    └─sdb1   8:17   1   3,7G  0 part /usr/lib/live/mount/medium
    sr0     11:0    1  1024M  0 rom 

</details>      

    root@CZ-LIVE:~# smartctl -t long /dev/sda
    smartctl 7.2 2020-12-30 r5155 [i686-linux-5.10.0-8-686] (local build)
    Copyright (C) 2002-20, Bruce Allen, Christian Franke, www.smartmontools.org

    === START OF OFFLINE IMMEDIATE AND SELF-TEST SECTION ===
    Sending command: "Execute SMART Extended self-test routine immediately in off-line mode".
    Drive command "Execute SMART Extended self-test routine immediately in off-line mode" successful.
    Testing has begun.
    Please wait 198 minutes for test to complete.
    Test will complete after Wed Dec  1 17:50:02 2021 UTC
    Use smartctl -X to abort test.
    root@CZ-LIVE:~# smartctl -l selftest /dev/sda
    smartctl 7.2 2020-12-30 r5155 [i686-linux-5.10.0-8-686] (local build)
    Copyright (C) 2002-20, Bruce Allen, Christian Franke, www.smartmontools.org

    === START OF READ SMART DATA SECTION ===
    SMART Self-test log structure revision number 1
    Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
    # 1  Extended offline    Completed: read failure       00%      9732         21732119

    # => read failure confirmée !


    # => recherche d'aide

    # => https://www.linuxquestions.org/questions/linux-hardware-18/smartctl-long-selftest-shows-read-error-4175488145-print/




    #" That could be just a single bad sector, which would not be considered sufficient cause for replacement. 
    #" Post the output from "smartctl -A /dev/sda". 


    root@CZ-LIVE:~# smartctl -A /dev/sda
    smartctl 7.2 2020-12-30 r5155 [i686-linux-5.10.0-8-686] (local build)
    Copyright (C) 2002-20, Bruce Allen, Christian Franke, www.smartmontools.org

    === START OF READ SMART DATA SECTION ===
    SMART Attributes Data Structure revision number: 16
    Vendor Specific SMART Attributes with Thresholds:
    ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
      1 Raw_Read_Error_Rate     0x000b   100   100   050    Pre-fail  Always       -       0
      2 Throughput_Performance  0x0005   100   100   050    Pre-fail  Offline      -       0
      3 Spin_Up_Time            0x0027   100   100   001    Pre-fail  Always       -       1662
      4 Start_Stop_Count        0x0032   100   100   000    Old_age   Always       -       2515
      5 Reallocated_Sector_Ct   0x0033   100   100   050    Pre-fail  Always       -       2
      7 Seek_Error_Rate         0x000b   100   100   050    Pre-fail  Always       -       0
      8 Seek_Time_Performance   0x0005   100   100   050    Pre-fail  Offline      -       0
      9 Power_On_Hours          0x0032   076   076   000    Old_age   Always       -       9732
     10 Spin_Retry_Count        0x0033   150   100   030    Pre-fail  Always       -       0
     12 Power_Cycle_Count       0x0032   100   100   000    Old_age   Always       -       2448
    191 G-Sense_Error_Rate      0x0032   100   100   000    Old_age   Always       -       374
    192 Power-Off_Retract_Count 0x0032   100   100   000    Old_age   Always       -       98
    193 Load_Cycle_Count        0x0032   084   084   000    Old_age   Always       -       160868
    194 Temperature_Celsius     0x0022   100   100   000    Old_age   Always       -       29 (Min/Max 9/49)
    196 Reallocated_Event_Count 0x0032   100   100   000    Old_age   Always       -       2
    197 Current_Pending_Sector  0x0032   100   100   000    Old_age   Always       -       1
    198 Offline_Uncorrectable   0x0030   100   100   000    Old_age   Offline      -       1
    199 UDMA_CRC_Error_Count    0x0032   200   200   000    Old_age   Always       -       0
    220 Disk_Shift              0x0002   100   100   000    Old_age   Always       -       8252
    222 Loaded_Hours            0x0032   081   081   000    Old_age   Always       -       7772
    223 Load_Retry_Count        0x0032   100   100   000    Old_age   Always       -       0
    224 Load_Friction           0x0022   100   100   000    Old_age   Always       -       0
    226 Load-in_Time            0x0026   100   100   000    Old_age   Always       -       342
    240 Head_Flying_Hours       0x0001   100   100   001    Pre-fail  Offline      -       0


    #" The parameters of particular interest are 
    #" Reallocated_Sector_Ct (ID #5), Reallocated_Event_Count (#196), Current_Pending_Sector (#197), and Offline_Uncorrectable (#198).

    # Il y a aussi G-Sense_Error_Rate qui est un indicateur de chocs et dont l'impact est laissé à la discretion du constructeur (voir après)
    # Il y a aussi Power_On_Hours qui horodate les résultats

    #" A drive with a small number of bad sectors is not necessarily on its way out. 
    #" Without knowing what caused them, it's hard to predict whether the problem will grow over time. 
    #" Procedures for dealing with a small number of bad sectors range from the ham-fisted "Overwrite the whole drive with zeros" 
    #" (which lets the drive reallocate any bad sectors), to the much more tedious but minimally invasive approach in http://smartmontools.sourceforge.net/badblockhowto.html.

    # Note : le lien cité n'existe plus, j'ai trouvé : https://www.smartmontools.org/wiki/BadBlockHowto


    root@CZ-LIVE:~# smartctl -A /dev/sda | grep -iE "Power_On_Hours|G-Sense_Error_Rate|Reallocated|Pending|Uncorrectable"
      5 Reallocated_Sector_Ct   0x0033   100   100   050    Pre-fail  Always       -       2
      9 Power_On_Hours          0x0032   076   076   000    Old_age   Always       -       9732 
    191 G-Sense_Error_Rate      0x0032   100   100   000    Old_age   Always       -       374
    196 Reallocated_Event_Count 0x0032   100   100   000    Old_age   Always       -       2
    197 Current_Pending_Sector  0x0032   100   100   000    Old_age   Always       -       1
    198 Offline_Uncorrectable   0x0030   100   100   000    Old_age   Offline      -       1

    # => Current_Pending_Sector est à 1


    # A noter : G-Sense_Error_Rate est de 374 et AIDANT répond dans une premier temps que 47 indique que le disque a été brutalisté 
    # (démenti plus tard par OP qui dit en avoir pris soin à sa connaissance)

    #" Looks like just one bad sector has been discovered so far. Looks like that drive has led a pretty rough life, 
    #" though. I believe that "47" in the G-Sense_Error_Rate is the number of times it has been dropped or bounced pretty hard while running.

    #" ... If that laptop has led a gentle life with you, then either I am misinterpreting that G-Sense error number 
    #" (the meaning of those raw numbers is really only known to the manufacturer) or something bad happened to it at the factory.

    # => G-Sense_Error_Rate est une valeur dont l'impact sur la vie du disque est laissée à la discretion du constructeur



    #" It might be a good idea to see whether there are any undiscovered bad sectors. 
    #" Simplest way to do that is just to read the entire drive:
    #" `dd if=/dev/sda of=/dev/null bs=64k`

    # Cette commande est incomplète car elle quitte à la première erreur

    #" Arrrgh!! :o That's because I forgot to include the "conv=noerrors" option on that dd command to tell it not to stop when it encountered the first I/O error. 

    # elle sera corrigée après en :
    # `dd if=/dev/sda of=/dev/null bs=64k conv=noerror`

    # A laquelle j'ai ajouté "status=progress" pour voir la progression



    root@CZ-LIVE:~# dd if=/dev/sda of=/dev/null bs=64k conv=noerror status=progress
    11126841344 octets (11 GB, 10 GiB) copiés, 160 s, 69,4 MB/s 
    dd: erreur de lecture dans '/dev/sda': Erreur d'entrée/sortie
    169782+1 enregistrements lus
    169782+1 enregistrements écrits
    11126841344 octets (11 GB, 10 GiB) copiés, 164,843 s, 67,5 MB/s
    500092444672 octets (500 GB, 466 GiB) copiés, 9216 s, 54,3 MB/s
    7631039+2 enregistrements lus
    7631039+2 enregistrements écrits
    500107804672 octets (500 GB, 466 GiB) copiés, 9216,99 s, 54,3 MB/s

    # => manifestement une erreur de lecture (ignorée pour voir les autres éventuelles)



    root@CZ-LIVE:~# smartctl -A /dev/sda | grep -iE "Power_On_Hours|G-Sense_Error_Rate|Reallocated|Pending|Uncorrectable"
      5 Reallocated_Sector_Ct   0x0033   100   100   050    Pre-fail  Always       -       2
      9 Power_On_Hours          0x0032   076   076   000    Old_age   Always       -       9732 
    191 G-Sense_Error_Rate      0x0032   100   100   000    Old_age   Always       -       374
    196 Reallocated_Event_Count 0x0032   100   100   000    Old_age   Always       -       2
    197 Current_Pending_Sector  0x0032   100   100   000    Old_age   Always       -       1
    198 Offline_Uncorrectable   0x0030   100   100   000    Old_age   Offline      -       1

    # => Current_Pending_Sector n'a pas changé et est toujours à 1 !


    #" Then, look again at the Current_Pending_Sector count in the "smartctl -A" output. 
    #" If that number is small, then the procedure in the Bad Block HOWTO is suitable. 

    # => c'est mon cas je peux à priori réparer !

    # => Youpppiiiii !!!! :DDDDDD




    #----------------------------------------------------------------------
    # Résultat détaillé (pour mémoire)

    root@CZ-LIVE:~# smartctl -a /dev/sda
    smartctl 7.2 2020-12-30 r5155 [i686-linux-5.10.0-8-686] (local build)
    Copyright (C) 2002-20, Bruce Allen, Christian Franke, www.smartmontools.org

    === START OF INFORMATION SECTION ===
    Device Model:     TOSHIBA MK5055GSXN
    Serial Number:    Z9PLS4HBS
    LU WWN Device Id: 5 000039 236009c55
    Firmware Version: GC002M
    User Capacity:    500 107 862 016 bytes [500 GB]
    Sector Size:      512 bytes logical/physical
    Device is:        Not in smartctl database [for details use: -P showall]
    ATA Version is:   ATA8-ACS (minor revision not indicated)
    SATA Version is:  SATA 2.6, 3.0 Gb/s
    Local Time is:    Wed Dec  1 20:54:25 2021 UTC
    SMART support is: Available - device has SMART capability.
    SMART support is: Enabled

    === START OF READ SMART DATA SECTION ===
    SMART overall-health self-assessment test result: PASSED

    General SMART Values:
    Offline data collection status:  (0x00)	Offline data collection activity
                        was never started.
                        Auto Offline Data Collection: Disabled.
    Self-test execution status:      ( 112)	The previous self-test completed having
                        the read element of the test failed.
    Total time to complete Offline 
    data collection: 		(  120) seconds.
    Offline data collection
    capabilities: 			 (0x5b) SMART execute Offline immediate.
                        Auto Offline data collection on/off support.
                        Suspend Offline collection upon new
                        command.
                        Offline surface scan supported.
                        Self-test supported.
                        No Conveyance Self-test supported.
                        Selective Self-test supported.
    SMART capabilities:            (0x0003)	Saves SMART data before entering
                        power-saving mode.
                        Supports SMART auto save timer.
    Error logging capability:        (0x01)	Error logging supported.
                        General Purpose Logging supported.
    Short self-test routine 
    recommended polling time: 	 (   2) minutes.
    Extended self-test routine
    recommended polling time: 	 ( 198) minutes.
    SCT capabilities: 	       (0x0039)	SCT Status supported.
                        SCT Error Recovery Control supported.
                        SCT Feature Control supported.
                        SCT Data Table supported.

    SMART Attributes Data Structure revision number: 16
    Vendor Specific SMART Attributes with Thresholds:
    ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
      1 Raw_Read_Error_Rate     0x000b   100   100   050    Pre-fail  Always       -       0
      2 Throughput_Performance  0x0005   100   100   050    Pre-fail  Offline      -       0
      3 Spin_Up_Time            0x0027   100   100   001    Pre-fail  Always       -       1662
      4 Start_Stop_Count        0x0032   100   100   000    Old_age   Always       -       2515
      5 Reallocated_Sector_Ct   0x0033   100   100   050    Pre-fail  Always       -       2
      7 Seek_Error_Rate         0x000b   100   100   050    Pre-fail  Always       -       0
      8 Seek_Time_Performance   0x0005   100   100   050    Pre-fail  Offline      -       0
      9 Power_On_Hours          0x0032   076   076   000    Old_age   Always       -       9736
     10 Spin_Retry_Count        0x0033   150   100   030    Pre-fail  Always       -       0
     12 Power_Cycle_Count       0x0032   100   100   000    Old_age   Always       -       2448
    191 G-Sense_Error_Rate      0x0032   100   100   000    Old_age   Always       -       374
    192 Power-Off_Retract_Count 0x0032   100   100   000    Old_age   Always       -       98
    193 Load_Cycle_Count        0x0032   084   084   000    Old_age   Always       -       160877
    194 Temperature_Celsius     0x0022   100   100   000    Old_age   Always       -       37 (Min/Max 9/49)
    196 Reallocated_Event_Count 0x0032   100   100   000    Old_age   Always       -       2
    197 Current_Pending_Sector  0x0032   100   100   000    Old_age   Always       -       1
    198 Offline_Uncorrectable   0x0030   100   100   000    Old_age   Offline      -       1
    199 UDMA_CRC_Error_Count    0x0032   200   200   000    Old_age   Always       -       0
    220 Disk_Shift              0x0002   100   100   000    Old_age   Always       -       8252
    222 Loaded_Hours            0x0032   081   081   000    Old_age   Always       -       7774
    223 Load_Retry_Count        0x0032   100   100   000    Old_age   Always       -       0
    224 Load_Friction           0x0022   100   100   000    Old_age   Always       -       0
    226 Load-in_Time            0x0026   100   100   000    Old_age   Always       -       351
    240 Head_Flying_Hours       0x0001   100   100   001    Pre-fail  Offline      -       0

    SMART Error Log Version: 1
    ATA Error Count: 40 (device log contains only the most recent five errors)
        CR = Command Register [HEX]
        FR = Features Register [HEX]
        SC = Sector Count Register [HEX]
        SN = Sector Number Register [HEX]
        CL = Cylinder Low Register [HEX]
        CH = Cylinder High Register [HEX]
        DH = Device/Head Register [HEX]
        DC = Device Command Register [HEX]
        ER = Error register [HEX]
        ST = Status register [HEX]
    Powered_Up_Time is measured from power on, and printed as
    DDd+hh:mm:SS.sss where DD=days, hh=hours, mm=minutes,
    SS=sec, and sss=millisec. It "wraps" after 49.710 days.

    Error 40 occurred at disk power-on lifetime: 9733 hours (405 days + 13 hours)
      When the command that caused the error occurred, the device was active or idle.

      After command completion occurred, registers were:
      ER ST SC SN CL CH DH
      -- -- -- -- -- -- --
      40 41 8a 17 9b 4b 61  Error: UNC at LBA = 0x014b9b17 = 21732119

      Commands leading to the command that caused the error were:
      CR FR SC SN CL CH DH DC   Powered_Up_Time  Command/Feature_Name
      -- -- -- -- -- -- -- --  ----------------  --------------------
      60 08 88 10 9b 4b 40 00      04:07:12.216  READ FPDMA QUEUED
      ef 10 03 00 00 00 a0 00      04:07:12.215  SET FEATURES [Enable SATA feature]
      ef 10 02 00 00 00 a0 00      04:07:12.215  SET FEATURES [Enable SATA feature]
      27 00 00 00 00 00 e0 00      04:07:12.215  READ NATIVE MAX ADDRESS EXT [OBS-ACS-3]
      ec 00 00 00 00 00 a0 00      04:07:12.214  IDENTIFY DEVICE

    Error 39 occurred at disk power-on lifetime: 9733 hours (405 days + 13 hours)
      When the command that caused the error occurred, the device was active or idle.

      After command completion occurred, registers were:
      ER ST SC SN CL CH DH
      -- -- -- -- -- -- --
      40 41 42 17 9b 4b 61  Error: UNC at LBA = 0x014b9b17 = 21732119

      Commands leading to the command that caused the error were:
      CR FR SC SN CL CH DH DC   Powered_Up_Time  Command/Feature_Name
      -- -- -- -- -- -- -- --  ----------------  --------------------
      60 08 40 10 9b 4b 40 00      04:07:07.836  READ FPDMA QUEUED
      60 08 38 08 9b 4b 40 00      04:07:07.836  READ FPDMA QUEUED
      60 08 28 00 9b 4b 40 00      04:07:07.804  READ FPDMA QUEUED
      ef 10 03 00 00 00 a0 00      04:07:07.804  SET FEATURES [Enable SATA feature]
      ef 10 02 00 00 00 a0 00      04:07:07.804  SET FEATURES [Enable SATA feature]

    Error 38 occurred at disk power-on lifetime: 9733 hours (405 days + 13 hours)
      When the command that caused the error occurred, the device was active or idle.

      After command completion occurred, registers were:
      ER ST SC SN CL CH DH
      -- -- -- -- -- -- --
      40 41 82 17 9b 4b 61  Error: UNC at LBA = 0x014b9b17 = 21732119

      Commands leading to the command that caused the error were:
      CR FR SC SN CL CH DH DC   Powered_Up_Time  Command/Feature_Name
      -- -- -- -- -- -- -- --  ----------------  --------------------
      60 00 80 00 9b 4b 40 00      04:07:03.436  READ FPDMA QUEUED
      60 00 78 00 9a 4b 40 00      04:07:03.434  READ FPDMA QUEUED
      60 00 70 00 99 4b 40 00      04:07:03.433  READ FPDMA QUEUED
      60 00 68 00 98 4b 40 00      04:07:03.431  READ FPDMA QUEUED
      60 00 60 00 97 4b 40 00      04:07:03.428  READ FPDMA QUEUED

    Error 37 occurred at disk power-on lifetime: 9733 hours (405 days + 13 hours)
      When the command that caused the error occurred, the device was active or idle.

      After command completion occurred, registers were:
      ER ST SC SN CL CH DH
      -- -- -- -- -- -- --
      40 41 2a 17 9b 4b 61  Error: UNC at LBA = 0x014b9b17 = 21732119

      Commands leading to the command that caused the error were:
      CR FR SC SN CL CH DH DC   Powered_Up_Time  Command/Feature_Name
      -- -- -- -- -- -- -- --  ----------------  --------------------
      60 08 28 10 9b 4b 40 00      03:54:51.281  READ FPDMA QUEUED
      ef 10 03 00 00 00 a0 00      03:54:51.281  SET FEATURES [Enable SATA feature]
      ef 10 02 00 00 00 a0 00      03:54:51.281  SET FEATURES [Enable SATA feature]
      27 00 00 00 00 00 e0 00      03:54:51.280  READ NATIVE MAX ADDRESS EXT [OBS-ACS-3]
      ec 00 00 00 00 00 a0 00      03:54:51.280  IDENTIFY DEVICE

    Error 36 occurred at disk power-on lifetime: 9733 hours (405 days + 13 hours)
      When the command that caused the error occurred, the device was active or idle.

      After command completion occurred, registers were:
      ER ST SC SN CL CH DH
      -- -- -- -- -- -- --
      40 41 0a 17 9b 4b 61  Error: UNC at LBA = 0x014b9b17 = 21732119

      Commands leading to the command that caused the error were:
      CR FR SC SN CL CH DH DC   Powered_Up_Time  Command/Feature_Name
      -- -- -- -- -- -- -- --  ----------------  --------------------
      60 08 08 10 9b 4b 40 00      03:54:46.913  READ FPDMA QUEUED
      60 08 00 08 9b 4b 40 00      03:54:46.913  READ FPDMA QUEUED
      60 08 10 00 9b 4b 40 00      03:54:46.881  READ FPDMA QUEUED
      ef 10 03 00 00 00 a0 00      03:54:46.881  SET FEATURES [Enable SATA feature]
      ef 10 02 00 00 00 a0 00      03:54:46.880  SET FEATURES [Enable SATA feature]

    SMART Self-test log structure revision number 1
    Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
    # 1  Extended offline    Completed: read failure       00%      9732         21732119

    SMART Selective self-test log data structure revision number 1
     SPAN  MIN_LBA  MAX_LBA  CURRENT_TEST_STATUS
        1        0        0  Not_testing
        2        0        0  Not_testing
        3        0        0  Not_testing
        4        0        0  Not_testing
        5        0        0  Not_testing
    Selective self-test flags (0x0):
      After scanning selected spans, do NOT read-scan remainder of disk.
    If Selective self-test is pending on power-up, resume after 0 minute delay.
    #----------------------------------------------------------------------




    # Juste le self-test
    root@CZ-LIVE:~# smartctl -l selftest /dev/sda
    SMART Self-test log structure revision number 1
    Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
    # 1  Extended offline    Completed: read failure       00%      9732         21732119

    # => erreur (LBA_of_first_error) ici : LBA 21732119


    # Vérification
    # https://www.smartmontools.org/wiki/BadBlockHowto#Badblockreassignment

    root@CZ-LIVE:~# sg_verify --lba=21732119 /dev/sda
    verify(10):
    Fixed format, current; Sense key: Medium Error
    Additional sense: Unrecovered read error - auto reallocate failed
      Info fld=0x14b9b17 [21732119] 
    VERIFY(10) medium or hardware error, reported lba=0x14b9b17
    sg_verify failed: Medium or hardware error with Info

    # => confirmé

</details>

# Réparer
https://www.smartmontools.org/wiki/BadBlockHowto#ext2ext3firstexample

    #( In this example, the disk is failing self-tests at Logical Block Address LBA = 0x016561e9 = 23421417. 
    #( The LBA counts sectors in units of 512 bytes, and starts at zero. 


    # Pour mon cas erreur au LBA 21732119 (rappel), donc L = 21732119

    # Aussi le "The LBA counts sectors in units of 512 bytes" (après nommé sect_size) n'est pas une vérité absolue 
    # et est dépendant du formatage du disque


    #( First Step: We need to locate the partition on which this sector of the disk lives: 

    root@CZ-LIVE:~# fdisk -lu /dev/sda
    Disk /dev/sda: 465,76 GiB, 500107862016 bytes, 976773168 sectors
    Disk model: TOSHIBA MK5055GS
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disklabel type: dos
    Disk identifier: 0x2f1ca158

    Device     Boot     Start       End   Sectors   Size Id Type
    /dev/sda1  *         2048  78125055  78123008  37,3G 83 Linux
    /dev/sda2        78125056 941406207 863281152 411,6G 83 Linux
    /dev/sda3       941406208 958007295  16601088   7,9G 82 Linux swap / Solaris
    /dev/sda4       958007296 976771071  18763776   8,9G 83 Linux
    # => /dev/sda1 car 2,048 < 21,732,119 < 78,125,055

    # => début de la partition /dev/sda1 S = 2048

    # => taille d'un secteur sect_size = 512


    #( To verify the type of the file system

    root@CZ-LIVE:~# lsblk -fpn -o NAME,FSTYPE /dev/sda1
    /dev/sda1 ext4

    # => ext4 n'est pas indiqué comme FS possible par la procédure
    # => je tente tout de même

    # Nota : toutefois il n'y a pas de note différenciant ext2 de ext3, donc peut être que ext4 suit le même principe



    #( Second Step: we need to find the block size of the file system (normally 4096 bytes for ext2): 
    root@CZ-LIVE:~# tune2fs -l /dev/sda1 | grep "Block size:"
    Block size:               4096
    # => Block size = 4096


    #( Third Step: we need to determine which File System Block contains this LBA. The formula is:
    #  b = (int)((L-S)*512/B)
    # where:
    # b = File System block number
    # B = File system block size in bytes
    # L = LBA of bad sector
    # S = Starting sector of partition as shown by fdisk -lu
    # and (int) denotes the integer part.

    # comme expliqué dessus sector_size dépend de la taille de secteur, j'adapterais donc par :
    #  b = (int)((L-S)*sect_site/B)
    # where:
    # b = File System block number
    # B = File system block size in bytes
    # L = LBA of bad sector
    # S = Starting sector of partition as shown by fdisk -lu
    # sect_size = size of a sector in bytes
    # and (int) denotes the integer part.

    # A noter que cette formule est incomplète pour prendre en compte d'autres choses comme par exemple avec un cryptage LUKS, etc.


    # Variables pour éviter les erreurs
    root@CZ-LIVE:~# L=21732119
    root@CZ-LIVE:~# S=2048
    root@CZ-LIVE:~# B=4096
    root@CZ-LIVE:~# sect_size=512

    root@CZ-LIVE:~# echo $L $S $sect_size $B
    21732119 2048 512 4096
    root@CZ-LIVE:~# b=$( echo "( ( $L - $S ) * $sect_size ) / $B" | bc )
    root@CZ-LIVE:~# echo $b
    2716258

    # Rappel : par défaut bc traite les nombres comme des entiers

    # => le bloc concerné est b = 2716258



    #( Fourth Step: we use debugfs to locate the inode stored in this block, and the file that contains that inode:

    root@CZ-LIVE:~# debugfs
    debugfs 1.46.2 (28-Feb-2021)
    debugfs:  open /dev/sda1
    debugfs:  testb 2716258
    Block 2716258 marked in use
    debugfs:  quit

    #( If the block is not in use, as in the above example, then you can skip the rest of this step and go ahead to Step Five. 
    # => ici ce n'est pas le cas le block est utilisé

    #( If, on the other hand, the block is in use, we want to identify the file that uses it: 

    root@CZ-LIVE:~# debugfs
    debugfs:  icheck 2716258
    Block	Inode number
    2716258	264670
    debugfs:  ncheck 264670
    Inode	Pathname
    264670	/var/log/syslog.1
    debugfs:  

    # => bloc utilisé par /var/log/syslog.1

    #( When we are working with an ext3 file system, it may happen that the affected file is the journal itself. 
    #( if this is the case, the inode number will be very small. In any case, debugfs will not be able to get the file name:
    #( To get around this situation, we can remove the journal altogether: `tune2fs -O ^has_journal /dev/sda1`
    #( and then start again with Step Four: we should see this time that the wrong block is not in use any more. 
    #( If we removed the journal file, at the end of the whole procedure we should remember to rebuild it: `tune2fs -j /dev/sda1`



    # Le numéro d'inode n'est PAS très petit et un fichier est retourné (sinon Inode Pathname seraient vides)
    # => ça ne concerne pas un fichier de journal



    # Vérification
    # https://www.smartmontools.org/wiki/BadBlockHowto#ext2ext3secondexample (après le debugfs)

    #( And finally, just to confirm that this is really the damaged file: 

    root@CZ-LIVE:~# mount /dev/sda1 /mnt/
    root@CZ-LIVE:~# md5sum /mnt/var/log/syslog.1
    md5sum: /mnt/var/log/syslog.1: Erreur d'entrée/sortie
    root@CZ-LIVE:~# umount /mnt/

    # => c'est bien le fichier /var/log/syslog.1 qui est impacté !



    # ATTENTION

    #( Fifth Step NOTE: This last step will permanently and irretrievably destroy the contents of the file system block that is damaged: 
    #( if the block was allocated to a file, some of the data that is in this file is going to be overwritten with zeros. 
    #( You will not be able to recover that data unless you can replace the file with a fresh or correct version. 


    # Essai de sauver le fichier impacté donc avant de réinitialiser le contenu du bloc


    #" for recovery it might be best to use ddrescue to do a best effort at making in image of this drive on another device 
    #" and then copy that image back to the original drive.

    #" Depending on how much data that is and how successful you are at copying it, it could be worthwhile to use ddrescue 
    #" to make an image of the whole drive (would need another device 1TB or larger) or perhaps just the relevant partition(s), 
    #" depending on how the drive is divided up. ddrescue will make a first pass copying all the sectors that can be read easily, 
    #" then go back and make more aggressive efforts to fill in the gaps.
    #" (There would be little point in running the above dd command if you are going to use ddrescue to try to image the whole disk.)

    # Astuce : on peut aussi utiliser ddrescue pour sauver un unique fichier
    # https://www.poftut.com/recover-data-ddrescue-command/

    root@CZ-LIVE:~# ddrescue /mnt/var/log/syslog.1 /mnt/syslog.1
    GNU ddrescue 1.23
    Press Ctrl-C to interrupt
         ipos:  648424 kB, non-trimmed:        0 B,  current rate:       0 B/s
         opos:  648424 kB, non-scraped:        0 B,  average rate:  11770 kB/s
    non-tried:        0 B,  bad-sector:     4096 B,    error rate:     102 B/s
      rescued:    2636 MB,   bad areas:        1,        run time:      3m 44s
    pct rescued:   99.99%, read errors:        9,  remaining time:         n/a
                                  time since last successful read:         27s

    root@CZ-LIVE:~# md5sum /mnt/syslog.1
    013a07620c90fd903a4fe56f5a0f1068  /mnt/syslog.1

    # => le fichier sain est récupéré et copié ailleurs !



    # => on réalloue le mauvais block

    #( To force the disk to reallocate this bad block we'll write zeros to the bad block, and sync the disk: 

    root@CZ-LIVE:~# dd if=/dev/zero of=/dev/sda1 bs=4096 count=1 seek=2716258
    1+0 enregistrements lus
    1+0 enregistrements écrits
    4096 octets (4,1 kB, 4,0 KiB) copiés, 0,0475258 s, 86,2 kB/s
    root@CZ-LIVE:~# sync

    # => PAS d'erreur


    #( On some occasions using dd may not result in the sector being reallocated, with an error along the following lines:
    #( ...
    #( May 24 14:19:30 ht-xm-2 kernel: Buffer I/O error on dev sda, logical block 2097857, async page read


    # Par acquis de conscience j'ai regardé dans les logs (sans trop savoir lequel, j'ai testé le retour de journalctl sans paramètres) 

    root@CZ-LIVE:~# journalctl | grep blk_update_request
    ...
    déc. 02 00:22:51 CZ-LIVE kernel: blk_update_request: I/O error, dev sda, sector 21732119 op 0x0:(READ) flags 0x0 phys_seg 1 prio class 0

    # => bingo ! je serais dans le cas où dd n'a pas réalloué le secteur

    # => secteur concerné 21732119


    #( In this case you may have better success using the hdparm tool. First confirm the read error on the sector, which would look like this for the above error
    #( ...
    #( And when you've confirmed the sector is indeed unreadable ...

    root@CZ-LIVE:~# hdparm --read-sector 21732119 /dev/sda

    /dev/sda:
    reading sector 21732119: succeeded
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000
    0000 0000 0000 0000 0000 0000 0000 0000

    # => la lecture a réussit ; les 0..0 semblent indiquer que le secteur a bien été réinitialisé (il y avait des données de fichier avant)

    # => à priori, je ne suis pas dans le cas où dd n'a pas réalloué le secteur


    # J'ai quand même tenté une écriture

    #(  And when you've confirmed the sector is indeed unreadable, use hdparm to write to that sector: 

    root@CZ-LIVE:~# hdparm --write-sector 21732119 /dev/sda

    /dev/sda:
    Use of --write-sector is VERY DANGEROUS.
    You are trying to deliberately overwrite a low-level sector on the media.
    This is a BAD idea, and can easily result in total data loss.
    Please supply the --yes-i-know-what-i-am-doing flag if you really want this.
    Program aborted.

    # => je ne tente pas le diable ici !



    #(  Now everything is back to normal: the sector has been reallocated. Compare the output just below to similar output near the top of this article: 


    # AVANT

    root@CZ-LIVE:~# smartctl -A /dev/sda | grep -iE "Power_On_Hours|G-Sense_Error_Rate|Reallocated|Pending|Uncorrectable"
      5 Reallocated_Sector_Ct   0x0033   100   100   050    Pre-fail  Always       -       2
      9 Power_On_Hours          0x0032   076   076   000    Old_age   Always       -       9732 
    191 G-Sense_Error_Rate      0x0032   100   100   000    Old_age   Always       -       374
    196 Reallocated_Event_Count 0x0032   100   100   000    Old_age   Always       -       2
    197 Current_Pending_Sector  0x0032   100   100   000    Old_age   Always       -       1
    198 Offline_Uncorrectable   0x0030   100   100   000    Old_age   Offline      -       1

    ...

    # APRES

    root@CZ-LIVE:~# smartctl -A /dev/sda | grep -iE "Power_On_Hours|G-Sense_Error_Rate|Reallocated|Pending|Uncorrectable"
      5 Reallocated_Sector_Ct   0x0033   100   100   050    Pre-fail  Always       -       3
      9 Power_On_Hours          0x0032   076   076   000    Old_age   Always       -       9733 
    191 G-Sense_Error_Rate      0x0032   100   100   000    Old_age   Always       -       374
    196 Reallocated_Event_Count 0x0032   100   100   000    Old_age   Always       -       3
    197 Current_Pending_Sector  0x0032   100   100   000    Old_age   Always       -       0
    198 Offline_Uncorrectable   0x0030   100   100   000    Old_age   Offline      -       1

    # => changement manifeste !

    # => plus de Current_Pending_Sector (dans un 1er temps) mais toujours Offline_Uncorrectable à 1


    #( Note: for some disks it may be necessary to update the SMART Attribute values by using smartctl -t offline /dev/hda

    root@CZ-LIVE:~# smartctl -t offline /dev/sda
    smartctl 7.2 2020-12-30 r5155 [i686-linux-5.10.0-8-686] (local build)
    Copyright (C) 2002-20, Bruce Allen, Christian Franke, www.smartmontools.org

    === START OF OFFLINE IMMEDIATE AND SELF-TEST SECTION ===
    Sending command: "Execute SMART off-line routine immediately in off-line mode".
    Drive command "Execute SMART off-line routine immediately in off-line mode" successful.
    Testing has begun.
    Please wait 120 seconds for test to complete.
    Test will complete after Thu Dec  2 00:53:23 2021 UTC
    Use smartctl -X to abort test.

    # => je ne sais pas quoi en faire

# Restaurer le fichier en erreur par une version saine

    # Je tente de restaurer le fichier par sa version saine
    root@CZ-LIVE:~# rm /mnt/var/log/syslog.1
    root@CZ-LIVE:~# cp /mnt/syslog.1 /mnt/var/log/syslog.1
    root@CZ-LIVE:~# md5sum /mnt/var/log/syslog.1
    013a07620c90fd903a4fe56f5a0f1068  /mnt/var/log/syslog.1
    root@CZ-LIVE:~# rm /mnt/syslog.1

    # => OK fichier restauré ne produit plus d'erreur

# Confirmer que tout va bien


    #( We have corrected the first errored block. If more than one blocks were errored, we should repeat all the steps for the subsequent ones. 

    # Test long vers 8h

    root@CZ-LIVE:~# smartctl -t long /dev/sda

    # AIE : j'ai oublié de copier l'affichage
    # => se finit vers 11h00

    # ...

    # Vers 12h

    #( ... After we do that, the disk will pass its self-tests again: 

    root@CZ-LIVE:~# smartctl -l selftest /dev/sda
    smartctl 7.2 2020-12-30 r5155 [i686-linux-5.10.0-8-686] (local build)
    Copyright (C) 2002-20, Bruce Allen, Christian Franke, www.smartmontools.org

    === START OF READ SMART DATA SECTION ===
    SMART Self-test log structure revision number 1
    Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
    # 1  Extended offline    Completed without error       00%      9743         -
    # 2  Extended offline    Completed: read failure       00%      9732         21732119
    1 of 1 failed self-tests are outdated by newer successful extended offline self-test # 1
    # => Completed without error at 9743h
    # => YESSSSSSSSSSSSSSSSSSs :DDDDDDDDD



    #----------------------------------------------------------------------
    # Résultat détaillé (pour mémoire)

    root@CZ-LIVE:~# smartctl -a /dev/sda
    smartctl 7.2 2020-12-30 r5155 [i686-linux-5.10.0-8-686] (local build)
    Copyright (C) 2002-20, Bruce Allen, Christian Franke, www.smartmontools.org

    === START OF INFORMATION SECTION ===
    Device Model:     TOSHIBA MK5055GSXN
    Serial Number:    Z9PLS4HBS
    LU WWN Device Id: 5 000039 236009c55
    Firmware Version: GC002M
    User Capacity:    500 107 862 016 bytes [500 GB]
    Sector Size:      512 bytes logical/physical
    Device is:        Not in smartctl database [for details use: -P showall]
    ATA Version is:   ATA8-ACS (minor revision not indicated)
    SATA Version is:  SATA 2.6, 3.0 Gb/s
    Local Time is:    Thu Dec  2 10:59:53 2021 UTC
    SMART support is: Available - device has SMART capability.
    SMART support is: Enabled

    === START OF READ SMART DATA SECTION ===
    SMART overall-health self-assessment test result: PASSED

    General SMART Values:
    Offline data collection status:  (0x02)	Offline data collection activity
                        was completed without error.
                        Auto Offline Data Collection: Disabled.
    Self-test execution status:      (   0)	The previous self-test routine completed
                        without error or no self-test has ever 
                        been run.
    Total time to complete Offline 
    data collection: 		(  120) seconds.
    Offline data collection
    capabilities: 			 (0x5b) SMART execute Offline immediate.
                        Auto Offline data collection on/off support.
                        Suspend Offline collection upon new
                        command.
                        Offline surface scan supported.
                        Self-test supported.
                        No Conveyance Self-test supported.
                        Selective Self-test supported.
    SMART capabilities:            (0x0003)	Saves SMART data before entering
                        power-saving mode.
                        Supports SMART auto save timer.
    Error logging capability:        (0x01)	Error logging supported.
                        General Purpose Logging supported.
    Short self-test routine 
    recommended polling time: 	 (   2) minutes.
    Extended self-test routine
    recommended polling time: 	 ( 198) minutes.
    SCT capabilities: 	       (0x0039)	SCT Status supported.
                        SCT Error Recovery Control supported.
                        SCT Feature Control supported.
                        SCT Data Table supported.

    SMART Attributes Data Structure revision number: 16
    Vendor Specific SMART Attributes with Thresholds:
    ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
      1 Raw_Read_Error_Rate     0x000b   100   100   050    Pre-fail  Always       -       0
      2 Throughput_Performance  0x0005   100   100   050    Pre-fail  Offline      -       0
      3 Spin_Up_Time            0x0027   100   100   001    Pre-fail  Always       -       1631
      4 Start_Stop_Count        0x0032   100   100   000    Old_age   Always       -       2516
      5 Reallocated_Sector_Ct   0x0033   100   100   050    Pre-fail  Always       -       3
      7 Seek_Error_Rate         0x000b   100   100   050    Pre-fail  Always       -       0
      8 Seek_Time_Performance   0x0005   100   100   050    Pre-fail  Offline      -       0
      9 Power_On_Hours          0x0032   076   076   000    Old_age   Always       -       9744
     10 Spin_Retry_Count        0x0033   150   100   030    Pre-fail  Always       -       0
     12 Power_Cycle_Count       0x0032   100   100   000    Old_age   Always       -       2449
    191 G-Sense_Error_Rate      0x0032   100   100   000    Old_age   Always       -       374
    192 Power-Off_Retract_Count 0x0032   100   100   000    Old_age   Always       -       98
    193 Load_Cycle_Count        0x0032   084   084   000    Old_age   Always       -       160924
    194 Temperature_Celsius     0x0022   100   100   000    Old_age   Always       -       28 (Min/Max 9/49)
    196 Reallocated_Event_Count 0x0032   100   100   000    Old_age   Always       -       3
    197 Current_Pending_Sector  0x0032   100   100   000    Old_age   Always       -       0
    198 Offline_Uncorrectable   0x0030   100   100   000    Old_age   Offline      -       0
    199 UDMA_CRC_Error_Count    0x0032   200   200   000    Old_age   Always       -       0
    220 Disk_Shift              0x0002   100   100   000    Old_age   Always       -       8251
    222 Loaded_Hours            0x0032   081   081   000    Old_age   Always       -       7778
    223 Load_Retry_Count        0x0032   100   100   000    Old_age   Always       -       0
    224 Load_Friction           0x0022   100   100   000    Old_age   Always       -       0
    226 Load-in_Time            0x0026   100   100   000    Old_age   Always       -       345
    240 Head_Flying_Hours       0x0001   100   100   001    Pre-fail  Offline      -       0

    SMART Error Log Version: 1
    ATA Error Count: 56 (device log contains only the most recent five errors)
        CR = Command Register [HEX]
        FR = Features Register [HEX]
        SC = Sector Count Register [HEX]
        SN = Sector Number Register [HEX]
        CL = Cylinder Low Register [HEX]
        CH = Cylinder High Register [HEX]
        DH = Device/Head Register [HEX]
        DC = Device Command Register [HEX]
        ER = Error register [HEX]
        ST = Status register [HEX]
    Powered_Up_Time is measured from power on, and printed as
    DDd+hh:mm:SS.sss where DD=days, hh=hours, mm=minutes,
    SS=sec, and sss=millisec. It "wraps" after 49.710 days.

    Error 56 occurred at disk power-on lifetime: 9739 hours (405 days + 19 hours)
      When the command that caused the error occurred, the device was active or idle.

      After command completion occurred, registers were:
      ER ST SC SN CL CH DH
      -- -- -- -- -- -- --
      40 41 e2 17 9b 4b 61  Error: UNC at LBA = 0x014b9b17 = 21732119

      Commands leading to the command that caused the error were:
      CR FR SC SN CL CH DH DC   Powered_Up_Time  Command/Feature_Name
      -- -- -- -- -- -- -- --  ----------------  --------------------
      60 08 e0 10 9b 4b 40 00      10:10:10.749  READ FPDMA QUEUED
      ef 10 03 00 00 00 a0 00      10:10:10.748  SET FEATURES [Enable SATA feature]
      ef 10 02 00 00 00 a0 00      10:10:10.748  SET FEATURES [Enable SATA feature]
      27 00 00 00 00 00 e0 00      10:10:10.748  READ NATIVE MAX ADDRESS EXT [OBS-ACS-3]
      ec 00 00 00 00 00 a0 00      10:10:10.748  IDENTIFY DEVICE

    Error 55 occurred at disk power-on lifetime: 9739 hours (405 days + 19 hours)
      When the command that caused the error occurred, the device was active or idle.

      After command completion occurred, registers were:
      ER ST SC SN CL CH DH
      -- -- -- -- -- -- --
      40 41 2a 17 9b 4b 61  Error: UNC at LBA = 0x014b9b17 = 21732119

      Commands leading to the command that caused the error were:
      CR FR SC SN CL CH DH DC   Powered_Up_Time  Command/Feature_Name
      -- -- -- -- -- -- -- --  ----------------  --------------------
      60 08 28 10 9b 4b 40 00      10:10:06.315  READ FPDMA QUEUED
      ef 10 03 00 00 00 a0 00      10:10:06.315  SET FEATURES [Enable SATA feature]
      ef 10 02 00 00 00 a0 00      10:10:06.315  SET FEATURES [Enable SATA feature]
      27 00 00 00 00 00 e0 00      10:10:06.315  READ NATIVE MAX ADDRESS EXT [OBS-ACS-3]
      ec 00 00 00 00 00 a0 00      10:10:06.314  IDENTIFY DEVICE

    Error 54 occurred at disk power-on lifetime: 9739 hours (405 days + 19 hours)
      When the command that caused the error occurred, the device was active or idle.

      After command completion occurred, registers were:
      ER ST SC SN CL CH DH
      -- -- -- -- -- -- --
      40 41 6a 17 9b 4b 61  Error: UNC at LBA = 0x014b9b17 = 21732119

      Commands leading to the command that caused the error were:
      CR FR SC SN CL CH DH DC   Powered_Up_Time  Command/Feature_Name
      -- -- -- -- -- -- -- --  ----------------  --------------------
      60 08 68 10 9b 4b 40 00      10:10:01.882  READ FPDMA QUEUED
      ef 10 03 00 00 00 a0 00      10:10:01.882  SET FEATURES [Enable SATA feature]
      ef 10 02 00 00 00 a0 00      10:10:01.881  SET FEATURES [Enable SATA feature]
      27 00 00 00 00 00 e0 00      10:10:01.881  READ NATIVE MAX ADDRESS EXT [OBS-ACS-3]
      ec 00 00 00 00 00 a0 00      10:10:01.881  IDENTIFY DEVICE

    Error 53 occurred at disk power-on lifetime: 9739 hours (405 days + 19 hours)
      When the command that caused the error occurred, the device was active or idle.

      After command completion occurred, registers were:
      ER ST SC SN CL CH DH
      -- -- -- -- -- -- --
      40 41 3a 17 9b 4b 61  Error: UNC at LBA = 0x014b9b17 = 21732119

      Commands leading to the command that caused the error were:
      CR FR SC SN CL CH DH DC   Powered_Up_Time  Command/Feature_Name
      -- -- -- -- -- -- -- --  ----------------  --------------------
      60 08 38 10 9b 4b 40 00      10:09:57.437  READ FPDMA QUEUED
      ea 00 00 00 00 00 a0 00      10:09:57.437  FLUSH CACHE EXT
      ef 10 03 00 00 00 a0 00      10:09:57.437  SET FEATURES [Enable SATA feature]
      ef 10 02 00 00 00 a0 00      10:09:57.437  SET FEATURES [Enable SATA feature]
      27 00 00 00 00 00 e0 00      10:09:57.437  READ NATIVE MAX ADDRESS EXT [OBS-ACS-3]

    Error 52 occurred at disk power-on lifetime: 9739 hours (405 days + 19 hours)
      When the command that caused the error occurred, the device was active or idle.

      After command completion occurred, registers were:
      ER ST SC SN CL CH DH
      -- -- -- -- -- -- --
      40 41 12 17 9b 4b 61  Error: WP at LBA = 0x014b9b17 = 21732119

      Commands leading to the command that caused the error were:
      CR FR SC SN CL CH DH DC   Powered_Up_Time  Command/Feature_Name
      -- -- -- -- -- -- -- --  ----------------  --------------------
      61 08 b0 58 08 00 40 00      10:09:52.971  WRITE FPDMA QUEUED
      60 08 10 10 9b 4b 40 00      10:09:52.971  READ FPDMA QUEUED
      61 08 a8 b8 0f 44 40 00      10:09:52.971  WRITE FPDMA QUEUED
      ea 00 00 00 00 00 a0 00      10:09:52.971  FLUSH CACHE EXT
      ef 10 03 00 00 00 a0 00      10:09:52.970  SET FEATURES [Enable SATA feature]

    SMART Self-test log structure revision number 1
    Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
    # 1  Extended offline    Completed without error       00%      9743         -
    # 2  Extended offline    Completed: read failure       00%      9732         21732119
    1 of 1 failed self-tests are outdated by newer successful extended offline self-test # 1

    SMART Selective self-test log data structure revision number 1
     SPAN  MIN_LBA  MAX_LBA  CURRENT_TEST_STATUS
        1        0        0  Not_testing
        2        0        0  Not_testing
        3        0        0  Not_testing
        4        0        0  Not_testing
        5        0        0  Not_testing
    Selective self-test flags (0x0):
      After scanning selected spans, do NOT read-scan remainder of disk.
    If Selective self-test is pending on power-up, resume after 0 minute delay.
    #----------------------------------------------------------------------



    #( and no longer shows any offline uncorrectable sectors:

    root@CZ-LIVE:~# smartctl -l selftest /dev/sda | grep "Completed without error"
    # 1  Extended offline    Completed without error       00%      9743         -
    root@CZ-LIVE:~# smartctl -A /dev/sda | grep -iE "Power_On_Hours|G-Sense_Error_Rate|Reallocated|Pending|Uncorrectable"
      5 Reallocated_Sector_Ct   0x0033   100   100   050    Pre-fail  Always       -       3
      9 Power_On_Hours          0x0032   076   076   000    Old_age   Always       -       9744
    191 G-Sense_Error_Rate      0x0032   100   100   000    Old_age   Always       -       374
    196 Reallocated_Event_Count 0x0032   100   100   000    Old_age   Always       -       3
    197 Current_Pending_Sector  0x0032   100   100   000    Old_age   Always       -       0
    198 Offline_Uncorrectable   0x0030   100   100   000    Old_age   Offline      -       0

    # une heure après la fin du test terminé à 9743h et contrôlée à 9744h

    # => Current_Pending_Sector et Offline_Uncorrectable à 0
    # => YES !



    # Test de lecture du disque avec dd !

    root@CZ-LIVE:~# dd if=/dev/sda of=/dev/null bs=64k conv=noerror status=progress
    500079722496 octets (500 GB, 466 GiB) copiés, 9200 s, 54,4 MB/s
    7631040+1 enregistrements lus
    7631040+1 enregistrements écrits
    500107862016 octets (500 GB, 466 GiB) copiés, 9201,35 s, 54,4 MB/s

    # => plus aucune erreur après 2h30 de lecture !!!!




    # Test clonezilla

    root@CZ-LIVE:~# clonezilla 
    Clonezilla mode is device-image
    ocsroot device is local_dev
    Preparing the mount point /home/partimag...
    Si vous désirez utiliser un périphérique USB pour le répertoire image de Clonezilla,
     * insérez ce périphérique *maintenant*.
     * Attendez env. 5 sec.
     * puis appuyez sur Entrée 
    pour laisser le temps de la détection au système. Ce périphérique sera alors monté sous /home/partimag.
    Appuyez sur "Entrée" pour continuer......

    [screen is terminating]
    Mounting local dev as /home/partimag...
    Excluding busy partition or disk...
    Excluding linux raid member partition...
    Excluding busy partition or disk...
    Excluding linux raid member partition...
    Finding partitions.....
    Partition number: 4
    Getting /dev/sda1 info...
    Getting /dev/sda2 info...
    Getting /dev/sda4 info...
    Getting /dev/sdc1 info...
    /dev/sdc1 filesystem: ext4
    LC_ALL=C mount -t auto -o noatime /dev/sdc1 /home/partimag
    Scanning dir /tmp/ocsroot_bind_root....Running: mount --bind -o noatime /tmp/ocsroot_bind_root /home/partimag
    Usage de l'espace disque:
    *****************************************************.
    SOURCE    FSTYPE   SIZE  USED AVAIL USE% TARGET
    /dev/sdc1 ext4   293,3G 19,3G  259G   7% /home/partimag
    *****************************************************.
    Appuyez sur "Entrée" pour continuer......
    done!
    Setting the TERM as xterm
    Starting /usr/sbin/ocs-sr at 2021-12-02 14:22:43 UTC...
    Choose the mode for ocs-sr
    *****************************************************.
    Clonezilla image dir: /home/partimag
    *****************************************************.
    Shutting down the Logical Volume Manager
    Finished Shutting down the Logical Volume Manager
    The image name is: 2021-12-02-14-img_user-tosh_system
    Excluding busy partition or disk...
    Excluding linux raid member partition...
    Finding partitions....
    Partition number: 3
    Getting /dev/sda1 info...
    Getting /dev/sda2 info...
    Getting /dev/sda4 info...
    Selected device [sda1] found!
    The selected devices: sda1
    *****************************************************.
    PS. La prochaine fois vous pourrez exécuter cette commande directement :
    /usr/sbin/ocs-sr -q2 -c -j2 -z1p -i 4096 -sfsck -senc -p choose saveparts 2021-12-02-14-img_user-tosh_system sda1
    Cette commande a été enregistrée sous le nom suivant pour usage ultérieur si nécessaire: /tmp/ocs-2021-12-02-14-img_user-tosh_system-2021-12-02-14-23
    *****************************************************.
    Appuyez sur "Entrée" pour continuer... 
    setterm: terminal xterm does not support --blank
    Activating the partition info in /proc... done!
    Selected device [sda1] found!
    The selected devices: sda1
    Getting /dev/sda1 info...
    *****************************************************.
    La prochaine étape consiste à sauvegarder le disque ou la partition de cette machine sous forme d'une image:
    *****************************************************.
    Machine: Satellite L550
    sda1 (37.3G_ext4(In_TOSHIBA_MK5055GS)_TOSHIBA_MK5055GSXN_Z9PLS4HBS)
    *****************************************************.
    -> "/home/partimag/2021-12-02-14-img_user-tosh_system".
    Etes-vous sûr de vouloir continuer? (y/n) y
    OK, c'est parti !!
    Shutting down the Logical Volume Manager
    Finished Shutting down the Logical Volume Manager
    Saving block devices info in /home/partimag/2021-12-02-14-img_user-tosh_system/blkdev.list...
    Saving block devices attributes in /home/partimag/2021-12-02-14-img_user-tosh_system/blkid.list...
    Checking the integrity of partition table in the disk /dev/sda... 
    Reading the partition table for /dev/sda...RETVAL=0
    *****************************************************.
    The first partition of disk /dev/sda starts at 2048.
    Saving the hidden data between MBR (1st sector, i.e. 512 bytes) and 1st partition, which might be useful for some recovery tool, by:
    dd if=/dev/sda of=/home/partimag/2021-12-02-14-img_user-tosh_system/sda-hidden-data-after-mbr skip=1 bs=512 count=2047
    2047+0 enregistrements lus
    2047+0 enregistrements écrits
    1048064 octets (1,0 MB, 1,0 MiB) copiés, 0,0261043 s, 40,1 MB/s
    *****************************************************.
    done!
    Saving the MBR data for sda...
    1+0 enregistrements lus
    1+0 enregistrements écrits
    512 octets copiés, 0,001437 s, 356 kB/s
    *****************************************************.
    *****************************************************.
    Starting saving /dev/sda1 as /home/partimag/2021-12-02-14-img_user-tosh_system/sda1.XXX...
    /dev/sda1 filesystem: ext4.
    *****************************************************.
    Checking the disk space... 
    *****************************************************.
    Use partclone with pigz to save the image.
    Image file will be split with size limit 4096 MB.
    *****************************************************.
    If this action fails or hangs, check:
    * Is the disk full ?
    *****************************************************.
    Run partclone: partclone.ext4 -z 10485760 -N -L /var/log/partclone.log -c -s /dev/sda1 --output - | pigz -c --fast -b 1024 --rsyncable | split -a 2 -b 4096MB - /home/partimag/2021-12-02-14-img_user-tosh_system/sda1.ext4-ptcl-img.gz. 2> /tmp/split_error.rH3Tug
    Cloned successfully.
    Checking the disk space... 
    >>> Time elapsed: 610.29 secs (~ 10.171 mins)
    Change mode to 600 for these files: /home/partimag/2021-12-02-14-img_user-tosh_system/sda1.ext4-ptcl-img.gz*
    *****************************************************.
    Finished saving /dev/sda1 as /home/partimag/2021-12-02-14-img_user-tosh_system/sda1.ext4-ptcl-img.gz
    *****************************************************.
    End of saveparts job for image /home/partimag/2021-12-02-14-img_user-tosh_system.
    Saving hardware info by lshw...
    Saving DMI info...          
    Saving PCI info...
    Saving S.M.A.R.T. data for the drive...
    Saving OS info from the device...
    Saving package info...
    *****************************************************
    *****************************************************
    Checking if udevd rules have to be restored...
    This program is not started by Clonezilla server, so skip notifying it the job is done.
    Finished!
    Generating a tag file for this image...
    Start checking the saved image 2021-12-02-14-img_user-tosh_system if restorable...
    Setting the TERM as xterm
    L'image à vérifier: 2021-12-02-14-img_user-tosh_system
    Ceci n'est pas une image d'un disque entier mais une image de partitions, sauvegardée depuis les partitions: sda1
    La partition à vérifier: sda1
    setterm: terminal xterm does not support --blank
    *****************************************************.
    Checking the partition table in the image "2021-12-02-14-img_user-tosh_system"...
    Checking sda...
    Partition table file for disk was found: sda
    *****************************************************.
    This is not an image for whole disk. Skip checking swap partition info...
    *****************************************************.
    Checking the MBR in the image "2021-12-02-14-img_user-tosh_system"...
    MBR file for this disk was found: sda
    *****************************************************.
    Checking the partition sda1 in the image "2021-12-02-14-img_user-tosh_system"...
    *****************************************************.
    Checked successfully.
    L'image de cette partition peut être restaurée: sda1
    *****************************************************.
    Toutes les images de partitions ou de périphériques LV de cette image ont été vérifiées et toutes sont restaurables: 2021-12-02-14-img_user-tosh_system
    Summary of image checking:
    ==========================
    Partition table file for disk was found: sda
    This is not an image for whole disk. Skip checking swap partition info...
    MBR file for this disk was found: sda
    L'image de cette partition peut être restaurée: sda1
    Toutes les images de partitions ou de périphériques LV de cette image ont été vérifiées et toutes sont restaurables: 2021-12-02-14-img_user-tosh_system
    ==========================
    The mounted bitlocker device was not found. Skip unmounting it.
    Now syncing - flush filesystem buffers...
    Ending /usr/sbin/ocs-sr at 2021-12-02 14:41:33 UTC...
    Appuyez sur "Entrée" pour continuer......


    root@CZ-LIVE:~# cat /tmp/ocs-2021-12-02-14-img_user-tosh_system-2021-12-02-14-23 
    /usr/sbin/ocs-sr -q2 -c -j2 -z1p -i 4096 -sfsck -senc -p choose saveparts 2021-12-02-14-img_user-tosh_system sda1




    # => pas d'erreur et image restorable !


    # => trop cooooool :DDDDDD !!!
# Si rien n'est possible pour réparer
Penser à photorec ou [ddrescue](https://www.kuhnline.com/how-to-clone-hard-drives-with-ddrescue/)

Voir aussi procédures complètes (entre autres : récupérations de disk HS d'une connaissance : linux/hdd/recovery_failed_disk)
    
