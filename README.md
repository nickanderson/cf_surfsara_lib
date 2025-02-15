<!-- vim-markdown-toc GFM -->

* [SURF CFEngine Library (SCL) for building services with json/mustache](#surf-cfengine-library-scl-for-building-services-with-jsonmustache)
    * [mustache/json rules](#mustachejson-rules)
        * [JSON Merge strategy](#json-merge-strategy)
    * [Installation](#installation)
        * [MPF installation](#mpf-installation)
            * [update](#update)
        * [Own framework](#own-framework)
        * [def.node\_template\_dir](#defnode_template_dir)
        * [CF-serverd shortcut configuration for cfengine version less then 3.10.1](#cf-serverd-shortcut-configuration-for-cfengine-version-less-then-3101)
    * [Usage](#usage)
        * [scl\_services\_enabled run method](#scl_services_enabled-run-method)
        * [def.json](#defjson)
            * [Service classes](#service-classes)
        * [lib/scl/def.cf](#libscldefcf)
    * [cf-agent command line options](#cf-agent-command-line-options)

<!-- vim-markdown-toc -->
# SURF CFEngine Library (SCL) for building services with json/mustache

The SCL distribution consists of a library and services part. The services are build with
the SCL building blocks that can be controlled/configured with json data. The SCL building blocks
are, eg:
 * easy generation of configuration files with mustache and json data (merging strategies)
 * package installation
 * tarball installation
 * copy directories
 * copy files
 * inventory framework. Which inventory programs must be run, default any class
 * services framework. Which services must be included and run, defauly any class
 * inline documentation

The goal is to setup a library were we can easily install/configure/maintain services. There are many
services included and these are used at SURF for our HPC clusters and Office Automation. We hope that
this is also useful for others and that will grow as the standard repo for cfengine services. In Ansible
they call it playbooks and in Salt terms formulas.

## mustache/json rules

The project started as library for generating configuration files with mustache/json.  In our CFengine
configuration we used different methods and we wanted a standard method for generating these configuration
files. This is the standard that we defined for the mustache/json files and json merging:
 * For all bundles the mustache/json file(s) will be copied to the local node directory (`$(def.node_template_dir)`
 * The json and template file(s) are copied from the policy hub shortcut: `templates/$(service_name)`
 * The copies are placed in the local node directory: `$(def.node_template_dir)/$(service_name)`
 * The following json must always be present and will always be copied: *default.json*
 * Extra json file(s) can be specified in *def.cf/json*: `$(service_name)[json_files]`
 * Scripts can generate json file(s) on a host/node. The json file must be copied into:
    * `$(def.node_template_dir)/$(service_name)`
    * The generated file(s) are specified in def.cf/json: `$(service_name)[local_generated_json_files]`
 * You can override values via *def.json*, Note: This one always wins.
 * CFEngine variables are expanded:
    *  Bundle data two levels.
    *  When all bundle data is parsed. This allows referencing a variable of another service bundle.

Both senarios will be described in the subsection below. For both senarios you can specifiy multiple
json files. The files will be merged and the last one wins if the same variable name is used,eg:
 * a.json defines: `a : 1`
 * b.json defines: `a : 2`

If the order is `{ "b.json", "a.json" }` the value of *a* would be *1*

This framework depends on the augemnts file `def.json`, we have developed a multiple augments strategy:
 * https://docs.cfengine.com/docs/3.15/reference-language-concepts-augments.html
 * https://basvandervlies.blogspot.com/2018/09/cfengine-312-new-features-missingok-and.html

### JSON Merge strategy

The merge strategy is::
  1. `default.json`
  1. `def.<service_name>_json_files` if defined
  1. `def.<service_name>[json_files]` if defined
  1. `def.<service_name>_local_generated_json_files` if defined
  1. `def.<service_name>[local_generated_json_files]` if defined
  1. `def.<service_name>` if defined in def.json or:
        * lib/scl/def.cf MPF setup
        * your own file with variable scope `def`

## Installation

there are two options
 * Include it in the Master Policy Framework (MPF)
 * Include it in your own framework

### MPF installation

1. Login on your policy server.
1. `./mpf_installation`
1. Enable autorun, if you have not done it. Add this line to your ```def.json``` file
```json
{
   "classes" :
   {
    "services_autorun" : "any"
   }
}
```

You can test your installation with
 * `cf-agent -Kv | grep scl\_autorun`

#### update ####

You can run the same script it will detect its an update. This script will overwrite:
 * scl library files: `masterfiles/lib/scl`
 * scl services files: `masterfiles/services/scl`
 * mustache template files and default.json files: `/var/cfengine/templates`

### Own framework

1. Login on your policy server.
1. `cp -a masterfiles/lib/scl <masterfiles>/lib/scl`
1. `cp -a templates/\* $(sys.workdir)/templates`
1. include `/lib/scl/stdlib.cf` in your inputs
```
body common control
{
    inputs => {
        ...
        "lib/scl/stdlib.cf",
        ...
    };
}
```

### def.node\_template\_dir

The  `def.node_template_dir` variable is set in `lib/scl/def.cf`, but can also be set
set in `def.json`. The *def.json* wins, eg:
```
vars:
{
   "node_template_dir" : "/etc/node_status/templates"
}
```

default value is: `/var/cfengine/node_templates`

### CF-serverd shortcut configuration for cfengine version less then 3.10.1

For older versions you have to manually add the `shorcut templates` to `controls/cf_serverd.cf`
```
      "$(sys.workdir)/templates"
      handle => "server_access_grant_access_templates",
      shortcut => "templates",
      comment => "Grant access to templates directory",
      admit => { @(def.acl) };
```
See above to add `templates shortcut` to cf-serverd.

## Usage

The documentattion is embedded in the source files, and generated:
 * [Library documentation](doc/library.md)
 * [Services documentation](doc/services.md)

There are several services setups included with inline documentation. These setups are
used in production at SURF.

To enable the service on your system use `def.scl_services_enabled` method in def.cf/def.json for
both installations method

###  scl\_services\_enabled run method

This is the prefered method for MPF and your own frameork. With this method you can contol which services are run
and which files are included, eg: def.json
```
"vars": {
    "scl_services_enabled": [
        "ntp",
        "resolv"
    ]
}
```
This will include the service files `ntp.cf` and `resolv.cf` and run all bundles that have the meta tag
`service\_ntp` and `service\_resolv`. The bundle run can be protected by an class statement, default is `any`, eg:
```
"ntp": {
    "run_class": "debian|centos"
    }
```
This will only run on  debian or centos hosts.

### def.json

In this file you can override settings for the services.
```json
"vars": {
    "ntp" : {
        "server": [ "<your_ip_server1>", "<your_ip_server2>" ]
    }
}
```
You can also specify json setup files:
```json
"vars": {
    "tcpwrappers": {
        "json_files": [ "allow_ssh.json", "allow_http.json" ]
    }
}
```

or:
```json
"vars": {
    "tcpwrappers_json_files": [ "allow_ssh.json", "allow_http.json" ]
}
```

You can use both definition at the same same time.  the `tcpwrappers_json_files` definition is read first and can be overriden by
the  `tcpwrappers: { json_files }` definition. See the merge strategy.

#### Service classes

For every service you dynamically set classes in the service data, eg:
```
"vars": {
    "dhclient": {
       "classes": {
           "RESOLV_CONF": [ "r24n2" ]
       }
    }
}
```
This will set the class `DHCLIENT\_RESOLV\_CONF` on host/node `r24n2`

### lib/scl/def.cf

You can also override settings in this file, eg:
 * One variable:
```
vars:
    "ntp" data => parsejson( '{ "server" : [ "<your_ip_server1>" ] }' );
```
 * json file:
```
vars:
    "tcpwrappers" data => parsejson( '{ "json_files": [ "allow_ssh.json", "allow_http.json" ] '} );
```

If you defined your own `def.cf` and do not want the one included in this framework you can set the following class:
 * `SCL_SKIP_DEF_CF_INCLUDE`


## cf-agent command line options

The SURF CFEngine library also checks for some classes:
 * To test with a local `templates` directory. This directory must be one level higher than your policy files directory (../templates):
  * `-DTEMPLATE_LOCAL_COPY`: Copy from local directory the mustache and json file(s).
  * `-DMUSTACHE_LOCAL_COPY`: Copy from local directory the mustache file(s)
  * `-DJSON_LOCAL_COPY`: Copy from local directory the json file(s)
 * To test local mustache/json changes in `$(def.node_template_dir)`, the copy of the json/mustache file(s) from the policy server can be skipped by:
  * `-DTEMPLATE_SKIP_COPY`: Skip copying of mustache and json files
  * `-DMUSTACHE_SKIP_COPY`:  Skip copying of the mustache files
  * `-DJSON_SKIP_COPY`: Skip copying of the json files
 * To debug the mustache setup: `-DDEBUG_MUSTACHE` (all service bundles)
 * To debug mustache for a service bundle, eg `-DDEBUG_ntp`

