# Container Policy Interface (CPI)

**note a** : This is a stub that have many sections copied from the [CSI spec](https://github.com/container-storage-interface/spec) as a base to jumpstart the CPI spec draft. Contents are subject to constant changes.

**note b**: The architecture description please refer to the [Policy WG Proposal Doc](https://docs.google.com/document/d/1Ht8wpj4j9YfAA7aVv9Yn3Ci1T_MLMWt0DBr0QmxI2OM/edit?usp=sharing)

**note c**: Discussion are encouraged to post on [Policy WG Google Group](https://groups.google.com/forum/#!forum/kubernetes-wg-policy)

## Authors:

* Howard Huang <<huangzhipeng@huawei.com>> 
* Torin Sandall <<torin@styra.com>>

## Notational Conventions

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" are to be interpreted as described in [RFC 2119](http://tools.ietf.org/html/rfc2119) (Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, March 1997).

The key words "unspecified", "undefined", and "implementation-defined" are to be interpreted as described in the [rationale for the C99 standard](http://www.open-std.org/jtc1/sc22/wg14/www/C99RationaleV5.10.pdf#page=18).

An implementation is not compliant if it fails to satisfy one or more of the MUST, REQUIRED, or SHALL requirements for the protocols it implements.
An implementation is compliant if it satisfies all the MUST, REQUIRED, and SHALL requirements for the protocols it implements.

## Terminology

| Term              | Definition                                       |
|-------------------|--------------------------------------------------|
| Policy            | A set of rules that will be made available inside of a CO-managed container, via the CPI.                             |                                                 |
| Volume            | A unit of storage that will be made available inside of a CO-managed container, via the CSI.                          |                                                 |
| Network           | A unit of network that will be made available inside of a CO-managed container, via the CNI.                          |                                                 |
| Quota             | A limit placed upon the resource usage.                             |                                                 |
| RPC               | [Remote Procedure Call](https://en.wikipedia.org/wiki/Remote_procedure_call).                                         |
| Node              | A host where the user workload will be running, uniquely identifiable from the perspective of a Plugin by a `NodeID`. |
| CO                | Container Orchestration system, communicates with Plugins using CPI service RPCs.                                     |
| Plugin            | Aka “plugin implementation”, a gRPC endpoint that implements the CPI Services.                                        |
| Workload          | The atomic unit of "work" scheduled by a CO. This may be a container or a collection of containers.                   |

## Motivation
At the moment, there are many policy related features implemented by various CO. However we do find the following aspects need significant improvement:

1. There is no reference policy driven architecture which provides a unified view of how policy is enforced on each level by the CO. Furthermore there is also no reference interface that containers implements for the policy enforcement.

2. There are virtually no policy related functionalities regarding the life cycle management of external resources that is used by CO (e.g. storage, network, devices, ...)

3. There also few policy related feature in the standardized interfaces that these external resources are managed through: CRI/CSI/CNI/CDI. Furthermore since for example CNI depends on how the plugin implements the standard, different plugin might cancel out the policy features each implements respectively. 

Therefore we want to propose a standard container interface that could provide a unified policy enforcement mechanism for the CO and its external resources.

## Objective

To define an industry standard “Container Policy Interface” (CPI) that will enable vendors to develop a policy plugin once and have it work across a number of container orchestration (CO) systems.

### Goals in MVP

The Container Policy Interface (CPI) will

* Enable vendors to write one CPI compliant Plugin that “just works” across all COs that implement CPI.
* Define API (RPCs) that enable:
  * Storage related policy enforcement.
* Define plugin protocol RECOMMENDATIONS.
  * Describe a process by which a Supervisor configures a Plugin.
  * Container deployment considerations (`CAP_SYS_ADMIN`, mount namespace, etc.).

### Non-Goals in MVP
TBD

## Solution Overview

This specification defines an interface along with the minimum operational and packaging recommendations for a policy aware resource provider (PARP) to implement a CPI compatible plugin.

The interface declares the RPCs that a plugin must expose: this is the **primary focus** of the CPI specification.
Any operational and packaging recommendations offer additional guidance to promote cross-CO compatibility.

### Architecture

TBD

### Policy Flavors

#### Storage Flavor

* SLA/QoS: service level definitions for virtual pools, physicial pools, etc.
* Storage System Capability Collection: collecting information on pool, disk, RAID, protocoal, thin provisioning, Tiering, QoS, etc.
* Resource Reclaim: automatically relaim storage resources
* Backup/Replication: backup tiering in storage, remote replication
* HA: high availability for volume/image
* Profile: profile with description of user service high level requirement on storage
* Snapshot: periodicall snapshot
* Taskflow: ensure an order of execution for a given storage resource

#### Network Flavor

TBD

## Protocol

### Connectivity

* A CO SHALL communicate with a Plugin using gRPC to access the `Identity`, and (optionally) the `Controller` and `Node` services.
  * proto3 SHOULD be used with gRPC, as per the [official recommendations](http://www.grpc.io/docs/guides/#protocol-buffer-versions).
  * All Plugins SHALL implement the REQUIRED Identity service RPCs.
    Support for OPTIONAL RPCs is reported by the `ControllerGetPolicies` and `NodeGetPolicies` RPC calls.
* The CO SHALL provide the listen-address for the Plugin by way of the `CPI_ENDPOINT` environment variable.
  Plugin components SHALL create, bind, and listen for RPCs on the specified listen address.
  * Only UNIX Domain Sockets may be used as endpoints.
    This will likely change in a future version of this specification to support non-UNIX platforms.
* All supported RPC services MUST be available at the listen address of the Plugin.

### Security

* The CO operator and Plugin Supervisor SHOULD take steps to ensure that any and all communication between the CO and Plugin Service are secured according to best practices.
* Communication between a CO and a Plugin SHALL be transported over UNIX Domain Sockets.
  * gRPC is compatible with UNIX Domain Sockets; it is the responsibility of the CO operator and Plugin Supervisor to properly secure access to the Domain Socket using OS filesystem ACLs and/or other OS-specific security context tooling.
  * SP’s supplying stand-alone Plugin controller appliances, or other remote components that are incompatible with UNIX Domain Sockets must provide a software component that proxies communication between a UNIX Domain Socket and the remote component(s).
    Proxy components transporting communication over IP networks SHALL be responsible for securing communications over such networks.
* Both the CO and Plugin SHOULD avoid accidental leakage of sensitive information (such as redacting such information from log files).

### Debugging

* Debugging and tracing are supported by external, CPI-independent additions and extensions to gRPC APIs, such as [OpenTracing](https://github.com/grpc-ecosystem/grpc-opentracing).

## Configuration and Operation

### General Configuration

* The `CPI_ENDPOINT` environment variable SHALL be supplied to the Plugin by the Plugin Supervisor.
* An operator SHALL configure the CO to connect to the Plugin via the listen address identified by `CPI_ENDPOINT` variable.
* With exception to sensitive data, Plugin configuration SHOULD be specified by environment variables, whenever possible, instead of by command line flags or bind-mounted/injected files.


### Supervised Lifecycle Management

* For Plugins packaged in software form:
  * Plugin Packages SHOULD use a well-documented container image format (e.g., Docker, OCI).
  * The chosen package image format MAY expose configurable Plugin properties as environment variables, unless otherwise indicated in the section below.
    Variables so exposed SHOULD be assigned default values in the image manifest.
  * A Plugin Supervisor MAY programmatically evaluate or otherwise scan a Plugin Package’s image manifest in order to discover configurable environment variables.
  * A Plugin SHALL NOT assume that an operator or Plugin Supervisor will scan an image manifest for environment variables.

#### Environment Variables

* Variables defined by this specification SHALL be identifiable by their `CPI_` name prefix.
* Configuration properties not defined by the CPI specification SHALL NOT use the same `CPI_` name prefix; this prefix is reserved for common configuration properties defined by the CSI specification.
* The Plugin Supervisor SHOULD supply all recommended CPI environment variables to a Plugin.
* The Plugin Supervisor SHALL supply all required CPI environment variables to a Plugin.

##### `CPI_ENDPOINT`

Network endpoint at which a Plugin SHALL host CPI RPC services. The general format is:

    {scheme}://{authority}{endpoint}

The following address types SHALL be supported by Plugins:

    unix://path/to/unix/socket.sock

Note: All UNIX endpoints SHALL end with `.sock`. See [gRPC Name Resolution](https://github.com/grpc/grpc/blob/master/doc/naming.md).  

This variable is REQUIRED.

#### Operational Recommendations

The Plugin Supervisor expects that a Plugin SHALL act as a long-running service vs. an on-demand, CLI-driven process.

Supervised plugins MAY be isolated and/or resource-bounded.

##### Logging

* Plugins SHOULD generate log messages to ONLY standard output and/or standard error.
  * In this case the Plugin Supervisor SHALL assume responsibility for all log lifecycle management.
* Plugin implementations that deviate from the above recommendation SHALL clearly and unambiguously document the following:
  * Logging configuration flags and/or variables, including working sample configurations.
  * Default log destination(s) (where do the logs go if no configuration is specified?)
  * Log lifecycle management ownership and related guidance (size limits, rate limits, rolling, archiving, expunging, etc.) applicable to the logging mechanism embedded within the Plugin.
* Plugins SHOULD NOT write potentially sensitive data to logs (e.g. `Credentials`, `PolicyHandle.Metadata`).

##### Available Services

* Plugin Packages MAY support all or a subset of CPI services; service combinations MAY be configurable at runtime by the Plugin Supervisor.
* Misconfigured plugin software SHOULD fail-fast with an OS-appropriate error code.

##### Linux Capabilities

* Plugin Supervisor SHALL guarantee that plugins will have `CAP_SYS_ADMIN` capability on Linux when running on Nodes.
* Plugins SHOULD clearly document any additionally required capabilities and/or security context.

##### Namespaces

* A Plugin SHOULD NOT assume that it is in the same [Linux namespaces](https://en.wikipedia.org/wiki/Linux_namespaces) as the Plugin Supervisor.
  The CO MUST clearly document the [mount propagation](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt) requirements for Node Plugins and the Plugin Supervisor SHALL satisfy the CO’s requirements.

##### Cgroup Isolation

* A Plugin MAY be constrained by cgroups.
* An operator or Plugin Supervisor MAY configure the devices cgroup subsystem to ensure that a Plugin may access requisite devices.
* A Plugin Supervisor MAY define resource limits for a Plugin.

