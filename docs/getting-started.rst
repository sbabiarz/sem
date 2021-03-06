Getting started
===============

`SEM` operates on simulation campaigns, which consist in a collection of results
obtained from running a specific `ns-3` simulation script with possibly varying
command line parameters.

Simulation campaigns are saved in a single folder for portability, and are
accessible through a `CampaignManager` object. Through this class it's possible
to create new campaigns, load existing ones, run simulations and export results
to a variety of formats for analysis and plotting.

In the following sections we will use `SEM` to go from a vanilla ns-3
installation (assumed to be available at `/tmp/ns-3`) to plots visualizing the
results, in as few commands as possible. A script containing the commands of
this section is available at `examples/wifi_plotting_xarray.py`.

Creating and loading a simulation campaign
------------------------------------------

Creation of a new campaign requires:

* The path of the ns-3 installation to use
* The name of the simulation script that will be run
* The location of the folder where the campaign will be saved.

We can create a campaign using the following instructions::

  >>> import sem
  >>> ns_path = 'examples/ns-3'
  >>> script = 'wifi-multi-tos'
  >>> campaign_dir = '/tmp/wifi-plotting-example'
  >>> campaign = sem.CampaignManager.new(ns_path, script, campaign_dir, overwrite=True)

Internally, `SEM` also checks whether the path points to a valid ns-3
installation, and whether the script is actually available for execution or not.
If a simulation campaign already exists at the specified `campaign_dir`, and if
it points to the same `ns_path` and `script` values, that campaign is loaded
instead.

`CampaignManager` objects can also be directly printed to inspect the status of
the campaign:

::

   >>> print(campaign) # doctest: +SKIP
   --- Campaign info ---
   script: wifi-multi-tos
   params: ['nWifi', 'distance', 'simulationTime', 'useRts', 'mcs',
            'channelWidth', 'useShortGuardInterval']
   commit: 9386dc7d106fd9241ff151195a0e6e5cb954d363
   ---------------------

Note that, additionally to the path and script we specified in the campaign
creation process, `SEM` also retrieved a list of the available script parameters
and the SHA of the current HEAD of the git repository at the `ns-3` path.

Running simulations
-------------------

Simulations can be run by specifying a list of parameter combinations.

::

  >>> param_combination = {
  ...    'nWifi': 1,
  ...    'distance': 1,
  ...    'simulationTime': 10,
  ...    'useRts': 'false',
  ...    'mcs': 7,
  ...    'channelWidth': 20,
  ...    'useShortGuardInterval': 'false'
  ... }
  >>> campaign.run_simulations([param_combination])

  Running simulations: 100% 1/1 [00:10<00:00, 10.81s/simulation]

The `run_simulations` method automatically queries the database looking for an
appropriate `RngRun` value that has not yet been used, and runs the requested
simulations. After the simulation finishes, the results (i.e., any generated
output files and the standard output) are added to the database and can be
retrieved later on. A progress bar is displayed to indicate progress and give an
estimate of the remaining time.

Multiple simulations corresponding to the exploration of a parameter space can
be easily run by employing the `list_param_combinations` function, which can
take a dictionary specifying multiple values for a key and translate it into a
list of dictionaries specifying all combinations of parameter values. For
example, let's try the same simulation parameters as before, but perform the
simulations both for the `true` and `false` values of the `useRts` parameter::

  >>> param_combinations = {
  ...     'nWifi': 1,
  ...     'distance': 1,
  ...     'simulationTime': 10,
  ...     'useRts': ['false', 'true'],
  ...     'mcs': 7,
  ...     'channelWidth': 20,
  ...     'useShortGuardInterval': 'false'
  ... }
  >>> campaign.run_missing_simulations(
  ...     sem.list_param_combinations(param_combinations),
  ...     runs=1)

  Running simulations: 100% 1/1 [00:09<00:00,  9.63s/simulation]


From the output, it can be noticed that only one simulation was run: in fact,
since we used the `run_missing_simulations` function, before running the
specified simulations, `SEM` checked whether some results were already available
in the database, found the previously executed simulation, and only performed
the simulation for which no result employing the requested parameter combination
was already available. Additionally, the `run_missing_simulations` function
requires a `runs` parameter, specifying how many runs should be performed for
each parameter combination.

Finally, let's make `SEM` run multiple simulations so that we have something to
plot. In order to do this, first we define a new `param_combinations`
dictionary, ranging the `mcs` parameter from 0 to 7 and turning on and off the
`RequestToSend` and `ShortGuardInterval` parameters::

  >>> param_combinations = {
  ...     'nWifi': 1,
  ...     'distance': 1,
  ...     'simulationTime': 10,
  ...     'useRts': ['false', 'true'],
  ...     'mcs': list(range(1, 8, 2)),
  ...     'channelWidth': 20,
  ...     'useShortGuardInterval': ['false', 'true']
  ... }
  >>> campaign.run_missing_simulations(
  ...             sem.list_param_combinations(param_combinations),
  ...             runs=2)

  Running simulations: 100% 32/32 [02:57<00:00,  3.86s/simulation]


Exporting results
-----------------

Available results can be inspected using the `DatabaseManager` object associated
to the `CampaignManager`, and available as the `db` attribute of the campaign.
For instance, let's check out the first result::

  >>> len(campaign.db.get_results())
  32
  >>> campaign.db.get_results()[0] # doctest: +SKIP
  {
    'nWifi': 1,
    'distance': 1,
    'simulationTime': 10,
    'useRts': 'false',
    'mcs': 7,
    'channelWidth': 20,
    'useShortGuardInterval': 'false',
    'RngRun': 1,
    'id': '771e0511-43b9-4e33-aa6a-dc4266be24f1',
    'elapsed_time': 4.270819187164307,
    'stdout': 'Aggregated throughput: 49.2696 Mbit/s\n'
  }

Results are returned as dictionaries, with a key-value pair for each available
script parameter, and the following additional fields:

  * `RngRun`: the `--RngRun` value that was used for this simulation;
  * `id`: an unique identifier for the simulation;
  * `elapsed_time`: the required time, in seconds, to run the simulation;
  * `stdout`: the output of the simulation script.

Finally, results can be exported to the `numpy` or `xarray` formats.

At its current state, the `SEM` library supports automatic parsing of the
`stdout` result field: in the following lines we will define a
`get_average_throughput` function, which transforms strings formatted like the
`stdout` field of the result above into float numbers containing the average
throughput measured by the simulation. `SEM` will then use the function to
automatically clean up the results before putting them in an `xarray`
structure::

  >>> import re  # Regular expressions to perform the parsing
  >>> def get_average_throughput(result):
  ...     # This function takes a result and parses its standard output to extract
  ...     # relevant information
  ...     available_files = campaign.db.get_result_files(result['meta']['id'])
  ...     with open(available_files['stdout'], 'r') as stdout:
  ...         stdout = stdout.read()
  ...         m = re.match('.*throughput: [-+]?([0-9]*\.?[0-9]+).*', stdout,
  ...                     re.DOTALL).group(1)
  ...         return float(m)

  >>> results = campaign.get_results_as_xarray(param_combinations,
  ...                                          get_average_throughput,
  ...                                          'AvgThroughput', runs=2)

      <xarray.DataArray (useRts: 2, mcs: 8, useShortGuardInterval: 2, runs: 2)>
      array([[[[10.8351 , 10.8057 , 10.8163 ],
              [11.849  , 11.8549 , 11.7901 ]],

              [...]

              [[35.2868 , 35.3763 , 35.3044 ],
              [36.4903 , 36.4137 , 36.4432 ]]]])
      Coordinates:
        * useRts                 (useRts) <U5 'false' 'true'
        * mcs                    (mcs) int64 0 1 2 3 4 5 6 7
        * useShortGuardInterval  (useShortGuardInterval) <U5 'false' 'true'
        * runs                   (runs) int64 0 1 2

Finally, we can easily plot the obtained results by appropriately slicing the
`DataArray`::

  >>> import matplotlib.pyplot as plt
  >>> import numpy as np
  >>> # Iterate over all possible parameter values
  >>> for useShortGuardInterval in ['false', 'true']:
  ...   for useRts in ['false', 'true']:
  ...       avg = results.sel(useShortGuardInterval=useShortGuardInterval,
  ...                         useRts=useRts).reduce(np.mean, 'runs')
  ...       std = results.sel(useShortGuardInterval=useShortGuardInterval,
  ...                         useRts=useRts).reduce(np.std, 'runs')
  ...       eb = plt.errorbar(x=param_combinations['mcs'], y=avg, yerr=6*std,
  ...                    label='SGI %s, RTS %s' % (useShortGuardInterval, useRts))
  ...       xlb = plt.xlabel('MCS')
  ...       ylb = plt.ylabel('Throughput [Mbit/s]')
  >>> legend = plt.legend(loc='best')
  >>> plt.savefig('docs/throughput.png')

.. figure:: throughput.png
    :width: 100%
    :align: center
    :figclass: align-center

    The plot obtained from the simulations.
