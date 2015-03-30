MariaDB 10.0 Galera
==================

Based on (nsasikumar/mariadb55-galera)

A reasonably simple Ubuntu 14.04 LTS (phusion/baseimage) container with the pieces to form
a MariaDB 10.0 Galera cluster. It's based on Nick Stenning's container
for MariaDB 5.5, and adds in default support for X.509 based
administrative authentication.

(Repeating the warning from Nick S)

**NB**: Please be aware that by default docker will make the MariaDB
port accessible from anywhere if the host firewall is unconfigured. It
also exposes the Galera wsrep port (by default 4567) globally, as well
as the snapshot transfer port (default 4444). Care should be taken,
perhaps, to firewall off these ports at the docker root level. While
port 3304 might need wider exposure, ports 4567 and 4444 have no need
to be exposed to anything other than members of the cluster.

The root password for the first node (and therefore the entire
cluster) is randomly generated. This password can only be used to
contact the server along the unix domain socket (ie, the default
localhost connection).

The container needs to import two volumes (one read/write, the other
read only). The first volume is the data volume, where the actual
MySQL data is stored.

The second volume (read-only) is the SSL container, which should
contain 3 files:

* a CA certificate (this can be a real CA certificate, or just a
  simple test one, generated via something like TinyCA. This CA
  file should be called `ca.pem`. [^1]
* an SSL private key. This should be set to ownership mode 0600 (or
  even 0400). The file needs to be called `server-key.pem`.
* an SSL server certificate. The CN component of the subject name on
  this certificate should be the DNS name of the host which will be
  the end point for the database service. Note that you can have
  multiple host names by using the subjectAltName X.509 v3 extension
  on the certificate. This file needs to be called `server-cert.pem`.

The data volume must be mounted at the container directory
`/var/lib/mysql`.

The SSL volume must be mounted (ideally read-only) at the container
directory `/etc/ssl/mysql`.

Within the SSL volume there should also be a directory called `root`
which will contain a set of *client* certificates with names like
`joe-root.pem`, `amy-root.pem`, `superman.pem` and so on. These names
will be registered as root capable users from any network host, with
no passwords, but only available using the SSL client certificates.
[^2]

The remote root access is derived from a set of X.509 certificates
which will be used via SSL to authenticate the user, so that passwords
are not passed in via environmental variables. The root password is
also encrypted via those keys and stored on the (exported) data volume
which the container needs.

Decrypting the Root Password
----------------------------

If you need to recover the root password, and you are the holder of
one of the root keys in /etc/ssl/mysql/root, then execute

$ cat _/path/to/mysql-dir/_rootpw.pem | openssl smime -decrypt -inkey
_path/to/private-key.pem_

e.g.

cat /data/mysql-node-1/rootpw.pem | openssl smime -decrypt -inkey
~/ssl-keys/joe-root-key.pem

Note that this prints the root password to standard output. You might
feel better outputting the key into a keyutils key, e.g.

cat /data/mysql-node-1/rootpw.pem | openssl smime -decrypt -inkey
~/ssl-keys/joe-root-key.pem | keyutil padd user mysql-root @us

This can then be used from the mysql command line like so

mysql -uroot -p$(keyctl pipe mysql-root) -h localhost

When your login session ends, the key will then be reaped.

Example Usage
-------------

This sets up a 3 node cluster on 3 different servers, all using
default ports.

We assume that /data/mysql holds the (host, not container) directory
where the MySQL database files need to go, and /data/mysql-ssl
contains the relevant keys, certificates and CA certificates. For the
first node standup, /data/mysql should be empty.

# Boot the first node in standalone mode #

DBID=$(sudo docker run -d -v /var/docker/maria01/mysql:/var/lib/mysql -e NODE_ADDR=my.host.name -v /var/docker/maria01/ssl:/etc/ssl/mysql -e CLUSTER=BOOT inspirationlabs/mariadb-10.0-galera)
DBID=$(sudo docker run -d -v /var/docker/maria02/mysql:/var/lib/mysql -e NODE_ADDR=my.host.name -v /var/docker/maria02/ssl:/etc/ssl/mysql -e CLUSTER=BOOT inspirationlabs/mariadb-10.0-galera)
DBID=$(sudo docker run -d -v /var/docker/maria03/mysql:/var/lib/mysql -e NODE_ADDR=my.host.name -v /var/docker/maria03/ssl:/etc/ssl/mysql -e CLUSTER=BOOT inspirationlabs/mariadb-10.0-galera)

(Note: no need to expose ports 4567 and 4444 - no-one is talking to
anyone just yet)

This should fire up a single MySQL server with the appropriate remote
root user permissions (using SSL client certificates only). Check the
contents of `/data/mysql/mysql.error.log` for whether things worked OK
or not.

Assuming it went well, continue.

# Stop the standalone mode #

sudo docker stop $DBID

This should (gracefully) stop the standalone mode MariaDB node

# Start the node in Primary Component mode #

To begin the cluster, you need a single node as a catch-all donor.

DBID=$(sudo docker run -d -v /var/docker/maria01/mysql:/var/lib/mysql -e NODE_ADDR=my.host.name -v /var/docker/maria01/ssl:/etc/ssl/mysql -p 3304:3304 -p 4567:4567 -p 4444:4444 -e CLUSTER=INIT inspirationlabs/mariadb-10.0-galera)

This should restart the server (with data intact) with the ability to
use it to populate new nodes to fill out the cluster. Node that the
primary component has a node number of 1 (this is fixed by scripting).

Note that ports 4567 and 4444 are open. 4567 is the Galera clustering
service, and 4444 is the service by which snapshots of the database
are transferred across to bring a new node into sync. The current
container implementation uses Xtrabackup, encrypting the transfers via
SSL between nodes. Xtrabackup has the nice property of not blocking
the donor server (well, it does, but only for a very short time),
which means you can continue to write into the donor node while other
nodes are coming online.

# Start the other nodes in the cluster

For each extra node that you need to turn up, set up the `/data/mysql`
and `/data/mysql-ssl` directories as per the primary node, and execute


DBID=$(sudo docker run -d -v /var/docker/maria02/mysql:/var/lib/mysql \
       -v /var/docker/maria02/ssl:/etc/ssl/mysql \
       -e CLUSTER= my.host.name,his.host.name,her.host.name \
       p 3304:3304 -p 4567:4567 -p 4444:4444 \
       -e NODE=2\
       -e NODE_ADDR=my.host.name \
       inspirationlabs/mariadb-10.0-galera)

DBID=$(sudo docker run -d -v /var/docker/maria03/mysql:/var/lib/mysql \
      -v /var/docker/maria03/ssl:/etc/ssl/mysql \
      -e CLUSTER= my.host.name,his.host.name,her.host.name \
      p 3304:3304 -p 4567:4567 -p 4444:4444 \
      -e NODE=2\
      -e NODE_ADDR=my.host.name \
      inspirationlabs/mariadb-10.0-galera)

where `<node number>` is some integer above 1, ideally sequential. The
`his.host.name`, `her.host.name` etc. should be the other nodes in the
cluster. You don't need to list all the nodes in the cluster - the
service will discover the other active nodes upon attempting to join
the cluster - if the list includes the host name of the node on which
the server is running, it will be excluded from the list of nodes to
be interrogated for cluster manifest.

Three nodes is the least number needed to avoid split brain
scenarios.

# (Optional) Restart the primary node as a normal cluster node #

Once the minimum number of synced nodes the cluster reaches 3, you can
stop and restart the primary component as just another node in the
cluster.

(On the server running the primary node)

sudo docker stop $DBID

DBID=$(sudo docker run -d /data/mysql:/var/lib/mysql \
       -v /data/mysql-ssl:/etc/ssl/mysql \
       -p 3304:3304 -p 4567:4567 -p 4444:4444 \
       -e CLUSTER= my.host.name,his.host.name,her.host.name \
       -e NODE=1 \
       -e NODE_ADDR=my.host.name inspirationlabs/mariadb-10.0-galera \
       mariadb-start)

# Generating SSL certificates

1. Generate private key for the CA
openssl genrsa 2048 > ca-key.pem

2. Generate CA
openssl req -new -x509 -nodes -days 3600 -key ca-key.pem -out ca.pem

3. Create server certificate key and request
openssl req -newkey rsa:2048 -days 3600 -nodes -keyout server-key.pem -out server-req.pem

4. Sign server key
openssl x509 -req -in server-req.pem -days 3600 -CA ca.pem -CAkey ca-key.pem -set_serial 01 -out server-cert.pem

5. Create client certificate key and request
openssl req -newkey rsa:2048 -days 3600 -nodes -keyout client-key.pem -out client-req.pem

6. Sign client key
openssl x509 -req -in client-req.pem -days 3600 -CA ca.pem -CAkey ca-key.pem -set_serial 01 -out client-cert.pem

7. Verify certificates
openssl verify -CAfile ca.pem server-cert.pem client-cert.pem

[^1]: http://tinyca.sm-zone.net

[^2]: Note that the root user certificates _must_ be signed by the
same CA as the one signing the server certificate.
