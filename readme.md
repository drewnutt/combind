# Fork

This is a fork of ComBind that is removing all uses of all proprietary API calls
or proprietary software. This will produce similar (if not the exact same) results
as the original ComBind and it will be completely free to use.

This fork will focus on the use of [Gnina](https://github.com/gnina/gnina) as the
docking software rather than Glide. Mostly because Glide is proprietary, but also because
Gnina is a deep-learning based docking pipeline.

# ComBind

ComBind integrates data-driven modeling and physics-based docking for
improved binding pose prediction and binding affinity prediction.

Given the chemical structures of several ligands that can bind
a given target protein, ComBind solves for a set of poses, one per ligand, that
are both highly scored by physics-based docking and display similar interactions
with the target protein. ComBind quantifies this vague notion of "similar" by
considering a diverse training set of protein complexes and computing the
overlap between protein–ligand interactions formed by distinct ligands when
they are in their correct poses, as compared to when they are in randomly
selected poses. To predict binding affinities, poses are predicted for
the known binders using ComBind, and then the candidate molecule is scored
according to the ComBind score w.r.t. the selected poses.

## Predicting poses for known binders

First, see instructuctions for software installation at the bottom of this page.

Running ComBind can be broken into several components: data curation,
data preparation (including docking), featurization of docked poses,
and the ComBind scoring itself.

Note that if you already have docked poses for your molecules of interest, you
can proceed to the featurization step. If you are knowledgable about your target
protein, you may well be able to get better docking results by manually
preparing the data than would be obtained using the automated procedure
implemented here.

### Curation of raw data

To produce poses for a particular protein, you'll need to provide a 3D structure
of the target protein and chemical structures of ligands to dock.

These raw inputs need to be properly stored so that the rest of the pipeline
can recognize them.

The structure(s) should be stored in a directory `structures/raw`.
Each structure should be split into two files `NAME_prot.pdb` and `NAME_lig.pdb`
containing only the protein and only the ligand, respectively.

If you'd prefer to prepare your structures yourself, save your
prepared files to `structures/proteins` and `structures/ligands`. Moreover,
you could even just begin with a GNINA docking template file (i.e. `gnina -r <path_to_receptor_file> --autobox_ligand <path_to_crystal_ligand> `)
placed on the first line of a file.

Ligands can be specified in a csv file with a header line containing at
least the entries "ID" and "SMILES", specifying the ligand name and the ligand
chemical structure. Alternatively you can specify your ligands with a sdf
containing an entry for each ligand.

### Data preparation and docking

Use the following command to prepare the structural data using [ProDy](https://github.com/prody/ProDy), 
align the structures to each other, and produce a docking template line.

```
python combind structprep
```

In parallel, you can prepare the ligand data using the following command.
By default, the ligands will be written to separate files (one ligand per file).
You can specify the `--multiplex` flag to write all of the ligands to the same
file.

```
python combind ligprep ligands.csv
```

Once the GNINA template file and ligand data have been prepared, you can run the
docking. The arguments to the dock command are a list of ligand files to be
docked. By default, the GNINA template file is the alphabetically first template present
in `structures/template`; use the `--template` option to specify a different template. Additionally,
you can utilize `--slurm` to create a tarball of all of the necessary files and update the docking `.txt`
file to use paths in the tarball.

```
python combind dock ligands/*/*.sdf
```

### Featurization

```
python combind featurize features docking/*/*.sdf.gz
```

### Pose prediction with ComBind

```
python combind pose-prediction features poses.csv
```

Optionally, you can extract the poses selected by ComBind to a single file.
The resulting file will contain the protein structure followed by one pose (the
one selected by ComBind) for each ligand.

```
python combind extract-top-poses poses.csv docking/*/*.sdf.gz
```

## ComBindVS

To run virtual screening using ComBindVS, you must begin with a structure of the
target protein, a set of helper ligands, and a library of compounds to screen.

The first two steps, which can be done in parallel, are to determine poses for
the helper ligands using ComBind and to produce an initial set of docked poses
for the library to be screened. Then, ComBindVS can be 

### Use ComBind to solve for poses of a set of helper ligands

Use ComBind to predict poses for the known binders and extract the selected
poses to a single file, as described above. In the below, we'll assume that this
file is named `helpers_pv.maegz`

### Dock the library to be screened

The library to be screened can be docked the same way as described above,
but here it is highly recommended that you use the `--multiplex` option during
ligprep (to write all the compounds to one file) and the `--screen` option
during docking, which will limit the number of poses per compound to 30 and
not used enhanced pose sampling.

```
combind ligprep library.csv --multiplex
combind dock ligands/library/library.maegz --screen
```

### ComBindVS
![#f03c15](https://via.placeholder.com/15/f03c15/000000?text=+)![#f03c15](https://via.placeholder.com/15/f03c15/000000?text=+) I have not yet implemented open source CombindVS ![#f03c15](https://via.placeholder.com/15/f03c15/000000?text=+)![#f03c15](https://via.placeholder.com/15/f03c15/000000?text=+)

To compute the ComBind scores for each pose, we need to compute the pairwise]
features between each candidate pose to the helper ligand poses.

```
combind featurize --no-mcss --screen --max-poses 100000 features_screen docking/library-to-grid/library-to-grid_pv.maegz helpers_pv.maegz
```

With these features in hand, you can then compute the ComBind scores. The ComBind
scores for each pose will be written to the indicated numpy file (here screen.npy).

```
combind screen screen.npy features_screen
```

It is often convenient to apply the scores to the original poseviewer file and
use existing schrodinger utilities to sort the results.

```
combind apply-scores docking/library-to-grid/library-to-grid_pv.maegz screen.npy combind_scores_added_pv.maegz
$SCHRODINGER/utilities/glide_sort -best_by_title -use_prop_d r_i_combind_score -o combind_pv.maegz combind_scores_added_pv.maegz
```

## Benchmarking data

See `stats_data/pdbs_for_benchmark.csv` for a list of PDBs used for benchmarking
ComBind. The "query" column gives the PDB for the ligand being docked, the
"grid" column gives the structure the query is docked to, and the "mcss<0.5"
column indicates whether the query ligand shares a common substructure with 
the co-crystal ligand in the structure being docked to.

See `stats_data/structures.tar.gz` for the raw structural data used for
benchmarking ComBind.

See `stats_data/helper_best_affinity_diverse.csv` and `stats_data/helper_best_mcss.csv`
for a list of the "helper ligands" used when benchmarking ComBind. Each row
lists a query ligand and one helper ligand; all the entries for each query ligand
should be aggregrated. (Most query ligands have 20 associated helper ligands.)

## Installation

Start by cloning this git repository.

ComBind requires access to [Gnina](https://github.com/gnina/gnina), [ProDy](https://github.com/prody/ProDy),
[OpenBabel](https://openbabel.org/wiki/Main_Page), and [RDKit](https://github.com/rdkit/rdkit).

To setup the environment before each use, run
`source setup.sh` to set combind specific environmental variables.
