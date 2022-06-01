# bc_rcc_rstudio_server
RStudio Server app for Open OnDemand

Based on https://github.com/OSC/bc_osc_rstudio_server.

# UPDATE
If you are running RStudio Server 1.4 or above, use the https://github.com/mcw-rcc/bc_rcc_rstudio_server/tree/rstudio-1.4-updates branch.

# Description
This guide is a help to install the OOD RStudio Server app without requiring PRoot and/or Singularity. Prior to RStudio Server 1.3###, some hard-coded file paths caused a conflict when running multiple RStudio Server instances on the same server, as is the case when more than one OOD job launches to the same compute node. PRoot/Singularity fixed this issue by creating a unique TMP_DIR which was mounted into the container at /tmp. This fix successfully separated multiple instances running on the same server. To my knowledge, this was the primary benefit of using PRoot/Singularity. 

Starting in RStudio Server 1.3###, the developers added environment variables to control the location of those hard-coded files. This means that (in our case at least) PRoot/Singularity is no longer required for the OOD RStudio Server App.

## Requirements
- [Lmod](https://lmod.readthedocs.io/en/latest/) or any environment module system
- [R](https://www.r-project.org/)
- [RStudio Server](https://www.rstudio.com/products/rstudio/download-server/) 1.4 or higher (any of the recent date based releases will work)

# Setup without PRoot/Singularity
RStudio Server 1.4.XXXX and above enables RStudio OOD app to run without PRoot or Singularity. The developers added two flags for rserver that allow control of previously hard-coded file paths. However, you might still need PRoot or Singularity for some other reason depending on your HPC environment.

## Install RStudio Server
The simplest way is to install using the RPM (we run CentOS 7) installer provided by RStudio.
```
wget https://download2.rstudio.org/server/centos7/x86_64/rstudio-server-rhel-2022.02.2-485-x86_64.rpm
sudo yum install rstudio-server-rhel-2022.02.2-485-x86_64.rpm
```
However, many HPC sites like to control installation prefix. The provided RPM is not relocatable, but you can extract the files and move them manually.
```
mkdir ~/rstudio_files && cd ~/rstudio_files
wget https://download2.rstudio.org/server/centos7/x86_64/rstudio-server-rhel-2022.02.2-485-x86_64.rpm
rpm2cpio rstudio-server-rhel-2022.02.2-485-x86_64.rpm | cpio -idmv
```
This creates a directory `~/rstudio_files/usr/lib/rstudio-server` that contains the files. Move these files to your shared filesystem.
```
# Our shared filesystem for apps is /hpc/apps. Modify to suit your needs.
cp -R ~/rstudio_files/usr/lib/rstudio-server/* /hpc/apps/rstudio-server/2022.02.2-485
```
Optionally add a modulefile to load env.

## RStudio Server OOD App
To install the app:
```
cd /var/www/ood/apps/sys 
git clone https://github.com/mcw-rcc/bc_rcc_rstudio_server.git
cd bc_rcc_rstudio_server
```
Modify `manifest.yml`, `form.yml`, and `submit.yml.erb` for your site. The `template/script.sh.erb` should also be modified to suit your site. For instance, we use a modulefile to load RStudio Server, which is specific by site and should be supplied by your site.
```
rserver \
  --www-port ${port} \
  --auth-none 0 \
  --auth-pam-helper-path "${RSTUDIO_AUTH}" \
  --auth-encrypt-password 0 \
  --rsession-path "${RSESSION_WRAPPER_FILE}" \
  --server-data-dir "${TMPDIR}" \ 
  --secure-cookie-key-file "${TMPDIR}/rstudio-server/secure-cookie-key"
  --database-config-file "${DBCONF}"
  --server-user $(whoami)
```
`--server-data-dir "${TMPDIR}"` redirects output of PIDs from /var/run/rstudio-rsession to ${TMPDIR}.

`--secure-cookie-key-file "${TMPDIR}/rstudio-server/secure-cookie-key"` redirects output of secure-cookie-key from /tmp/rstudio-server to $TMPDIR. Necessary to avoid conflict where multiple RStudio Server instances are running and generate same hard-coded file.

`--database-config-file "${DBCONF}"` sets the path for a new database config requirement in 1.4.1717.

Your $TMPDIR location needs to be unique for this to work. We use our queueing system to set a unique $TMPDIR per job. You can also use `export TMPDIR="$(mktemp -d)"` within `template/script.sh.erb`.
