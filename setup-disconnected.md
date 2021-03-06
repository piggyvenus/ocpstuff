## This doc describes how setup content mirrors for disconnected installs

#### # You must first sync the development rpm repos from the internal build servers
###### # make sure you have ample space available on your local repo box (called repo.home.nicknach.net in my lab).  
###### # You need at least 30GB total.  (~10GB  for rpms and ~20GB for images)

##### # subscribe your repo box to the proper channels for OCP
```
subscription-manager register --username=nnachefs@redhat.com --password <REDACTED> --force
subscription-manager attach --pool=8a85f98260c27fc50160c323263339ff
subscription-manager repos --disable="*"
subscription-manager repos \
   --enable=rhel-7-server-rpms \
   --enable=rhel-7-server-extras-rpms \
   --enable=rhel-7-server-ose-3.7-rpms \
   --enable=rhel-7-fast-datapath-rpms \
   --enable=rhel-7-server-rhscl-rpms \
   --enable=rhel-7-server-optional-rpms 
#   --enable=rh-gluster-3-for-rhel-7-server-rpms \ 
#   --enable=rhel-7-server-3scale-amp-2.0-rpms
```
##### # install/enable/start httpd
```
yum -y install httpd && systemctl enable httpd --now
mkdir /var/www/html/repos
```
##### # start the reposync
```
cd ~ && wget https://raw.githubusercontent.com/nnachefski/ocpstuff/master/sync_repos.sh && chmod +x sync_repos.sh
./sync_repos.sh
```
##### # fix selinux
``` 
restorecon -R /var/www/html/repos
```
#### # now lets create the docker image mirror on our repo server
##### # install/enable/start docker-distribution on the repo box
```
yum -y install docker-distribution.x86_64 && systemctl enable docker-distribution --now
```
##### # open the firewall up
```
firewall-cmd --set-default-zone trusted
```
##### # now run the import-image.py script
```
cd ~ && wget https://raw.githubusercontent.com/nnachefski/ocpstuff/master/images/import-images.py && wget https://raw.githubusercontent.com/nnachefski/ocpstuff/master/images/images.txt && chmod +x import-images.py
./import-images.py docker registry.access.redhat.com repo.home.nicknach.net:5000 -t v3.7 -d
```
#### # done with repo box now

#### # on your client boxes now
##### # add your rpm repos
```
yum-config-manager --disable \* && rm -rf /etc/yum.repos.d/*.repo && yum clean all
yum-config-manager --add-repo http://repo.home.nicknach.net/repo/rhel-7-server-rpms
yum-config-manager --add-repo http://repo.home.nicknach.net/repo/rhel-7-fast-datapath-rpms
yum-config-manager --add-repo http://repo.home.nicknach.net/repo/rhel-7-server-extras-rpms
yum-config-manager --add-repo http://repo.home.nicknach.net/repo/rhel-7-server-ose-3.7-rpms
yum-config-manager --add-repo http://repo.home.nicknach.net/repo/rhel-server-rhscl-7-rpms
yum-config-manager --add-repo http://repo.home.nicknach.net/repo/rhel-7-server-optional-rpms 
#yum-config-manager --add-repo http://repo.home.nicknach.net/repo/rh-gluster-3-for-rhel-7-server-rpms
```
##### # add your docker registry
```
sed -i '16,/registries =/s/\[\]/\[\"repo.home.nicknach.net:5000\"\]/' /etc/containers/registries.conf
systemctl restart docker
```

