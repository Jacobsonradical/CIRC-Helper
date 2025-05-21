# CIRC-Helper
The Center for Research Integrated Computing (CIRC) of SOME SCHOOL is so hard to use due to the significant lack of proper documentation and examples. **My man, no one has this much time participating your 3-hour long training session for weeks, especially people who really need it are PhDs, who are typically really busy. Just provide good documentation and example, so people can start working on it.**

To this end, I am creating this repostiroy to store necessary commands and tips for everyone to start using this. 

### Register Account:
Use this link: https://registration.circ.rochester.edu/account

### Two Ways to Use:
There are primarily two ways to use CIRC:
1. Using Terminal and SLURM job scheduler.

   Pro:
   - Transparent and flexible: you can work just like on any other remote server.
   - Great for running scripts in Python (I am not sure about C because you need C compiler and I don't know if they have it)
   - After creating virtual environment, you don't need spend anytime to connect it to Jupyter kernel
   - You have to set up your Terminal anyways, and it is a good pratice to upload/download files via *ssh*
  
   Con:
   - Bad experience if you never used server and worked in Terminal.
   - Learning curve for SLURM for anyone who never used computing cluster scheduler (meaning all of us).
   - Hard to use R without RStudio or R in JupyterLab
   - Have to write a script first and test it several times locally. 

2. Using JupyterLab.

   Pro:
   - Easy to use. No need to learn Terminal or SLURM.
   - Most people are used to working in Jupyter Notebook.
   - Easy for R. 
  
   Con:
   - You have to set up your Terminal anyways.
   - Kernel dead problem is serious when connect your own environment to JupyterLab kernel. CIRC allows really low flexibility, so it is also hard to debug it. 

<div align="center">
  
## Initial Setup Using Terminal
  
</div>

This is something you have to do because a virtual envinroment must be created. 

### How to connect
```bash
ssh YourNetIDHere@bluehive.circ.rochester.edu
```

### Python version
The default python version should be 3.6.x, which is really low. However, the system does have other versions. You can see them by running:
```bash
module avail python3
```
![image](https://github.com/user-attachments/assets/b805657f-8619-4dc3-89a3-1192d22bf5bf)

You can unload the current python and load a new python version with:
```bash
module unload python3
module load python3/3.12.4/b1
```

Python 3.12.4 is the only version I confirmed with CIRC that contains *ssl* library which you need for *pip*. You cannot install *ssl* because it requires **sudo**. 

### Python venv
Here is an example of creating an virtual environment for a project mainly using UMAP library. As you don't have **sudo**, packages like *pipx* and *virtualenv* can't be installed. This is why I am using the python-inherited virtual environment creation tool. 
```bash
python3 -m venv ~/myvenv/umap
```
Use the following line to activate your virtual environment before installing any python package. 
```bash
source ~/myvenv/umap/bin/activate
```

### Package Install
The followings are package I need to do the project using UMAP and HDBSCAN. You can feel free to install anything you want. 
```bash
pip install pyarrow
pip install polars
pip install pandas
pip install atomicwrites
pip install "matplotlib<3.10"
pip install "numpy >= 1.23"
pip install "scipy >= 1.3.1"
pip install "scikit-learn >= 1.6"
pip install "numba >= 0.51.2"
pip install "pynndescent >= 0.5"
pip install tqdm
pip install umap-learn
pip install hdbscan
```

### Two Storage Sections
You have two stroage spaces. The first one is in 
```bash
/home/YourNetIDHere
```
This only has 20 GB spaces, so I suggest not to upload anything here. This directory only contains your virtual environments.

The second one is in
```bash
/scratch/YourNetIDHere
```
Here, you have 200 GB spaces, so your scripts/data should be here. Note that this section is not backed up by CIRC, so after computing, download them immeidately. 


### How to upload
The good thing about CIRC is that they have *rsync*, so please also install *rsync* in your computer. To upload, you want to use the following scripts

If you only want to upload a file
```bash
rsync -avz --info=progress2 -e ssh local/file/path sli73@bluehive.circ.rochester.edu:/scratch/YourNetIDHere/
```

If you want to upload the whole folder
```bash
rsync -avz --info=progress2 -e ssh local/folder/directory sli73@bluehive.circ.rochester.edu:/scratch/YourNetIDHere/
```

If you want to upload all the things within a folder (but the folder is not moved entirely)
```bash
rsync -avz --info=progress2 -e ssh local/folder/directory/ sli73@bluehive.circ.rochester.edu:/scratch/YourNetIDHere/
```


### Best Way to Access
The best way to access this shit is using JupyterHub.
1. Go to: https://info.circ.rochester.edu/#Web_Applications/JupyterHub/
2. Click the "JupyterHub" link in the first sentence.
3. Log in.
4. You have a request window to start a session as follows. I will talk about it in detail later. But once you requested your session, it will spawn a JupyterLab, and then you can play around.
![image](https://github.com/user-attachments/assets/ad3bffcf-31c6-4f45-a109-3771d8ca412e)


### How to request session:
Okay here is the hardest part.
1. Go to this link: https://info.circ.rochester.edu/#BlueHive/Compute_Nodes/
2. You can see a bunch of these things: ![image](https://github.com/user-attachments/assets/142cd412-b250-46af-bb20-4f337abef652)
3. Click one, say, I click **gpu**:![image](https://github.com/user-attachments/assets/98acb67a-5439-449c-a319-6bd0e2eaea3d)

Yeah, as you can see, we don't have many nodes for GPUs. Let's take the **first row** as an example. It means, we have 12 nodes, and each node contains 24 cores, 62GB CPU RAM, 2 Tesla K20Xm GPUs (I don't even know why we provide this, this costs about 200 dollars nowadays). 

How this thing works is that the system assigns you **node**, not specific resources. Hence, simply request the node that is >= what you need. To request, in the request window, ignore everything else, directly put the following command in the **additional option**:
```bash
-p gpu -t 2-00:00:00 --nodelist=bhg[0012-0018,0020,0022,0024-0027]
```
It tells the system you want
1. **gpu partition** (so for example, if you want to **debug** partition, you do *-p debug*)
2. 2 hours of time
3. Any of the node in the node list. 

Another example: 
```bash
-p preempt -t 2-00:00:00 --nodelist=bhg0059
```

### How long can you own:
1. Go to the same link: https://info.circ.rochester.edu/#BlueHive/Compute_Nodes/
2. At the very above you can see this:
![image](https://github.com/user-attachments/assets/fcd89fea-2bfc-452b-a311-c74f3172a7a5)
3. It tells you the maximal amount of time you can use. For example, for GPU nodes, you have maximal 5 days. 
























### Connect the virtual envirnoment to JupyerLab
You have to do this or you cannot use this environment for your Jupyter session. 

First, ensure that your virtual envirnoment is runnning:
```bash
source ~/myvenv/umap/bin/activate
```

Then, install *ipykernel*
```bash
pip install ipykernel
```

Next, run the following line with your own modification.
```bash
python -m ipykernel install --user --name=umap --display-name="Python3.12.4-b1 (UMAP)"
```
You can customize the *--display-name* to whatever you like. However, keep the *--name* to the folder name before */bin/activate*. For example, if your activate virtual environment by the code *source /some/directory/to/hello/bin/activate*, then you should use *--name=hello*.

Now, go your JupyterLab, refresh your browser. You should see something like this. As you can see, the "myvenv" folder is there. 
![Pasted image](https://github.com/user-attachments/assets/5377785e-11b5-401e-ad1a-a406129618c4)

Next, click the "New" botton on the top right corner, scroll down, you should see your environment, like mine:
![Pasted image (2)](https://github.com/user-attachments/assets/d4c55ef3-e4ec-4d6f-9433-a0e1b17a3676)

*If you want to uninstall this setting from Jupyter*:
```bash
jupyter kernelspec uninstall umap
```
Change *umap* to anything you put for *--name*. For example, if you put *--name=hello*, then replace *umap* with *hello*.



