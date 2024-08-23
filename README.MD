# Enroll hosts in Fleet without letting Fleet manage queries

The setup here woould result in hosts that Do:

- Enroll in Fleet
- Have host details provided at enrollment present in the Fleet Database
- Auto-update fleetd components (configurable in Agent Options)
- Run locally-defined scheduled queries
- Use locally-defined osquery configuration
- Send osquery results and status logs to Fleet to be forwarded to your logging destination
- Update osquery command_line flags as defined in Fleet Agent options *

And do not:

- Respond to Live queries
  - This includes the detail queries used by Fleet to gather software inventory, host information, and policy status
- Get scheduled queries from Fleet
- Update osquery options defined in Fleet, with the exception of command_line flags *


> * This may be able to be changed, but I have not tested that behavior

## Setup

Update any file paths in `osquery.conf` and `osquery.flags` to match where you'll be placing things on the host

### Fleet

Create a new team for these hosts. I'd reccomend setting up the Team's Agent options like this:

```
  command_line_flags:
    config_path: /path/to/local/osquery.conf
    config_plugin: filesystem
    disable_distributed: true
```

There isn't a way that I have been able to find to disable setting command_line_flags in Agent Options that wouldn't 
disable other functionality that we do want. In that light, I feel like explicitly setting the flags here as well makes sense.

### Build the package

Locally, create a package and pass the path to the `osquery.flags` file to `--osquery-flagfile`

`fleetctl package [...other flags] --osquery-flagfile /path/to/local/osquery.flags`

> If you don't mind osquery checking in once to get the initial config and you've already got the agent options set
  up as described above, you can skip building a special package and just enroll as you usually do.

### On the host

Place an `osquery.conf` file and an optional `packs` directory with pack configuration files

> Examples in `config`

Install fleetd (use the custom package, or let fleetd pick up the config on first launch)

## Notes on the config files

The osquery docs are pretty thurough in describing your config options:

https://osquery.readthedocs.io/en/stable/deployment/configuration/

If you want to match your Fleet agent options, the `config` object in Fleet is the `osquery.conf` object and things 
can be carried over from there.

The example osquery.conf would match these Agent Options:

```
config:
  options:
    pack_delimiter: /
    logger_tls_period: 10
    logger_tls_endpoint: /api/osquery/log
  decorators:
    load:
      - SELECT uuid AS host_uuid FROM system_info;
      - SELECT hostname AS hostname FROM system_info;
command_line_flags:
  config_path: /path/to/local/osquery.conf
  config_plugin: filesystem
  disable_distributed: true
```


What doesn't show up in Agent Options is the schedule.

Under the hood, Fleet uses `packs` to set the osquery schedule, so I've used that in the example as well.

If you'd like the results logs to look as similar as possible to the results sent by other hosts, you'd use a `Global` pack for queries assigned to "All Teams" in fleet, and then a `team-<id> `pack for the team schedule, so that's how I have things set up. 

I've also included a `packs` directory. If you're working with a lot of queries and would rather manage extra files than one very 
long file, you can use this setup for packs:

```
 "packs": {
    "*": "/path/to/local/packs/*"
 }
```

Then add your files to that directory. The name of the file will be used as the pack name (Global.conf would be `Global`). I've included an example that lines up with the Fleet pack naming conventions as well.

> Excuse my very lazy query definitions there.