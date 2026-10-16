# File Descriptor

Because the table is per-process, `fd` 3 in one process and `fd` 3 in another are completely unrelated. The number only has meaning relative to the process holding it.

## Everything is a file descriptor

Unix generalizes the concept far past actual files. An `fd` can refer to a regular file, a directory, a pipe, a socket, a terminal, a block or character device, an `eventfd`, a `timerfd`, an `epoll` instance, a `signalfd`, or a shared-memory object. The payoff is that a huge amount of code becomes polymorphic for free: `read()` and `write()` work on all of them.

In Unix-like systems, a network connection is treated as a file. When a backend server receives an incoming HTTP request, or when an application opens a connection to a database, the operating system assigns a file descriptor to that TCP socket.

If a server is handling 10,000 concurrent user requests, it requires 10,000 open file descriptors just for the network sockets.

Infrastructure heavily relies on file descriptors to route traffic efficiently:

- **Database poolers:** Tools like PgBouncer maintain a pool of active database connections. Every client connecting to the pooler uses one file descriptor, and every connection from the pooler to the actual PostgreSQL database uses another.
- **Load balancers:** Reverse proxies like HAProxy or Nginx have to hold open a connection from the client and simultaneously open a connection to the backend server.

If these tools are not configured with a high enough file descriptor limit, they physically cannot accept new connections, resulting in dropped traffic and 502/503 errors, even if CPU and memory are sitting at 10%.

![File descriptor, open file description and inode](image-12.png)

## Descriptor, open file description, inode

The **open file description** (sometimes called the open file table entry) is system-wide and represents one act of opening. This is where the file offset lives — the position where the next read or write will happen. It also holds the access mode (`O_RDONLY`/`O_WRONLY`/`O_RDWR`) and the status flags (`O_APPEND`, `O_NONBLOCK`, `O_SYNC`).

The **inode** is the file itself: size, permissions, owner, timestamps, and the map of disk blocks. One inode, many possible open file descriptions.

- Two separate `open()` calls on the same file create two independent open file descriptions pointing at one inode. Two independent offsets. Reading through one doesn't move the other.
- `dup()` / `dup2()` create a new `fd` number in the same process pointing at the same open file description. Shared offset, shared status flags. Writing through `fd` 3 advances the position seen by `fd` 4.
- `fork()` copies the fd table, so the child's fd 3 points at the parent's open file description. Shared offset across two processes. This is why a parent and child both writing to an inherited stdout don't clobber each other's position (though their userspace buffers still can).

Every syscall that moves data takes an fd: `read`, `write`, `send`, `recv`, `mmap`, `ioctl`, `fstat`, `sendfile`, `splice`.

```text
  PROCESS A                          SYSTEM-WIDE                    ON-DISK / DEVICE
  ┌─────────────────┐
  │ files_struct     │                ┌──────────────────┐
  │  fd_array[]     │                │  struct file      │           ┌──────────────┐
  │   [0] ──────────┼───────────────►│  f_pos  = 4096   │──────────►│ struct inode │
  │   [1] ──────────┼──────┐         │  f_flags= O_RDWR  │      ┌───►│  i_size      │
  │   [2] ──────────┼───┐  │         │  f_ref  = 2      │      │    │  i_mode      │
  │   [3] ──────────┼─┐ │  │         │  f_op   ────────►│      │    │  i_op        │
  └─────────────────┘ │ │  └────────►└──────────────────┘      │    └──────────────┘
                      │ │                                      │           ▲
                      │ │            ┌──────────────────┐      │           │
  PROCESS B           │ └───────────►│  struct file      │──────┘           │
  ┌─────────────────┐ │              │  f_pos  = 0      │                  │
  │  fd_array[]     │ │              │  f_ref  = 1      │                  │
  │   [3] ──────────┼─┘              └──────────────────┘                  │
  └─────────────────┘                                                      │
                                     ┌──────────────────┐                  │
                                     │  struct file      │──────────────────┘
                                     │  f_pos  = 512    │
                                     └──────────────────┘
```

| Level | Kernel type | Scope | Holds |
| --- | --- | --- | --- |
| 1. File descriptor | `int` (array index) | Per-process | Nothing — it's just a number |
| 2. Open file description | `struct file` | Shared, refcounted | Offset, flags, `f_op`, credentials |
| 3. File | `struct inode` | One per file on disk | Size, permissions, blocks |

### Open file table

- Global table in the kernel
- Contains:
  - current file position
  - reference count of processes pointing to it
  - pointer to an entry in the v-node table

## Every high-performance server is an event loop over fds

`select`, `poll`, `epoll`, `kqueue`, and `io_uring` all exist to answer one question: which of these thousands of fds is ready? Nginx, Redis, Node's libuv, Go's netpoller, and every async runtime you've used are built on this. The reason it's possible to wait on sockets, timers, signals, and child-process exits in one loop is that Linux made all of them fds (`timerfd`, `signalfd`, `pidfd`).

This is why `epoll` exists. A proxy holding 100,000 connections holds 100,000 fds and must answer "which of these is readable right now?" thousands of times a second. `select()` can't — it's O(n) per call and capped by `FD_SETSIZE`. `epoll` (Linux), `kqueue` (BSD), and `io_uring` are all fd-registration systems: you hand the kernel a set of fds once and get back only the ready ones.

`connect()` returns an fd. Getting that fd cost you a TCP handshake (1 RTT), probably a TLS handshake (1–2 more RTTs), and often an application-level login — Postgres auth, HTTP/2 settings exchange. Closing the fd throws all of that away. A pool is a list of fds kept in the table so the next request skips the setup.

Threads are the same problem without the duplication. Threads share one fd table, so a socket fd is process-global state. Two threads writing to the same fd interleave their bytes just as badly.

Every proxied connection costs two fds. The proxy `accept()`s a client socket and `connect()`s an upstream socket, then shuttles bytes between them. Nginx documents this explicitly — `worker_connections` must be roughly half of `worker_rlimit_nofile` for a reverse proxy, because each connection consumes a descriptor on both sides. If you configure `worker_connections` 10240 with a default 1024 fd limit, you get "worker_connections exceed open file resource limit" and dropped connections under load.

Zero-copy is an fd-level API. `splice()` moves bytes between two fds inside the kernel without ever copying into the proxy's userspace memory. HAProxy uses it for exactly this reason. You can only express that operation because both endpoints are descriptors.

### Passing fds: zero-downtime reload

`SCM_RIGHTS` lets a process hand a live socket fd to another process over a Unix domain socket. This is the mechanism behind:

- `systemd` socket activation — `systemd` opens port 80, hands the listener fd to your service, so nothing is refused during startup.
- Nginx binary upgrade and Envoy hot restart — the new binary receives the listening socket fd, so there's no window where the port is unbound and no connection is dropped.
- Privilege separation — a root helper binds port 443 and hands the fd to an unprivileged worker that could never have bound it itself.

## Postgres session state and pooling

When you connect to Postgres, the server forks a dedicated backend process for you. Anything you'd call "session state" — prepared statements, `SET` variables, temp tables, `LISTEN` registrations, advisory locks, cursors — lives in that backend process's memory, keyed to that specific connection.

It doesn't live in your application. It doesn't live in the pooler. It lives on one side of one socket, in one OS process on the database server. That's the whole thing.

In Postgres, the postmaster process accepts client connections and forks a child process. The child process inherits the file descriptors of its parent.

**Session mode:** when a client connects to PgBouncer, it's handed a server connection and keeps it until the client disconnects. One client ↔ one backend, exclusively, for the client's whole lifetime. Everything behaves exactly like talking to Postgres directly — but you get almost no multiplexing benefit, because an idle client still ties up a backend.

**Transaction mode:** the client connection stays open, but PgBouncer only binds it to a server connection from `BEGIN` until `COMMIT`/`ROLLBACK`. The instant the transaction ends, that server connection is released back into the pool and can be handed to a completely different client. Your next transaction gets whichever backend happens to be free — very likely not the one you had before.

That's why prepared statements, session variables, `LISTEN/NOTIFY`, and advisory locks break under it — those are state living on the server fd, and you don't get the same one back.

What breaks in transaction mode:

- **Session advisory locks** — `pg_advisory_lock()` is held by the backend that took it. When the transaction ends, that backend goes to someone else while still holding your lock. You may never be routed back to it, so `pg_advisory_unlock()` can't reach it. The lock leaks until that backend disconnects. `pg_advisory_xact_lock()` works, because it releases at commit.
- **`LISTEN / NOTIFY`** — `LISTEN` registers the channel on a backend. That backend gets handed to another client, and notifications go to a process that isn't yours. Meanwhile your next transaction is on a backend that isn't listening. This doesn't work in transaction mode at all; clients that need it have to use session mode.
- **Temp tables** — created in the backend's private temp schema. Your next transaction is on a different backend and can't see the table, while the original one carries an orphaned table around. The workaround is `CREATE TEMP TABLE ... ON COMMIT DROP` so nothing outlives the binding.

When you call `connection.close()` on a pooled connection, you're not closing anything. HikariCP hands you a `ProxyConnection`; its `close()` returns the entry to the pool. The TCP socket stays open, the fd stays in your table, and the Postgres backend process on the other end has no idea anything happened. From Postgres's point of view there was never a break — it's one continuous session that has now been executing statements for 30 minutes on behalf of thousands of unrelated HTTP requests.

## Client and server are roles

It's a role in one connection. Whoever called `connect()` is the client; whoever called `accept()` is the server. Your app process is a server to the browser and a client to PgBouncer. PgBouncer is a server to your app and a client to Postgres.

The client is your database driver — the library code inside your app process (HikariCP, pgx, asyncpg, psycopg) that called `connect()` on a socket to PgBouncer. PgBouncer uses the word in exactly this narrow way. `SHOW CLIENTS` lists the sockets it accepted; `SHOW SERVERS` lists the sockets it opened. Two lists, same box, different sides.

At the fd level, the roles evaporate after setup. Once the handshake completes, both ends hold an ordinary socket fd and both can read and write freely.

## Inside the kernel

A file descriptor is an index into the file descriptor table. Put differently, a file `fd` is a pointer to an array of pointers to `struct file`.

```c
struct file __rcu **fd;      /* current fd array */
```

The standard descriptors:

```c
#define	STDIN_FILENO	0	/* standard input file descriptor */
#define	STDOUT_FILENO	1	/* standard output file descriptor */
#define	STDERR_FILENO	2	/* standard error file descriptor */
```

### `struct fdtable`

```c
struct fdtable {
	unsigned int max_fds;
	struct file __rcu **fd;      /* current fd array */
	unsigned long *close_on_exec;
	unsigned long *open_fds;
	unsigned long *full_fds_bits;
	struct rcu_head rcu;
};
```

- `fd` is the actual array. `fd[3]` is a `struct file *`, or `NULL` if descriptor 3 is closed.
- `open_fds` is a bitmap of which slots are in use — one bit per fd.
- `full_fds_bits` is a second-level bitmap: one bit per 64 fds, set when that entire word of `open_fds` is full.

### `struct files_struct`

Small processes never allocate any of this. `files_struct` embeds `fd_array[NR_OPEN_DEFAULT]` (64 entries, `BITS_PER_LONG`) directly inside itself, so the common case is zero extra allocations.

```c
/*
 * Open file table structure
 */
struct files_struct {
  /*
   * read mostly part
   */
	atomic_t count;
	bool resize_in_progress;
	wait_queue_head_t resize_wait;

	struct fdtable __rcu *fdt;
	struct fdtable fdtab;
  /*
   * written part on a separate cache line in SMP
   */
	spinlock_t file_lock ____cacheline_aligned_in_smp;
	unsigned int next_fd;
	unsigned long close_on_exec_init[1];
	unsigned long open_fds_init[1];
	unsigned long full_fds_bits_init[1];
	struct file __rcu * fd_array[NR_OPEN_DEFAULT];
};
```

### `struct file`

One open file. One per `open()` call.

```c
struct file {
	fmode_t                      f_mode;      /* FMODE_READ / FMODE_WRITE */
	const struct file_operations *f_op;       /* the vtable */
	unsigned int                 f_flags;     /* O_APPEND, O_NONBLOCK, ... */
	const struct cred           *f_cred;      /* who opened it */
	loff_t                       f_pos;       /* THE SEEK OFFSET */
	void                        *private_data;
	file_ref_t                   f_ref;       /* reference count */
};
```

- `f_pos` lives here, not in the fd. So the seek offset is shared by every descriptor pointing at this `struct file`, and independent between separate `open()` calls of the same path. This single fact explains most fd surprises.
- `f_op` is a vtable, and this is what makes "everything is a file" work. A `struct file` for a regular ext4 file, a TCP socket, a pipe, an `eventfd`, an `epoll` instance, a `/dev/null` character device, and a `timerfd` are all the same type. They differ only in which `file_operations` they point at, and what they stash in `private_data`.

### Why an index instead of just handing userspace the pointer?

Two reasons matter here:

- **Security.** A kernel pointer must never reach userspace. Leaking one defeats KASLR and hands an attacker a target address. And a pointer from userspace can't be trusted at all — you'd be letting a program name an arbitrary kernel address to read from.
- **Ownership.** Because the array is per-process, the index is scoped. My `fd` 3 and your `fd` 3 are unrelated, and the kernel gets that isolation for free from the indirection. A global pointer would have no owner.

### What "index" means in memory

A pointer is 8 bytes on x86-64, so the array is a run of 8-byte boxes. Say it starts at address `0x1000`:

```text
                     ADDRESS      CONTENTS OF THE BOX
                   ┌──────────┬──────────────────────┐
  fdt->fd ───────► │  0x1000  │  0xffff8881_0a3c0000 │  slot 0  (stdin)
                   ├──────────┼──────────────────────┤
                   │  0x1008  │  0xffff8881_0b120000 │  slot 1  (stdout)
                   ├──────────┼──────────────────────┤
                   │  0x1010  │  0xffff8881_0b120000 │  slot 2  (stderr — same as 1!)
                   ├──────────┼──────────────────────┤
     fd 3 ───────► │  0x1018  │  0xffff8881_0c7d8000 │  slot 3  ◄── your socket
                   ├──────────┼──────────────────────┤
                   │  0x1020  │  NULL                │  slot 4  (closed)
                   └──────────┴──────────────────────┘
                                        │
                                        │ this number IS an address
                                        ▼
                              0xffff8881_0c7d8000
                            ┌──────────────────────┐
                            │  struct file          │
                            │    f_pos  = 4096     │
                            │    f_op   = socket   │
                            │    f_ref  = 1        │
                            └──────────────────────┘
```

The integer 3 does not appear anywhere in memory. It's used once, arithmetically, to pick a box:

```text
   address of slot 3  =  0x1000  +  3 × 8  =  0x1018
                         ^^^^^^     ^   ^
                       base addr   fd   sizeof(pointer)
```

That's the whole meaning of "index." The CPU computes `base + fd × 8`, loads the 8 bytes there, and now it has a pointer to a `struct file`.

### Why two stars?

`**` is a pointer to a pointer. Each `*` is one level of "address of something."

```text
struct file __rcu **fd;
            ^^^^^^^  ^^
              │      └── fd is...
              │
   ...a pointer to a pointer to a struct file
```

The array's elements are themselves pointers. In C, an array decays to a pointer to its first element. So:

| If the array holds… | …a pointer to element 0 has type |
| --- | --- |
| `int` | `int *` |
| `struct file` | `struct file *` |
| `struct file *` ← our case | `struct file **` |

The `fd` array holds `struct file *` values.
