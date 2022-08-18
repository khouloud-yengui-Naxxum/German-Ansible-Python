This project allows you to configure the servers to be able to run python projects, by preparing
virtual environments that contain installed packages following .txt files
containing the necessary packages with their versions for each environment.
this project contains 3 folders: venv/ , Ansible/ and .github/workflows/

-the venv/ folder: contains .txt files that contain the python packages that will be installed

-the Ansible/ folder: which contains two .yaml files which are the playbooks that will be
 executed when launching the pipeline which is located in the .github/workflows/ folder.


-the .github/workflows/ folder: contains a pipeline called install-venv.yaml which
will be launched at each commit or pull request on the main branch which contains 2 jobs
 which will be executed on a github runner: ubuntu-latest 

  * the first is "Validate": which checks if the repository under $GITHUB_WORKSPACE,
  so that your work can access it

  * the second is "Run playbook": which contains several steps
      -the first step: install python 3.9 and ansible in the runner to be able
      run ansible playbooks
      -the second step "Install python venv on new server": which allows to run the playbook Ansible/playbook.yaml 
      -the third step "Get changed files": which will return the files changed during the last commit which are
       under venv/ folder and which have .txt format using github actions "tj-actions/changed-files@v26.1".
      -the fourth step "Install the new python packages":which allows to run the playbook Ansible/playbook-changed-files.yaml 
       and it will only be executed when the files which are under the venv/ folder and  have the .txt format have been changed
Ansible/playbook.yaml : this playbook will be run on the localhost, it contains two tasks :
            -the first checks whether the folder /opt/python_venvs/ exists or not on the server with the ansible module "stat" which returns a
             boolean value that will be stored in the variable p 
            -the second task will only be executed if /opt/python_venvs/ does not exist so that means that we are running the playbook on a new server,
             this task access to the ../venv/ folder and iterate all files with .txt format , using the ansible module "Loop" and for
             each file under ../venv/ it installs the virtual python environment , where it installs all the packages mentionned in this file 
             using the ansible "shell" module.  
Ansible/playbook-changed-files.yaml: this playbook will be run on the localhost and which will take as an argument an environment variable that contains 
the changed files.this playbook contains:
         - a task which allows to iterate the environment variable with the "with_items" ansible module,
           to extract each file  separately into this variable and run a shell script on it, using the ansible "shell" module.
           This script allows you to check if there is a folder named like "/opt/python_venvs/{{name_of_the_changed_file}}/" on the server
           so this file was modified on the git repository , so the script will  activate this environment 
           and put in the Installed_package variable the packages already installed in this environment
           and with a loop it checks for each line in this file if the package exists or not in this variable
           if it does not exist, it installs it . At the end of the loop, this environment will be deactivated.
           Otherwise if the /opt/python_venvs/{{name_of_the_changed_file}}/ does not exist on the server. so that means , it is a new file which has
           been added to the git repository , so we install  a virtual environment with the same file name,  under  /opt/python_venvs/ and we
           install on it all the packages mentioned in this file.      
       
