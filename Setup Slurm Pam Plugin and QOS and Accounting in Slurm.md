<h1 class="post-title pad">Setup Slurm Pam Plugin & QOS &Accounting in Slurm</a></h1>

<ol>
<li>Install MariaDB </li>
<li>Slurm database & QOS</li>
<li>Setup Slurm PAM plugin</li>
<li>Accounting in Slurm</li>
</ol>

<p>&nbsp;</p>
<h4><strong>Install MariaDB</strong></h4>
<p>You can install MariaDB to store the accounting that Slurm provides. If you want to store accounting, here&#8217;s the time to do so. I only install this on the server node, buhpc3. I use the server node as our SlurmDB node.</p>
<pre crayon="false" class="">yum install mariadb-server mariadb-devel -y</pre>
<p>We&#8217;ll setup MariaDB later. We just need to install it before building the Slurm RPMs.</p>

<p>&nbsp;</p>
<h4><strong>Slurm database & QOS</strong></h4>
<p>Make sure the MariaDB packages were installed before you built the Slurm RPMs:</p>
<pre crayon="false" class="">
rpm -q mariadb-server mariadb-devel
rpm -ql slurm-sql | grep accounting_storage_mysql.so
</pre>
<p>Start the MariaDB service:</p>
<pre>
systemctl start mariadb
systemctl enable mariadb
systemctl status mariadb
</pre>
<p> Make sure to configure the MariaDB database's root password as instructed at first invocation of the mariadb service, or run this command:</p>
<pre>/usr/bin/mysql_secure_installation </pre>

<p> Select a suitable slurm user's database password. Now follow the accounting page instructions (using -p to enter the database password):</p>
<pre> # mysql -p
mysql> grant all on slurm_acct_db.* TO 'slurm'@'localhost' identified by 'password_of_database' with grant option;
mysql> SHOW VARIABLES LIKE 'have_innodb';
mysql> create database slurm_acct_db;
mysql> quit;</pre>

<h5><strong>MySQL configuration</strong></h5>

<p>The following is recommended for /etc/my.cnf, but on CentOS 7 you should create a new file /etc/my.cnf.d/innodb.cnf containing: </p>
<pre>[mysqld]
innodb_buffer_pool_size=1024M
innodb_log_file_size=64M
innodb_lock_wait_timeout=900 </pre>

<p>The innodb_buffer_pool_size might be even larger, like 50%-80% of the server's RAM size.</p>

<p>To implement this change you have to shut down the database and remove logfiles: </p>
<pre>systemctl stop mariadb
rm /var/lib/mysql/ib_logfile?
systemctl start mariadb </pre>

<p>You can check the current setting in MySQL like so: </p>
<pre>mysql> SHOW VARIABLES LIKE 'innodb_buffer_pool_size';</pre>

<h5><strong>SlurmDBD Configuration</strong></h5>
<p> While the slurmdbd will work with a flat text file for recording job completions and such this configuration will not allow "associations" between a user and account. A database allows such a configuration. </p>
<p> MySQL or MariaDB is the preferred database. To enable this database support one only needs to have the development package for the database they wish to use on the system. Slurm uses the InnoDB storage engine in MySQL to make rollback possible. This must be available on your MySQL installation or rollback will not work. </p>
<p>slurmdbd requires its own configuration file called slurmdbd.conf. Start by copying the example file:</p>
<pre>cp /etc/slurm/slurmdbd.conf.example /etc/slurm/slurmdbd.conf </pre>

<p> The file slurmdbd.conf should be only on the computer where slurmdbd executes and should only be readable by the user which executes slurmdbd (e.g. "slurm"). It must be protected from unauthorized access since it contains a database login name and password. See the slurmdbd.conf man-page for a more complete description of the configuration parameters. </p>
<p> Set up files and permissions: </p>
<pre>chown slurm: /etc/slurm/slurmdbd.conf
chmod 600 /etc/slurm/slurmdbd.conf
touch /var/log/slurm/slurmdbd.log
chown slurm: /var/log/slurm/slurmdbd.log</pre>

<p> </p>
<pre> </pre>

<p> Configure some of the /etc/slurm/slurmdbd.conf variables: </p>
<pre>
DbdHost=XXXX    # Replace by the slurmdbd server hostname 
SlurmUser=slurm
StorageHost=localhost
StoragePass=password    # The above defined database password
StorageLoc=slurm_acct_db</pre>

<h5><strong>Customize the Slurm service files</strong></h5>
<p>If you use a database slurmdbd daemon on the same server as the slurmctld service, the database must be started first. In addition, all Slurm daemons requires the MUNGE service . </p>
<p>Locally customized systemd files must be placed in /etc/systemd/system/, and slurmdbd must depend on the database service, so a more correct solution is: </p>
<p> Copy the delivered service files:</p>
<pre>cp /usr/lib/systemd/system/slurmctld.service /usr/lib/systemd/system/slurmd.service /usr/lib/systemd/system/slurmdbd.service /etc/systemd/system/ </pre>

<p>Add the prerequisite After= services to the file /etc/systemd/system/slurmdbd.service: </p>
<pre>[Unit]
Description=Slurm controller daemon
After=network.target mariadb.service
ConditionPathExists=/etc/slurm/slurmdbd.conf
... </pre>

<p>On compute nodes /etc/systemd/system/slurmd.service should be modified: </p>
<pre>[Unit]
Description=Slurm node daemon
After=network.target munge.service
ConditionPathExists=/etc/slurm/slurm.conf
... </pre>

<p>Only if you use a local database, add the prerequisite After= services to the file /etc/systemd/system/slurmctld.service so that slurmctld depends on the slurmdbd and MUNGE services: </p>
<pre>[Unit]
Description=Slurm controller daemon
After=network.target slurmdbd.service munge.service
ConditionPathExists=/etc/slurm/slurm.conf
... </pre>

<h5><strong>Start the slurmdbd service</strong></h5>
<p>Start the slurmdbd service: </p>
<pre>systemctl enable slurmdbd
systemctl start slurmdbd
systemctl status slurmdbd </pre>

<h5><strong>Configure database accounting in slurm.conf</strong></h5>

<p>In slurm.conf (see slurm.conf) you must configure accounting so that the database will be used through the slurmdbd database daemon: </p>
<pre>AccountingStorageType=accounting_storage/slurmdbd </pre>

<p>&nbsp;</p>

<h4><strong>Setup Slurm PAM plugin</strong></h4>
<p>Setting this up with help block users from casually accessing the compute nodes.</p>
<pre>
# ssh test@buhpc1
test@buhpc1's password: 
Access denied: user alicia (uid=1450) has no active jobs on this node.
Connection closed by 192.168.126.32
</pre>

<p>First make sure Slurm’s PAM module has been installed, it’s supplied by slurm-pam_slurm package:</p>
<pre>
# ls -l /usr/lib64/security/pam_slurm_adopt.so 
-rwxr-xr-x. 1 root root 26368 May 25 14:27 /usr/lib64/security/pam_slurm_adopt.so
</pre>
	
<p>Enable PAM module in Slurm:</p>
<pre>
## add a line in /etc/slurm/slurm.conf
UsePAM=1
</pre>

<p>Enable PAM module in Slurm:</p>
<pre>
## add a line in /etc/pam.d/sshd before any other account setting other than login nodes
account    required     pam_slurm.so
</pre>

<p> Configure pam_access module to always allow admin group (hpcadmins); you may have other rules in access.conf, be careful of their ordering, only first matched rule applies:</p>
<pre>
# cat /etc/security/access.conf
+ : root (hpcadmins) : ALL
- : ALL : ALL
</pre>

<p> Add a pam.d file for slurm: /etc/pam.d/slurm </p>
<pre>
auth    required        pam_localuser.so
account required        pam_unix.so
session required        pam_limits.so
</pre>

<p>&nbsp;</p>
<h4><strong>Accounting in Slurm</strong></h4>

<p>Enable Accounting in Slurm:</p>
<pre>
# add these lines in /etc/slurm/slurm.conf
# ACCOUNTING
AccountingStorageEnforce=1
AccountingStorageLoc=/opt/slurm/acct
AccountingStorageType=accounting_storage/slurmdbd

JobCompLoc=/opt/slurm/jobcomp
JobCompType=jobcomp/slurmdbd

JobAcctGatherFrequency=30
JobAcctGatherType=jobacct_gather/linux
</pre>

<p> restart service:</p>
<pre>
systemctl restart slurmctld  ##in control node
systemctl restart slurmd    
systemctl restart slurmdbd
</pre>

<p>&nbsp;</p>
<h3> Attention！！！</strong></h3>
- <h5>make sure the /etc/slurm/slurm.conf remains consistent in all nodes.</h5>
