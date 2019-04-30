Parameter optimization via mlrMBO
================
Robin Browaeys
2018-02-20

<!-- github markdown built using 
rmarkdown::render("vignettes/parameter_optimization.Rmd", output_format = "github_document") # please, don't run this!!
-->

This vignette shows how we optimized both hyperparameters and data source weights via model-based optimization (see manuscript for more information). Because the optimization requires intensive parallel computation, we performed optimization in parallel on a gridengine cluster via the qsub package (https://cran.r-project.org/web/packages/qsub/qsub.pdf). This script is merely illustrative and should be adapted by the user to work on its own system.

The input data used in this vignette can be found at: [![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.1484138.svg)](https://doi.org/10.5281/zenodo.1484138). 

First, we will load in the required packages and networks we will use to construct the models which we will evaluate during the optimization procedure.
```{r}
library(nichenetr)
library(dplyr)
library(qsub)
library(mlrMBO)

# in the NicheNet framework, ligand-target links are predicted based on collected biological knowledge on ligand-receptor, signaling and gene regulatory interactions
lr_network = readRDS(url("https://zenodo.org/record/1484138/files/lr_network.rds"))
sig_network = readRDS(url("https://zenodo.org/record/1484138/files/signaling_network.rds"))
gr_network = readRDS(url("https://zenodo.org/record/1484138/files/gr_network.rds"))
```

Because we optimize the paramters to maximize target gene and ligand activity prediction, we will load in the validation datasets. In this vignette, we do the optimization on all datasets (so no cross-validation).

```{r}
# The ligand treatment expression datasets used for validation can be downloaded from Zenodo:
expression_settings_validation = readRDS(url("https://zenodo.org/record/1484138/files/expression_settings.rds"))
```

Define the optimization wrapper function and config information for the qsub package
```{r}
mlrmbo_optimization_wrapper = function(...){
  library(nichenetr)
  library(mlrMBO)
  library(parallelMap)
  library(dplyr)
  output = mlrmbo_optimization(...)
  return(output)
}

qsub_config = create_qsub_config(
  remote = "myuser@mycluster.address.org:1234",
  local_tmp_path = "/tmp/r2gridengine",
  remote_tmp_path = "/scratch/personal/myuser/r2gridengine",
  num_cores = 24,
  memory = "10G",
  wait = FALSE, 
  max_wall_time = "500:00:00"
)
```

Perform optimization fold 1: optimize model performances on expression datasets groups 1 and 2

```{r}
additional_arguments_topology_correction = list(source_names = source_weights_df$source %>% unique(), 
                                                algorithm = "PPR", 
                                                correct_topology = FALSE,
                                                lr_network = lr_network, 
                                                sig_network = sig_network, 
                                                gr_network = gr_network, 
                                                settings = lapply(expression_settings_validation,convert_expression_settings_evaluation), 
                                                secondary_targets = FALSE, 
                                                remove_direct_links = "no", 
                                                cutoff_method = "quantile")
nr_datasources = additional_arguments_topology_correction$source_names %>% length()

obj_fun_multi_topology_correction = makeMultiObjectiveFunction(name = "nichenet_optimization",
                                                               description = "data source weight and hyperparameter optimization: expensive black-box function", 
                                                               fn = model_evaluation_optimization, 
                                                               par.set = makeParamSet(
                                                                 makeNumericVectorParam("source_weights", len = nr_datasources, lower = 0, upper = 1), 
                                                                 makeNumericVectorParam("lr_sig_hub", len = 1, lower = 0, upper = 1),  
                                                                 makeNumericVectorParam("gr_hub", len = 1, lower = 0, upper = 1),  
                                                                 makeNumericVectorParam("ltf_cutoff", len = 1, lower = 0.9, upper = 0.999),  
                                                                 makeNumericVectorParam("damping_factor", len = 1, lower = 0.01, upper = 0.99)), 
                                                               has.simple.signature = FALSE,
                                                               n.objectives = 4, 
                                                               noisy = FALSE,
                                                               minimize = c(FALSE,FALSE,FALSE,FALSE))
set.seed(1)
job_mlrmbo = qsub_lapply(X = 1,
                               FUN = mlrmbo_optimization_wrapper,
                               object_envir = environment(mlrmbo_optimization_wrapper),
                               qsub_config = qsub_config,
                               qsub_environment = NULL, 
                               qsub_packages = NULL,
                               obj_fun_multi_topology_correction, 
                               50, 24, 240, 
                               additional_arguments_topology_correction)
res_job_mlrmbo = qsub_retrieve(job_mlrmbo)
```

Get now the most optimal parameter setting as a result of this analysis

```{r}
optimized_parameters = res_job_mlrmbo %>% process_mlrmbo_nichenet_optimization(source_names = source_weights_df$source %>% unique())
```
