# Notebook开发环境

## 启动环境

如需启动一个Jupyter Server开发环境，登录ConvectHub然后点击"Start server"。如果已经有一个开发服务器被启动了，则直接点击"My server"就可访问该服务器。

点击创建后，按照提示选择服务器资源规格。

![Select server spec](Notebook%20s%20b4d80/Untitled.png)

等待开发环境完成初始化后，会自动跳转到服务器。

![Server starting](Notebook%20s%20b4d80/Untitled%201.png)

进入环境后，使用方法同正常的[JupyterLab](https://jupyterlab.readthedocs.io/en/stable/)。

![Main environment](Notebook%20s%20b4d80/Untitled%202.png)

## 管理python依赖和环境

默认的python开发环境已经包含了一些最常见的数据科学包，例如pandas，scikit-learn等。在启动一个notebook的时候，可以在启动器里面选择相应的环境作为运行该notebook的kernel。

在编辑的过程中，通过点击右上角的kernel指示器，也可以动态的切换运行的kernel。

![Select kernel](Notebook%20s%20b4d80/Untitled%203.png)

如需要额外的第三方包，可以直接通过conda来安装使用。从启动器里进入terminal app，然后运行

```bash
conda activate default
conda install pytorch torchvision torchaudio cpuonly -c pytorch
```

也可以从新创建一个新的conda环境

```bash
conda create -n my-conda-env python==3.9
conda activate my-conda-env
# install more packages
conda install scikit-learn
```

新创建的环境在启动器中会自动被作为kernel选项侦测到。如需了解如何使用conda管理python运行环境，可以参考[conda’s doc](https://docs.conda.io/en/latest/)。

## 访问远程数据

如需从开发服务器中访问远程数据，方法和从您本地的电脑一样。例如，如需访问一个远程的S3文件，可以在notebook里运行如下代码

```python
import os
os.environ["AWS_ACCESS_KEY_ID"] = "YOUR-AWS-KEY-ID"
os.environ["AWS_SECRET_ACCESS_KEY"] = "YOUR-AWS-SECRET-ACCESS-KEY"

data_path = "s3://PATH/TO/YOUR/REMOTE/FILE/data.csv"
df = pd.read_csv(data_path)
```

## SSH和SFTP访问

ConvectHub允许直接从本地使用SSH或者SFTP访问开发服务器。

首先进入首页的Token页面，创建一个新的token。这个token会作为连接服务器的密码，请妥善保管。

![Generate a new token](Notebook%20s%20b4d80/Untitled%204.png)


然后，使用你的ssh客户端，用如下的设置来连接远端服务器

```
User=<The user displayed on the right upper corner>
Host=jhub.convect.ai
Port=8022
Password=<The token you generated from the last step>
```

例如,

```bash
$ ssh -o User=yuhui@convect.ai jhub.convect.ai  -p 8022
Warning: Permanently added the RSA host key for IP address '[54.188.2.122]:8022' to the list of known hosts.
Password:
/opt/conda/condabin/conda
/opt/conda/condabin/conda
convect-user@jupyter-yuhui-40convect-2eai:~$ echo 'I am on the remote server'
```

SFTP的连接方法类似，只需要将端口号改为`8023`. 

## 其他开发环境

除开使用JupyterLab作为开发环境，我们也支持其他的开发环境。例如，你可以使用VSCode来开发你的代码。

![VSCode support](Notebook%20s%20b4d80/Untitled%205.png)

只需从启动器里面选择相应的环境，就会自动跳转到开发界面。

![VSCode env](Notebook%20s%20b4d80/Untitled%206.png)

我们也支持使用Rstudio来开发R语言的代码。

我们也支持直接从本地，利用做远端开发的IDE例如PyCharm，VSCode直接连接服务器进行开发。