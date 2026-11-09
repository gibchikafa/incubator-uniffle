@Library("jenkins-library@main")

import com.logicalclocks.jenkins.k8s.ImageBuilder

// Builds the rss distribution, publishes the shaded client jar to repo.hops.works and
// builds the hopsworks/rss, hopsworks/rss-controller and hopsworks/rss-webhook images.
//
// The images go through the shared ImageBuilder, like the hive and docker-images jobs:
// it builds and pushes each image to every registry it is configured with (the private
// registry and the public ECR dev and prod registries) and creates a missing ECR
// repository. The tag is version.txt. A version that the private registry already holds
// is not built again: the job stops before the distribution build, so the jar under
// /opt/repository/master/rss/<version>/ and the images always come from the same commit.
// Bump version.txt for a change that should ship, or run with FORCE_BUILD to republish
// the current version everywhere.
//
// Written with plain loops and String methods on purpose: the Jenkins script sandbox
// does not approve the Groovy collection helpers (collect, findAll, find, plus on a
// list) by default, and a build would stop on the first use until an administrator
// approves the signature. Same rule as hopsworks-ee/docker/Jenkinsfile.
properties([
  parameters([
    booleanParam(
      name: 'FORCE_BUILD',
      defaultValue: false,
      description: 'Build and publish even if the private registry already holds hopsworks/rss:<version.txt>. The jar on repo.hops.works and the images in every registry are overwritten with this commit; a registry that refuses to overwrite a tag still wins.'
    )
  ])
])

// Look up a registry by id without relying on List.find being approved.
def registryById(builder, String id) {
  for (def r : builder.REGISTRIES) {
    if (r.id == id) {
      return r
    }
  }
  return null
}

// image-build.env, written by build.sh --prepare-only: one KEY=VALUE per line, the
// build arguments of the rss image. Returned as the list withEnv takes.
def readImageBuildEnv(String path) {
  def envList = []
  def lines = readFile(path).split("\n")
  for (int i = 0; i < lines.length; i++) {
    def line = lines[i].trim()
    if (line && !line.startsWith('#')) {
      envList.add(line)
    }
  }
  return envList
}

node("local") {
  def uniffleVersion = "0.10.1"
  // The Hadoop distribution shipped in the rss image.
  def hadoopVersion = "3.2.0.15-EE-SNAPSHOT"

  stage('Clone repository') {
      checkout scm
  }

  def version = readFile("${env.WORKSPACE}/version.txt").trim()
  def builder = new ImageBuilder(this)
  def publish = true

  stage('Check published version') {
    // The private registry stands for all of them: it is the first one ImageBuilder
    // pushes to, so a version it holds has been through this job before.
    def plocal = registryById(builder, "plocal")
    if (plocal == null) {
      error("ImageBuilder registers no registry with id plocal; cannot tell whether ${version} is published")
    }
    plocal.auth()
    def published = builder.imageExists(plocal.buildImageName("hopsworks/rss", version))
    if (published && !params.FORCE_BUILD) {
      publish = false
      echo "SKIP  hopsworks/rss:${version} is already published. Bump version.txt to ship this commit, or run with FORCE_BUILD."
    } else if (published) {
      // FORCE_BUILD means the current version is rewritten everywhere. Leaving the
      // check on would skip every registry that holds the tag, so the jar would move
      // and the images would not.
      builder.immutable = false
      echo "FORCE hopsworks/rss:${version} is already published and will be rebuilt."
    } else {
      echo "BUILD hopsworks/rss:${version} is not published yet."
    }
  }

  if (publish) {
    withCredentials([usernamePassword(credentialsId: 'a0770738-4ef3-4acc-a6ba-097ee6c85b44', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
      stage('Build distribution') {
        sh """
          set -ex
          git status

          docker run --rm -v .:/incubator-uniffle -w /incubator-uniffle  eclipse-temurin:8-jdk /bin/bash build_distribution.sh --spark3-profile spark3.5 --hadoop-profile hadoop3.2 --without-mr --without-tez --without-spark2

          mkdir -p /opt/repository/master/rss/$version/
          cp  client-spark/spark3-shaded/target/rss-client-spark3-shaded-${uniffleVersion}.jar /opt/repository/master/rss/${version}/rss-client-spark3-shaded-${version}.jar
        """
      }

      stage('Prepare rss image context') {
        // Downloads the Hadoop distribution, stages the rss tarball next to the Dockerfile
        // and writes the build arguments to image-build.env. The image itself is built below.
        dir('deploy/kubernetes/docker') {
          sh """
            set -ex
            ./build.sh --hadoop-version ${hadoopVersion} --nexus-user \$USERNAME --nexus-password \$PASSWORD --prepare-only true
            cat image-build.env
          """
        }
      }

      stage('Build and push images') {
        // The rss manifest lists the image-build.env keys as its `variables`, which fill
        // the ${VAR} placeholders of its extraDockerArgs; the --build-arg flags themselves
        // are written in extraDockerArgs.
        def imageBuildEnv = readImageBuildEnv("${env.WORKSPACE}/deploy/kubernetes/docker/image-build.env")
        imageBuildEnv.add("RSS_IMAGE_VERSION=${version}")

        withEnv(imageBuildEnv) {
          dir('deploy/kubernetes/docker') {
            builder.run(readFile('build-manifest.json'))
          }
          // The operator Dockerfile lives in hack/ but builds from the module root, so the
          // manifest keeps the build in this directory (changeDirectory false).
          dir('deploy/kubernetes/operator') {
            builder.run(readFile('build-manifest.json'))
          }
        }
      }
    }
  }
}
