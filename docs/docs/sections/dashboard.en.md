# Dashboards

ConvectHub can create interative dashboard apps that use popular frmeworks such as streamlit, plotly, viola and share them with other team members.

For example, to create a streamlit app, we first create python file `app.py` under `convect-mlp-examples/dashboard/streamlit/` as below

[mlp-examples/app.py at main · convect-ai/mlp-examples](https://github.com/convect-ai/mlp-examples/blob/main/dashboard/streamlit/app.py)

Then from the Dashbaord portal, create a new dashboard app like below

![Untitled](Dashboards%2011636/Untitled.png)

Choose `streamlit` as the framework and enter the relative path to `[app.py](http://app.py)` in the relative path input box. After clicking save, choose the server spec to host the app, here we chose the smallest server since it is a simple app.

![Untitled](Dashboards%2011636/Untitled%201.png)

Like the notebook server starting process, we wait until the server is started and will be auto redirected to the app once finish. 

![Untitled](Dashboards%2011636/Untitled%202.png)

Once the server is started, we can start to interact with the deployed dashboard app

![Untitled](Dashboards%2011636/Untitled%203.png)