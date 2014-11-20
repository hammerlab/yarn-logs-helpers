yarn-logs-helpers
=================
Scripts for parsing / making sense of yarn logs
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

* It also creates a directory per node containing symlinks to the log-files of all containers that ran on that node:

    ```
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

    * This functionality lives in [`rename-and-link-container-logs`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/rename-and-link-container-logs).
    * In this example, the per-node directories have a common suffix `.rest.of.domain.name_<port>` removed for brevity; this is enabled by setting the `$YARN_HELPERS_DROP_HOST_SUFFIX_FROM` environment variable.
    * See the [Installing](#installing) section for more details on setting `$YARN_HELPERS_DROP_HOST_SUFFIX_FROM`.

##### Spark-specific parsing
A common use case is parsing logs from Spark jobs running on YARN, for which `yarn-container-logs` has some specific functionality:
* It can identify the logs corresponding to Spark "driver"(s); if the `$YARN_HELPERS_DRIVER_GREP_NEEDLE` environment variable is set, then `yarn-container-logs` will `grep` over all container log-files for its value, and upon finding matches, will create a `drivers` directory with symlinks to those log-files, by name and by index in which they were found:

        $ ls -l drivers
        lrwxrwxrwx 1 <user> <group> 41 Nov 20 04:42 0 -> ../container_1416279928169_0018_01_000015
        lrwxrwxrwx 1 <user> <group> 41 Nov 20 04:42 container_1416279928169_0018_01_000015 -> ../container_1416279928169_0018_01_000015

    * If exactly one was found, an additional `driver` symlink will point to it:

            $ ls -l driver
            lrwxrwxrwx 1 <user> <group> 9 Nov 20 04:42 driver -> drivers/0

    * This functionality lives in [`link-driver-logs`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/link-driver-logs).
    * See the [Installing](#installing) section for more details on setting `$YARN_HELPERS_DRIVER_GREP_NEEDLE`.
* It will create a `tid` directory and populate it with symlinks for each Spark task ID that it finds evidence of in the logs to the container-log-file where that TID seemingly ran.
    * This functionality lives in [`link-tids`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/link-tids).
    * Note that if a Spark job had multiple Application Masters ("drivers"), it will likely have had multiple tasks with some task IDs, which will cause errors to be emitted by this stage. See discussion at [#2](https://github.com/hammerlab/yarn-logs-helpers/issues/2#issuecomment-63861447).

#### Other Miscellaneous Scripts
This repo contains several other scripts that basically wrap YARN commands in calls to [`yarn-appid`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/yarn-appid), allowing last-4-lookup of application IDs:
* [`yarn-kill`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/yarn-kill): wrapper for `yarn application -kill <appid>`.
* [`yarn-logs`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/yarn-logs): wrapper for `yarn logs -applicationId <appid>`.
    * [`yarn-logs-less`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/yarn-logs-less): pipes `yarn-logs` to `less`.

# Installing
There are a few things you should do in your `.bashrc` (or equivalent):
* Set `yarn_aliases_dir` if you want [`.yarn-logs-helpers.sourceme`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/.yarn-logs-helpers.sourceme) to create symlinks to most scripts in this repo for you:

        export yarn_aliases_dir=/some/arbitrary/writable/path

* Source [`.yarn-logs-helpers.sourceme`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/.yarn-logs-helpers.sourceme):

        $ source /path/to/repo/.yarn-logs-helpers.sourceme

    * This will try to fetch your cluster's ID using the [`yarn-refresh-cluster-id`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/yarn-refresh-cluster-id) script.
    * If found, the result will be cached in `$yarn_cluster_id_file` ([default: `$HOME/.yarn-cluster-id`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/.yarn-logs-helpers.sourceme#L15)).
    * This will allow all scripts in this repo to look up YARN application IDs by their last 4 digits (using [`yarn-appid`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/yarn-appid)).
* Export values environment variables discussed above, as appropriate for your cluster / job type:

        # leaves the <port> unspecified, but everything from this to the right is dropped.
        export YARN_HELPERS_DROP_HOST_SUFFIX_FROM=".rest.of.domain.name_"

        # Set to a piece of output you expect to only find in "driver"s' logs.
        export YARN_HELPERS_DRIVER_GREP_NEEDLE="Job starting!"

With those done you should be all set!
