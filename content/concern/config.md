---
title: "Config"
date: 2017-11-05T10:24:06-05:00
description: ""
categories: [concerns]
keywords: []
aliases: []
toc: false
draft: false
---

A configuration module that supports externalized config in official standalone
deployment or docker container. It is encouraged that every module should have
its own configuration file and these files can be served by light-config-server
which aggregates/merges config files from different level of organizations in
github or other git servers.


## Introduction

Externalized configuration from the application package is very important. It
allows us to deploy the same package to DEV/SIT/UAT/PROD environment with 
different configuration packages without reopening the delivery package. For
development, embedded configuration give us the flexibility to take default
values out of the box.

When dockerizing our application, the external configuration can be mapped to a
volume which can be updated and restarted the container to take effect.

When Kubernetes is used for docker orchestration, config files can be managed by
ConfigMap and Secret

The reason we encourage every module has its own configuration file is to ease
the management and facilitate individual module independence. Every module can
be upgraded independently and change its config format without impact other
modules. 

Most config files are not sensitive and it is recommended to manage them in git
so that all the changes can be tracked and audited. With proper access control
to the organization/lob level of config repo, only certain users can update the
config files per environment. 

There is one separate config file secret.yml that contains all the secrets in
the application like database password, private key etc. Although this file can
be checked into the git along with other config files, the content should be
just placeholders and it needs to be updated during deployment. With Kubernets,
this file will be mapped to Secrets.


For now, only file system configuration is supported; however, config files
can be served by light-config-server in a zip format and the server module
can be started to load all config from light-config-server in a zip file
format. It will automatically unzip it and put the config files into the 
right location to bootstrap the server. Given the risk of single point failure, 
only when the server start, it will contact external light-config-sever to load 
(first time) or update local file system so that config files are cached. Except 
first time start the server, the server can use the local cache to start if 
config service is not available.


## Singleton

For one JVM, there should be only one instance of Config and it should be
shared by all the modules within the application. The class is designed as a
singleton class and has several abstract methods that should be implemented
in sub classes. A default File System based implementation is provided. Other
implementation will be provided in the future when it is needed.
 
## Interface

The following abstract methods are implemented in the default service provider.

```
    public abstract Map<String, Object> getJsonMapConfig(String configName);

    public abstract Map<String, Object> getJsonMapConfigNoCache(String configName);

    public abstract Object getJsonObjectConfig(String configName, Class clazz);

    public abstract String getStringFromFile(String filename);

    public abstract InputStream getInputStreamFromFile(String filename);

    public abstract ObjectMapper getMapper();

    public abstract Yaml getYaml();

    public abstract void clear();

```

## Usage

It is encouraged to create a POJO for your config if the JSON structure is static.
However, some of the config file have repeatable elements, a map or list is the 
only options to represent the data in this case.

The two methods return String and InputStream is for loading files such as text
file with different format and X509 certificates etc.

getMapper() is a convenient way to get Jackson ObjectMapper instead of create one 
which is too heavy.


## Cache

Loading configuration files from file system takes time so the config file will
be cached once it is loaded the first time. All subsequent access to the same
config file (maybe different key/value pair) will read from the cached copy.
In order to make sure that server start up won't be impacted by all modules to
load config files, lazy loading is encouraged unless the module must be
initialized during server start up.

The server [info][] module will output the config files in map format for each 
enabled module and it creates another un-cached copy with getJsonMapConfigNoCache 
as some of the config contains sensitive info that needs to be masked. For example, 
secret.yml.


## Loading sequence

Each module should have a config file that named the same as the module name. And
each application or service built on top of light-4j framework should have a
config file named with application or service name if necessary. 

For example, Security module as a config file named security.yml and it is
reside in the module /src/main/resources/config folder by default.

For application that uses Security module, it can override the default module
level config by provide security.yml in application's resource folder 
/src/main/resources/config.

Config will also be loaded from class path of the application and this is
mainly used for testing as it is easy to inject different combination of
config copies in test cases into class path for unit testing.

For official runtime environment, config files should be externalized to a
separate folder outside of the package by a system property
"light-4j-config-dir". You can start the server with
a "-Dlight-4j-config-dir=/config" option and put all your files
into /config. Once in docker image, it can be mapped to a host volume
with -v /etc/light-4j:/config in docker run command.

Given above explanation, the loading sequence is:

1. System property "light-4j-config-dir" specified directory
2. Class Path
3. Application resources/config folder
4. Module resources/config folder

Most modules will have a default config file with yml format and it can be
overwritten by putting a yml config in application/service resources/config
folder. For unit test and end-to-end test, you can create another copy of
yml config file under test/resources/config folder. If you need to test the
different behaviours under different configuration, you need to manually
manipulate config file in classpath in your test case in order to simulate
different configurations. 

For official deploy environment, externalized config in light-4j-config-dir
is recommended; however, not all config files need to be externalized. Only
the ones that you might change from one environment to another or might be
changed on production under certain conditions. 

The above loading sequence assumes that yml is used as config file extension. If
yaml or json is used, the loading sequence is the same. Remember you only use
one extension for a module and these extensions cannot be mixed. 

The above loading sequence is for one extension and one extension only. For
different extensions, the loading sequence is:

1. yml
2. yaml
3. json


## Format

The config module supports two file formats and three file extensions.

* YAML with extension yml and yaml
* JSON with extension json

YAML is highly recommended because it is easy to read/edit with comments inline

For each module, you can choose either YAML or JSON as config file format but
you cannot use both as there is a priority in loading sequence: 

yml > yaml > json

Note that all default config files for light-4j and other framework built on
top of light-4j modules are using yml as config format except swagger.json
which is generated from swagger editor. If you want to overwrite the default
config for one of the modules, you must use the {module name}.yml in your app
config folder or externalized to config directory specified by system properties.

## Environment Variable

Some of the customers and vendors asked for environment variable support in config
module and this feature was added in 1.5.7 release. As environment variables are
highly dependable on the environment, we don't want to put it into our config module
directly but provide a utility for users to substitute values from environment props.

Here is an utility method added to StringUtil in utility module. 

```java
    public static String expandEnvVars(String text) {
        Map<String, String> envMap = System.getenv();
        String pattern = "\\$\\{([A-Za-z0-9-_]+)\\}";
        Pattern expr = Pattern.compile(pattern);
        Matcher matcher = expr.matcher(text);
        while (matcher.find()) {
            String envValue = envMap.get(matcher.group(1).toUpperCase());
            if (envValue == null) {
                envValue = "";
            } else {
                envValue = envValue.replace("\\", "\\\\");
            }
            Pattern subexpr = Pattern.compile(Pattern.quote(matcher.group(0)));
            text = subexpr.matcher(text).replaceAll(envValue);
        }
        return text;
    }

```

The format of the config string should be something like this. "IP=${DOCKER_HOST_IP}"
and ${DOCKER_HOST_IP} will be replaced with the environment variable DOCKER_HOST_IP.

This is a the test case that show you have it can be used.

```java
    public void testExpandEnvVars() {
        String s = "IP=${DOCKER_HOST_IP}";
        Assert.assertEquals("IP=192.168.1.120", StringUtil.expandEnvVars(s));
    }

```

Before you run the test case above, please export the environment variable. 

```
export DOCKER_HOST_IP=192.168.1.120
```

On my desktop, I have it defined in the .bashrc in my home directory so I don't need
to run the export every time. 

## Example

Load config into static variable

```
static final Map<String, Object> config = Config.getInstance().getJsonMapConfig(JwtHelper.SECURITY_CONFIG);
```

Load config when it is used

```
ValidatorConfig config = (ValidatorConfig)Config.getInstance().getJsonObjectConfig(CONFIG_NAME, ValidatorConfig.class);
```

## Config Server

With every module or application has its own config file, it is hard to manage
these files for different version of framework, different deployment environment.

In order to make it easier, we have provided a [light-config-server](https://github.com/networknt/light-config-server) 
to manage config files in multiple git organizations. The default config files
are provided by networknt organization in github. For companies to use
the framework, they can have an overwritten default config repo for each version
of framework for each environment. Also, each individual application or service
can have its overwritten config files for a framework version on a specific env.

For more information about light-config-server please check [README.md](https://github.com/networknt/light-config-server)

[info]: /concern/info/
