.. _label_line_by_line:

=========================
Line-by-line (LBL) module
=========================

This is the core of RADIS: it calculates the spectral densities for a homogeneous
slab of gas, and returns a :py:class:`~radis.spectrum.spectrum.Spectrum` object.


.. minigallery:: radis.calc_spectrum


---------------------------------------------------------------------

.. toctree::
   :maxdepth: 3

   lbl

For any other question you can use the `Q&A forum <https://groups.google.com/forum/#!forum/radis-radiation>`__,
the `GitHub issues <https://github.com/radis/radis/issues>`__ or the
community chats on `Gitter <https://gitter.im/radis-radiation/community>`__ or
`Slack <https://radis.github.io/slack-invite/>`__ . |badge_gitter| |badge_slack|


.. include:: databases.rst

Calculating spectra
===================


Calculate one molecule spectrum
-------------------------------

In the following example, we calculate a CO spectrum at equilibrium
from the latest HITRAN database, and plot the transmittance: ::

	s = calc_spectrum(
        		wavenum_min=1900,
        		wavenum_max=2300,
        		Tgas=700,
        		path_length=0.1,
        		molecule='CO',
        		mole_fraction=0.5,
        		isotope=1,
            databank='hitran'   # or 'hitemp'
    		  	)
	s.plot('transmittance_noslit')


Calculate multiple molecules spectrum
-------------------------------------

RADIS can also calculate the spectra of multiple molecules. In the
following example, we add the contribution of CO2 and plot the
transmittance: ::

	s = calc_spectrum(
        	wavenum_min=1900,
        	wavenum_max=2300,
        	Tgas=700,
        	path_length=0.1,
        	mole_fraction={'CO2':0.5, 'CO':0.5},
        	isotope=1,
    		)
	s.plot('transmittance_noslit')


Note that you can indicate the considered molecules either as a list
in the `molecule` parameter, or in `isotope` or `mole_fraction`. The
following commands give the same result: ::


    # Give molecule:
    s = calc_spectrum(
            wavelength_min=4165,
            wavelength_max=5000,
            Tgas=1000,
            path_length=0.1,
            molecule=["CO2", "CO"],
            mole_fraction=1,
            isotope={"CO2": "1,2", "CO": "1,2,3"},
            verbose=verbose,
      )


    # Give isotope only
    s = calc_spectrum(
        wavelength_min=4165,
        wavelength_max=5000,
        Tgas=1000,
        path_length=0.1,
        isotope={"CO2": "1,2", "CO": "1,2,3"},
        verbose=verbose,
    )

    # Give mole fractions only
    s = calc_spectrum(
        wavelength_min=4165,
        wavelength_max=5000,
        Tgas=1000,
        path_length=0.1,
        mole_fraction={"CO2": 0.2, "CO": 0.8},
        isotope="1,2",
        verbose=verbose,
    )

Be careful to be consistent and not to give partial or contradictory inputs. ::

   # Contradictory input:
   s = calc_spectrum(
            wavelength_min=4165,
            wavelength_max=5000,
            Tgas=1000,
            path_length=0.1,
            molecule=["CO2"],  # contradictory
            mole_fraction=1,
            isotope={"CO2": "1,2", "CO": "1,2,3"},
            verbose=verbose,
        )

    # Partial input:
    s = calc_spectrum(
            wavelength_min=4165,
            wavelength_max=5000,
            Tgas=1000,
            path_length=0.1,
            molecule=["CO2", "CO"],  # contradictory
            mole_fraction=1,
            isotope={"CO2": "1,2"},  # unclear for CO
            verbose=verbose,
        )



Flow Chart
----------

Under the hood, RADIS will calculate populations by scaling tabulated data (equilibrium)
or from the rovibrational energies (nonequilibrium), get the emission and absorption coefficients
from :ref:`Line Databases <label_line_databases>`, calculate the line broadening using
various strategies to improve :ref:`Performances <label_lbl_performance>`,
and produce a :ref:`Spectrum object <label_spectrum>`. These steps can be summarized in
the flow chart below:

.. image:: https://radis.readthedocs.io/en/latest/_images/RADIS_flow_chart.svg
    :alt: https://radis.readthedocs.io/en/latest/_images/RADIS_flow_chart.svg
    :scale: 100 %

The detail of the functions that perform each step of the RADIS calculation flow chart
is given in :ref:`Architecture <label_dev_architecture>`.


Equilibrium Conditions
----------------------

By default RADIS calculates spectra at thermal equilibrium (one temperature).

The :py:func:`~radis.lbl.calc.calc_spectrum` function requires a
given mole fraction, which may be different from chemical equilibrium.

You can also compute the chemical equilibrium composition in
other codes like [CANTERA]_, and feed the output to
RADIS :py:func:`~radis.lbl.calc.calc_spectrum`. The
:py:func:`~radis.tools.gascomp.get_eq_mole_fraction` function
provides an interace to [CANTERA]_ directly from RADIS ::

    from radis import calc_spectrum, get_eq_mole_fraction

    # calculate gas composition of a 50% CO2, 50% H2O mixture at 1600 K:
    gas = get_eq_mole_fraction('CO2:0.5, H2O:0.5', 1600, # K
                                         1e5  # Pa
                               )
    # calculate the contribution of H2O to the spectrum:
    calc_spectrum(...,
                  mole_fraction=gas['H2O']
                  )



Nonequilibrium Calculations
---------------------------

Nonequilibrium calculations (multiple temperatures) require to know the
vibrational and rotational energies of each
level in order to calculate the nonequilibrium populations.

You can either let RADIS calculate rovibrational energies
with its built-in :ref:`spectroscopic constants <label_db_spectroscopic_constants>`,
or supply an energy level database. In the latter case, you need to edit the
:ref:`Configuration file <label_lbl_config_file>` .

Calculating spectrum using GPU
------------------------------

RADIS supports calculation of spectra at thermal equilibrium using GPU for the
calculation of lineshapes and broadening. If your system supports it, the spectrum
can be calculated on the GPU using the :py:func:`~radis.lbl.calc.calc_spectrum`
function with parameter `mode` set to `gpu`.

GPU-enabled spectrum calculations can be done using either the standard RADIS
databank loader or using databank that has been preprocessed and saved in numpy
array (`npy`) format. In case the standard loader is used for loading the data
for GPU-powered spectrum calculation, some preprocessing is done on that data
before the spectrum calculation begins.
# TODO: perform timing test to see how much time calculating log_2gs separately takes

Currently, GPU-powered spectra calculations are supported only at thermal equilibrium
and therefore, the method to calculate the spectra has been named :py:func:`~radis.lbl.calc.eq_spectrum_gpu`.
In order to use this method to calculate the spectra, follow the same steps as in the
case of a normal equilibrium spectra, and if using :py:func:`~radis.lbl.calc.calc_spectrum`
function set the parameter `mode` to `gpu`, or use :py:func:`~radis.lbl.calc.eq_spectrum_gpu`

Consider the following example which demonstrates the above information::

    from radis import SpectrumFactory
    from radis.test.utils import getTestFile
    T = 1000
    p = 0.1
    wstep = 0.001
    wmin = 2200  # cm-1
    wmax = 2400  # cm-1
    sf = SpectrumFactory(
            wavenum_min=wmin,
            wavenum_max=wmax,
            mole_fraction=1,
            path_length=1,  # doesnt change anything
            wstep=wstep,
            pressure=p,
            isotope="1",
            chunksize="DLM"
        )
    sf.load_databank(getTestFile("cdsd_hitemp_09_fragment.txt"), format="cdsd-hitemp", parfuncfmt="hapi")
    s_gpu = sf.eq_spectrum_gpu(Tgas=T)

Alternatively, one could compute the spectra with the assistance of GPU using the
following code as well ::

    s = calc_spectrum(
        	wavenum_min=1900,
        	wavenum_max=2300,
        	Tgas=700,
        	path_length=0.1,
        	mole_fraction=0.01,
        	isotope=1,
        	mode='gpu'
    		)

As mentioned previously, the GPU-enabled spectrum calculations can also be done
using databank that has been preprocessed and saved in numpy's `npy` format.

In order to calculate the data using the `npy` files, first place all the 7 files in the
same directory. Then, set the `databank` parameter of :py:func:`~radis.lbl.calc.calc_spectrum`
to point to one of the 7 files in the directory. The program will automatically detect and read the
other files present in the same folder ::

      s = calc_spectrum(
        	wavenum_min=1900,
        	wavenum_max=2300,
        	Tgas=700,
        	path_length=0.1,
        	databank='/path/to/v0.npy',
        	mole_fraction=0.01,
        	isotope=1,
        	mode='gpu'
    		)

## TODO: Once the npy2df implementation is complete, also mention about loading the npy files
from different directories by passing a dictionary instead

The `npy` files that are needed for calculating the spectra on GPU can be extracted
from any databank and stored in the following format. The name of the file is written
first, followed by the physical quantity it stores and it's name and position in the
CDSD-4000 database.

`v0.npy`: wavenumber in vacuum; `v0`, line[3:15]
`da.npy`: air-pressure induced shift; `d_air`, line[59:67]
`El.npy`: low-state energy; `Elow`, line[45:55]
`na.npy`: temperature dependence exponent for air; `n_air`, line[55:59]

In addition to the above 4 quantities, we also need 3 more quantities which are not
directly stored in the databank. They are explained below:

`log_2gs.npy`: np.log(2*gs), where `gs` is HITRAN/HITEMP HWHM pressure broadening constant
for self-broadening; `gamma_self`, line[40:45]
`log_2vMm.npy`: np.log(2*v0) + 0.5*np.log(2*k*np.log(2)/(c**2*Mm)), where `v0` is the
wavenumber in vacuum, `k` is Boltzmann's constant, `c` is speed of light in vacuum
and `Mm` is the molecular mass of gas molecule in kilogram.
`S0.npy`: f_ab * gu * A21 / (8*pi*c_cm*v0**2) where,
`f_ab`: np.array([ 0.98420, 0.01106, 0.0039471])[iso.astype(int)-1],
`gu`: 2*Ju + 1, where
`Ju` = Jl + DJ, where
`DJ` = ord(line[117:118])-ord('Q')
`Jl` = int(line[118:121])
`A21` is the Einstein's coefficient, line[25:35]
`c_cm` is speed of light in vacuum in centimeters/second,
`v0` is wavenumber in vacuum.

In order to facililate the conversion of data from the CDSD-4000 par format to the format explained
above, users can use the scripts present in `/radis/misc/prepare-npy-data`.

`par2npy.py` extracts the relevant information from the dataset files and stores them in `npy` files
where each file contains all the information for multiple lines.

`reshape_arrays.py` extracts and separates the different fields for each line, and saves the values of
a specific field for all the lines in a separate file as explained above, e.g. `v0.npy`, 'da.npy`, etc.

The Spectrum Factory
--------------------

Most RADIS calculations can be done using the :py:func:`~radis.lbl.calc.calc_spectrum` function.
Advanced examples require to use the :py:class:`~radis.lbl.factory.SpectrumFactory`
class, which is the core of RADIS line-by-line calculations.
:py:func:`~radis.lbl.calc.calc_spectrum` is a wrapper to :py:class:`~radis.lbl.factory.SpectrumFactory`
for the simple cases.

The :py:class:`~radis.lbl.factory.SpectrumFactory` allows you to :

- calculate multiple spectra (batch processing) with a same line database
- edit the line database manually
- have access to intermediary calculation variables
- connect to a database of precomputed spectra on your computer

To use the :py:class:`~radis.lbl.factory.SpectrumFactory`, first
load your own line database with :py:meth:`~radis.lbl.loader.DatabankLoader.load_databank`,
and then calculate several spectra in batch using
:py:meth:`~radis.lbl.factory.SpectrumFactory.eq_spectrum` and
:py:meth:`~radis.lbl.factory.SpectrumFactory.non_eq_spectrum`,
and :py:mod:`~astropy.units` ::

    import astropy.units as u
    from radis import SpectrumFactory
    sf = SpectrumFactory(wavelength_min=4165 * u.nm,
                         wavelength_max=4200 * u.nm,
                         path_length=0.1 * u.m,
                         pressure=20 * u.mbar,
                         molecule='CO2',
                         isotope='1,2',
                         cutoff=1e-25,              # cm/molecule
                         broadening_max_width=10,   # cm-1
                         )
    sf.load_databank('HITRAN-CO2-TEST')        # this database must be defined in ~/radis.json
    s1 = sf.eq_spectrum(Tgas=300 * u.K)
    s2 = sf.eq_spectrum(Tgas=2000 * u.K)
    s3 = sf.non_eq_spectrum(Tvib=2000 * u.K, Trot=300 * u.K)

.. _label_lbl_config_file:

Configuration file
------------------

The ``~/radis.json`` configuration file is used to store the list and attributes of the Line databases
available on your computer.

Without a configuration file, you can still:

- download the corresponding [HITRAN-2016]_ line database automatically,
  either with the (default) ``databank='hitran'`` option in :py:func:`~radis.lbl.calc.calc_spectrum`,
  or the :py:meth:`~radis.lbl.loader.DatabankLoader.fetch_databank` method of
  :py:class:`~radis.lbl.factory.SpectrumFactory`,
- give a single file as an input to the ``databank=`` parameter of :py:func:`~radis.lbl.calc.calc_spectrum`

A configuration file will help to:

- handle line databases that contains multiple files
- use custom tabulated partition functions for equilibrium calculations
- use custom, precomputed energy levels for nonequilibrium calculations

.. note::

    it is also possible to give :py:meth:`~radis.lbl.loader.DatabankLoader.load_databank` the line database path,
    format, and partition function format directly, but this is not recommended and should only be used if for some
    reason you cannot create a configuration file.

A ``~/radis.json`` is user-dependant, and machine-dependant. It contains a list of database, everyone of which
is specific to a given molecule. It typically looks like::

str: Typical expected format of a ~/radis.json entry::

    {
      "database": {                                     # database key: all databanks information are stored in this key
          "MY-HITEMP-CO2": {                            # your databank name: use this in calc_spectrum()
                                                        # or SpectrumFactory.load_databank()
            "path": [                                   # no "", multipath allowed
                "D:\\Databases\\HITEMP-CO2\\hitemp_07",
                "D:\\Databases\\HITEMP-CO2\\hitemp_08",
                "D:\\Databases\\HITEMP-CO2\\hitemp_09"
            ],
            "format": "hitran",                         # 'hitran' (HITRAN/HITEMP), 'cdsd-hitemp', 'cdsd-4000'
                                                        # databank text file format. More info in
                                                        # SpectrumFactory.load_databank function.
            "parfuncfmt": "hapi"                        # calculate partition functions
          }
      }
    }

Following is an example where the path variable uses a wildcard ``*`` to find all the files that have ``hitemp_*`` in their names::

    {
      "database": {                                     # database key: all databanks information are stored in this key
          "MY-HITEMP-CO2": {                            # your databank name: use this in calc_spectrum()
                                                        # or SpectrumFactory.load_databank()
            "path": "D:\\Databases\\HITEMP-CO2\\hitemp_*",   # To load all hitemp files directly
            "format": "hitran",                         # 'hitran' (HITRAN/HITEMP), 'cdsd-hitemp', 'cdsd-4000'
                                                        # databank text file format. More info in
                                                        # SpectrumFactory.load_databank function.
            "parfuncfmt": "hapi"                        # calculate partition functions
          }
      }
    }


In the former example, for equilibrium calculations, RADIS uses [HAPI]_ to retrieve
partition functions tabulated with TIPS-2017. It is also possible to use your own
partition functions, for instance::

    {
      "database": {                                       # database key: all databanks information are stored in this key
          "MY-HITEMP-CO2": {                              # your databank name: use this in calc_spectrum()
                                                          # or SpectrumFactory.load_databank()
            "path": [                                     # no "", multipath allowed
                "D:\\Databases\\HITEMP-CO2\\hitemp_07",
                "D:\\Databases\\HITEMP-CO2\\hitemp_08",
                "D:\\Databases\\HITEMP-CO2\\hitemp_09"
            ],
            "format": "hitran",                           # 'hitran' (HITRAN/HITEMP), 'cdsd-hitemp', 'cdsd-4000'
                                                          # databank text file format. More info in
                                                          # SpectrumFactory.load_databank function.
            "parfuncfmt": "cdsd",                         # 'cdsd', 'hapi', etc.
                                                          # format to read tabulated partition function
                                                          # file. If `hapi`, then HAPI (HITRAN Python
                                                          # interface) is used to retrieve them (valid if
                                                          # your databank is HITRAN data). HAPI is embedded
                                                          # into RADIS. Check the version. If not specified then 'hapi'
                                                          # is used as default
            "parfunc": "PATH/TO/cdsd_partition_functions.txt"
                                                          # path to tabulated partition function to use.
                                                          # If `parfuncfmt` is `hapi` then `parfunc`
                                                          # should be the link to the hapi.py file. If
                                                          # not given, then the hapi.py embedded in RADIS
                                                          # is used (check version)
          }
      }
    }

By default, for nonequilibrium calculations, RADIS built-in :ref:`spectroscopic constants <label_db_spectroscopic_constants>`
are used to calculate the energy levels for CO2.
It is also possible to use your own Energy level database. For instance::


    {
      "database": {                                        # database key: all databanks information are stored in this key
          "MY-HITEMP-CO2": {                               # your databank name: use this in calc_spectrum()
                                                           # or SpectrumFactory.load_databank()
            "path": [                                      # no "", multipath allowed
                "D:\\Databases\\HITEMP-CO2\\hitemp_07",
                "D:\\Databases\\HITEMP-CO2\\hitemp_08",
                "D:\\Databases\\HITEMP-CO2\\hitemp_09"
            ],
            "format": "hitran",                             # 'hitran' (HITRAN/HITEMP), 'cdsd-hitemp', 'cdsd-4000'
                                                            # databank text file format. More info in
                                                            # SpectrumFactory.load_databank function.
                                                            # is used (check version)
            "levels_iso1": "D:\\PATH_TO\\energies_of_626_isotope.levels",
            "levels_iso2": "D:\\PATH_TO\\energies_of_636_isotope.levels",
            "levelsfmt": "cdsd",                            # 'cdsd', etc.
                                                            # how to read the previous file. Default None.
            "levelszpe": "2531.828"                         # zero-point-energy (cm-1): offset for all level
                                                            # energies. Default 0 (if not given)
          }
      }
    }

The full description of a `~/radis.json` entry is given in :py:data:`~radis.misc.config.DBFORMAT`:

- ``path`` corresponds to Line databases (here: downloaded from [HITEMP-2010]_) and the ``levels_iso``
  are user generated Energy databases (here: calculated from the [CDSD-4000]_ Hamiltonian on non-distributed code,
  which takes into account non diagonal coupling terms).

- ``format`` is the databank text file format. It can be one of ``'hitran'`` (for HITRAN / HITEMP 2010),
  ``'cdsd-hitemp'`` and ``'cdsd-4000'`` for the different CDSD versions (for CO2 only). See full list in
  :py:data:`~radis.lbl.loader.KNOWN_DBFORMAT`.

- ``parfuncfmt``: ``cdsd``, ``hapi`` is the format of the tabulated partition functions used.
  If ``'hapi'``, then [HAPI]_ is used to retrieve them (valid if your databank is HITRAN data).
  See full list in :py:data:`~radis.lbl.loader.KNOWN_PARFUNCFORMAT`

- ``parfunc`` is the path to the tabulated partition function to use in in equilibrium calculations
  (:py:meth:`~radis.lbl.factory.SpectrumFactory.eq_spectrum`). If ``parfuncfmt`` is ``'hapi'`` then `parfunc` should be
  the link to the hapi.py file. If not given, then the :py:mod:`~radis.io.hitran.hapi` embedded in RADIS
  is used (check version)

- ``levels_iso#`` are the path to the energy levels to use for each isotope, which are needed for
  nonequilibrium calculations (:py:meth:`~radis.lbl.factory.SpectrumFactory.non_eq_spectrum`).

- ``levelsfmt`` is the energy levels database format. Typically, ``'radis'``, and various implementation of [CDSD-4000]_
  nonequilibrium partitioning of vibrational and rotational energy: ``'cdsd-pc'``, ``'cdsd-pcN'``, ``'cdsd-hamil'``.
  See full list in :py:data:`~radis.lbl.loader.KNOWN_LVLFORMAT`

*How to create the configuration file?*

A default ``~/radis.json`` configuration file can be generated with :py:func:`~radis.test.utils.setup_test_line_databases`, which
creates two test databases from fragments of [HITRAN-2016]_ line databases::

    from radis.test.utils import setup_test_line_databases
    setup_test_line_databases()

which will create a ``~/radis.json`` file with the following content ::


    {
      "database": {
          "HITRAN-CO2-TEST": {
            "info": "HITRAN 2016 database, CO2, 1 main isotope (CO2-626), bandhead: 2380-2398 cm-1 (4165-4200 nm)",
            "path": "PATH_TO\\radis\\radis\\test\\files\\hitran_co2_626_bandhead_4165_4200nm.par",
            "format": "hitran",
            "parfuncfmt": "hapi",
            "levelsfmt": "radis"
          },
          "HITRAN-CO-TEST": {
            "info": "HITRAN 2016 database, CO, 3 main isotopes (CO-26, 36, 28), 2000-2300 cm-1",
            "path": "PATH_TO\\radis\\radis\\test\\files\\hitran_co_3iso_2000_2300cm.par",
            "format": "hitran",
            "parfuncfmt": "hapi",
            "levelsfmt": "radis"
          },
          "HITEMP-CO2-TEST": {
            "info": "HITEMP-2010, CO2, 3 main isotope (CO2-626, 636, 628), 2283.7-2285.1 cm-1",
            "path": "PATH_TO\\radis\\radis\\test\\files\\cdsd_hitemp_09_fragment.txt",
            "format": "cdsd-hitemp",
            "parfuncfmt": "hapi",
            "levelsfmt": "radis"
          }
      }
    }


If you configuration file exists already, the test databases will simply be appended.


Advanced
========

Calculation Flow Chart
----------------------

Refer to :ref:`Architecture <label_dev_architecture>` for an overview of how equilibrium
and nonequilibrium calculations are conducted.

.. _label_lbl_custom_constants:

Use Custom Spectroscopic constants
----------------------------------

Spectroscopic constants are a property of the RADIS :py:class:`~radis.db.classes.ElectronicState`
class. All molecules are stored in the :py:class:`~radis.db.molecules.Molecules` dictionary.
You need to update this dictionary before running your calculation in order to use your
own spectroscopic constants.

An example of how to use your own spectroscopic constants::

    from radis import calc_spectrum
    from radis.db.molecules import Molecules, ElectronicState

    Molecules['CO2'][1]['X'] = ElectronicState('CO2', isotope=1, state='X', term_symbol='1Σu+',
                                spectroscopic_constants='my_constants.json',  # <<< YOUR FILE HERE
                                spectroscopic_constants_type='dunham',
                                Ediss=44600,
                                )
    s = calc_spectrum(...)



Vibrational bands
-----------------

To calculate vibrational bands of a given spectrum separately, use the
:meth:`~radis.lbl.bands.BandFactory.eq_bands` and  :meth:`~radis.lbl.bands.BandFactory.non_eq_bands`
methods. See the :py:func:`~radis.test.lbl.test_bands.test_plot_all_CO2_bandheads` example in
``radis/test/lbl/test_bands.py`` for more information.


Connect to a Spectrum Database
------------------------------

In RADIS, the same code can be used to retrieve precomputed spectra if they exist,
or calculate them and store them if they don't. See :ref:`Precompute Spectra <label_lbl_precompute_spectra>`



.. _label_lbl_performance:

Performance
===========

RADIS is very optimized, making use of C-compiled libraries (NumPy, Numba) for computationally intensive steps,
and data analysis libraries (Pandas) to handle lines databases efficiently.
Additionaly, different strategies and parameters are used to improve performances further:

Line Database Reduction Strategies
----------------------------------

By default:

- *linestrength cutoff* : lines with low linestrength are discarded after the new
  populations are calculated.
  Parameter: :py:attr:`~radis.lbl.loader.Input.cutoff`
  (see the default value in the arguments of :py:meth:`~radis.lbl.factory.SpectrumFactory.eq_spectrum`)

Additional strategies (deactivated by default):

- *weak lines* (pseudo-continuum): lines which are close to a much stronger line are called weak lines.
  They are added to a pseudo-continuum and their lineshape is calculated with a simple
  rectangular approximation.
  See the default value in the arguments of :py:attr:`~radis.lbl.loader.Parameters.pseudo_continuum_threshold`
  (see arguments of :py:meth:`~radis.lbl.factory.SpectrumFactory.eq_spectrum`)


Lineshape optimizations
-----------------------

Lineshape convolution is usually the performance bottleneck in any line-by-line code.

Two approaches can be used:

- improve the convolution efficiency. This involves using an efficient convolution algorithm,
  using a reduced convolution kernel, analytical approximations, or multiple spectral grid.
- reduce the number of convolutions (for a given number of lines): this is done using the DLM strategy.

RADIS implements the two approaches as well as various strategies and parameters
to calculate the lineshapes efficiently.

- *broadening width* : lineshapes are calculated on a reduced spectral range.
  Voigt computation calculation times scale linearly with that parameter.
  Gaussian x Lorentzian calculation times scale as a square with that parameter.
  parameters: broadening_max_width

- *Voigt approximation* : Voigt is calculated with an analytical approximation.
  Parameter : :py:attr:`~radis.lbl.loader.Parameters.broadening_max_width` and
  default values in the arguments of :py:meth:`~radis.lbl.factory.SpectrumFactory.eq_spectrum`.
  See :py:func:`~radis.lbl.broadening.voigt_lineshape`.

- *Fortran precompiled* : previous Voigt analytical approximation is
  precompiled in Fortran to improve performance times. This is always the
  case and cannot be changed on the user side at the moment. See the source code
  of :py:func:`~radis.lbl.broadening.voigt_lineshape`.

- *Multiple spectral grids* : many LBL codes use different spectral grids to
  calculate the lineshape wings with a lower resolution. This strategy is not
  implemented in RADIS.

- *DLM* :  lines are projected on a Lineshape database to reduce the number of calculated
  lineshapes from millions to a few dozens.
  With this optimization strategy, the lineshape convolution becomes almost instantaneous
  and all the other strategies are rendered useless. Projection of all lines on the lineshape
  database becomes the performance bottleneck.
  parameters: :py:attr:`~radis.lbl.loader.Parameters.dlm_res_L`,
  :py:attr:`~radis.lbl.loader.Parameters.dlm_res_G`.
  (this is the default strategy implemented in RADIS). Learn more in [Spectral-Synthesis-Algorithm]_

More details on the parameters below:

Computation parameters
----------------------

If performance is an issue (for instance when calculating polyatomic spectra on large spectral ranges), you
may want to tweak the computation parameters in :py:func:`~radis.lbl.calc.calc_spectrum` and
:py:class:`~radis.lbl.factory.SpectrumFactory`. In particular, the parameters that have the highest
impact on the calculation performances are:

- The ``broadening_max_width``, which defines the spectral range over which the broadening is calculated.
- The linestrength ``cutoff``, which defines which low intensity lines should be discarded. See
  :meth:`~radis.lbl.base.BaseFactory.plot_linestrength_hist` to choose a correct cutoff.

Check the [RADIS-2018]_ article for a quantitative assessment of the influence of the different parameters.

Other strategies are possible, such as calculating the weak lines in a pseudo-continuum. This can
result in orders of magnitude improvements in computation performances.:

- The ``pseudo_continuum_threshold`` defines which treshold should be used.

See the :py:func:`~radis.test.lbl.test_broadening.test_abscoeff_continuum` case in ``radis/test/lbl/test_broadening.py``
for an example, which can be run with (you will need the CDSD-HITEMP database installed) ::

    pytest radis/test/lbl/test_broadening.py -m "test_abscoeff_continuum"


Database loading
----------------

Line database can be a performance bottleneck, especially for large polyatomic molecules in the [HITEMP-2010]_
or [CDSD-4000]_ databases.
Line database files are automatically cached by RADIS under a ``.h5`` format after they are loaded the first time.
If you want to deactivate this behaviour, use ``use_cached=False`` in :py:func:`~radis.lbl.calc.calc_spectrum`,
or ``db_use_cached=False, lvl_use_cached=False`` in :py:class:`~radis.lbl.factory.SpectrumFactory`.

You can also use :py:meth:`~radis.lbl.loader.DatabankLoader.init_databank` instead of the default
:py:meth:`~radis.lbl.loader.DatabankLoader.load_databank`. The former will save the line database parameter,
and only load them if needed. This is useful if used in conjonction with
:py:meth:`~radis.lbl.loader.DatabankLoader.init_database`, which will retrieve precomputed spectra from
a database if they exist.


Manipulate the database
-----------------------

If for any reason, you want to manipulate the line database manually (for instance, keeping only lines emitting
by a particular level), you need to access the :py:attr:`~radis.lbl.loader.DatabankLoader.df0` attribute of
:py:class:`~radis.lbl.factory.SpectrumFactory`.

.. warning::

    never overwrite the ``df0`` attribute, else some metadata may be lost in the process. Only use inplace operations.

For instance::

    sf = SpectrumFactory(
        wavenum_min= 2150.4,
        wavenum_max=2151.4,
        pressure=1,
        isotope=1)
    sf.load_databank('HITRAN-CO-TEST')
    sf.df0.drop(sf.df0[sf.df0.vu!=1].index, inplace=True)   # keep lines emitted by v'=1 only
    sf.eq_spectrum(Tgas=3000, name='vu=1').plot()

:py:attr:`~radis.lbl.loader.DatabankLoader.df0` contains the lines as they are loaded from the database.
:py:attr:`~radis.lbl.loader.DatabankLoader.df1` is generated during the spectrum calculation, after the
line database reduction steps, population calculation, and scaling of intensity and broadening parameters
with the calculated conditions.

Parallelization
---------------

Note : internal CPU-parallelization was discarded in RADIS 0.9.28, as not efficient
enough with the new lineshape algorithms implemented with radis==0.9.20.

A much faster GPU-parallelization is available for equilibrium calculation::

    SpectrumFactory.eq_spectrum(mode='gpu')


GPU accelerated spectrum calculation
------------------------------------

Apart from the parallelization method mentioned above, RADIS also supports CUDA-native parallel computation, specifically
for lineshape calculation and broadening. To use these GPU-accelerated methods to compute the spectra, use either :py:func:`~radis.lbl.calc.calc_spectrum`
function with parameter `mode` set to `gpu`, or :py:func:`~radis.lbl.calc.eq_spectrum_gpu`. In order to use these methods,
ensure that your system has an Nvidia GPU with compute capability of atleast 3.0 and CUDA Toolkit 8.0 or above. Refer to
:ref:`GPU Spectrum Calculation on RADIS <label_radis_gpu>` to see how to setup your system to run GPU accelerated spectrum
calculation methods, examples and performance tests.

Profiler
--------

You may want to track where the calculation is taking some time.
You can set ``verbose=2`` to print the time spent on different operations. Example::

    s = calc_spectrum(1900, 2300,         # cm-1
                      molecule='CO',
                      isotope='1,2,3',
                      pressure=1.01325,   # bar
                      Tvib=1000,          # K
                      Trot=300,           # K
                      mole_fraction=0.1,
                      verbose=2,
                      )

::

    >>> ...
    >>> Fetching vib / rot energies for all 749 transitions
    >>> Fetched energies in 0s
    >>> Calculate weighted transition moment
    >>> Calculated weighted transition moment in 0.0
    >>> Calculating nonequilibrium populations
    >>> sorting lines by vibrational bands
    >>> lines sorted in 0.0s
    >>> Calculated nonequilibrium populations in 0.1s
    >>> scale nonequilibrium linestrength
    >>> scaled nonequilibrium linestrength in 0.0s
    >>> calculated emission integral
    >>> calculated emission integral in 0.0s
    >>> Applying linestrength cutoff
    >>> Applied linestrength cutoff in 0.0s (expected time saved ~ 0.0s)
    >>> Calculating lineshift
    >>> Calculated lineshift in 0.0s
    >>> Calculate broadening FWHM
    >>> Calculated broadening FWHM in 0.0s
    >>> Calculating line broadening (695 lines: expect ~ 0.1s on 1 CPU)
    >>> Calculated line broadening in 0.1s
    >>> process done in 0.4s
    >>> ...

.. _label_lbl_precompute_spectra:

Precompute Spectra
------------------

See :py:meth:`~radis.lbl.loader.DatabankLoader.init_database`, which is the direct integration
of :py:class:`~radis.tools.database.SpecDatabase` in a :py:class:`~radis.lbl.factory.SpectrumFactory`



.. |badge_gitter| image:: https://badges.gitter.im/Join%20Chat.svg
                  :target: https://gitter.im/radis-radiation/community
                  :alt: Gitter

.. |badge_slack| image:: https://img.shields.io/badge/slack-join-green.svg?logo=slack
                  :target: https://radis.github.io/slack-invite/
                  :alt: Slack

