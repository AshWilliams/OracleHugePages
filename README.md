# Configuring HugePages for Oracle on Linux (x86-64) 

## Introduction 
For large SGA sizes, HugePages can give substantial benefits in virtual memory management. Without HugePages, the memory of the SGA is divided into 4K pages, which have to be managed by the Linux kernel. Using HugePages, the page size is increased to 2MB (configurable to 1G if supported by the hardware), thereby reducing the total number of pages to be managed by the kernel and therefore reducing the amount of memory required to hold the page table in memory. In addition to these changes, the memory associated with HugePages can not be swapped out, which forces the SGA to stay memory resident. The savings in memory and the effort of page management make HugePages pretty much mandatory for Oracle 11g systems running on x86-64 architectures. Just because you have a large SGA, it doesn't automatically mean you will have a problem if you don't use HugePages. It is typically the combination of a large SGA and lots database connections that leads to problems. To determine how much memory you are currently using to support the page table, run the following command at a time when the server is under normal/heavy load.

     grep PageTables /proc/meminfo
     PageTables:      1244880 kB


Automatic Memory Management (AMM) is not compatible with Linux HugePages, so apart from ASM instances and small unimportant databases, you will probably have no need for AMM on a real database running on Linux. Instead, Automatic Shared Memory Management and Automatic PGA Management should be used as they are compatible with HugePages.

## Configuring HugePages
Run the following command to determine the current HugePage usage. The default HugePage size is 2MB on Oracle Linux 5.x and as you can see from the output below, by default no HugePages are defined.

    $ grep Huge /proc/meminfo
     AnonHugePages:         0 kB
     HugePages\_Total:       0
     HugePages\_Free:        0
     HugePages\_Rsvd:        0
     HugePages\_Surp:        0
     Hugepagesize:       2048 kB
    $

Depending on the size of your SGA, you may wish to increase the value of \[Hugepagesize to 1G\](#configuring-1g-hugepagesize). Create a file called "hugepages\_setting.sh" with the following contents.

    !/bin/bash

    #
    # hugepages\_settings.sh
    #
    # Linux bash script to compute values for the
    # recommended HugePages/HugeTLB configuration
    #
    # Note: This script does calculation for all shared memory
    # segments available when the script is run, no matter it
    # is an Oracle RDBMS shared memory segment or not.
    # Check for the kernel version
    KERN=`uname -r | awk -F. '{ printf("%d.%d\\n",$1,$2); }'`
    # Find out the HugePage size
    HPG_SZ=`grep Hugepagesize /proc/meminfo | awk {'print $2'}`
    # Start from 1 pages to be on the safe side and guarantee 1 free HugePage
    NUM_PG=1
    # Cumulative number of pages required to handle the running shared memory segments
    for SEG_BYTES in `ipcs -m | awk {'print $5'} | grep "[0-9][0-9]*"`
    do
      MIN_PG=`echo "$SEG_BYTES/($HPG_SZ*1024)" | bc -q`
      if [ $MIN_PG -gt 0 ]; then
        NUM_PG=`echo "$NUM_PG+$MIN_PG+1" | bc -q`
      fi
    done
    # Finish with results
    case $KERN in
       '2.4') HUGETLB_POOL=`echo "$NUM_PG*$HPG_SZ/1024" | bc -q`;
          echo "Recommended setting: vm.hugetlb_pool = $HUGETLB_POOL" ;;
       '2.6' | '3.8' | '3.10' | '4.1' | '4.14' ) echo "Recommended setting: vm.nr_hugepages = $NUM_PG" ;;
    *) echo "Unrecognized kernel version $KERN. Exiting." ;;
    esac
    # End


Make the file executable.

    $ chmod u+x hugepages\_setting.sh

Edit the "/etc/sysctl.conf" file as the "root" user, adding the following entry, adjusted based on your output from the script. You should set the value greater than or equal to the value displayed by the script. You only need 1 or 2 spare pages.

     vm.nr_hugepages=306
One person reported also needing the hugetlb_shm_group setting on Oracle Linux 6.5. I did not and it is listed as a requirement for SUSE only. If you want to set it, get the ID of the dba group.

    # fgrep dba /etc/group
    dba:x:54322:oracle
    #
Use the resulting group ID in the "/etc/sysctl.conf" file.

    vm.hugetlb_shm_group=54322
Run the following command as the "root" user.

    # sysctl -p
Alternatively, edit the "/etc/grub.conf" file, adding "hugepages=306" to the end of the kernel line for the default kernel and reboot.

You can now see the HugePages have been created, but are currently not being used.

    $ grep Huge /proc/meminfo
    AnonHugePages:         0 kB
    HugePages_Total:     306
    HugePages_Free:      306
    HugePages_Rsvd:        0
    HugePages_Surp:        0
    Hugepagesize:       2048 kB
    $
Add the following entries into the "/etc/security/limits.conf" script or "/etc/security/limits.d/99-grid-oracle-limits.conf" script, where the setting is at least the size of the HugePages allocation in KB (HugePages * Hugepagesize). In this case the value is 306*2048=626688.

    * soft memlock 626688
    * hard memlock 626688
 If you prefer, you can set these parameters to a value just below the size of physical memory of the server. This way you can forget about it, unless you add more physical memory.

Check the MEMORY_TARGET parameters are not set for the database and SGA_TARGET and PGA_AGGREGATE_TARGET parameters are being used instead.

    SQL> show parameter target

    NAME                                 TYPE        VALUE
    ------------------------------------ ----------- ------------------------------
    archive_lag_target                   integer     0
    db_flashback_retention_target        integer     1440
    fast_start_io_target                 integer     0
    fast_start_mttr_target               integer     0
    memory_max_target                    big integer 0
    memory_target                        big integer 0
    parallel_servers_target              integer     16
    pga_aggregate_target                 big integer 200M
    sga_target                           big integer 600M
    SQL>
    Restart the server and restart the database services as required.

Check the HugePages information again.

    $ grep Huge /proc/meminfo
    AnonHugePages:         0 kB
    HugePages_Total:     306
    HugePages_Free:       98
    HugePages_Rsvd:       93
    HugePages_Surp:        0
    Hugepagesize:       2048 kB
    $
You can see the HugePages are now being used.

Remember, if you increase your memory allocation or add new instances, you need to retest the required number of HugePages, or risk Oracle running without them.

## Force Oracle to use HugePages (USE_LARGE_PAGES)
Sizing the number of HugePages correctly is important because prior to 11.2.0.3, if the whole SGA doesn't fit into the available HugePages, the instance will start up without using any. From 11.2.0.3 onward, the SGA can run partly in HugePages and partly not, so the impact of this issue is not so great. Incorrect sizing may not be obvious to spot. Later releases of the database display a "Large Pages Information" section in the alert log during startup.

    ****************** Large Pages Information *****************

    Total Shared Global Region in Large Pages = 602 MB (100%)

    Large Pages used by this instance: 301 (602 MB)
     Pages unused system wide = 5 (10 MB) (alloc incr 4096 KB)
    Large Pages configured system wide = 306 (612 MB)
    Large Page size = 2048 KB
    ***********************************************************
If you are running Oracle 11.2.0.2 or later, you can set the USE_LARGE_PAGES initialization parameter to "only" so the database fails to start if it is not backed by hugepages. You can read more about this here.

    ALTER SYSTEM SET use_large_pages=only SCOPE=SPFILE;
    SHUTDOWN IMMEDIATE;
    STARTUP;
On startup the "Large Page Information" in the alert log reflects the use of this parameter.

    ****************** Large Pages Information *****************
    Parameter use_large_pages = ONLY

    Total Shared Global Region in Large Pages = 602 MB (100%)

    Large Pages used by this instance: 301 (602 MB)
    Large Pages unused system wide = 5 (10 MB) (alloc incr 4096 KB)
    Large Pages configured system wide = 306 (612 MB)
    Large Page size = 2048 KB
    ***********************************************************
Attempting to start the database when there aren't enough HugePages to hold the SGA will now return the following error.

    SQL> STARTUP
    ORA-27137: unable to allocate large pages to create a shared memory segment
    Linux-x86_64 Error: 12: Cannot allocate memory
    SQL> 
    The "Large Pages Information" section of the alert log output describes the startup failure and the appropriate action to take.

    ****************** Large Pages Information *****************
    Parameter use_large_pages = ONLY

    Large Pages unused system wide = 0 (0 KB) (alloc incr 4096 KB)
    Large Pages configured system wide = 0 (0 KB)
    Large Page size = 2048 KB

    ERROR:
      Failed to allocate shared global region with large pages, unix errno = 12.
      Aborting Instance startup.
      ORA-27137: unable to allocate Large Pages to create a shared memory segment

    ACTION:
      Total Shared Global Region size is 608 MB. Increase the number of
      unused large pages to atleast 304 (608 MB) to allocate 100% Shared Global
      Region with Large Pages.
    ***********************************************************
