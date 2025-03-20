# Jasmine cutsom accre_dwi_connectomes
Code to run MRtrix3 with Singularity 3.4 (https://sylabs.io/guides/3.4/user-guide/) container on Vanderbilt's ACCRE.

Make sure Singularity 3.7.3 (sams as ACCRE version at the time) is installed locally to build singularity and installed on ACCRE so that we can run singularity there. 

This will create the following outputs based on Desikan-Killiany + custom atlases atlas:
SIFT2-weighted connectome, 
FA connectome, 
diff_atlas_JWJ_registered2DWI.nii,
5TT, 
T1 registered to DWI

The input argument to the singularity 'exec' call in the .slurm file dictates how many times MRTrix's 'tckgen' will be called and thus how many redundant connectomes will be generated. Multiple connectomes are generated in attamp to cpature stochasticity of probablistic tractography. 

0) Ensure that your local singularity version is the same version as on ACCRE

0.1) Can use a virtual environment (e.g. conda)

0.1.1) conda create -n hypo python=3.7 anaconda

0.1.2) https://sylabs.io/guides/3.7/user-guide/quick_start.html (troubleshooting with Go version: https://stackoverflow.com/questions/63865962/singularity-3-6-2-installation, Golang install: https://golangci-lint.run/usage/install/#other-ci )

0.1.3) Use these custom steps (as of March 2022) to install correct version of Go/Singularity

sudo apt-get update && \
sudo apt-get install -y build-essential \
libseccomp-dev pkg-config squashfs-tools cryptsetup

sudo rm -r /usr/local/go

export VERSION=1.13.15 OS=linux ARCH=amd64  # change this as you need

wget -O /tmp/go${VERSION}.${OS}-${ARCH}.tar.gz https://dl.google.com/go/go${VERSION}.${OS}-${ARCH}.tar.gz && \
sudo tar -C /usr/local -xzf /tmp/go${VERSION}.${OS}-${ARCH}.tar.gz

echo 'export GOPATH=${HOME}/go' >> ~/.bashrc && \
echo 'export PATH=/usr/local/go/bin:${PATH}:${GOPATH}/bin' >> ~/.bashrc && \
source ~/.bashrc

curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin v1.44.2

mkdir -p ${GOPATH}/src/github.com/sylabs && \
cd ${GOPATH}/src/github.com/sylabs && \
git clone https://github.com/sylabs/singularity.git && \
cd singularity

git checkout v3.7.3

cd ${GOPATH}/src/github.com/sylabs/singularity && \
./mconfig && \
cd ./builddir && \
make && \
sudo make install

singularity version

1) Need to add 'inputs' folder and 'outputs' folder to the project directory (see the .txt files in /code for input and output subdirectory setup). Also add a /code/logs directory. 

2) Each input directory must have the following:

t1.nii.gz
dwmri.bval
dwmri.bvec
dwmri.nii.gz
diff_atlas_JWJ.nii (DK is from 'recon-all':https://surfer.nmr.mgh.harvard.edu/, the rest are custom generated)

3) Build Singularity:
Required files (should all be in GitHub repository listed in recipe file):
ROBEX/
main.py
main.sh

Example build call (change the directories). This assumes you have the required files/directories in the github repository listed in the recipe file
$ sudo singularity build jasmine_recipe_dwi_connectomes_v1.0.simg jasmine_recipe_dwi_connectomes_v1.0.txt

4) Run Singularity Image (locally to check if it works, kill it after one iteration)
$ singularity exec -e --contain -B /tmp:/tmp -B /home/graham/PROJECTS/jasmine_hypothalamus/inputs/108996:/INPUTS -B /home/graham/PROJECTS/jasmine_hypothalamus/outputs/108996:/OUTPUTS /home/graham/PROJECTS/jasmine_hypothalamus/singularity/jasmine_recipe_dwi_connectomes_v1.0.simg bash /CODE/main.sh 1 8 1

5) Copy to ACCRE
$ scp dwi_connectomes user@login.accre.vanderbilt.edu:/scratch/user/projects/

6) Check if it works on ACCRE default gateway
$ singularity exec -e --contain -B /tmp:/tmp -B /scratch/user/projects/dwi_connectomes/inputs/subdir1/:/INPUTS -B /scratch/user/projects/dwi_connectomes/outputs/subdir1/:/OUTPUTS /scratch/user/projects/dwi_connectomes/singularity/dwi_connectomes_v1.simg bash /CODE/main.sh 10

7) Submit slurm 
check all absolute paths in dwi_connectomes.slurm and the input_dir_list.txt and output_dir_list.txt are all correct.
check the permissions of all of the paths are 755
$ sbatch dwi_connectomes.slurm



TROUBLESHOOTING
1) Jobs failing on ACCRE because /OUTPUTS is 'read-only'
You will get this error if there is anything wrong with the path -B bound to /OUTPUTS
Be very careful with the .txt directory lists. To avoid ACCRE jobs failing, I have had to: 
'cat' the .txt file on ACCRE
copy the list
rename the file to OLD
'touch' new .txt files
copy the list and save 


