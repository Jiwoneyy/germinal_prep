**Germinal** Setup & Run codes

Other links worth looking into:

https://www.biorxiv.org/content/10.1101/2024.09.19.613838v1
https://github.com/TencentAI4S/IgGM

**Logins:**
3TB (1-8): ssh -i .ssh/id_ed25519 ubuntu@185.216.22.4
2TB (9-16): ssh -i .ssh/id_ed25519 ubuntu@38.128.232.237

**Download:**
3TB (1-8): sftp -i .ssh/id_ed25519 ubuntu@185.216.22.4
2TB (9-16): sftp -i .ssh/id_ed25519 ubuntu@38.128.232.237

OR, faster:

rsync -avzP -e "ssh -i ~/.ssh/id_ed25519" ubuntu@185.216.22.4:~/germinal/logs ~/Documents/Peptobiotics/Bioinformatics/EHP/logs/20251029_run
rsync -avzP -e "ssh -i ~/.ssh/id_ed25519" ubuntu@38.128.232.237:~/germinal/logs ~/Documents/Peptobiotics/Bioinformatics/EHP/logs/20251029_run


get -R germinal/logs ./Documents/Peptobiotics/Bioinformatics/EHP/logs/

**//Germinal Install**

wget "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh"

bash Miniforge3-$(uname)-$(uname -m).sh
exec bash

**//Run**

conda create --name germinal python=3.10
conda activate germinal

**//Run**

git clone https://github.com/SantiagoMille/germinal
cd germinal

pip install uv
uv pip install pandas matplotlib numpy biopython scipy seaborn tqdm ffmpeg py3dmol \
  chex dm-haiku dm-tree joblib ml-collections immutabledict optax cvxopt mdtraj colabfold

uv pip install -e colabdesign
uv pip install pyrosetta-installer
python -c 'import pyrosetta_installer; pyrosetta_installer.install_pyrosetta()'
uv pip install iglm torchvision==0.21.* chai-lab==0.6.1 \
  torch==2.6.* torchaudio==2.6.* torchtyping==0.1.5 torch_geometric==2.6.*
uv pip install -e .
uv pip install jax==0.5.3
uv pip install dm-haiku==0.0.13 
uv pip install hydra-core omegaconf
uv pip install "jax[cuda12_pip]==0.5.3" -f https://storage.googleapis.com/jax-releases/jax_cuda_releases.html
uv pip install aria2
cd params
aria2c -x 16 https://storage.googleapis.com/alphafold/alphafold_params_2022-12-06.tar
tar -xf alphafold_params_2022-12-06.tar -C .
cd ..

python validate_install.py

**//Prep files**

rm -rf germinal_prep
git clone https://github.com/Jiwoneyy/germinal_prep

rm -rf ./germinal/filters/chai.py
mv germinal_prep/chai.py germinal/filters/chai.py

rm -rf configs pdbs
mv germinal_prep/configs/ configs
mv germinal_prep/pdbs pdbs

sudo rm -rf results logs
mkdir results
mkdir logs

**//Docker install & run (chai)**

docker build -t germinal .

tmux new -s germinal
Ctrl+B, then D
tmux attach -t germinal

cd ~/germinal
docker run -it --rm --gpus all \
  -v "$PWD/results:/workspace/results" \
  -v "$PWD/pdbs:/workspace/pdbs" \
  -v "$PWD/configs:/workspace/configs" \
  -v "$PWD/logs:/workspace/logs" \
  -v "$PWD/germinal/filters:/workspace/germinal/filters" \
  germinal bash

**//Singularity install & run (af3)**

sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository -y ppa:apptainer/ppa
sudo apt update
sudo apt install -y apptainer

singularity pull germinal.sif docker://jwang003/germinal:latest
singularity exec germinal.sif python -c "print('ok')"

singularity shell --nv \
  --bind "$PWD/results:/workspace/results" \
  --bind "$PWD/logs:/workspace/logs"  \
  --bind "$PWD/configs:/workspace/configs" \
  --bind "$PWD/pdbs:/workspace/pdbs" \
  --pwd /workspace \
  germinal.sif

**//GPU assign & code to run**
for i in {0..7}; do
n=$((i+1))
CUDA_VISIBLE_DEVICES=$i python run_germinal.py target=AQP$n max_passing_designs=15 experiment_name=AQP$n > logs/AQP$n.out 2>&1 &
done

for i in {0..7}; do
n=$((i+9))
CUDA_VISIBLE_DEVICES=$i python run_germinal.py target=AQP$n max_passing_designs=15 experiment_name=AQP$n > logs/AQP$n.out 2>&1 &
done

pkill -f run_germinal.py










