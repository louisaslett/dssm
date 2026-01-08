FROM ubuntu:noble

## Set locale
RUN <<EOF
  apt-get update -qq
  apt-get install locales
  echo "en_GB.UTF-8 UTF-8" > /etc/locale.gen
  locale-gen en_GB.utf8
  /usr/sbin/update-locale LANG=en_GB.UTF-8
EOF
ENV LC_ALL=en_GB.UTF-8
ENV LANG=en_GB.UTF-8

## Set other env vars
ENV DEBIAN_FRONTEND=noninteractive
ENV TZ=UTC

## Setup R and r2u for fast package install
## Next RUN statement from r2u source, https://github.com/eddelbuettel/r2u/blob/master/inst/scripts/add_cranapt_noble.sh
## Only changes:
##   - apt does not like "python3-{dbus,gi,apt}" in Dockerfile, expand to "python3-dbus python3-gi python3-apt"
##   - commands written to Rprofile.site (change order and add bspm.sudo = TRUE)
RUN <<EOF
  ## First: update apt and get keys
  apt-get install --yes --no-install-recommends ca-certificates gnupg
  gpg --homedir /tmp --no-default-keyring --keyring /usr/share/keyrings/r2u.gpg --keyserver keyserver.ubuntu.com --recv-keys A1489FE2AB99A21A 67C2D66C4B1D4339 51716619E084DAB9
  ## Second: add the repo -- here we use the well-connected mirror
  cat > /etc/apt/sources.list.d/r2u.sources <<EOF2
Types: deb
URIs: https://r2u.stat.illinois.edu/ubuntu
Suites: noble
Components: main
Arch: amd64, arm64
Signed-By: /usr/share/keyrings/r2u.gpg
EOF2
  ## Third: ensure current R is used
  cat > /etc/apt/sources.list.d/cran.sources <<EOF2
Types: deb
URIs: https://cloud.r-project.org/bin/linux/ubuntu
Suites: noble-cran40/
Components:
Arch: amd64, arm64
Signed-By: /usr/share/keyrings/r2u.gpg
EOF2
  apt-get update -qq
  apt-get install --yes --no-install-recommends r-base-core
  ## Fourth: add pinning to ensure package sorting
  cat > /etc/apt/preferences.d/99cranapt <<EOF2
Package: *
Pin: release o=CRAN-Apt Project
Pin: release l=CRAN-Apt Packages
Pin-Priority: 700
EOF2
  ## Fifth: install bspm (and its Python requirements) and enable it
  ## If needed (in bare container, say) install python tools for bspm and R itself
  apt-get install --yes --no-install-recommends python3-dbus python3-gi python3-apt make
  ## Then install bspm (as root) and enable it, and enable a speed optimization
  Rscript -e 'install.packages("bspm")'
  cat >> /etc/R/Rprofile.site <<EOF2
options(bspm.version.check = FALSE, bspm.sudo = TRUE)
suppressMessages(bspm::enable())
EOF2
EOF

## Install git and RStudio Server
RUN <<EOF
  apt-get install --yes --no-install-recommends gdebi-core git-all wget
  wget -q https://download2.rstudio.org/server/jammy/amd64/rstudio-server-2026.01.0-392-amd64.deb
  apt-get install --yes --no-install-recommends ./rstudio-server-2026.01.0-392-amd64.deb
  rm rstudio-server-2026.01.0-392-amd64.deb
EOF

## Next, setup user and sudo privileges
RUN <<EOF
  apt-get install --yes --no-install-recommends sudo
  useradd -m -s /bin/bash rstudio
  echo "rstudio ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
  # Disable login requirements for RStudio and ensure only listens on localhost
  echo "auth-none=1\nwww-address=127.0.0.1\nwww-port=8787" >> /etc/rstudio/rserver.conf
  echo "USER=rstudio" >> /etc/environment
  # Enable rstudio to write to packages folder
  chown root:rstudio "/usr/lib/R/site-library"
  chmod g+ws "/usr/lib/R/site-library"
EOF

## APT config & tidyup
RUN <<EOF
  echo 'APT::Install-Recommends "false";' > /etc/apt/apt.conf.d/90local-no-recommends
  rm -rf /tmp/downloaded_packages/ /tmp/*.rds
  rm -rf /var/lib/apt/lists/*
EOF
