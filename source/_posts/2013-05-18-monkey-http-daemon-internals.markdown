---
layout: post
title: "Monkey HTTP Daemon Internals"
date: 2013-05-18 09:21
comments: true
categories: [Monkey, HTTP, Web Server, Source Code]
---

{% img /images/blog/monkey.jpg 'Go! Monkey!' 'Go! Monkey!'%}

Table of Contents
-----------------
* [Introduction](#intro)
* [Basic Data Structures](#data)
* [Brief Workflow](#workflow)
* [Configuration](#config)
* [Scheduler](#sched)
* [Clock](#clock)
* [Hooks](#hooks)
* [HTTP Requests and Responses](#http)
* [API Exposure](#api)
* [Plugins](#plugins)
* [Appendix](#appendix)

Introduction<a id='intro'></a>
------------
[Monkey HTTP Daemon](http://monkey-project.com) is a web server that aims to get
the most out of the Linux platform. It comes with low resource comsumption and 
great scalability which makes it a perfect solution for embedded devices. 

Due to the compact size of Monkey's codebase, it is a proper place for revealing
the internals of a event-driven HTTP web server. Spending some time on analysing
the source code of Monkey will surely worth the cost.

This post is about the internals of Monkey's critical components from the
perspective of source code, you may consider it as a supplement of [this
post](http://edsiper.linuxchile.cl/blog/2013/02/27/architecture-of-a-linux-based-web-server/)
(written by the author of Monkey). You are supposed to read it first to get a
general view of Monkey.

**Notice**: This post is based on Monkey 1.2.0, you may want to get the latest code.
(check out [home page](http://monkey-project.com) for more information). If you
are using an older version, there might be something different, e.g. red-black
tree is introduced since version 1.2.0.

<!-- more -->

Basic Data Structures<a id='data'></a>
---------------------
This section discusses some common data structures used in Monkey. It is relatively
independent and fundamental so I put it as first.

* ###Circular Doubly Linked List

Linked List is heavily used within Monkey's source code. It is used to maintain
a list of given structures.

The definition of List is given by `mk_list.h`:

{% codeblock lang:c %}
struct mk_list
{
    struct mk_list *prev, *next;
};
{% endcodeblock %}

Pretty simple and straight forward. But wait for a minute, where is the payload
of a list node? Aha, the list used in Monkey is like the one used in Linux kernel.
The payload of a list node is not held inside the `mk_list` structure, on the
contrary a list node is embedded in a *host structure* (see Figure 1-1).

{% img /images/blog/linked-list.png 'Circular Doubly Linked List' 'Figure 1-1 Circular Doubly Linked List' %}

Taken `mk_config_entry` structure(from `mk_config.h`) as an example, it contains
a `mk_list` which is used to connect configuration entries together:

{% codeblock lang:c %}
struct mk_config_entry
{
    char *key;
    char *val;
    struct mk_list _head;
};
{% endcodeblock %}

As you may have noticed, there's an extra `mk_list` in Figure 1-1 that acts as a
sentinel node. This convention simplifies list handling by ensuring that the list
always has a *first* and *last* node(even with a empty list). When initializing a
list, Monkey takes a sentinel node and set its `prev` and `next` fields to point
to the node itself:

{% codeblock lang:c %}
static inline void mk_list_init(struct mk_list *list)
{
    list->next = list;
    list->prev = list;
}
{% endcodeblock %}

Besides some common list processing functions, there are some handy macros for
list manipulation(`mk_config.h`):

`mk_list_foreach(curr, head)`: iterate a given list and do something with every
node of the list. `curr` is a temporary `mk_list` pointer used to hold the current
visited node. `head` is the pointer to the head of the given list.

`mk_list_foreach_safe(curr, n, head)`: iterate a given list and do some delete
operation to the node of the list. `curr` is a temporary `mk_list` pointer used to
hold the current visited node. Another `mk_list` temporary pointer `n` is needed to
retrieve the next node of `curr` in advance because `curr` is no longer exists
before the next iteration. `head` is the pointer to the head of the given list.

`mk_list_entry(ptr, type, member)`: get the start address of a *host structure*
of a given list node. `ptr` is a pointer to a given `mk_list` node. `type` is the
type of the *host structure*. `member` is the name of `mk_list` field within the
*host structure*.

Example Usage:
{% codeblock lang:c %}
/*
 * assume the type of host structure is banana
 * struct banana {
 *     ...
 *     mk_list _head;
 * };
 * monkey loves bananas :D
 *
 */
struct banana * b;
struct mk_list *curr, *next;

mk_list_foreach(curr, &store->bananas) {
    b = mk_list_entry(curr, struct banana, _head);
    /* do something with b */
    wash(b);
}

mk_list_foreach_safe(curr, next, &store->bananas) {
    b = mk_list_entry(curr, struct banana, _head);
    /* delete from the list */
    mk_list_del(&b->_head);
    /* do some delete operation */
    eat(b);
}
{% endcodeblock %}

* ###String

String is the most common data structure used in Monkey, and how it is represented
will affect the efficiency and size of Monkey. As we know, the major target of
Monkey is embedded devices so every byte counts.

There are two types of string representation used in Monkey, one is the native
`char *`(a.k.a. null-terminated) and the other one is defined as `mk_pointer`
(`mk_memory.h`):

{% codeblock lang:c %}
typedef struct
{
    char *data;
    unsigned long len;
} mk_pointer;
{% endcodeblock %}

The main difference between them is: the string that referred by one `mk_pointer`
may also be shared by other `mk_pointer`(see Figure 1-2) while a `char *` string
normally have a standalone copy.

By using `mk_pointer` we can save some memory allocation and release by reusing
the same string. This technique can also reduce memory usage and make Monkey more
compact.

{% img /images/blog/pointer.png 'String used by multi mk_pointer' 'Figure 1-2 String used by multi mk_pointer' %}

When we need to make some change to a given string that is shared by some
`mk_pointer`, we may not manipulate in the origin string, instead we duplicate a
separate copy. Otherwise it may change some `mk_pointer` silently which can lead
to unexpected result. `char *mk_pointer_to_buf(mk_pointer p)` is responsible for
this job. It takes a `mk_pointer` as input and return a newly allocated string,
the caller shall free the string after use.

* ###Red-Black Tree

Searching is a common task for almost every program. In Monkey, we will need to
look up the relative information frequently given a socket file descriptor. If
we used list to organize our data, we can only do a linear search because list is
out of order.

In the worst case(the one we are looking for is the last), we need to travel
through the whole list. This is an O(N) operation and as list size increases
this maybe unacceptable for a high performance web server. And this is why
reb-black tree is introduced into Monkey.

If you are unfamiliar with red-black tree, it is time to pick up your undergraduate
data structure textbook and have a quick review. Or alternatively you may refer
to [wikipedia](http://en.wikipedia.org/wiki/Red–black_tree) for introduction.

{% img /images/blog/rbtree.png 'Red-Black Tree' 'Figure 1-3 Red-Black Tree' %}

The definition of red-black tree is given by `mk_rbtree.h`:

{% codeblock lang:c %}
struct rb_node
{
	unsigned long  rb_parent_color;
#define	RB_RED		0
#define	RB_BLACK	1
	struct rb_node *rb_right;
	struct rb_node *rb_left;
} __attribute__((aligned(sizeof(long))));
    /* The alignment might seem pointless, but allegedly CRIS needs it */
{% endcodeblock %}

As with `mk_list`, an `mk_rbtree` node is also embedded in a *host structure*.

Brief Workflow<a id='workflow'></a>
--------------
Before we dive into the nitty-gritty of Monkey, it is helpful to walk through the
main function(`monkey.c`) to get a first impression of Monkey.

{% img /images/blog/bootup.jpg 'Monkey Bootup' 'Figure 2-1 Money Bootup' %}
Main function demonstrates the workflow of Monkey in a high level view:

* As most traditional Unix program, Monkey starts with parsing command line
options, the options are all very self-explanatory. Use `monkey -h` to get full
description of options.

{% codeblock lang:c %}
while ((opt = getopt_long(argc, argv, "DSvhp:w:c:", long_opts, NULL)) != -1) {
    switch (opt) {
    case 'v':
        ...
    }
}
{% endcodeblock %}

* After parsing options Monkey sets up signal handlers for its interesting signals
and leave others as default (`mk_signals.c`).

{% codeblock lang:c %}
mk_signals_init();
{% endcodeblock %}

* Monkey builds a configuration for itself, some entries of the configuration may
be overrided by the previous parsed options. For more details of the building
process please refer to *[Configuration](#config)* section.

{% codeblock lang:c %}
mk_config_start_configure();
{% endcodeblock %}

* Monkey allocates an array `sched_list` of type `sched_list_node`(defined in
`mk_scheduler.h`) for worker management. For example, when choosing a worker to
handle a new user request, Monkey's scheduler uses the information maintained by
`sched_list`.

{% codeblock lang:c %}
mk_sched_init();
{% endcodeblock %}

* Monkey starts to load the plugins enabled by user and read extra plugin
configurations. For more details of plugins please refer to *[Plugins](#plugins)*
section.

{% codeblock lang:c %}
mk_plugin_init();
mk_plugin_read_config();
{% endcodeblock %}

* Monkey creates listening socket and binds it to default port `2001`(if not
overrided).

{% codeblock lang:c %}
/* Server listening socket */
config->server_fd = mk_socket_server(config->serverport, config->listen_addr);
{% endcodeblock %}

* Monkey process will turn into a daemon if it is started in daemon mode.

{% codeblock lang:c %}
/* Running Monkey as daemon */
if (config->is_daemon == MK_TRUE) {
    mk_utils_set_daemon();
}
{% endcodeblock %}

* Monkey then creates a pid file(`logs/monkey.pid.2001` by default) which contains
the process identification number (pid) of itself. Pid file is used for
administrating Monkey later.

{% codeblock lang:c %}
/* Register PID of Monkey */
mk_utils_register_pid();
{% endcodeblock %}

* Monkey sets the timestamp of when it is started and set up two formatted human
readable time strings, one is for logger and the other is for HTTP response headers.
This must be done before any threads are created, otherwise some thread may
execute before the start time recorded by Monkey.

{% codeblock lang:c %}
/* Clock init that must happen before starting threads */
mk_clock_sequential_init();
{% endcodeblock %}

* Once timestamp is set, Monkey can start the clock thread. The clock thread will
run in an infinite loop and update the two time strings mentioned above per second.

{% codeblock lang:c %}
mk_utils_worker_spawn((void *) mk_clock_worker_init, NULL);
{% endcodeblock %}

* Monkey creates some thread specific keys for later use by HTTP workers, such as
`worker_sched_node` for scheduling client connections, etc.

{% codeblock lang:c %}
/* Init thread keys */
mk_thread_keys_init();
{% endcodeblock %}

* If Monkey is started by root user, it will set the uid/gid to the user specified
in configuration.

{% codeblock lang:c %}
/* Change process owner */
mk_user_set_uidgid();
{% endcodeblock %}

* Check if the effective user of Monkey can set the `O_NOATIME` flag for file accessing.
If so, Monkey will request Linux not to update the last access time on the file.
Updating last access time makes hardly any sence as normally a web server just
send the contents of files requested by clients back.  This will improve Monkey's
performance because we can save a lot of time for very often accessed files.

{% codeblock lang:c %}
mk_config_sanity_check();
{% endcodeblock %}

* Monkey invokes process hooks(`PRCTX`) of all the plugins, this is useful for
plugins to set up something before entering the main infinite loop. For example,
a plugin may require Monkey to spawn an independent thread for it.

{% codeblock lang:c %}
/* Invoke Plugin PRCTX hooks */
mk_plugin_core_process();
{% endcodeblock %}

* Monkey spawns HTTP worker threads for HTTP requests handling. The number of
workers is specified in configuration, if the number is 0 Monkey will spawn HTTP
workers depending on the cores of your CPU(one worker per core).

{% codeblock lang:c %}
/* Launch monkey http workers */
mk_server_launch_workers();
{% endcodeblock %}

* Monkey waits for all HTTP worker threads to be ready by checking the `initialized`
field of all worker schedule nodes(`sched_list_node`) every 10 milliseconds. The
mutex lock `mutex_worker_init` is used to synchronize the status of workers, this
ensures the status of workers won't change during the checking process. Once all
threads are marked as ready the mutex lock is no longer necessary, making the
following server loop lock-free.

{% codeblock lang:c %}
/* Wait until all workers report as ready */
while (1) {
    int i, ready = 0;

    pthread_mutex_lock(&mutex_worker_init);
    for (i = 0; i < config->workers; i++) {
        if (sched_list[i].initialized)
            ready++;
    }
    pthread_mutex_unlock(&mutex_worker_init);

    if (ready == config->workers) break;
    /* wait for 10 milliseconds */
    usleep(10000);
}
{% endcodeblock %}

* Monkey enters the main loop of server(`mk_server.c`), starts listening for
incoming sockets and then dispatches them to workers for processing. From now on,
Monkey is good and ready! It will keep serving requests until you tell it to exit.

{% codeblock lang:c %}
mk_server_loop(config->server_fd);
{% endcodeblock %}

Configuration<a id='config'></a>
-------------
Configuration makes Monkey easier to tweak and more flexible, this section will
demonstrate how Monkey deal with it. 

* ###Configuration File Format

Monkey uses a hunman readable file format for configuration, the file is
line-oriented and uses indent level for hierarchy. Figure 3-1 shows the hierarchy
of a configuration file. Within the file there are one or more *sections*, under
every section there are one or more *entries*. Besides these there are comments,
comments are lines that start with sharp mark(`#`).

{% img /images/blog/config.png 'Configuration File Format' 'Figure 3-1 Configuration File Format' %}

A section groups some relative entries for certain subject, and an entry represents
a key-value pair. Section sould be surrounded by brackets(`[]`) and every entry's
key and value are separated by a whitespace. Monkey comes with comprehensive
configuration documentation, you may want to check out `conf/monkey.conf` for more
details about how every entry affect Monkey's behaviour.

* ###File to Memory Mapping

Before Monkey starts serving, it first parse some configuration files to construct
corresponding structures(`mk_config`, defined in `mk_config.h`) that resides in
memory. Once parse is finished, Monkey can speed up configuration information
accessing by reading directly from memory.

For every hierarchy level of a Monkey configuration file, there is a mapping
structure in `mk_config.h`:

{% codeblock lang:c %}
/*
 *          hierarchy level mapping
 *
 * configuration file      ===>    mk_config
 * section                 ===>    mk_config_section
 * entry(key-value pair)   ===>    mk_config_entry
 */
struct mk_config
{
    int created;
    char *file;
    /* list of sections */
    struct mk_list sections;
};
struct mk_config_section
{
    char *name;
    /* list of entries */
    struct mk_list entries;
    struct mk_list _head;
};
struct mk_config_entry
{
    char *key;
    char *val;
    struct mk_list _head;
};
{% endcodeblock %}

A bunch of functions(in `mk_config.c`) are used for handling configuration file:

`mk_config_create`: given the location of a configuration file, allocate a brand
new `mk_config`. For every section found in the file allocate a `mk_config_section`
and for every entry allocate a `mk_config_entry`.  All the `mk_config_section` are
linked together as a linked list, the `sections` field of `mk_config` serves as the
sentinel node of the list and every `_head` field of `mk_config_section` is a
list node. The relation between `entries` field of `mk_config_section` and `_head`
field of `mk_config_entry` are likewise.

`mk_config_section_add`: add a `mk_config_section` to the section list of a
specified `mk_config`. Used by `mk_config_create` to construct section list.

`mk_config_section_get`: given a section name, find the pointer to the corresponding
`mk_config_section`.

`mk_config_entry_add`: add a `mk_config_entry` to the entry list of a specified
`mk_config_section`. Used by `mk_config_create` to construct entry list.

`mk_config_section_getval`: given a `mk_config_section` and a key, find the entry
matching the key under the section. If an entry is found, return its value field,
otherwise return `NULL`. There are several types of entry value: string, number,
boolean and list. We need to specify which type will the value be interpreted,
and Monkey will parse the value string to correct type for us, the type codes is
defined as follows(`mk_config.h`):

{% codeblock lang:c %}
#define MK_CONFIG_VAL_STR 0
#define MK_CONFIG_VAL_NUM 1
#define MK_CONFIG_VAL_BOOL 2
#define MK_CONFIG_VAL_LIST 3
{% endcodeblock %}

Example Usage:

{% codeblock lang:c %}
int port = (int) mk_config_section_getval(section, "Port", MK_CONFIG_VAL_NUM);
{% endcodeblock %}

`mk_config_error`: raise configuration parsing error message and exit Monkey.

`mk_config_free`: free all the allocated resource for a specified `mk_config`.

`mk_config_free_entries`: free all the allocated resource for a specified
`mk_config_section`.

* ###Virtual Hosting Configuration

Monkey supports virtual hosting, which makes it possible for a single server to
host separate host names and share resources among these hosts. For more detailed
introduction please refer to Monkey's [documentation](http://monkey-project.com/documentation/virtual_hosts).

Monkey requires every virtual host to have an independent host configuration file
and all the entries are under a single section `[HOST]`. All the host configuration
files are under `conf/sites/`, by default, there is only a file named `default`.
Every host is represented as a `host` structure within Monkey(`mk_config.h`):

{% codeblock lang:c %}
struct host
{
    char *file;                   /* configuration file location*/
    struct mk_list server_names;  /* host names (a b c...) */
    mk_pointer documentroot;
    char *host_signature;
    mk_pointer header_host_signature;
    struct mk_config *config;     /* source configuration */
    struct mk_list error_pages;   /* custom error pages */
    struct mk_list _head;         /* link node */
};
{% endcodeblock %}

`file` denotes the host file location of current `host`, `server_names` is a list
of host names(every host name is a `host_alias`), `documentroot` is the document
directory path that is treated by current `host` as root directory,
`host_signature` stores the web server name of current `host`, `header_host_signature`
is similar to `host_signature` except that it is formatted as a HTTP header field,
`config` is undoubtedly the memory mapping of the host configuration file,
`error_pages` is a list of pages to be displayed when certain HTTP error is
encountered.

{% codeblock lang:c %}
/* Virtual host name , every virtual host may contain one or more of this */
struct host_alias
{
    char *name;
    unsigned int len;
    struct mk_list _head;
};
/* Custom error page */
struct error_page {
    short int status;       /* HTTP error code */
    char *file;             /* file location relative to documentroot */
    char *real_path;        /* absolute location of file in the file system */
    struct mk_list _head;
};
{% endcodeblock %}

All the host configuration information can be found in `config` field, and why
do we need other fields? There are three reasons for doing so:

1. some fields require further parsing, such as `server_names` is a list built
from the plain text value of entry `Servername`.
2. stores the entry value in a field can make code for retrieving entry value
shorter.
3. some fields can only be evaluated when Monkey is running.

In Monkey we use `mk_config_get_host` to get a `host` from a host configuration
file path.

* ###Server Configuration

Once we gain the necessary knowledge, we can start our journey to Monkey's global
server configuration. The main server configuration is defined in `mk_config.h`:

{% codeblock lang:c %}
extern struct server_config *config;
{% endcodeblock %}

And the `server_config` structure is a bit larger than anyone we have encountered
so far, thus it is not appropriate to display all the fields of `server_config`
here and only those really matter will be listed(a reader clever like you can
easily figure out what the rest fields mean by their names :D):

{% codeblock lang:c %}
struct server_config
{
    int server_fd;                /* server socket file descriptor */
    unsigned int worker_capacity; /* how many clients per thread */
    short int workers;            /* number of worker threads */
    int8_t is_daemon;             /* start Monkey as a daemon process */
    char *serverconf;             /* path to server configuration files */
    char *listen_addr;            /* address to listen, 0.0.0.0(all) by default */
    char *pid_file_path;          /* pid of server */
    int pid_status;               /* whether pid file is created */
    int8_t resume;                /* allow file range(partial contents)? */
    int8_t symlink;               /* allow symbolic link requests? */
    int8_t keep_alive;            /* allow persisten connection? */
    int max_keep_alive_request;   /* max persistent connections to allow */
    int keep_alive_timeout;       /* persistent connection timeout */
    uid_t euid;                   /* effective user id */
    gid_t egid;                   /* effective group id */
    struct mk_list *index_files;  /* different index file names */
    struct mk_list hosts;         /* list of virtuals hosts served by Monkey */
    struct mk_list *plugins;      /* plugins enabled by Monkey */
                                  /* ...(some omitted fields) */
    mode_t open_flags;            /* flags used to open files that required by clients */
    int safe_event_write;         /* safe EPOLLOUT event */
    /* Transport type: HTTP or HTTPS, useful for redirections */
    char *transport;
    /* Define the plugin which provides the transport layer */
    char *transport_layer;
    struct plugin *transport_layer_plugin;
    /* Define the default MIME type when it is not possible to find a proper one */
    char *default_mimetype;
    struct mk_config *config;
};
{% endcodeblock %}

Now let's take a closer look at how server configuration is built by Monkey:

* As we have seen in the [*Brief Workflow*](#workflow) section, Monkey enters server
configuration building process via `mk_config_start_configure`. Before reading
configuration file into memory, Monkey initializes some fields of `server_config`
with default values(by calling `mk_config_set_init_values`).

* Monkey then uses `mk_config_read_files` to map the server configuration file
into memory(`config` field of `server_config`). If an entry is defined in the
file, its default value will be overrided:

{% codeblock lang:c %}
/* in function `mk_config_set_init_values` */
config->keep_alive = MK_TRUE;   /* default value */

/* in function `mk_config_read_files` */
/* overrided by configuration file */
config->keep_alive = (size_t) mk_config_section_getval(section, "KeepAlive",
                                                       MK_CONFIG_VAL_BOOL);
{% endcodeblock %}

* After the main configuration file is parsed, Monkey starts mapping all the
virtual host configuration files into memory. This is achieved by calling
`mk_config_read_hosts`, it will go through all the files under `conf/sites/`(except
`.`, `..` and `default`, `default` has been read before any virtual hosts) and
construct the corresponding `host`(`mk_config_get_host`). All the `host` are linked
together as a list via the `hosts` field of `server_config`.

* Monkey reads the MIME types configuration file(`conf/monkey.mime`) and stores
all the types in an array for later use. For more details of MIME types please
refer to [*HTTP*](#http) section.

Up to this point, the configuration building process of Monkey is finished.

Scheduler<a id='sched'></a>
---------

Recall that Monkey spawns several HTTP workers for client requests processing,
one thing we might concern about is how Monkey will schedule a new incoming
connection. With this question in mind, let's dig into Monkey's scheduler strategy
and implementation.

Within Monkey the scheduling unit is abstracted as structure `sched_connection`
(defined in `mk_scheduler.h`).

{% codeblock lang:c %}
struct sched_connection
{
    int socket;              /* file descriptor of remote socket */
    int status;              /* connection status */
    uint32_t events;         /* epoll events that Monkey is intereted about */
    time_t arrive_time;      /* arrived time(when a connection is established) */
    struct rb_node _rb_head; /* red-black tree head(for quick accessing) */
    struct mk_list _head;    /* list head */
};
{% endcodeblock %}

When a client issues a new connection to the Monkey server, a new socket file
descriptor is associated with it. Then Monkey will schedule this connection based
on a simple load balance strategy: find the HTTP worker that holds the least active
connections and dispatch the new connection to it. Although this is a trivial
strategy, it works fine because mostly HTTP request processing and response
generating won't take a long time. But if there is an HTTP request requires enormous
amount of computation, other requests handled by the same worker will wait for a
long time while other workers may be idle.

In order to schedule connections, Monkey needs to keep track of some extra information
of every HTTP worker. For every HTTP worker, there is a associated `sched_list_node`
(defined in `mk_scheduler.h`) structure for the sake of maintaining such information:

{% codeblock lang:c %}
struct sched_list_node
{
    unsigned long long accepted_connections;
    unsigned long long closed_connections;

    /*
     * Red-Black tree queue to perform fast lookup over
     * the scheduler busy queue
     */
    struct rb_root rb_queue;

    /* Available and busy queue */
    struct mk_list busy_queue;
    struct mk_list av_queue;

    short int idx; /* worker index */
    pthread_t tid; /* worker thread id */
    pid_t pid;     /* worker process id, thread is just light weight process in Linux */
    int epoll_fd;  /* epoll instance for monitoring file descriptors */
    unsigned char initialized; /* whether this worker has been initialized */
};
{% endcodeblock %}

`accepted_connections` records the total number of connections accepted and
hanlded by the corresponding worker, and `closed_connections` records the total
number of connections closed by the corresponding worker. With these two numbers
the active connections of a worker is given by the difference of them:

{% codeblock lang:c %}
/* the active connection number will never be negative */
unsigned long long active_connections = accepted_connections - closed_connections;
{% endcodeblock %}

`rb_queue` refers to the root of a red-black tree that maintains all the active
connections of the corresponding worker according to their file descriptors. The
`_rb_head` field of active `sched_connection` serve as tree nodes and are linked
together as a red-black tree.

`busy_queue` is a linked list of used scheduling connection nodes while `av_queue`
is a linked list of available scheduling connection nodes.

{% codeblock lang:c %}
static inline int _next_target()
{
    int i;
    int target = 0;
    unsigned long long tmp = 0, cur = 0;

    /* calculate the active connections of worker */
    cur = sched_list[0].accepted_connections - sched_list[0].closed_connections;
    if (cur == 0)
        return 0;

    /* Finds the worker with lowest load */
    for (i = 1; i < config->workers; i++) {
        tmp = sched_list[i].accepted_connections - sched_list[i].closed_connections;
        if (tmp < cur) {
            target = i;
            cur = tmp;

            if (cur == 0)
                break;
        }
    }

    /*
     * If sched_list[target] worker is full then the whole server too, because
     * it has the lowest load.
     */
    if (mk_unlikely(cur >= config->worker_capacity)) {
        MK_TRACE("Too many clients: %i", config->worker_capacity * config->workers);
        return -1;
    }

    return target;
}
{% endcodeblock %}

As we can see above, Monkey will go through the `sched_list_node` array in order
to pick up the worker with lowest load and return its index.

When the socket is finally established, a new `sched_connection` structure is
allocated for the socket to maintain some connection related information. All the
scheduling units of a worker are pre-allocated as the maximum connection number
a worker is able to handle is already known. What can we benefit from this strategy?
If we allocate a new `sched_connection` every time a new connection comes and
release the structure when the request reaches its end, it will generate some
overhead. For a high loads Monkey instance, this may result in a degradation of
performance.

When a worker is lauched by Monkey, an array of `sched_connection` structures are
initialized and all the elements of that array are linked to the `av_queue` list
of the worker (function `mk_sched_register_thread` in `mk_scheduler.c`).

{% codeblock lang:c %}
/* Start filling the array */
array = mk_mem_malloc_z(sizeof(struct sched_connection) * config->worker_capacity);
for (i = 0; i < config->worker_capacity; i++) {
    sched_conn = &array[i];
    sched_conn->status = MK_SCHEDULER_CONN_AVAILABLE;
    sched_conn->socket = -1;
    sched_conn->arrive_time = 0;

    mk_list_add(&sched_conn->_head, &sl->av_queue);
}
{% endcodeblock %}

If a new `sched_connection` is required, the worker will remove one from the
`av_queue` and add it to the `busy_queue` (function `mk_sched_register_client` in
`mk_scheduler.c`).  If all the available nodes are used, the new connection will
be dropped by Monkey.

{% codeblock lang:c %}
sched_conn = mk_list_entry_first(av_queue, struct sched_connection, _head);
...
/* Move to busy queue */
mk_list_del(&sched_conn->_head);
mk_list_add(&sched_conn->_head, &sched->busy_queue);
{% endcodeblock %}

Figure 4-1 shows a possible layout of the array and how the scheduling connection
nodes are linked together by two queues.

{% img /images/blog/scheduling_connection_list.jpg 'Scheduling Connection List' 'Figure 4-1 Scheduling Connection List'%}

* ###Epoll

As we have mentioned above, Monkey is an event-driven web server and the I/O event
notification facility used by it is `epoll` (Linux-specific).

For every worker thread spawned by Monkey, there is an epoll instance for monitoring
socket file descriptors that are dispatched it. When a new connection is accepted
by Monkey, the socket fd is first dispatched to the worker with lowest load and
added to its epoll instance (see `mk_sched_add_client` function in `mk_scheduler.c`
for more information).

The main loop of every worker then waits for events that occurs on the socket fds that
it is responsible to monitor, and processes the events properly by using the event
handler of the sockets. According to the event types, different routines of the
event handler will be called. When read event is fired on the socket, the read
routine of event handler will be called, and when it is a write event, the
corresponding write routine will be called, etc. (Of course there can be one than
one events occur on a single socket fd simultaneously)

Source code never lies (see `mk_epoll_init` in `mk_epoll.c`):

{% codeblock lang:c %}
/* worker thread main loop */
while (1) {
    ret = -1;
    /*
     * wait for upcoming events.
     * notice: the timeout of epoll_wait is defined as MK_EPOLL_WAIT_TIMEOUT,
     * currently the value is 3000 milliseconds, that is 3 seconds, if not any
     * events are fired after 3 seconds after the last check, the worker will
     * quit waiting and check if it needs to close some sockets due to their
     * timeouts.
     */
    num_fds = epoll_wait(efd, events, max_events, MK_EPOLL_WAIT_TIMEOUT);

    for (i = 0; i < num_fds; i++) {
        fd = events[i].data.fd;

        /* call the corresponding handler routine */
        if (events[i].events & EPOLLIN) {
            MK_TRACE("[FD %i] EPoll Event READ", fd);
            ret = (*handler->read) (fd);
        }
        else if (events[i].events & EPOLLOUT) {
            MK_TRACE("[FD %i] EPoll Event WRITE", fd);
            ret = (*handler->write) (fd);
        }
        else if (events[i].events & (EPOLLHUP | EPOLLERR | EPOLLRDHUP)) {
            MK_TRACE("[FD %i] EPoll Event EPOLLHUP/EPOLLER", fd);
            ret = (*handler->error) (fd);
        }

        if (ret < 0) {
            MK_TRACE("[FD %i] Epoll Event FORCE CLOSE | ret = %i", fd, ret);
            (*handler->close) (fd);
        }
    }

    /* Check timeouts and update next one */
    if (log_current_utime >= fds_timeout) {
        mk_sched_check_timeouts(sched);
        fds_timeout = log_current_utime + config->timeout;
    }
}
{% endcodeblock %}

Sometimes a socket may need to be put into sleep because it needs to wait for some
resources, that is, epoll will not monitor any events for this socket, and the socket
will be waked up when the resources are ready and restored to its state before sleeping.
For this reason, a worker thread needs to maintain some extra information for every
socket that is registered to the its epoll instance. The structure is defined as
`epoll_state` (`mk_epoll.h`):

{% codeblock lang:c %}
/*
 * An epoll_state represents the state of the descriptor from
 * a Monkey core point of view.
 */
struct epoll_state
{
    int          fd;            /* File descriptor                    */
    uint8_t      mode;          /* Operation mode                     */
    uint32_t     events;        /* Events mask                        */
    unsigned int behavior;      /* Triggered behavior                 */

    struct rb_node _rb_head;
    struct mk_list _head;
};
{% endcodeblock %}

All the `epoll_state` are linked together as a reb-black tree for quick access
and modification. For the same sake of reducing overhead, the `epoll_state`
nodes are pre-allocated and maintained by two queues: an available queue and a
busy queue, it is similar to the ones of `sched_connection`, `epoll_state_index`
(`mk_epoll.h`) plays the role of managing these nodes:

{% codeblock lang:c %}
struct epoll_state_index
{
    int size;

    struct rb_root rb_queue;
    struct mk_list busy_queue;
    struct mk_list av_queue;
};
{% endcodeblock %}

After all the requests of a socket are successfully processed or some errors
occur on that socket, the socket will be deleted from the epoll instance and the
corresponding `epoll_state` will be removed from the `epoll_state_index` tree it
belongs to.

Clock<a id='clock'></a>
-----

Clock is important for a web server as both log and HTTP header rely on the correctness
of it. Monkey spawns an extra thread for timestamp maintaining and updating. The
clock thread maintains two buffers of formatted strings: one is used by the logger
thread of Monkey and the other one is used by HTTP worker threads to set header
fields for sending back to clients.

{% img /images/blog/clock.jpg 'Clock' 'Figure 5-1 Clock'%}

If you explore `mk_clock.h`, you can easily find out two `mk_pointer`s that refer
to the log and HTTP header formatted strings, which are very self-explanatory:

{% codeblock lang:c %}
extern mk_pointer log_current_time;     /* used by logger */
extern mk_pointer header_current_time;  /* used by HTTP workers */
{% endcodeblock %}

The main loop of clock worker thread is pretty simple: it just update both buffers
every second:

{% codeblock lang:c %}
while (1) {
    cur_time = time(NULL);

    if(cur_time != ((time_t)-1)) {
        mk_clock_log_set_time(cur_time);
        mk_clock_header_set_time(cur_time);
    }

    /* update every second */
    sleep(1);
}
{% endcodeblock %}

Everything just seems fine, but there's a pitfall: what if a thread (either a logger
thread or an HTTP worker thread) access the buffers when they are being updated?
We may get the improper formatted string results!

So, what shall we do? One idea is to use mutex lock, lock the buffers before they
are updated and release the lock after the update is finished. As Monkey emphasizes
on efficiency, we want to avoid using locks whenever it is possible. Luckily we
do get a better solution for it by using a simple trick: use double buffers for
updating. Taken `log_current_time` as an example, Monkey allocates a string array
of size 2 and uses one element for logger thread accessing and updates the other
array element with new formatted string. In the end of the update, `log_current_time`
will refer to the newly updated array element:

{% codeblock lang:c %}
/* define double buffers */
static char *log_time_buffers[2];

/* The mk_pointers have two buffers for avoid in half-way access from
 * another thread while a buffer is being modified. The function below returns
 * one of two buffers to work with.
 */
static inline char *_next_buffer(mk_pointer *pointer, char **buffers)
{
    /* get the array element for updating */
    if(pointer->data == buffers[0]) {
        return buffers[1];
    } else {
        return buffers[0];
    }
}

/* update the string buffer used by logger thread */
static void mk_clock_log_set_time(time_t utime)
{
    char *time_string;
    struct tm result;

    time_string = _next_buffer(&log_current_time, log_time_buffers);
    log_current_utime = utime;

    /* update the array element that is not currently in use */
    strftime(time_string, LOG_TIME_BUFFER_SIZE, "[%d/%b/%G %T %z]",
             localtime_r(&utime, &result));

    /* update the pointer to the newer formatted string */
    log_current_time.data = time_string;
}
{% endcodeblock %}

Hooks<a id='hooks'></a>
-----

HTTP Requests and Responses<a id='http'></a>
---------------------------
* ###HTTP Header
* ###Header Cache
* ###MIME Type
* ###HTTP Method
* ###Connection Management

API Exposure<a id='api'></a>
------------

Plugins<a id='plugins'></a>
-------
* ###Logger
* ###Cheetah
* ###Liana

Appendix<a id='appendix'></a>
--------
* Monkey source code structure:

{% codeblock %}
$ tree
.
|-- include
|   |-- mk_env.h         [contains environment variables, generated by configure file]
|   |-- mk_http_status.h [defines HTTP status codes]
|   |-- mk_info.h        [contains released information]
|   |-- mk_limits.h      [defines upper limit of server]
|   |-- mk_list.h        [contains linked list implementation]
|   |-- mk_macros.h      [contains some handy helper macros]
|   |-- MKPlugin.h       [code that related to Monkey API exposure]
|   `-- *.h              [omitted header files that have corresponding c files below]
|-- mk_cache.c           [allows Monkey to cache HTTP headers]
|-- mk_clock.c           [code for Monkey's inner clock]
|-- mk_config.c          [code related to configuration management]
|-- mk_connection.c      [code related to HTTP connections management]
|-- mk_epoll.c           [integrates epoll for fd monitoring and event management]
|-- mk_file.c            [code for retrieving file information]
|-- mk_header.c          [HTTP header management]
|-- mk_http.c            [code related to HTTP lifecycle]
|-- mk_iov.c             [code for scatter/gather I/O]
|-- mk_lib.c             [code related to shared library support]
|-- mk_memory.c          [memory management]
|-- mk_method.c          [HTTP method parse helper]
|-- mk_mimetype.c        [HTTP mimetype management]
|-- mk_plugin.c          [code that powers Monkey's plugin mechanism]
|-- mk_rbtree.c          [contains red-black tree implementation]
|-- mk_request.c         [in charge of HTTP requests and responses handling]
|-- mk_scheduler.c       [scheduler for load balancing]
|-- mk_server.c          [code related to server establishment]
|-- mk_signals.c         [code for signals handling]
|-- mk_socket.c          [socket manipulation utilities]
|-- mk_string.c          [string manipulation utilities]
|-- mk_user.c            [code for uid/pid convertion]
|-- mk_utils.c           [contains some handy helper]
`-- monkey.c             [main entry point of Monkey]
{% endcodeblock %}

* Useful utilities for analysing:

Here are some tools I found very helpful while analysing source code, this is for
your reference only and please consult man pages for full infomatioin. (If you
use other cool tools, please let me know):

### [curl](http://en.wikipedia.org/wiki/CURL)
`curl` is a command line tool for transferring data with URL syntax using various
protocols. As with Monkey, I use `curl` as an HTTP client to communicate with the
Monkey server.

{% codeblock %}
/* retrieve the main page from Monkey server */
curl localhost:2001
/* same as above but with verbose information */
curl -v localhost:2001
{% endcodeblock %}

With this command I can fire a HTTP request without opening my Firefox/Chromium,
it is also much light weight than a comprehensive browser.

### [netcat](http://en.wikipedia.org/wiki/Netcat)
`netcat` is a computer networking service for reading from and writing to network
connections using TCP or UDP. It is often referred as the swiss army knife for
TCP/IP.

{% codeblock %}
/* test timeout */
date; netcat -C 127.0.0.1 2001; date
/* manually send an HTTP request */
netcat -C 127.0.0.1 2001
GET / HTTP/1.1
Host: localhost:2001

{% endcodeblock %}

Note: option `-C` make `netcat` to use CRLF for EOL sequence, the last blank line
of HTTP request header is required since that means the end of the request.

With this command I can explore a request to Monkey in a lower level than `curl`,
it helps to understand the TCP connection of Monkey.

### [lsof](http://en.wikipedia.org/wiki/Lsof)
`lsof` stands for list open files. As we know, one of the most important abstraction
of Linux is files: regular file, pipe, fifo, device, etc. For this reason, `lsof`
becomes a very neat and invaluable tools, it's the swiss army knife for examining
open files.

{% codeblock %}
/* list all the files open by Monkey */
lsof -c monkey
{% endcodeblock %}

With this command I can be informed which plugins(saved as dynamic library) are
loaded by Monkey and where they reside in the file system, how many pipes are
opened for logging, which socket is used for waiting new incoming connections,
etc.

### [tcpdump](http://en.wikipedia.org/wiki/Tcpdump)
`tcpdump` is a common packet analyzer that allows the user to intercept and display
TCP/IP and other packets being transmitted or received over a network to which
the computer is attached. 

{% codeblock %}
/* for our interest, we want to capture packets related to Monkey */
sudo tcpdump -i lo port 2001
{% endcodeblock %}

With this command I can figure out how every single byte is exchange between Monkey
server and the client.

Note: I am testing Monkey in local mode and thus the interface option is specified
as `lo`, you should change it to the appropriate interface(`eth0`, etc.) if the
client and server aren't in the same computer.

### [htop](http://en.wikipedia.org/wiki/Htop)
`htop` is an interactive system-monitor process-viewer that aims to replace `top`.
Once you are used to `htop`, you will never want to go back to `top`.

{% codeblock %}
htop
{% endcodeblock %}

Within `htop`, press `F4` then enter keyword `monkey` to filt out threads that
belong to Monkey. On my Linux box, the cunstom thread names are not displayed
by default. If this is the same case for you, you can change this behaviour by
the following steps:

* press `F2` to enter the setup panel of `htop`
* navigate to `Display options` section
* toggle options `Show custom thread names` on
* press `F10` and you are done

With this command I can explore the internal structure of Monkey, check out how
many resources are consumed by Monkey, send signals to Monkey:

* press `l` to display output of `lsof` within `htop`
* press `s` to display output of `strace` within `htop`
* press `F5` or `t` to display the process and thread hierarchy of Monkey
* press `F6` or `>` to sort by time, CPU usage, etc.
