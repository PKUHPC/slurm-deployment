<h1 class="post-title pad">How to Install Slurm on CentOS 7 Cluster</a></h1>

<p class="">Slurm is an open-source workload manager designed for Linux clusters of all sizes. It&#8217;s a great system for queuing jobs for your HPC applications. I&#8217;m going to show you how to install Slurm on a CentOS 7 cluster.</p>
<ol>
<li>Delete failed installation of Slurm</li>
<li>Create the global users</li>
<li>Install  Munge</li>
<li>Install Slurm</li>
<li>Use Slurm</li>
</ol>
<p>&nbsp;</p>
<h4><strong>Cluster Server and Compute Nodes</strong></h4>
<p>I configured our nodes with the following hostnames using these <a href="http://www.slothparadise.com/change-host-name-centos-7/">steps</a>. Our server is:</p>
<pre crayon="false" class="">buhpc3</pre>
<p>The clients are:</p>
<pre crayon="false" class="">buhpc1
buhpc2
buhpc3
buhpc4
buhpc5
buhpc6</pre>
<p>&nbsp;</p>
<h4><strong>Delete failed installation of Slurm</strong></h4>
<p>I leave this optional step in case you tried to install Slurm, and it didn&#8217;t work. We want to uninstall the parts related to Slurm unless you&#8217;re using the dependencies for something else.</p>
<p>First, I remove the database where I kept Slurm&#8217;s accounting.</p>
<pre crayon="false" class="">yum remove mariadb-server mariadb-devel -y</pre>
<p>Next, I remove Slurm and Munge. Munge is an authentication tool used to identify messaging from the Slurm machines.</p>
<pre crayon="false" class="">yum remove slurm munge munge-libs munge-devel -y</pre>
<p>I check if the slurm and munge users exist.</p>
<pre crayon="false" class="">cat /etc/passwd | grep slurm</pre>
<p>Then, I delete the users and corresponding folders.</p>
<pre crayon="false" class="">userdel - r slurm</pre>
<pre crayon="false" class="">userdel -r munge
userdel: user munge is currently used by process 26278</pre>
<pre crayon="false">kill 26278</pre>
<pre crayon="false" class="">userdel -r munge</pre>
<p>Slurm, Munge, and Mariadb should be adequately wiped. Now, we can start a fresh installation that actually works.</p>
<p>&nbsp;</p>
<h4><strong>Create the global users</strong></h4>
<p>Slurm and Munge require consistent UID and GID across every node in the cluster.</p>

<p>If your cluster has been configured, just add some new nodes, you should copy the /etc/munge/munge.key from your configured nodes to all your new nodes. </p>
<pre crayon="false" class="">
scp /etc/munge/munge.key buhpc02:/etc/munge/munge.key</pre>

<p>If you create a new cluster, For all the nodes, before you install Slurm or Munge:</p>
<pre crayon="false" class="">export MUNGEUSER=1001
groupadd -g $MUNGEUSER munge
useradd  -m -c "MUNGE Uid 'N' Gid Emporium" -d /var/lib/munge -u $MUNGEUSER -g munge  -s /sbin/nologin munge
export SLURMUSER=1002
groupadd -g $SLURMUSER slurm
useradd  -m -c "SLURM workload manager" -d /var/lib/slurm -u $SLURMUSER -g slurm  -s /bin/bash slurm</pre>

<p>&nbsp;</p>
<h4><strong>Install Munge</strong></h4>
<p>Since I&#8217;m using CentOS 7, I need to get the latest EPEL repository.</p>
<pre crayon="false" class="">yum install epel-release -y</pre>
<p>Now, I can install Munge.</p>
<pre crayon="false" class="">yum install munge munge-libs munge-devel -y</pre>
<p>After installing Munge, I need to create a secret key on the Server. My server is on the node with hostname, buhpc3. Choose one of your nodes to be the server node.</p>
<p>First, we install rng-tools to properly create the key.</p>
<pre crayon="false" class="">yum install rng-tools -y
rngd -r /dev/urandom</pre>
<p>Now, we create the secret key. You only have to do the creation of the secret key on the server.</p>
<pre crayon="false" class="">/usr/sbin/create-munge-key -r</pre>
<pre crayon="false" class="">dd if=/dev/urandom bs=1 count=1024 &gt; /etc/munge/munge.key
chown munge: /etc/munge/munge.key
chmod 400 /etc/munge/munge.key</pre>
<p>After the secret key is created, you will need to send this key to all of the compute nodes.</p>
<pre crayon="false" class="">scp /etc/munge/munge.key root@1.buhpc.com:/etc/munge
scp /etc/munge/munge.key root@2.buhpc.com:/etc/munge
scp /etc/munge/munge.key root@4.buhpc.com:/etc/munge
scp /etc/munge/munge.key root@5.buhpc.com:/etc/munge
scp /etc/munge/munge.key root@6.buhpc.com:/etc/munge</pre>
<p>Now, we SSH into every node and correct the permissions as well as start the Munge service.</p>
<pre crayon="false" class="">chown -R munge: /etc/munge/ /var/log/munge/
chmod 0700 /etc/munge/ /var/log/munge/</pre>
<pre crayon="false" class="">systemctl enable munge
systemctl start munge</pre>
<p>To test Munge, we can try to access another node with Munge from our server node, buhpc3.</p>
<pre crayon="false" class="">munge -n
munge -n | unmunge
munge -n | ssh 3.buhpc.com unmunge
remunge</pre>
<p>If you encounter no errors, then Munge is working as expected.</p>
<p>&nbsp;</p>
<h4><strong>Install Slurm</strong></h4>
<p>Slurm has a few dependencies that we need to install before proceeding.</p>
<pre crayon="false" class="">yum install openssl openssl-devel pam-devel mariadb-server mariadb-devel numactl numactl-devel hwloc hwloc-devel lua lua-devel readline-devel rrdtool-devel ncurses-devel man2html libibmad libibumad perl-Switch http-parser-devel json-c-devel lua-json -y</pre>
<p>Now, we download the latest version of Slurm preferably in our shared folder. The latest version of Slurm may be different from our version.</p>
<p>download the latest stable version of slurm by click: <a href="https://www.schedmd.com/downloads.php">slurm-17.11.2.TAR.BZ2</a></p>
<pre crayon="false" class="">cd /nfs
</pre>
<p>If you don&#8217;t have rpmbuild yet:</p>
<pre crayon="false" class="">yum install rpm-build python3 cpanm* gcc gcc-c++ -y
rpmbuild -ta -with lua -with hwloc slurm-17.11.2.tar.bz2
</pre>
<p>We will check the rpms created by rpmbuild.</p>
<pre crayon="false" class="">cd /root/rpmbuild/RPMS/x86_64</pre>
<p>Now, we will move the Slurm rpms for installation for the server and computer nodes.</p>
<pre crayon="false" class="">mkdir /nfs/slurm-rpms
cp /root/rpmbuild/BUILD/slurm-17.11.2/src/plugins/auth/munge/.libs/auth_munge.so /usr/lib64/slurm/
cp /root/rpmbuild/BUILD/slurm-17.11.2/src/plugins/cred/munge/.libs/cred_munge.so /usr/lib64/slurm/</pre>
<p>On every node that you want to be a server and compute node, we install those rpms. In our case, I want every node to be a compute node.</p>
<pre crayon="false" class="">yum --nogpgcheck localinstall slurm-17.11.2-1.el7.centos.x86_64.rpm slurm-devel-17.11.2-1.el7.centos.x86_64.rpm 
	slurm-munge-17.11.2-1.el7.centos.x86_64.rpm slurm-perlapi-17.11.2-1.el7.centos.x86_64.rpm slurm-plugins-17.11.2-1.el7.centos.x86_64.rpm 
	slurm-sjobexit-17.11.2-1.el7.centos.x86_64.rpm slurm-sjstat-17.11.2-1.el7.centos.x86_64.rpm slurm-torque-17.11.2-1.el7.centos.x86_64.rpm</pre>
<p>After we have installed Slurm on every machine, we will configure Slurm properly.</p>

<p>I leave everything default except:</p>
<pre crayon="false" class="">ControlMachine: buhpc3
ControlAddr: 128.197.116.18
NodeName: buhpc[1-6]
CPUs: 4
StateSaveLocation: /var/spool/slurmctld
SlurmctldLogFile: /var/log/slurm/slurmctld.log
SlurmdLogFile: /var/log/slurm/slurmd.log
ClusterName: buhpc</pre>
<p>After you hit Submit on the form, you will be given the full Slurm configuration file to copy.</p>
<p>On the server node, which is buhpc3:</p>
<pre crayon="false" class="">cd /etc/slurm
vim slurm.conf</pre>
<p>Copy the form&#8217;s Slurm configuration file that was created from the website and paste it into slurm.conf. We still need to change something in that file.</p>
<p>Underneathe slurm.conf &#8220;# COMPUTE NODES,&#8221; we see that Slurm tries to determine the IP addresses automatically with the one line.</p>
<pre crayon="false" class="">NodeName=buhpc[1-6] CPUs = 4 State = UNKOWN</pre>
<p>I don&#8217;t use IP addresses in order, so I manually delete this one line and change it to:</p>
<pre crayon="false" class="">NodeName=buhpc1 NodeAddr=128.197.115.158 CPUs=4 State=UNKNOWN
NodeName=buhpc2 NodeAddr=128.197.115.7 CPUs=4 State=UNKNOWN
NodeName=buhpc3 NodeAddr=128.197.115.176 CPUs=4 State=UNKNOWN
NodeName=buhpc4 NodeAddr=128.197.115.17 CPUs=4 State=UNKNOWN
NodeName=buhpc5 NodeAddr=128.197.115.9 CPUs=4 State=UNKNOWN
NodeName=buhpc6 NodeAddr=128.197.115.15 CPUs=4 State=UNKNOWN</pre>

<p>After you explicitly put in the NodeAddr IP Addresses, you can save and quit. Here is my full slurm.conf and what it looks like:</p>
<pre crayon="false" class=""># slurm.conf file generated by configurator easy.html.
# Put this file on all nodes of your cluster.
# See the slurm.conf man page for more information.
#
ControlMachine=buhpc3
ControlAddr=128.197.115.176
#
#MailProg=/bin/mail
MpiDefault=none
#MpiParams=ports=#-#
ProctrackType=proctrack/pgid
ReturnToService=1
SlurmctldPidFile=/var/run/slurmctld.pid
#SlurmctldPort=6817
SlurmdPidFile=/var/run/slurmd.pid
#SlurmdPort=6818
SlurmdSpoolDir=/var/spool/slurmd
SlurmUser=slurm
#SlurmdUser=root
StateSaveLocation=/var/spool/slurmctld
SwitchType=switch/none
TaskPlugin=task/none
#
#
# TIMERS
#KillWait=30
#MinJobAge=300
#SlurmctldTimeout=120
#SlurmdTimeout=300
#
#
# SCHEDULING
FastSchedule=1
SchedulerType=sched/backfill
#SchedulerPort=7321
SelectType=select/linear
#
#
# LOGGING AND ACCOUNTING
AccountingStorageType=accounting_storage/none
ClusterName=buhpc
#JobAcctGatherFrequency=30
JobAcctGatherType=jobacct_gather/none
#SlurmctldDebug=3
SlurmctldLogFile=/var/log/slurm/slurmctld.log
#SlurmdDebug=3
SlurmdLogFile=/var/log/slurm/slurmd.log
#
#
# COMPUTE NODES
NodeName=buhpc1 NodeAddr=128.197.115.158 CPUs=4 State=UNKNOWN
NodeName=buhpc2 NodeAddr=128.197.115.7 CPUs=4 State=UNKNOWN
NodeName=buhpc3 NodeAddr=128.197.115.176 CPUs=4 State=UNKNOWN
NodeName=buhpc4 NodeAddr=128.197.115.17 CPUs=4 State=UNKNOWN
NodeName=buhpc5 NodeAddr=128.197.115.9 CPUs=4 State=UNKNOWN
NodeName=buhpc6 NodeAddr=128.197.115.15 CPUs=4 State=UNKNOWN
PartitionName=debug Nodes=buhpc[1-6] Default=YES MaxTime=INFINITE State=UP</pre>

<p>Now that the server node has the slurm.conf correctly, we need to send this file to the other compute nodes.</p>
<pre crayon="false" class="">scp slurm.conf root@1.buhpc.com/etc/slurm/slurm.conf
scp slurm.conf root@2.buhpc.com/etc/slurm/slurm.conf
scp slurm.conf root@4.buhpc.com/etc/slurm/slurm.conf
scp slurm.conf root@5.buhpc.com/etc/slurm/slurm.conf
scp slurm.conf root@6.buhpc.com/etc/slurm/slurm.conf</pre>
<p>Or, you can do this in the manager node to send your file to all nodes in the cluster. </p>
<pre crayon="false" class="">
xdcp all  /etc/slurm/slurm.conf  /etc/slurm/slurm.conf</pre>
<p>Now, we will configure the server node, buhpc3. We need to make sure that the server has all the right configurations and files.</p>
<pre crayon="false" class="">mkdir /var/spool/slurmctld
chown slurm: /var/spool/slurmctld
chmod 755 /var/spool/slurmctld
mkdir  /var/log/slurm
touch /var/log/slurm/slurmctld.log
touch /var/log/slurm/slurm_jobacct.log /var/log/slurm/slurm_jobcomp.log
chown -R slurm:slurm /var/log/slurm</pre>
<p>Now, we will configure all the compute nodes, buhpc[1-6]. We need to make sure that all the compute nodes have the right configurations and files.</p>
<pre crayon="false" class="">mkdir /var/spool/slurmd
chown slurm: /var/spool/slurmd
chmod 755 /var/spool/slurmd
mkdir /var/log/slurm
touch /var/log/slurm/slurmd.log
chown -R slurm:slurm /var/log/slurm</pre>
<p>Use the following command to make sure that slurmd is configured properly.</p>
<pre crayon="false" class="">slurmd -C</pre>
<p>You should get something like this:</p>
<pre crayon="false" class="">ClusterName=(null) NodeName=buhpc3 CPUs=4 Boards=1 SocketsPerBoard=2 CoresPerSocket=2 ThreadsPerCore=1 RealMemory=7822 TmpDisk=45753
UpTime=13-14:27:52</pre>
<p>The firewall will block connections between nodes, so I normally disable the firewall on the compute nodes except for buhpc3.</p>
<pre crayon="false" class="">systemctl stop firewalld
systemctl disable firewalld</pre>
<p>On the server node, buhpc3, I usually open the default ports that Slurm uses:</p>
<pre crayon="false" class="">firewall-cmd --permanent --zone=public --add-port=6817/udp
firewall-cmd --permanent --zone=public --add-port=6817/tcp
firewall-cmd --permanent --zone=public --add-port=6817/udp
firewall-cmd --permanent --zone=public --add-port=6818/tcp
firewall-cmd --permanent --zone=public --add-port=6818/udp
firewall-cmd --permanent --zone=public --add-port=7321/tcp
firewall-cmd --permanent --zone=public --add-port=7321/udp
firewall-cmd --reload</pre>
<p>If the port freeing does not work, stop the firewalld for testing. Next, we need to check for out of sync clocks on the cluster. On every node:</p>
<pre crayon="false" class="">yum install ntp -y
chkconfig ntpd on
ntpdate pool.ntp.org
systemctl start ntpd</pre>
<p class=""><span style="line-height: 1.5;">create cluster with command:</span></p>
<pre crayon="false" class="">sacctmgr create cluster clustername</pre>
<p class=""><span style="line-height: 1.5;">The clocks should be synced, so we can try starting Slurm! On all the compute nodes, buhpc[1-6]:</span></p>
<pre crayon="false" class="">systemctl enable slurmd.service
systemctl start slurmd.service
systemctl status slurmd.service</pre>
<p class="">Now, on the server node, buhpc3:</p>
<pre crayon="false" class="">systemctl enable slurmctld.service
systemctl start slurmctld.service
systemctl status slurmctld.service</pre>
<p class="">When you check the status of slurmd and slurmctld, we should see if they successfully completed or not. If problems happen, check the logs!</p>
<pre crayon="false" class="">Compute node bugs: tail /var/log/slurm/slurmd.log
Server node bugs: tail /var/log/slurm/slurmctld.log</pre>
<p>&nbsp;</p>
<h4><strong>Use Slurm</strong></h4>
<p class="">To display the compute nodes:</p>
<pre crayon="false" class="">scontrol show nodes</pre>
<p class="">-N allows you to choose how many compute nodes that you want to use. To run jobs on the server node, buhpc3:</p>
<pre crayon="false" class="">srun -N5 /bin/hostname</pre>
<pre crayon="false" class="">buhpc3
buhpc2
buhpc4
buhpc5
buhpc1</pre>
<p class="">To display the job queue:</p>
<pre crayon="false" class="">scontrol show jobs</pre>
<pre crayon="false" class="">JobId=16 JobName=hostname
UserId=root(0) GroupId=root(0)
Priority=4294901746 Nice=0 Account=(null) QOS=(null)
JobState=COMPLETED Reason=None Dependency=(null)
Requeue=1 Restarts=0 BatchFlag=0 Reboot=0 ExitCode=0:0
RunTime=00:00:00 TimeLimit=UNLIMITED TimeMin=N/A
SubmitTime=2016-04-10T16:26:04 EligibleTime=2016-04-10T16:26:04
StartTime=2016-04-10T16:26:04 EndTime=2016-04-10T16:26:04
PreemptTime=None SuspendTime=None SecsPreSuspend=0
Partition=debug AllocNode:Sid=buhpc3:1834
ReqNodeList=(null) ExcNodeList=(null)
NodeList=buhpc[1-5]
BatchHost=buhpc1
NumNodes=5 NumCPUs=20 CPUs/Task=1 ReqB:S:C:T=0:0:*:*
TRES=cpu=20,node=5
Socks/Node=* NtasksPerN:B:S:C=0:0:*:* CoreSpec=*
MinCPUsNode=1 MinMemoryNode=0 MinTmpDiskNode=0
Features=(null) Gres=(null) Reservation=(null)
Shared=0 Contiguous=0 Licenses=(null) Network=(null)
Command=/bin/hostname
WorkDir=/root
Power= SICP=0</pre>
<p class="">To submit script jobs, create a script file that contains the commands that you want to run. Then:</p>
<pre crayon="false" class="">sbatch -N2 script-file</pre>
<p class="">Slurm has a lot of useful commands. You may have heard of other queuing tools like torque. Here&#8217;s a useful link for the command differences: <a href="http://www.sdsc.edu/~hocks/FG/PBS.slurm.html" target="_blank">http://www.sdsc.edu/~hocks/FG/PBS.slurm.html</a></p>
<p>&nbsp;</p>



