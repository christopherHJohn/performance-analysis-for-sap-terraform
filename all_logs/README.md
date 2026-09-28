# All Logs

If you want to dive into all these logs, here some infos on the naming of the files:

- `p<parallelism used>` - default: p10
- `f<factor used>` - default: f1
- `v<execution index>` - default: v1


## Notes

- I did some changes while writing these blogs on what resources are included and the amount of them. So for example the destination resource will be missing in some of the outputs.
- Additionaly, there is a bug in some outputs, where resources are missing in the runs or even later on in the final/ grouped results.
- Lastly, I renamed from `specific_test` to `special_test` during later execution for no reason. These use the same terraform states and structure.


## Structure of output files

### (1) Execution details

Shows the input values for the script I used, including the run count and parallelism.

```
Running terraform plan -refresh-only 3 time(s)
Using Terraform parallelism: 50
```

_The parallelism is omitted if it is 10._

### (2) Runs

Showing the API execution time for each run and each ressource as well as a total runtime of the `terraform plan` (this number is not an aggregate of all API calls, but calculated seperatly). If there is any error during a run it will also be shown at the top of the run with a generic message.

```
Run 1/20:
  Total run time: 19.720483s
  btp_directory.directory                                                4289ms
  btp_subaccount.subaccount                                              2659ms
  btp_subaccount_destination_generic.destination                         2843ms
...
  sci_user.user_1                                                        292ms
  sci_user.user_2                                                        315ms
  sci_user.user_3                                                        307ms
  sci_user.user_4                                                        306ms
```

### (3) Final Results

Gives the aggregate data for all resources by average, minumum and maximum.

```
Final results:
-------                                               ------------    -----------    -----------
Element                                               Average (ms)   Minimum (ms)   Maximum (ms)
btp_directory.directory_1                                  1816.10            772           3001
btp_directory.directory_10                                 1974.90           1176           3699
btp_directory.directory_11                                 2108.20            525           3290
btp_directory.directory_12                                 1997.50            113           3733
...
sci_user.user_98                                             98.30             16            137
sci_user.user_99                                             93.90             79            123
```


### (4) Grouped results

Show the final results, but grouped one more time. Meaning each resource is now listed only once.

```
Grouped results:
-------                                 ------------    -----------    -----------
Element                                 Average (ms)   Minimum (ms)   Maximum (ms)
btp_directory.*                              1968.51            113           6091
btp_subaccount.*                             1019.54             47           2977
btp_subaccount_destination_generic.*         1099.19            156           3316
...
sci_group_base.*                              100.05             11            365
sci_user.*                                     97.01             13            379
```


### (5) Runtime details

The footer shows the total `terraform plan` runtime data including again the average, minumum and maximum as well as a total of all runs. Lastly it also shows how many resources where included in this file.

```
Total wall-clock time: 2238.711041s
Average wall-clock time: 223.871104s
Minimum wall-clock time: 206.668659s
Maximum wall-clock time: 241.650295s
Total Ressources: 1075
```