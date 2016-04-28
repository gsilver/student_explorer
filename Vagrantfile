# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
    config.vm.box = "ubuntu/trusty64"
    config.vm.provider "virtualbox" do |v|
        v.memory = 2048
    end

    config.vm.network "forwarded_port", guest: 8000, host: 2081
    config.vm.network "forwarded_port", guest: 3306, host: 2034

    if File.directory?("../django-unizindata/unizindata/")
        config.vm.synced_folder "../django-unizindata/unizindata/", "/usr/local/lib/python2.7/dist-packages/unizindata"
    end

    config.vm.provision "shell", inline: <<-SHELL
        set -xe

        cd /vagrant

        echo "America/Detroit" > /etc/timezone
        dpkg-reconfigure -f noninteractive tzdata

        apt-get update
        apt-get dist-upgrade -y

        # "debconf-set-selections" lines are need to prevent the mysql-server install from prompting for a password
        debconf-set-selections <<< 'mysql-server mysql-server/root_password password 12345'
        debconf-set-selections <<< 'mysql-server mysql-server/root_password_again password 12345'

        apt-get --no-install-recommends install --yes mysql-server libmysqlclient-dev
        apt-get --no-install-recommends install --yes python-pip python-dev
        apt-get --no-install-recommends install --yes apache2 apache2-utils
        apt-get --no-install-recommends install --yes libldap2-dev libsasl2-dev
        apt-get --no-install-recommends install --yes libfontconfig

        echo -e "[mysqld]\nbind-address = 0.0.0.0" > /tmp/mysqld_bind_vagrant.cnf
        mv -f /tmp/mysqld_bind_vagrant.cnf /etc/mysql/conf.d/
        service mysql restart

        cat <<EOM > /etc/apache2/sites-available/001-se_proxy.conf
<VirtualHost *:80>
        ErrorLog \${APACHE_LOG_DIR}/error.log
        CustomLog \${APACHE_LOG_DIR}/access.log combined
        ProxyPass / http://0.0.0.0:8000/
        ProxyPassReverse / http://0.0.0.0:8000/

        RequestHeader set Proxy-User "foobar"

</VirtualHost>
EOM
        a2enmod rewrite headers proxy_*

        a2dissite 000-default.conf
        a2ensite 001-se_proxy.conf

        service apache2 restart

        echo "CREATE DATABASE IF NOT EXISTS student_explorer;" | mysql -v -u root -p12345
        echo "GRANT ALL PRIVILEGES ON *.* TO 'student_explorer'@'%' IDENTIFIED BY 'student_explorer';" | mysql -v -u root -p12345

        pip install coverage
        pip install -r requirements.txt

        if [ ! -e student_explorer/settings/local.py ]; then
            cp student_explorer/settings/local_sample.py student_explorer/settings/local.py
        fi

        echo "Installing Bower..."        
        cd /vagrant
        apt-get update
        apt-get install --yes git
        curl --silent --location https://deb.nodesource.com/setup_4.x | sudo bash -
        apt-get install --yes nodejs
        npm install --global npm@latest
        npm install --global bower

        if [ -d /vagrant/student_explorer/extras ]; then
            cd /vagrant/student_explorer/extras
            ls *.deb; if [ $? -eq 0 ]; then
                apt-get --no-install-recommends install --yes libaio1 libaio-dev
                dpkg -i *.deb
                export ORACLE_HOME=/usr/lib/oracle/12.1/client64
                export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$ORACLE_HOME/lib
                export PATH=$PATH:$ORACLE_HOME/bin
                echo "$ORACLE_HOME/lib" > /etc/ld.so.conf.d/oracle.conf;
                ldconfig
                pip install cx_Oracle
            fi
        fi
    SHELL
end
