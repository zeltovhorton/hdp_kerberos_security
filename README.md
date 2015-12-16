
Links:


https://github.com/abajwa-hw/security-workshops/blob/master/Setup-kerberos-IPA-23.md
https://github.com/abajwa-hw/security-workshops/blob/master/Security-workshop-HDP%202_3-IPA.md



##Vagrant:

https://github.com/zeltovhorton/vagrant-scp

    git clone https://github.com/u39kun/ambari-vagrant.git
    
    azeltov@hw11813:~/dev/git/ambari-vagrant$ vagrant plugin install vagrant-scp

    azeltov@hw11813:~/dev/git/kerberos/ambari-vagrant/centos7.0$ ./up.sh 3
    
    azeltov@hw11813:~/dev/git/ambari-vagrant/centos7.0$ vagrant scp insecure_private_key c7002:.
    Warning: Permanently added '[127.0.0.1]:2200' (RSA) to the list of known hosts.
    insecure_private_key

##PRE-Req ON ALL Servers:

    systemctl stop firewalld
    systemctl disable firewalld

    systemctl is-enabled ntpd
    systemctl enable ntpd
    
    systemctl stop ntpd
    ntpdate pool.ntp.org
    systemctl start ntpd


##HDP NODES##

https://cwiki.apache.org/confluence/display/AMBARI/Quick+Start+Guide

    sudo su -
   
    wget -nv http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.1.2.1/ambari.repo -O /etc/yum.repos.d/ambari.repo
    
    yum install ambari-server -y
    ambari-server setup -s
    ambari-server start

Start WEB browser: http://c7001.ambari.apache.org:8080/


Do the ambari cluster setup: c70[01-02].ambari.apache.org

Specify the the non-root SSH user ##vagrant##, and upload insecure_private_key file that you copied earlier as the private key.




Host Checks found 18 issues on 2 hosts.
After manually resolving the issues, click Rerun Checks.
To manually resolve issues on each host run the HostCleanup script (Python 2.6 or greater is required):

    python /usr/lib/python2.6/site-packages/ambari_agent/HostCleanup.py --silent --skip=users

Follow the onscreen instructions to install your cluster.
When done testing, run vagrant destroy -f to purge the VMs.


If Hive server does not start, and you get this error:
java.sql.SQLException: Access denied for user 'hive'@'c7002.ambari.apache.org' (using password: YES)

You need to update mysql permissions

    root@c7002:~# sudo mysql
    
    CREATE USER 'hive'@'%';
    GRANT ALL PRIVILEGES ON *.* to 'hive'@'%' WITH GRANT OPTION;
    SET PASSWORD FOR 'hive'@'%' = PASSWORD('pass_goes_here');
    SET PASSWORD = PASSWORD('pass_goes_here');
    FLUSH PRIVILEGES;

## IPA server:
c7003 is the IPA server

http://www.unixmen.com/configure-freeipa-server-centos-7/

https://c7003.ambari.apache.org/ipa/ui/#/e/realmdomains/details

we will install Free IPA server. But there is a client server installation also. We will see that part on later posts. It’s recommended to use RHEL/CentOS >= 6.x or Fedora >= 14. Simply perform a yum install.

    (rhel/centos) # yum install ipa-server
    
    ipa-server-install 

    **Do you want to configure integrated DNS (BIND)? [no]: no**
    
    Enter the fully qualified domain name of the computer
    on which you're setting up server software. Using the form
    <hostname>.<domainname>
    Example: master.example.com.

After login, we need to Change the default shell for all users to /bin/bash. This is done by choosing IPA Server ->Configuration. Once modified, click Update.

/bin/bash

Enable start ipa on reboot

    chkconfig ipa on


On IPA server if you get WARNING: Your system is running out of entropy, you may experience long delays
Run:

Installing haveged on RHEL/CentOS/Fedora
To install haveged on RHEL/CentOS (skip this step for Fedora), you first need to add the EPEL repository by following the instructions on the official site.

Once you've installed and enabled the EPEL repo (on RHEL/CentOS), you can install haveged by running the following command:

     yum install haveged

Fedora users can run the above yum install command with no repository changes. The default options are usually fine, so just make sure it's configured to start at boot:

    chkconfig haveged on

RUN the tutorial to import users:

https://github.com/abajwa-hw/security-workshops/blob/master/Setup-LDAP-IPA.md

##ON IPA Server + HDP nodes:

Setup time to be updated on regular basis to avoid kerberos errors

    echo "service ntpd stop" > /root/updateclock.sh
    echo "ntpdate pool.ntp.org" >> /root/updateclock.sh
    echo "service ntpd start" >> /root/updateclock.sh
    chmod 755 /root/updateclock.sh
    echo "*/2  *  *  *  * root /root/updateclock.sh > /dev/null 2>&1" >> /etc/crontab

## SETUP Kerberos Using Ali Bajwa tutorial


https://github.com/abajwa-hw/security-workshops/blob/master/Setup-kerberos-IPA-23.md


##ON CLIENTS:

    yum install ipa-client openldap-clients -y
    ipa-client-install --domain=ambari.apache.org --server=c7003.ambari.apache.org  --mkhomedir --force-ntpd --ntp-server=north-america.pool.ntp.org -p admin@AMBARI.APACHE.ORG -W


Copy the downloaded csv file to all nodes:

      vagrant scp kerberos.csv c7003:/tmp/


**On the IPA node** Create principals using csv file

 authenticate:

    kinit admin
    awk -F"," '/SERVICE/ {print "ipa service-add --force "$3}' /tmp/kerberos.csv | sort -u > ipa-add-spn.sh
    awk -F"," '/USER/ {print "ipa user-add "$5" --first="$5" --last=Hadoop --shell=/sbin/nologin"}' /tmp/kerberos.csv > ipa-add-upn.sh
    sh ipa-add-spn.sh
    sh ipa-add-upn.sh


###On the HDP node authenticate and create the keytabs
authenticate

    vagrant scp kerberos.csv c7001:/tmp/
    vagrant scp kerberos.csv c7002:/tmp/
      
    sudo kinit admin
    ipa_server=$(cat /etc/ipa/default.conf | awk '/^server =/ {print $3}')
    sudo mkdir /etc/security/keytabs/
    sudo chown root:hadoop /etc/security/keytabs/
    awk -F"," '/'$(hostname -f)'/ {print "ipa-getkeytab -s '${ipa_server}' -p "$3" -k "$6";chown "$7":"$9,$6";chmod "$11,$6}' /tmp/kerberos.csv | sort -u > gen_keytabs.sh
    sudo bash ./gen_keytabs.sh



    sudo sudo -u hdfs kinit -kt /etc/security/keytabs/nn.service.keytab nn/$(hostname -f)@AMBARI.APACHE.ORG
    sudo sudo -u ambari-qa kinit -kt /etc/security/keytabs/smokeuser.headless.keytab ambari-qa@AMBARI.APACHE.ORG
    sudo sudo -u hdfs kinit -kt /etc/security/keytabs/hdfs.headless.keytab hdfs@AMBARI.APACHE.ORG

==============================================





Appendix:

export SERVICE=FREEIPA
export PASSWORD=admin
export AMBARI_HOST=sandbox.hortonworks.com
export CLUSTER=Sandbox

#get service status
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X GET http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/$SERVICE


curl -u admin:admin -i -H 'X-Requested-By: ambari' -X DELETE http://c7001.ambari.apache.org:8080/api/v1/clusters/$CLUSTER/services/FREEIPA

> Written with [StackEdit](https://stackedit.io/).





















