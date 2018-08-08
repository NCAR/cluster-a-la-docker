ARG DEBRELEASE

#
# debian setup with sources 
#
FROM debian:$DEBRELEASE as deb
ARG PKGCACHE
ENV PKGCACHE=${PKGCACHE}
RUN bash -c 'echo "Acquire::http::proxy \"http://${PKGCACHE}/\";" > /etc/apt/apt.conf.d/00Proxy '
RUN apt-get -y update && \
    apt-get dist-upgrade -y && \
    apt-get -qy install apt-utils less

#
# Basic centos image on a debian container
#
FROM deb as centos_image
ARG PKGCACHE
ENV PKGCACHE=${PKGCACHE}
ARG CENTOS_RELEASE
ENV CENTOS_RELEASE=${CENTOS_RELEASE}
COPY client/uname client/yum.conf /root/
RUN mkdir -p /image/etc/pki/ /image/usr/lib/ && \
    apt-get -qy install yum git && \
    bash -c 'sed -e "s#\$releasever#${CENTOS_RELEASE}#g" -e "s#\${PKGCACHE}#${PKGCACHE}#g" /root/yum.conf > /image/etc/yum.conf' && \
    mkdir -p /image/ 

#Create centos rootfs install to compiling mofed against the current centos kernel
ENV YUM="yum --config /image/etc/yum.conf --installroot=/image/ --nogpgcheck --releasever=${CENTOS_RELEASE} -y"
RUN $YUM clean metadata && \
    $YUM --nogpgcheck install bash dracut-network openssl kernel openssh-server yum rpm nspr centos-release \
	iputils irqbalance procps rsyslog busybox biosdevname perl kexec-tools yum-plugin-ovl && \
    cp -par /usr/lib/rpm /image/usr/lib/ && \
    cp -parv /etc/hosts /etc/resolv.conf /image/etc/ && \
    #Override uname output to match centos image instead of the hypervisor or the docker container
    cp /image/usr/bin/uname /image/usr/bin/uname.orig && \
    cp -vf /root/uname /image/usr/bin/uname 

FROM centos_image as centos_build
RUN apt-get -qy install git && \
    chroot /image bash -c ' \
	yum -y install bash dracut-network openssl kernel openssh-server \
	    iputils irqbalance procps ntp rsyslog pciutils pciutils-libs kernel-headers \
	    cpuspeed cpufrequtils numactl mcelog usbutils busybox biosdevname perl kexec-tools \
	    gtk2 atk cairo libxml2-python gcc-gfortran tcsh libnl lsof ethtool tcl tk kernel-devel  \
	    createrepo python-devel lsof redhat-rpm-config rpm-build ethtool gcc gtk2 atk cairo \
	    gcc-gfortran tcsh libnl tcl tk ksh libaio m4 net-tools gcc-c++ less vim autoconf \
	    texinfo automake  && \
	yum group install -y "Development Tools"   && \
	rpm --rebuilddb && \
	yum clean all \
    '

#Compile mofed against the centos kernel
FROM centos_build as mofed_build
ARG MOFED_TARBALL
ENV MOFED_TARBALL=${MOFED_TARBALL}
COPY client/${MOFED_TARBALL} /root/${MOFED_TARBALL}
RUN mkdir -p /image/tmp/mofed/ && \
    tar --strip-components=1 -C /image/tmp/mofed/ -xzf /root/${MOFED_TARBALL} && \
    chroot /image/ perl /tmp/mofed/mlnx_add_kernel_support.sh -y -m /tmp/mofed/ --make-tgz --skip-repo --name mofed

#Compile pam_radius
FROM centos_build as radius_build
RUN chroot /image bash -c ' \
	yum clean all && \
	yum -y install freeradius-devel pam-devel \
    '
RUN mkdir -p /image/tmp/pam_radius && \
    git clone --single-branch https://github.com/FreeRADIUS/pam_radius /image/tmp/pam_radius -b release_1_4_0
RUN chroot /image bash -c ' \
	cd /tmp/pam_radius/ && \
	sed "s#-Wall##g" -i configure* Make* && \
	autoreconf -i && \
	./configure && \
	make \
    '

#compile gpfs kmod
FROM centos_build as gpfs_build
COPY client/gpfs/*.rpm /image/tmp/
RUN mkdir -p /image/tmp/gpfs/ && \
    chroot /image/ rpm -Uvh /tmp/*.rpm && \
    chroot /image/ env LINUX_DISTRIBUTION=REDHAT_AS_LINUX /usr/lpp/mmfs/bin/mmbuildgpl --build-package

#compile msmtp for mail
FROM centos_build as msmtp_build
ARG MSMTP_TARBALL
ENV MSMTP_TARBALL=${MSMTP_TARBALL}
COPY client/${MSMTP_TARBALL} /tmp/msmtp.tar.xz
RUN mkdir -p /image/tmp/msmtp  && \
    tar --strip-components=1 -C /image/tmp/msmtp -xf /tmp/msmtp.tar.xz && \
    chroot /image/ bash -c ' \
	mkdir -p /tmp/msmtp && \
	cd /tmp/msmtp; \
	autoreconf -i; \
	./configure; \
	make; \
	make install  \
    '

#compile slurm and virtualgl
FROM centos_build as slurm_build
ENV PKGCONFIG=/usr/local/lib/pkgconfig/:/usr/local/lib64/pkgconfig/
RUN chroot /image yum install --nogpgcheck -y libgcrypt-devel openssl-devel jansson-devel \
    libevent-devel libX11-devel gettext-devel gettext-common-devel glib2-devel gtk2-devel \
    pam-devel numactl-devel xz-devel rrdtool-devel cmake libzip-devel \
    mariadb-devel ncurses-devel lua-devel readline-devel libcurl-devel \
    json-c-devel perl-ExtUtils-MakeMaker.noarch apr-devel \
    cmake nasm java-1.8.0-openjdk-devel java-1.8.0-openjdk help2man.noarch \
    libX11-devel libXi-devel libXinerama-devel glibc-devel pam-devel libXt-devel freeglut-devel freeglut libXtst-devel
RUN bash -c 'mkdir -p /image/tmp/{confuse,autoconf,munge,nhc,hwloc,netloc,nvme,slurm,pmix,inception,ganglia,turbovnc,virtualgl,cmake,libjpeg-turbo}' && \
    git clone --depth 1 --single-branch https://github.com/martinh/libconfuse.git /image/tmp/confuse -b v3.2.1 && \
    git clone --depth 1 --single-branch https://github.com/ganglia/monitor-core.git /image/tmp/ganglia -b release/3.6 && \
    git clone --recurse-submodules --shallow-submodules --single-branch git://git.sv.gnu.org/autoconf /image/tmp/autoconf -b master && \
    git clone --depth 1 --single-branch https://github.com/linux-nvme/nvme-cli.git /image/tmp/nvme -b v1.5 && \
    git clone --depth 1 --single-branch https://github.com/Kitware/CMake.git /image/tmp/cmake -b v3.1.3 && \
    git clone --depth 1 --single-branch https://github.com/libjpeg-turbo/libjpeg-turbo.git /image/tmp/libjpeg-turbo -b 1.5.90 && \
    git clone --depth 1 --single-branch https://github.com/VirtualGL/virtualgl.git /image/tmp/virtualgl -b 2.6beta1 && \
    git clone --depth 1 --single-branch https://github.com/TurboVNC/turbovnc.git /image/tmp/turbovnc -b 2.1.2  && \
    git clone --depth 1 --single-branch https://github.com/dun/munge.git /image/tmp/munge -b master && \
    git clone --depth 1 --single-branch https://github.com/open-mpi/hwloc.git /image/tmp/hwloc -b hwloc-1.11.8 && \
    git clone --depth 1 --single-branch https://github.com/open-mpi/netloc.git /image/tmp/netloc -b v0.5 && \
    git clone --depth 1 --single-branch https://github.com/pmix/pmix.git /image/tmp/pmix -b v2.0.3 && \
    git clone --depth 1 --single-branch https://github.com/lz4/lz4.git /image/tmp/lz4 -b v1.8.1.2 && \
    git clone --depth 1 --single-branch https://github.com/libssh2/libssh2.git /image/tmp/libssh2 -b libssh2-1.8.0 && \
    git clone --depth 1 --single-branch https://github.com/mej/nhc.git /image/tmp/nhc -b 1.4.2 && \
    git clone --depth 1 --single-branch https://github.com/NCAR/Inception.git /image/tmp/inception -b master && \
    git clone --depth 1 --single-branch https://github.com/SchedMD/slurm.git /image/tmp/slurm -b slurm-17-11-5-1
RUN chroot /image bash -c '    \
	cd /tmp/autoconf && \
	git checkout e5f5e535adec83b146bfc921d9005ecf6a846464 && \
	autoreconf -i && ./configure && make -j60 && make install && \
	cd /tmp/confuse && \
	/usr/local/bin/autoreconf -i && ./configure && make -j60 && make install && \
	cd /tmp/ganglia && \
	./bootstrap && ./configure --enable-status && make -j60 && make install && \
	mkdir -p /tmp/cmake/build && \
	cd /tmp/cmake/build && \
	cmake .. && make -j40 && make install  && \
	export LD_LIBRARY_PATH=/usr/lib/jvm/java/lib/amd64/jli:$LD_LIBRARY_PATH && \
	mkdir -p /tmp/turbovnc/build /tmp/virtualgl/build /tmp/libjpeg-turbo/build && \
	cd /tmp/libjpeg-turbo/build && \
	/usr/local/bin/cmake -DCMAKE_INSTALL_PREFIX:PATH=/usr/local/ -DCMAKE_POSITION_INDEPENDENT_CODE=on -DWITH_JAVA=1 .. && make -j40 && make install && \
	export LD_LIBRARY_PATH=/usr/lib/jvm/java/lib/amd64/jli:$LD_LIBRARY_PATH && \
	#find /usr/local -name \*.jar && false && \
	cd /tmp/turbovnc/build && \
	/usr/local/bin/cmake -DCMAKE_INSTALL_PREFIX:PATH=/usr/local/ -DTJPEG_JNILIBRARY=/usr/local/lib64/libturbojpeg.so -DTJPEG_JAR=/usr/local/share/java/turbojpeg.jar  -G"Unix Makefiles"  -DTVNC_DLOPENSSL=0 ..  && \
	make -j40 && make install && \
	export LD_LIBRARY_PATH=/usr/lib/jvm/java/lib/amd64/jli:$LD_LIBRARY_PATH && \
	cd /tmp/virtualgl/build && \
	/usr/local/bin/cmake -DCMAKE_INSTALL_PREFIX:PATH=/usr/local/ -G"Unix Makefiles" -DVGL_USESSL=1 ..  && \
	make && make install && \
	\
	cd /tmp/munge && \
	autoreconf -i && \
	env CFLAGS="-D_GNU_SOURCE " ./configure --with-munge-socket=/var/run/munge/munge.socket.2 --with-crypto-lib=openssl && \
	make -j 60 && \
	make install && \
	\
	cd /tmp/hwloc && \
	autoreconf -i && \
	./configure && \
	make -j 60 && \
	make install && \
	\
	cd /tmp/netloc && \
	autoreconf -i && \
	./configure --with-hwloc=/usr/local/ && \
	make -j 60 && \
	make install && \
	\
	cd /tmp/pmix && \
	./autogen.sh  && \
	./configure && \
	make -j 60 && \
	make install && \
	cd /tmp/libssh2 && \
	mkdir bin && \
	cd bin && \
	/usr/local/bin/cmake -D BUILD_SHARED_LIBS=ON .. && \
	/usr/local/bin/cmake -D BUILD_SHARED_LIBS=ON --build . && \
	make -j 60 && \
	make install && \
	cd /tmp/lz4 && \
	make -j 60 && \
	make install && \
	cd /tmp/slurm && \
	autoreconf -i && \
	env CFLAGS="-DGPL_LICENSED -Wl,-rpath=/usr/local/lib -Wl,-rpath=/usr/local/lib64 -L/usr/local/lib64/ -L/usr/lib64/ " \
	./configure --with-munge=/usr/local/ --sysconfdir=/etc/slurm/ \
	--enable-static --disable-gtktest --enable-pam --with-hdf5=no --with-freeipmi=no \
	--enable-really-no-cray --enable-salloc-kill-cmd --with-ssl=/usr/local/ && \
	make -j 60 && \
	make install && \
	cd contribs/pam_slurm_adopt && \
	make install && \
	cd /tmp/inception && \
	mkdir build && \
	/usr/local/bin/cmake -DCMAKE_INSTALL_PREFIX:PATH=/usr/local/ -DINCEPTION_CONFIG_PATH=/ncar/ssg/inception/etc/inception.json && \
	make && make install && \
	chmod 6755 /usr/local/bin/inception && \
	cd /tmp/inception/slurm && \
	gcc -DINCEPTION_CONFIG_PATH=\"/ncar/ssg/inception/etc/inception.json\" -I/usr/local/include -shared -g -fPIC -o /usr/local/lib/slurm-inception.so slurm-inception.c -I. -I.. -L../build -linception -Wl,--rpath=/usr/local/bin/,--rpath=/usr/local/lib -ljansson && \
	cd /tmp/nvme && \
	make && make install \
    '
#RUN chroot /image bash -c '    \
#	cd /tmp/nhc && \
#	autoreconf -i && \
#	./configure && \
#	make && make install \
#    '

#
#Create centos rootfs install to create image for pxe booting
#
FROM centos_image as pxe_image_build
ARG SRV_IP
ENV SRV_IP=${SRV_IP}
ARG SYSLOG_IP
ENV SYSLOG_IP=${SYSLOG_IP}
ARG NVIDIA_INSTALL
ENV NVIDIA_INSTALL=${NVIDIA_INSTALL}

RUN apt-get -qy install squashfs-tools pigz && \
    chroot /image bash -c ' \
	yum --nogpgcheck -y install bash dracut-network nfs-utils openssl kernel kernel-devel openssh-server openssh-clients \
	iputils irqbalance procps wget ntp rpm rsync rsyslog e2fsprogs parted pciutils pciutils-libs \
	cpufrequtils numactl mcelog usbutils biosdevname perl kexec-tools \
	createrepo python-devel lsof redhat-rpm-config rpm-build ethtool gcc gtk2 atk cairo gcc-gfortran tcsh libnl tcl tk \
	openssh-clients less iptables tcpdump vim which cronie iptables-services iptables-utils \
	chrony ksh libaio net-tools m4 strace ltrace \
	xorg-x11-server-Xorg gdm glx-utils libXext fontconfig freeglut xorg-x11-utils libXrender \
	mesa-libGL mesa-libGLU mesa-libGLw mesa-libGL-devel mesa-libGLU-devel mesa-libGLw-devel \
	gtk2 atk cairo libxml2-python gcc-gfortran tcsh libnl lsof ethtool tcl tk kernel-devel  \
	createrepo python-devel lsof redhat-rpm-config rpm-build ethtool gcc gtk2 atk cairo gcc-gfortran tcsh \
	libnl tcl tk coreutils sudo mailx tuned lshw unzip \
	bash dracut-network openssl kernel openssh-server yum rpm dbus-glib-devel \
        iputils irqbalance procps rsyslog busybox biosdevname perl kexec-tools yum-plugin-ovl  \
	openssl kernel openssh-server libconfig-devel libwnck3-devel flex-devel libglade2-devel \
	iputils irqbalance procps ntp rsyslog pciutils pciutils-libs kernel-headers \
	cpuspeed cpufrequtils numactl mcelog usbutils busybox biosdevname perl kexec-tools \
	gtk2 atk cairo libxml2-python gcc-gfortran tcsh libnl lsof ethtool tcl tk kernel-devel  \
	createrepo python-devel lsof redhat-rpm-config rpm-build ethtool gcc gtk2 atk cairo \
	gcc-gfortran tcsh libnl tcl tk ksh libaio m4 net-tools gcc-c++ less vim autoconf \
	texinfo automake xterm gnome-terminal xauth readline-devel jasper-devel dconf-devel \
	readline-static texlive xorg-x11-apps mariadb-devel libevent-devel ipmitool openldap-devel \
	readline-devel gperf gperftools gperftools-devel perftools-libs libsndfile libsndfile-devel libsndfile-utils \
	libXScrnSaver-devel libXevie-devel libXfont-devel libXfont2-devel libXres-devel libXtst-devel \
	libXv compat-db42 xvattr man-pages-overrides fontpackages-filesystem metacity liboil antlr tftp xorg-x11-utils perl-Module-Build bash libpcap powertop cvs nss-tools xml-commons-apis python-simplejson compat-expat1 zsh tcp_wrappers-libs texlive-texmf-fonts libuser jakarta-commons-httpclient blktrace ql2100-firmware ftp xz-libs libXi-devel pygtksourceview ttmkfdir libcgroup docbook-style-xsl bridge-utils m4 xmlto automake chkconfig glibc-static libspectre eclipse-nls-fr libdaemon perl-Module-Pluggable perl-List-MoreUtils eclipse-nls-hi boost-system libselinux-utils evince-libs keyutils-libs kernel-headers elfutils-libs emacs-gnuplot libwacom bcel cpio enchant iwl1000-firmware device-mapper-event-libs ant-apache-oro coreutils libidn-devel lvm2-libs ant-jdepend libpciaccess mesa-libGL-devel libgcrypt-devel dhcp-common foomatic e2fsprogs-libs perl-MailTools giflib pexpect slf4j crda setup pinfo cifs-utils libXfixes pciutils ca-certificates redhat-bookmarks gtk2 ctan-cm-lgc-sans-fonts rpmdevtools gnome-python2-gnome ncurses-base pulseaudio-libs-glib2 docbook-dtds libspiro sudo ws-commons-util qt-sqlite numactl-devel cairomm-devel chrpath xmlrpc-c-client python-dmidecode xalan-j2 texlive-texmf-errata-fonts libXau-devel python-setuptools gnome-icon-theme libtiff-devel bzip2 gnome-python2-desktop expect pcmciautils zlib perl-CPANPLUS icu4j-eclipse at-spi eclipse-nls-he perl-version abrt-addon-kerneloops dejavu-fonts-common eclipse-nls-de perl-Params-Validate avahi-libs geronimo-specs perl-libwww-perl system-config-network-tui dbus policycoreutils ant-commons-net iwl6000g2a-firmware boost-regex pcre samba-common gnuplot ant-swing perl-libxml-perl pixman plymouth-gdm-hooks iputils paps perl-Term-UI redhat-lsb-printing compat-openldap perl-Digest-SHA systemtap-runtime udisks libtasn1 subversion-javahl ruby-devel boost-signals libXtst bfa-firmware pygpgme libbonoboui lockdev cracklib xml-common mysql-devel eclipse-svnkit libmcpp sgml-common bison openmotif-devel dbus-x11 cairo-devel pth texinfo popt libXres libgweather docbook-utils tree python-magic curl libgnomecanvas gpm-libs vconfig dbus-devel libXft-devel python-dateutil zd1211-firmware lcms-libs mcelog gvfs eclipse-nls jzlib tk rsync rpm-python libjpeg-turbo unique gettext eclipse-nls-no fprintd libuuid perl-rrdtool system-gnome-theme ql2400-firmware gegl zlib-devel ant-nodeps iwl100-firmware xorg-x11-drv-evdev perl-IPC-Cmd nodejs ant-jsch gnome-settings-daemon hwdata junit4 mesa-dri-drivers plymouth-scripts eggdbus virt-what perl-Object-Accessor boost-wave kexec-tools libtar DeviceKit-power python-urlgrabber libtheora lftp lvm2 libX11 rpmlint boost-program-options cpuspeed libXaw time latencytop eclipse-birt gnome-keyring-pam xorg-x11-xinit gcc-java gnome-session libattr ConsoleKit-libs e2fsprogs m17n-lib ImageMagick swig python-msgpack libasyncns at gtk2-devel java authconfig libreport-plugin-logger classpathx-jaf pcre-devel eject tex-preview libXdamage-devel libarchive libwnck lsof iwl3945-firmware tex-cm-lgc libart_lgpl xorg-x11-fonts-ISO8859 unixODBC ws-jaxme libcollection eclipse-linuxprofilingframework gnome-panel xz dbus-libs fontconfig-devel pcp-conf libvisual eclipse-nls-zh_TW gawk libselinux-devel poppler eclipse-nls-ko boost-filesystem perl-Test-Harness libical wsdl4j iwl6000g2b-firmware libxcb libotf xorg-x11-server-Xorg ant-commons-logging readahead python-libs openssh-clients jna libssh2 jakarta-commons-discovery qt-x11 perl-ExtUtils-CBuilder selinux-policy knem.x86_64 elfutils net-tools python-pycurl dhclient xkeyboard-config emacs cronie-anacron ssm libXpm ncurses-devel uuidd mtools filesystem eclipse-pde libgcj jpackage-utils patchutils ctan-cm-lgc-typewriter-fonts basesystem mxm gnome-python2-gnomevfs xorg-x11-server-common apache-tomcat-apis zenity xfig-common ctags grubby libbonobo nano hesiod libxml2-devel cscope paps-libs python-jinja2 bzip2-libs libXcomposite-devel wget libffi diffstat lcms-devel libreport-plugin-rhtsupport dbus-python lzo pygtk2 freeglut-devel dbus-glib abrt-tui sg3_utils-libs traceroute junit desktop-file-utils texlive-texmf-errata-latex libxcb-devel python-gudev compat-libcap1 xmldb-api libXcursor-devel polkit-gnome PackageKit-gtk-module libxslt pyOpenSSL ipw2100-firmware xorg-x11-fonts-ISO8859 fuse nautilus glibc eclipse-mylyn-wikitext perl-Time-HiRes jakarta-commons-lang eclipse-callgraph libwmf libcom_err eclipse-subclipse-graph abrt kernel-devel fakeroot-libs perl-Pod-Escapes python-pcp yum-utils dejavu-sans-fonts expat redhat-menus perl-DateTime libstdc++-devel poppler-data libtiff tcl plymouth-core-libs system-config-firewall-base perl-XML-Parser perl-devel libxkbfile ethtool batik libblkid patch iptables axis perl-Module-Load b43-openfwwf xorg-x11-drv-void Xaw3d ant-apache-resolver perl-Package-Constants libfontenc samba-winbind-clients gnuplot-common sac python perl-XML-Twig openssh blas jetty-eclipse libgfortran seekwatcher gdm libsndfile mysql-libs phonon-backend-gstreamer polkit-desktop-policy perl-Log-Message-Simple redhat-lsb-compat perl-Test-Simple nfs-utils perl-Module-Loaded xorg-x11-font-utils tk-devel systemtap-client freeglut gnome-disk-utility-libs libsemanage sg3_utils autofs libexif liberation-fonts-common unzip crontabs libXmu boost-test gpgme perl-XML-SAX libgnomeui libXxf86vm ncurses libnih rng-tools fprintd-pam pango-devel postgresql-libs vim-X11 gnutls-devel libXdmcp-devel tcpdump gmp-devel libicu pm-utils libusb1 eclipse-nls-bg eclipse-nls-tr eclipse-nls-hu eclipse-nls-uk man-pages ntp emacs-nox biosdevname ypbind libgphoto2 gd eclipse-gef compat-libtermcap compat-db43 texlive-texmf-errata gimp-libs svnkit libmng-devel hicolor-icon-theme cdparanoia-libs xmlrpc3-common gdm-libs ncurses-libs libdmx sqlite-devel jakarta-commons-io dosfstools python-babel perl-IO-Compress-Bzip2 pakchois libproxy-python mtr slang pax libacl gtk-doc PyYAML nss-sysinit pycairo sinjdoc libsepol sshpass libreport mesa-libGLw lsscsi jakarta-commons-logging libreport-plugin-mailx rfkill theora-tools jakarta-commons-codec libogg libXrender-devel iwl5000-firmware PyXML jdom libXxf86vm-devel compat-gcc iwl4965-firmware python-paramiko libnotify PackageKit-yum-plugin xorg-x11-fonts-Type1 cpp mesa-libGLw-devel rootfiles opensp bsf tzdata glib2-devel eclipse-mylyn-pde hal-libs libstdc++ db4-utils gcalctool transfig gtk2-engines sed rhino webkitgtk dejavu-sans-mono-fonts perl rpm-build perl-HTML-Tagset m17n-db-datafiles mpfr perl-Time-ParseDate perl-Params-Check openssl-devel krb5-libs libevent classpathx-mail libtirpc libgcrypt libwacom-data regexp bc libtdb initscripts hunspell jasper-libs device-mapper ant-junit libtool-ltdl pam dracut flex sat4j tar dmraid gimp-help gstreamer jdepend libxml2-python gtkspell usermode perl-Compress-Zlib compat-db ConsoleKit-x11 freetype-devel bzr ppl perl-Git upstart lapack GConf2 foomatic-db portreserve boost-devel perl-TimeDate libgomp shared-mime-info rpcbind libXt exempi ilmbase libXcursor pam_passwdqc pciutils-libs geronimo-specs-compat yum-plugin-security libXaw-devel texlive-latex xorg-x11-apps bind python-devel psacct oprofile readline-devel alsa-utils ncompress telnet eclipse-nls-da eclipse-nls-zh eclipse-nls-ar xorg-x11-drv-ati-firmware cryptsetup-luks-libs libdrm-devel salt java java glx-utils samba-client smartmontools ghostscript gthumb libcdio fontconfig enscript eclipse-dtp gnome-desktop qperf libgcj-devel lucene compat-libf2c jakarta-commons-el openmotif libcap cairomm bzip2-devel eclipse-mylyn perl-CPAN libICE mingetty plpa-libs attr libproxy glibmm24-devel hdparm newt compat-libstdc++ nss elfutils-libelf pangomm-devel java_cup libreport-compat gamin boost-python texlive-texmf-errata-dvips libreport-cli gnome-python2 boost b43-fwcutter libXext-devel yum-metadata-parser libertas-usb8388-firmware libtool libXrandr-devel lua comps-extras notification-daemon ql2200-firmware keyutils-libs-devel xorg-x11-fonts openjade device-mapper-persistent-data gettext-devel fipscheck-lib eclipse-oprofile evince perl-Time-Piece freetype glibc-utils xorg-x11-fonts-misc man nss-util OpenEXR-libs sos audit-libs rrdtool xmlrpc3-client perl-HTML-Parser krb5-devel libudev texlive-texmf poppler-glib gstreamer-plugins-base perl-ExtUtils-MakeMaker libXp kernel evolution-data-server file jakarta-commons-net setuptool util-linux-ng libXau tmpwatch aic94xx-firmware device-mapper-libs procps ant-apache-bsf acpid redhat-lsb-core openssl sysvinit-tools rome intltool dmraid-events kpathsea trilead-ssh2 python-ethtool tcp_wrappers perl-IO-Zlib autoconf libao libpng-devel cloog-ppl perl-Module-CoreList perl-Error kernel-mft.x86_64 foomatic-db-ppds system-config-firewall-tui ORBit2 mailx python-lxml numactl xz-lzma-compat dstat mesa-dri-filesystem checkpolicy yp-tools ruby perl-DBD-MySQL libXi cronie pinentry perl-XML-NamespaceSupport perl-Crypt-SSLeay libXinerama libgail-gnome libopenraw libopenraw-gnome iptables-ipv6 abrt-cli gcc-gfortran NetworkManager-glib graphviz irqbalance nmap ksh mlocate cyrus-sasl-plain ntsysv Red_Hat_Enterprise_Linux-Release_Notes openjpeg-libs eclipse-nls-fi eclipse-nls-mn eclipse-nls-pt_BR eclipse-nls-it device-mapper-multipath-libs redhat-lsb-graphics mdadm tzdata-java busybox hsqldb qt-devel control-center avahi cracklib-dicts libXxf86misc eclipse-jdt latex2html gettext-libs perl-XML-Dumper eclipse-subclipse foomatic-db-filesystem bupc-2.2-375.x86_64 urw-fonts xorg-x11-server-utils rarian-compat nss-softokn-freebl netpbm libXxf86dga fakeroot babl mysql lynx java neon docbook-style-dsssl python-zmq libthai python-iniparse openldap xerces-j2 dos2unix newt-python abrt-libs shadow-utils librsvg2 setserial dmidecode passwd ecj words libXt-devel libgpg-error jakarta-oro atmel-firmware pytz libXinerama-devel java diffutils PackageKit-glib xmldb-api-sdk ql23xx-firmware xmlgraphics-commons avahi-glib xorg-x11-proto-devel java perl-CGI eclipse-mylyn-trac libcap-ng glibc-headers glib2 GConf2-gtk eclipse-mylyn-cdt rpm libpng system-icon-theme sound-theme-freedesktop abrt-addon-ccpp perl-PCP-LogImport perl-Pod-Simple smp_utils perl-DBIx-Simple libcom_err-devel libutempter perl-DateTime-Format-DateParse libSM dmz-cursor-themes dash file-libs poppler-utils SDL libtalloc ant latencytop-common binutils ant-apache-bcel tcl-devel libdrm iwl6050-firmware boost-date-time perl-Log-Message ant-apache-log4j xorg-x11-xauth kpartx zip coreutils-libs ant-antlr startup-notification xorg-x11-drv-wacom prelink qt hamcrest dracut-kernel compat-glibc texlive-dvips kbd-misc mesa-dri1-drivers perl-URI logrotate polkit perl-Archive-Extract kbd libsepol-devel libSM-devel boost-graph libss usbutils glibmm24 make hal python-markupsafe cups-libs wireless-tools boost-iostreams libXext boost-serialization pango gnome-vfs2 eog groff pulseaudio-libs eclipse-rcp quota gcc-c++ libcurl-devel firefox libgcj-src environment-modules bind-utils perf genisoimage binutils-devel valgrind indent byacc nfs4-acl-tools eclipse-nls-sv eclipse-nls-pl eclipse-nls-ro ql2500-firmware mesa-libGLU-devel subversion kernel-firmware selinux-policy-targeted device-mapper-multipath samba xinetd gimp-help-browser eclipse-changelog xorg-x11-xkb-utils compat-readline5 xdg-utils rcs libxslt-devel gnome-session-xsession libXScrnSaver libaio doxygen libsoup sgpio libyaml atk-devel pygobject2 compat-libgfortran grub libreport-python libglade2 rdate jasper libXfixes-devel objectweb-asm python-crypto rt61pci-firmware efibootmgr gcc avalon-framework control-center-filesystem eclipse-cdt trace-cmd libmng gedit rpm-libs gnome-themes nspr jline eclipse-nls-nl net-snmp-libs compat-glibc-headers perl-Locale-Maketext-Simple eclipse-nls-es libnl mtdev btparser libXp-devel udev fop perl-IO-Compress-Base libwmf-lite device-mapper-event qdox m2crypto gstreamer-tools pygtk2-libglade mesa-libGL perl-IO-Compress-Zlib ConsoleKit cmake ntpdate git apr apr-util perl-SGMLSpm hal-info vim-common cryptsetup-luks libXrender screen rsyslog eclipse-swt libxklavier gzip ed cyrus-sasl eclipse-rse imake iso-codes redhat-logos mcpp unixODBC-devel liberation-sans-fonts gnome-terminal openpgm gdb pixman-devel gtkmm24-devel gsl atk libreport-plugin-kerneloops log4j tcsh texlive-texmf-latex which libXmu-devel gtksourceview2 zeromq3 PackageKit-yum rt73usb-firmware libgcc nautilus-extensions jsch ltrace glibc-common eclipse-rpm-editor eclipse-valgrind libxml2 yum pangomm eclipse-nls-cs readline libXfont mozilla-filesystem eclipse-nls-el perl-ExtUtils-ParseXS gimp apache-jasper perl-Module-Load-Conditional xorg-x11-drv-vesa ant-trax gdbm module-init-tools emacs-common libgpg-error-devel gtkmm24 bind-libs mesa-libGLU MAKEDEV keyutils nfs-utils-lib nss-softokn vim-minimal systemtap system-setup-keyboard iotop acl libX11-common openssh-server postfix libesmtp perl-XML-LibXML libedit libXdamage microcode_ctl eclipse-emf libXcomposite ctan-cm-lgc-roman-fonts alsa-lib hunspell-en m17n-db perl-core rarian texlive-utils netpbm-progs elfutils-libelf-devel info redhat-indexhtml atlas java krb5-workstation libcurl xml-commons-resolver gnome-menus audit xulrunner libgnomekbd python-nose expat-devel compat-gcc libXpm-devel PackageKit gnome-python2-canvas ipw2200-firmware strace eclipse-mylyn-webtasks perl-Parse-CPAN-Meta libselinux ghostscript-fonts eclipse-nls-ja abrt-addon-python perl-libs perl-Class-Singleton lucene-contrib perl-DBD-SQLite psmisc eclipse-nls-et gmp mailcap libgssglue redhat-release-server xmlrpc-c ant-apache-regexp iw boost-thread plymouth grep ant-javamail xcb-util gimp-data-extras plymouth-utils libXdmcp libcanberra perl-DBI cups perl-File-Fetch libcroco libjpeg-turbo-devel Unix-Mknod subscription-manager gnutls python-iwlib libtool-ltdl-devel libICE-devel xterm parted libXft libgnome mercurial less eclipse-platform cairo gnome-keyring psutils ctan-cm-lgc-fonts-common xfig ustr numpy units eclipse-mylyn-java pkgconfig libproxy-bin libsigc++20-devel vim-enhanced sdparm sqlite libreport-plugin-reportuploader libcanberra-gtk2 sysstat scrub texlive-texmf-dvips findutils libX11-devel libatasmart ivtv-firmware python-enchant perl-ExtUtils-Embed libcgroup-pam geoclue libdhash libidn fipscheck perl-parent glibc-devel libfprint eclipse-nls-pt db4 python-rhsm dejavu-lgc-sans-mono-fonts python-matplotlib eclipse-nls-ru libsigc++20 cyrus-sasl-lib libvorbis kernel-devel libgudev1 iproute mx4j libusb perl-Compress-Raw-Zlib samba-winbind lpg-java-compat evince-dvi texlive flac pulseaudio-gdm-hooks ctags-etags perl-Archive-Tar libIDL redhat-lsb libao-devel perl-Compress-Raw-Bzip2 systemtap-devel python-beaker lm_sensors-libs libgsf salt-minion vte ruby-libs libXrandr gnupg2 net-snmp gnome-panel-libs libiptcdata \
	qt5-qtbase qt5-qtbase-common qt5-qtbase-devel qt5-qtbase-gui qt5-qtbase-mysql qt5-qtbase-odbc qt5-qtbase-postgresql \
	mesa-libOSMesa mesa-libOSMesa-devel fltk fltk-devel fltk-fluid \
	&& \
	yum group install -y "Development Tools" "GNOME Desktop" "System Administration Tools" \
	"Compute Node" "Development and Creative Workstation" "Legacy UNIX Compatibility" \
	"Scientific Support" "Console Internet Tools" "Server with GUI" "KDE Plasma Workspaces" \
    '
#Disable desktop daemons
RUN chroot /image/ /usr/bin/systemctl --root=/ disable postfix.service lvm2-lvmetad.service ModemManager.service libstoragemgmt.service \
	alsa-state.service avahi-daemon.service abrtd.service accounts-daemon.service abrt-oops.service  cups.service \
	libvirtd.service atd.service xinetd.service wpa_supplicant.service rngd.service rtkit-daemon.service alsa-state.service \
	gssproxy.service lvm2-lvmetad.service acpid.service ksmtuned.service ksm.service colord.service

#Install MOFED
COPY --from=mofed_build /image/tmp/mofed.tgz /image/tmp/mofed.tgz
RUN mkdir -p /image/tmp/mofed/ && \
    tar --strip-components=1 -C /image/tmp/mofed/ -xzf /image/tmp/mofed.tgz && \
    unlink /image/tmp/mofed.tgz  && \
    bash -c ' \
	chroot /image/ perl /tmp/mofed/mlnxofedinstall --without-32bit --force \
	--umad-dev-na --enable-affinity --without-fw-update --all --skip-distro-check \
	--skip-repo \
    ' && \
    chroot /image/ systemctl disable srpd.service

#Install GPFS
COPY --from=gpfs_build /image/root/rpmbuild/RPMS/x86_64/\*.rpm /image/tmp/gpfs/
COPY client/gpfs/*.rpm /image/tmp/gpfs/
RUN mkdir -p /image/tmp/gpfs/ /image/root/gpfs_dump && \
    chroot /image/ rpm -Uvh /tmp/gpfs/*.rpm && \
    chroot /image/ systemctl --root=/ disable gpfs mmautoload.service

#setup mail
COPY --from=msmtp_build /image/tmp/msmtp /image/tmp/msmtp
RUN chroot /image/ bash -c 'cd /tmp/msmtp; make install'

#install Nvidia
#COPY client/${NVIDIA_INSTALL} /image/tmp/nvidia.run
COPY client/NVIDIA-Linux-x86_64-396.26.run /image/tmp/nvidia.run
#COPY client/cuda_9.2.88.1_linux  /image/tmp/cuda-patch.run
#COPY client/cuda_9.2.88_396.26_linux    /image/tmp/cuda.run
#COPY client/cudnn-9.2-linux-x64-v7.1.tgz    /image/tmp/cudnn.tgz
#Nvidia expects to see alot of things all around the system so just fake them
RUN chroot /image/ mkdir -p /proc/sys/kernel/ /dev/ && \
    chroot /image/ bash -c 'echo "Linux version $(uname -r) (builder@kbuilder.dev.centos.org) ($(gcc -v 2>&1 | tail -1)) #1 SMP Thu Jan 25 20:13:58 UTC 2018 " > /proc/version ' && \
    chroot /image/ mv /sbin/modprobe /sbin/modprobe.real && \
    chroot /image/ cp /bin/true /sbin/modprobe  && \
    bash -c 'cp -parv /proc/{mounts,meminfo,filesystems}  /image/proc/' && \
    touch /image/dev/kmsg && \
    echo 'BOOT_IMAGE=/boot/vmlinuz root=/dev/sda ro console=ttyS1,115200n8' > /image/dev/kmsg && \
    echo '/sbin/modprobe' > /image/proc/sys/kernel/modprobe && \
    bash -c 'cp -parv /dev/{null,zero,pts,tty} /image/dev/' && \
    chroot /image/ bash -c 'rm -Rvf /usr/lib64/libGL.so*' && \
    chroot /image/ bash -c 'bash /tmp/nvidia.run -e -s -k $(uname -r) --opengl-headers --no-check-for-alternate-installs -j 60' && \
    chroot /image/ mv /sbin/modprobe.real /sbin/modprobe 
#    && \
#    chroot /image/ bash -c ' \
#	cd /tmp/ && \
#	bash cuda.run --silent --override --toolkit --samples --samplespath=/usr/local/cuda/samples/ --verbose && \
#	bash cuda-patch.run --silent --accept-eula && \
#	tar -C /usr/local/ -xvzf cudnn.tgz \
#    '

#install compiled applications
COPY --from=slurm_build /image/usr/local /image/usr/local
COPY --from=slurm_build /image/lib64/security/pam_slurm_adopt.la /image/lib64/security/pam_slurm_adopt.so /image/lib64/security/

#Install pam_radius
COPY --from=radius_build /image/tmp/pam_radius/pam_radius_auth.so /image/lib64/security/

#copy over the configs
RUN mkdir -p /tmp/server
COPY client /tmp/client/
COPY server/kdump.id_rsa server/root.ssh/id_ecdsa.pub /tmp/server/

RUN set -x && \
    #configure ganglia
    mkdir -p /image/etc/ganglia/conf.d && \
    cp -vf /tmp/client/gmond.conf /image/etc/ganglia/ && \
    cp -vf /tmp/client/ganglia.service /image/etc/systemd/system/ && \
    #disable gnome idle timeout
    cp -vf /tmp/client/dconf.ncar /image/etc/dconf/db/local.d/01-ncar && \
    cp -vf /tmp/client/dconf.locks.ncar /image/etc/dconf/db/local.d/locks/01-ncar && \
    chroot /image/ dconf update && \
    #disable screen saver https://help.gnome.org/admin/system-admin-guide/stable/desktop-lockscreen.html.en
    unlink /image/usr/bin/xdg-screensaver && \
    ln -s /bin/true /image/usr/bin/xdg-screensaver && \
    #add vtune 
    cp -vf /tmp/client/vtune.service /image/etc/systemd/system/  && \
    #configure munge
    cp -vf /tmp/client/munge.service /image/etc/systemd/system/  && \
    chroot /image /usr/sbin/groupadd -r munge && \
    chroot /image /usr/sbin/useradd -c "MUNGE authentication service" -d "/usr/local/etc/munge" -g munge -s /bin/false -r munge && \
    mkdir -m 0700 -p /image/etc/munge/ && \
    cp -vf /tmp/client/munge.key /image/etc/munge/munge.key && \
    touch /image/etc/munge/munge.seed && \
    chroot /image/ chown munge:munge -R /etc/munge && \
    chmod 0400 /image/etc/munge/munge.key /image/etc/munge/munge.seed && \
    #configure slurm
    ln -s /sys/fs/cgroup /image/cgroup && \
    chroot /image /usr/sbin/groupadd -r lsfadmin -g 101 && \
    chroot /image /usr/sbin/useradd -c "schduler daemon" -d "/etc/slurm" -g lsfadmin -s /bin/bash -u 210 -r lsfadmin && \
    mkdir -p /image/etc/slurm/ /image/slurm/tmp/slurmd && \
    cp -vf /tmp/client/slurm/slurmd.service /image/etc/systemd/system/  && \
    cp -vf /tmp/client/slurm/cgroup.conf /image/etc/slurm/  && \
    cp -vf /tmp/client/slurm/plugstack.conf /image/etc/slurm/ && \
    cp -vf /tmp/client/slurm/gres.conf /image/etc/slurm/ && \
    cp -vf /tmp/client/slurm/slurm.conf /image/etc/slurm/ && \
    chroot /image chown lsfadmin:lsfadmin -R /etc/slurm && \
    #configure msmtp
    cp -vf /tmp/client/msmtprc /image/usr/local/etc/msmtprc  && \
    chroot /image/ bash -c 'unlink /usr/sbin/sendmail; ln -s /usr/local/bin/msmtp /usr/sbin/sendmail' && \
    #configure mofed
    cp -vf /tmp/client/openib.conf /image/etc/infiniband/openib.conf && \
    #configure opensm
    rm -rvf /image/etc/opensm && \
    cp -Rvf /tmp/client/opensm /image/etc/opensm && \
    cp -vf /tmp/client/ncar-opensm.service /image/usr/lib/systemd/system/ && \
    #Setup syslog 
    cp -vf /tmp/client/rsyslog.conf /image/etc/rsyslog.conf && \
    #clone syslog tailer
    cp -vf /tmp/client/tail_to_syslog /image/usr/bin/tail_to_syslog && \
    chmod 0755 /image/usr/bin/tail_to_syslog && \
    #enable useful daemons
    chroot /image/ systemctl --root=/ enable rsyslog chronyd crond kdump iptables ip6tables munge slurmd ncar-opensm vtune ganglia && \
    #Disable machine id service since these are stateless 
    chroot /image/ systemctl --root=/ disable systemd-machine-id-commit.service && \
    #Disable gdm by default and enable by cluster config on nodes with GPUs
    chroot /image/ systemctl --root=/ disable gdm.service && \
    #setup gdm to configure xorg at startup
    chroot /image/ mkdir -p /etc/systemd/system/gdm.service.d/ && \
    cp -vf /tmp/client/gdm.override.service /image/etc/systemd/system/gdm.service.d/10-override-pre.conf && \
    #Setup VNC to use otp
    cp -vf /tmp/client/turbovncserver-security.conf /image/etc/turbovncserver-security.conf && \
    chroot /image/ touch /etc/sysconfig/network && \
    #setup SSH
    mkdir -m 0700 -p /image/root/.ssh && \
    echo "kernel.sysrq = 1" >> /image/etc/sysctl.conf && \
    cp -vf /tmp/client/ssh/moduli /image/etc/ssh/ && \
    cp -vf /tmp/client/ssh/* /image/etc/ssh/ && \
    cp -vf /tmp/client/rc.local /image/etc/rc.local && \
    chmod 0750 /image/etc/rc.local  && \
    chmod -R 0700 /image/etc/ssh/ && \
    cp -vf /tmp/client/root.ssh/* /image/root/.ssh/ && \
    bash -c 'sed "s#\${SRV_IP}#${SRV_IP}#g" -i /image/root/.ssh/config'  && \
    cat /tmp/server/id_ecdsa.pub >> /image/root/.ssh/authorized_keys && \
    chmod 0700 -R /image/root/.ssh && \
    chmod u+s /image/usr/libexec/openssh/ssh-keysign && \
    chmod 0755 /image/etc/ssh/  && \
    chmod 0744 /image/etc/ssh/ssh_config /image/etc/ssh/*.pub && \
    #setup supermod for SWEG
    mkdir -p /image/home/supermod/bin/ /image/home/supermod/.ssh/ && \
    chroot /image /usr/sbin/groupadd -r supermod && \
    chroot /image /usr/sbin/useradd -c "supermod access" -d "/home/supermod" -g supermod -s /bin/bash -r supermod && \
    cp -vf /tmp/client/slurm/ssh.xdmod.sacct /image/home/supermod/bin/ssh.xdmod.sacct.ncar && \
    chmod 0700 /image/home/supermod/.ssh/ && \
    cp -vf /tmp/client/ssh/supermod.authorized_keys /image/home/supermod/.ssh/authorized_keys && \
    chroot /image chown supermod:supermod -R /home/supermod && \
    #Setup rippersnapper
    cp -vf /tmp/client/rippersnapper/* /image/usr/local/sbin/ && \
    #Setup NTP
    cp -vf /tmp/client/chrony.conf /image/etc/chrony.conf && \
    #Setup Kdump
    cp -vf /tmp/client/kdump.conf /image/etc/kdump.conf && \
    cp -vf /tmp/server/kdump.id_rsa /image/root/.ssh/kdump_id_rsa && \
    bash -c 'sed "s#\${SRV_IP}#${SRV_IP}#g" -i /image/etc/kdump.conf' && \
    chmod 0700 /image/root/.ssh/kdump_id_rsa && \
    #configure network manager
    cp -vf /tmp/client/NetworkManager.conf /image/etc/NetworkManager/NetworkManager.conf && \
    #Copy resolv to offical location and a backup since dhclient likes to overwrite it
    #fix this by making dhclient enforce the correct resolv file
    cp -vf /tmp/client/sysconfig/network /image/etc/sysconfig/network && \
    cp -vf /tmp/client/resolv.conf /image/etc/resolv.conf && \
    cp -vf /tmp/client/resolv.conf /image/etc/.resolv.conf && \
    cp -vf /tmp/client/sysconfig/iptables /image/etc/sysconfig/iptables && \
    cp -vf /tmp/client/sysconfig/ip6tables /image/etc/sysconfig/ip6tables && \
    #Setup iptables cron checker
    cp -vf /tmp/client/iptables_check.sh /image/usr/sbin/iptables_check.sh  && \
    chmod 0755 /image/usr/sbin/iptables_check.sh && \
    #Setup Kerberos
    cp -vf /tmp/client/krb5.conf /image/etc/krb5.conf && \
    #Setup ulimits
    cp -vf /tmp/client/limits.conf /image/etc/security/limits.conf && \
    #Set timezone to Denver
    #RUN chroot /image/ timedatectl set-timezone America/Denver
    chroot /image/ ln -nfs /usr/share/zoneinfo/America/Denver /etc/localtime && \
    #setup fstab
    cp -vf /tmp/client/fstab /image/etc/fstab && \
    #setup gpfs
    mkdir -p /image/var/mmfs/gen/ && \
    cp -vf /tmp/client/gpfs/mmsdrfs /image/var/mmfs/gen/mmsdrfs && \
    #setup gpfsmond
    mkdir -p /image/usr/sbin/gpfsmond.d/ && \
    cp -vf /tmp/client/gpfs/gpfsmond /image/usr/sbin/gpfsmond  && \ 
    cp -rvf /tmp/client/gpfs/gpfsmond.d/dav/* /image/usr/sbin/gpfsmond.d && \
    chmod 0750 /image/usr/sbin/gpfsmond && \
    #add all the YS symlinks
    chroot /image/ bash -c ' \
	mkdir -p /glade/; \
	ln -nfs /glade/u/ssg/ys /ncar; \
	ln -nfs /gpfs/u /glade/u; \
	ln -nfs /gpfs/flash /glade/flash; \
	ln -nfs /gpfs/p /glade/p_old; \
	ln -nfs /glade2/scratch2 /glade/scratch; \
	ln -nfs /gpfs/fs1/scratch /glade/scratch_new; \
	ln -nfs /gpfs/fs1/work /glade/work; \
	ln -nfs /gpfs/fs1/collections /glade/collections; \
	ln -nfs /gpfs/fs1/p /glade/p; \
    ' && \
    #Setup gpfs systemd unit files and handlers
    cp -vf /tmp/client/gpfs/ncar-gpfs-startup.service \
	/tmp/client/gpfs/ncar-gpfs-mount.service \
	/tmp/client/gpfs/ncar-gpfsmond.service \
	/tmp/client/gpfs/ncar-gpfs.target \
	/image/usr/lib/systemd/system/ && \
    cp -vf /tmp/client/gpfs/stop-gpfs-mounts.ncar  \
	/tmp/client/gpfs/stop-gpfs.ncar \
	/tmp/client/gpfs/check_mmgetstate.ncar \
	/tmp/client/gpfs/check_gpfsmond.ncar \
	/image/usr/sbin/ && \
    chroot /image/ chmod 0755 \
	/usr/sbin/check_gpfsmond.ncar \
	/usr/sbin/check_mmgetstate.ncar \
	/usr/sbin/stop-gpfs.ncar \
	/usr/sbin/check_mmgetstate.ncar \
	/usr/sbin/stop-gpfs.ncar \
	/usr/sbin/stop-gpfs-mounts.ncar  && \
    #enable gpfs and friends. only statefuls has install target to autostart
    chroot /image/ systemctl --root=/ disable \
	ncar-gpfs.target \
	ncar-gpfs-startup.service \
	ncar-gpfs-mount.service ncar-gpfsmond.service && \
    #don't use the IBM GPFS since it handles clusters very poorly
    chroot /image/ systemctl --root=/ disable gpfs && \
    #remove MPI selector
    rm -vf /image/etc/profile.d/mpi-selector.csh /image/etc/profile.d/mpi-selector.sh && \
    #add profiles
    rm -vf /image/etc/profile.d/modules.* && \
    cp -vf /tmp/client/profile.d/* /image/etc/profile.d/ && \
    #create CSG requested bash.bashrc for modules
    chroot /image/ bash -c 'ln -nfs /etc/bashrc /etc/bash.bashrc' && \
    #CSG wants to be able to edit motd EV#101069
    chown 0:1564 /image/etc/motd && \
    chmod 0664 /image/etc/motd && \
    #setup sudo
    cp -vf /tmp/client/sudoers /image/etc/sudoers && \
    #Setup sysctl overrides
    cp -vf /tmp/client/sysctl.conf /image/etc/sysctl.conf && \
    #Remove unwanted crons
    chroot /image/ rm -Rvf /etc/cron.daily/logrotate /etc/cron.hourly/mcelog.cron \
	/etc/cron.daily/mlocate.cron /etc/cron.daily/cups /etc/cron.daily/rhsmd  \
	/etc/cron.daily/certwatch /etc/cron.daily/man-db.cron /etc/cron.daily/mlocate  \
	/etc/cron.daily/prelink /etc/cron.d/pcp-pmie /etc/cron.d/pcp-pmlogger \
	/etc/cron.d/sysstat && \
    #add upd pull script
    cp -vf /tmp/client/pull_ldap.sh /image/usr/sbin/pull_ldap.sh && \
    chmod 0755 /image/usr/sbin/pull_ldap.sh && \
    #install cronjobs
    cp -vf /tmp/client/cron.d/* /image/etc/cron.d/ && \
    chown root:root /image/etc/cron.d/* && \
    chmod 0640 /image/etc/cron.d/* && \
    #configure SERVERIP into the cronjobs
    bash -c 'sed "s#\${SRV_IP}#${SRV_IP}#g" -i /image/etc/cron.d/*' && \
    #Lock out normal users from cron
    cp -vf /tmp/client/pam/crond /image/etc/pam.d/crond && \
    cp -vf /tmp/client/crond.sysconfig /image/etc/sysconfig/crond && \
    cp -vf /tmp/client/cron.group.allow /image/etc/cron.group.allow && \
    #configure VirtualGL for all users
    chroot /image/ /usr/local/bin/vglserver_config -config +s +f -t  && \
    #only allow root login on SOL
    chroot /image/ bash -c 'echo tty0 > /etc/security/securetty' && \
    chroot /image/ bash -c 'echo ttyS0 >> /etc/security/securetty' && \
    chroot /image/ bash -c 'echo ttyS1 >> /etc/security/securetty' && \
    chroot /image/ bash -c 'find /etc/pam.d/ -type f | xargs sed -i "1i auth [user_unknown=ignore success=ok ignore=ignore default=bad] pam_securetty.so "  ' && \
    #configure pam for radius
    cp -vf /tmp/client/pam/sshd /image/etc/pam.d/ && \
    cp -vf /tmp/client/pam/system-auth /image/etc/pam.d/ && \
    cp -vf /tmp/client/pam/login /image/etc/pam.d/ && \
    #add lockout message place holder
    echo -n > /image/etc/ssh/lockout_message && \
    #setup access to enforce slurm adopt
    cp -vf /tmp/client/access.conf /image/etc/security/access.conf && \
    #Copy over all modprobe configs
    cp -vf /tmp/client/modprobe.d/* /image/etc/modprobe.d/ && \
    #control journald to keep sane file sizes
    cp -vf /tmp/client/journald.conf /image/etc/systemd/journald.conf && \
    #RPM queries for users, user-specific rpmdb does not work consistently. copy rpmdb for user querying
    cp -parv /image/root/.rpmdb /image/usr/local/rpm && \
    chmod 0755 -R /image/usr/local/rpm && \
    #tune cpu for perf
    #https://plus.google.com/u/0/+SubhenduGhosh/posts/BJ9YhWDRaJR
    mkdir -p /image/usr/lib/tuned/ncar && \
    echo "ncar" > /image/etc/tuned/active_profile && \
    cp -vf /tmp/client/tuned/tuned-main.conf /image/etc/tuned/tuned-main.conf &&  \
    cp -vf /tmp/client/tuned/tuned.conf /image/usr/lib/tuned/ncar/tuned.conf &&  \
    #fix machine id failing since systemd doesnt know about overlayfs yet
    chroot /image bash -c 'unlink /usr/lib/systemd/systemd-machine-id-commit; ln -s /bin/true /usr/lib/systemd/systemd-machine-id-commit' && \
    sed '/Condition/d' -i /image/usr/lib/systemd/system/systemd-machine-id-commit.service && \
    #Make audit log to syslog and not locally
    cp -vf /tmp/client/audisp.syslog.conf /image/etc/audisp/plugins.d/syslog.conf  && \
    cp -vf /tmp/client/auditd.conf /image/etc/audit/auditd.conf  && \
    #enable user namespace (kindof) 
    echo dockremap:808080:1000 >> /image/etc/subuid && \
    echo dockremap:808080:1000 >> /image/etc/subgid && \
    #setup Supermicro ipmi tool
    cp -vf /tmp/client/IPMICFG_1.28.0_build.180302.zip /image/tmp/ && \
    chroot /image bash -c "cd /tmp/; unzip /tmp/IPMICFG_1.28.0_build.180302.zip; mv -v /tmp/IPMICFG_1.28.0_build.180302/Linux/64bit /usr/local/ipmicfg/"

#Setup root login
#TODO: FIXME
#RUN chroot /image/ bash -c 'echo "root:password" | chpasswd'

#Reverse uname override
RUN mv /image/usr/bin/uname.orig /image/usr/bin/uname && \
    #override dracut code with out changes
    cp -vf /tmp/client/dracut.network.sh /image/usr/lib/dracut/modules.d/90dmsquash-live/dmsquash-live-root.sh && \
    cp -vf /tmp/client/dracut.liveimage.sh /image/usr/lib/dracut/modules.d/90livenet/livenetroot.sh && \
    chmod 0755 /image/usr/lib/dracut/modules.d/90dmsquash-live/dmsquash-live-root.sh && \
    cp -vf /tmp/client/dracut.conf /image/etc/dracut.conf.d/pxe.conf && \
    bash -c 'sed "s#\${SRV_IP}#${SRV_IP}#g" -i /image/usr/lib/dracut/modules.d/90dmsquash-live/dmsquash-live-root.sh' && \
    #cat /image/usr/lib/dracut/modules.d/90dmsquash-live/dmsquash-live-root.sh && \
    chroot /image/ bash -c 'dracut -f --xz -v /boot/initrd $(rpm -qa kernel | sed "s#kernel-##g")' 

#add the libs to ldconfig
RUN chroot /image ldconfig /usr/local/lib /usr/local/lib64

FROM pxe_image_build as pxe_image
RUN mkdir -p /pxe/ 
    #/usr/bin/mksquashfs /image/ /image/image -comp xz -Xbcj x86 -no-exports -no-xattrs -noappend -progress 
    #too slow for testing
RUN /usr/bin/mksquashfs /image/ /pxe/image -comp xz -no-exports -no-xattrs -noappend -progress -wildcards -ef /tmp/client/squashfs.exclude && \
    bash -c 'cp /image/boot/vmlinuz* /pxe/kernel'  && \
    bash -c 'cp /image/boot/initrd /pxe/initrd'

FROM deb as pxe
ARG MGT_DEV
ENV MGT_DEV=${MGT_DEV}
ARG DHCP_SUBNET
ENV DHCP_SUBNET=${DHCP_SUBNET}
ARG DHCP_NETMASK
ENV DHCP_NETMASK=${DHCP_NETMASK}
ARG DHCP_POOL_START
ENV DHCP_POOL_START=${DHCP_POOL_START}
ARG DHCP_POOL_STOP
ENV DHCP_POOL_STOP=${DHCP_POOL_STOP}
ARG SRV_IP
ENV SRV_IP=${SRV_IP}
WORKDIR /

RUN apt-get update && apt-get -q -y install isc-dhcp-server pxelinux syslinux-efi atftpd supervisor nginx \
    yum openssh-server conman ipmitool git python python-setuptools libnss3  \
    cron coreutils perl git autoconf gcc make libnet-ldap-perl vim libyaml-dev \
    python-dev python-pexpect unzip telnet

#Install Cronie
RUN mkdir -p /tmp/cronie &&  \
    git clone https://github.com/cronie-crond/cronie.git /tmp/cronie && \
    bash -c ' \
	cd /tmp/cronie && \
	bash autogen.sh && \
	./configure --prefix=/ --enable-syscrontab && \
	make -j120 && \
	make install && \
	rm -Rf /cronie \
    ' && \
    mkdir -p /tmp/clustershell && \
    git clone https://github.com/cea-hpc/clustershell.git /tmp/clustershell && \
    bash -c ' \
	cd /tmp/clustershell && \
	python setup.py install && \
	unlink /etc/clustershell/groups.d/local.cfg && \
	touch /etc/clustershell/groups.d/local.cfg && \
	rm -Rf /root/clustershell \
    '

RUN mkdir -p /root/ldap /tmp/client/root.ssh
COPY server /tmp/server
COPY client/root.ssh /tmp/client/root.ssh
COPY client/ssh /tmp/client/ssh
COPY client/hosts /tmp/client/hosts
COPY --from=pxe_image  /image/etc/passwd /root/ldap/passwd.tpl
COPY --from=pxe_image  /image/etc/group /root/ldap/group.tpl
COPY --from=pxe_image  /image/etc/shadow /root/ldap/shadow.tpl
COPY configure_cluster.sh /usr/local/bin/ 

RUN set -x && \
    #Setup upd for LDAP
    cp -vf /tmp/server/upd/ldapprocess.pl /usr/sbin/ldapprocess.pl && \
    cp -vf /tmp/server/upd/upd /usr/sbin/upd && \
    cp -vf /tmp/server/upd/upd.crontab /etc/cron.d/upd && \
    bash -c 'sed "s#\${SRV_IP}#${SRV_IP}#g" -i /etc/cron.d/upd' && \
    chown root:root /etc/cron.d/upd && \
    chmod 0644 /etc/cron.d/upd && \
    mkdir -m 0700 -p /root/ldap && \
    chmod 0750 /usr/sbin/upd /usr/sbin/ldapprocess.pl && \
    mkdir -p /srv/tftp/ldap/ && \
    bash /usr/sbin/upd nopush 0 && \
    #install cronjobs
    cp -vf /tmp/server/cron.d/* /etc/cron.d/ && \
    chown root:root /etc/cron.d/* && \
    chmod 0755 /etc/cron.d/* && \
    #configure SERVERIP into the cronjobs
    bash -c 'sed "s#\${SRV_IP}#${SRV_IP}#g" -i /etc/cron.d/*' && \
    #setup PXE linux config
    mkdir -p /srv/tftp /srv/tftp/pxelinux.cfg && \
    cp -vf \
      /usr/lib/PXELINUX/lpxelinux.0 \
      /usr/lib/syslinux/modules/efi64/ldlinux.e64 \
      /usr/lib/syslinux/modules/efi32/ldlinux.e32 \
      /usr/lib/syslinux/modules/bios/ldlinux.c32 \
      /srv/tftp/ && \
    cp -vf /usr/lib/SYSLINUX.EFI/efi32/syslinux.efi /srv/tftp/syslinux.efi && \
    cp -vf /usr/lib/SYSLINUX.EFI/efi64/syslinux.efi /srv/tftp/bootx64.efi && \
    #setup DHCP
    cp -vf /tmp/server/dhcpd.conf /etc/dhcp/dhcpd.conf && \
    bash -c 'sed \
	-e "s#\${DHCP_SUBNET}#${DHCP_SUBNET}#g"  \
	-e "s#\${DHCP_NETMASK}#${DHCP_NETMASK}#g"  \
	-e "s#\${DHCP_POOL_START}#${DHCP_POOL_START}#g"  \
	-e "s#\${DHCP_POOL_STOP}#${DHCP_POOL_STOP}#g"  \
	-e "s#\${SRV_IP}#${SRV_IP}#g"  \
	-i /etc/dhcp/dhcpd.conf' && \
    touch /var/lib/dhcp/dhcpd.leases && \
    unlink /etc/dhcp/dhcpd6.conf && \
    #setup nginx to serve out files
    cp -vf /tmp/server/nginx.conf /etc/nginx/sites-available/default && \
    bash -c 'sed "s#\${SRV_IP}#${SRV_IP}#g" -i /etc/nginx/sites-available/default' && \
    #setup supervisord
    cp -vf /tmp/server/supervisord.conf /etc/supervisord.conf && \
    bash -c 'sed -e "s#\${MGT_DEV}#${MGT_DEV}#g" -e "s#\${SRV_IP}#${SRV_IP}#g" -i /etc/supervisord.conf' && \
    #setup conman
    cp -vf /tmp/server/conman.conf /etc/conman.conf && \
    mkdir -p /logs/ && \
    #setup SSH on master
    mkdir -m 0700 -p /root/.ssh && \
    touch /root/.ssh/known_hosts && \
    chmod 0700 /root/.ssh/known_hosts && \
    cp -vf /tmp/server/root.ssh/id_ecdsa /root/.ssh/ && \
    cp -vf /tmp/server/root.ssh/id_ecdsa.pub /root/.ssh/ && \
    mkdir -p /root/client/ssh/ && \
    cp -vf /tmp/client/ssh/shosts.equiv /etc/ssh/shosts.equiv && \
    cp -vf /tmp/client/ssh/ssh_host_ecdsa_key.pub \
	/tmp/client/ssh/ssh_host_ed25519_key.pub \
	/tmp/client/ssh/ssh_host_dsa_key.pub \
	/tmp/client/ssh/ssh_host_rsa_key.pub \
	/root/client/ssh/ && \
    cp -vf /tmp/client/ssh/ssh_known_hosts /root/.ssh/known_hosts && \
    chmod 0700 -R /root/.ssh/ && \
    #Setup Kdump sshd to recieve the cores
    #TODO: Fix kdump on node. kdump refuses to load ...need ticket with rhel
    cp -vf /tmp/server/kdump.sshd_config /etc/ssh/kdump.sshd_config && \
    mkdir -p /kdump && \
    echo 'kdump:x:901:901::/home/kdump:/sbin/nologin' >> /etc/passwd && \
    echo 'kdump:x:901:kdump' >> /etc/group && \
    echo 'kdump:*:::::::' >> /etc/shadow && \
    bash -c 'sed "s#\${SRV_IP}#${SRV_IP}#g" -i /etc/ssh/kdump.sshd_config' && \
    mkdir -p /run/sshd /home/kdump/.ssh && \
    cp -vf /tmp/server/kdump.id_rsa /home/kdump/.ssh/id_rsa && \
    cp -vf /tmp/server/kdump.id_rsa.pub /home/kdump/.ssh/id_rsa.pub && \
    cp /home/kdump/.ssh/id_rsa.pub /home/kdump/.ssh/authorized_keys && \
    chown -R kdump:kdump /home/kdump  && \
    chmod -R 0700 /home/kdump/.ssh && \
    mkdir -m 0700 -p /srv/kdump/ && \
    chown -R kdump:kdump /srv/kdump/ && \
    #RUN mkdir -m 0700 -p /srv/kdump/lib/x86_64-linux-gnu/ /srv/kdump/bin /srv/kdump/lib64/
    #RUN cp /bin/bash /srv/kdump/bin/bash
    #RUN cp /bin/sh /srv/kdump/bin/sh
    #RUN cp /lib/x86_64-linux-gnu/libtinfo.so.5 /lib/x86_64-linux-gnu/libdl.so.2 /lib/x86_64-linux-gnu/libc.so.6 /srv/kdump/lib/x86_64-linux-gnu/
    #RUN cp /lib64/ld-linux-x86-64.so.2 /srv/kdump/lib64/
    mkdir -m 0755 -p /srv/tftp/ifcfg/ && \
    #work around for docker overridding /etc/hosts
    cp -vf /tmp/client/hosts /etc/hosts.docker && \
    cp -rvf /tmp/server/scripts /root/scripts && \
    chmod 0755 -R /root/scripts && \
    mv /root/scripts/* /usr/local/bin/ && \
    rm -Rf /root/scripts && \
    mkdir -p /usr/local/sum/ && \
    tar -C /usr/local/sum/ --strip-components=1 -xvzf /tmp/server/sum_2.0.1_Linux_x86_64_20180420.tar.gz

RUN mkdir -p /usr/local/SMCIPMITool  && \
    tar --strip-components=1 -C /usr/local/SMCIPMITool -xvzf /tmp/server/SMCIPMITool_2.20.0_build.180525_bundleJRE_Linux_x64.tar.gz

RUN bash /usr/local/bin/configure_cluster.sh  && \
    #compress each host override filesystem
    mkdir -p /srv/tftp/hosts/ && \
    bash -c 'find /srv/hosts/ -maxdepth 1 -mindepth 1 -type d | \
    while read p; do \
	mkdir -p $p/etc/ssh/ $p/etc/X11/; \
	cp -f /etc/hosts.slurm $p/etc/hosts; \
	cp -f /etc/ssh/shosts.equiv $p/etc/ssh/shosts.equiv; \
	cp -f /root/.ssh/known_hosts $p/etc/ssh/ssh_known_hosts; \
	cp -f /srv/tftp/ldap/passwd $p/etc/passwd; \
	cp -f /srv/tftp/ldap/shadow $p/etc/shadow; \
	cp -f /srv/tftp/ldap/group $p/etc/group; \
	chmod 0755 $p/etc/ssh $p/etc/ssh/ssh_known_hosts $p/etc/ssh/shosts.equiv $p/etc/hosts; \
	tar -C $p/ -czf /srv/tftp/hosts/$(basename $p).tgz . ; \
    done'

#Pull over image for tftp to serve
COPY --from=pxe_image /pxe/kernel /srv/tftp/
COPY --from=pxe_image /pxe/initrd /srv/tftp/
COPY --from=pxe_image /pxe/image /srv/tftp/
#COPY --from=pxe_image /pxe/fatimage.tar.gz /srv/tftp/
RUN chmod 0755 -R /srv
WORKDIR /root/

CMD cat /etc/hosts.docker > /etc/hosts && /usr/bin/supervisord -c /etc/supervisord.conf
