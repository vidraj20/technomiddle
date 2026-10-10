# technomiddle
mw

Ansible Ping-Pong Project
A beginner-friendly Ansible project to test connectivity between an Ubuntu WSL control node and a Rocky Linux 9.8 managed node.

Architecture
Control node: Ubuntu on Windows WSL
Managed node: Rocky Linux 9.8 running in Oracle VirtualBox
Automation tool: Ansible
Version control: Git and GitHub
Project Structure
technomiddle/
├── ping.yml
├── inventory.example.ini
├── .gitignore
└── README.md
The real inventory.ini file is kept locally and excluded from Git using .gitignore.

Run the Project
Create a local inventory.ini based on inventory.example.ini.
Configure SSH access to the Rocky Linux managed node.
Test connectivity:
   ansible rocky -i inventory.ini -m ansible.builtin.ping
Run the playbook:
   ansible-playbook -i inventory.ini ping.yml
Expected Result
rockyvm | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
What I Learned
Basic Ansible inventory configuration
YAML playbook creation
SSH connectivity between Linux systems
Ansible ping module
Git, GitHub, and .gitignore
