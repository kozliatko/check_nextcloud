# Nagios/Centron check | Nextcloud serverinfo

[![Security](https://github.com/kozliatko/check_nextcloud/actions/workflows/security.yml/badge.svg)](https://github.com/kozliatko/check_nextcloud/actions/workflows/security.yml)
[![Lint](https://github.com/kozliatko/check_nextcloud/actions/workflows/lint.yml/badge.svg)](https://github.com/kozliatko/check_nextcloud/actions/workflows/lint.yml)
![License](https://img.shields.io/github/license/kozliatko/check_nextcloud)
![Python](https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white)
![Nagios/Icinga](https://img.shields.io/badge/Nagios%2FIcinga-plugin-004357)
![Last commit](https://img.shields.io/github/last-commit/kozliatko/check_nextcloud)

Nagios/Centreon plugin for nextcloud serverinfo API (https://github.com/nextcloud/serverinfo)

This branch contains the check for Python 3. A version for Python 2.7 can be found [here](https://github.com/BornToBeRoot/check_nextcloud/tree/stable-python2.7).

## Syntax / Help

```
./check_nextcloud.py -u username -p password -H cloud.example.com -c [system|storage|shares|webserver|php|database|users|apps]


Options:
  -h, --help            show this help message and exit
  -v, --version         Print the version of this script
  -u USERNAME, --username=USERNAME
                        Username of the user with administrative permissions
                        on the nextcloud server
  -p PASSWORD, --password=PASSWORD
                        Password of the user
  -t TOKEN, --nc-token=TOKEN
                        NC-Token for the Serverinfo API
  -H HOSTNAME, --hostname=HOSTNAME
                        Nextcloud server address (make sure that the address
                        is a trusted domain in the config.php)
  -c CHECK, --check=CHECK
                        The thing you want to check
                        [system|storage|shares|webserver|php|database|activeUsers|uploadFilesize|apps]
  --upload-filesize     Filesize in MiB, GiB without spaces (default="512.0GiB")
  --protocol=PROTOCOL   Protocol you want to use [http|https]
                        (default="https")
  --ignore-proxy        Ignore any configured proxy server on this system for
                        this request
  --max-major-behind=MAX_MAJOR_BEHIND
                        Check "system": WARNING when the installed major
                        version is more than N majors behind the latest
                        release on download.nextcloud.com
  --release-cache=RELEASE_CACHE
                        Cache file for the latest release, refreshed every 12
                        hours
                        (default="/var/tmp/check_nextcloud_latest_release")
  --api-url=API_URL     Url of the api
                        (default="/ocs/v2.php/apps/serverinfo/api/v1/info")

```

## Install

* Copy the check (python script) in your nagios/centreon plugins folder
* Create a check with the following command line:
  ```
  $USER1$/plugins_custom/check_nextcloud.py -u $_HOSTCLOUDUSER$ -p $_HOSTCLOUDPWD$ -H $HOSTNAME$ -c $ARG1$ --ignore-proxy
  ```
  or for the check: `uploadFilesize`
  ```
  $USER1$/plugins_custom/check_nextcloud.py -u $_HOSTCLOUDUSER$ -p $_HOSTCLOUDPWD$ -H $HOSTNAME$ -c $ARG1$ --upload-filesize=$ARG2$ --ignore-proxy
  ```
  or for the check `system` with a warning for outdated versions (see [Example 4](#example-4))
  ```
  $USER1$/plugins_custom/check_nextcloud.py -u $_HOSTCLOUDUSER$ -p $_HOSTCLOUDPWD$ -H $HOSTNAME$ -c system --max-major-behind=$ARG2$ --ignore-proxy
  ```
* Create a service for each thing you want to check (system, storage, etc.) and link it to your host(s)
* Add the credentials of your nextcloud admin user as custom macro (CLOUDUSER, COUDPWD) to your host.

## Example 1

```
./check_nextcloud.py -u adminUser -p secretPassword -H cloud.example.com -c system --ignore-proxy

OK - Nextcloud version: 12.0.3.3
```

## Example 2

```
./check_nextcloud.py -u adminUser -p secretPassword -H cloud.example.com -c activeUsers --ignore-proxy

OK - Last 5 minutes: 3 user(s), last 1 hour: 10 user(s), last 24 hour: 44 user(s) | users_last_5_minutes=3, users_last_1_hour=10, users_last_24_hours=44
```

This will return a status message and create a graph based on the performance data.

## Example 3

```
./check_nextcloud.py -u adminUser -p secretPassword -H cloud.example.com -c uploadFilesize --upload-filesize=2.0GiB --ignore-proxy

OK - Upload max filesize: 2.0GiB

# Or, when changed after an update...

CRITICAL - Upload max filesize is set to 512.0MiB, but should be 2.0GiB

```



## Example 4

```
./check_nextcloud.py -u adminUser -p secretPassword -H cloud.example.com -c system --max-major-behind=2

OK - Nextcloud version: 33.0.9.1 (latest release 35.0.2)

# Or, when the server falls too far behind...

WARNING - Nextcloud version: 32.0.6.1, 3 major versions behind the latest release 35.0.2
```

With `--max-major-behind=N` the check `system` compares the installed major version with the newest stable release
listed on https://download.nextcloud.com/server/releases/ and returns WARNING when the server is more than N major
versions behind. Only the major number counts: with `N=2` and the latest release 35.x, versions 33.x and newer are OK
and 32.x is WARNING. Without the parameter the check behaves as before and always returns OK.

The latest release is cached in the file given by `--release-cache` (default `/var/tmp/check_nextcloud_latest_release`)
and refreshed every 12 hours, so checking many servers makes at most one request to download.nextcloud.com. The file
must be writable by the user running the check. When download.nextcloud.com cannot be reached, the last cached value
is used; without one the check returns OK with the note `(latest release unknown)`, so an outage of the download
server never raises an alert.

## Icinga config example


Adjust the command path to your local situation.

```
object CheckCommand "check_nextcloud" {
  command = [ "/var/lib/nagios/src/check_nextcloud/check/check_nextcloud.py" ]
  arguments = {
    "--nc-token" = {
      value = "$nextcloud_token$"
      description = "NC-Token for the Serverinfo API"
    }
    "--hostname" = {
      value = "$nextcloud_hostname$"
      description = "Hostname"
    }
    "--api-url" = {
      value = "$nextcloud_api_url$"
      set_if = "$nextcloud_api_url$"
      description = "Api-url"
    }
    "--check" = {
      value = "$nextcloud_check$"
      description = "Which check to run"
    }
    "--perfdata-format" = {
      value = "nagios"
      description = "The perfdata format we like"
    }
    "--max-major-behind" = {
      value = "$nextcloud_max_major_behind$"
      set_if = "$nextcloud_max_major_behind$"
      description = "Check system: WARNING when more than N major versions behind the latest release"
    }
  }
}
```

```
apply Service for (checkname in ["system","storage","shares","webserver","php","database","activeUsers","uploadFilesize","apps"]) {
  import "generic-service"
  name = "check-nextcloud-" + checkname
  check_interval = 30m
  retry_interval = 10m
  display_name = "Nextcloud monitor " + checkname
  vars.notification_interval = 1d

  vars.nextcloud_check = checkname
  vars.nextcloud_hostname = host.vars.nextcloud_hostname
  vars.nextcloud_token = host.vars.nextcloud_token
  vars.nextcloud_api_url = host.vars.nextcloud_api_url
  if (checkname == "system") {
    vars.nextcloud_max_major_behind = 2
  }
  vars.notification["mail"] = {  }
  check_command = "check_nextcloud"

  assign where (host.address || host.address6) && host.vars.nextcloud_token
}
```

```
object Host "server42.example.com" {

  display_name = "My Nextcloud server"
  address = "<IP>"

  ...

  # The token can be set with: occ config:app:set serverinfo token --value yourtoken
  vars.nextcloud_token = "XXX"
  vars.nextcloud_hostname = "nextcloud.example.com"

  # Optional if you e.g. use a subdirectory.
  vars.nextcloud_api_url = "/subdir/ocs/v2.php/apps/serverinfo/api/v1/info"
}

```
