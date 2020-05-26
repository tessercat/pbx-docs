# Nginx and nchan

Debian buster nginx packages are 1.14.2,
and it's hard to tell
which version of nchan is provided
by the `libnginx-mod-nchan` package since
the `nchan_stub_status` directive
doesn't list version.
According to the
[nchan changelog](https://github.com/slact/nchan/blob/master/changelog.txt),
it's a version previous to 1.1.5 (May 3 2017)
that adds version to `nchan_stub_status` output.

I tried compiling nchan as a module
to work with the default buster nginx package,
but convoluted `configure` params
make it basically impossible
to compile a module that
the nginx binary will load,
but the nchan module compiles and loads
with no problem using the nginx Linux package repos.


## Links

- [nchan](https://github.com/slact/nchan)
- [Linux packages](https://nginx.org/en/linux_packages.html)
- [Dynamic modules](https://www.nginx.com/blog/compiling-dynamic-modules-nginx-plus/)


## Compile nchan

Create a new Debian buster VM and prep it.

    apt -y update
    apt -y full-upgrade
    reboot

Install nginx.

    apt -y install curl
    echo "deb http://nginx.org/packages/debian `lsb_release -cs` nginx"     | sudo tee /etc/apt/sources.list.d/nginx.list
    curl -fsSL https://nginx.org/keys/nginx_signing.key | sudo apt-key add -
    apt -y update
    apt -y install nginx

Compile nchan.

    apt -y install git build-essential libpcre3-dev zlib1g-dev
    cd /usr/local/src/
    git clone https://github.com/slact/nchan

    nginx -v
    VERSION=1.18.0
    wget https://nginx.org/download/nginx-$VERSION.tar.gz
    tar -xzvf nginx-$VERSION.tar.gz
    cd nginx-$VERSION/
    ./configure --with-compat --add-dynamic-module=../nchan
    cp objs/ngx_nchan_module.so /etc/nginx/modules/

Load the module in `/etc/nginx/nginx.conf`.

    load_module modules/ngx_nchan_module.so;

Add a stub status location
to the default server
in `/etc/nginx/conf.d/default.conf`.

    location /nchan_stub_status {
        nchan_stub_status;
    }

Test and archive the module.

    nginx -t
    curl localhost/nchan_stub_status
    tar -czvf ngx_nchan_module-$VERSION.tgz ngx_nchan_module.so
