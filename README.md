# <span style="font-size:larger;">MACE</span>

[![GitHub release](https://img.shields.io/github/release/ACEsuit/mace.svg)](https://GitHub.com/ACEsuit/mace/releases/)
[![Paper](https://img.shields.io/badge/Paper-NeurIPs2022-blue)](https://openreview.net/forum?id=YPpSngE-ZU)
[![License](https://img.shields.io/badge/License-MIT%202.0-blue.svg)](https://opensource.org/licenses/mit)
[![GitHub issues](https://img.shields.io/github/issues/ACEsuit/mace.svg)](https://GitHub.com/ACEsuit/mace/issues/)
[![Documentation Status](https://readthedocs.org/projects/mace/badge/)](https://mace-docs.readthedocs.io/en/latest/)
[![DOI](https://zenodo.org/badge/505964914.svg)](https://doi.org/10.5281/zenodo.14103332)

## Table of contents

- [MACE](#mace)
  - [Table of contents](#table-of-contents)
  - [About MACE](#about-mace)
  - [Documentation](#documentation)
  - [Installation](#installation)
    - [pip installation](#installation-from-pypi)
    - [pip installation from source](#installation-from-source)
  - [Usage](#usage)
    - [Training](#training)
    - [Evaluation](#evaluation)
  - [Tutorials](#tutorials)
  - [CUDA acceleration with cuEquivariance](#cuda-acceleration-with-cuequivariance)
  - [Weights and Biases for experiment tracking](#weights-and-biases-for-experiment-tracking)
  - [Pretrained Foundation Models](#pretrained-foundation-models)
    - [MACE-MP: Materials Project Force Fields](#mace-mp-materials-project-force-fields)
      - [Example usage in ASE](#example-usage-in-ase)
    - [MACE-OFF: Transferable Organic Force Fields](#mace-off-transferable-organic-force-fields)
      - [Example usage in ASE](#example-usage-in-ase-1)
    - [Finetuning foundation models](#finetuning-foundation-models)
    - [Latest recommended foundation models](#latest-recommended-foundation-models)
  - [Caching](#caching)
  - [Development](#development)
  - [References](#references)
  - [Contact](#contact)
  - [License](#license)

## About MACE

MACE provides fast and accurate machine learning interatomic potentials with higher order equivariant message passing.

This repository contains the MACE reference implementation developed by
Ilyes Batatia, Gregor Simm, David Kovacs, and the group of Gabor Csanyi, and friends (see Contributors).

Also available:

- [MACE in JAX](https://github.com/ACEsuit/mace-jax), currently about 2x times faster at evaluation, but training is recommended in Pytorch for optimal performances.
- [MACE layers](https://github.com/ACEsuit/mace-layer) for constructing higher order equivariant graph neural networks for arbitrary 3D point clouds.

## Documentation

A partial documentation is available at: https://mace-docs.readthedocs.io

## Installation

### 1. Requirements

- Python >= 3.7  (for openMM, use Python = 3.9)
- [PyTorch](https://pytorch.org/) >= 1.12 **(training with float64 is not supported with PyTorch 2.1 but is supported with 2.2 and later, Pytorch 2.4.1 is not supported)**

**Make sure to install PyTorch.** Please refer to the [official PyTorch installation](https://pytorch.org/get-started/locally/) for the installation instructions. Select the appropriate options for your system.

### Installation from PyPI

This is the recommended way to install MACE.

```sh
pip install --upgrade pip
pip install mace-torch
```

**Note:** The homonymous package on [PyPI](https://pypi.org/project/MACE/) has nothing to do with this one.

### Installation from source

```sh
git clone https://github.com/ACEsuit/mace.git
pip install ./mace
```

## Usage

### Training

To train a MACE model, you can use the `mace_run_train` script, which should be in the usual place that pip places binaries (or you can explicitly run `python3 <path_to_cloned_dir>/mace/cli/run_train.py`)

```sh
mace_run_train \
    --name="MACE_model" \
    --train_file="train.xyz" \
    --valid_fraction=0.05 \
    --test_file="test.xyz" \
    --config_type_weights='{"Default":1.0}' \
    --E0s='{1:-13.663181292231226, 6:-1029.2809654211628, 7:-1484.1187695035828, 8:-2042.0330099956639}' \
    --model="MACE" \
    --hidden_irreps='128x0e + 128x1o' \
    --r_max=5.0 \
    --batch_size=10 \
    --max_num_epochs=1500 \
    --stage_two \
    --start_stage_two=1200 \
    --ema \
    --ema_decay=0.99 \
    --amsgrad \
    --restart_latest \
    --device=cuda \
```

To give a specific validation set, use the argument `--valid_file`. To set a larger batch size for evaluating the validation set, specify `--valid_batch_size`.

To control the model's size, you need to change `--hidden_irreps`. For most applications, the recommended default model size is `--hidden_irreps='256x0e'` (meaning 256 invariant messages) or `--hidden_irreps='128x0e + 128x1o'`. If the model is not accurate enough, you can include higher order features, e.g., `128x0e + 128x1o + 128x2e`, or increase the number of channels to `256`. It is also possible to specify the model using the     `--num_channels=128` and `--max_L=1`keys.

It is usually preferred to add the isolated atoms to the training set, rather than reading in their energies through the command line like in the example above. To label them in the training set, set `config_type=IsolatedAtom` in their info fields. If you prefer not to use or do not know the energies of the isolated atoms, you can use the option `--E0s="average"` which estimates the atomic energies using least squares regression. Note that using fitted E0s corresponds to fitting the deviations of the atomic energies from the average, rather than fitting the atomization energy (which is the case when using isolated-atom E0s), and this will most likely result in less stable potentials for molecular dynamics applications.

If the keyword `--stage_two` (previously called swa) is enabled, the energy weight of the loss is increased for the last ~20% of the training epochs (from `--start_stage_two` epochs). This setting usually helps lower the energy errors.

The precision can be changed using the keyword `--default_dtype`, the default is `float64` but `float32` gives a significant speed-up (usually a factor of x2 in training).

The keywords `--batch_size` and `--max_num_epochs` should be adapted based on the size of the training set. The batch size should be increased when the number of training data increases, and the number of epochs should be decreased. An heuristic for initial settings, is to consider the number of gradient update constant to 200 000, which can be computed as $\text{max-num-epochs}*\frac{\text{num-configs-training}}{\text{batch-size}}$.

The code can handle training set with heterogeneous labels, for example containing both bulk structures with stress and isolated molecules. In this example, to make the code ignore stress on molecules, append to your molecules configuration a `config_stress_weight = 0.0`.

By default, a figure displaying the progression of loss and RMSEs during training, along with a scatter plot of the model's inferences on the train, validation, and test sets, will be generated in the results folder at the end of training. This can be disabled using `--plot False`. To track these metrics throughout training (excluding inference on the test set), you can enable periodic plotting for the train and validation sets by specifying `--plot_frequency N`, which updates the plots every Nth epoch.

#### Apple Silicon GPU acceleration

To use Apple Silicon GPU acceleration make sure to install the latest PyTorch version and specify `--device=mps`.

#### Multi-GPU training

For multi-GPU training, use the `--distributed` flag. This will use PyTorch's DistributedDataParallel module to train the model on multiple GPUs. Combine with on-line data loading for large datasets (see below). An example slurm script can be found in `mace/scripts/distributed_example.sbatch`.

#### YAML configuration

Option to parse all or some arguments using a YAML is available. For example, to train a model using the arguments above, you can create a YAML file `your_configs.yaml` with the following content:

```yaml
name: nacl
seed: 2024
train_file: train.xyz
stage_two: yes
start_stage_two: 1200
max_num_epochs: 1500
device: cpu
test_file: test.xyz
E0s:
  41: -1029.2809654211628
  38: -1484.1187695035828
  8: -2042.0330099956639
config_type_weights:
  Default: 1.0

```

And append to the command line `--config="your_configs.yaml"`. Any argument specified in the command line will overwrite the one in the YAML file.

### Evaluation

To evaluate your MACE model on an XYZ file, run the `mace_eval_configs`:

```sh
mace_eval_configs \
    --configs="your_configs.xyz" \
    --model="your_model.model" \
    --output="./your_output.xyz"
```

## Tutorials

You can run our [Colab tutorial](https://colab.research.google.com/drive/1D6EtMUjQPey_GkuxUAbPgld6_9ibIa-V?authuser=1#scrollTo=Z10787RE1N8T) to quickly get started with MACE.

We also have a more detailed Colab tutorials on:

- [Introduction to MACE training and evaluation](https://colab.research.google.com/drive/1ZrTuTvavXiCxTFyjBV4GqlARxgFwYAtX)
- [Introduction to MACE active learning and fine-tuning](https://colab.research.google.com/drive/1oCSVfMhWrqHTeHbKgUSQN9hTKxLzoNyb)
- [MACE theory and code (advanced)](https://colab.research.google.com/drive/1AlfjQETV_jZ0JQnV5M3FGwAM2SGCl2aU)

## CUDA acceleration with cuEquivariance

MACE supports CUDA acceleration with the cuEquivariance library. To install the library and use the acceleration, see our documentation at https://mace-docs.readthedocs.io/en/latest/guide/cuda_acceleration.html.

## On-line data loading for large datasets

If you have a large dataset that might not fit into the GPU memory it is recommended to preprocess the data on a CPU and use on-line dataloading for training the model. To preprocess your dataset specified as an xyz file run the `preprocess_data.py` script. An example is given here:

```sh
mkdir processed_data
python ./mace/scripts/preprocess_data.py \
    --train_file="/path/to/train_large.xyz" \
    --valid_fraction=0.05 \
    --test_file="/path/to/test_large.xyz" \
    --atomic_numbers="[1, 6, 7, 8, 9, 15, 16, 17, 35, 53]" \
    --r_max=4.5 \
    --h5_prefix="processed_data/" \
    --compute_statistics \
    --E0s="average" \
    --seed=123 \
```

To see all options and a little description of them run `python ./mace/scripts/preprocess_data.py --help` . The script will create a number of HDF5 files in the `processed_data` folder which can be used for training. There will be one folder for training, one for validation and a separate one for each `config_type` in the test set. To train the model use the `run_train.py` script as follows:

```sh
python ./mace/scripts/run_train.py \
    --name="MACE_on_big_data" \
    --num_workers=16 \
    --train_file="./processed_data/train.h5" \
    --valid_file="./processed_data/valid.h5" \
    --test_dir="./processed_data" \
    --statistics_file="./processed_data/statistics.json" \
    --model="ScaleShiftMACE" \
    --num_interactions=2 \
    --num_channels=128 \
    --max_L=1 \
    --correlation=3 \
    --batch_size=32 \
    --valid_batch_size=32 \
    --max_num_epochs=100 \
    --stage_two \
    --start_stage_two=60 \
    --ema \
    --ema_decay=0.99 \
    --amsgrad \
    --error_table='PerAtomMAE' \
    --device=cuda \
    --seed=123 \
```

## Weights and Biases for experiment tracking

If you would like to use MACE with Weights and Biases to log your experiments simply install with

```sh
pip install ./mace[wandb]
```

And specify the necessary keyword arguments (`--wandb`, `--wandb_project`, `--wandb_entity`, `--wandb_name`, `--wandb_log_hypers`)

## Pretrained Foundation Models

We provide a series of pretrained foundation models for various applications. These models can be used directly for inference, or as a starting point for fine-tuning on a new dataset.
Foundation models are a rapidly evolving field. Please look at the [MACE-MP GitHub repository](https://github.com/ACEsuit/mace-foundations/releases) and the [MACE-OFF23 GitHub repository](https://github.com/ACEsuit/mace-off/releases) for the latest releases.

### Latest Recommended Foundation Models

| Model Name           | Elements Covered | Training Dataset | Level of Theory     | Target System     | Model Size                                                                                                                                                                                                                                                                                                                                                                        | GitHub Release | Notes                                                              | License |
| -------------------- | ---------------- | ---------------- | ------------------- | ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------- | ------------------------------------------------------------------ | ------- |
| MACE-MP-0a           | 89               | MPTrj            | DFT (PBE+U)         | Materials         | [small](https://github.com/ACEsuit/mace-foundations/releases/download/mace_mp_0/2023-12-10-mace-128-L0_energy_epoch-249.model), [medium](https://github.com/ACEsuit/mace-foundations/releases/download/mace_mp_0/2023-12-03-mace-128-L1_epoch-199.model), [large](https://github.com/ACEsuit/mace-foundations/releases/download/mace_mp_0/2024-01-07-mace-128-L2_epoch-199.model) | >=v0.3.6       | Initial release of foundation model.                               | MIT     |
| MACE-MP-0b3          | 89               | MPTrj            | DFT (PBE+U)         | Materials         | [medium](https://github.com/ACEsuit/mace-foundations/releases/download/mace_mp_0b3/mace-mp-0b3-medium.model)                                                                                                                                                                                                                                                                      | >=v0.3.10      | Improved high pressure stability and reference energies.           | MIT     |
| MACE-MPA-0           | 89               | MPTrj + sAlex    | DFT (PBE+U)         | Materials         | [medium-mpa-0](https://github.com/ACEsuit/mace-foundations/releases/download/mace_mpa_0/mace-mpa-0-medium.model)                                                                                                                                                                                                                                                                  | >=v0.3.10      | Improved accuracy for materials, improved high pressure stability. | MIT     |
| MACE-OMAT-0          | 89               | OMAT             | DFT (PBE+U) VASP 54 | Materials         | [medium-omat-0](https://github.com/ACEsuit/mace-foundations/releases/download/mace_omat_0/mace-omat-0-medium.model)                                                                                                                                                                                                                                                               | >=v0.3.10      |                                                                    | ASL     |
| MACE-OFF23           | 10               | SPICE v1         | DFT (wB97M+D3)      | Organic Chemistry | [small](https://github.com/ACEsuit/mace-off/blob/main/mace_off23/MACE-OFF23_small.model), [medium](https://github.com/ACEsuit/mace-off/blob/main/mace_off23/MACE-OFF23_medium.model), [large](https://github.com/ACEsuit/mace-off/blob/main/mace_off23/MACE-OFF23_large.model)                                                                                                    | >=v0.3.6       | Initial release covering neutral organic chemistry.                | ASL     |
| MACE-MATPES-PBE-0    | 89               | MATPES-PBE       | DFT (PBE)           | Materials         | [medium](https://github.com/ACEsuit/mace-foundations/releases/download/mace_matpes_0/MACE-matpes-pbe-omat-ft.model)                                                                                                                                                                                                                                                               | >=v0.3.10      | No +U correction.                                                  | ASL     |
| MACE-MATPES-r2SCAN-0 | 89               | MATPES-r2SCAN    | DFT (r2SCAN)        | Materials         | [medium](https://github.com/ACEsuit/mace-foundations/releases/download/mace_matpes_0/MACE-matpes-r2scan-omat-ft.model)                                                                                                                                                                                                                                                            | >=v0.3.10      | Better functional for materials.                                   | ASL     |

### MACE-MP: Materials Project Force Fields

We have collaborated with the Materials Project (MP) to train a universal MACE potential covering 89 elements on 1.6 M bulk crystals in the [MPTrj dataset](https://figshare.com/articles/dataset/23713842) selected from MP relaxation trajectories.
The models are releaed on GitHub at https://github.com/ACEsuit/mace-foundations.
If you use them please cite [our paper](https://arxiv.org/abs/2401.00096) which also contains an large range of example applications and benchmarks.

> [!CAUTION]
> The MACE-MP models are trained on MPTrj raw DFT energies from VASP outputs, and are not directly comparable to the MP's DFT energies or CHGNet's energies, which have been applied MP2020Compatibility corrections for some transition metal oxides, fluorides (GGA/GGA+U mixing corrections), and 14 anions species (anion corrections). For more details, please refer to the [MP Documentation](https://docs.materialsproject.org/methodology/materials-methodology/thermodynamic-stability/thermodynamic-stability/anion-and-gga-gga+u-mixing) and [MP2020Compatibility.yaml](https://github.com/materialsproject/pymatgen/blob/master/pymatgen/entries/MP2020Compatibility.yaml).

#### Example usage in ASE

```py
from mace.calculators import mace_mp
from ase import build

atoms = build.molecule('H2O')
calc = mace_mp(model="medium", dispersion=False, default_dtype="float32", device='cuda')
atoms.calc = calc
print(atoms.get_potential_energy())
```

### MACE-OFF: Transferable Organic Force Fields

There is a series (small, medium, large) transferable organic force fields. These can be used for the simulation of organic molecules, crystals and molecular liquids, or as a starting point for fine-tuning on a new dataset. The models are released under the [ASL license](https://github.com/gabor1/ASL).
The models are releaed on GitHub at https://github.com/ACEsuit/mace-off.
If you use them please cite [our paper](https://arxiv.org/abs/2312.15211) which also contains detailed benchmarks and example applications.

#### Example usage in ASE

```py
from mace.calculators import mace_off
from ase import build

atoms = build.molecule('H2O')
calc = mace_off(model="medium", device='cuda')
atoms.calc = calc
print(atoms.get_potential_energy())
```

### Finetuning foundation models

To finetune one of the mace-mp-0 foundation model, you can use the `mace_run_train` script with the extra argument `--foundation_model=model_type`. For example to finetune the small model on a new dataset, you can use:

```sh
mace_run_train \
  --name="MACE" \
  --foundation_model="small" \
  --train_file="train.xyz" \
  --valid_fraction=0.05 \
  --test_file="test.xyz" \
  --energy_weight=1.0 \
  --forces_weight=1.0 \
  --E0s="average" \
  --lr=0.01 \
  --scaling="rms_forces_scaling" \
  --batch_size=2 \
  --max_num_epochs=6 \
  --ema \
  --ema_decay=0.99 \
  --amsgrad \
  --default_dtype="float32" \
  --device=cuda \
  --seed=3
```

Other options are "medium" and "large", or the path to a foundation model.
If you want to finetune another model, the model will be loaded from the path provided `--foundation_model=$path_model`, all the hypers will be extracted automatically.

## Caching

By default automatically downloaded models, like mace_mp, mace_off and data for fine tuning, end up in `~/.cache/mace`. The path can be changed by using
the environment variable XDG_CACHE_HOME. When set, the new cache path expands to $XDG_CACHE_HOME/.cache/mace

## Development

This project uses [pre-commit](https://pre-commit.com/) to execute code formatting and linting on commit.
We also use `black`, `isort`, `pylint`, and `mypy`.
We recommend setting up your development environment by installing the `dev` packages
into your python environment:

```bash
pip install -e ".[dev]"
pre-commit install
```

The second line will initialise `pre-commit` to automaticaly run code checks on commit.
We have CI set up to check this, but we _highly_ recommend that you run those commands
before you commit (and push) to avoid accidentally committing bad code.

We are happy to accept pull requests under an [MIT license](https://choosealicense.com/licenses/mit/). Please copy/paste the license text as a comment into your pull request.

## References

If you use this code, please cite our papers:

```bibtex
@inproceedings{Batatia2022mace,
  title={{MACE}: Higher Order Equivariant Message Passing Neural Networks for Fast and Accurate Force Fields},
  author={Ilyes Batatia and David Peter Kovacs and Gregor N. C. Simm and Christoph Ortner and Gabor Csanyi},
  booktitle={Advances in Neural Information Processing Systems},
  editor={Alice H. Oh and Alekh Agarwal and Danielle Belgrave and Kyunghyun Cho},
  year={2022},
  url={https://openreview.net/forum?id=YPpSngE-ZU}
}

@misc{Batatia2022Design,
  title = {The Design Space of E(3)-Equivariant Atom-Centered Interatomic Potentials},
  author = {Batatia, Ilyes and Batzner, Simon and Kov{\'a}cs, D{\'a}vid P{\'e}ter and Musaelian, Albert and Simm, Gregor N. C. and Drautz, Ralf and Ortner, Christoph and Kozinsky, Boris and Cs{\'a}nyi, G{\'a}bor},
  year = {2022},
  number = {arXiv:2205.06643},
  eprint = {2205.06643},
  eprinttype = {arxiv},
  doi = {10.48550/arXiv.2205.06643},
  archiveprefix = {arXiv}
 }
```

## Contact

If you have any questions, please contact us at ilyes.batatia@ens-paris-saclay.fr.

For bugs or feature requests, please use [GitHub Issues](https://github.com/ACEsuit/mace/issues).

## License

The MACE code is published and distributed under the [MIT License](MIT.md). (Note that some of the models linked above come with different licenses).
