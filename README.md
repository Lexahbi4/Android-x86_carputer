# Android-x86_carputer
Сборка Android-X86 из исходников. (ver.11 r-x86)

SYSTEM REQUIREMENTS
-------------------
Latest Ubuntu LTS Releases https://www.ubuntu.com/download/server
Decent CPU (Dual Core or better for a faster performance)
8GB RAM (16GB for Virtual Machine)
250GB Hard Drive (about 170GB for the Repo and then building space needed)

INSTALLING JAVA 8
-----------------
sudo apt-get update && upgrade
sudo apt-get install openjdk-8-jdk-headless
update-alternatives --config java (make sure Java 8 is selected)
update-alternatives --config javac (make sure Java 8 is selected)

GRABBING DEPENDENCIES
---------------------
sudo apt-get install git-core gnupg flex bison maven gperf build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386  lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z-dev ccache libgl1-mesa-dev libxml2-utils xsltproc unzip squashfs-tools python-mako libssl-dev ninja-build lunzip syslinux syslinux-utils gettext genisoimage gettext bc xorriso libncurses5 xmlstarlet build-essential git imagemagick lib32readline-dev lib32z1-dev liblz4-tool libncurses5-dev libsdl1.2-dev libxml2 lzop pngcrush rsync schedtool python-enum34 python3-mako libelf-dev 

If you plan on building the kernel with the NO_KERNEL_CROSS_COMPILE flag, you will need to also have gcc-10+ installed:
sudo apt-get install gcc-10 g++-10

In builder VM run:
------------------
sudo ln -s /sbin/mkdosfs /usr/local/bin/mkdosfs
sudo pip install prettytable Mako pyaml dateutils --upgrade
export _JAVA_OPTIONS="-Xmx8G"
echo 'export _JAVA_OPTIONS="-Xmx8G"' >> ~/.profile
echo "sudo swapon /tmp/swapfile" >> /rw/config/rc.local

Download android-x86 sources:
-----------------------------
mkdir -p ~/.bin
PATH="${HOME}/.bin:${PATH}"
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/.bin/repo
chmod a+rx ~/.bin/repo

mkdir android-x86
cd android-x86

git config --global user.name "Lexins"
git config --global user.email "trifonoff.aleksey@gmail.com"
repo init -u git://git.osdn.net/gitroot/android-x86/manifest -b r-x86

To add GAPPS to your build you need to add the build system, and the wanted sources to your manifest.
-----------------------------------------------------------------------------------------------------
# Edit .repo/manifests/android-x86.xml and add the following towards the end:
<remote name="opengapps" fetch="https://github.com/opengapps/"  />
<remote name="opengapps-gitlab" fetch="https://gitlab.opengapps.org/opengapps/"  />

<project path="vendor/opengapps/build" name="aosp_build" revision="master" remote="opengapps" />
<project path="vendor/opengapps/sources/all" name="all" clone-depth="1" revision="master" remote="opengapps-gitlab" />

<!-- arm64 depends on arm -->
<!--project path="vendor/opengapps/sources/arm" name="arm" clone-depth="1" revision="master" remote="opengapps-gitlab" />
<project path="vendor/opengapps/sources/arm64" name="arm64" clone-depth="1" revision="master" remote="opengapps-gitlab" /-->

<project path="vendor/opengapps/sources/x86" name="x86" clone-depth="1" revision="master" remote="opengapps-gitlab" />
<project path="vendor/opengapps/sources/x86_64" name="x86_64" clone-depth="1" revision="master" remote="opengapps-gitlab" />

Download sources:
-----------------
repo sync --no-tags --no-clone-bundle --force-sync -j$( nproc --all )

# If you choose to add GAPPS, then edit file device/generic/common/device.mk and add at the beginning:
#OpenGAPPS

#GAPPS_VARIANT := pico

GAPPS_PRODUCT_PACKAGES := Chrome
# \
#    WebViewGoogle

#GAPPS_FORCE_PACKAGE_OVERRIDES := true
#GAPPS_FORCE_WEBVIEW_OVERRIDES := true
#GAPPS_FORCE_BROWSER_OVERRIDES := true

GAPPS_EXCLUDED_PACKAGES := FaceLock \
    GoogleBackupTransport \
    SetupWizard
#    AndroidPlatformServices \
#    PrebuiltGmsCoreInstantApps \


# And at the end add:
# OpenGAPPS
# $(call inherit-product, vendor/opengapps/build/opengapps-packages.mk)

# OpenGapps changed their repo to require git-lfs. There may be a better way to do this, but if you're building with GApps, this gets the right files. It takes awhile:

cd /vendor/opengapps/sources

for d in ./*/ ; do (cd "$d" && git lfs pull); done

cd android-x86
curl https://github.com/cwhuang/aosp_build/commit/384cdac7930e7a2b67fd287cfae943fdaf7e5ca3.patch | git -C vendor/opengapps/build apply -v --index
curl https://github.com/cwhuang/aosp_build/commit/3bb6f0804fe5d516b6b0bc68d8a45a2e57f147d5.patch | git -C vendor/opengapps/build apply -v --index
curl https://raw.githubusercontent.com/Lexahbi4/Android-x86_carputer/main/PowerOff_without_confirm.patch | git -C frameworks apply -v

# Edit android-x86 sources for Debian build environment:
sed -i -e 's|genisoimage|xorriso -as mkisofs|g' bootable/newinstaller/Android.mk


# Configure build target:
. build/envsetup.sh
lunch android_x86_64-userdebug


# Configure kernel:
/usr/bin/make -C kernel O=$OUT/obj/kernel ARCH=x86_64 menuconfig

# You need to edit these parameters:
SECURITY_SELINUX_BOOTPARAM=yes
SECURITY_SELINUX_BOOTPARAM_VALUE=1
SECURITY_SELINUX_DISABLE=yes
DEFAULT_SECURITY_SELINUX=yes

# Start the build:
make -j$( nproc --all ) iso_img
