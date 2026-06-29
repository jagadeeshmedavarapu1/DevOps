Role Name
=========

A brief description of the role goes here.

Requirements
------------

Any pre-requisites that may not be covered by Ansible itself or the role should be mentioned here. For instance, if the role uses the EC2 module, it may be a good idea to mention in this section that the boto package is required.

Role Variables
--------------

A description of the settable variables for this role should go here, including any variables that are in defaults/main.yml, vars/main.yml, and any variables that can/should be set via parameters to the role. Any variables that are read from other roles and/or the global scope (ie. hostvars, group vars, etc.) should be mentioned here as well.

Dependencies
------------

A list of other roles hosted on Galaxy should go here, plus any details in regards to parameters that may need to be set for other roles, or variables that are used from other roles.

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: servers
      roles:
         - { role: username.rolename, x: 42 }

License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).

## Installation of Spring Pet Clinic using java21 and tomcat10 roles

### Workflow Architecture Breakdown
  1. **System Provisioning**: Installing dependencies based on the host OS family.
  2. **Build Automation**: Cleaning past footprints, pulling source code, and compiling code.
  3. **Target Environment Structuring**: Organizing folders, extracting artifacts, and configuring bindings.
  4. **Daemon Control & Verification**: Generating systemd targets and validating live uptime.

#### System Provisioning Phase
  * **Install required packages on Ubuntu**
    * Ansible Module: `ansible.builtin.apt`
    * Purpose: Installs `git` and `maven` on Debian/Ubuntu targets.
    * Mechanism: Updates the local apt repository cache (`update_cache: yes`) before installation. Runs only if `ansible_os_family == "Debian"`
  * **Install required packages on RedHat**
    * Ansible Module: `ansible.builtin.dnf`
    * Purpose: Installs `git` and `maven` on RHEL, Rocky Linux, or CentOS targets.
    * Mechanism: Leverages the DNF package manager. Runs only if `ansible_os_family == "RedHat"`.

#### Build Automation Phase
  * **Remove old source directory**
    * Ansible Module: `ansible.builtin.file`
    * Purpose: Deletes the directory specified in `{{ build_dir }}` (`state: absent`).
    * Mechanism: Guarantees a clean pipeline state by purging stale artifacts or incomplete past builds.
  * **Clone Spring PetClinic repository**
    * Ansible Module: `ansible.builtin.git`
    * Purpose: Pulls the source code down to the target node.
    * Mechanism: Targets the `main` branch. The `force: yes` flag discards any uncommitted local changes on the managed node, ensuring a clean sync.
  * **Build Spring PetClinic**
    * Ansible Module: `ansible.builtin.shell`
    * Purpose: Compiles the source code into a runnable deployment package.
    * Mechanism: Navigates inside `{{ build_dir }}` and invokes `mvn clean package -DskipTests` to build the application binary while skipping time-consuming unit tests.

#### Target Environment Structuring Phase
  * **Create application directory**
    * Ansible Module: `ansible.builtin.file`
    * Purpose: Creates the production home directory `{{ app_home }}` if it does not exist.
    * Mechanism: Enforces standard security access controls (`0755`) and sets initial ownership to your specific runtime service user and group.
  * **Copy generated JAR**
    * Ansible Module: `ansible.builtin.shell`
    * Purpose: Moves the compiled executable block to its final production path.
    * Mechanism: Uses a `/bin/bash` shell wrapper to resolve the folder wildcard (`*.jar`) and safely renames the file to `spring-petclinic.jar`.
  * **Set ownership on application directory**
    * Ansible Module: `ansible.builtin.file`
    * Purpose: Standardises user and group execution permissions across all nested components.
    * Mechanism: Recursively tracks down (`recurse: yes`) every file inside `{{ app_home }}` and transfers ownership to the designated unprivileged app user (`{{ user_name }}`).
  * **Create application.properties**
    * Ansible Module: `ansible.builtin.copy`
    * Purpose: Generates the external file-based runtime settings configuration profile.
    * Mechanism: Writes out a custom `server.port` entry bound to your dynamic `{{ app_port }}` variable with non-executable user-read permissions (`0644`).

#### Daemon Control & Verification Phase
  * **Create Spring PetClinic systemd service**
    * Ansible Module: `ansible.builtin.template`
    * Purpose: Installs a unified Linux system daemon unit configuration file into `/etc/systemd/system/`.
    * Mechanism: Parses variables inside a Jinja2 template (`spring-petclinic.service.j2`). Signals the handler stack (`notify`) to reload systemd configurations and restart the application if any changes are made to the file.
  * **Flush handlers**
    * Ansible Module: `ansible.builtin.meta`
    * Purpose: Forces Ansible to run pending notification loops immediately instead of waiting until the end of the entire playbook run.
    * Mechanism: Triggers the `Reload systemd` and `Restart Spring PetClinic` handlers instantly so the service starts up before the next step runs.
  * **Wait for application to start**
    * Ansible Module: `ansible.builtin.wait_for`
    * Purpose: Pauses playbook execution to verify that the application has successfully started up.
    * Mechanism: Waits up to 120 seconds after an initial 10-second delay for the designated application network port (`{{ app_port }}`) to open and begin accepting active TCP traffic.
  
#### Execution & Deployment
  * Once your inventory, variables, and role dependencies are configured, trigger the automated deployment pipeline by running the following command from your playbook root directory i.e `ansible-playbook -i springPetClinic/tests/inventory deploy_springPetClinic.yml --ask-vault-pass`
    * **`--ask-vault-pass`**: Prompts you to securely enter your Ansible Vault password in the terminal. This is required to decrypt protected variables (such as sensitive system passwords or keys) on the fly during runtime (`tomcat@123`).
  * Once the playbook successfully finishes and the Wait for application to start task passes, your Spring PetClinic application will be fully live and accepting traffic.
  * Open your web browser and navigate to the following URL: `http://<public-ip>:8082`. ![preview](/Images/spring-petclinic1.png)