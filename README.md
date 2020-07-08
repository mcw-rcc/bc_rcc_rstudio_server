# bc_rcc_rstudio_server
RStudio Server app for Open OnDemand

Based on https://github.com/OSC/bc_osc_rstudio_server.

# Setup without PRoot/Singularity
RStudio Server 1.3.959 enables RStudio OOD app to run without PRoot or Singularity. The developers added two flags for rserver that allow control of previously hard-coded file paths. You might still need PRoot or Singularity depending on your HPC environment, so your mileage may vary.

## Install RStudio Server 1.3.959 CentOS 6/7
The simplest way is to install using the RPM installer provided by RStudio.
```
wget https://download2.rstudio.org/server/centos6/x86_64/rstudio-server-rhel-1.3.959-x86_64.rpm
sudo yum install rstudio-server-rhel-1.3.959-x86_64.rpm
```
However, many HPC sites like to control installation prefix. The provided RPM is not relocatable, but you can extract the files and move them manually.
```
mkdir ~/rstudio_files && cd ~/rstudio_files
wget https://download2.rstudio.org/server/centos6/x86_64/rstudio-server-rhel-1.3.959-x86_64.rpm
rpm2cpio rstudio-1.2.5042-x86_64.rpm | cpio -idmv
```
This creates a directory `~/rstudio_files/usr/lib/rstudio-server` that contains the files. Move these files to your shared filesystem.
```
# we use BCM so our shared filesystem for apps is /cm/shared/apps. Modify to suit your needs.
cp -R ~/rstudio_files/usr/lib/rstudio-server/* /cm/shared/apps/rstudio-server/1.3.959
```
Optionally add a modulefile to load env.

## RStudio Server OOD App
To install the app:
```
cd /var/www/ood/apps/sys 
git clone https://github.com/mcw-rcc/bc_rcc_rstudio_server.git
```
Modify `manifest.yml`, `form.yml`, and `submit.yml` for your site. The `template/script.sh.erb` should also be modified to suit your site. For instance, we use a modulefile to load RStudio Server, which is specific by site and should be supplied by your site. The `rserver` command in `template/script.sh.erb` contains two new flags that allow 1.3.959 to function in a multi-user environment without PRoot or Singularity.
```
rserver \
  --www-port ${port} \
  --auth-none 0 \
  --auth-pam-helper-path "${RSTUDIO_AUTH}" \
  --auth-encrypt-password 0 \
  --rsession-path "${RSESSION_WRAPPER_FILE}" \
  --server-data-dir "${TMPDIR}" \ 
  --secure-cookie-key-file "${TMPDIR}/rstudio-server/secure-cookie-key"
```
`--server-data-dir "${TMPDIR}"` redirects output of PIDs from /var/run/rstudio-rsession to ${TMPDIR}.

`--secure-cookie-key-file "${TMPDIR}/rstudio-server/secure-cookie-key"` redirects output of secure-cookie-key from /tmp/rstudio-server to $TMPDIR. Necessary to avoid conflict where multiple RStudio Server instances are running and generate same hard-coded file.

Your $TMPDIR location needs to be unique for this to work. We use our queueing system to set a unique $TMPDIR per job. You can also use `export TMPDIR="$(mktemp -d)"` within `template/script.sh.erb`.
