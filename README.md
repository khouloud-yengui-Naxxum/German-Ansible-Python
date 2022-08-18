This project allows you to configure the servers to be able to run python projects, by preparing
virtual environments that contain installed packages following .txt files
containing the necessary packages with their versions for each environment.
this project contains 3 folders: venv/ , Ansible/ and .github/workflows/ .

*   the venv/ folder: contains .txt files that contain the python packages that will be installed

*   the Ansible/ folder: which contains two .yaml files which are the playbooks that will be
 executed when launching the pipeline which is located in the .github/workflows/ folder.


*   the .github/workflows/ folder: contains a pipeline called install-venv.yaml which
will be launched at each commit or pull request on the main branch which contains 2 jobs
 which will be executed on a github runner ubuntu-latest :

  1. the first is "Validate": which checks if the repository under $GITHUB_WORKSPACE,
     so that your work can access it
```yaml
  validate:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest
    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@v3
      - name: Run ansible-lint
        # replace `main` with any valid ref, or tags like `v6`
        uses: ansible-community/ansible-lint-action@v6.0.2
        with:
          args: "ansible" # my ansible files in a folder
```
  2. the second is "Run playbook": which contains several steps
  
*   the first step: install python 3.9 and ansible in the runner to be able
      run ansible playbooks
```yaml
  - uses: actions/checkout@v3
      - name: Set up Python 3.9
        uses: actions/setup-python@v2
        with:
          python-version: 3.9
      - name: Install dependencies Including Ansible
        run: |
          python -m pip install --upgrade pip
          python3 -m pip install  ansible
```  
*   the second step "Install python venv on new server": which allows to run the playbook Ansible/playbook.yaml 
```yaml
      - name: Install python venv on new server
        run: |
          ansible-playbook  Ansible/playbook.yaml
```  
*   the third step "Get changed files": which will return the files changed during the last commit which are
       under venv/ folder and which have .txt format using github actions "tj-actions/changed-files@v26.1".
```yaml
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0  # OR "2" -> To retrieve the preceding commit.    
      - name: Get changed files
        id: changed-files
        uses: tj-actions/changed-files@v26.1
        with:
          files: venv/*.txt 
```   
*   the fourth step "Install the new python packages":which allows to run the playbook Ansible/playbook-changed-files.yaml 
       and it will only be executed when the files which are under the venv/ folder and  have the .txt format have been changed
```yaml
      - name: Install  the new python packages
        if: ${{ steps.changed-files.outputs.all_changed_files }}
        run: |
          echo "Changed files: $changed_files_var"
          ansible-playbook  Ansible/playbook-changed-files.yaml 
        env:
          changed_files_var: ${{ steps.changed-files.outputs.all_changed_files }}
```        
       
*   Ansible/playbook.yaml : this playbook will be run on the localhost, it contains two tasks :

  1. the first checks whether the folder /opt/python_venvs/ exists or not on the server with the ansible module "stat" which returns a boolean value that will be           stored in the variable p
```yaml
- name: "Ansible check /opt/python_venvs/ exists"
    stat:
      path: /opt/python_venvs/
    register: p
```   
  2. the second task will only be executed if /opt/python_venvs/ does not exist so that means that we are running the playbook on a new server,
        this task access to the ../venv/ folder and iterate all files with .txt format , using the ansible module "Loop" and for
        each file under ../venv/ it installs the virtual python environment , where it installs all the packages mentionned in this file 
```yaml
  - name: "create python virtual environment from the venv folder /opt/python_venvs/ doesn't exist  "
    shell: |
      python3 -m venv /opt/python_venvs/{{ item | basename }}/
      source "/opt/python_venvs/{{ item | basename }}/bin/activate"
      pip install -r "../venv/{{ item | basename }}.txt"
      deactivate
    args:
      executable: /bin/bash
      chdir: "../venv/"
    with_fileglob: "../venv/*.txt"
    loop_control:
      label: "{{ item | basename }}"
    when: p.stat.exists == False
```  
*   Ansible/playbook-changed-files.yaml: this playbook will be run on the localhost and which will take as an argument an environment variable that contains 
the changed files.this playbook contains:
    *   a task which allows to iterate the environment variable with the "with_items" ansible module,
        to extract each file  separately into this variable and run a shell script on it, using the ansible "shell" module.
        This script allows you to check if there is a folder named like "/opt/python_venvs/{{name_of_the_changed_file}}/" on the server
        so this file was modified on the git repository , so the script will  activate this environment 
        and put in the Installed_package variable the packages already installed in this environment
        and with a loop it checks for each line in this file if the package exists or not in this variable
        if it does not exist, it installs it . At the end of the loop, this environment will be deactivated.
        Otherwise if the /opt/python_venvs/{{name_of_the_changed_file}}/ does not exist on the server. so that means , it is a new file which has
        been added to the git repository , so we install  a virtual environment with the same file name,  under  /opt/python_venvs/ and we
        install on it all the packages mentioned in this file.   
```yaml
 name: Install the python venv
  hosts: localhost
  user: root
  vars:
    my_changed_files: "{{ lookup('env', 'changed_files_var') }}"

  tasks: 
  - name: "create python virtual environment from the  changed files under venv  "
    shell: |
      if  [ -d /opt/python_venvs/{{ item | basename }}/ ]; then 
         source "/opt/python_venvs/{{ item | basename }}/bin/activate"
         Installed_package= `pip freez`
         for line in $(cat ../venv/{{ item | basename }}.txt) ; do
            if ! grep $line $Installed_package ;then
                 pip install $line
            fi;
         done      
         deactivate
      else
          python3 -m venv /opt/python_venvs/{{ item | basename }}/
          source "/opt/python_venvs/{{ item | basename }}/bin/activate"
          pip install -r "../venv/{{ item | basename }}.txt"
          deactivate
      fi;
    args:
      executable: /bin/bash
    with_items: "{{ my_changed_files }}"
``` 
