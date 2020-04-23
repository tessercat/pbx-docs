    export VERSION=1.2.16
    apt -y install g++ make libboost-dev libssl-dev
    cd /usr/local/src
    git clone --depth 1 https://github.com/jselbie/stunserver
    cd stunserver
    make
    tar -czvf stunserver-$VERSION.tgz stunserver
    ./stunserver --family 4 --primaryinterface <ipv4-addr>
    ./stunserver --family 6 --primaryinterface <ipv6-addr>
