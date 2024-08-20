# Psi4 Codebase Overview
The main location for all files exists under the `./psi4/` directory 
and then is largely split into two categories:
1. `./psi4/driver/` - The main driver for the program. This is where all
    of the top-level python functions and logic exists. While the main
    computationally intensive code is written in C++, the driver handles
    most of the logic for the program. Improvements in abstracting the C++ code,
    has enabled entire modules to be written purely in python but still largely
    having the performance of C++ (e.g. SAPT(DFT) module located `./psi4/driver/procrouting/sapt_proc.py`).
2. `./psi4/src/` - The main location for all C++ code. This is where the
    majority of the computational code exists. 

## Python Driver
The python driver is the main entry point for the program. 

One of the most directories is `./psi4/driver/procrouting/` which contains most
of the logic for setting up new methods and making them accessible through
drivers like `psi4.energy()`, `psi4.gradient()`, `psi4.optimize()`,
`psi4.hessian()` and `psi4.properties()`. 

### `./psi4/driver/procrouting/proc_table.py`
This file provides a dictionary of all available procedures in the program.
Here is a snippet from the file:

```py
procedures = {
    'energy': {
        'hf'            : proc.run_scf,
        'scf'           : proc.run_scf,
        'mcscf'         : proc.run_mcscf,
        'dct'           : proc.run_dct,
        'ep2'           : proc.run_dfep2,
        'mp2'           : proc.select_mp2,
        ...
        'sapt(dft)'     : sapt.run_sapt_dft,
        ...
    },
    'gradient' : {
        'hf'            : proc.select_scf_gradient,
        'scf'           : proc.select_scf_gradient,
        'cc2'           : proc.select_cc2_gradient,
        'ccsd'          : proc.select_ccsd_gradient,
        'ccsd(t)'       : proc.select_ccsd_t__gradient,
        ...
    },
    ...
}
```
The layout effectively maps the driver+method to a specific function to call
from `./psi4/driver/procrouting/proc.py`. 

### `./psi4/driver/procrouting/proc.py`

This file contains the logic for the functions that are called from the
`proc_table.py` file. One of the most common functions in any quantum chemistry
program is the self-consistent-field (SCF) routine (HF & DFT). Below is a
snippet from the `run_scf` function in `proc.py` (from psi4=1.10.0dev). You
will notice that the function is largely a sequence of calls to other functions
in the driver and lots of logic for handling different options (see below for
more discussion on creating new psi4 options). Note: whenever you see `core` in
the code, it is referring to the C++ codebase that are made accessible in
python through [pybind11](https://github.com/pybind/pybind11) in the 
build process.

```py
...

def run_scf(name, **kwargs):
    """Function encoding sequence of PSI module calls for
    a self-consistent-field theory (HF & DFT) calculation.
    """
    optstash_mp2 = p4util.OptionsState(
        ['DF_BASIS_MP2'],
        ['DFMP2', 'MP2_OS_SCALE'],
        ['DFMP2', 'MP2_SS_SCALE'])

    dft_func = False
    if "dft_functional" in kwargs:
        dft_func = True

    optstash_scf = proc_util.scf_set_reference_local(name, is_dft=dft_func)

    # See if we're doing TDSCF after, keep JK if so
    if sum(core.get_option("SCF", "TDSCF_STATES")) > 0:
        core.set_local_option("SCF", "SAVE_JK", True)

    # Alter default algorithm
    if not core.has_global_option_changed('SCF_TYPE'):
        core.set_global_option('SCF_TYPE', 'DF')

    scf_wfn = scf_helper(name, post_scf=False, **kwargs)
    returnvalue = scf_wfn.energy()

    ssuper = scf_wfn.functional()

    if ssuper.is_c_hybrid():

        # throw exception for CONV/CD MP2
        if (mp2_type := core.get_global_option("MP2_TYPE")) != "DF":
            raise ValidationError(f"Invalid MP2 type {mp2_type} for DF-DFT energy. See capabilities Table.")

        core.tstart()
        aux_basis = core.BasisSet.build(scf_wfn.molecule(), "DF_BASIS_MP2",
                                        core.get_option("DFMP2", "DF_BASIS_MP2"),
                                        "RIFIT", core.get_global_option('BASIS'),
                                        puream=-1)
        scf_wfn.set_basisset("DF_BASIS_MP2", aux_basis)
        if ssuper.is_c_scs_hybrid():
            core.set_local_option('DFMP2', 'MP2_OS_SCALE', ssuper.c_os_alpha())
            core.set_local_option('DFMP2', 'MP2_SS_SCALE', ssuper.c_ss_alpha())
            dfmp2_wfn = core.dfmp2(scf_wfn)
            dfmp2_wfn.compute_energy()

            vdh = dfmp2_wfn.variable('CUSTOM SCS-MP2 CORRELATION ENERGY')

        else:
            dfmp2_wfn = core.dfmp2(scf_wfn)
            dfmp2_wfn.compute_energy()
            vdh = ssuper.c_alpha() * dfmp2_wfn.variable('MP2 CORRELATION ENERGY')

        # remove misleading MP2 psivars computed with DFT, not HF, reference
        for var in dfmp2_wfn.variables():
            if var.startswith('MP2 ') and ssuper.name() not in ['MP2D']:
                scf_wfn.del_variable(var)

        scf_wfn.set_variable("DOUBLE-HYBRID CORRECTION ENERGY", vdh)  # P::e SCF
        scf_wfn.set_variable("{} DOUBLE-HYBRID CORRECTION ENERGY".format(ssuper.name()), vdh)
        returnvalue += vdh
        scf_wfn.set_variable("DFT TOTAL ENERGY", returnvalue)  # P::e SCF
        for pv, pvv in scf_wfn.variables().items():
            if pv.endswith('DISPERSION CORRECTION ENERGY') and pv.startswith(ssuper.name()):
                fctl_plus_disp_name = pv.split()[0]
                scf_wfn.set_variable(fctl_plus_disp_name + ' TOTAL ENERGY', returnvalue)
                break
        else:
            scf_wfn.set_variable('{} TOTAL ENERGY'.format(ssuper.name()), returnvalue)

        scf_wfn.set_variable('CURRENT ENERGY', returnvalue)
        scf_wfn.set_energy(returnvalue)
        core.print_out('\n\n')
        core.print_out('    %s Energy Summary\n' % (name.upper()))
        core.print_out('    ' + '-' * (15 + len(name)) + '\n')
        core.print_out('    DFT Reference Energy                  = %22.16lf\n' % (returnvalue - vdh))
        core.print_out('    Scaled MP2 Correlation                = %22.16lf\n' % (vdh))
        core.print_out('    @Final double-hybrid DFT total energy = %22.16lf\n\n' % (returnvalue))
        core.tstop()

        if ssuper.name() == 'MP2D':
            for pv, pvv in dfmp2_wfn.variables().items():
                scf_wfn.set_variable(pv, pvv)

            # Conversely, remove DFT qcvars from MP2D
            for var in scf_wfn.variables():
                if 'DFT ' in var or 'DOUBLE-HYBRID ' in var:
                    scf_wfn.del_variable(var)

            # DFT groups dispersion with SCF. Reshuffle so dispersion with MP2 for MP2D.
            for pv in ['SCF TOTAL ENERGY', 'SCF ITERATION ENERGY', 'MP2 TOTAL ENERGY']:
                scf_wfn.set_variable(pv, scf_wfn.variable(pv) - scf_wfn.variable('DISPERSION CORRECTION ENERGY'))

            scf_wfn.set_variable('MP2D CORRELATION ENERGY', scf_wfn.variable('MP2 CORRELATION ENERGY') + scf_wfn.variable('DISPERSION CORRECTION ENERGY'))
            scf_wfn.set_variable('MP2D TOTAL ENERGY', scf_wfn.variable('MP2D CORRELATION ENERGY') + scf_wfn.variable('HF TOTAL ENERGY'))
            scf_wfn.set_variable('CURRENT ENERGY', scf_wfn.variable('MP2D TOTAL ENERGY'))
            scf_wfn.set_variable('CURRENT CORRELATION ENERGY', scf_wfn.variable('MP2D CORRELATION ENERGY'))
            scf_wfn.set_variable('CURRENT REFERENCE ENERGY', scf_wfn.variable('SCF TOTAL ENERGY'))

    # Shove variables into global space
    for k, v in scf_wfn.variables().items():
        core.set_variable(k, v)

    optstash_scf.restore()
    optstash_mp2.restore()
    return scf_wfn
...

```

Hence, if you are adding a new method to the program, you will need to
minimally select a driver name and method to add to the `proc_table.py` file
and then create a new function in the `proc.py` file. Largely `proc.py` is for
logic of handling options and making calls to other functions in the driver or
core. Alternatively, if you are writing the module in mostly python, you could
have the `proc_table.py` call a function in a different file like in
the SAPT(DFT) case where `'sapt(dft)'     : sapt.run_sapt_dft,` allowing you to
avoid `proc.py`.

## C++ Source Code

One of the most important files in the C++ side is `./psi4/src/read_options.cc`
where psi4 options are initialized with default values and acceptable
alternative values.

```cpp

int read_options(const std::string &name, Options &options, bool suppress_printing) {
    // ...
    options.add("DOCC", new ArrayType());
    options.add_int("NUM_FROZEN_DOCC", 0);
    options.add_str("FREEZE_CORE", "FALSE", "FALSE TRUE 1 0 -1 -2 -3 POLICY");
    options.add_bool("PUREAM", true);
    options.add_str_i("WRITER_FILE_LABEL", "");
    options.add_double("CUBEPROP_ISOCONTOUR_THRESHOLD", 0.85);
    // ...
    if (name == "SAPT" || options.read_globals()) {
        // ...
        options.add_str("SAPT_LEVEL", "SAPT0", "SAPT0 SAPT2 SAPT2+ SAPT2+3");
        options.add_bool("SAPT0_E10", false);
        // ...
    }
    // ...
}

```

<!-- continue... -->

