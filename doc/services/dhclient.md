# dhclient

Source: [dhclient.cf](/services/dhclient.cf)

For now this bundle generate only the 'dhclient-enter-hooks' file:
 * debian: /etc/dhcp/dhclient-enter-hooks.d/cfengine_generated
 * centos: /etc/dhcp/dhclient-enter-hooks

You can control what will be in in the hooks file by setting classes:
 *  def.cf/def.json, override json file. This will set `DHCLIENT_RESOLV_CONF` for `any`.
```json
"dhclient": {
    "classes": {
        "RESOLV_CONF": "any"
    }
}
```

## Usage

The bundle can be run via:
 * `def.scl_services_enabled`
```json
"vars": {
    "scl_services_enabled": [
            "...",
            "dhclient",
            "..."
    ]
}
```

The bundle will aways read the [default.json](/templates/dhclient/json/default.json) file
and extra json file(s) can be specified via:
 * def.cf
```
vars:
    any::
        "dhclient_json_files" slist => { "override_resolv.json" };
```

The variable must be `dhclient_json_files` and with this setup 1 extra json file will be  merged.

### DEBUG

f you want to debug this bundle set the `DEBUG_dhclient` class, eg:
 * `-DDEBUG_dhclient`

### def.cf/json

See [default.json](/templates/dhclient/json/default.json) what the default values are and
which variables can be overriden.

The following must be set in host specific host file: `def.json`
```json
"classes": {
    "DHCLIENT_BUNDLE": "any",
},
"vars": {
    "dhclient": {
        "classes": {
            "RESOLV_CONF": "any"
        }
    }
}
```

In def.cf/def.json you can override the `resolv_conf` function, eg def.json:
```json
"vars": {
    "dhclient": {
        "resolv_conf":
"
make_resol_conf()
{
    <something you wrote>
}
"
    },
}
```
