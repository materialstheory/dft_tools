def triqsBranch = env.CHANGE_TARGET ?: env.BRANCH_NAME
def triqsProject = '/TRIQS/triqs/' + triqsBranch.replaceAll('/', '%2F')

properties([
  disableConcurrentBuilds(),
  buildDiscarder(logRotator(numToKeepStr: '10', daysToKeepStr: '30')),
  pipelineTriggers([
    upstream(
      threshold: 'SUCCESS',
      upstreamProjects: triqsProject
    )
  ])
])

def platforms = [:]

def dockerPlatforms = ["ubuntu-clang", "ubuntu-gcc", "centos-gcc"]
for (int i = 0; i < dockerPlatforms.size(); i++) {
  def platform = dockerPlatforms[i]
  platforms[platform] = { -> stage(platform) {
    timeout(time: 1, unit: 'HOURS') {
      node('docker') {
        checkout scm
        /* construct a Dockerfile for this base */
        sh """
          ( echo "FROM flatironinstitute/triqs:${triqsBranch}-${env.STAGE_NAME}" ; sed '0,/^FROM /d' Dockerfile ) > Dockerfile.jenkins
          mv -f Dockerfile.jenkins Dockerfile
        """
        /* build and tag */
        def img = docker.build("flatironinstitute/dft_tools:${env.BRANCH_NAME}-${env.STAGE_NAME}")
        /* but we don't need the tag so clean it up (alternatively, could refacter to run in container) */
        sh "docker rmi ${img.imageName()}"
      }
    }
  } }
}

def osxPlatforms = [
  ["gcc", ['CC=gcc-7', 'CXX=g++-7']],
  ["clang", ['CC=/usr/local/opt/llvm/bin/clang', 'CXX=/usr/local/opt/llvm/bin/clang++', 'CXXFLAGS=-I/usr/local/opt/llvm/include', 'LDFLAGS=-L/usr/local/opt/llvm/lib']]
]
for (int i = 0; i < osxPlatforms.size(); i++) {
  def platformEnv = osxPlatforms[i]
  def platform = platformEnv[0]
  platforms["osx-$platform"] = { -> stage("osx-$platform") {
    timeout(time: 1, unit: 'HOURS') {
      node('osx && triqs') {
        def srcDir = pwd()
        def tmpDir = pwd(tmp:true)
        def buildDir = "$tmpDir/build"
        def installDir = "$tmpDir/install"

        dir(installDir) {
          deleteDir()
        }

        copyArtifacts(projectName: triqsProject, selector: upstream(fallbackToLastSuccessful: true), filter: "osx-${platform}.zip")
        unzip(zipFile: "osx-${platform}.zip", dir: installDir)
        /* fixup zip-stripped permissions (JENKINS-13128) */
        sh "chmod +x $installDir/bin/*"

        checkout scm

        dir(buildDir) { withEnv(platformEnv[1]+[
            "PATH=$installDir/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin",
            "CPATH=$installDir/include",
            "LIBRARY_PATH=$installDir/lib",
            "CMAKE_PREFIX_PATH=$installDir/share/cmake"]) {
          deleteDir()
          sh "cmake $srcDir -DTRIQS_ROOT=$installDir"
          sh "make -j2"
          try {
            sh "make test"
          } catch (exc) {
            archiveArtifacts(artifacts: 'Testing/Temporary/LastTest.log')
            throw exc
          }
          sh "make install"
        } }
        // zip(zipFile: "osx-${platform}.zip", archive: true, dir: installDir)
      }
    }
  } }
}

try {
  parallel platforms
} catch (err) {
  emailext(
    subject: "\$PROJECT_NAME - Build # \$BUILD_NUMBER - FAILED",
    body: """\$PROJECT_NAME - Build # \$BUILD_NUMBER - FAILED

$err

Check console output at \$BUILD_URL to view full results.

Building \$BRANCH_NAME for \$CAUSE
\$JOB_DESCRIPTION

Chages:
\$CHANGES

End of build log:
\${BUILD_LOG,maxLines=60}
    """,
    to: 'mzingl@flatironinstitute.org, hstrand@flatironinstitute.org, nils.wentzell@gmail.com, dsimon@flatironinstitute.org',
    recipientProviders: [
      [$class: 'DevelopersRecipientProvider'],
    ],
    replyTo: '$DEFAULT_REPLYTO'
  )
  throw err
}