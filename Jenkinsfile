pipeline {
    agent { label 'maestro' }
    stages {
        stage('Install rust toolchains') {
            steps {
                cache(caches: [ arbitraryFileCache(
                    path: '~/.rustup/toolchains',
                    cacheName: 'rust-toolchains',
                    cacheValidityDecidingFile: 'rust-toolchain.toml')
                ]) {
                    // used for builds with 'cargo install'
                    sh 'rustup toolchain install stable'
                    // install from rust-toolchain.toml
                    sh 'rustup show'
                }
            }
        }
        stage('Lint') {
            matrix {
                axes {
                    axis {
                        name 'DIR'
                        values 'macros', 'utils', 'kernel', 'inttest'
                    }
                }
                stages {
                    stage('Clippy') {
                        steps {
                            sh 'cd $DIR && cargo clippy --all-features --all-targets -- -D warnings'
                        }
                    }
                    stage('Format') {
                        steps {
                            sh 'cd $DIR && cargo fmt --check'
                        }
                    }
                }
            }
        }
        stage('Book') {
            steps {
                cache(caches: [ arbitraryFileCache(
                    path: '~/.cargo/bin',
                    cacheName: 'cargo-bin')
                ]) {
                    sh 'cargo +stable install mdbook mdbook-mermaid'
                }
                sh 'PATH=$HOME/.cargo/bin:$PATH mdbook-mermaid install doc/'
                sh 'PATH=$HOME/.cargo/bin:$PATH mdbook build doc/'
            }
        }
        stage('Documentation') {
            matrix {
                axes {
                    axis {
                        name 'ARCH'
                        values 'x86', 'x86_64'
                    }
                }
                stages {
                    stage('Build references') {
                        steps {
                            dir('kernel') {
                                sh 'cargo doc --target arch/${ARCH}/${ARCH}.json'
                            }
                        }
                    }
                }
            }
        }
        stage('Build') {
            matrix {
                axes {
                    axis {
                        name 'ARCH'
                        values 'x86', 'x86_64'
                    }
                    axis {
                        name 'PROFILE'
                        values 'dev', 'release'
                    }
                    axis {
                        name 'PROFILE_DIR'
                        values 'debug', 'release'
                    }
                }
                stages {
                    stage('Build & Check Multiboot2') {
                        steps {
                            dir('kernel') {
                                sh 'cargo build --target arch/${ARCH}/${ARCH}.json --profile ${PROFILE}'
                                sh 'grub-file --is-x86-multiboot2 target/${ARCH}/${PROFILE_DIR}/maestro'
                            }
                        }
                    }
                }
            }
        }
        stage('Miri') {
            steps {
                dir('utils') {
                    withEnv(['MIRIFLAGS=-Zmiri-disable-stacked-borrows']) {
                        sh 'cargo miri test'
                    }
                }
            }
        }
        stage('Module') {
            matrix {
                axes {
                    axis {
                        name 'MOD'
                        values 'e1000', 'ps2'
                    }
                    axis {
                        name 'ARCH'
                        values 'x86', 'x86_64'
                    }
                    axis {
                        name 'PROFILE'
                        values 'debug', 'release'
                    }
                }
                stages {
                    stage('Clippy') {
                        steps {
                            dir("mod/${MOD}") {
                                withEnv(["ARCH=${ARCH}", "CMD=clippy", "PROFILE=${PROFILE}"]) {
                                    sh '../build -- -D warnings'
                                }
                            }
                        }
                    }
                    stage('Format') {
                        steps {
                            dir("mod/${MOD}") {
                                sh 'cargo fmt --check'
                            }
                        }
                    }
                    stage('Build') {
                        steps {
                            dir("mod/${MOD}") {
                                withEnv(["ARCH=${ARCH}", "PROFILE=${PROFILE}"]) {
                                    sh '../build'
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('Selftest') {
            matrix {
                axes {
                    axis {
                        name 'ARCH'
                        values 'x86', 'x86_64'
                    }
                }
                stages {
                    stage('Run utils tests') {
                        steps {
                            dir('utils') {
                                sh 'cargo test'
                            }
                        }
                    }
                    stage('Run kernel tests') {
                        steps {
                            dir('kernel') {
                                withEnv(["CARGOFLAGS=--target arch/${ARCH}/${ARCH}.json"]) {
                                    sh 'ci/test.sh self'
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('Build inttest') {
            matrix {
                axes {
                    axis {
                        name 'ARCH_KERNEL'
                        values 'x86', 'x86_64'
                    }
                    axis {
                        name 'ARCH_USER'
                        values 'i686-unknown-linux-musl', 'x86_64-unknown-linux-musl'
                    }
                    axis {
                        name 'ARCH_USER_COMPAT'
                        values '', 'i686-unknown-linux-musl'
                    }
                }
                stages {
                    stage('Build tests') {
                        steps {
                            dir('inttest') {
                                withEnv(["ARCH=${ARCH_KERNEL}", "TARGET=${ARCH_USER}"]) {
                                    sh './build.sh'
                                    sh 'mv disk disk-${ARCH_KERNEL}'
                                }
                            }
                        }
                    }
                    stage('Build tests (compat)') {
                        when {
                            expression { ARCH_USER_COMPAT != '' }
                        }
                        steps {
                            dir('inttest') {
                                withEnv(["ARCH=${ARCH_KERNEL}", "TARGET=${ARCH_USER_COMPAT}"]) {
                                    sh './build.sh'
                                    sh 'mv disk disk-${ARCH_KERNEL}-compat'
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('Inttest') {
            matrix {
                axes {
                    axis {
                        name 'ARCH_KERNEL'
                        values 'x86', 'x86_64'
                    }
                    axis {
                        name 'ARCH_USER'
                        values 'i686-unknown-linux-musl', 'x86_64-unknown-linux-musl'
                    }
                    axis {
                        name 'ARCH_USER_COMPAT'
                        values '', 'i686-unknown-linux-musl'
                    }
                    axis {
                        name 'CORES'
                        values '1', '2'
                    }
                }
                stages {
                    stage('Run') {
                        steps {
                            dir('kernel') {
                                withEnv([
                                    "CARGOFLAGS=--target arch/${ARCH_KERNEL}/${ARCH_KERNEL}.json",
                                    "QEMUFLAGS=-smp ${CORES}"
                                ]) {
                                    sh 'cp ../inttest/disk-${ARCH_KERNEL} qemu_disk'
                                    sh 'ci/test.sh int'
                                }
                            }
                        }
                    }
                    stage('Run (compat)') {
                        when {
                            expression { ARCH_USER_COMPAT != '' }
                        }
                        steps {
                            dir('kernel') {
                                withEnv([
                                    "CARGOFLAGS=--target arch/${ARCH_KERNEL}/${ARCH_KERNEL}.json",
                                    "QEMUFLAGS=-smp ${CORES}"
                                ]) {
                                    sh 'cp ../inttest/disk-${ARCH_KERNEL}-compat qemu_disk'
                                    sh 'ci/test.sh int'
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
