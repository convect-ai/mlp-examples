# 看板应用

ConvectHub允许直接将notebook文件部署为看板应用。支持多个看板框架，例如streamlit, plotly, viola。

例如，如需使用streamlit创建一个看板应用，首先建立一个python文件`app.py` 在如下路径 `convect-mlp-examples/dashboard/streamlit/` 

[mlp-examples/app.py at main · convect-ai/mlp-examples](https://github.com/convect-ai/mlp-examples/blob/main/dashboard/streamlit/app.py)

然后在首页的Dashboard页面下，创建一个如下的一个看板应用。

![Untitled](Dashboards%2011636/Untitled.png)

选择`streamlit`作为框架，将创建的文件`[app.py]`作为相对路径输入。 完成创建后，选择部署这个看板的服务器资源需求。由于这个例子比较简单，我们选择最小的服务器资源。

![Untitled](Dashboards%2011636/Untitled%201.png)

和启动一个Jupyter开发环境一下，我们需要等待服务初始化。完成之后会自动跳转到生成的看板应用。

![Untitled](Dashboards%2011636/Untitled%202.png)

跳转完成后我们就可以和这个看板应用互动了。

![Untitled](Dashboards%2011636/Untitled%203.png)