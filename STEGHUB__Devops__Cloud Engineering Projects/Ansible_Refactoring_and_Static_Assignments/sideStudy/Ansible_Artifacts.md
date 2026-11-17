# Ansible Artifact Re-Use

## Introduction

Ansible allows automation content to be organized into reusable components instead of repeating the same tasks and configuration across multiple playbooks. This makes automation easier to maintain, helps keep implementations consistent, and allows common functionality to be shared between projects.

Ansible supports several forms of reuse. Roles group related tasks and supporting files into a standard directory structure. Task files can be included dynamically or imported statically. Variables and templates also help separate reusable configuration from the tasks that apply it.

This guide introduces these concepts and demonstrates how they can be used in playbooks. It follows the principles described in the [official Ansible documentation on reusing playbooks, roles, and other artifacts](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse.html).

## Concepts

### 1. Roles

A **role** is a way to organize related Ansible content into a reusable structure. A role can contain tasks, handlers, variables, default values, files, templates, and metadata. Instead of keeping all configuration and automation logic in one large playbook, related components can be grouped into a role and called from one or more playbooks.

A typical role directory may look like this:

```text
roles/
└── example_role/
    ├── tasks/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    ├── files/
    ├── templates/
    ├── vars/
    │   └── main.yml
    ├── defaults/
    │   └── main.yml
    ├── meta/
    │   └── main.yml
    └── tests/
        ├── inventory
        └── test.yml
```

Each directory has a particular purpose:

- `tasks/` contains the tasks that make up the role. Ansible normally starts with `tasks/main.yml`.
- `handlers/` contains handlers that can be notified by tasks, such as a service restart after a configuration change.
- `files/` stores static files that tasks can copy to managed hosts.
- `templates/` stores Jinja2 templates used to generate files from variables and other data.
- `vars/` contains role variables. These have a higher precedence than role defaults.
- `defaults/` contains default variable values. These have a lower variable precedence and are generally intended to be easy to override.
- `meta/` contains role metadata, including information such as role dependencies.
- `tests/` can contain files used to test the role. It is not required for every role.

The example shows a commonly used role layout; it does not mean every directory must be created. A role only needs the directories and files required by its implementation.

### 2. Includes and Imports

Ansible provides **dynamic includes** and **static imports** to reuse content stored outside the current playbook. Both approaches help divide larger automation tasks into smaller files, but they are processed differently.

With a dynamic include, Ansible loads the referenced content when execution reaches the include statement. With a static import, Ansible processes the referenced content while parsing the playbook, before task execution begins.

The directives depend on the type of content being reused:

| Content being reused | Dynamic reuse | Static reuse |
|---|---|---|
| Task files | `include_tasks` | `import_tasks` |
| Roles | `include_role` | `import_role` |
| Playbooks | Not supported | `import_playbook` |
| Variable files | `include_vars` | `vars_files` |

There is no `include_playbook` directive. An entire playbook can be brought into another playbook using `import_playbook`, which is written at the top level of the importing playbook rather than inside a play's `tasks` list.

#### How includes and imports differ

The main distinction is when Ansible processes the referenced content:

- **Dynamic include:** Ansible processes the included file or role when it encounters the include during execution. This makes it useful when the content to run depends on earlier task results or when the include needs to be used with a loop.
- **Static import:** Ansible processes the imported content before running the play. The imported tasks become part of the playbook's task list for execution and inspection.

This difference also affects task keywords. A condition placed on a dynamic include controls whether Ansible processes that include. A condition placed on a static import is applied to the imported tasks individually. Similarly, keywords such as tags can behave differently: keywords on an include generally apply to the include statement itself, while keywords on an import are inherited by the imported tasks.

Dynamic includes can be used with loops, causing the included tasks or role to run for each loop item. Static imports cannot be looped in the same way. Imported tasks are also visible to commands such as `--list-tasks`, while tasks inside a dynamic include are not expanded in the same way during that pre-execution listing.

Both mechanisms are useful. Choose the one that matches the execution behavior required by the playbook, and be deliberate when mixing them because their different processing rules can make execution harder to follow.

### 3. Variables

Variables allow values to be defined once and reused in different parts of an Ansible project. They make playbooks more flexible because a value can be changed without editing every task that uses it.

Variables can be defined in several places, including inventory files, playbooks, roles, and external variable files. The appropriate location depends on how widely the value should be used and whether it needs to be overridden for particular hosts or groups.

Ansible applies variable-precedence rules when the same variable is defined in more than one place. For this reason, variable placement should be intentional. Role defaults are commonly used for values that users may need to override, while role variables are generally used for values that should be less easily overridden.

### 4. Templates

Ansible templates use the Jinja2 templating engine to create or configure files dynamically. A template contains ordinary file content together with expressions or variables that Ansible replaces when it renders the template on behalf of a managed host.

Templates are useful when the same configuration structure must be deployed with different values. Rather than maintaining a separate configuration file for every host, a single template can be reused and populated with the appropriate variables.

### 5. Dynamic Includes

Dynamic includes allow Ansible to load task files, roles, or variable files while a playbook is running. Because the content is loaded when Ansible reaches the include statement, the decision to include it can depend on conditions or information gathered by earlier tasks.

For task files, `include_tasks` is the directive used for dynamic reuse. For roles, use `include_role`; for variable files, use `include_vars`.

## Implementing Reuse in Ansible

### Using Roles

#### 1. Creating a Role

The `ansible-galaxy` command can generate the standard directory structure for a new role. The following command creates a role named `example_role`:

```bash
ansible-galaxy init example_role
```

The generated structure provides a starting point. Directories that are not needed by the role can be left unused or removed.

#### 2. Using a Role in a Playbook

Once the role is available in a location Ansible searches for roles, it can be called from a play using the `roles` keyword:

```yaml
- hosts: webservers
  roles:
    - example_role
```

In this example, Ansible applies `example_role` to the hosts in the `webservers` inventory group. When a role is listed under `roles`, Ansible processes it as static role reuse. The role's standard entry-point files, such as `tasks/main.yml`, are loaded according to the role structure.

### Using Includes and Imports

Task files can be reused either dynamically or statically. The choice affects when Ansible loads the file and how task keywords such as conditions, loops, and tags are applied.

#### 1. Including a Task File Dynamically

Use `include_tasks` when the task file should be loaded dynamically during playbook execution:

```yaml
- include_tasks: tasks/example_tasks.yml
```

Ansible processes `tasks/example_tasks.yml` when execution reaches this statement. This approach is useful when the decision to include the file depends on runtime conditions or when the include needs to be used with a loop.

#### 2. Importing a Task File Statically

Use `import_tasks` when the task file should be incorporated into the playbook before task execution:

```yaml
- import_tasks: tasks/example_tasks.yml
```

The tasks in `tasks/example_tasks.yml` are processed as part of the playbook during parsing. Static imports are useful when the task sequence is known in advance and you want the imported tasks to be visible during playbook inspection.

#### 3. Including a Role Dynamically

A role can also be included from within a play's `tasks` section. This is useful when the role should run at a particular point in the task sequence or when its execution depends on a runtime condition:

```yaml
- include_role:
    name: example_role
```

Unlike a role listed under the play-level `roles` keyword, a role included with `include_role` runs when Ansible reaches that task in the play.

### Using Variables

#### 1. Defining Variables in a Playbook

Variables can be declared directly in a play using the `vars` keyword:

```yaml
- hosts: webservers
  vars:
    http_port: 80
```

Here, `http_port` is defined with a value of `80`. Tasks in the play can reference this variable wherever the web server port is required.

#### 2. Using External Variable Files

Variables can also be kept in a separate file and loaded into a play with `vars_files`:

```yaml
- hosts: webservers
  vars_files:
    - vars/external_vars.yml
```

This separates variable definitions from the playbook itself. It can make the playbook easier to read and allows the same variable file to be referenced by other playbooks where appropriate.

### Using Templates

#### 1. Creating a Template

Create a Jinja2 template file, for example `templates/example_template.j2`:

```jinja2
server {
    listen {{ http_port }};
    server_name {{ server_name }};
}
```

The expressions `{{ http_port }}` and `{{ server_name }}` are placeholders. When Ansible renders the template, it substitutes them with the values of the corresponding variables.

#### 2. Using a Template in a Playbook

The `template` module transfers a rendered template to a destination on the managed host:

```yaml
- name: Apply web server configuration
  template:
    src: templates/example_template.j2
    dest: /etc/nginx/sites-available/example
```

In this example, Ansible renders `templates/example_template.j2` and places the resulting configuration at `/etc/nginx/sites-available/example`.

The destination shown is commonly used by Debian- and Ubuntu-style Nginx configurations. It is not a universal Nginx configuration path. The file must also be included by the active Nginx configuration before Nginx will use it. On other Linux distributions, the appropriate configuration location may be different.

### Using Dynamic Includes

#### 1. Conditionally Including a Task File

A task file can be included only when a condition is met. For example:

```yaml
- name: Include tasks when the condition is true
  include_tasks: tasks/example_tasks.yml
  when: condition
```

In this example, Ansible includes `tasks/example_tasks.yml` only when `condition` evaluates to true. The variable should be defined as a Boolean value, rather than as the text string `"true"` or `"false"`.

Because this is a dynamic include, the condition controls whether the include is processed. It is not automatically applied as a separate condition to every task in the included file.

## Best Practices

1. **Keep playbooks modular.** When a playbook becomes difficult to read or maintain, move related tasks into reusable task files or roles. This keeps individual components focused and makes them easier to use elsewhere.

2. **Place variables deliberately.** Define variables at a level that reflects how they will be used. Play-level variables are useful for values shared by tasks in a play, while external variable files help separate configuration data from automation logic. Be aware of Ansible's variable-precedence rules when a value is defined in multiple places.

3. **Reuse templates for repeated configuration.** When several hosts or environments need the same configuration structure with different values, use a Jinja2 template rather than maintaining multiple nearly identical files.

4. **Document reusable components.** Explain the purpose of each role, task file, variable file, and template. Clear documentation helps other users understand what a component does, what variables it expects, and where it can be reused.

5. **Choose includes and imports intentionally.** Use dynamic includes when runtime decisions or loops are required. Use static imports when the content is known before execution and should be incorporated into the playbook during parsing. Understand how conditions, tags, and task listings behave for each method.

## Conclusion

Reusing artifacts is an important part of building maintainable Ansible automation. Roles provide a structured way to group related automation content, while task includes and imports allow task files to be reused dynamically or statically. Variables make playbooks adaptable, and templates make it possible to generate configuration files from reusable definitions.

Using these features appropriately reduces duplicated work and makes automation easier to understand, extend, and maintain. The most suitable reuse method depends on the content being shared and on how and when that content needs to be processed.
