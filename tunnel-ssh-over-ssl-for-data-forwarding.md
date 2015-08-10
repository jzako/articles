# Tunnel SSH over SSL for data forwarding

## Background

I'm in two restrictive firewalls:

1. Local working network blocks some ports (e.g. port 22) and probably allows outbound https traffic. VPN is also not allowed (protocols like pptp/l2tp are disabled).
1. PRC's GFW blocks specified outbound traffic.

## Goal

Accoss GFW to access blocked services at local working network.

## How

**Data Flow：**
Local TCP Application <-> Local SSH Client <-> Remote SSH Server <-> Destination Server

SSH provides a useful function called port-forwarding (SSH tunnel). Basically there are 3 ways of forwarding:

1. Local forwarding. (ssh -Nfl 8888:www.google.com:80 a.b.c.d)
1. Remote forwarding. (ssh -NFR 2222:localhost:22 a.b.c.d)
1. Dynamic forwarding. (ssh -NfD 2222 a.b.c.d)

Here what we need is using **dynamic forwarding** to build a **socks proxy**. To achieve our goal, we have to prepare a remote server that could access services we need.

> **Recommendation:** AWS EC2 has one year free charge, build your instance in a nearby region (singapore or japan for me).

To push through the limitation of port 22 blocking at local network, one possible solution is to build tunnel ssh over ssl. **Stunnel** is choosed to do the trick.

Stunnel can be used to provide secure encrypted connections for clients or servers that do not speak TLS or SSL natively. It relies on a separate library, such as OpenSSL or SSLeay, to implement the underlying TLS or SSL protocol.

### Server Side Instructions

    sudo apt-get install stunnel4

    openssl genrsa 1024 > stunnel.key
    openssl req -new -key stunnel.key -x509 -days 3650 -out stunnel.crt
    cat stunnel.crt stunnel.key > stunnel.pem
    sudo mv stunnel.pem /etc/stunnel/

    sudo tee /etc/stunnel/stunnel.conf << EOF
    pid = /var/run/stunnel.pid
    cert = /etc/stunnel/stunnel.pem

    [ssh]
    accept = 443
    connect = 127.0.0.1:22
    EOF

    # Enable auto start
    sudo sed -i 's/ENABLED=0/ENABLED=1/' /etc/default/stunnel4

    sudo service stunnel4 start

    netstat -natp | grep :443

### Client Side Instructions (OSX)

    brew install stunnel

    # Here I failed to execute this command for I do not have write access to /usr/local/share/man/man8 folder. So I change the folder owner first.
    # sudo chown my_name /usr/local/share/man/man8
    # brew link stunnel

The client configuration also needs the same SSL certificate that we generated above. copy/paste or otherwise duplicate the .pem file we generated in the Server Side Instructions and save it to the same location, /usr/local/etc/stunnel/stunnel.pem.

    sudo tee /usr/local/etc/stunnel/stunnel.conf << EOF
    client = yes
    pid = /usr/local/var/run/stunnel.pid  # may need **mkdir -p /usr/local/var/run** first
    cert = /usr/local/etc/stunnel/stunnel.pem  # point to the certification file we just copy-pasted.

    [ssh]
    accept = 127.0.0.1:2200
    connect = remote_server_ip:443
    EOF

    /usr/local/bin/stunnel /usr/local/etc/stunnel/stunnel.conf

Now you can make the ssh connection over ssl and server as a socks5 proxy. To ease you life, make your ssh config:

    .ssh/config

    Host proxy
      Hostname 127.0.0.1
      Port 2200
      User ubuntu
      DynamicForward 7070

### Last mile

The Last mile totally depends on your need:

- If you want global access to your socks proxy, make the configuration on your network.
- If you just want web page surfing to use proxy, make the configuration for specified browser.
- If you just want some application (e.g. git command to github.com) to use proxy, make the configuration separately.

## Reference

- [Tunnel SSH over SSL](https://ubuntu-tutorials.com/2013/11/27/tunnel-ssh-over-ssl/) - By Christer Edwards
- [SSH Forward](https://www.ibm.com/developerworks/cn/linux/l-cn-sshforward/)
- [Stunnel Wikipedia](https://en.wikipedia.org/wiki/Stunnel)
