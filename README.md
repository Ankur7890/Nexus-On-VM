# Important Disclaimer

This setup demonstrates a lab-based approach to running Nexus Repository on two VMs with an external PostgreSQL database and a shared NFS blob-store directory.

The systemd service configuration, shared-storage setup, and manual failover procedure are an implementation approach tested in my lab. They are not intended to represent Sonatype's official or certified high-availability architecture.

The objective is to explore whether a passive Nexus VM can be activated when the active VM becomes unavailable, provided the database and blob-store dependencies remain accessible and the former active instance is confirmed to be stopped.

For production deployments, review Sonatype's official documentation, supported architecture, edition-specific capabilities, and licensing requirements before adopting this design.

# Nexus Repository Active-Standby on RHEL AMI 9.8 (Manual Setup)

This README filters the useful installation and configuration steps from my setup notes. It removes terminal output, repeated commands, troubleshooting attempts, and sensitive values.

## 1. Environment

- AWS region: `ap-south-1`
- OS: Red Hat Enterprise Linux 9.8 AMI
- Nexus Repository: `3.96.0-09
- PostgreSQL: `18.4`
- Database name: `nexus`
- PostgreSQL schema: `nexus`
- PostgreSQL extension: `pg_trgm`
- Nexus data directory: `/opt/sonatype-work/nexus3`
- Nexus installation: `/opt/nexus-3.96.0-09`
- Blob-store mount point on Nexus: `/data/nexus-blob`
- NFS export directory on the current active node: `/mnt/nexus-blob`

The lab used three EC2 instances: one PostgreSQL VM, one active Nexus VM, and one passive Nexus VM. Instance sizing in the lab was `c7i-flex.large` (2 vCPU, 4 GiB RAM), with a 10 GiB root disk. Size storage according to actual artifact volume and operational requirements before using this design beyond a lab.

**Note**: The VPC, security groups, and Availability Zone (AZ) configuration should be planned according to your environment and availability requirements.

In the current setup, three VMs are deployed across separate AZs within the same AWS Region: two VMs for Nexus Repository and one VM for PostgreSQL. Public connectivity is provided through an Internet Gateway, while internal communication between the VMs is controlled using security groups.

> **Important topology note:** In this lab, the NFS server is installed on the active Nexus VM. That means the NFS service and the blob store are not independent of the active VM. If that VM fails, the passive Nexus node can lose access to the blob store. For a production/real-time design, place the shared storage on an independent NAS/NFS service accessible from both Nexus nodes.
