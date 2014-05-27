vzhaipd
=======

#### Name

vzhaipd -- a set of Virtuozzo/OpenVZ mount/umount Container scripts that will make sure a HA IP is always present on one of two or more Containers

### Description

The purpose of this script is to make sure that one or more IP addresses are always available on one of two or more containers

 WARNING: Make sure this script as well as HA_IP_FILE and VZHAIPD_CONFIG are chmod 0700 resp. 0600.
          Also note that if either of the two reside within a container,
          the container's root user might mess up your host system!
          And please be careful what START_CMD and STOP_CMD you define. Both can be
          overridden in VZHAIPD_CONFIG!
NOTE: This script is meant to be run with two or more containers each running on a different host system. if you run the script with two containers on the same host system it will work, however it will constantly try to become master, fail and fall back to slave mode since it won't get it's arp requests answered. this is not a problem per se but it's ugly and also having a ha script run on the same hardware doesn't seem to be a very smart choice now does it?

### Installation overview

Just copy the two scripts to your /vz/private/<CTID>/scripts/ directory. If you're still using Virtuozzo 3 copy them to /etc/sysconfig/vz-scripts/<CTID>.mount and /etc/sysconfig/vz-scripts/<CTID>.umount. Though I strongly suggest to upgrade to Virtuozzo 4 and vzctl convert <CTID>. The reason for this is that while VZ3 only migrates it's mount script, VZ4 will migrate the whole 'scripts' directory. So with VZ4 it is much easier to maintain an external config file.

### Configuration overview

- VZHAIPD_CONFIG There's several options you can set either directly within this script (not recommended) or inside an external config file/script. The path and name of the external config file are defined in VZHAIPD_CONFIG. The default value is /vz/private/<CTID>/scripts/vzhaipd.conf If you need to change this path you obviously have to do it inside the mount script. It is important that this script has EXACTLY the access rights defined in FILE_PERMISSIONS. Otherwise the mount script will shut down upon start. More explanation about this behaviour follows in the FILE_PERMISSIONS section.

- HA_CLUSTER_IP The IP address this script will check for. To do so what this script does is it sends out an arp request on HA_INTERFACE. If it gets an answer it will assume that HA_CLUSTER_IP is available and go to slave mode. However should it get no answer after ON_FAILURE_TAKE_OVER_AFTER tries it will switch to master mode. Which means that it will first add HA_CLUSTER_IP as well as any additional IPs defined in HA_IP_FILE (if any) to the container. If this succeeds it will execute START_CMD.

- HA_INTERFACE The host system's network interface the mount script will send out it's arp requests on. Usually eth0, eth1 or bond0.

- HA_IP_FILE Path to the file that contains additional IPs that should be added/removed to the container upon becoming master/slave. The mount script will check for this file upon config load and than again every HA_IP_FILE_CHECK_INTERVAL seconds. The format of the file should be one IPv4 address per line. If a line does not look like an IP address the script will ignore it and continue with the next line. The HA_IP_FILE is not affected by the FILE_PERMISSIONS setting as we only read from it and don't execute anything. However you should still make sure the file has proper access permissions as you might not want unauthorized users to add/delete IP addresses without you knowing.

- START_CMD This command is executed when we become master. To be exact it is executed after all IPs have been added to the container and only if we were able to add the successfully. Be aware that this command is executed on the host system! If you like to execute a command inside the container you should prefix this with 'vzctl exec $CONTAINER_ID ...' The return value of this command does not have any effect on anything. If this command is not defined nothing will be executed

- STOP_CMD This command is executed when we become slave. It is executed before all IPs are removed from the container. Also it is executed 30 seconds after the container is started and changes it's state to 'running'. The return value of this command does not have any effect on anything. If this command is not defined nothing will be executed

- LOG_LEVEL A number between 0 and 900. It defines what messages are logged. Messages with higher severity have lower logging numbers. Something between 70-99 will produce a reasonable amount of output. Anything equal to or above 100 will produce quite alot of output.

- LOG_FILE The file we are supposed to write our log output to. If this isn't set the script will output to stdout/stderr

- DAEMON_PIDFILE The script's pid file. It is strongly recommended that it contains the CONTAINER_ID. Otherwise, if you run multiple instances of this script they will kill each other. The pid file is created with FILE_UMASK access permissions. The file is checked every PID_CONSISTENCY_CHECK_INTERVAL seconds for consistency. That is the script will check if the access permissions are equal to FILE_PERMISSIONS. Also if will check if it's own pid is the same as the one from the pid file. If it isn't it will check if the pid defined in the file is a running process. If so it will end itself, assuming that it was replaced by a new instance of this script. However if there's no other process running with this pid it will overwrite the file with it's own pid, thereby making itself the authoritative instance of this script again.

- HA_CLUSTER_IP_CHECK_INTERVAL Defines the interval between arp checks in seconds. Something between 3 - 30 is a reasonable number. To put it in simple words this is how often the script will check if HA_CLUSTER_IP is available.

- HA_IP_FILE_CHECK_INTERVAL Defines how often we should check for changes of HA_IP_FILE. This means that you can add or remove IPs to HA_IP_FILE while this script is running, without the need to restart the container or this script. The script will check the last modified timestamp of HA_IP_FILE and if it differs from the one of it's last load it will reload the file. Any new IP addresses will automatically be added after MASTER_FRESHNESS_CHECK_INTERVAL.

- ON_FAILURE_CHECK_INTERVAL Defines the interval between arp checks if the first one failed. I.e. you might define something like HA_IP_FILE_CHECK_INTERVAL=15 and ON_FAILURE_CHECK_INTERVAL=3. This means that the script will check if HA_CLUSTER_IP is available every 15 seconds. If a check should fail all following checks (defined by ON_FAILURE_TAKE_OVER_AFTER) will be done after waiting only 3 seconds instead of 15.

- ON_FAILURE_TAKE_OVER_AFTER As explained in section ON_FAILURE_CHECK_INTERVAL this defines the number of IP availability checks the script will do before deciding that the other container is probably down and becoming master itself. Something like 3 or 5 is a reasonable value.

- MASTER_FRESHNESS_CHECK_INTERVAL Defines how often we try to make sure that the container has all IPs assigned that are defined in HA_IP_FILE. In other words when the script sees that HA_IP_FILE has changed and reloads it, this function will make sure that the newly added IPs are also added to the container. However it will not remove any IPs from the container. So if you removed IPs from HA_IP_FILE the only way to remove them from the container is to either shut it down / restart it or to let the container become slave and than master again. How this is done is explained in a later section.

- PID_CONSISTENCY_CHECK_INTERVAL How often we check if our pid file has the correct access permissions and if the pid inside the file is the same as our own. If the file does not have the correct access permissions the function will return and not read from the pid file. So make sure that FILE_UMASK (the permissions the file is created with) and FILE_PERMISSIONS (the permissions the script checks for) are the same. Though not literally - the one is a umask the other are permission bits. Or to make it easy FILE_UMASK+FILE_PERMISSIONS must add up to 777 otherwise this check will return without further checking the pid files content. If the permission check succeeeds the script will read the pid inside the file and continue it's checks as described in section DAEMON_PIDFILE.

- USE_ACCURATE_TIME This can be 'true' or 'false'. If it's set to 'true' than the script will execute the 'date' command whenever it displays or calculates a time/date related thing. If it's set to false it will only update the current time value every daemon cycle which is relatively close to HA_CLUSTER_IP_CHECK_INTERVAL seconds. For normal operation you can leave this set to false. However if you prefer a more accurate timing and more accurate time stamps in your log you can set it to true. However know this will execute 'date' like 50 times per cycle. Which on modern systems won't have any considerable impact. Still it's usually an unnecessary waste of resources.

- FILE_UMASK The umask files will get created with. Currently only the pid file is affected by this setting.

- FILE_PERMISSIONS The permission bits we should check for when checking file permissions. This should be 777-FILE_UMASK.

- CMD_ARPSEND The arpsend command. On default Virtuozzo installations it is safe to just leave this as 'arpsend' as the command is within your $PATH. However if you want to you can set the full path to the binary.

- CMD_VZCTL The vzctl command. Everything said for CMD_ARPSEND applies here too.

- CMD_VZLIST The vzlist command. Everything said for CMD_ARPSEND applies here too.

### License and Copyright

   Copyright [2008-2009] [scoyo GmbH]

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
