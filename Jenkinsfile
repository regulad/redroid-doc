pipeline {
    agent {
        docker {
            image 'ubuntu:20.04'
            args '-v /var/run/docker.sock:/var/run/docker.sock -v /mnt/nas/android_repos:/opt/android_repos -u root'
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
        stage('Cleanup and Prepare') {
            steps {
                sh '''
                    # Remove existing work directory if it exists
                    rm -rf ${WORK_DIR}
                    
                    # Create fresh work directory
                    mkdir -p ${WORK_DIR}
                    
                    # Initialize repo and sync
                    cd ${WORK_DIR}
                    repo init -u "${REPO_URL}" --git-lfs --depth=1 -b "${REPO_BRANCH}"
                    git clone "${LOCAL_MANIFESTS_URL}" .repo/local_manifests -b "${LOCAL_MANIFESTS_BRANCH}"
                    repo sync -c -j$(nproc)
                '''
            }
        }

        stage('Setup Environment') {
            steps {
                sh '''
                    apt-get update -y && apt-get upgrade -y
                    DEBIAN_FRONTEND=noninteractive apt-get install -y curl libxml2-utils docker.io docker-buildx git-lfs jq \
                        openjdk-11-jdk python3 python3-pip unzip zip qemu-user-static sudo
                    apt-get remove repo -y && apt-get autoremove -y
                    
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
                    
                    # Setup Docker buildx
                    docker buildx create --name mybuilder --use
                    docker buildx inspect --bootstrap
                '''
            }
        }
        
        stage('Apply Redroid Patches') {
            steps {
                dir("${WORK_DIR}") {
                    sh '''
                        git clone ${REDROID_PATCHES_URL} ${WORKSPACE}/redroid-patches
                        ${WORKSPACE}/redroid-patches/apply-patch.sh ${WORK_DIR}
                    '''
                }
            }
        }
        
        stage('Build Redroid') {
            parallel {
                stage('Build ARM64') {
                    steps {
                        dir("${WORK_DIR}") {
                            sh '''
                                docker buildx build --build-arg userid=$(id -u) --build-arg groupid=$(id -g) --build-arg username=$(id -un) -t redroid-builder-arm64 --platform linux/arm64 --load .
                                docker run --rm --privileged -v ${WORK_DIR}:/src redroid-builder-arm64 bash -c "
                                    cd /src && 
                                    . build/envsetup.sh && 
                                    lunch redroid_arm64-userdebug && 
                                    m -j$(nproc)
                                "
                            '''
                        }
                    }
                }
                stage('Build AMD64') {
                    steps {
                        dir("${WORK_DIR}") {
                            sh '''
                                docker buildx build --build-arg userid=$(id -u) --build-arg groupid=$(id -g) --build-arg username=$(id -un) -t redroid-builder-amd64 --platform linux/amd64 --load .
                                docker run --rm --privileged -v ${WORK_DIR}:/src redroid-builder-amd64 bash -c "
                                    cd /src && 
                                    . build/envsetup.sh && 
                                    lunch redroid_x86_64-userdebug && 
                                    m -j$(nproc)
                                "
                            '''
                        }
                    }
                }
            }
        }
        
        stage('Create Docker Images') {
            parallel {
                stage('Create ARM64 Image') {
                    steps {
                        dir("${WORK_DIR}/out/target/product/redroid_arm64") {
                            sh '''
                                mount system.img system -o ro
                                mount vendor.img vendor -o ro
                                tar --xattrs -c vendor -C system --exclude="./vendor" . | docker import --platform=linux/arm64 -c 'ENTRYPOINT ["/init", "qemu=1", "androidboot.hardware=redroid"]' - ${DOCKER_REGISTRY}/redroid:${BUILD_NUMBER}-arm64
                                umount system vendor
                            '''
                        }
                    }
                }
                stage('Create AMD64 Image') {
                    steps {
                        dir("${WORK_DIR}/out/target/product/redroid_x86_64") {
                            sh '''
                                mount system.img system -o ro
                                mount vendor.img vendor -o ro
                                tar --xattrs -c vendor -C system --exclude="./vendor" . | docker import --platform=linux/amd64 -c 'ENTRYPOINT ["/init", "qemu=1", "androidboot.hardware=redroid"]' - ${DOCKER_REGISTRY}/redroid:${BUILD_NUMBER}-amd64
                                umount system vendor
                            '''
                        }
                    }
                }
            }
        }
        
        stage('Push to Docker Registry') {
            steps {
                sh '''
                    docker push ${DOCKER_REGISTRY}/redroid:${BUILD_NUMBER}-arm64
                    docker push ${DOCKER_REGISTRY}/redroid:${BUILD_NUMBER}-amd64
                    
                    # Create and push multi-arch manifest
                    docker manifest create ${DOCKER_REGISTRY}/redroid:${BUILD_NUMBER} \
                        ${DOCKER_REGISTRY}/redroid:${BUILD_NUMBER}-arm64 \
                        ${DOCKER_REGISTRY}/redroid:${BUILD_NUMBER}-amd64
                    docker manifest push ${DOCKER_REGISTRY}/redroid:${BUILD_NUMBER}
                '''
            }
        }
    }
    
    post {
        always {
            sh "rm -rf ${WORK_DIR}"
        }
    }
}
