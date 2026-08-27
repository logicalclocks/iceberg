/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *    http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

// Builds the one artifact Hopsworks ships from this fork: the Spark 4.1 Iceberg
// runtime jar that spark-feature-pipeline puts on the Spark classpath. Nothing
// depends on io.hops Iceberg through a POM, so unlike the Hudi job this one does
// no Maven deploy -- it only drops the jar into the repo.hops.works tree.

pipeline {
  agent { label 'local' }

  options {
    disableConcurrentBuilds()
    skipDefaultCheckout(true)
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  parameters {
    string(name: 'BRANCH_TO_BUILD', defaultValue: 'branch-1.11.0', description: 'Git branch to build.')
    booleanParam(name: 'FORCE_UPDATE', defaultValue: false, description: 'Pass --refresh-dependencies to Gradle to re-resolve cached dependencies. Leave off for normal builds.')
  }

  environment {
    // The Gradle wrapper pins 8.14.4, so the image only has to supply a JDK 17.
    // No git binary is needed: gradle-git-properties reads .git through JGit.
    DOCKER_IMAGE = 'eclipse-temurin:17-jdk'
    HOST_GRADLE_HOME = '/home/jenkinsmaster/.gradle-iceberg'
    CONTAINER_GRADLE_HOME = '/gradle-home'
    GRADLE_OPTS = '-Xmx4G'
    ICEBERG_REPOSITORY = '/opt/repository/master/iceberg'
    RUNTIME_PROJECT = ':iceberg-spark:iceberg-spark-runtime-4.1_2.13'
    // Only the Spark 4.1 runtime ships. Clearing flinkVersions and kafkaVersions keeps
    // those subprojects out of the configured build instead of merely unbuilt.
    BUILD_MATRIX_ARGS = '-DsparkVersions=4.1 -DflinkVersions= -DkafkaVersions='
  }

  stages {
    stage('Checkout') {
      steps {
        deleteDir()
        // Repo URL and credentials come from the job's SCM configuration
        // ("Pipeline script from SCM"); only the branch is parameterized.
        // The checkout must keep .git: the build fails without it
        // (generateGitProperties sets failOnNoGitDirectory = true).
        checkout([$class: 'GitSCM',
          branches: [[name: "${params.BRANCH_TO_BUILD}"]],
          userRemoteConfigs: scm.userRemoteConfigs
        ])
      }
    }

    stage('Resolve Version') {
      steps {
        sh '''#!/bin/bash -eu
          # version.txt is the single source of truth. Without it Iceberg derives the
          # version from git describe and bumps the MINOR, so a branch one commit past
          # apache-iceberg-1.11.0 would build a jar named 1.12.0-SNAPSHOT.
          test -s version.txt
          tr -d '[:space:]' < version.txt > version.log
          echo "ICEBERG_VERSION=$(cat version.log)"
        '''
      }
    }

    stage('Build Runtime Jar') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'a0770738-4ef3-4acc-a6ba-097ee6c85b44', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
          sh '''#!/bin/bash -eu
            UPDATE_ARG=""
            if [ "$FORCE_UPDATE" = "true" ]; then
              UPDATE_ARG="--refresh-dependencies"
            fi

            mkdir -p "$HOST_GRADLE_HOME"

            docker run --rm \
              -u "$(id -u):$(id -g)" \
              -v "$WORKSPACE:$WORKSPACE" \
              -v "$HOST_GRADLE_HOME:$CONTAINER_GRADLE_HOME" \
              -w "$WORKSPACE" \
              -e HOME=/tmp \
              -e GRADLE_USER_HOME="$CONTAINER_GRADLE_HOME" \
              -e GRADLE_OPTS="$GRADLE_OPTS" \
              -e RUNTIME_PROJECT="$RUNTIME_PROJECT" \
              -e BUILD_MATRIX_ARGS="$BUILD_MATRIX_ARGS" \
              -e UPDATE_ARG="$UPDATE_ARG" \
              -e HOPS_USER="$USERNAME" \
              -e HOPS_PASSWORD="$PASSWORD" \
              "$DOCKER_IMAGE" \
              bash -lc '
                set -eu
                JAVA_VERSION="$(java -XshowSettings:properties -version 2>&1 | awk -F"= " "/java.specification.version =/{print \\$2; exit}")"
                if [ "$JAVA_VERSION" != "17" ]; then
                  echo "Java 17 is required, but this image reports java.specification.version=$JAVA_VERSION" >&2
                  exit 1
                fi
                # HOPS_USER/HOPS_PASSWORD authenticate the nexus.hops.works repository that
                # serves io.hops.hive and io.hops hadoop. Resolution fails outright without them.
                ./gradlew --no-daemon $UPDATE_ARG $BUILD_MATRIX_ARGS \
                  "${RUNTIME_PROJECT}:shadowJar" -x test -x integrationTest
              '
          '''
        }
      }
    }

    stage('Publish Bundle Jar') {
      steps {
        sh '''#!/bin/bash -eu
          ICEBERG_VERSION="$(cat version.log)"
          JAR="iceberg-spark-runtime-4.1_2.13-${ICEBERG_VERSION}.jar"
          BUILT="spark/v4.1/spark-runtime/build/libs/${JAR}"
          TARGET_DIR="$ICEBERG_REPOSITORY/$ICEBERG_VERSION"

          # Fail loudly rather than publishing nothing: a version.txt that disagrees with
          # what Gradle produced would otherwise leave the target directory empty.
          test -f "$BUILT"

          mkdir -p "$TARGET_DIR"
          cp "$BUILT" "$TARGET_DIR/"

          ls -l "$TARGET_DIR/$JAR"
        '''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'version.log', allowEmptyArchive: true
    }
  }
}
