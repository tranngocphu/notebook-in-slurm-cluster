# Running Jupyter Notebook in SLURM cluster @Brandeis

The following steps require that your local computer is connected to campus network. Otherwise, first connect to university VPN if you are working remotely.

## Steps:

1. Make sure you have **your own** Python virtual environment running. The Anaconda **base** environment pre-installed in the login node is read-only, which means installation of new packages is not permitted.

2. Activate the virtual environment that you want to use with Jupyter Notebook (e.g. **testenv** in this tutorial). 

3. Verify that the environment comes with **pip**:

        (testenv) [<your_account>@hpcc ~]$ which pip

    This should indicate the path to **pip** command, which must reside in the environment directory.

4. Then install **jupyter lab** by running the following command:
        
        (testenv)[<your_account>@hpcc ~]$ pip install jupyterlab

    Note that **jupyter lab** installation comes with **jupyter notebook** too.
    
5. Next, install **jupyter_nbextensions_configurator**:
        
        (testenv) [<your_account>@hpcc ~]$ pip install jupyter_nbextensions_configurator

6. Now, you are going to generate a password to access your notebook remotely, instead of using authorization token. Run:

        (testenv) [<your_account>@hpcc ~]$ jupyter notebook password

    If password is successfully recored, you will see this:

        [NotebookPasswordApp] Wrote hashed password to /home/<your_account>/.jupyter/jupyter_notebook_config.json

7. Now we are ready to initialize a **Notebook Server** in SLURM cluster, and connect to it remotely using a browser. We are not doing this in the Login Node. We must submit a SLURM job using SBATCH to start the Notebook Server in a node that meets your resource requirements. For example, I need to use GPU in the notebook, my batch script should look like

        #!/bin/bash
        #SBATCH --job-name=notebook
        #SBATCH --account=hagan-lab
        #SBATCH --partition=hagan-gpu
        #SBATCH --time=08:00:00
        #SBATCH --gres=gpu:TitanX:1
        #SBATCH --output=./notebook.out
        #SBATCH --error=./notebook.err

        module load cuda/10.2
        source ~/miniconda3/bin/activate
        conda activate testenv
        jupyter notebook ~/code --ip=0.0.0.0 --no-browser

    Explain:
    
    * ```module load cuda/10.2``` is required for PyTorch or TensorFlow to recognize GPU
    * ```source ~/miniconda3/bin/activate``` activate the conda using miniconda3. Note that in this case **miniconda3** was installed by myself. So that I don't have to rely on the pre-installed module ANACONDA/5.3_py3 in the HPC. If you are using the pre-installed module ANACONDA/5.3_py3, then you should load the module at this step by including:

            module load shared_modules/ANACONDA/5.3_py3
    * then ```conda activate testenv``` activates your environment 
    * ```jupyter notebook ~/code --ip=0.0.0.0 --no-browser``` is the actual command to init the Notebook Server. Here, the notebook server's root directory will be my **code** folder in the **home** directory. ```--ip=0.0.0.0``` is to bind the notebook URL address to the node you are submitting this job to. ```--no-browser``` is to suppress the browser because in the server there is no web browser!
    * Note that this SLURM batch job will output error and information to the 2 files ```/notebook.out``` and ```/notebook.err```. If the notebook server is up, **after a while** you will find the following contents in the ERROR file (but not neccessarily error!):

            [I 2022-03-11 16:48:20.578 ServerApp] jupyterlab | extension was successfully linked.
            [W 2022-03-11 16:48:20.600 NotebookApp] 'password' has moved from NotebookApp to ServerApp. This config will be passed to ServerApp. Be sure to update your config before our next release.
            [W 2022-03-11 16:48:20.600 NotebookApp] 'password' has moved from NotebookApp to ServerApp. This config will be passed to ServerApp. Be sure to update your config before our next release.
            [I 2022-03-11 16:48:20.605 ServerApp] nbclassic | extension was successfully linked.
            [I 2022-03-11 16:48:21.592 ServerApp] jupyter_nbextensions_configurator | extension was found and enabled by notebook_shim. Consider moving the extension to Jupyter Server's extension paths.
            [I 2022-03-11 16:48:21.592 ServerApp] jupyter_nbextensions_configurator | extension was successfully linked.
            [I 2022-03-11 16:48:21.592 ServerApp] notebook_shim | extension was successfully linked.
            [I 2022-03-11 16:48:21.889 ServerApp] notebook_shim | extension was successfully loaded.
            [W 2022-03-11 16:48:21.913 ServerApp] jupyter_nbextensions_configurator | extension failed loading with message: 'nbextensions_path'
            [I 2022-03-11 16:48:21.914 LabApp] JupyterLab extension loaded from /home/phutran/miniconda3/envs/testenv/lib/python3.8/site-packages/jupyterlab
            [I 2022-03-11 16:48:21.914 LabApp] JupyterLab application directory is /home/phutran/miniconda3/envs/testenv/share/jupyter/lab
            [I 2022-03-11 16:48:21.917 ServerApp] jupyterlab | extension was successfully loaded.
            [I 2022-03-11 16:48:22.023 ServerApp] nbclassic | extension was successfully loaded.
            [I 2022-03-11 16:48:22.024 ServerApp] Serving notebooks from local directory: /home/phutran/mrsec/code
            [I 2022-03-11 16:48:22.024 ServerApp] Jupyter Server 1.13.5 is running at:
            [I 2022-03-11 16:48:22.024 ServerApp] http://gpu-6-13:8888/
            [I 2022-03-11 16:48:22.024 ServerApp]  or http://127.0.0.1:8888/
            [I 2022-03-11 16:48:22.024 ServerApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
    * Look at the bottom of the error file, you will see 2 URLs ```http://gpu-6-13:8888/``` and ```http://127.0.0.1:8888/```. You can ignore the 2nd URL (and you may NOT see the 2nd URL). In the 1st URL, take note of the <node_name> (e.g. **gpu-6-13** in this example) and the \<PORT\> (e.g. **8888** in this example) 
    * Now we need to establish a SSH tunnel to direct traffic from our local computer (your laptop or PC) to the Notebook Sever. This is done by running the following command in a terminal app or command prompt of **YOUR LOCAL COMPUTER**:

            ssh -N -L 8888:<node_name>:<PORT> <your_unet_account>@hpcc.brandeis.edu

        For example: ```ssh -N -L 8888:gpu-6-13:8888 phutran@hpcc.brandeis.edu```
        
        You will be promted for your UNET account password if you haven't copied a SSH key to the HPC. Enter your UNET password if neccessary. Note that in the above command, **the number ```8888``` before the <node_name> is the LOCAL PORT**, which can be any number between 1024 and 49151, depending on if your computer is running other apps that are using ports.

    * Finally, take note of LOCAL PORT, open a web browser in your local computer and access the URL below:

            http://localhost:<LOCAL PORT>, e.g. http://localhost:8888/ 

        You will be prompted for the Jupyter notebook password that you created at the beginning. Hope you are still remembering it :))


## HAPPY NOTEBOOKING!