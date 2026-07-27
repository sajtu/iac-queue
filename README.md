# iac-queue
`iac-queue` is a small, provider-neutral, directory-backed task queue originally developed for IaC pipeline and workflow.

It stores opaque plain-text task messages and moves tasks through:

```text
backlog -> inprogress -> completed
```

Cancelled backlog or in-progress tasks move to `completed` with the result
`cancelled`.

The program knows nothing about Jenkins, Terraform, Ansible, Puppet, Proxmox,
VMware, or other providers. Those systems are clients of the queue.

## Keys

Every task belongs to one caller-defined key, for example(s):

```text
iac-deploy-stage
build-my-app
team-b-patching
```

The key is required for every operation, including listings. There is no
unfiltered list command.

Keys create logical namespaces. They are filters, not authentication
credentials. Operating-system permissions still control access to the
flat-file database.

External-ID uniqueness is scoped to a key. Therefore, two different keys may
both contain external ID `118`.

## Install

Run the installer as the account that will operate the queue:
```bash
sudo -u <username> ./iac-queue install /path/to/install/dir
```

To install inside an existing IaC repository:
```bash
[sudo -u <username>] ./iac-queue install /path/to/your/iac/repo/dir
```
Example:
```bash
sudo -u iac-tf ./iac-queue install /opt/iac
```

With the /opt/iac installed path example, this creates:

```text
/opt/iac/iac-queue/
├── bin/iac-queue
├── logs/
└── queue/
    ├── backlog/
    ├── inprogress/
    └── completed/
```

If the installation is inside a Git worktree, the installer adds only
`iac-queue/bin/` and `iac-queue/logs/` to `.gitignore`. Queue records remain
visible to Git.

## Usage Example: Add new queue record:

To add a "queue record" to the queue, a origin provider can call `iac-queue` with `add` sub-command to do so:
```bash
iac-queue add <key> <external_ID> [<optional_message_for_message_queue>]
```
Required arguments:
* <key> is required external key provided by the provider. It can be any plaintext word. No spaces.
* <external_ID> is a required external queue ID provided by the provider. This is any alphanumerical combination for ID.

Optional arguments:
* <optional_message_for_message_queue> is completely optional. This adds this initial message to the queue message queue payload system.  The optional text supplied to `add` becomes the first queued message.

For example, a Jenkins pipeline could run a script that wants to add an item to the queue for a specific pipeline for Terraform and Ansible.
perhaps after it orchestrated Terraform to apply and add new VMs to Proxmox (or VMWare), it needs to queue up a stager to configure or stage each newly provisioned system for Ansible to setup.

Jenkins pipeline script could call `iac-queue`:

```bash
/opt/iac/iac-queue/bin/iac-queue add iac-deploy-stage 118
```
NOTE: the example opts out of including any message to the message queue system.

The `<key>` here is `iac-deploy-stage` and the `<external_ID>` is `118`.  `118` happens to also represent VMID of the newly provisioned VM.

Add the initial payload from a file:

```bash
/opt/iac/iac-queue/bin/iac-queue add iac-deploy-stage 118 --file payload.txt
````

## Usage Example: Advance queue record to next queue:

There are THREE queues:
* backlog
* inprogress
* complete

When new queue records are added, they are automatically added to the `backlog` queue.

External providers and consumers are responsible for managing their pipeline's queue records.

For example, if Jenkins pipeline script adds new queue records and kicks off Ansible, Ansible consumer process can advance the queue record item to the next queue with the appropriate subcommands:
```bash
/path/to/iac-queue start <key> <external_ID>
/path/to/iac-queue complete <key> <external_ID>
/path/to/iac-queue cancel <key> <external_ID>
```
* `start <key> <external_ID>`: advance queue record in backlog to `inprogress`.
* `complete <key> <external_ID>`: advance queue record in backlog to `completed` (success).
* `cancel <key> <external_ID>`: advance queue record in backlog to `completed` (cancelled).

Both `<key>` and `<external_ID>` arguments must be provided.

For example:
```bash
/opt/iac/iac-queue/bin/iac-queue start iac-deploy-stage 118
/opt/iac/iac-queue/bin/iac-queue complete iac-deploy-stage 118
/opt/iac/iac-queue/bin/iac-queue cancel iac-deploy-stage 118
```
NOTE: `iac-queue` does not handle or manage the queue records. The external providers and consumers are responsible for managing the queue and cleaning up.

## Usage Example: Display list of queue records in a specific queue:

* `backlog <key> [--long]`: list all queue records in `backlog` tied to <key>.
* `inprogress <key> [--long]`: list all queue records in `inprogress` tied to <key>.
* `completed <key> [--long]`: list all queue records in `completed` tied to <key>.
NOTE: `<key>` must be provided. There is no "list all" records. `<key>` exist to create boundaries so no pipeline process can view another's set of queued records.

Example:
```bash
/opt/iac/iac-queue/bin/iac-queue backlog iac-deploy-stage
/opt/iac/iac-queue/bin/iac-queue inprogress iac-deploy-stage
/opt/iac/iac-queue/bin/iac-queue completed iac-deploy-stage
/opt/iac/iac-queue/bin/iac-queue backlog iac-deploy-stage --long
/opt/iac/iac-queue/bin/iac-queue inprogress iac-deploy-stage --long
/opt/iac/iac-queue/bin/iac-queue completed iac-deploy-stage --long
```

## Message queue subsystem

Each non-completed task has an optional FIFO message queue. You can optionally push a new message to the new record's message queue with the add subcommand. (see `Usage Example: Add new queue record`)

Messages must be ASCII plaintext. Providers are responsible for encrypting and ASCII-encoding sensitive data before pushing it. ASCII-armored or Base64
ciphertext can therefore be carried as an opaque payload.

Moving a task to completed or cancelled removes all of its live message payload files. A process cannot push, peek, or shift messages afterward.

Important: deleting a live payload does not remove copies from Git history. Do not commit plaintext secrets. Encrypt sensitive payloads before adding
them, or keep transient queue data outside Git.

If you want to use the message queue syster after the new queue record was added to the backlog, use these subcommands:
```text
push     Add a message to the end
peek     Read the oldest message without removing it
shift    Read and remove the oldest message
```

NOTE: One one message can be read at a time, to read the next message in FIFO order, you must remove the oldest message first.

FIFO (First In, First Out), if you add five message in oder: 'A', 'B', 'C', 'D', and 'E', where 'A' is added first, and 'E' is last, then 'A' is the oldest message.  Visually the queue looks like: 'E', 'D', 'C', 'B', 'A'

If you use `push` to add another message, 'F', then the queue becomes: 'F', 'E', 'D', 'C', 'B', 'A'.

You can read the current/oldest message in the queue with `peek`.  You cannot read the entire queue, only one message at the time in FIFO order. So in this example, if `peek` is used, only 'A' will be returned.

In order to read the next message in the message queue, 'A' must be discarded. Use `shift` to read and discard the oldest/current message in the queue. When 'A' is discarded the queue becomes: 'F', 'E', 'D', 'C', 'B'.  Now when running `peek`, 'B' returns.

NOTE: When queue record is completed (and is in completed queue), all messages in its queue record (if any) is flushed and deleted. You cannot read any messages from completed queue record's message queue.

Default limits per task:

| Limit | Default |
|---|---:|
| Messages | 10 |
| Bytes per message | 8192 |


## Message queue subsystem: Usage: push: Add message to message queue
```bash
iac-queue push <key> <external_ID> "<message>"
```
Example, push 'A', 'B', 'C', 'D', and 'E', one at a time to the record, key=iac-deploy-stage id=118:
```bash
/opt/iac/iac-queue/bin/iac-queue push iac-deploy-stage 118 "A"
/opt/iac/iac-queue/bin/iac-queue push iac-deploy-stage 118 "B"
/opt/iac/iac-queue/bin/iac-queue push iac-deploy-stage 118 "C"
/opt/iac/iac-queue/bin/iac-queue push iac-deploy-stage 118 "D"
/opt/iac/iac-queue/bin/iac-queue push iac-deploy-stage 118 "E"
```

## Message queue subsystem: Usage: peek: Read current message from message queue
```bash
iac-queue peek <key> <external_ID>
```
Example, read current message from key=iac-deploy-stage id=118's message queue:
```bash
/opt/iac/iac-queue/bin/iac-queue peek iac-deploy-stage 118
```
This should return, 'A' each time until it's discarded.

Alternative methods to add a message to message queue:
* Read message from txt file: `iac-queue peek <key> <external_ID> --file /path/to/message.txt`
* Read message from standard-in: `echo  "message" | iac-queue peek <key> <external_ID> --stdin`

NOTE: Completed queue records have empty message queues. All messages are deleted when records move to `completed` queue. `peek` return exit code `2` when the task's message queue is empty.

## Message queue subsystem: Usage: shift: discard (and read) current message from message queue
```bash
iac-queue shift <key> <external_ID>
```
Example, discard current message from key=iac-deploy-stage id=118's message queue:
```bash
/opt/iac/iac-queue/bin/iac-queue shift iac-deploy-stage 118
```
This should discard, 'A', and also display/return 'A'. Next time you run peek or discard, 'B' should be read and/or affected.

NOTE: Completed queue records have empty message queues. All messages are deleted when records move to `completed` queue. `shift` return exit code `2` when the task's message queue is empty.




Display one record:

```bash
iac-queue show iac-deploy-stage 118
```

## IDs

The caller chooses an arbitrary, nonempty, single-line external ID.
`iac-queue` does not interpret it.

`iac-queue` assigns a separate internal ID such as `Q000000001`. Queue
directories use that internal ID. Client commands continue to use the key and
external ID.

Use `--long` on a listing to see the internal ID, key, and external ID.
