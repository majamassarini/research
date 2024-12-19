---
title: Global configuration
authors:
  - mmassari
sidebar_position: 1
---

# Global configuration styles

I will analyze 3 approaches for managing configuration relationships:

- templating
- global configuration + overlays
- inheritance

# 1. Templating

## https://github.com/packit/templeates/config/simple-pull-from-upstream.yaml.j2

```yml
# Packit pull-from-upstream config
specfile_path: { { specfile_path } }

upstream_package_name: { { upstream_package_name } }
downstream_package_name: { { downstream_package_name } }
upstream_project_url: { { upstream_package_url } }
upstream_tag_template: v{version}

jobs:
  - job: pull_from_upstream
    trigger: release
    dist_git_branches:
      - fedora-rawhide
  - job: koji_build
    trigger: commit
    allowed_pr_authors: ["packit", { { allowed_pr_authors } }]
    dist_git_branches:
      - fedora-rawhide
```

## https://gitlab.gnome.org/packit/templates/configs/gnome-tests.yaml.j2

```yml
# Gnome default tests config
jobs:
  - job: tests
    trigger: pull_request
    packages: [{{ downstream_package_name }}]
    tmt_plan: "smoke|full|packit-integration|{{ tmt_other_plans }}"
    targets:
      - fedora-rawhide
  {% if tests_on_commit %}
  - job: tests
    trigger: commit
    packages: [{{ downstream_package_name }}]
    tmt_plan: "smoke|full|packit-integration|{{ tmt_other_plans }}"
    targets:
      - fedora-rawhide
  {% endif %}
```

## https://gitlab.gnome.org/package/packit.yaml

```yml
# A gnome package packit config
templates:
  - https://github.com/packit/templeates/config/simple-pull-from-upstream.yaml.j2
    vars:
      specfile_path: specfile_path
      upstream_package_name: upstream_package_name
      downstream_package_name: downstream_package_name
      upstream_project_url: upstream_project_url
      allowed_pr_authors: allowed_pr_authors
  - https://gitlab.gnome.org/packit/templates/configs/gnome-tests.yaml.j2
      downstream_package_name: downstream_package_name
      tmt_other_plans: tmt_other_plans
      tests_on_commit: false
```

# 2. Global config + overlay

## https://github.com/packit/templates/configs/standard-pull-from-upstream.yaml.j2

```yml
# Packit pull-from-upstream config
specfile_path: { { specfile_path } }

upstream_package_name: { { upstream_package_name } }
downstream_package_name: { { downstream_package_name } }
upstream_project_url: { { upstream_package_url } }
upstream_tag_template: v{version}

jobs:
  - job: pull_from_upstream
    trigger: release
    dist_git_branches:
      - fedora-rawhide
  - job: koji_build
    trigger: commit
    allowed_pr_authors: ["packit", { { allowed_pr_authors } }]
    dist_git_branches:
      - fedora-rawhide
```

## https://gitlab.gnome.org/packit/templates/configs/default_packit.yaml.j2

```yml
# Gnome default packit config
config:
  base: https://github.com/packit/templates/configs/standard-pull-from-upstream.yaml.j2
  values:
    allowed_pr_authors: gnome-admins

jobs:
  - job: tests
    trigger: pull_request
    packages: [{{ downstream_package_name }}]
    tmt_plan: "smoke|full|packit-integration|{{ tmt_other_plans }}"
    targets:
      - fedora-rawhide
  {% if tests_on_commit %}
  - job: tests
    trigger: commit
    packages: [{{ downstream_package_name }}]
    tmt_plan: "smoke|full|packit-integration|{{ tmt_other_plans }}"
    targets:
      - fedora-rawhide
  {% endif %}
```

## https://gitlab.gnome.org/package/packit.yaml

```yml
# A gnome package packit config
config:
  base: https://gitlab.gnome.org/packit/templates/configs/default_packit.yaml.j2
  values:
    specfile_path: specfile_path
    upstream_package_name: upstream_package_name
    downstream_package_name: downstream_package_name
    upstream_project_url: upstream_package_url
    tmt_other_plans: package-tests
    tests_on_commit: false
```

# 3. Inheritance

## https://github.com/packit/templates/configs/standard-pull-from-upstream.yaml

```yml
# Packit pull-from-upstream config
specfile_path: -OVERRIDE ME-

upstream_package_name: -OVERRIDE ME-
downstream_package_name: -OVERRIDE ME-
upstream_project_url: -OVERRIDE ME-

upstream_tag_template: v{version}

jobs:
  - job: pull_from_upstream
    trigger: release
    dist_git_branches:
      - fedora-rawhide
  - job: koji_build
    trigger: commit
    allowed_pr_authors: ["packit"]
    dist_git_branches:
      - fedora-rawhide
```

## https://gitlab.gnome.org/packit/templates/configs/default_packit.yaml

```yml
# Gnome default packit config
inherit: https://github.com/packit/templates/configs/standard-pull-from-upstream.yaml

jobs:
  - job: koji_build
    allowed_pr_authors: ["packit", "gnome-admins"]

  - job: tests
    trigger: pull_request
    tmt_plan: "smoke|full|packit-integration"
    targets:
      - fedora-rawhide
```

## https://gitlab.gnome.org/package/packit.yaml

```yml
# A gnome package packit config
inherit: https://gitlab.gnome.org/packit/templates/configs/default_packit.yaml

specfile_path: specfile_path
upstream_package_name: upstream_package_name
downstream_package_name: downstream_package_name
upstream_project_url: upstream_package_url

jobs:
  - job: tests
    packages: ["downstream_package_name"]
    tmt_plan: "smoke|full|packit-integration|package-tests"
```

# PROs and CONs

## Templating

### Pros

Flexible, probably the most flexible implementation that allows to freely mix configuration snippets for creating a customized final configuration.

### Cons

The "pure" templating mechanism, in the above example, requires the package maintainer to know that koji builds, in the gnome ecosystem, should be allowed for any **gnome-admin**, instead _inheritance_ and _global config + overlays_ encapsulate well the knowledge in the middle layer packit config.

Templating is flexible but on the other end it is more error prone; there is no _base configuration_ and a packager can list templates in the wrong order.

Probably, in the end, the packager will use smaller config snippets, decreasing performance and readability.

## Global config + overlays

### Pros

Good knowledge encapsulation in middle layers (see the **gnome-admin** for allowed_pr_authors in the above example).

Explicit and thus easily readable, since the use of templating.

Flexible, config snippets can easily be removed using template conditional functionalities (as in the above example for the test job with trigger commit).

### Cons

The templating syntax can be more error prone if compared with inheritance.

## Inheritance

### Pros

Concise, it's the most concise syntax we could use and probably the least error prone.

### Cons

Poor flexibility, I don't see an easy way to disable the above test job with trigger commit.

Not really explicit, even though we use a placeholder it is harder to recognize the keys that need overriding.

# Implementation

Personally I find the _global config + overlays_ approach the best and in this case we would need to:

- add the following keys to the `PackageConfig` class:

```yml
config:
  base: https://gitlab.gnome.org/packit/templates/configs/default_packit.yaml.j2
  values: ...
```

- load the packit.yaml file and search for the `config` key in it. If a `config` key is found we need to **recursively** look for the _base config_ and start processing all the templates in the chain, creating a new temporary packit.yml that will be used instead of the original one.
  I see this code tied with the `LocalProject` class but I can be wrong.
  We should make the new code work both for the packit CLI and the packit-service. Thinking at packit CLI, we should probably stay flexible and let the `base: URI` be also a local url (like `file:///`).

- let the user know what the final configuration looks like (both for CLI and service).

## Jinja2 vs Ansible library

For template management I would probably just use the **jinja2 template library**, even though we can also think about the ansible library.
The ansible library could let us use `built-in filters and functions` but I don't see use cases for them and as a cons it has a heavier dependency footprint.

# Performances

Splitting the configuration in multiple configuration files will lead obviously to worst performance. Personally I don't see a way to prevent it.

We can limit the number of recursion steps; 3/4 steps are, from my point of view, more than enough. Having a recursion limit will avoid an infinite recursion for malformed configurations.

# packit-service defaults

It could happen that the packit-service config defaults for the "Fedora CI instance" and those for the "Usual instance" (as an example) diverge.

If it happens we could create two _hidden_, _inner_ packit config bases, one per instance, which will always be used for merging any packit config we process; in this way the differences will be grouped explicitly in a single place and we could, probably, enable and disable jobs for one instance or the other (as an example `pull-from-upstream` should not appear in fedora ci packit configuration) just using templating.
