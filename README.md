yarn-logs-helpers
=================
Scripts for parsing / making sense of yarn logs.

# Contents
#### `yarn-container-logs`
The main script of note here is [`yarn-container-logs`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/yarn-container-logs):

```
$ yarn-container-logs 0018
```

*  It can take a full application ID (e.g. `application_1416279928169_0018`) or **just the last 4 digits of one** (`0018`).
*  It downloads the YARN logs for that application into a local directory (defaulting to the application ID, but can be overriden with an optional second argument, after the app ID) and splits them into per-container files:

    ```
    # Directory created by yarn-container-logs
    $ cd application_1416279928169_0018

    # Directory with per-container logs
    $ cd containers

    # Per-container log files have prefix /container_/
    $ ls container_*
    container_1416279928169_0018_01_000015
    container_1416279928169_0018_01_000016
    container_1416279928169_0018_01_000017
    ...

    # The files contain exactly what was pulled down from YARN.
    $ head container_1416279928169_0018_01_000015
    Container: container_1416279928169_0018_01_000015 on my-node-11-10.rest.of.domain.name_port
    ===================================================================================================
    LogType: stderr
    LogLength: 700
    Log Contents:
    ...
    ```

* It also creates a directory per node (a.k.a. "host") containing symlinks to the log-files of all containers that ran on that node:

    ```
    $ cd hosts
    $ ls -l my-node-*
    my-node-08-1:
    lrwxrwxrwx 1 <user> <group> 41 Nov 20 04:42 container_1416279928169_0018_01_000065 -> ../container_1416279928169_0018_01_000065
    lrwxrwxrwx 1 <user> <group> 41 Nov 20 04:42 container_1416279928169_0018_01_000094 -> ../container_1416279928169_0018_01_000094
    lrwxrwxrwx 1 <user> <group> 41 Nov 20 04:42 container_1416279928169_0018_01_000123 -> ../container_1416279928169_0018_01_000123
    lrwxrwxrwx 1 <user> <group> 41 Nov 20 04:42 container_1416279928169_0018_01_000258 -> ../container_1416279928169_0018_01_000258
    lrwxrwxrwx 1 <user> <group> 41 Nov 20 04:42 container_1416279928169_0018_01_000338 -> ../container_1416279928169_0018_01_000338

    my-node-08-10:
    lrwxrwxrwx 1 <user> <group> 41 Nov 20 04:42 container_1416279928169_0018_01_000041 -> ../container_1416279928169_0018_01_000041
    lrwxrwxrwx 1 <user> <group> 41 Nov 20 04:42 container_1416279928169_0018_01_000158 -> ../container_1416279928169_0018_01_000158
    lrwxrwxrwx 1 <user> <group> 41 Nov 20 04:42 container_1416279928169_0018_01_000275 -> ../container_1416279928169_0018_01_000275
    lrwxrwxrwx 1 <user> <group> 41 Nov 20 04:42 container_1416279928169_0018_01_000354 -> ../container_1416279928169_0018_01_000354
    lrwxrwxrwx 1 <user> <group> 41 Nov 20 04:42 container_1416279928169_0018_01_000424 -> ../container_1416279928169_0018_01_000424
    ...
    ```

    * This functionality lives in [`rename-and-link-hosts`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/rename-and-link-hosts).
    * In this example, the per-node directories have had a shared suffix of the form `.rest.of.domain.name_<port>` removed for brevity; this is enabled by setting the `$YARN_HELPERS_DROP_HOST_SUFFIX_FROM` environment variable; see the [Installing](#installing) section for more details on setting `$YARN_HELPERS_DROP_HOST_SUFFIX_FROM`.

##### Spark-specific parsing
A common use case is parsing logs from Spark apps running on YARN, for which `yarn-container-logs` has some specific functionality:
* It can identify the logs corresponding to Spark driver containers. It `grep`s all container logs for `spark.SparkContext` to identify drivers (you can override this by setting the `$YARN_HELPERS_DRIVER_GREP_NEEDLE` environment variable), and creates symlinks to them in the `drivers` directory:

        $ ls -l drivers
        lrwxrwxrwx 1 <user> <group> 41 Nov 20 04:42 0 -> ../container_1416279928169_0018_01_000015
        lrwxrwxrwx 1 <user> <group> 41 Nov 20 04:42 container_1416279928169_0018_01_000015 -> ../container_1416279928169_0018_01_000015

    If exactly one was found, an additional top-level `driver` symlink will point to it:

        $ ls -l driver
        lrwxrwxrwx 1 <user> <group> 9 Nov 20 04:42 driver -> drivers/0

    This functionality lives in [`link-driver-logs`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/link-driver-logs).

* It will create a `tids` directory and populate it with symlinks for each Spark task ID that it finds evidence of in the logs to the container-log-file where that TID seemingly ran.
    * This functionality lives in [`link-tids`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/link-tids).
    * Note that if a Spark job had multiple Application Masters ("drivers"), it will likely have had multiple tasks with some task IDs, which will cause errors to be emitted by this stage. See discussion at [#2](https://github.com/hammerlab/yarn-logs-helpers/issues/2#issuecomment-63861447).

#### Stack Trace Parsing / Histogram
`yarn-logs-stack-traces` uses a [stack-trace-parsing library](https://github.com/ryan-williams/stack-traces) on the output of `yarn-logs`. Example usage:
```
$ yls 0018 -d  # -d means "show a histogram in descending order"
635 stacks in total

71 occurrences:
org.apache.spark.shuffle.MetadataFetchFailedException: Missing an output location for shuffle 4
        at org.apache.spark.MapOutputTracker$$anonfun$org$apache$spark$MapOutputTracker$$convertMapStatuses$1.apply(MapOutputTracker.scala:386)
        at org.apache.spark.MapOutputTracker$$anonfun$org$apache$spark$MapOutputTracker$$convertMapStatuses$1.apply(MapOutputTracker.scala:383)
        at scala.collection.TraversableLike$$anonfun$map$1.apply(TraversableLike.scala:244)
        ...
        at java.lang.Thread.run(Thread.java:744)

60 occurrences:
java.io.IOException: Failed to connect to demeter-csmaz11-16.demeter.hpc.mssm.edu/172.29.46.86:33263
        at org.apache.spark.network.client.TransportClientFactory.createClient(TransportClientFactory.java:141)
        at org.apache.spark.network.netty.NettyBlockTransferService$$anon$1.createAndStart(NettyBlockTransferService.scala:78)
        at org.apache.spark.network.shuffle.RetryingBlockFetcher.fetchAllOutstanding(RetryingBlockFetcher.java:140)
        ...
        at io.netty.util.concurrent.SingleThreadEventExecutor$2.run(SingleThreadEventExecutor.java:116)

...
```

#### Other Miscellaneous Scripts
This repo contains several other scripts that basically wrap YARN commands in calls to [`yarn-appid`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/yarn-appid), allowing last-4-lookup of application IDs:
* [`yarn-kill`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/yarn-kill): wrapper for `yarn application -kill <appid>`.
* [`yarn-logs`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/yarn-logs): wrapper for `yarn logs -applicationId <appid>`.
    * [`yarn-logs-less`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/yarn-logs-less): pipes `yarn-logs` to `less`.

# Installing
Download this repository with:

        git clone --recursive https://github.com/hammerlab/yarn-logs-helpers.git

In your `.bashrc` (or equivalent), source [`.yarn-logs-helpers.sourceme`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/.yarn-logs-helpers.sourceme):

        $ source /path/to/repo/.yarn-logs-helpers.sourceme

This will:
* try to fetch your cluster's ID using the [`yarn-refresh-cluster-id`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/yarn-refresh-cluster-id) script.
    * If found, the result will be cached in `$yarn_cluster_id_file` ([default: `$HOME/.yarn-cluster-id`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/.yarn-logs-helpers.sourceme#L15)).
    * This will allow all scripts in this repo to look up YARN application IDs by their last 4 digits (using [`yarn-appid`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/yarn-appid)).
* [set aliases](https://github.com/hammerlab/yarn-logs-helpers/blob/master/.yarn-logs-helpers.sourceme#L22-L31) for most functionality in this repo.
* add the root directory of this repo to your `$PATH`.

## Env vars
Setting `$YARN_LOGS_USER` may allow `yarn-container-logs` to fetch logs from apps run by users other than you.

You can set it permanently in your `.bashrc` to a user that has permissions to read all YARN users' logs, or just on the cmdline for one call:

```bash
YARN_LOGS_USER=someone yarn-logs 1234
```

You may also want to export `YARN_HELPERS_DROP_HOST_SUFFIX_FROM` (discussed above):

        # Pattern for abbreviating host names when creating per-host log directories.
        export YARN_HELPERS_DROP_HOST_SUFFIX_FROM=".rest.of.domain.name_"

##### `stack-traces` submodule
Finally, [ryan-williams/stack-traces](https://github.com/ryan-williams/stack-traces) is included in this repository as a git submodule, and used by [`yarn-log-stack-traces`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/yarn-log-stack-traces).

You'll need to `git clone --recursive` when you check out the project, or run `git submodule init && git submodule update` from within the `stack-traces` subdirectory, for it to work. [git-scm.com](http://git-scm.com/book/en/v2/Git-Tools-Submodules#Cloning-a-Project-with-Submodules) has a good intro to using git submodules if you are not familiar.

With those done you should be all set!
