pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: builder
    image: ubuntu:22.04
    command:
    - sleep
    args:
    - infinity
    resources:
      requests:
        cpu: "2000m"
        memory: "4Gi"
      limits:
        cpu: "4000m"
        memory: "8Gi"
'''
            defaultContainer 'builder'
        }
    }

    environment {
        REPO_URL = "https://github.com/coolsnowwolf/lede"
        REPO_BRANCH = "master"
        FEEDS_CONF = "feeds.conf.default"
        CONFIG_FILE = "x86_64.config"
        TZ = "Asia/Shanghai"
        FORCE_UNSAFE_CONFIGURE = "1"
    }

    triggers {
        githubPush()
    }

    stages {
        stage('Prepare Environment') {
            steps {
                container('builder') {
                    sh '''
                        export DEBIAN_FRONTEND=noninteractive
                        export TZ=$TZ
                        apt-get update -y
                        apt-get install -y ack antlr3 asciidoc autoconf automake autopoint binutils bison build-essential \
                        bzip2 ccache clang cmake cpio curl device-tree-compiler flex gawk gcc-multilib g++-multilib gettext \
                        genisoimage git gperf haveged help2man intltool libc6-dev-i386 libelf-dev libfuse-dev libglib2.0-dev \
                        libgmp3-dev libltdl-dev libmpc-dev libmpfr-dev libncurses5-dev libncursesw5-dev libpython3-dev \
                        libreadline-dev libssl-dev libtool llvm lrzsz msmtp ninja-build p7zip p7zip-full patch pkgconf \
                        python3 python3-pyelftools python3-setuptools qemu-utils rsync scons squashfs-tools subversion \
                        swig texinfo uglifyjs upx-ucl unzip vim wget xmlto xxd zlib1g-dev
                    '''
                }
            }
        }

        stage('Checkout Config Repo') {
            steps {
                container('builder') {
                    git branch: 'main',
                        url: 'https://github.com/lxzcl/Actions-OpenWrt.git'
                }
            }
        }

        stage('Clone LEDE Source') {
            steps {
                container('builder') {
                    sh '''
                        rm -rf lede || true
                        git clone --depth=1 $REPO_URL -b $REPO_BRANCH lede
                    '''
                }
            }
        }

        stage('Apply Feeds and Config') {
            steps {
                container('builder') {
                    sh '''
                        [ -e $FEEDS_CONF ] && cp $FEEDS_CONF lede/feeds.conf.default
                        [ -e $CONFIG_FILE ] && cp $CONFIG_FILE lede/.config

                        cd lede
                        ./scripts/feeds update -a
                        ./scripts/feeds install -a
                        make defconfig
                    '''
                }
            }
        }

        stage('Compile Firmware') {
            steps {
                container('builder') {
                    sh '''
                        set +e
                        cd lede
                        echo -e "$(nproc) threads compiling..."
                        make -j$(nproc)
                        result=$?
                        if [ $result -ne 0 ]; then
                            echo "Parallel build failed, retrying with -j1..."
                            make -j1 V=s
                        fi
                    '''
                }
            }
        }

        stage('Archive Artifacts') {
            steps {
                archiveArtifacts artifacts: 'lede/bin/targets/**/*', fingerprint: true
            }
        }
    }
}
