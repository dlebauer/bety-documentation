### Overview

BETYdb runs on a Red Hat Enterprise Linux version 5.8 Server. To simulate this environment, we have set up a CentOS 5.8 server at pecandev.igb.illinois.edu for testing 

### Create an netinstall of the CentOS ISO 

### Boot from CD and Install 

* Following instructions here: http://www.if-not-true-then-false.com/2010/centos-netinstall-network-installation/
* Download this iso: http://vault.centos.org/5.8/isos/x86_64/CentOS-5.8-x86_64-netinstall.iso
 * it is the "netinstall" version, small enough to fit on a CD, but requires internet to install
 * burn to CD
 * boot from CD
* ftp server: `vault.centos.org`
* directory: `5.8/os/x86_64`

### Configuration

1. Add new user
 ```
adduser johndoe
 ```
2. add user to root
 ```
sudo su
emacs /etc/sudoers
 ```
3. add the line 
```
johndoe  ALL=(ALL)  ALL
```

4. Keep time updated (source: [ServerFault](http://serverfault.com/q/368602))

```{bash}
yum install ntp
/sbin/chkconfig ntpd on         ## check that it is on
/usr/sbin/ntpdate pool.ntp.org  ## set system date
/etc/init.d/ntpd start
```

### Add new repository

instructions here: http://www.rackspace.com/knowledge_center/article/installing-rhel-epel-repo-on-centos-5x-or-6x

```
wget http://dl.fedoraproject.org/pub/epel/5/x86_64/epel-release-5-4.noarch.rpm
sudo rpm -Uvh epel-release-5*.rpm
```

... move to PEcAn wiki page for build environment https://github.com/PecanProject/pecan/wiki/Development-Environment-Setup-and-VM-Creation#install-build-environment

Remember to install R 3.0:
```
wget http://cran.us.r-project.org/src/base/R-3/R-3.0.1.tar.gz
tar xzf R-3.0.1.tar.gz
cd R-3.0.1
./configure
make
sudo make install
```

###Site data installation

```
cd /usr/local/ebi

rm -rf sites
curl -o sites.tgz http://isda.ncsa.illinois.edu/~kooper/EBI/sites.tgz
tar zxf sites.tgz
sed -i -e "s#/home/kooper/projects/EBI#${PWD}#" sites/*/ED_MET_DRIVER_HEADER
rm sites.tgz

rm -rf inputs
wget http://isda.ncsa.illinois.edu/~kooper/EBI/inputs.tgz
tar zxf inputs.tgz
rm inputs.tgz
```

###Database Creation

```
#sets up the user for the bety database
mysql -u root -p -e "create database test; create database production;"
mysql -u root -p -e "grant all on bety.* to bety@localhost identified by 'bety';"
wget -O /usr/local/ebi/updatedb.sh http://isda.ncsa.illinois.edu/~kooper/EBI/updatedb.sh
chmod 755 /usr/local/ebi/updatedb.sh
```

Open up updatedb.sh and remove the two lines

```
#change to home
cd
```

If these are left in, the script will attempt to put the site data in ~/sites instead of /usr/local/ebi/sites.

```
#fetch and updates the bety database
./updatedb.sh
```

###Ruby installation
The version of ruby available through yum is too low, so we have to use rvm
```
user$ \curl -L https://get.rvm.io | sudo bash -s stable
rvm install 1.9
rvm use 1.9

yum upgrade rubygem

yum install mysql-devel.x86_64
yum install ImageMagick-devel.x86_64
yum install rubygem-rails
yum install httpd-devel
wget http://www.sqlite.org/2013/sqlite-autoconf-3071700.tar.gz
tar -xzf sqlite-autoconf-3071700.tar.gz
cd sqlite-autoconf-3071700.tar.gz
./configure
make
make install
```

Then all the ruby gems bety needs.
```
cd /usr/local/bety
gem install bundler
bundle install
```
Configuration for Bety:
```
cd /usr/local/ebi/bety

# create folders for upload folders
mkdir paperclip/files paperclip/file_names
chmod 777 paperclip/files paperclip/file_names

# create folder for log files
mkdir log
touch log/production.log
chmod 0666 log/production.log
touch log/test.log
chmod 0666 log/test.log

cat > config/database.yml << EOF
production:
  adapter: mysql2
  encoding: latin1
  reconnect: false
  database: bety
  pool: 5
  username: bety
  password: bety

test:
  adapter: mysql2
  encoding: latin1
  reconnect: false
  database: test
  pool: 5
  username: bety
  password: bety
EOF


# setup login tokens ( actual REST_AUTH_SITE_KEY needs to be replaced with one matching the DB )
cat > config/initializers/site_keys.rb << EOF
REST_AUTH_SITE_KEY         = 'thisisnotasecret'
REST_AUTH_DIGEST_STRETCHES = 10
EOF



# configure apache
ln -s /usr/local/ebi/bety/public /var/www/bety

# http://wiki.apache.org/httpd/DistrosDefaultLayout
cat > /etc/httpd/conf.d/bety.conf << EOF
RailsEnv production
RailsBaseURI /bety
DocumentRoot /usr/local/ebi/bety/public/
ServerName pecandev.igb.illinois.edu
<Directory /usr/local/beta/public >
        Options FollowSymLinks
        AllowOverride None
        Order allow,deny
        Allow from all
</Directory>
EOF
```
You may have to change your DocumentRoot in /etc/httpd/conf/httpd.conf from "/var/www/html" to "/var/www" if you get the error message 'Passenger error #2 An error occurred while trying to access '/var/www/html/bety': Cannot resolve possible symlink '/var/www/html/bety': No such file or directory (2)'.
Up next make apache2 and passenger play nicely:
```
rvmsudo passenger-install-apache2-module
```
If that fails, try 
```
sudo -s
source `rvm gemdir`
```

Finally run the tests 
```
cd /usr/local/eby/bety/
rake db:test:load && rake db:fixtures:load RAILS_ENV=test && rspec spec/
```


###Install Models
Install the models biocro, sipnet, and ED as they are installed [here](https://github.com/PecanProject/pecan/wiki/Installing-PEcAn#install-models). 

For biocro, there are some dependencies:
```
echo "install.packages('devtools',repos='http://cran.rstudio.com/')"| R --vanilla
yum install udunits2-devel
sudo R CMD INSTALL udunits2 --configure-args='--with-udunits2-include=/usr/include/udunits2'
```

###Pecan Installation

```
# download pecan
cd /usr/local/ebi
git clone https://github.com/PecanProject/pecan.git


yum install netcdf-devel.x86_64 netcdf-static.x85_64 openmpi hdf5-devel.x86_64 openmpi-devel
#install PEcAn dependencies
wget http://cran.r-project.org/src/contrib/ncdf4_1.6.1.tar.gz


rpm -Uvh http://www.hdfgroup.org/ftp/HDF5/current/bin/RPMS/hdf5-1.8.11-1.with.szip.encoder.el5.x86_64.rpm

wget http://cran.r-project.org/src/contrib/rjags_3-10.tar.gz

wget http://www.unidata.ucar.edu/downloads/netcdf/ftp/netcdf-4.3.0.tar.gz

tar -xzf netcdf-4.3.0.tar.gz
cd netcdf-4.3.0
export LDFLAGS="/usr/include/netcdf-3/"
sudo ./configure
sudo make install
cd ..
tar -xzf ncdf4_1.6.1.tar.gz
sudo R CMD INSTALL --configure-args="--with-nc-config=~/netcdf-4.3.0/nc-config" ncdf4


rpm -Uvh http://www6.atomicorp.com/channels/atomic/centos/5/x86_64/RPMS/sqlite-3.7.0.1-1.el5.art.x86_64.rpm

wget   http://download.osgeo.org/gdal/1.10.0/gdal-1.10.0.tar.gz
tar -xzf gdal-1.10.0.tar.gz
cd gdal-1.10.0
sudo ./configure
sudo make install

sudo yum install atlas blas.x86_64
sudo rpm -Uvh  http://download.opensuse.org/repositories/home:/cornell_vrdc/CentOS_CentOS-5/x86_64/jags3-3.3.0-48.1.x86_64.rpm
sudo rpm -Uvh http://download.opensuse.org/repositories/home:/cornell_vrdc/CentOS_CentOS-5/x86_64/jags3-devel-3.3.0-48.1.x86_64.rpm

yum install proj-devel.x86_64 udunits2-devel.x86_64

wget http://cran.r-project.org/src/contrib/rgdal_0.8-9.tar.gz
tar -xzf rgdal_0.8-9.tar.gz
echo "/usr/local/lib" >> /etc/ld.so.conf.d/gdal.conf
ldconfig -v
R CMD INSTALL rgdal --configure-args='--with-gdal-config=/usr/local/bin/gdal-config'

# install PEcAn packages in R
cd pecan
R --vanilla < scripts/install.dependencies.R

# compile pecan
./scripts/build.sh
```

### PostgreSQL

Although BETY currently uses MySQL, support for or migration to PostgresSQL is planned.

#### Installing Postgres

[reference](http://wiki.postgresql.org/wiki/YUM_Installation)
add `exclude=postgresql*` to the `[base]` and `[updates]` sections of  `/etc/yum.repos.d/CentOS-Base.repo`

```
# download repo
curl -O http://yum.postgresql.org/9.2/redhat/rhel-5-x86_64/pgdg-centos92-9.2-6.noarch.rpm
# install repo
rpm -ivh pgdg-centos92-9.2-6.noarch.rpm 
# list available packages
yum list postgres*
# install postgres 9.2
yum install postgresql92-server

# initialize database
 /sbin/service postgresql-9.2 initdb

# automatically start postgres 
/sbin/chkconfig postgresql-9.2 on
```

#### configuring postgres [reference](http://wiki.postgresql.org/wiki/First_steps)

```
# set root (postgres) password [reference](http://stackoverflow.com/q/12720967)
sudo -u postgres psql
alter user postgres with password 'new_password' 
```

## Installing Virtualbox

[reference](http://wiki.centos.org/HowTos/Virtualization/VirtualBox#head-81de410879b8e7f18a127f638160e036ab99684e)

```
cd /etc/yum.repos.d
wget http://download.virtualbox.org/virtualbox/rpm/rhel/virtualbox.repo
yum install dkms
yum groupinstall "Development Tools"
yum install kernel-devel
yum install VirtualBox-4.2
usermod -a -G vboxusers <username>
```