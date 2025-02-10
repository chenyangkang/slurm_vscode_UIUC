## A brief instruction on how to access compute node on slurm using VScode (to run apps, including jupyter notebook) at UIUC illnois campus cluster (ICC)

Update on Feb 9, 2025:

The Campus Cluster team has updated the system which makes all the previous method invalid. I spent a lot of time figuring out how to connect to the compute node with my local VScode but failed. So I decided to use the `code-server` that runs on the compute node, and connect it with browser.

1. Add this to you local `~/.ssh/config` file:

```bash
Host ICC-login
  User yc85
  Hostname cc-login.campuscluster.illinois.edu
  ServerAliveInterval 30
  ServerAliveCountMax 1200
  ControlMaster auto
  ControlPersist 3600
  ControlPath ~/.ssh/%r@klone-login:%p
```

where yc85 should be your user name.


2. Generate Private-public keys pair if you haven't, and copy the public key to the cloud.
3. In terminal: `ssh ICC-login`, duo login.
4. On login node, Install `code-server`: `micromamba install code-server`. You can also use conda: `conda install code-server`.
5. On login node, make a file with name `code-server_no_password.sh`:

```bash
#!/bin/bash 
 
#SBATCH --job-name=vscode
#SBATCH --output=%x_%j_%N.log 
#SBATCH --time 00-12:00:00
#SBATCH --cpus-per-task 2
#SBATCH --mem 16G
#SBATCH --partition IllinoisComputes
 
source ~/.bashrc

micromamba activate base


# PORT=$(python -c 'import socket; s=socket.socket(); s.bind(("", 0)); print(s.getsockname()[1]); s.close()')
PORT=7077

echo "********************************************************************" 
echo "Starting code-server in Slurm"
echo "Environment information:" 
echo "Date:" $(date)
echo "Allocated node:" $(hostname)
echo "Node IP:" $(ip a | grep 128.146)
echo "Path:" $(pwd)
echo "Password to access VSCode:" $PASSWORD
echo "Listening on:" $PORT
echo "********************************************************************"
echo "to port forward connection:"
echo "ssh -t -t $(whoami)@cc-login.campuscluster.illinois.edu -L 7077:localhost:7077 ssh $(basename $(hostname) .campuscluster.illinois.edu) -L 7077:localhost:$PORT"
echo "http://127.0.0.1:7077"
echo "********************************************************************" 
echo "using jump: "
echo "ssh -t -t $(whoami)@cc-login.campuscluster.illinois.edu -J jump -L 7077:localhost:7077 ssh $(hostname) -L 7077:localhost:$PORT"
echo "********************************************************************"
echo "////////////////////"
echo "scancel $SLURM_JOB_ID "
echo "////////////////////"
echo "********************************************************************" 

code-server --bind-addr 127.0.0.1:$PORT --auth none --disable-telemetry
```

Notice that proxy jump seems to be forbidden on the new Campus Cluster system some how. So use port forward.

5. `sbatch code-server_no_password.sh`
6. Find out wich node this script is running on (assume it's node ccc0365).
7. Open a new terminal on you local machine, make a tunnel:
   
```bash
ssh -t -t  yc85@cc-login.campuscluster.illinois.edu -L 7077:localhost:7077 ssh ccc0365 -L 7077:127.0.0.1:7077
```

This will tunnel your local computer to the compute node ccc0365.

8. Open your local browser, connect to `http://127.0.0.1:7077/`

9. Add shortcut to `~/.zshrc` / `~/.bashrc`:
```bash
get_icc() {
    ssh -t -t yc85@cc-login.campuscluster.illinois.edu -L 7077:localhost:7077 ssh "$(ssh ICC-login "jobid=\$(sbatch code-server_no_password.sh | awk '{print \$NF}'); while true; do state=\$(squeue -j \$jobid -o '%T' --noheader); if [[ \$state == 'RUNNING' ]]; then squeue -j \$jobid -o '%N' --noheader; break; fi; sleep 5; done")" -L 7077:localhost:7077
}
```

Where 

1) ssh `ICC-login "..."` is to first ssh then excecute the code in quote.
2) `jobid=\$(sbatch code-server_no_password.sh | awk '{print \$NF}'); while true; do state=\$(squeue -j \$jobid -o '%T' --noheader); if [[ \$state == 'RUNNING' ]]; then squeue -j \$jobid -o '%N' --noheader; break; fi; sleep 5; done")` is to submit the code-server job and get the node name.
3) `ssh -t -t yc85@cc-login.campuscluster.illinois.edu -L 7077:localhost:7077 ssh $NODE_NAME -L 7077:localhost:7077` is finally making the tunnel.


-- Happy coding --

**The followings are deprecated for Illinois Campus Cluster:** 

------

This repo is a note for how to setup compute node access on slurm and access with VSCode to run apps like jupyter notebook. Tested on UIUC Illnois Campus Cluster (ICC).

There are apparently many different ways to access the jupyter notebook on slurm:

1. run jupyter notebook on login node
2. Run notebook on compute node (persistently), and access it each time.
3. Each time ssh to the compute node, and start a jupyter server.

We can't run jupyter on slurm login node, especially in systems like UIUC ICC where the job will be killed after 30 CPU-minutes. So we rule out the fist method.

The following method will be based on running notebook on compute node, and this is only the one best way I figured among the others. This method is only tested on MAC OS.

The most important reference is from: https://docs.chpc.wustl.edu/software/visual-studio-code.html.

## 1. Generate SSH key pairs on your local machine

- 1.1 Open the terminal of your local machine, run `ssh-keygen`. Press enter for all default setting. A pair of keys will be generated as ~/.ssh/id_rsa (private key) and ~/.ssh/id_rsa.pub (public key). If you already have a key pairs, just use it and no need to regenerate.

- 1.2 Copy the public key to the login node of the cluster server. 

```sh
ssh-copy-id yc85@cc-login.campuscluster.illinois.edu
```

where yc85 is your user name.

## 2. Setup VScode node:

- 2.1 VS Code has a little bit of trouble with Slurm's module system. Append the following to your ~/.bashrc on the cluster:

```sh
export MODULEPATH=$MODULEPATH:/opt/modulefiles
```

- 2.2 Modify ssh config file on your local machine:

Add these to your `~/.ssh/config` file on your local machine

```
Host ICC-compute
  Hostname 127.0.0.1 
  User yc85
  IdentityFile ~/.ssh/id_rsa
  Port 8781

```

Port `8781` is not special, so you can change it. Don't change the `127.0.0.1`. The `~/.ssh/id_rsa` corresponds to your private key.


- 2.3 Configure the cluster for X11 forwarding

![](./fig1.png)

So do these on your cluster login node:

```
[clusteruser@login02 ~] echo "export XDG_RUNTIME_DIR=$HOME/.runtime-dir" >> ~/.bashrc
[clusteruser@login02 ~] mkdir -p ~/.runtime-dir
[clusteruser@login02 ~] chown $USER ~/.runtime-dir
[clusteruser@login02 ~] chmod 700 ~/.runtime-dir
```



## 3. Login

- 3.1 The first step of the login is to make a tunnel from your local machine, though the cluster login node, to the compute node.

Open a terminal on your local machine, run:

```sh
ssh -L 8781:ccc0335:22 yc85@cc-login.campuscluster.illinois.edu
```
This setup forwards any connections made to localhost:8781 on your local machine to port 22 on the ccc0335 host (SSH port on the computation node).

This should login to your login node on the cluster.

- 3.2 Submit a srun job

To allocate resources, submit a srun job:

```sh
srun -w ccc0335 --partition=aces --nodes=1 --time=29-00:00:00 --mem=50GB --cpus-per-task=5 --pty bash
```

For convenience, I made an alias of this command and add it to my `~/.bashrc` file on the login node:

```sh
alias get_compute_resources_ccc0335='srun -w ccc0335 --partition=aces --nodes=1 --time=29-00:00:00 --mem=50GB --cpus-per-task=5 --pty bash'
```

So that each time I only need to call `get_compute_resources_ccc0335`.

Notice that 3.1 and 3.2 are consecutive, so you should be able to finish these two steps in the same terminal.

If you close this terminal, then everything, including the tunnel and the srun job will be terminated. So you will keep it open throughout the coding today, shut down when you finish today's work (then everything automatically stop), and redo this tomorrow.

- 3.3 Open your VScode, on the left lower corner you should have a link icon to click on, if you have the ssh extension installed. Click - connect to host - ICC-compute. This should open a new window of VScode for you. Open a vscode terminal inside this window. If it shows `(base) [yc85@ccc0335 ~]$ `, then you are on the compute node! Now you can do the things you want, like open a jupyter notebook.


## 4. It's a new day

It's a new day! To start working on vscode + slurm + jupyter, you:

1. Open your local terminal, run `ssh -L 8781:ccc0335:22 yc85@cc-login.campuscluster.illinois.edu`
2. You will login to the login node of cluster. Run `get_compute_resources_ccc0335`.
3. Open your VScode, connet to host `ICC-compute`.
4. Start working!

-------

- Other tips
Becuase certain nodes are not always available, we can use a two-step method to let slurm decide which node is available, then mannually query the resources of that node.

Some useful lines to add to the ~/.bashrc:

```
get_aces() {
    echo "Hello, $1!"
    srun -w $1 --partition=aces --nodes=1 --time=02-23:59:00 --mem=40GB --cpus-per-task=2 --pty bash
}

get_ic() {
    echo "Hello, $1!"
    srun -w $1 --account=yc85-ic --partition=IllinoisComputes --nodes=1 --time=00-23:59:00 --mem=60GB --cpus-per-task=2 --pty bash
}

test_ic() {
    srun --account=yc85-ic --partition=IllinoisComputes --nodes=1 --time=00-23:59:00 --mem=60GB --cpus-per-task=2 --pty bash
}
```

So run test_ic() first, see which node is available. Then close the terminal and open a new one. Then `ssh -L 8781:THE_AVAILABLE_NODE:22 yc85@cc-login.campuscluster.illinois.edu`, then `get_ic THE_AVAILABLE_NODE`.


