yarn-logs-helpers
=================

Scripts for parsing / making sense of yarn logs

# Contents

#### `yarn-container-logs`
The main script of note here is `yarn-container-logs`:

```
$ yarn-container-logs 0018
```

*  It can take a full application ID (e.g. `application_1416279928169_0018`) or just the last 4 digits of one (`0018`).
*  It downloads the YARN logs for that application into a local directory and splits them into per-container files:
    ```
    # Directory created by yarn-container-logs
    $ cd application_1416279928169_0018

    # Per-container log files have prefix /container_/
    $ ls container_*
    container_1416279928169_0018_01_000015
    container_1416279928169_0018_01_000016
    container_1416279928169_0018_01_000017
    ...

    # The files contain exactly what we pulled down from YARN.
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
    ```
    In this example, the per-node directories have a common suffix `.rest.of.domain.name_port` removed (node hostnames are parsed from log files themselves; see earlier example of the full hostname of `my-node-11-10`), for brevity/clarity; this is enabled by setting the `$YARN_HELPERS_DROP_HOST_SUFFIX_FROM` environment variable. I do this in my `.bashrc` so that it is always set by default:

        # leaves the <port> unspecified, but everything from this to the right is dropped.
        export YARN_HELPERS_DROP_HOST_SUFFIX_FROM=".rest.of.domain.name_"

    This functionality lives in [`rename-and-link-container-logs`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/rename-and-link-container-logs).

* Finally, we are typically parsing logs from Spark jobs running on YARN, and it is useful to specifically identify the logs corresponding to Spark "driver"(s); if the `$YARN_HELPERS_DRIVER_GREP_NEEDLE` environment variable is set, then `yarn-container-logs` will `grep` over all container log-files for its value, and upon finding matches, will create a `drivers` directory with symlinks to those log-files, by name and by index in which they were found:

        $ ls -l drivers
        lrwxrwxrwx 1 <user> <group> 41 Nov 20 04:42 0 -> ../container_1416279928169_0018_01_000015
        lrwxrwxrwx 1 <user> <group> 41 Nov 20 04:42 container_1416279928169_0018_01_000015 -> ../container_1416279928169_0018_01_000015

    If exactly one was found, an additional `driver` symlink will point to it:

        $ ls -l driver
        lrwxrwxrwx 1 <user> <group> 9 Nov 20 04:42 driver -> drivers/0

    I also enable this via my `.bashrc`, with a line like "Job starting!" that I expect to only find in the stdout of driver workers.

        export YARN_HELPERS_DRIVER_GREP_NEEDLE="Job starting!"

    This functionality lives in [`link-driver-logs`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/link-driver-logs).

#### Other Miscellaneous Scripts
This repo contains several other scripts that basically wrap YARN commands in calls to [`yarn-appid`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/yarn-appid), allowing last-4-lookup of application IDs:
* `yarn-kill`: wrapper for `yarn application -kill`
* `yarn-logs`: wrapper for `yarn logs -applicationId`
    * `yarn-logs-less`: pipes `yarn-logs` to `less`

##### Looking up Applications by Their Last 4 Digits
All scripts in this repo can look up YARN application IDs by their last 4 digits using code that your `.yarn-logs-helpers-config.sourceme` [reads in](https://github.com/hammerlab/yarn-logs-helpers/blob/master/.yarn-logs-helpers-config.sourceme.sample#L13) from [`.yarn-logs-helpers.sourcme`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/.yarn-logs-helpers.sourceme).

When `source`d, `.yarn-logs-helpers.sourcme` will try to fetch your cluster's ID using the [`yarn-refresh-cluster-id`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/yarn-refresh-cluster-id) script, and cache the result in `$yarn_cluster_id_file` ([default: `$HOME/.yarn-cluster-id`](https://github.com/hammerlab/yarn-logs-helpers/blob/master/.yarn-logs-helpers.sourceme#L10)).

Other scripts will use this cached value to look up full application IDs. If your cluster restarts, you'll need to run `yarn-refresh-cluster-id` again to pick up the new value.


# Installing
* Copy the provided sample "sourceme" file to `.yarn-logs-helpers-config.sourceme`:

        $ cp .yarn-logs-helpers-config.sourceme{.sample,}
* Next, set the values of `YARN_HELPERS_DRIVER_GREP_NEEDLE` and `YARN_HELPERS_DROP_HOST_SUFFIX_FROM` in `.yarn-logs-helpers-config.sourceme` appropriately for your cluster / job type.
* Add a line to your `.bashrc` or equivalent to `source` this file on shell startup:
        source .yarn-logs-helpers-config.sourceme

