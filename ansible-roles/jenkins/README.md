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

## Deploy Jenkins by using java and tomcat roles
  * First, create the jenkins role inside the `Ansible/ansible-roles` directory by running the command `ansible-galaxy role init jenkins`, which will automatically generate the default directory structure and files.
  * Update defaults/main.yml with the top-level variables required for the deployment.
    ```yml
    ---
    # defaults file for jenkins
    # LTS Source URL: https://get.jenkins.io/war-stable/2.555.3/jenkins.war

    java_version: 21
    jenkins_version: 2.555.3
    webapps_path: /opt/tomcat/webapps
    ```
    * We want to deploy Jenkins to the Tomcat application directory (`/opt/tomcat/webapps`) because Tomcat can host and manage multiple web applications simultaneously.
  * Next, before deploying Jenkins, define the role dependencies inside `meta/main.yml`. We must include both the **geerlingguy.java** and **tomcat10** roles because Java must be installed first, followed by Tomcat (which depends on Java), and finally Jenkins, which relies on Tomcat to host and manage its deployment.
  ```yml
  galaxy_info:
    author: jagadeesh
    description: ansible role for jenkins
    company: your company (optional)
    license: license (GPL-2.0-or-later, MIT, etc)

    min_ansible_version: "2.2"  
    galaxy_tags: []  
  dependencies:
    - role: geerlingguy.java
      vars: 
        java_packages:
          - "{{ 'java-' ~ java_version ~ '-openjdk-devel' if ansible_os_family == 'RedHat' else 'openjdk-' ~ java_version ~ '-jdk' }}"
    - role: tomcat10     
  ```
    * **Note**:
      * Here we are parameterize the version number using a new variable (e.g., `java_version`) and the Jinja2 string concatenation operator (`~`)
      * `java_version`: Added as a standalone variable so you can easily change it to 11, 17, etc.
      * `~` Operator: Used to dynamically glue the version number into the package names depending on the OS family.
  * Next, add a task to download the Jenkins WAR file. Note that we do not need to use the unarchive module, as Tomcat deploys .war files directly. To ensure the application loads, trigger a systemd daemon reload and restart Tomcat using a handler attached via a notify statement on the download task.
    ```yml
    ---
    # tasks file for jenkins
    - name: Download jenkins war file
      ansible.builtin.get_url:
        url: "https://get.jenkins.io/war-stable/{{ jenkins_version }}/jenkins.war"
        dest: "{{ webapps_path }}"
        owner: tomcat
        group: tomcat
        mode: '0644'
      notify: Reload and Restart systemd daemon
    ```
  * define handlers in `handlers/main.yml`
    ```yml
    ---
    # handlers file for jenkins
    - name: Reload and Restart systemd daemon
      ansible.builtin.systemd:
        name: tomcat
        state: restarted   
        enabled: yes      
        daemon_reload: true
    ```
  * If you want to deploy Jenkins to specific worker nodes, you can define them in your inventory file. When running the playbook, pass this file using the `-i jenkins/tests/inventory` flag. Alternatively, you can reuse the inventory file previously configured for the tomcat10 role; the key requirement is ensuring that the file path is specified correctly.
    ```ini
    [redhat_nodes]
    172.31.30.244

    [all:vars]
    ansible_user=ansible
    ansible_ssh_private_key_file=~/.ssh/ansible_key
    ```
  * Now, configure the deploy_jenkins.yml playbook file. Because the manager-app and host-manager credentials required to run Tomcat are stored securely in Ansible Vault, you must specify the path to these encrypted variables using the vars_files directive.
    ```yml
    ---
    - name: Deploy Jenkins using Apache Tomcat Server on Linux (Redhat/Ubuntu) Nodes
      hosts: redhat_nodes
      gather_facts: true
      become: yes
      vars_files:
        - tomcat10/vars/tomcat_manager.yml  

      roles:
        - role: jenkins
    ```
  * Finally, run the ansible-playbook command with the `--ask-vault-pass` flag to deploy Jenkins i.e `ansible-playbook -i jenkins/tests/inventory deploy_jenkins.yml --ask-vault-pass`. When prompted, enter the vault password `tomcat@123` to trigger the playbook execution. 
  * Next, open your web browser and navigate to `http://<YOUR_SERVER_PUBLIC_IP>:8080/jenkins`. When the Jenkins setup page prompts you for the Administrator password, log into your worker node via SSH, retrieve the password string from the path displayed on the screen, and paste it into the UI to unlock Jenkins. ![preview](/Images/tomcat27.png) 
  * After passing the administrator password, select install suggested plugins, enter your required user details, and click Save and Continue. ![preview](/Images/tomcat28.png)
  * Finally, under the **Instance Configuration** section, provide the URL `http://<YOUR_SERVER_PUBLIC_IP>:8080/jenkins/` and proceed to access the **Jenkins Home Page**. ![preview](/Images/tomcat29.png)