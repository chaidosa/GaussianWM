# HPC Conda Setup Notes

These notes capture the Conda/NFS setup path that worked on the HPC cluster. They are meant as a reproducible alternative to the default `uv sync` flow in `README.md`.

## 1. Create a Clean Conda Environment

Use a path-based Conda environment if your cluster keeps environments under a shared location:

```bash
conda deactivate
conda create --prefix /nfs/hpc/share/vermapri/anaconda/envs/envs/GaussianWM python=3.10 pip
conda activate /nfs/hpc/share/vermapri/anaconda/envs/envs/GaussianWM
```

Install `uv` into the new environment:

```bash
python -m pip install uv
```

## 2. Export Requirements From uv

On this cluster, `uv pip install -e .` can hang during the wheel installation phase when installing into a Conda environment on NFS. Use `uv` only to export the locked requirements, then use `pip` for installation.

```bash
cd /nfs/hpc/share/vermapri/Research-Projects/GaussianWM

export UV_CACHE_DIR=/tmp/$USER/uv-cache
export UV_LOCK_TIMEOUT=1200
export UV_LINK_MODE=copy

uv export --format requirements.txt --no-hashes --no-emit-project -o /tmp/$USER/gwm-requirements.txt
```

Patch `open3d` for Python 3.10 on this Linux environment:

```bash
sed -i 's/^open3d==0.19.0/open3d==0.18.0/' /tmp/$USER/gwm-requirements.txt
```

## 3. Install Python Dependencies

Install the exported requirements with pip. The extra PyTorch index is needed for CUDA 12.1 wheels.

```bash
python -m pip install --no-cache-dir -r /tmp/$USER/gwm-requirements.txt --extra-index-url https://download.pytorch.org/whl/cu121
python -m pip install -e . --no-deps
```

Verify the core imports:

```bash
python -c "import torch; print(torch.__version__, torch.cuda.is_available())"
python -c "import gaussianwm; print('gaussianwm ok')"
python -c "import open3d; print('open3d ok')"
```

`torch.cuda.is_available()` may be `False` on a login/submit node. Test CUDA availability inside a GPU job before treating that as an error.

## 4. Build CUDA Extension Packages

The rasterization package needs a CUDA toolkit at build time. Load CUDA before installing it:

```bash
module avail cuda
module load cuda/12.1

which nvcc
nvcc -V

export CUDA_HOME=$(dirname $(dirname $(which nvcc)))
export PATH=$CUDA_HOME/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH
```

Then install the extra packages from the project README:

```bash
export TORCH_CUDA_ARCH_LIST="8.0"
python -m pip install git+https://github.com/dcharatan/diff-gaussian-rasterization-modified --no-build-isolation
```

Set `TORCH_CUDA_ARCH_LIST` for the GPU type on the node. Common values are `7.0` for V100, `8.0` for A100, `8.6` for RTX 3090/A6000, `8.9` for L40/L40S, and `9.0` for H100. If the rasterizer build fails with `IndexError: list index out of range` inside `_get_cuda_arch_flags`, this variable was missing or PyTorch could not see a GPU during compilation.

Do not install PyTorch3D with `conda install -c pytorch3d pytorch3d` in this environment unless you are also rebuilding the entire PyTorch/CUDA stack with Conda. It can pull in Conda CUDA libraries that conflict with the pip-installed PyTorch CUDA wheels.

For PyTorch3D, make sure the active C++ compiler is GCC 9 or newer. If the build fails with `You're trying to build PyTorch with a too old version of GCC`, load a newer GCC module and point the build at it:

```bash
module avail gcc
module load gcc/11

which gcc
which g++
gcc --version
g++ --version

export CC=$(which gcc)
export CXX=$(which g++)
python -m pip install ninja

export TMPDIR=/tmp/$USER/pip-build
mkdir -p "$TMPDIR"
export MAX_JOBS=4

python -m pip install git+https://github.com/facebookresearch/pytorch3d.git --no-build-isolation --no-cache-dir
```

Verify:

```bash
python -c "import diff_gaussian_rasterization; print('diff gaussian ok')"
python -c "import pytorch3d; print('pytorch3d ok')"
```

If `CUDA_HOME environment variable is not set`, reload the CUDA module and re-run the `CUDA_HOME` export above. Some clusters expose `nvcc` only inside an interactive GPU job.

## 5. Optional Splatt3r Setup

This checkout expects the Splatt3r code at `third_party/splatt3r`, but the directory may be missing after a normal clone. From the project root, try initializing submodules first:

```bash
git submodule update --init --recursive
```

If that does not create `third_party/splatt3r`, clone it directly:

```bash
mkdir -p third_party
git clone https://github.com/btsmart/splatt3r third_party/splatt3r
```

Then compile the optional CUDA kernels:

```bash
cd third_party/splatt3r/src/mast3r_src/dust3r/croco/models/curope/
python setup.py build_ext --inplace
cd ../../../../../../../..
```

Download the Splatt3r checkpoint:

```bash
mkdir -p third_party/splatt3r/checkpoints/splatt3r_v1.0
cd third_party/splatt3r/checkpoints/splatt3r_v1.0
wget https://huggingface.co/brandonsmart/splatt3r_v1.0/resolve/main/epoch%3D19-step%3D1200.ckpt
cd ../../../../..
```

## Troubleshooting

If `uv` reports a stale cache lock on NFS, use a local cache:

```bash
export UV_CACHE_DIR=/tmp/$USER/uv-cache
export UV_LOCK_TIMEOUT=1200
export UV_LINK_MODE=copy
```

If an interrupted install leaves missing `METADATA` files in `site-packages`, the Conda environment is partially corrupted. Recreate the environment and use the `uv export` plus `pip install` flow above. Avoid running `uv pip install -e .` again in the NFS Conda environment.

If a terminal appears stuck, inspect from another session:

```bash
ps -u "$USER" -o pid,ppid,stat,etime,cmd | grep -E 'uv|python|pip'
```

`[uv] <defunct>` entries are already-dead zombie processes. They are not active installs.
