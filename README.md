# CIRC-Helper
The Center for Research Integrated Computing (CIRC) of SOME SCHOOL is so hard to use due to the significant lack of proper documentation and examples. **My man, no one has this much time participating your 3-hour long training session for weeks, especially people who really need it are PhDs, who are typically really busy. Just provide good documentation and example, so people can start working on it.**

To this end, I am creating this repostiroy to store necessary commands and tips for everyone to start using this shit. 

### Register Account:
Use this link: https://registration.circ.rochester.edu/account

### Best Way to Access
The best way to access this shit is using JupyterHub.
1. Go to: https://info.circ.rochester.edu/#Web_Applications/JupyterHub/
2. Click the "JupyterHub" link in the first sentence.
3. Log in.
4. You have a request window to start a session as follows. I will talk about it in detail later. But once you requested your session, it will spawn a JupyterLab, and then you can fuck around.
![image](https://github.com/user-attachments/assets/ad3bffcf-31c6-4f45-a109-3771d8ca412e)


### How to request session:
Okay here is the hardest part.
1. Go to this link: https://info.circ.rochester.edu/#BlueHive/Compute_Nodes/
2. You can see a bunch of these things: ![image](https://github.com/user-attachments/assets/142cd412-b250-46af-bb20-4f337abef652)
3. Click one, say, I click **gpu**:![image](https://github.com/user-attachments/assets/98acb67a-5439-449c-a319-6bd0e2eaea3d)

Yeah, as you can see, we don't have many nodes for GPUs. Let's take the **first row** as an example. It means, we have 12 nodes, and each node contains 24 cores, 62GB CPU RAM, 2 Tesla K20Xm GPUs (I don't even know why we provide this, this costs about 200 dollars nowadays). 

To request, in the request window, ignore everything else, directly put the following command in the **additional option**:

```bash
-p gpu -t 06:00:00 --constraint="E52695v2&K20X"
```
It tells the system you want
1. **gpu partition** (so for example, if you want to **debug** partition, you do *-p debug*)
2. 6 hours of time
3. The CPU + GPU combination stated in the last column of the firs row.

The system is going to then assign you into the node that has this GPU, which means one of the 13 nodes in the first row. Therefore, **specifying the constraints** is the key.  You are assigned to the node, and you own these RAMs/Cores even if you request less than what the node has. So requesting for RAM/Core is useless. 

For those resources that have only one node, you can simply **enforce** the node. For example, the third to the last row for 2 days:
```bash
-p gpu -t 2-00:00:00 --nodelist=bhg0050
```

### How long can you own:
1. Go to the same link: https://info.circ.rochester.edu/#BlueHive/Compute_Nodes/
2. At the very above you can see this:
![image](https://github.com/user-attachments/assets/fcd89fea-2bfc-452b-a311-c74f3172a7a5)
3. It tells you the maximal amount of time you can use. For example, for GPU nodes, you have maximal 5 days. 


<div align="center">
  
## Initial Setup Using Terminal
  
</div>

Okay, I understand that you may not like this, but we have to. JupyterHab also provides terminal, but it is really slow. 

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
```bash
python3 -m venv ~/myvenv/umap
```

```bash
source ~myvenv/umap/bin/activate
```


