## 3. Download Nexus Repository tar and install on both Nexus VMs

Perform these installation steps on **both** Nexus VMs. Use the same Nexus version and keep the application installation and working directories consistent.

```bash
sudo dnf install -y wget
cd /opt
sudo wget https://download.sonatype.com/nexus/3/nexus-3.96.0-09-linux-x86_64.tar.gz
sudo useradd --system --home-dir /opt/sonatype --shell /sbin/nologin nexus
sudo mkdir -p /opt/sonatype
sudo mkdir -p /opt/sonatype-work/nexus3
sudo tar -xvzf nexus-3.96.0-09-linux-x86_64.tar.gz
sudo ln -s /opt/nexus-3.96.0-09 /opt/sonatype/nexus
sudo chown -R nexus:nexus /opt/sonatype
sudo chown -R nexus:nexus /opt/sonatype-work
```

If the `nexus` user already exists, do not create it again. Check the existing user and paths first.

### Configure the Nexus service user

Edit `/opt/sonatype/nexus/bin/nexus.rc`:

```bash
run_as_user="nexus"
```

### Configure the external PostgreSQL datastore

Create the configuration directory:

```bash
sudo mkdir -p /opt/sonatype-work/nexus3/etc/fabric
```

Create `/opt/sonatype-work/nexus3/etc/fabric/nexus-store.properties` with the database credentials and private database address:

```properties
username=nexus
password=<REPLACE_WITH_STRONG_PASSWORD> ###Password for nexus db user
jdbcUrl=jdbc\:postgresql\://<POSTGRESQL_PRIVATE_IP>\:5432/nexus?currentSchema=nexus
```

Create or update `/opt/sonatype-work/nexus3/etc/nexus.properties`:

```properties
nexus.datastore.enabled=true
```

Set ownership:

```bash
sudo chown -R nexus:nexus /opt/sonatype-work/nexus3
```

Do not put real credentials or private IP addresses in a public repository.

## 4. Configure NFS on the current active Nexus VM

In my lab: the **active Nexus VM also hosts the NFS export**.

Install the NFS utilities:

```bash
sudo dnf install -y nfs-utils
```

Create the export directory and set its ownership and permissions:

```bash
sudo mkdir -p /mnt/nexus-blob
sudo chown nexus:nexus /mnt/nexus-blob
sudo chmod 770 /mnt/nexus-blob
```

Edit `/etc/exports` to allow only the private IP addresses of the active and passive Nexus clients:

```exports
/mnt/nexus-blob <ACTIVE_NEXUS_PRIVATE_IP>(rw,sync,no_subtree_check) <PASSIVE_NEXUS_PRIVATE_IP>(rw,sync,no_subtree_check)
```

Apply and verify the export:

```bash
sudo exportfs -rav
sudo exportfs -v
sudo systemctl enable --now nfs-server
sudo systemctl status nfs-server
```


## 5. Mount the NFS export on both Nexus nodes

Create the mount point on each Nexus node:

```bash
sudo mkdir -p /data/nexus-blob
```

Mount the export, replacing `<ACTIVE_NEXUS_PRIVATE_IP>` with the private IP of the VM currently hosting NFS:

```bash
sudo mount -t nfs4 \
  -o defaults,noatime,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,vers=4.1 \
  <ACTIVE_NEXUS_PRIVATE_IP>:/mnt/nexus-blob \
  /data/nexus-blob
```

Validate the mount and write access:

```bash
findmnt /data/nexus-blob
df -h /data/nexus-blob
#Ensure nexus user has ownership and required permissions, if not please provide the same.
sudo -u nexus touch /data/nexus-blob/nfs-write-test
sudo -u nexus ls -l /data/nexus-blob/nfs-write-test
```

**Security note:**  During lab testing, I encountered permission errors when accessing the NFS-mounted blob-store directory at /data/nexus-blob. Although the directory was owned by the nexus user on the NFS server, access through the mounted path did not behave as expected. Disabling SELinux allowed the write operation to proceed, indicating that SELinux restrictions may have contributed to the issue.

I also tested the no_root_squash export option while troubleshooting NFS permissions:

/mnt/nexus-blob <ACTIVE_NEXUS_PRIVATE_IP>(rw,sync,no_subtree_check,no_root_squash) <PASSIVE_NEXUS_PRIVATE_IP>(rw,sync,no_subtree_check,no_root_squash)

The `no_root_squash` option prevents the NFS server from mapping root requests from the authorized clients to the anonymous user. It increases security risk because remote root users can retain root privileges on the exported filesystem. It does not directly resolve permission errors for processes running as the nexus user.

For production deployments, keep the default root_squash behavior where possible, verify numeric UID/GID consistency between the NFS server and Nexus nodes, and troubleshoot filesystem permissions and SELinux denials. Disabling SELinux or enabling no_root_squash should not be treated as a general-purpose fix for NFS permission issues.


For persistent mounts, add the following line to `/etc/fstab` on both Nexus nodes:

```fstab
<ACTIVE_NEXUS_PRIVATE_IP>:/mnt/nexus-blob /data/nexus-blob nfs4 defaults,noatime,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,vers=4.1,_netdev 0 0
```

Validate the entry:

```bash
sudo systemctl daemon-reload
sudo mount -a
findmnt /data/nexus-blob
```

The export IP in my lab is the active Nexus VM. In a production topology with independent NAS/NFS, use the NAS/NFS endpoint instead.

## 6. Create the Nexus systemd service

Create `/etc/systemd/system/nexus.service` on both Nexus VMs:

```ini
[Unit]
Description=Nexus Repository
After=network-online.target
Wants=network-online.target

[Service]
Type=forking
LimitNOFILE=65536
ExecStart=/opt/sonatype/nexus/bin/nexus start
ExecStop=/opt/sonatype/nexus/bin/nexus stop
User=nexus
Restart=on-abort
TimeoutSec=600

[Install]
WantedBy=multi-user.target
```

Enable the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable nexus.service
```

Start Nexus on the node you intend to use as active:

```bash
sudo systemctl start nexus
sudo systemctl status nexus
curl -I http://localhost:8081
```

**Only one Nexus instance should actively use the same repository database and shared blob-store data at a time.** 
Keep the Passive instance stopped until it is deliberately activated under the manual failover procedure.

## 7. Configure the blob store in the Nexus UI

After Nexus starts, sign in to the UI and create a blob store say `ankur-blob` that points to the mounted path:

```text
/data/nexus-blob
```

Then configure repositories to use the intended blob store where the installed Nexus edition allows it. 

# Note:- We need to create new repos to use our new blob stores, so for instance if we are using maven then we can create 

```text
ankur-maven-release
ankur-maven-snapshot
ankur-maven-proxy
ankur-maven-group

```

All pointing to our nexus blob store with the maven group has mapping of proxy, snapshot and release.


Please note, the default blob store association for built-in repositories such as Maven Public, Maven Releases, and Maven Snapshots could not be edited and needs separate blob-store to be created other than default.

Verify the configured blob store by uploading a small test artifact and checking that Nexus can read it back.

## 8. Stop nexus services on active and start on passive. Ensure NFS mount and DB is accessible.

```bash
sudo systemctl start nexus
sudo systemctl status nexus
curl -I http://localhost:8081
```

Validate that able to write to the blob store.

## 9. What my lab demonstrate

### Demonstrated in the setup notes

- Nexus Repository `3.96.0-09` running on RHEL 9.8.
- External PostgreSQL `18.4` datastore configuration.
- NFSv4.1 export and mount configuration for the blob-store path.
- A systemd unit for managing the Nexus process.
- Basic service and mount validation commands.

### Limitations to address before production use

- **NFS single point of failure:** The export is hosted on the active Nexus VM. Failure of that VM can make the blob store unavailable to the passive node. Use independent shared storage for the intended production architecture.
- **PostgreSQL availability:** The database is on a separate VM in my lab, but a single database VM is still a dependency and a potential single point of failure.
- **Manual failover:** Traffic switching and activating the passive node must be controlled to avoid both nodes writing to the same data simultaneously.
- **Storage sizing:**  EC2 root disk was 10 GiB. This is not a general storage recommendation for Nexus artifacts; size persistent storage based on repository growth, retention, and recovery requirements.
- **Failover validation:** Test the exact failure scenarios you intend to recover from. Stopping the Nexus process is not the same as losing the VM that hosts the NFS export.
- **Security:** Restrict database and NFS access to the required private clients. Do not publish passwords, admin tokens, private IP addresses, or unredacted command output.

This document is a manual lab based implementation test, not a claim that it is Sonatype's documented or certified high-availability architecture. Review the official Sonatype documentation and licensing requirements before designing a production HA deployment.                                
