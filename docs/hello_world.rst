"Hello, World"
______________

Bayesian estimation via Stan's HMC-NUTS sampler 
------------------------------------------------

To exercise the essential functions of CmdStanPy, we will
compile the example Stan model ``bernoulli.stan``, which is
distributed with CmdStan and then fit the model to example data
``bernoulli.data.json``, also distributed with CmdStan using
Stan's HMC-NUTS sampler in order to estimate the posterior probability
of the model parameters conditioned on the data.


Specify a Stan model
^^^^^^^^^^^^^^^^^^^^

The ``CmdStanModel`` class specifies the Stan program and its corresponding compiled executable.
By default, the Stan program is compiled on instantiation.

.. code-block:: python

    import os
    from cmdstanpy import cmdstan_path, CmdStanModel

    bernoulli_stan = os.path.join(cmdstan_path(), 'examples', 'bernoulli', 'bernoulli.stan')
    bernoulli_model = CmdStanModel(stan_file=bernoulli_stan)

The ``CmdStanModel`` class provides properties and functions to inspect the model code and filepaths.

.. code-block:: python

    bernoulli_model.name
    bernoulli_model.stan_file
    bernoulli_model.exe_file
    bernoulli_model.code()


            
Run the HMC-NUTS sampler
^^^^^^^^^^^^^^^^^^^^^^^^

The ``CmdStanModel`` method ``sample`` runs the Stan HMC-NUTS sampler on the model and data
and returns a ``CmdStanMCMC`` object:

.. code-block:: python

    bernoulli_data = { "N" : 10, "y" : [0,1,0,0,0,0,0,0,0,1] }
    bern_fit = bernoulli_model.sample(data=bernoulli_data, output_dir='.')

By default, the ``sample`` command runs 4 sampler chains.
The ``output_dir`` argument specifies the path to the sampler output files.
If no output file path is specified, the sampler outputs
are written to a temporary directory which is deleted
when the current Python session is terminated.

The ``CmdStanMLE`` object records the command, the return code,
and the paths to the optimize method output csv and console files.
The output files are written either to a specified output directory
or to a temporary directory which is deleted upon session exit.

Output filenames are composed of the model name, a timestamp
in the form YYYYMMDDhhmm and the chain id, plus the corresponding
filetype suffix, either '.csv' for the CmdStan output or '.txt' for
the console messages, e.g. ``bernoulli-201912081451-1.csv``. Output files
written to the temporary directory contain an additional 8-character
random string, e.g. ``bernoulli-201912081451-1-5nm6as7u.csv``.


Access the sample
^^^^^^^^^^^^^^^^^

The ``sample`` command returns a ``CmdStanMCMC`` object
which provides methods to retrieve the sampler outputs,
the arguments used to run Cmdstan, and names of the
the per-chain stan-csv output files, and per-chain console messages files.

.. code-block:: python

   print(bern_fit)

The resulting sample from the posterior is lazily instantiated
the first time that any of the properties
``sample``, ``metric``, or ``stepsize`` are accessed.
At this point the stan-csv output files are read into memory.
For large files this may take several seconds; for the example
dataset, this should take less than a second.
The ``sample`` property of the ``CmdStanMCMC`` object
is a 3-D ``numpy.ndarray`` (i.e., a multi-dimensional array)
which contains the set of all draws from all chains 
arranged as dimensions: (draws, chains, columns).

.. code-block:: python

    bern_fit.sample.shape


The ``get_drawset`` method returns the draws from
all chains as a ``pandas.DataFrame``, one draw per row, one column per
model parameter, transformed parameter, generated quantity variable.
The ``params`` argument is used to restrict the DataFrame
columns to just the specified parameter names.

.. code-block:: python

    bern_fit.get_drawset(params=['theta'])

Python's index slicing operations can be used to access the information by chain.
For example, to select all draws and all output columns from the first chain,
we specify the chain index (2nd index dimension).  As arrays indexing starts at 0,
the index '0' corresponds to the first chain in the ``CmdStanMCMC``:

.. code-block:: python

    chain_1 = bern_fit.sample[:,0,:]
    chain_1.shape       # (1000, 8)
    chain_1[0]          # sample first draw:
                        # array([-7.99462  ,  0.578072 ,  0.955103 ,  2.       ,  7.       ,
                        # 0.       ,  9.44788  ,  0.0934208])

Summarize or save the results
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

CmdStan is distributed with a posterior analysis utility ``stansummary``
that reads the outputs of all chains and computes summary statistics
on the model fit for all parameters. The ``CmdStanMCMC`` method ``summary``
runs the CmdStan ``stansummary`` utility and returns the output as a pandas.DataFrame:

.. code-block:: python

    bern_fit.summary()

CmdStan is distributed with a second posterior analysis utility ``diagnose``
that reads the outputs of all chains and checks for the following
potential problems:

+ Transitions that hit the maximum treedepth
+ Divergent transitions
+ Low E-BFMI values (sampler transitions HMC potential energy)
+ Low effective sample sizes
+ High R-hat values

The ``CmdStanMCMC`` method ``diagnose`` runs the CmdStan ``diagnose`` utility
and prints the output to the console.

.. code-block:: python

    bern_fit.diagnose()

The sampler output files are written to a temporary directory which
is deleted upon session exit unless the ``output_dir`` argument is specified.
The ``save_csvfiles`` function moves the CmdStan csv output files
to a specified directory without having to re-run the sampler.

.. code-block:: python

    bern_fit.save_csvfiles(dir='some/path')

.. comment
  Progress bar
  ^^^^^^^^^^^^
  
  User can enable progress bar for the sampling if ``tqdm`` package
  has been installed.
  
  .. code-block:: python
  
      bern_fit = bernoulli_model.sample(data=bernoulli_data, show_progress=True)
  
  On Jupyter Notebook environment user should use notebook version
  by using ``show_progress='notebook'``.
  
  .. code-block:: python
  
      bern_fit = bernoulli_model.sample(data=bernoulli_data, show_progress='notebook')
  
  To enable javascript progress bar on Jupyter Lab Notebook user needs to install
  nodejs and ipywidgets. Following the instructions in
  `tqdm issue #394 <https://github.com/tqdm/tqdm/issues/394#issuecomment-384743637>`
  For ``conda`` users installing nodejs can be done with ``conda``.
  
  .. code-block:: bash
  
      conda install nodejs
  
  After nodejs has been installed, user needs to install ipywidgets and enable it.
  
  .. code-block:: bash
  
      pip install ipywidgets
      jupyter nbextension enable --py widgetsnbextension
  
  Jupyter Lab still needs widgets manager.
  
  .. code-block:: bash
  
      jupyter labextension install @jupyter-widgets/jupyterlab-manager
