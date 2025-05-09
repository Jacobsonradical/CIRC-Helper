# UrCirc
The Center for Research Integrated Computing (CIRC) of UR is so fucking hard to use due to the significant lack of proper documentation and examples. **My man, no one has this much time participating your 3-hour long training session for weeks, especially people who really need it are PhDs, who are typically really busy. Just provide good documentation and example, so people can start working on it.**

To this end, I am creating this repostiroy to store necessary commands and tips for everyone to start using this shit. 

### Register Account:
Use this link: https://registration.circ.rochester.edu/account

### Best Way to Access
The best way to access this shit is using JupyterHub. I know, you can be really smart and using terminal, but JupyterHub also provides terminal. Anyways, just use JupyterHub. 
1. Go to: https://info.circ.rochester.edu/#Web_Applications/JupyterHub/
2. Click the "JupyterHub" link in the first sentence.
3. Log in.
4. You have a request window to start a session. I will talk about it in detail later. But once you requested your session, it will spawn a JupyterLab, and then you can fuck around.

### Python version
The default python version should be 3.6.x, which is really low. However, the system does have Python 3.11. To unload the current one and load a new one, use:

```bash
module unload python3
module load python3/3.11.0
```
