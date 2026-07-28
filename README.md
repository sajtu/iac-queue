# iac-queue
`iac-queue` is a small, provider-neutral, directory-backed task queue originally developed for IaC pipelines and workflows.

It stores and moves tasks (also known as queue records) through:

```text
backlog -> inprogress -> completed
```

Cancelled backlog or in-progress tasks (or queue records) move to `completed` with the result
`cancelled`.

Abandoned backlog or in-progress tasks that reach the maximum age move to the
read-only `abort` queue with the result `aborted`.

The program knows nothing about Jenkins, Terraform, Ansible, Puppet, Proxmox,
VMware, or other providers. Those systems are clients of the queue.

## Terms:

* Producers: these processes/users provide (or produces) queue record(s) (AKA task(s)) and queue record data to the iac-queue.
* Consumers: these processes/users reads (or consumes)  queue record data data.

For example:  Jenkins --> iac-queue --> Ansible  
              Producer                  Consumer  

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

Commands shown with `<key>` can instead obtain the key from a protected file
or the `IAC_QUEUE_KEY` environment variable. When either method is used, omit
`<key>` from its normal command position.

Use a protected key file:

```bash
iac-queue --key-file /path/to/queue.key backlog
iac-queue --key-file /path/to/queue.key show 118
```

Use an environment variable:

```bash
IAC_QUEUE_KEY="your-key" iac-queue backlog
IAC_QUEUE_KEY="your-key" iac-queue show 118
```

If both are supplied, `--key-file` takes precedence. A key file should be
readable only by the account running `iac-queue`.

Keys must contain printable ASCII without spaces or other whitespace.

External IDs must start with a letter or number. They may contain letters,
numbers, periods (`.`), underscores (`_`), plus signs (`+`), and hyphens
(`-`). Spaces are not allowed.

External-ID uniqueness is scoped to a key. Therefore, two different keys may
both contain external ID `118`.

**DO NOT FORGET OR LOSE YOUR KEY(S)!!!* You will not be able to manage, list, or show anything from you queue without your keys. There is no recovery system if you lost or forget your key(s)!

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

With the `/opt/iac` installed path example, this creates:

```text
/opt/iac/iac-queue/
├── bin/iac-queue
├── logs/
└── queue/
    ├── backlog/
    ├── inprogress/
    ├── completed/
    ├── abort/
    ├── trash/
    └── namespaces/
```

If the installation is inside a Git worktree, the installer adds only
`iac-queue/bin/`, `iac-queue/logs/`, and `iac-queue/queue/trash/` to
`.gitignore`. Active, completed, and aborted queue records remain visible to
Git. Records moved to trash are not tracked.

For multiple operating-system users, place those users in the same group and
make the installation owned by that group. Runtime directories use set-group-ID
and group-write permissions so new queue files inherit the shared group.

## Concurrent access

`iac-queue` supports multiple concurrent processes:

* Reads of the same record use shared locks and may run concurrently.
* Queue listings do not take a queue-wide write lock.
* Changes to the same key and external ID are serialized with a record lock.
* Changes to different records may run concurrently.
* Internal queue-ID allocation uses a separate, short-lived counter lock.
* Expiration, cleanup, and trash purging use a maintenance lock.

On Linux, locks use `flock`. A portable exclusive lock-directory fallback is
used when `flock` is unavailable; on that fallback, same-record reads are
serialized. Lock files are runtime data under `logs/` and are not tracked by
Git.

After upgrading an existing installation, run the installer again so existing
runtime and queue paths receive the shared-group permissions and the trash
ignore rule:

```bash
sudo -u <username> ./iac-queue install /path/to/install/dir
```

## Usage Example: Add new queue record:

To add a "queue record" to the queue, an origin provider can call `iac-queue` with the `add` subcommand:
```bash
iac-queue add <key> <external_ID> [<optional_message_for_message_queue>]
```
Required arguments:
* `<key>` is the required opaque key provided by the provider. It must use the supported characters. No spaces are allowed.
* `<external_ID>` is a required external queue ID provided by the provider. It can contain any supported alphanumeric identifier combination. No spaces are allowed.

Optional arguments:
* `<optional_message_for_message_queue>` is completely optional. It adds an initial message to the record's message queue payload system. The optional text supplied to `add` becomes the first queued message.

For example, a Jenkins pipeline could run a script that wants to add an item to the queue for a specific Terraform and Ansible pipeline.
Perhaps after it orchestrates Terraform to apply and add new VMs to Proxmox (or VMware), it needs to queue up a stager to configure each newly provisioned system for Ansible to set up.

The Jenkins pipeline script could call `iac-queue`:

```bash
/opt/iac/iac-queue/bin/iac-queue add iac-deploy-stage 118
```
NOTE: This example opts out of including any message in the message queue system.

The `<key>` here is `iac-deploy-stage` and the `<external_ID>` is `118`. `118` happens to also represent the VMID of the newly provisioned VM.

Add the initial payload from a file:

```bash
/opt/iac/iac-queue/bin/iac-queue add iac-deploy-stage 118 --file payload.txt
```

## Usage Example: Advance queue record to next queue:

There are FOUR queues:
* backlog
* inprogress
* completed
* abort

When new queue records are added, they are automatically added to the `backlog` queue.

The `abort` queue is managed automatically by `iac-queue`. Users and providers
cannot move records into it directly.

External providers and consumers are responsible for deciding when to advance their pipeline's queue records.

For example, if a Jenkins pipeline script adds new queue records and starts Ansible, the Ansible consumer process can advance each queue record to the next queue with the appropriate subcommands:
```bash
/path/to/iac-queue start <key> <external_ID>
/path/to/iac-queue complete <key> <external_ID>
/path/to/iac-queue cancel <key> <external_ID>
```
* `start <key> <external_ID>`: advance a queue record from `backlog` to `inprogress`.
* `complete <key> <external_ID>`: advance a queue record from `inprogress` to `completed` (success).
* `cancel <key> <external_ID>`: advance a queue record from `backlog` or `inprogress` to `completed` (cancelled).

The key must be supplied using one of the supported key-input methods.
`<external_ID>` must also be provided.

For example:
```bash
/opt/iac/iac-queue/bin/iac-queue start iac-deploy-stage 118
/opt/iac/iac-queue/bin/iac-queue complete iac-deploy-stage 118
/opt/iac/iac-queue/bin/iac-queue cancel iac-deploy-stage 118
```
NOTE: `iac-queue` performs requested queue transitions, but it does not decide when a record should be started, completed, or cancelled. External providers and consumers make those decisions.

## Usage Example: Display list of queue records in a specific queue:

* `backlog <key> [--long]`: list all queue records in `backlog` tied to `<key>`.
* `inprogress <key> [--long]`: list all queue records in `inprogress` tied to `<key>`.
* `completed <key> [--long]`: list all queue records in `completed` tied to `<key>`.
* `abort <key> [--long]`: list all automatically aborted queue records tied to `<key>`.

NOTE: `<key>` must be provided. There is no "list all" records command. `<key>` exists to create logical boundaries so a pipeline process can request only its own set of queued records. Keys are filters, not security credentials.

Example:
```bash
/opt/iac/iac-queue/bin/iac-queue backlog iac-deploy-stage
/opt/iac/iac-queue/bin/iac-queue inprogress iac-deploy-stage
/opt/iac/iac-queue/bin/iac-queue completed iac-deploy-stage
/opt/iac/iac-queue/bin/iac-queue abort iac-deploy-stage
/opt/iac/iac-queue/bin/iac-queue backlog iac-deploy-stage --long
/opt/iac/iac-queue/bin/iac-queue inprogress iac-deploy-stage --long
/opt/iac/iac-queue/bin/iac-queue completed iac-deploy-stage --long
/opt/iac/iac-queue/bin/iac-queue abort iac-deploy-stage --long
```

Long listings display the optional label instead of the submitted key.

## Optional queue group label

A key group may have an optional plaintext label:

```bash
iac-queue label <key> "Terraform VM staging"
```

Display the current label:

```bash
iac-queue label <key>
```

Remove the label:

```bash
iac-queue label <key> --clear
```

The label is descriptive only. It cannot replace the key when accessing queue
records. Labels are optional and should not be used when the group should
remain unnamed.

## Automatic abort queue

Each new queue record stores:
* An ISO-8601 creation timestamp.
* An immutable Unix epoch creation time representing the record's birth.

On every operational invocation, `iac-queue` checks the `backlog` and
`inprogress` queues for abandoned records.

An active queue record that is 366 days old is automatically:
* Removed from its active queue.
* Marked with the result `aborted`.
* Moved to the read-only `abort` queue.
* Stripped of all live message queue payloads.
* Retained with its metadata and history for auditing.

Records in `completed` or `abort` are terminal and are not automatically moved
again.

## Optional terminal record cleanup

Completed and aborted records are retained forever unless cleanup is
explicitly requested.

Move terminal records that have been completed or aborted for at least 365
days into the untracked `trash` directory:

```bash
iac-queue cleanup 365
```

`DAYS` is required and may be zero or a positive integer. Using zero moves all
completed and aborted records to trash.

Records in trash are excluded from queue listings, task lookup, expiration
checks, and normal queue operations. Moving tracked records to trash appears
to Git as deletion of those records.

Permanently delete every record currently in trash:

```bash
iac-queue purge-trash --yes
```

`purge-trash` requires `--yes` because deletion is permanent. If `cleanup` and
`purge-trash` are never run, terminal records continue to be retained.

## Message queue subsystem

Each non-completed and non-aborted task has an optional FIFO message queue. You can optionally push a new message to the new record's message queue with the `add` subcommand. (See `Usage Example: Add new queue record`.)

Messages must be ASCII plaintext. Providers are responsible for encrypting and ASCII-encoding sensitive data before pushing it. ASCII-armored or Base64
ciphertext can therefore be carried as an opaque payload.

Moving a task to completed, cancelled, or aborted removes all of its live message payload files. A process cannot push, peek, or shift messages afterward.

Important: deleting a live payload does not remove copies from Git history. Do not commit plaintext secrets. Encrypt sensitive payloads before adding
them, or keep transient queue data outside Git.

If you want to use the message queue system after the new queue record is added to the backlog, use these subcommands:
```text
push     Add a message to the end
peek     Read the oldest message without removing it
shift    Read and remove the oldest message
```

NOTE: Only one message can be read at a time. To read the next message in FIFO order, you must remove the oldest message first.

FIFO means First In, First Out. If you add five messages in order: `A`, `B`, `C`, `D`, and `E`, where `A` is added first and `E` is last, then `A` is the oldest message. Visually, the queue looks like: `E`, `D`, `C`, `B`, `A`.

If you use `push` to add another message, `F`, then the queue becomes: `F`, `E`, `D`, `C`, `B`, `A`.

You can read the current/oldest message in the queue with `peek`. You cannot read the entire queue, only one message at a time in FIFO order. In this example, if `peek` is used, only `A` will be returned.

In order to read the next message in the message queue, `A` must be discarded. Use `shift` to read and discard the oldest/current message in the queue. When `A` is discarded, the queue becomes: `F`, `E`, `D`, `C`, `B`. Now when running `peek`, `B` returns.

NOTE: When a queue record is completed, cancelled, or aborted, all messages in its queue record (if any) are flushed and deleted. You cannot read messages from a terminal queue record's message queue.

Default limits per task:

| Limit | Default |
|---|---:|
| Messages | 10 |
| Bytes per message | 8192 |


## Message queue subsystem: Usage: push: Add message to message queue
```bash
iac-queue push <key> <external_ID> "<message>"
```
Example, push `A`, `B`, `C`, `D`, and `E`, one at a time to the record, key=`iac-deploy-stage` id=`118`:
```bash
/opt/iac/iac-queue/bin/iac-queue push iac-deploy-stage 118 "A"
/opt/iac/iac-queue/bin/iac-queue push iac-deploy-stage 118 "B"
/opt/iac/iac-queue/bin/iac-queue push iac-deploy-stage 118 "C"
/opt/iac/iac-queue/bin/iac-queue push iac-deploy-stage 118 "D"
/opt/iac/iac-queue/bin/iac-queue push iac-deploy-stage 118 "E"
```

Alternative methods to add a message to the message queue:
* Read message from a text file: `iac-queue push <key> <external_ID> --file /path/to/message.txt`
* Read message from standard input: `echo  "message" | iac-queue push <key> <external_ID> --stdin`

## Message queue subsystem: Usage: peek: Read current message from message queue
```bash
iac-queue peek <key> <external_ID>
```
Example, read the current message from key=`iac-deploy-stage` id=`118`'s message queue:
```bash
/opt/iac/iac-queue/bin/iac-queue peek iac-deploy-stage 118
```
This should return `A` each time until it is discarded.

NOTE: `peek` returns exit code `2` when an open task's message queue is empty. Message operations against completed or aborted records return exit code `1` because their message queues are closed.

## Message queue subsystem: Usage: shift: Discard (and read) current message from message queue
```bash
iac-queue shift <key> <external_ID>
```
Example, discard the current message from key=`iac-deploy-stage` id=`118`'s message queue:
```bash
/opt/iac/iac-queue/bin/iac-queue shift iac-deploy-stage 118
```
This should discard `A` and also display/return `A`. The next time you run `peek` or `shift`, `B` should be read and/or affected.

NOTE: `shift` returns exit code `2` when an open task's message queue is empty. Message operations against completed or aborted records return exit code `1` because their message queues are closed.




Display one record:

```bash
iac-queue show iac-deploy-stage 118
```

## IDs

The caller chooses the key and external ID. Both must be nonempty, single-line
identifiers without spaces and must use the supported characters described in
the `Keys` section.

`iac-queue` assigns a separate internal ID such as `Q000000001`. Queue
directories use that internal ID. Client commands continue to use the key and
external ID.

The submitted key is not returned by listings or `show`.

Use `--long` on a listing to see the internal ID, optional label, and external
ID.

