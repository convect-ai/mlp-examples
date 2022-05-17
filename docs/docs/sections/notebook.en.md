# Notebook server

## Start a server

To start a Jupyter Server, log into ConvectHub and click “Start server” if you do not have one, or “My server” if a server already exists. 

Once prompted, choose the specifications of the server that suits your need.

![Select server spec](Notebook%20s%20b4d80/Untitled.png)

Wait until the server is started, you will be automatically redirected to it. 

![Server starting](Notebook%20s%20b4d80/Untitled%201.png)

You can use the server as a normal [JupyterLab](https://jupyterlab.readthedocs.io/en/stable/) environment.

![Main environment](Notebook%20s%20b4d80/Untitled%202.png)


### GPU instances

To start a Jupyter Server with GPU, when prompted with the server specifications, choose the one with GPU. 

![GPU spec](Notebook%20s%20b4d80/select-gpu-server.png)

Once you are redirected into the server, launch a terminal app, execute `nvidia-smi` and you will see the GPU you are assigned to. 

![nvidia-smi](Notebook%20s%20b4d80/nvidia-smi.png)

You can use `conda` to install `cudatoolkit` and frameworks that support GPU like `pytorch` or `tensorflow` to start running workloads on GPU.


## Managing dependencies and environment

The managed Jupyter Server comes with pre-shipped python environments that include the most common used data science packages such as pandas, scikit-learn. To start using an environment as the Jupyter kernel, just choose it from the launcher. 

You can also change the kernel while editing a notebook by clicking on the upper right corner kernel indicator button and choose one from the dropdown menu.

![Select kernel](Notebook%20s%20b4d80/Untitled%203.png)

You can install additional packages into the environment if needed. To do so, launch a terminal app from the launcher and use `conda` to install packages. For example,

```bash
conda activate default
conda install pytorch torchvision torchaudio cpuonly -c pytorch
```

You can also create new conda environnement by 

```bash
conda create -n my-conda-env python==3.9
conda activate my-conda-env
# install more packages
conda install scikit-learn
```

Once created, the kernel will be available from the launcher automatically. To see more details on managing environments using conda, refer to [conda’s doc](https://docs.conda.io/en/latest/).

## Accessing remote data sources

To access remote data sources from your notebook server, you can use the same method when doing so from your local laptop. For example, to access data from a S3 source, run the following from your notebook

```python
import os
os.environ["AWS_ACCESS_KEY_ID"] = "YOUR-AWS-KEY-ID"
os.environ["AWS_SECRET_ACCESS_KEY"] = "YOUR-AWS-SECRET-ACCESS-KEY"

data_path = "s3://PATH/TO/YOUR/REMOTE/FILE/data.csv"
df = pd.read_csv(data_path)
```

## SSH and SFTP access

We allow remote access to your Jupyter Server Environment via SSH / SFTP. 

To do so, request a new token from you control panel. This will serve as your password to the remote server.

![Generate a new token](Notebook%20s%20b4d80/Untitled%204.png)

Generate a new token

Then from your ssh client, use the following configuration

```
User=<The user displayed on the right upper corner>
Host=jhub.convect.ai
Port=8022
Password=<The token you generated from the last step>
```

For example,

```bash
$ ssh -o User=yuhui@convect.ai jhub.convect.ai  -p 8022
Warning: Permanently added the RSA host key for IP address '[54.188.2.122]:8022' to the list of known hosts.
Password:
/opt/conda/condabin/conda
/opt/conda/condabin/conda
convect-user@jupyter-yuhui-40convect-2eai:~$ echo 'I am on the remote server'
```

For SFTP access, the port number is `8023`. 

## Alternative IDEs

In addition to JupyterLab environment, we also provide alternative IDEs that can be started from the launcher. For example, you can start a VScode server from the launcher. 

![VSCode support](Notebook%20s%20b4d80/Untitled%205.png)

You will be automatically redirected to VSCode once it is started.

![VSCode env](Notebook%20s%20b4d80/Untitled%206.png)

Rstudio is also supported. To launch one, choose Rstudio from the launcher.

![RStudio support](Notebook%20s%20b4d80/rstudio-launcher.png)

You will be automatically redirected to RStudio once it is started.

![RStudio env](Notebook%20s%20b4d80/rstudio-ide.png)

Currently ssh remote plugins of VSCode and PyCharm is not supported. We encourage users to use the web-based IDEs mentioned above.


## Using tensorboard

To use tensorboard in the JupyterLab environment, simply execute the below in a cell

```
%load_ext tensorboard

%tensorboard --logdir <YOUR_LOG_DIR>
```

![Tensorboard](Notebook%20s%20b4d80/tensorboard.png)