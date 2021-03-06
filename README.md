# Heart Failure Predication using Azure ML
## Table of contents
   * [Overview](#Overview)
   * [Project Set Up and Installation](#Project-Set-Up-and-Installation)
   * [Dataset](#Dataset)
   * [Automated ML](#Automated-ML)
   * [Hyperparameter Tuning](#Hyperparameter-Tuning)
   * [Model Deployment](#Model-Deployment)
   * [Screen Recording](#Screen-Recording)
   * [Comments and future improvements](#Comments-and-future-improvements)   
   * [References](#References)
***
## Overview
The current project uses machine learning to predict patients’ survival based on their medical data. 
I create two models in the environment of Azure Machine Learning Studio: one using Automated Machine Learning (i.e. AutoML) and one customized model whose hyperparameters are tuned using HyperDrive. I then compare the performance of both models and deploy the best performing model as a service using Azure Container Instances (ACI).
The diagram below is a visualization of the rough overview of the operations that take place in this project:
![Project Workflow](img/final_project_workflow.JPG?raw=true "Project Workflow") 

Refernce for study
- [Deploying Machine Learning Models to Azure Kubernetes Service](https://clemenssiebler.com/deploying-machine-learning-models-to-azure-kubernetes-service/)
- [Deploy a model to an Azure Kubernetes Service cluster](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-deploy-azure-kubernetes-service?tabs=python)
- [How Azure Machine Learning Service deployment to Azure Kubernetes Service works](https://liupeirong.github.io/amlKubernetesDeployment/)
- [End-to-End Pipeline Example on Azure](https://www.kubeflow.org/docs/azure/azureendtoend/)
- [How to bring your Data Science Project in production](https://towardsdatascience.com/how-to-bring-your-data-science-project-in-production-b36ae4c02b46)
- [Mounting Datasets to a Compute Instance in Azure Machine Learning](https://clemenssiebler.com/mount-datasets-compute-instance-azure-machine-learning/)
- [Deploy and serve model from Azure Databricks onto Azure Machine Learning](https://www.iteblog.com/ppt/sparkaisummit-north-america-2020-iteblog/deploy-and-serve-model-from-azure-databricks-onto-azure-machine-learning-iteblog.com.pdf)
- [Azure Machine Learning Services: a complete toolbox for AI](https://www.element61.be/en/resource/azure-machine-learning-services-complete-toolbox-ai)
- [Using AI-Demand Forecasting to drive Supply Chain Planning](https://www.element61.be/en/project/using-ai-demand-forecasting-drive-supply-chain-planning)
- [Azure Container Solutions](https://www.wintellect.com/wp-content/uploads/2020/04/containers.pdf)
- [Databricks MLOps - Deploy Machine Learning Model On Azure](https://www.youtube.com/watch?v=fv3p3r3ByfY)
- [Machine Learning Crash Course](https://developers.google.com/machine-learning/crash-course)
- [MLOps](https://azure.microsoft.com/en-us/services/machine-learning/mlops/#resources)

Equivalent typical Azure ML Architecture as below:
![Azure Architecture](img/azure_architecture.JPG?raw=true "Azure ML Project Workflow") 

## Project Set Up and Installation
In order to run the project in Azure Machine Learning Studio, we will need the two Jupyter Notebooks:
- `automl.ipynb`: for the AutoML experiment;
- `hyperparameter_tuning.ipynb`: for the HyperDrive experiment.

The following files are also necessary:
- `heart_failure_clinical_records_dataset.csv`: the dataset file. It can also be taken directly from Kaggle; 
- `train.py`: a basic script for manipulating the data used in the HyperDrive experiment;
- `score.py`: the script used to deploy the model which is downloaded from within Azure Machine Learning Studio; &
- `env.yml`: the environment file which is also downloaded from within Azure Machine Learning Studio.

## Dataset
### Overview
Cardiovascular diseases (CVDs) kill approximately 18 million people globally every year, being the number 1 cause of death globally. Heart failure is one of the two ways CVDs exhibit (the other one being myocardial infarctions) and occurs when the heart cannot pump enough blood to meet the needs of the body. People with cardiovascular disease or who are at high cardiovascular risk need early detection and management wherein Machine Learning would be of great help. This is what this project attempts to do: create an ML model that could help predicting patients’ survival based on their medical data.

The dataset used is taken from [Kaggle](https://www.kaggle.com/andrewmvd/heart-failure-clinical-data) and the data comes from 299 patients with heart failure collected at the Faisalabad Institute of Cardiology and at the Allied Hospital in Faisalabad,Pakistan, during April–December 2015. The patients consisted of 105 women and 194 men, and their ages range between 40 and 95 years old.

The dataset contains 13 features:
| Feature | Explanation | Measurement |
| :--- | :--- | :--- |
| *age* | Age of patient | Years (40-95) |
| *anaemia* | Decrease of red blood cells or hemoglobin | Boolean (0=No, 1=Yes) |
| *creatinine-phosphokinase* | Level of the CPK enzyme in the blood | mcg/L |
| *diabetes* | Whether the patient has diabetes or not | Boolean (0=No, 1=Yes) |
| *ejection_fraction* | Percentage of blood leaving the heart at each contraction | Percentage |
| *high_blood_pressure* | Whether the patient has hypertension or not | Boolean (0=No, 1=Yes) |
| *platelets* | Platelets in the blood | kiloplatelets/mL	|
| *serum_creatinine* | Level of creatinine in the blood | mg/dL |
| *serum_sodium* | Level of sodium in the blood | mEq/L |
| *sex* | Female (F) or Male (M) | Binary (0=F, 1=M) |
| *smoking* | Whether the patient smokes or not | Boolean (0=No, 1=Yes) |
| *time* | Follow-up period | Days |
| *DEATH_EVENT* | Whether the patient died during the follow-up period | Boolean (0=No, 1=Yes) |

### Task 
The main task that I seek to solve with this project & dataset is to classify patients based on their odds of survival. The prediction is based on the first 12 features included in the above table, while the classification result is reflected in the last column named _Death event (target)_ and it is either `0` (_`no`_) or `1` (_`yes`_).

### Access
First, I made the data publicly accessible in the current GitHub repository via this link:
[https://raw.githubusercontent.com/ddgope/Udacity-Capstone-Heart-Failure-Prediction/master/heart_failure_clinical_records_dataset.csv](https://raw.githubusercontent.com/ddgope/Udacity-Capstone-Heart-Failure-Prediction/master/heart_failure_clinical_records_dataset.csv)
and then create the dataset:
![Dataset creation](img/00.JPG?raw=true "heart-failure-prediction dataset creation")

As it is depicted below, the dataset is registered in Azure Machine Learning Studio:

***Registered datasets:*** _Dataset heart-failure-prediction registered_
![Registered datasets](img/01.JPG?raw=true "heart-failure-prediction dataset registered")

I am also accessing the data directly via:
```
data = pd.read_csv('./heart_failure_clinical_records_dataset.csv')
```

## Automated ML
***AutoML settings and configuration:***
Below you can see an overview of the `automl` settings and configuration I used for the AutoML run:
```
automl_settings = {"n_cross_validations": 2,
                    "primary_metric": 'accuracy',
                    "enable_early_stopping": True,
                    "max_concurrent_iterations": 4,
                    "experiment_timeout_minutes": 25,
                    "blocked_models":['XGBoostClassifier'],
                    "verbosity": logging.INFO                   
                    }
```
```
automl_config = AutoMLConfig(compute_target = compute_target,
                            task='classification',
                            training_data=dataset,
                            label_column_name='DEATH_EVENT',
                            path = project_folder,
                            featurization= 'auto',
                            debug_log = "automl_errors.log",
                            enable_onnx_compatible_models=False,                            
                            **automl_settings
                            )
```
`"n_cross_validations": 2`
This parameter sets how many cross validations to perform, based on the same number of folds (number of subsets). As one cross-validation could result in overfit, in my code I chose 2 folds for cross-validation; thus the metrics are calculated with the average of the 2 validation metrics.

`"primary_metric": 'accuracy'`
I chose accuracy as the primary metric as it is the default metric used for classification tasks.

`"enable_early_stopping": True`
It defines to enable early termination if the score is not improving in the short term. In this experiment, it could also be omitted because the _experiment_timeout_minutes_ is already defined below.

`"max_concurrent_iterations": 4`
It represents the maximum number of iterations that would be executed in parallel.

`"experiment_timeout_minutes": 20`
This is an exit criterion and is used to define how long, in minutes, the experiment should continue to run. To help avoid experiment time out failures, I used the value of 20 minutes.

`"verbosity": logging.INFO`
The verbosity level for writing to the log file.

`compute_target = compute_target`
The Azure Machine Learning compute target to run the Automated Machine Learning experiment on.

`task = 'classification'`
This defines the experiment type which in this case is classification. Other options are _regression_ and _forecasting_.

`training_data = dataset`
The training data to be used within the experiment. It should contain both training features and a label column - see next parameter.

`label_column_name = 'DEATH_EVENT'` 
The name of the label column i.e. the target column based on which the prediction is done.

`path = project_folder`
The full path to the Azure Machine Learning project folder.

`featurization = 'auto'`
This parameter defines whether featurization step should be done automatically as in this case (_auto_) or not (_off_).

`debug_log = 'automl_errors.log`
The log file to write debug information to.

`enable_onnx_compatible_models = False`
I chose not to enable enforcing the ONNX-compatible models at this stage. However, I will try it in the future. For more info on Open Neural Network Exchange (ONNX), please see [here](https://docs.microsoft.com/en-us/azure/machine-learning/concept-onnx).

### Results
During the AutoML run, the _Data Guardrails_ are run when automatic featurization is enabled. As we can see in the screenshot below, the dataset passed all three checks:

***Data Guardrails Checks in Azure Machine Learning Studio***
![Data Guardrails Checks](img/10.JPG?raw=true "Data Guardrails Checks in Azure Machine Learning Studio")

#### Best model
After the completion, we can see and take the metrics and details of the best run:
![Best run metrics and details](img/11.JPG?raw=true "Best run metrics and details")

![Best run properties](img/12.JPG?raw=true "Best run properties")

![Fitted model parameters](img/13.JPG?raw=true "Fitted model parameters")

***Screenshots from Azure ML Studio***
_AutoML models_
![AutoML models](img/20.JPG?raw=true "AutoML models")

_Charts_
![Best model metrics - Charts](img/13.JPG?raw=true "Best model metrics - Charts")

![Best model metrics - Charts](img/14.JPG?raw=true "Best model metrics - Charts")

## Hyperparameter Tuning
For this experiment I am using a custom Scikit-learn Logistic Regression model, whose hyperparameters I am optimising using HyperDrive. Logistic regression is best suited for binary classification models like this one and this is the main reason I chose it.
I specify the parameter sampler using the parameters C and max_iter and chose discrete values with choice for both parameters.

**Parameter sampler**
I specified the parameter sampler as such:
```
ps = RandomParameterSampling(
    {
        '--C' : choice(0.001,0.01,0.1,1,10,20,50,100,200,500,1000),
        '--max_iter': choice(50,100,200,300)
    }
)
```
I chose discrete values with _choice_ for both parameters, _C_ and _max_iter_.
_C_ is the Regularization while _max_iter_ is the maximum number of iterations.
_RandomParameterSampling_ is one of the choices available for the sampler and I chose it because it is the faster and supports early termination of low-performance runs. If budget is not an issue, we could use _GridParameterSampling_ to exhaustively search over the search space or _BayesianParameterSampling_ to explore the hyperparameter space. 

**Early stopping policy**
An early stopping policy is used to automatically terminate poorly performing runs thus improving computational efficiency. I chose the _BanditPolicy_ which I specified as follows:
```
policy = BanditPolicy(evaluation_interval=2, slack_factor=0.1)
```

_evaluation_interval_: This is optional and represents the frequency for applying the policy. Each time the training script logs the primary metric counts as one interval.

_slack_factor_: The amount of slack allowed with respect to the best performing training run. This factor specifies the slack as a ratio.
Any run that doesn't fall within the slack factor or slack amount of the evaluation metric with respect to the best performing run will be terminated. This means that with this policy, the best performing runs will execute until they finish and this is the reason I chose it.

### Results

#### Completion of the HyperDrive Run:
![HyperDrive run](img/h01.JPG?raw=true "HyperDrive run")

#### Best model
After the completion, we can see and get the metrics and details of the best run:
![HyperDrive run hyperparameters](img/h02.JPG?raw=true "HyperDrive run hyperparameters")

***Screenshots from Azure ML Studio***
_HyperDrive model_
![HyperDrive model](img/h03.JPG?raw=true "HyperDrive model")

_Best model data and details_
![Best model details](img/h04.JPG?raw=true "Best model details")

_Best model metrics_
![Best model metrics](img/h05.JPG?raw=true "Best model metrics")

## Model Deployment
The deployment is done following the steps below:
* Selection of an already registered model
* Preparation of an inference configuration
* Preparation of an entry script
* Choosing a compute target
* Deployment of the model
* Testing the resulting web service

### Registered model
Using as basis the `accuracy` metric, we can state that the best AutoML model is superior to the best model that resulted from the HyperDrive run. For this reason, I choose to deploy the best model from AutoML run (`best_run_automl.pkl`, Version 2). 
_Registered models in Azure Machine Learning Studio_
![Registered models](img/R01.JPG?raw=true "Registered models")

_Runs of the experiment_
![Runs of the experiment](img/R02.JPG?raw=true "Runs of the experiment")

### Inference configuration
The inference configuration defines the environment used to run the deployed model. An Azure Machine Learning environment, named `env.yml` in this case. The environment defines the software dependencies needed to run the model and entry script.

### Entry script
The entry script is the `score.py` file. The entry script loads the model when the deployed service starts and it is also responsible for receiving data, passing it to the model, and then returning a response.

### Compute target
As compute target, I chose the Azure Container Instances (ACI) service, which is used for low-scale CPU-based workloads that require less than 48 GB of RAM.
The AciWebservice Class represents a machine learning model deployed as a web service endpoint on Azure Container Instances. The deployed service is created from the model, script, and associated files, as I explain above. The resulting web service is a load-balanced, HTTP endpoint with a REST API. We can send data to this API and receive the prediction returned by the model.
![Compute target](img/C01.JPG?raw=true "Compute target")
`cpu_cores` : It is the number of CPU cores to allocate for this Webservice. Can also be a decimal.
`memory_gb` : The amount of memory (in GB) to allocate for this Webservice. Can be a decimal as well.
`auth_enabled` : I set it to _True_ in order to enable auth for the Webservice.
`enable_app_insights` : I set it to _True_ in order to enable AppInsights for this Webservice.

### Deployment
Bringing all of the above together, here is the actual deployment in action:
![Model deployment](img/MD01.JPG?raw=true "Model deployment")

_Best AutoML model deployed (Azure Machine Learning Studio)_
![Best model deployment](img/BM01.JPG?raw=true "Best model deployment")
Deployment takes some time to conclude, but when it finishes successfully the ACI web service has a status of ***Healthy*** and the model is deployed correctly. We can now move to the next step of actually testing the endpoint.

### Consuming/testing the REST API Endpoint(ACI service)
_Endpoint (Azure Machine Learning Studio)_
![ACI service](img/ACI01.JPG?raw=true "ACI service")
After the successful deployment of the model and with a _Healthy_ service, I can print the _scoring URI_, the _Swagger URI_ and the _primary authentication key_:
![ACI service status and data](img/ACI02.JPG?raw=true "ACI service status and data")
The scoring URI can be used by clients to submit requests to the service.
In order to test the deployed model, I use a _Python_ file, named `endpoint.py`:
![endpoint.py file](img/ACI03.JPG?raw=true "endpoint.py file")
In the beginning, I fill in the `scoring_uri` and `key` with the data of the _aciservice_ printed above. We can test our deployed service, using test data in JSON format, to make sure the web service returns a result.
In order to request data, the REST API expects the body of the request to be a JSON document with the following structure: 
```
{
    "data":
        [
            <model-specific-data-structure>
        ]
}
```
Below is the data structure:
![Data structure](img/ACI04.jfif?raw=true "Data structure")

The data is then converted to JSON string format as below:
![Conversion to JSON string format](img/JSON1.JPG?raw=true "Conversion to JSON string format")

JSON content type looks like as below:
![Setting the content type](img/ACI07.jfif?raw=true "Setting the content type")
![Setting the content type](img/ACI06.jfif?raw=true "Setting the content type")

Finally, we make the request and print the response on screen:
![Request and response](img/ACI07.jfif?raw=true "Request and response")

## Screen Recording
The screen recording can be found [here](https://youtu.be/gFxeAVxdrUc) and it shows the project in action. 
More specifically, the screencast demonstrates:
- What is the project about and what is execpted from predication
- Two different types of Model using Hpyerparameter and AutoML
- Find the best working model
- Demo of the deployed best model
- Demo of a sample request sent to the endpoint and its response

## Comments and future improvements
* The first factor that could improve the model is increasing the training time. This suggestion might be seen as a no-brainer, but it would also increase costs and this is a limitation that can be very difficult to overcome: there must always be a balance between minimum required accuracy and assigned budget.
* Continuing the above point, it would be great to be able to experiment more with the hyperparameters chosen for the HyperDrive model or even try running it with more of the available hyperparameters, with less time contraints.
* Another thing I would try is deploying the best models to the Edge using Azure IoT Edge and enabling logging in the deployed web apps.
* I would certainly try to deploy the HyperDrive model as well, since the deployment procedure is a bit defferent than the one used for the AutoML model.


## References
- Udacity Nanodegree material
- [Heart Failure Prediction Dataset](https://www.kaggle.com/andrewmvd/heart-failure-clinical-data)
- [Consume an Azure Machine Learning model deployed as a web service](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-consume-web-service?tabs=python)
- [Deploy machine learning models to Azure](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-deploy-and-where?tabs=azcli)
- [A Review of Azure Automated Machine Learning (AutoML)](https://medium.com/microsoftazure/a-review-of-azure-automated-machine-learning-automl-5d2f98512406)
- [What is Azure Container Instances (ACI)?](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-overview)
- [AutoMLConfig Class](https://docs.microsoft.com/en-us/python/api/azureml-train-automl-client/azureml.train.automl.automlconfig.automlconfig?view=azure-ml-py)
- [Using Azure Machine Learning for Hyperparameter Optimization](https://dev.to/azure/using-azure-machine-learning-for-hyperparameter-optimization-3kgj)
- [hyperdrive Package](https://docs.microsoft.com/en-us/python/api/azureml-train-core/azureml.train.hyperdrive?view=azure-ml-py)
- [Tune hyperparameters for your model with Azure Machine Learning](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-tune-hyperparameters)
- [Configure automated ML experiments in Python](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-configure-auto-train)
- [How Azure Machine Learning works: Architecture and concepts](https://docs.microsoft.com/en-us/azure/machine-learning/concept-azure-machine-learning-architecture)
- [Configure data splits and cross-validation in automated machine learning](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-configure-cross-validation-data-splits)
