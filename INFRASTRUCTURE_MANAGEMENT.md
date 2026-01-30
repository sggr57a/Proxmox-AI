# Infrastructure Management Features

This document describes the advanced infrastructure management capabilities integrated into the Proxmox AI monitoring system (Phases 5-6).

## Overview

Beyond basic monitoring and alerting, the system provides comprehensive infrastructure management including:
- **Snapshot Management**: LVM and Proxmox snapshots for safe operations
- **Backup Orchestration**: Automated backup, verification, and restoration
- **Network Configuration**: Sequential IP allocation, MAC management, hostname assignment
- **Multi-Node Replication**: Template-based cloning across Proxmox clusters
- **Naming Convention Enforcement**: Consistent resource naming across all nodes

## Architecture Principles

### Primary Template Server
- **`pve-node-01`** is designated as the primary template server
- All standard VM/LXC templates live on this node
- Resources created on `pve-node-01` are automatically replicated to other nodes
- Cross-node resources (created on other servers) are also replicated cluster-wide

### Naming Convention
All resources follow the pattern: `pve-{node}-{type}-{seq}`

**Examples:**
```
pve-node-01:
  - pve-test-01      (test container)
  - pve-av-01        (antivirus VM)
  - pve-db-01        (database server)

pve-node-02:
  - pve-test-02      (replica from node-01)
  - pve-av-02        (replica from node-01)
  - pve-web-01       (created locally, replicated to other nodes)
```

### Network Architecture
Each node has its own subnet with dedicated ranges:

| Node | Subnet | VM Range | LXC Range | Gateway |
|------|--------|----------|-----------|---------|
| node-01 | 192.168.1.0/24 | .100-.199 | .200-.254 | .1 |
| node-02 | 192.168.2.0/24 | .100-.199 | .200-.254 | .1 |
| node-03 | 192.168.3.0/24 | .100-.199 | .200-.254 | .1 |

**IP Assignment:**
- Sequential allocation within each range
- Automatic next-available-IP detection
- Database-tracked allocations prevent conflicts
- Static assignment on clone/restore operations

## Feature Details

### 1. LVM Snapshot Management

**Purpose:** Safe rollback for system upgrades and risky operations

**Capabilities:**
- Create LVM snapshots (e.g., `/dev/pve/root`) before major upgrades
- Restore from snapshots on failure
- Automatic retention policies (max age, max count)
- Auto-cleanup of expired snapshots

**Workflow:**
```
Before Major Upgrade:
1. AI detects upgrade action
2. Creates LVM snapshot automatically
3. Proceeds with upgrade
4. On failure: Automatic rollback to snapshot
5. On success: Snapshot retained per retention policy
```

**API Endpoints:**
```bash
POST /api/snapshot/create
{
  "volume": "/dev/pve/root",
  "name": "pre-upgrade-2026-01-29",
  "description": "Before kernel upgrade to 6.5"
}

POST /api/snapshot/restore
{
  "snapshot_id": "pre-upgrade-2026-01-29"
}
```

### 2. VM/LXC Snapshot Management

**Purpose:** Application-level snapshots via Proxmox API

**Capabilities:**
- Create VM/LXC snapshots before modifications
- Restore to previous snapshots
- Snapshot-based rollback on action failure
- Integration with backup workflows

**Use Cases:**
- Before container rebuild
- Before configuration changes
- Before network reconfiguration
- Scheduled snapshot policies

### 3. Backup Orchestration

**Purpose:** Comprehensive backup management with verification

**Capabilities:**
- **Create:** Full VM/LXC backups to configured storage
- **Verify:** Integrity checks on backup files
- **Restore:** Automated restoration with proper renaming
- **Sync:** Cross-node backup synchronization
- **Retention:** Configurable retention (daily/weekly/monthly)

**Backup Types:**
- `snapshot`: Live backup (VM stays running)
- `suspend`: Pause VM during backup
- `stop`: Full stop for consistency

**Workflow:**
```
Automated Backup:
1. Monitoring detects backup schedule
2. Creates snapshot-based backup
3. Verifies backup integrity
4. Syncs to secondary storage
5. Updates backup catalog
6. Cleanup old backups per retention policy
```

**Restoration with Renaming:**
```bash
POST /api/backup/restore
{
  "backup_id": "vzdump-lxc-101-2026_01_29.tar.gz",
  "target_node": "pve-node-02",
  "new_name": "pve-db-02",
  "new_ip": "192.168.2.101",
  "new_mac": "auto-generate"
}
```

### 4. Network Configuration Management

**Purpose:** Automated network setup following conventions

**IP Allocation:**
- Sequential assignment per node
- Next-available-IP lookup
- Database-tracked to prevent conflicts
- DNS integration (optional)

**MAC Address Management:**
- Unique MAC generation using prefix `02:00:00:xx:xx:xx`
- Collision detection
- Persistent MAC assignment per resource

**Hostname Management:**
- Follow naming convention: `pve-{node}-{type}-{seq}`
- Automatic hostname on clone
- DNS record updates (if configured)

**Workflow:**
```
When Cloning VM to new node:
1. Detect target node from metadata
2. Query next available IP in target subnet
3. Generate unique MAC address
4. Set hostname per naming convention
5. Update resource network config
6. Record allocation in database
```

**API Examples:**
```bash
# Get next available IP
GET /api/network/next-ip?node=node-02&type=lxc
Response: { "ip": "192.168.2.201", "gateway": "192.168.2.1" }

# Assign static IP
POST /api/network/assign-ip
{
  "resource_id": "101",
  "resource_type": "lxc",
  "node": "node-02",
  "ip": "192.168.2.201",
  "mac": "02:00:00:a2:c8:01",
  "hostname": "pve-db-02"
}
```

### 5. Multi-Node Replication

**Purpose:** Maintain consistent infrastructure across cluster

**Replication Rules:**
1. **Primary Template Replication**: Resources on `pve-node-01` auto-replicate to all nodes
2. **Cross-Node Replication**: Resources created on other nodes also replicate cluster-wide
3. **Metadata Tracking**: Origin node and template source tracked in database
4. **Naming Enforcement**: Clones renamed per convention with sequential numbering

**Metadata Detection:**
- Resources tagged with origin node on creation
- Clone operations preserve origin metadata
- System detects if resource should be replicated
- Replication tasks queued and tracked

**Clone Workflow:**
```
Template on pve-node-01: pve-db-01 (192.168.1.101)

Replication to pve-node-02:
1. Clone VM/LXC to node-02
2. Rename: pve-db-02
3. Assign IP: 192.168.2.101 (sequential in node-02 range)
4. Generate MAC: 02:00:00:a2:65:01
5. Set hostname: pve-db-02
6. Update metadata:
   - origin_node: pve-node-01
   - origin_template_id: 101
   - is_replica: true
7. Verify clone success
8. Record replication task completion
```

**API Examples:**
```bash
# Get resource metadata
GET /api/metadata/101
Response:
{
  "resource_id": "101",
  "resource_name": "pve-db-01",
  "node_name": "pve-node-01",
  "origin_node": "pve-node-01",
  "is_template": true,
  "is_replica": false,
  "replicated_nodes": ["pve-node-02", "pve-node-03"],
  "network_config": {
    "ip": "192.168.1.101",
    "mac": "02:00:00:a1:65:01",
    "hostname": "pve-db-01"
  }
}

# Trigger replication
POST /api/replicate/101
{
  "target_nodes": ["pve-node-02", "pve-node-03"],
  "clone_type": "full",
  "auto_network": true
}

Response:
{
  "task_id": "repl-101-20260129",
  "status": "running",
  "targets": 2,
  "progress": 0
}

# Check replication status
GET /api/clone/status/repl-101-20260129
Response:
{
  "task_id": "repl-101-20260129",
  "status": "completed",
  "progress": 100,
  "results": [
    {
      "target_node": "pve-node-02",
      "new_resource_id": "201",
      "new_resource_name": "pve-db-02",
      "ip": "192.168.2.101",
      "mac": "02:00:00:a2:65:01",
      "status": "success"
    },
    {
      "target_node": "pve-node-03",
      "new_resource_id": "301",
      "new_resource_name": "pve-db-03",
      "ip": "192.168.3.101",
      "mac": "02:00:00:a3:65:01",
      "status": "success"
    }
  ]
}
```

## Database Schema

### Resource Metadata Tracking
```sql
resource_metadata:
  - Tracks origin node, template source
  - Records replication status
  - Stores network configuration
  - Flags naming convention compliance
  
ip_allocations:
  - Tracks all IP assignments
  - Prevents conflicts
  - Records allocation/release dates
  
snapshots:
  - Catalogs all snapshots (LVM + Proxmox)
  - Retention policies
  - Auto-cleanup tracking
  
backups:
  - Backup inventory
  - Verification status
  - Retention policies
  
replication_tasks:
  - Async replication job tracking
  - Progress monitoring
  - Error logging
```

## Configuration Files

### `config/naming-conventions.yml`

Defines naming patterns, IP ranges, and sequential numbering:

```yaml
naming_pattern: "pve-{node}-{type}-{seq:02d}"

nodes:
  - name: "node-01"
    is_primary_template: true
  - name: "node-02"
  - name: "node-03"

ip_ranges:
  node-01:
    subnet: "192.168.1.0/24"
    vm_range_start: "192.168.1.100"
    lxc_range_start: "192.168.1.200"
```

### `config/action-policies.yml`

Controls which operations require approval:

```yaml
approval_required:
  - restore_snapshot
  - create_backup
  - restore_backup
  - clone_vm
  - clone_lxc
  - replicate_resource
  - assign_static_ip

snapshot_retention:
  max_age_days: 30
  max_snapshots_per_volume: 10

replication:
  primary_node: "pve-node-01"
  auto_replicate_from_primary: true
  cross_node_replication: true
```

## AI Integration

The AI can suggest and execute infrastructure operations:

**Conversation Examples:**

```
User: I need to upgrade the kernel on pve-node-01
AI: Before upgrading, I recommend creating an LVM snapshot of /dev/pve/root. 
    This allows rollback if the upgrade fails. Should I proceed?
    
    Proposed Actions:
    1. Create LVM snapshot (auto-approval)
    2. Upgrade kernel (requires approval)
    3. Verify boot after upgrade
    4. Keep snapshot for 7 days as safety net

User: Clone pve-db-01 to all other nodes
AI: I'll clone pve-db-01 from pve-node-01 to pve-node-02 and pve-node-03.
    
    Clones will be created as:
    - pve-node-02: pve-db-02 (IP: 192.168.2.101)
    - pve-node-03: pve-db-03 (IP: 192.168.3.101)
    
    Each clone will receive a unique MAC address and hostname.
    Do you approve this replication?

User: Restore backup of pve-web-01 to pve-node-02
AI: I found backup: vzdump-lxc-105-2026_01_28.tar.gz
    
    Restore Plan:
    - Target Node: pve-node-02
    - New Name: pve-web-02 (following naming convention)
    - New IP: 192.168.2.102 (next available)
    - New MAC: 02:00:00:a2:66:02 (auto-generated)
    
    Before restoring, should I create a snapshot of the target node?
```

## Safety & Safeguards

**Automatic Snapshots:**
- Before any rebuild operation
- Before configuration changes
- Before major upgrades

**Approval Requirements:**
- Snapshot restoration (can affect data)
- Backup creation (resource intensive)
- Replication (creates multiple resources)
- Network changes (can break connectivity)

**Prohibited Actions:**
- Delete primary templates on `pve-node-01`
- Delete snapshots (only auto-cleanup allowed)
- Modify monitoring infrastructure

**Self-Protection:**
- Cannot replicate or modify N8N containers
- Cannot modify monitoring database
- Cannot delete snapshots created for safety

## Implementation Timeline

These features are implemented across multiple phases:

- **Phase 5**: Metadata collection and tracking
- **Phase 6**: Snapshot, backup, network, replication management
- **Phase 7**: Learning from past operations
- **Phase 8**: Testing and validation

## Future Enhancements

Potential future additions:
- Automated disaster recovery planning
- Cross-datacenter replication
- Bandwidth-aware replication scheduling
- Template versioning and rollback
- Automated capacity planning based on growth patterns
- Integration with external IPAM systems
- DNS record automation (via PowerDNS, BIND, etc.)
