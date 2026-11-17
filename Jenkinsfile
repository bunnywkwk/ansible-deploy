// CD for ansible-deploy: this repo's own pipeline (Jenkins job cd/ansible-deploy), separate from the role CI.
//
// What runs when:
//   Branch or pull request   Version, YAML lint, Ansible lint
//   Merge to main            Release: build deploy-<VERSION>.tar.gz + .sha256, publish to the repo server, tag v<VERSION>
// Role code is not tested here: requirements.yml pins role tags that the role CI already tested and released.
// A release is never replaced: an existing tag skips the release, an existing releases/<VERSION>/ fails it.
pipeline {
  agent { label 'ansible' }

  options {
    timestamps()
    timeout(time: 30, unit: 'MINUTES')
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '30'))
  }

  environment {
    // Plain console log, no colour codes
    ANSIBLE_FORCE_COLOR = '0'
    PY_COLORS = '0'
    NO_COLOR = '1'
  }

  stages {
    // Every build checks VERSION, so a bad one fails its pull request, not main after the merge.
    stage('Version') {
      steps {
        sh '''
          V=$(cat VERSION)
          echo "$V" | grep -qxE '[0-9]+[.][0-9]+[.][0-9]+' || { echo "VERSION must be X.Y.Z, got '$V'"; exit 1; }
          echo "VERSION $V"
        '''
      }
    }

    stage('YAML lint') {
      when { not { branch 'main' } }
      steps {
        sh '''
          . /opt/ci/bin/activate
          yamllint -f parsable .
          echo "yamllint: no problems found"
        '''
      }
    }

    // Also checks the play's syntax; it fetches the pinned roles from requirements.yml to read their tasks.
    stage('Ansible lint') {
      when { not { branch 'main' } }
      steps {
        sh '''
          . /opt/ci/bin/activate
          ansible-lint --version | head -1
          ansible-lint --profile production
        '''
      }
    }

    // main only: the PR already linted this exact code.
    stage('Release') {
      when { branch 'main' }
      steps {
        withCredentials([usernamePassword(credentialsId: 'github-crendentials',
                                          usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN'),
                         sshUserPrivateKey(credentialsId: 'repo-server-publish', keyFileVariable: 'KEY'),
                         // Repo server address from Jenkins, not the repo (Jenkins masks it as **** in the log)
                         string(credentialsId: 'repo-id', variable: 'REPO_SERVER')]) {
          sh '''
            . /opt/ci/bin/activate
            # Only a PR merge commit (two parents) was linted; a direct commit is not.
            if ! git rev-parse -q --verify 'HEAD^2' >/dev/null; then echo "Not a pull request merge: no release."; exit 0; fi
            V=$(cat VERSION)
            # Token read from the environment by git, never put in a URL or the log
            H='!f() { echo "username=$GIT_USER"; echo "password=$GIT_TOKEN"; }; f'
            if git -c credential.helper= -c credential.helper="$H" ls-remote --exit-code --tags origin "refs/tags/v$V" >/dev/null; then
              echo "Tag v$V already exists: release skipped. Bump VERSION in a pull request to release."; exit 0
            fi

            # Build: everything the machine needs, so it never touches the internet
            B=build/$V
            rm -rf build && mkdir -p "$B"
            cp -r ansible.cfg requirements.yml requirements-ansible.txt sysconfig playbooks "$B/"
            echo "$V" > "$B/RELEASE"
            ansible-galaxy role install -r requirements.yml -p "$B/roles"
            ansible-galaxy collection install -r requirements.yml -p "$B/collections"
            # ansible-core for Python 3.11 on RHEL 9 (glibc 2.34), not for this build node
            pip download -d "$B/wheels" -r requirements-ansible.txt --only-binary=:all: --python-version 3.11 \
              --platform manylinux_2_34_x86_64 --platform manylinux_2_28_x86_64 --platform manylinux2014_x86_64
            tar -czf "build/deploy-$V.tar.gz" -C "$B" .
            (cd build && sha256sum "deploy-$V.tar.gz" > "deploy-$V.tar.gz.sha256")
            cat "build/deploy-$V.tar.gz.sha256"

            # Publish: upload to a temporary folder, then rename, so an install never sees half a release
            # shortcut: accept-new trusts the repo server's host key on first contact; pin it in known_hosts before real use.
            SSH="ssh -i $KEY -o BatchMode=yes -o StrictHostKeyChecking=accept-new publish@$REPO_SERVER"
            D=/var/www/html/releases/$V
            $SSH "test -e $D" && { echo "releases/$V already exists on the repo server: a release is never replaced."; exit 1; }
            $SSH "rm -rf $D.tmp && mkdir $D.tmp"
            scp -i "$KEY" -o BatchMode=yes "build/deploy-$V.tar.gz" "build/deploy-$V.tar.gz.sha256" "publish@$REPO_SERVER:$D.tmp/"
            $SSH "mv $D.tmp $D"
            curl -fsSI "http://$REPO_SERVER/releases/$V/deploy-$V.tar.gz" | head -1

            # Tag last: a tag means "published". If this push fails, push the tag by hand; a rerun stops at the
            # existing release folder.
            git -c user.name=jenkins -c user.email=jenkins@localhost tag -fa "v$V" -m "Release v$V"
            git -c credential.helper= -c credential.helper="$H" push origin "refs/tags/v$V"
            echo "Released v$V: http://$REPO_SERVER/releases/$V/"
          '''
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'build/deploy-*', allowEmptyArchive: true
    }
  }
}
