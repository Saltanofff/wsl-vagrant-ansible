## Important notes for installation
0. Install Vagrant, Ansible within the WSL2. Install Virtualbox on Windows host. Remove Hyper-v as hypervisor from the Windows features.
1. Prepare shell environment, add as below:
    ```bash
    export PATH="/mnt/c/Program Files/Oracle/VirtualBox:$PATH"
    #allow vagrant in wsl access to windows features including Vbox 
    export VAGRANT_WSL_ENABLE_WINDOWS_ACCESS=1
    ```
2. Install the virtualbox_WSL2 Plugin: This vagrant plugin addresses compatibility issues between virtualBox and WSL2. 
`vagrant plugin install virtualbox_WSL2`
3. Add into the Vagrantfile the following line to prevent error if this file not in the `/mnt/c/...` path
    ```bash
    # action prevents Vagrant from attempting to sync the project directory, thereby bypassing the error
    config.vm.synced_folder ".", "/vagrant", disabled: true 
    ```
4. Add custom ssh public key dedicated for Ansible to use instead of the default insecure vagrant key (as in the `Vagrantfile`)
5. Ad-hoc commands:
- `vagrant up`
- `vagrant status`
- `vagrant suspend`
- `vagrant resume`
- `vagrant destroy`
- `vagrant ssh`
- `vagrant ssh-config`
- `vagrant global-status`
