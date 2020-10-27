# Firecracker Snapshotting

## What is microVM snapshotting?

MicroVM snapshotting is a mechanism through which a running microVM and its
resources can be serialized and saved to an external medium in the form of a
`snapshot`. This snapshot can be later used to restore a microVM with its
guest workload at that particular point in time.

## Snapshotting in Firecracker

### Supported platforms

The Firecracker snapshot feature is in [developer preview](docs/RELEASE_POLICY.md) 
on all CPU micro-architectures listed in [README](../README.md#supported-platforms) 
except ARM which is not supported.

### Overview
A Firecracker microVM snapshot can be used for loading it later in a different
Firecracker process, and the original guest workload is being simply resumed.

The original guest which the snapshot is created from, should see no side
effects from this process (other than the latency introduced by the snapshot
creation process).

Both network and vsock packet loss can be expected on guests that are resumed
from snapshots in another Firecracker process.
It is also not guaranteed that the state of the network connections survives
the process.

In order to make restoring possible, Firecracker snapshots save the full state
of the following resources:
- the guest memory,
- the emulated HW state (both KVM and Firecracker emulated HW).

The state of the components listed above is generated independently, which brings
flexibility to our snapshotting support. This means that taking a snapshot results
in multiple files that are composing the full microVM snapshot:
- the guest memory file,
- the microVM state file,
- zero or more disk files (depending on how many the guest had; these are  
**managed by the users**).

The design allows sharing of memory pages and read only disks between multiple
microVMs. When loading a snapshot, instead of loading at resume time the full
contents from file to memory, Firecracker creates a
[MAP_PRIVATE mapping](http://man7.org/linux/man-pages/man2/mmap.2.html) of the
memory file, resulting in runtime on-demand loading of memory pages. Any subsequent
memory writes go to a copy-on-write anonymous memory mapping.
This has the advantage of very fast snapshot loading times, but comes with the cost
of having to keep the guest memory file around for the entire lifetime of the
resumed microVM.

## Performance

The Firecracker snapshot create/resume performance depends on the memory size,
vCPU count and emulated devices count. The Firecracker CI runs snapshots tests 
on AWS **m5d.metal** instances and the baseline for snapshot resume latency 
target is under **8ms** with 5ms p90 for a microvm with this specs: 
2vCPU/512MB/1 block/1 net device.

## Known issues and limitations

- High snapshot latency on 5.4+ host kernels - 
[#2129](https://github.com/firecracker-microvm/firecracker/issues/2129)
- Guest network connectivity is not guaranteed to be preserved after resume
- Restoring microVMs with vsock devices doesn't work.
- Poor entropy and replayable randomness when resuming multiple microvms which 
deal with cryptographic secrets. Please see [Snapshot security and uniqueness](#snapshot-security-and-uniqueness)

## Firecracker Snapshotting characteristics

- Fresh Firecracker microVMs are booted using `anonymous` memory, while microVMs
  resumed from snapshot load memory on-demand from the snapshot and copy-on-write
  to anonymous memory.
- Resuming from a snapshot is optimized for speed, while taking a snapshot involves
  some extra CPU cycles for synchronously writing dirty memory pages to the memory
  snapshot file. Taking a snapshot of a fresh microVM, on which dirty pages tracking
  is not enabled, results in the full contents of guest memory being written to the
  snapshot.
- The _memory file_ and _microVM state file_ are generated by Firecracker on snapshot
  creation. The disk contents are _not_ explicitely flushed to their backing files.
- The API calls exposing the snapshotting functionality have clear **Prerequisites**
  that describe the requirements on when/how they should be used.
 
## Snapshot versioning

Firecracker snapshotting implementation offers support for microVM versioning
(`cross-version snapshots`) in the following contexts:
- saving snapshots at older versions (being able to create a snapshot with any version
in the `[N, N + o]` interval, while being in Firecracker version `N+o`),
- loading snapshots from older versions (being able to load a snapshot created by any
Firecracker version in the `[N, N + o]` interval, in a Firecracker version `N+o`).

The design supports an unlimited number of versions, the value of `o` (maximum number
of older versions that we can restore from / save a snapshot to, from the current
version) will be defined later.

## Snapshot API

Firecracker exposes the following APIs for manipulating snapshots: `Pause`, `Resume`
and `CreateSnapshot` can be called only after booting the microVM, while `LoadSnapshot`
is allowed only before boot.

### Pausing the microVM

To create a snapshot, first you have to pause the running microVM and its vCPUs with
the following API command:

```bash
curl --unix-socket /tmp/firecracker.socket -i \
    -X PATCH 'http://localhost/vm' \
    -H 'Accept: application/json' \
    -H 'Content-Type: application/json' \
    -d '{
            "state": "Paused"
    }'
```

**Prerequisites**: The microVM is booted.
                   Successive calls of this request keep the microVM in the `Paused`
                   state.
**Effects**:
- _on success_: microVM is guaranteed to be `Paused`.
- _on failure_: no side-effects.

## Creating snapshots

Now that the microVM is paused, you can create a snapshot. Full snapshots always
create a complete, resumeable snapshot of the current microVM state and memory.
`Diff` snapshots functionality is not yet supported.

### Creating full snapshots

For creating a full snapshot, you can use the following API command:

```bash
curl --unix-socket /tmp/firecracker.socket -i \
    -X PUT 'http://localhost/snapshot/create' \
    -H  'Accept: application/json' \
    -H  'Content-Type: application/json' \
    -d '{
            "snapshot_type": "Full",
            "snapshot_path": "./snapshot_file",
            "mem_file_path": "./mem_file",
            "version": "0.23.0"
    }'
```

Details about the required and optional fields can be found in the
[swagger definition](../../src/api_server/swagger/firecracker.yaml).
*Note*: If the files indicated by `snapshot_path` and `mem_file_path` don't exist at
        the specified paths, then they will be created right before generating the
        snapshot.

**Prerequisites**: The microVM is `Paused`.
**Effects**:
- _on success_:
  - The file indicated by `snapshot_path` (e.g. `/path/to/snapshot_file`) contains the
    devices' model state and emulation state. The one indicated by `mem_file_path`
    (e.g. `/path/to/mem_file`) contains a full copy of the guest memory.
  - The generated snapshot files are immediately available to be used (current process
    releases ownership). At this point, the block devices backing files should be
    backed up externally by the user.
    Please note that block device contents are only guaranteed to be committed/flushed
    to the host FS, but not necessarily to the underlying persistent storage
    (could still live in host FS cache).
  - If diff snapshots were enabled, the snapshot creation resets then the dirtied page
    bitmap and marks all pages clean (from a diff snapshot point of view).

If a `version` is specified, the new snapshot is saved at that version, otherwise
it will be saved at the same version of the running Firecracker. The version is only
used for the microVM state file as it contains internal state structures for device
emulation, vCPUs and others that can change their format from a Firecracker version
to another. Versioning is not required for the block and memory files. The separate
block device file components of the snapshot have to be handled by the user.

- _on failure_: no side-effects.

### Creating diff snapshots

Not yet supported.

### Resuming the microVM

You can resume the microVM by sending the following API command:

```bash
curl --unix-socket /tmp/firecracker.socket -i \
    -X PATCH 'http://localhost/vm' \
    -H 'Accept: application/json' \
    -H 'Content-Type: application/json' \
    -d '{
            "state": "Resumed"
    }'
```

**Prerequisites**: The microVM is `Paused`.
                   Successive calls of this request are ignored (microVM remains
                   in the running state).
**Effects**:
- _on success_: microVM is guaranteed to be `Resumed`.
- _on failure_: no side-effects.

## Loading snapshots

If you want to load a snapshot, you can do that only **before** the microVM is configured
(the only resources that can be configured prior are the Logger and the Metrics systems)
by sending the following API command:

```bash
curl --unix-socket /tmp/firecracker.socket -i \
    -X PUT 'http://localhost/snapshot/load' \
    -H  'Accept: application/json' \
    -H  'Content-Type: application/json' \
    -d '{
            "snapshot_path": "./snapshot_file",
            "mem_file_path": "./mem_file",
            "enable_diff_snapshots": true
    }'
```

Details about the required and optional fields can be found in the
[swagger definition](../../src/api_server/swagger/firecracker.yaml).

**Prerequisites**: A full memory snapshot and a microVM state file **must** be provided.
                   The disk backing files, network interfaces backing TAPs and/or vsock
                   backing socket that were used for the original microVM's configuration
                   should be set up and accessible to the new Firecracker process (in
                   which the microVM is resumed). These host-resources need to be
                   accessible at the same relative paths to the new Firecracker process
                   as they were to the original one.
**Effects:**
- _on success_:
  - The complete microVM state is loaded from snapshot into the current Firecracker
    process.
  - It then resets the dirtied page bitmap and marks all pages clean (from a diff
    snapshot point of view).
  - The loaded microVM is now in the `Paused` state, so it needs to be resumed for it
    to run.
  - The memory file pointed by `mem_file_path` **must** be considered immutable from
    Firecracker and host point of view. It backs the guest OS memory for read access
    through the page cache. External modification to this file corrupts the guest
    memory and leads to undefined behavior.
  - The file indicated by `snapshot_path`, that is used to load from, is released and no
    longer used by this process.
  - If `enable_diff_snapshots` is set, then diff snapshots can be taken afterwards.
- _on failure_: A specific error is reported and then the current Firecracker process
                is ended (as it might be in an invalid state).

*Notes*:
`enable_diff_snapshots` on the snapshot load API is currently just an equivalent for
`track_dirty_pages` from the vm config API. Actual diff snapshots are not yet supported.
Keeping the name snapshot-related so that it will not be an API breaking change when
we add the diff snapshots support.

With these in mind, some possible snapshotting scenarios are the following:
- `Boot from a fresh microVM` -> `Pause` -> `Create snapshot` -> `Resume` -> `Pause` ->
  `Create snapshot` -> ... ;
- `Boot from a fresh microVM` -> `Pause` -> `Create snapshot` -> `Resume` -> `Pause` ->
  `Resume` -> ... -> `Pause` -> `Create snapshot` -> ... ;
- `Load snapshot` -> `Resume` -> `Pause` -> `Create snapshot` -> `Resume` -> `Pause` ->
  `Create snapshot` -> ... ;
- `Load snapshot` -> `Resume` -> `Pause` -> `Create snapshot` -> `Resume` -> `Pause` ->
  `Resume` -> ... -> `Pause` -> `Create snapshot` -> ... ;

### Timekeeping and snapshots

It is also worth knowing, a microVM that is restored from snapshot will be resumed with
the guest OS wall-clock continuing from the moment of the snapshot creation. For this
reason, the wall-clock should be updated to the current time, on the guest-side.
More details on how you could do this can be found at a
[related FAQ](../../FAQ.md#my-guest-wall-clock-is-drifting-how-can-i-fix-it).

### Provisioning host disk space for snapshots

Depending on VM memory size, snapshots can consume a lot of disk space. Firecracker 
integrators **must** ensure that the provisioned disk space is sufficient for normal
operation of their service as well as during failure scenarios. If the service exposes
the snapshot triggers to customers, integrators **must** enforce proper disk quotas to 
avoid any DoS threats that would cause the service to fail or function abnormally.

### Snapshot security and uniqueness

When snapshots are used in a such a manner that a given guest's state is resumed
from more than once, guest information assumed to be unique may in fact not be;
this information can include identifiers, random numbers and random number
seeds, the guest OS entropy pool, as well as cryptographic tokens. Without a
strong mechanism that enables users to guarantee that unique things stay unique
across snapshot restores, we consider resuming execution from the same state
more than once insecure.

#### Usage examples

##### Example 1: secure usage (currently in dev preview)

```
Boot microVM A -> ... -> Create snapshot S -> Terminate
                                           -> Load S in microVM B -> Resume -> ...
```

Here, microVM A terminates after creating the snapshot without ever resuming
work, and a single microVM B resumes execution from snapshot S. In this case,
unique identifiers, random numbers, and cryptographic tokens that are meant to
be used once are indeed only used once. In this example, we consider microVM B
secure.

##### Example 2: potentially insecure usage

```
Boot microVM A -> ... -> Create snapshot S -> Resume -> ...
                                           -> Load S in microVM B -> Resume -> ...
```

Here, both microVM A and B do work staring from the state stored in snapshot S.
Unique identifiers, random numbers, and cryptographic tokens that are meant to
be used once may be used twice. It doesn't matter if microVM A is terminated
before microVM B resumes execution from snapshot S or not. In this example, we
consider both microVMs insecure as soon as microVM A resumes execution.

##### Example 3: potentially insecure usage
```
Boot microVM A -> ... -> Create snapshot S -> ...
                                           -> Load S in microVM B -> Resume -> ...
                                           -> Load S in microVM C -> Resume -> ...
                                           [...]
```

Here, both microVM B and C do work starting from the state stored in snapshot S.
Unique identifiers, random numbers, and cryptographic tokens that are meant to
be used once may be used twice. It doesn't matter at which points in time
microVMs B and C resume execution, or if microVM A terminates or not after the
snapshot is created. In this example, we consider microVMs B and C insecure, and
we also consider microVM A insecure if it resumes execution.

#### Reusing snapshotted states securely

We are currently working to add a functionality that will notify guest operating
systems of the snapshot event in order to enable secure reuse of snapshotted
microVM states, guest operating systems, language runtimes, and cryptographic
libraries. In some cases, user applications will need to handle the snapshot
create/restore events in such a way that the uniqueness and randomness
properties are preserved and guaranteed before resuming the workload.

We've started a discussion on how the Linux operating system might securely
handle being snapshotted [here](https://lkml.org/lkml/2020/10/16/629).

## Known Issues

### Vsock must be inactive during snapshot

Vsock device can break if snapshotted while having active connections.
Firecracker snapshots do not capture any inflight network or vsock (through the
linux unix domain socket backend) traffic that has left or not yet entered
Firecracker.

The above, coupled with the fact that Vsock control protocol is not resilient
to vsock packet loss leads to Vsock device breakage when doing a snapshot while
there are active Vsock connections.

#### Workaround

Close all active Vsock connections prior to snapshotting the VM.