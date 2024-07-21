pipeline {
    agent {
        docker {
            args '-v /var/run/docker.sock:/var/run/docker.sock -v /mnt/nas/android_repos:/opt/android_repos -u root --privileged'
        }
    }
    
    environment {
        ANDROID_HOME = '/opt/android-sdk'
        CMDLINE_TOOLS_URL = 'https://dl.google.com/android/repository/commandlinetools-linux-8512546_latest.zip'
        PATH = "${ANDROID_HOME}/cmdline-tools/latest/bin:${ANDROID_HOME}/platform-tools:${PATH}"
        REPO_URL = 'https://android.googlesource.com/platform/manifest'
        REPO_BRANCH = 'android-12.0.0_r34'
        LOCAL_MANIFESTS_URL = 'https://github.com/remote-android/local_manifests.git'
        LOCAL_MANIFESTS_BRANCH = '12.0.0'
        REDROID_PATCHES_URL = 'https://github.com/remote-android/redroid-patches.git'
        WORK_DIR = "/opt/android_repos/redroid_${BUILD_NUMBER}"
        DOCKER_REGISTRY = 'cr.regulad.xyz'
    }
    
    stages {
        stage('Setup Environment') {
            steps {
                sh '''
                    apt-get update -y
                    apt-get upgrade -y
                    DEBIAN_FRONTEND=noninteractive apt-get install -y curl libxml2-utils docker.io docker-buildx git-lfs jq \
                        openjdk-11-jdk python3 python3-pip unzip zip qemu-user-static sudo
                    
                    # Install repo tool
                    mkdir -p ~/.bin
                    curl https://storage.googleapis.com/git-repo-downloads/repo > ~/.bin/repo
                    chmod a+x ~/.bin/repo
                    export PATH=~/.bin:$PATH
                    
                    # Install Android SDK
                    mkdir -p ${ANDROID_HOME}
                    curl -o cmdline-tools.zip ${CMDLINE_TOOLS_URL}
                    unzip cmdline-tools.zip -d ${ANDROID_HOME}
                    mv ${ANDROID_HOME}/cmdline-tools ${ANDROID_HOME}/cmdline-tools/latest
                    rm cmdline-tools.zip
                    
                    # Accept licenses
                    yes | sdkmanager --licenses
                    
                    # Install necessary SDK components
                    sdkmanager "platform-tools" "platforms;android-31" "build-tools;31.0.0"
                    
                    git config --global user.email "jenkins@example.com"
                    git config --global user.name "Jenkins"
