# Nginx with HTTP3 QUIC BoringSSL Brotli
Compile Nginx with BoringSSL to support HTTP3 &amp; QUIC along with Brotli for better Compression

## Installing Required Libraries, Dependencies and Tools
We are choosing "/root" folder for all of our compilations and installations. 
Note: Be really careful, if you decide to change location.
      
    cd /root

Check if Server is up-to-date with Latest Patches 
   
    sudo apt-get update
    sudo apt-get upgrade

Install Dev tools needed to compile Nginx    
    
    sudo apt-get install -y \
          dpkg-dev
          uuid-dev
          build-essential \
          cmake \
          git \
          gnupg \
          gnupg-curl \
          golang \
          libunwind-dev \
          libpcre3-dev \
          curl \
          zlib1g-dev \
          libcurl4-openssl-dev


## Installing Nginx from Mainline

Create Key Signature to Download Latest Repo from Nginx. Although Nginx Team is maintaining a separate branch to support HTTP3 at https://hg.nginx.org/nginx-quic
    
    sudo wget https://nginx.org/keys/nginx_signing.key
    sudo apt-key add nginx_signing.key

Add Nginx Repository to our Source List at **/etc/apt/sources.list** and add following lines

    deb [arch=amd64] https://nginx.org/packages/mainline/ubuntu YourUbuntuVersion nginx
    deb-src https://nginx.org/packages/mainline/ubuntu YourUbuntuVersion nginx

Replace YourUbuntuVersion in above lines with the correct version of your Ubuntu installation 
e.g jammy would replace YourUbuntuVersion for Ubuntu 22
Ubuntu 22.04: jammy
Ubuntu 20.10: groovy
Ubuntu 20.04.1 LTS: focal
Ubuntu 18.04.5 LTS: bionic
Ubuntu 16.04.7 LTS: xenial
Ubuntu 14.04.6 LTS: trusty
Ubuntu 12.04.5 LTS: precise

Download, Update and Upgrade our Repositories
    
    sudo apt-get update
    sudo apt-get upgrade

Build Dependencies for Nginx and pull Source Code to directory
    
    sudo apt-get build-dep nginx
    sudo apt-get source nginx

After executing code above, it will create a folder named nginx_XXXXX. XXXXX is the version number of Ubuntu.

## Installing Nginx QUIC

Since we have now the NGINX source code, what we need to do now is to replace the code with NGINX quic. 
But before that, we have to install mercurial as this is needed to compile the NGINX.
    
    sudo apt-get install mercurial

Then we will clone the latest NGINX repo from https://hg.nginx.org/nginx-quic.
    
    hg clone -b quic https://hg.nginx.org/nginx-quic

Let’s overwrite all content of our nginx-quic directory to the nginx source code folder.
    
    rsync -r nginx-quic/ nginx-XXXXX

## Installing BoringSSL

Install BoringSSL (A Fork version of OpenSSL by Google to support QUIC and HTTP3 Protocol)
    
    git clone https://github.com/google/boringssl     

Installation of BoringSSL has not effect on Nginx or Nginx QUIC. You can install BoringSSL before Nginx too.

Above command will clone and extract a folder named boringssl in /root. If the directory name is change like
boringssl-master etc then change it to boringssl. 
Remember boringssl directory should contain all files like crypto, include, ssl etc
then execute these commands

    cd boringssl    
    mkdir build 
    cd /root/boringssl/build
    cmake ..
    make

    cd /root/boringssl
    mkdir -p .openssl/lib
    ln -s /include/ /.openssl/include
    cp /build/crypto/libcrypto.a /.openssl/lib
    cp /build/ssl/libssl.a /.openssl/lib


## Adding BoringSSL and HTTP/3 to NGINX

After that, we have to add important configuration for our NGINX quic, 
we can add inside the rules file at NGINX source code folder.
    
    vim  /root/nginx-XXXXX/debian/rules

Then add the following lines that says config.env.nginx and config.env.nginx_debug, 
you can find it at line 41 and line 46. Then add the following after –with-stream_ssl_preread_module:
    
    --with-http_v3_module --with-http_quic_module --with-stream_quic_module 

Then we need to add -Wno-ignored-qualifiers at the CFLAGS = “” to disable compiler from throwing qualifiers error. 
It should be:
    
    CFLAGS="-Wno-ignored-qualifiers"

Next we have to add the boringssl module to –with-cc-opt and –with-ld-opt with the following values. 
You need to replace the existing option at the bottom of the config, or else you’ll get an saying like,
            “./auto/configure: error: certain modules require OpenSSL QUIC support. 
             You can either do not enable the modules, or install the OpenSSL library into the system, 
             or build the OpenSSL library statically from the source with nginx by using 
            –with-openssl= option.”. 
See the final rules for reference.
    
    --with-cc-opt="-I/root/boringssl/include $(CFLAGS)" --with-ld-opt="-L/root/boringssl/build/ssl -L/root/boringssl/build/crypto $(LDFLAGS)"

Make sure you do the same changes on both line config.env.nginx and config.env.nginx_debug.


## Installing Brotli

Just clone the repository in root folder
    
    cd /root
    git clone --recursive https://github.com/google/ngx_brotli

To add Brotli to NGINX, similar to adding BoringSSL and http/3 on our NGINX, 
we have to add it on the debian/rules compile configuration.
    
    vim  /root/nginx-XXXXX/debian/rules

At the ./configure of config.env.nginx and config.env.nginx_debug, 
add the following lines right after --sbin-path=/usr/sbin/nginx:
    
    --add-module="/root/ngx_brotli"

(If for some reason Brotli causes problem, try moving /root/ngx_brotli directory inside /root/nginx-XXXXX/debian/modules
and change /root/nginx-XXXXX/debian/rules respectively)


## Compiling NGINX with HTTP3 + QUIC + BoringSSL + Brotli

Before we compile, we have to add some finishing touch so we distinguish we are using mod build for NGINX.
First, we have to edit the changelog file.
    
    vim /root/nginx-XXXXX/debian/changelog

Then add somename like XXXXX-2~jammy+nginx+boringssl on the first line. XXXXX is the Nginx latest version we just downloaded like 1.23.2-1
    
    nginx (XXXXX-2~jammy+nginx+boringssl) jammy; urgency=low

    * XXXXX-2

    -- YOUR_NAME <your_name@domain.com>  Tue, 24 Nov 2022 16:02:03 +0300

After that, we can now compile our NGINX.
    
    cd /root/nginx-XXXXX/
    sudo dpkg-buildpackage -b

Once it started compiling, it will asked if you want to include PSOL debug version, 
just type yes. After it completes, it will create the deb file at the parent directory. 

You can then use those to install nginx.
    
    sudo dpkg -i nginx_XXXXX-2~jammy+nginx+boringssl_amd64.deb

From here we can now check if our new NGINX quic is now running.
    
    nginx -t
    
If some error is caused, it is always helpful to compile Nginx with BoringSSL only and test it. Then compile with Brotli, test it separately.


If you want to include any external module then you might want to compile it with latest version of Nginx. Follow

https://www.nginx.com/blog/failed-nginx-plus-upgrade-module-version-x-instead-of-y/


There is always someone to Thank alongside. Some of the Good Articles for compiling Nginx with Boring SSL

https://codefaq.org/server/how-to-install-http-3-quic-on-nginx-server-for-ubuntu/#installing-boringssl
https://quic.nginx.org/readme.html
